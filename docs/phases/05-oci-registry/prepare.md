# Phase 5 事前学習ガイド - OCI レジストリ

このドキュメントは、Phase 5でOCI準拠のコンテナレジストリを実装するために必要な基礎知識をまとめたものです。

## 1. OCI Distribution Specification

### 1.1 Distribution Specificationとは

**定義と目的**
- コンテナイメージの配布方法を標準化
- レジストリ間の相互運用性を実現
- プッシュ・プル操作のプロトコル統一

**仕様の範囲**
- RESTful HTTPインターフェース
- イメージの格納形式
- 認証・認可機能
- エラーハンドリング

### 1.2 主要なAPI エンドポイント

#### 1. バージョン確認
```
GET /v2/
```
- レジストリのバージョン情報を取得
- 200 OKでサポートを確認

#### 2. イメージマニフェスト操作
```
GET /v2/<name>/manifests/<reference>
PUT /v2/<name>/manifests/<reference>
DELETE /v2/<name>/manifests/<reference>
```
- イメージのメタデータを管理
- referenceはタグまたはダイジェスト

#### 3. ブロブ操作
```
GET /v2/<name>/blobs/<digest>
POST /v2/<name>/blobs/uploads/
PUT /v2/<name>/blobs/uploads/<uuid>
DELETE /v2/<name>/blobs/<digest>
```
- イメージレイヤーの実データを管理
- チャンクアップロード対応

## 2. Content-Addressable Storage

### 2.1 コンテンツアドレッシング

**基本概念**
- コンテンツのハッシュ値をアドレスとして使用
- データの整合性を保証
- 重複排除が自動的に実現

**ハッシュ関数**
```go
import (
    "crypto/sha256"
    "fmt"
    "io"
)

func calculateDigest(data io.Reader) (string, error) {
    hash := sha256.New()
    if _, err := io.Copy(hash, data); err != nil {
        return "", err
    }
    
    return fmt.Sprintf("sha256:%x", hash.Sum(nil)), nil
}
```

### 2.2 ストレージ設計

**ディレクトリ構造**
```
registry/
├── repositories/
│   └── <name>/
│       ├── _manifests/
│       │   └── revisions/
│       │       └── sha256/
│       │           └── <digest>/
│       └── _uploads/
└── blobs/
    └── sha256/
        └── <digest>/
            └── data
```

**Go実装例**
```go
type BlobStore struct {
    root string
    mu   sync.RWMutex
}

func (bs *BlobStore) Put(digest string, data io.Reader) error {
    path := bs.blobPath(digest)
    
    if err := os.MkdirAll(filepath.Dir(path), 0755); err != nil {
        return err
    }
    
    file, err := os.Create(path)
    if err != nil {
        return err
    }
    defer file.Close()
    
    // データの書き込みと同時にハッシュ値を計算
    hash := sha256.New()
    writer := io.MultiWriter(file, hash)
    
    if _, err := io.Copy(writer, data); err != nil {
        return err
    }
    
    // ハッシュ値の検証
    calculatedDigest := fmt.Sprintf("sha256:%x", hash.Sum(nil))
    if calculatedDigest != digest {
        os.Remove(path)
        return fmt.Errorf("digest mismatch: expected %s, got %s", 
            digest, calculatedDigest)
    }
    
    return nil
}

func (bs *BlobStore) Get(digest string) (io.ReadCloser, error) {
    path := bs.blobPath(digest)
    return os.Open(path)
}

func (bs *BlobStore) blobPath(digest string) string {
    return filepath.Join(bs.root, "blobs", digest[:7], digest[7:])
}
```

## 3. HTTP API実装

### 3.1 基本的なHTTPハンドラー

```go
type Registry struct {
    blobStore     *BlobStore
    manifestStore *ManifestStore
    auth          Authenticator
}

func (r *Registry) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // CORS ヘッダーの設定
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
    
    if req.Method == "OPTIONS" {
        return
    }
    
    // ルーティング
    switch {
    case req.URL.Path == "/v2/":
        r.handleVersion(w, req)
    case strings.HasPrefix(req.URL.Path, "/v2/") && strings.Contains(req.URL.Path, "/manifests/"):
        r.handleManifests(w, req)
    case strings.HasPrefix(req.URL.Path, "/v2/") && strings.Contains(req.URL.Path, "/blobs/"):
        r.handleBlobs(w, req)
    default:
        http.NotFound(w, req)
    }
}
```

### 3.2 マニフェスト操作

```go
func (r *Registry) handleManifests(w http.ResponseWriter, req *http.Request) {
    parts := strings.Split(req.URL.Path, "/")
    if len(parts) < 5 {
        http.Error(w, "Invalid path", http.StatusBadRequest)
        return
    }
    
    name := parts[2]
    reference := parts[4]
    
    switch req.Method {
    case "GET":
        r.getManifest(w, req, name, reference)
    case "PUT":
        r.putManifest(w, req, name, reference)
    case "DELETE":
        r.deleteManifest(w, req, name, reference)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (r *Registry) getManifest(w http.ResponseWriter, req *http.Request, name, reference string) {
    manifest, err := r.manifestStore.Get(name, reference)
    if err != nil {
        if os.IsNotExist(err) {
            http.Error(w, "Manifest not found", http.StatusNotFound)
        } else {
            http.Error(w, "Internal server error", http.StatusInternalServerError)
        }
        return
    }
    
    w.Header().Set("Content-Type", "application/vnd.docker.distribution.manifest.v2+json")
    w.Header().Set("Docker-Content-Digest", manifest.Digest)
    w.Write(manifest.Data)
}
```

### 3.3 ブロブ操作

```go
func (r *Registry) handleBlobs(w http.ResponseWriter, req *http.Request) {
    parts := strings.Split(req.URL.Path, "/")
    if len(parts) < 5 {
        http.Error(w, "Invalid path", http.StatusBadRequest)
        return
    }
    
    name := parts[2]
    
    switch req.Method {
    case "GET":
        r.getBlob(w, req, name, parts[4])
    case "POST":
        r.startBlobUpload(w, req, name)
    case "PUT":
        r.completeBlobUpload(w, req, name)
    case "DELETE":
        r.deleteBlob(w, req, name, parts[4])
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (r *Registry) getBlob(w http.ResponseWriter, req *http.Request, name, digest string) {
    reader, err := r.blobStore.Get(digest)
    if err != nil {
        if os.IsNotExist(err) {
            http.Error(w, "Blob not found", http.StatusNotFound)
        } else {
            http.Error(w, "Internal server error", http.StatusInternalServerError)
        }
        return
    }
    defer reader.Close()
    
    w.Header().Set("Content-Type", "application/octet-stream")
    w.Header().Set("Docker-Content-Digest", digest)
    
    // Range リクエストの処理
    if rangeHeader := req.Header.Get("Range"); rangeHeader != "" {
        r.handleRangeRequest(w, req, reader, rangeHeader)
        return
    }
    
    io.Copy(w, reader)
}
```

## 4. チャンクアップロード

### 4.1 チャンクアップロードの流れ

1. POST /v2/<name>/blobs/uploads/ でアップロード開始
2. PUT /v2/<name>/blobs/uploads/<uuid> でチャンクを送信
3. 必要に応じて複数回チャンクを送信
4. 最後にdigestパラメータ付きでアップロード完了

### 4.2 実装例

```go
type Upload struct {
    UUID   string
    Name   string
    Offset int64
    Size   int64
    File   *os.File
}

type UploadManager struct {
    uploads map[string]*Upload
    mu      sync.RWMutex
}

func (um *UploadManager) StartUpload(name string) (*Upload, error) {
    um.mu.Lock()
    defer um.mu.Unlock()
    
    uuid := generateUUID()
    tempFile, err := ioutil.TempFile("", "upload-"+uuid)
    if err != nil {
        return nil, err
    }
    
    upload := &Upload{
        UUID: uuid,
        Name: name,
        File: tempFile,
    }
    
    um.uploads[uuid] = upload
    return upload, nil
}

func (um *UploadManager) AppendChunk(uuid string, data io.Reader) error {
    um.mu.Lock()
    upload, exists := um.uploads[uuid]
    um.mu.Unlock()
    
    if !exists {
        return fmt.Errorf("upload not found: %s", uuid)
    }
    
    n, err := io.Copy(upload.File, data)
    if err != nil {
        return err
    }
    
    upload.Offset += n
    return nil
}

func (um *UploadManager) CompleteUpload(uuid, digest string) error {
    um.mu.Lock()
    upload, exists := um.uploads[uuid]
    delete(um.uploads, uuid)
    um.mu.Unlock()
    
    if !exists {
        return fmt.Errorf("upload not found: %s", uuid)
    }
    defer upload.File.Close()
    defer os.Remove(upload.File.Name())
    
    // ファイルの先頭に戻る
    upload.File.Seek(0, 0)
    
    // BlobStoreに保存
    return um.blobStore.Put(digest, upload.File)
}
```

## 5. 認証・認可

### 5.1 Bearer Token認証

```go
type JWTAuthenticator struct {
    secretKey []byte
    realm     string
    service   string
}

func (ja *JWTAuthenticator) Authenticate(req *http.Request) (*User, error) {
    authHeader := req.Header.Get("Authorization")
    if authHeader == "" {
        return nil, fmt.Errorf("missing authorization header")
    }
    
    if !strings.HasPrefix(authHeader, "Bearer ") {
        return nil, fmt.Errorf("invalid authorization header format")
    }
    
    tokenString := strings.TrimPrefix(authHeader, "Bearer ")
    
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return ja.secretKey, nil
    })
    
    if err != nil || !token.Valid {
        return nil, fmt.Errorf("invalid token")
    }
    
    if claims, ok := token.Claims.(jwt.MapClaims); ok {
        return &User{
            Name:  claims["sub"].(string),
            Scope: claims["scope"].(string),
        }, nil
    }
    
    return nil, fmt.Errorf("invalid token claims")
}

func (ja *JWTAuthenticator) AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
        user, err := ja.Authenticate(req)
        if err != nil {
            ja.requestAuth(w, req)
            return
        }
        
        // 権限チェック
        if !ja.checkPermission(user, req) {
            http.Error(w, "Forbidden", http.StatusForbidden)
            return
        }
        
        next.ServeHTTP(w, req)
    })
}

func (ja *JWTAuthenticator) requestAuth(w http.ResponseWriter, req *http.Request) {
    realm := fmt.Sprintf(`Bearer realm="%s",service="%s"`, ja.realm, ja.service)
    w.Header().Set("WWW-Authenticate", realm)
    w.WriteHeader(http.StatusUnauthorized)
}
```

### 5.2 Basic認証

```go
func basicAuth(username, password string) bool {
    // ハッシュ化されたパスワードとの比較
    hashedPassword, exists := users[username]
    if !exists {
        return false
    }
    
    return bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password)) == nil
}

func (r *Registry) BasicAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
        username, password, ok := req.BasicAuth()
        if !ok || !basicAuth(username, password) {
            w.Header().Set("WWW-Authenticate", `Basic realm="Registry"`)
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        next.ServeHTTP(w, req)
    })
}
```

## 6. エラーハンドリングと仕様準拠

### 6.1 標準エラーレスポンス

```go
type ErrorResponse struct {
    Errors []ErrorDetail `json:"errors"`
}

type ErrorDetail struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Detail  interface{} `json:"detail,omitempty"`
}

func writeErrorResponse(w http.ResponseWriter, code int, errorCode, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    
    response := ErrorResponse{
        Errors: []ErrorDetail{
            {
                Code:    errorCode,
                Message: message,
            },
        },
    }
    
    json.NewEncoder(w).Encode(response)
}

// 使用例
func (r *Registry) handleNotFound(w http.ResponseWriter) {
    writeErrorResponse(w, http.StatusNotFound, "MANIFEST_UNKNOWN", "manifest unknown")
}

func (r *Registry) handleBlobNotFound(w http.ResponseWriter) {
    writeErrorResponse(w, http.StatusNotFound, "BLOB_UNKNOWN", "blob unknown to registry")
}
```

### 6.2 Range リクエストの処理

```go
func (r *Registry) handleRangeRequest(w http.ResponseWriter, req *http.Request, reader io.Reader, rangeHeader string) {
    // Range: bytes=0-499
    ranges := strings.TrimPrefix(rangeHeader, "bytes=")
    parts := strings.Split(ranges, "-")
    
    if len(parts) != 2 {
        http.Error(w, "Invalid range", http.StatusBadRequest)
        return
    }
    
    start, err := strconv.ParseInt(parts[0], 10, 64)
    if err != nil {
        http.Error(w, "Invalid range start", http.StatusBadRequest)
        return
    }
    
    var end int64
    if parts[1] != "" {
        end, err = strconv.ParseInt(parts[1], 10, 64)
        if err != nil {
            http.Error(w, "Invalid range end", http.StatusBadRequest)
            return
        }
    }
    
    // シーカブルなリーダーの場合
    if seeker, ok := reader.(io.Seeker); ok {
        seeker.Seek(start, 0)
        
        w.Header().Set("Content-Range", fmt.Sprintf("bytes %d-%d/*", start, end))
        w.WriteHeader(http.StatusPartialContent)
        
        if end > 0 {
            io.CopyN(w, reader, end-start+1)
        } else {
            io.Copy(w, reader)
        }
    } else {
        http.Error(w, "Range not supported", http.StatusRequestedRangeNotSatisfiable)
    }
}
```

## 7. パフォーマンス最適化

### 7.1 キャッシュ機能

```go
type CacheEntry struct {
    Data       []byte
    Timestamp  time.Time
    Expiry     time.Time
}

type MemoryCache struct {
    entries map[string]*CacheEntry
    mu      sync.RWMutex
    ttl     time.Duration
}

func (mc *MemoryCache) Get(key string) ([]byte, bool) {
    mc.mu.RLock()
    defer mc.mu.RUnlock()
    
    entry, exists := mc.entries[key]
    if !exists || time.Now().After(entry.Expiry) {
        return nil, false
    }
    
    return entry.Data, true
}

func (mc *MemoryCache) Set(key string, data []byte) {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    
    mc.entries[key] = &CacheEntry{
        Data:      data,
        Timestamp: time.Now(),
        Expiry:    time.Now().Add(mc.ttl),
    }
}

func (mc *MemoryCache) Cleanup() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            mc.mu.Lock()
            now := time.Now()
            for key, entry := range mc.entries {
                if now.After(entry.Expiry) {
                    delete(mc.entries, key)
                }
            }
            mc.mu.Unlock()
        }
    }
}
```

### 7.2 並行処理最適化

```go
type WorkerPool struct {
    workers    int
    jobQueue   chan Job
    quit       chan bool
    wg         sync.WaitGroup
}

type Job struct {
    Handler func() error
    Result  chan error
}

func NewWorkerPool(workers int) *WorkerPool {
    return &WorkerPool{
        workers:  workers,
        jobQueue: make(chan Job, workers*2),
        quit:     make(chan bool),
    }
}

func (wp *WorkerPool) Start() {
    for i := 0; i < wp.workers; i++ {
        wp.wg.Add(1)
        go wp.worker()
    }
}

func (wp *WorkerPool) worker() {
    defer wp.wg.Done()
    
    for {
        select {
        case job := <-wp.jobQueue:
            err := job.Handler()
            job.Result <- err
        case <-wp.quit:
            return
        }
    }
}

func (wp *WorkerPool) Submit(handler func() error) error {
    job := Job{
        Handler: handler,
        Result:  make(chan error, 1),
    }
    
    wp.jobQueue <- job
    return <-job.Result
}
```

## 8. 参考資料

### 8.1 必読ドキュメント

- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec)
- [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)
- [JWT Authentication](https://jwt.io/introduction/)

### 8.2 参考実装

- [Distribution (Docker Registry)](https://github.com/distribution/distribution)
- [Harbor](https://github.com/goharbor/harbor)
- [Quay](https://github.com/quay/quay)

### 8.3 理解度確認チェックリスト

- [ ] OCI Distribution APIの仕様を理解している
- [ ] Content-Addressable Storageの概念を理解している
- [ ] チャンクアップロードとBlobの処理を実装できる
- [ ] HTTP Range Requestsを処理できる
- [ ] 認証・認可機能を実装できる
- [ ] エラーハンドリングの仕様に準拠している

これらの知識を身につけることで、Phase 5のOCI レジストリ実装に効果的に取り組むことができます。

---

## 補助教材

Phase 5をより深く理解するために、以下の補助教材を参照することをお勧めします：

- **[OCI Specifications - 理論的理解](../../resources/02-container-fundamentals/oci-specifications.md)**
  - Distribution Specificationの詳細
  - RESTful APIの設計原則
  - 標準化の理論的背景

- **[Service Discovery Patterns - 理論的理解](../../resources/05-distributed-systems/service-discovery-patterns.md)**
  - レジストリサービスの理論
  - 分散システムでの統合パターン
  - 可用性とパフォーマンスの権衡

これらの資料は、Phase 5で実装するOCIレジストリの理論的基盤を深く理解するのに役立ちます。特に、分散システムにおけるレジストリの役割と、標準化された配布プロトコルの重要性を学ぶことができます。