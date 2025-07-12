# Phase 5: 独自OCI Registry - 詳細学習計画

## 学習目標
OCI Distribution Specification準拠のコンテナレジストリを実装し、コンテナイメージの保存・配布・管理を行うHTTP APIサーバーとContent-Addressable Storage（CAS）システムを構築する

## 事前準備

### 環境要件確認
- Phase 4の完了（ポートマッピングとサービス公開）
- HTTP サーバー開発の基礎知識
- SHA256ハッシュの理解
- JSON/RESTful API の設計経験

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- OCI Distribution Specificationとは？Docker Registryとの関係
- Content-Addressable Storage（CAS）の概念と利点
- HTTP Range Requestsの仕組み
- Docker Image Manifestの構造
- Blobとレイヤーの関係

## Step 1: OCI Distribution Specificationの理解

### このステップの目的
OCI Distribution Specの詳細を理解し、実装すべきAPIエンドポイントとデータ構造を把握する

### 調査すべき内容
- OCI Distribution Spec v1.0の全APIエンドポイント
- Image ManifestとManifest Listの違い
- Blobアップロード/ダウンロードのプロトコル
- タグ管理とreferenceの概念

### 実装前調査課題
以下のAPIエンドポイントについて調べ、各々の役割を整理せよ：

```
# Repository API
GET /v2/_catalog
GET /v2/<name>/tags/list

# Manifest API  
GET /v2/<name>/manifests/<reference>
PUT /v2/<name>/manifests/<reference>
DELETE /v2/<name>/manifests/<reference>
HEAD /v2/<name>/manifests/<reference>

# Blob API
GET /v2/<name>/blobs/<digest>
DELETE /v2/<name>/blobs/<digest>
HEAD /v2/<name>/blobs/<digest>

# Upload API
POST /v2/<name>/blobs/uploads/
PUT /v2/<name>/blobs/uploads/<uuid>
PATCH /v2/<name>/blobs/uploads/<uuid>
GET /v2/<name>/blobs/uploads/<uuid>
DELETE /v2/<name>/blobs/uploads/<uuid>
```

### データ構造分析課題
以下のJSONデータ構造を分析し、各フィールドの意味を理解せよ：

```json
// Image Manifest
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "digest": "sha256:...",
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 1234
  },
  "layers": [
    {
      "digest": "sha256:...", 
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 5678
    }
  ]
}
```

### 考察課題
- なぜContent-Addressableなストレージが必要なのか？
- manifest、config、layersの役割分担は？
- タグとdigestによる参照の使い分けは？

## Step 2: Content-Addressable Storageの実装

### このステップの目的
SHA256ハッシュベースのファイルストレージシステムを実装し、データの整合性とデデュプリケーションを実現する

### 実装課題
以下の機能を含むCASを実装せよ：

```go
type ContentStore struct {
    rootPath    string
    blobPath    string  // blobs/sha256/
    uploadPath  string  // uploads/
    // 他に必要なフィールドは？
}

func (cs *ContentStore) Put(reader io.Reader) (string, int64, error) {
    // 1. データの読み取りとSHA256ハッシュ計算
    // 2. 一時ファイルへの書き込み
    // 3. ハッシュ検証
    // 4. 最終的な保存場所への移動
    
    hasher := sha256.New()
    tempFile, err := cs.createTempFile()
    
    // どのように実装するか？
    return digest, size, nil
}

func (cs *ContentStore) Get(digest string) (io.ReadCloser, int64, error) {
    // digestからファイルパス生成
    // ファイルの存在確認
    // ファイルサイズとReaderの返却
    return nil, 0, nil
}

func (cs *ContentStore) Exists(digest string) bool {
    // digestのファイル存在確認
    return false
}
```

### 設計課題
1. **ディレクトリ構造**
   ```
   /var/lib/registry/
   ├── blobs/
   │   └── sha256/
   │       ├── ab/
   │       │   └── abcd1234...ef/  # 最初の2文字でディレクトリ分散
   │       │       └── data
   │       └── cd/
   │           └── cdef5678...90/
   │               └── data
   ├── uploads/  # 一時アップロード用
   └── repositories/  # manifest管理用
   ```

2. **並行処理とロック**
   ```go
   type SafeContentStore struct {
       store *ContentStore
       locks sync.Map  // digest -> *sync.Mutex
   }
   
   func (scs *SafeContentStore) acquireLock(digest string) *sync.Mutex {
       // digestごとのファイルロック取得
   }
   ```

3. **エラーハンドリング**
   - ディスク容量不足の処理
   - ハッシュ不一致の処理
   - 並行アクセス時の競合処理

### 実装検証
```bash
# ファイルの保存テスト
echo "test content" | go run cas_test.go put
# Output: sha256:9a271f2a9162543b6e5f1dd7f93a5b3b5f6b7c8d...

# ファイルの取得テスト
go run cas_test.go get sha256:9a271f2a9162543b...
# Output: test content
```

## Step 3: HTTP APIサーバーの実装

### このステップの目的
OCI Distribution Specification準拠のHTTP APIサーバーを実装する

### 実装課題
1. **基本的なサーバー構造**
   ```go
   type RegistryServer struct {
       contentStore *ContentStore
       manifestStore *ManifestStore
       router       *http.ServeMux
       // 他に必要なフィールドは？
   }
   
   func (rs *RegistryServer) SetupRoutes() {
       // APIエンドポイントの登録
       rs.router.HandleFunc("/v2/_catalog", rs.handleCatalog)
       rs.router.HandleFunc("/v2/{name}/manifests/{reference}", rs.handleManifest)
       rs.router.HandleFunc("/v2/{name}/blobs/{digest}", rs.handleBlob)
       // 他のエンドポイントは？
   }
   ```

2. **Manifestハンドラーの実装**
   ```go
   func (rs *RegistryServer) handleManifest(w http.ResponseWriter, r *http.Request) {
       name := mux.Vars(r)["name"]
       reference := mux.Vars(r)["reference"]
       
       switch r.Method {
       case "GET":
           // Manifestの取得
       case "PUT":
           // Manifestの保存
       case "DELETE":
           // Manifestの削除
       case "HEAD":
           // Manifest存在確認
       }
   }
   ```

3. **Blobハンドラーの実装**
   ```go
   func (rs *RegistryServer) handleBlob(w http.ResponseWriter, r *http.Request) {
       digest := mux.Vars(r)["digest"]
       
       switch r.Method {
       case "GET":
           // Blobのダウンロード（Range Request対応）
       case "HEAD":
           // Blob存在確認
       case "DELETE":
           // Blob削除
       }
   }
   ```

### HTTP Range Request対応
```go
func (rs *RegistryServer) serveBlobWithRange(w http.ResponseWriter, r *http.Request, digest string) {
    // Rangeヘッダーの解析
    rangeHeader := r.Header.Get("Range")
    if rangeHeader != "" {
        // 範囲指定ダウンロードの実装
        // bytes=0-1023 のような形式をパース
    }
    
    // 適切なHTTPステータスコードの設定
    // 206 Partial Content vs 200 OK
}
```

### エラーレスポンス
```go
type ErrorResponse struct {
    Errors []ErrorDetail `json:"errors"`
}

type ErrorDetail struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Detail  interface{} `json:"detail,omitempty"`
}

// エラーレスポンスの統一処理
func (rs *RegistryServer) writeError(w http.ResponseWriter, code string, message string, status int) {
    // JSON形式でエラーを返却
}
```

## Step 4: アップロード機能の実装

### このステップの目的
分割アップロード（chunked upload）に対応したBlob保存機能を実装する

### 実装課題
1. **アップロードセッション管理**
   ```go
   type UploadSession struct {
       UUID        string
       Repository  string
       TempFile    string
       BytesWritten int64
       StartedAt   time.Time
       // 他に必要なフィールドは？
   }
   
   type UploadManager struct {
       sessions map[string]*UploadSession
       mutex    sync.RWMutex
       tempDir  string
   }
   
   func (um *UploadManager) StartUpload(repository string) (*UploadSession, error) {
       // 新しいアップロードセッションの開始
       uuid := generateUUID()
       session := &UploadSession{
           UUID:       uuid,
           Repository: repository,
           TempFile:   filepath.Join(um.tempDir, uuid),
           StartedAt:  time.Now(),
       }
       return session, nil
   }
   ```

2. **分割アップロード処理**
   ```go
   func (rs *RegistryServer) handleUploadPatch(w http.ResponseWriter, r *http.Request) {
       uuid := mux.Vars(r)["uuid"]
       session := rs.uploadManager.GetSession(uuid)
       
       // Content-Rangeヘッダーの確認
       contentRange := r.Header.Get("Content-Range")
       
       // ファイルへのデータ追記
       // 進捗状況の更新
       // レスポンスヘッダーの設定
   }
   ```

3. **アップロード完了処理**
   ```go
   func (rs *RegistryServer) handleUploadPut(w http.ResponseWriter, r *http.Request) {
       uuid := mux.Vars(r)["uuid"]
       expectedDigest := r.URL.Query().Get("digest")
       
       // アップロードファイルのハッシュ計算
       // 期待されるdigestとの比較
       // Content Storeへの移動
       // セッションのクリーンアップ
   }
   ```

### 設計課題
- アップロードタイムアウトの処理
- 失敗したアップロードのクリーンアップ
- 並行アップロードの制限

### 実装検証
```bash
# アップロード開始
curl -X POST http://localhost:5000/v2/myapp/blobs/uploads/
# Location: /v2/myapp/blobs/uploads/12345-uuid

# データのアップロード
curl -X PATCH -H "Content-Range: 0-1023" --data-binary @data.tar.gz \
    http://localhost:5000/v2/myapp/blobs/uploads/12345-uuid

# アップロード完了
curl -X PUT -H "Content-Length: 0" \
    "http://localhost:5000/v2/myapp/blobs/uploads/12345-uuid?digest=sha256:..."
```

## Step 5: ManifestとRepositoryの管理

### このステップの目的
イメージのメタデータ管理とRepository構造の実装を行う

### 実装課題
1. **Manifest Store実装**
   ```go
   type ManifestStore struct {
       repoPath string  // repositories/
       // どのような構造でmanifestを管理するか？
   }
   
   func (ms *ManifestStore) PutManifest(repo, reference string, manifest []byte) error {
       // manifestの保存
       // タグとdigestの両方で参照可能にする
       // どのようなファイル構造にするか？
   }
   
   func (ms *ManifestStore) GetManifest(repo, reference string) ([]byte, string, error) {
       // manifestの取得
       // referenceがタグかdigestかを判定
       // 適切なmanifestの返却
   }
   ```

2. **Repository管理**
   ```go
   func (ms *ManifestStore) ListRepositories() ([]string, error) {
       // 全リポジトリ一覧の取得
   }
   
   func (ms *ManifestStore) ListTags(repository string) ([]string, error) {
       // 指定リポジトリのタグ一覧
   }
   ```

3. **Manifestの検証**
   ```go
   func (ms *ManifestStore) ValidateManifest(manifestData []byte) error {
       // JSON形式の検証
       // 必須フィールドの確認
       // 参照されるBlobの存在確認
   }
   ```

### ストレージ構造設計
```
repositories/
├── myapp/
│   ├── _manifests/
│   │   ├── tags/
│   │   │   ├── latest/
│   │   │   │   └── current -> ../../revisions/sha256:abcd.../
│   │   │   └── v1.0/
│   │   │       └── current -> ../../revisions/sha256:efgh.../
│   │   └── revisions/
│   │       └── sha256/
│   │           ├── abcd.../
│   │           │   └── link
│   │           └── efgh.../
│   │               └── link
│   └── _uploads/  # アップロード用一時ディレクトリ
```

### 実装検証
```bash
# Manifest保存
curl -X PUT -H "Content-Type: application/vnd.oci.image.manifest.v1+json" \
    --data @manifest.json \
    http://localhost:5000/v2/myapp/manifests/latest

# Manifest取得
curl http://localhost:5000/v2/myapp/manifests/latest

# タグ一覧
curl http://localhost:5000/v2/myapp/tags/list
```

## Step 6: クライアント実装

### このステップの目的
レジストリとやり取りするクライアントツールを実装し、push/pull機能を完成させる

### 実装課題
1. **Push機能**
   ```go
   type RegistryClient struct {
       baseURL    string
       httpClient *http.Client
       // 認証情報など
   }
   
   func (rc *RegistryClient) PushLayer(repository string, layerData io.Reader) (string, error) {
       // 1. アップロード開始
       location, err := rc.startUpload(repository)
       
       // 2. データのアップロード
       digest, size, err := rc.uploadData(location, layerData)
       
       // 3. アップロード完了
       err = rc.completeUpload(location, digest)
       
       return digest, err
   }
   
   func (rc *RegistryClient) PushManifest(repository, tag string, manifest []byte) error {
       // Manifestのpush
   }
   ```

2. **Pull機能**
   ```go
   func (rc *RegistryClient) PullManifest(repository, reference string) (*Manifest, error) {
       // Manifestの取得とパース
   }
   
   func (rc *RegistryClient) PullLayer(repository, digest string) (io.ReadCloser, error) {
       // レイヤーデータの取得
   }
   ```

3. **CLI統合**
   ```bash
   # レジストリサーバー起動
   ./shibafu registry serve --port 5000
   
   # レイヤーをpush
   ./shibafu push localhost:5000/myapp:latest --layers alpine-base,python-runtime
   
   # イメージをpull
   ./shibafu pull localhost:5000/myapp:latest
   
   # pullしたイメージでコンテナ実行
   ./shibafu run localhost:5000/myapp:latest python app.py
   ```

## Step 7: 高度な機能とセキュリティ

### このステップの目的
実用的なレジストリに必要な高度な機能とセキュリティ対策を実装する

### 実装課題
1. **認証・認可**
   ```go
   type AuthManager struct {
       // Token based authentication
       // どのような認証方式を採用するか？
   }
   
   func (am *AuthManager) ValidateToken(token string) (*User, error) {
       // JWT token validation?
       // Basic authentication?
   }
   
   func (am *AuthManager) HasPermission(user *User, repo string, action string) bool {
       // push/pull権限の確認
   }
   ```

2. **ガベージコレクション**
   ```go
   func (rs *RegistryServer) RunGarbageCollection() error {
       // 参照されていないBlobの削除
       // 古いアップロードセッションのクリーンアップ
       // 削除マークされたManifestの物理削除
   }
   ```

3. **メトリクスとログ**
   ```go
   func (rs *RegistryServer) logRequest(r *http.Request, status int, size int64) {
       // アクセスログの記録
       // Prometheus metricsの更新
   }
   ```

4. **レプリケーション**
   ```go
   func (rs *RegistryServer) ReplicateToMirror(mirrorURL string) error {
       // 他のレジストリへの複製
   }
   ```

### セキュリティ強化
- HTTPS対応（TLS証明書）
- レート制限の実装
- ファイルスキャン（マルウェア検査）
- 署名検証機能

## 発展課題

### 課題1: 分散ストレージ対応
- S3/MinIO backend実装
- 複数ストレージの冗長化
- 地理的分散配置

### 課題2: 高性能化
- Blobのキャッシング戦略
- 並行ダウンロード対応
- CDN統合

### 課題3: 運用機能
- ヘルスチェックエンドポイント
- 管理用WebUI
- バックアップ・リストア機能

### 課題4: 互換性拡張
- Docker Registry API v2対応
- Harbor/GitLab Registryとの互換性
- Helm Chart対応

## 参考資料
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec)
- [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)
- [Content-Addressable Storage](https://en.wikipedia.org/wiki/Content-addressable_storage)
- [HTTP Range Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)

## 完了チェックリスト
- [ ] OCI Distribution Specificationを理解できた
- [ ] Content-Addressable Storageを実装できた
- [ ] HTTP APIサーバーを実装できた
- [ ] アップロード機能を実装できた
- [ ] ManifestとRepositoryの管理を実装できた
- [ ] クライアント実装でpush/pullができた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. OCI Distribution Specificationの詳細と実装方法
2. Content-Addressable Storageの設計と実装
3. 高性能なHTTP APIサーバーの構築技術
4. コンテナレジストリの運用に必要な機能

次のPhase 6では、コンテナ間で名前解決可能な内部ネットワークを構築し、マルチネットワーク環境とDNSサーバーを実装します。