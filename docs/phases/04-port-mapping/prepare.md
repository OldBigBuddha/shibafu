# Phase 4 事前学習ガイド - ポートマッピングとリバースプロキシ

このドキュメントは、Phase 4でポートマッピングとリバースプロキシシステムを実装するために必要な基礎知識をまとめたものです。

## 1. ポートマッピングの基礎概念

### 1.1 ポートマッピングとは

**定義と目的**
- 外部からの通信を内部のコンテナに転送する技術
- ホストのポートとコンテナのポートを関連付ける
- コンテナネットワークの抽象化により外部アクセスを可能にする

**主要な用途**
- Webアプリケーションの公開
- データベースアクセス
- API サーバーの外部公開
- 開発環境での動的ポート割り当て

### 1.2 ポートマッピングの種類

#### 1. Static Port Mapping
```
Host:8080 → Container:80
Host:3306 → Container:3306
```
- 固定的なポートマッピング
- 設定が簡単、管理しやすい
- ポート競合のリスクあり

#### 2. Dynamic Port Mapping
```
Host:random → Container:80
Host:32768 → Container:80 (例)
```
- 動的なポート割り当て
- ポート競合の回避
- サービスディスカバリとの組み合わせが必要

## 2. iptables による NAT 実装

### 2.1 NAT の基本概念

**SNAT (Source NAT)**
- 送信元IPアドレスの変換
- 内部ネットワークから外部への通信で使用
- MASQUERADEルールで実現

**DNAT (Destination NAT)**
- 宛先IPアドレス・ポートの変換
- 外部から内部サービスへのアクセスで使用
- PREROUTINGチェーンで実行

### 2.2 iptables テーブル構造

```
        ┌─────────────┐
        │ PREROUTING  │ ← パケット受信時
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │   ROUTING   │ ← ルーティング決定
        └──┬──────┬───┘
           │      │
    ┌──────┴──┐ ┌─┴────────┐
    │ INPUT   │ │ FORWARD  │
    └─────────┘ └─────┬────┘
                     │
              ┌──────┴──────┐
              │POSTROUTING  │ ← パケット送信時
              └─────────────┘
```

### 2.3 実装例

**基本的なDNATルール**
```bash
# ホストの8080ポートをコンテナの80ポートに転送
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.20.0.2:80

# 転送されたパケットを許可
iptables -A FORWARD -p tcp -d 172.20.0.2 --dport 80 -j ACCEPT
```

**動的ポートマッピング**
```go
import (
    "fmt"
    "net"
    "os/exec"
)

type PortMapping struct {
    HostPort      int
    ContainerPort int
    ContainerIP   net.IP
    Protocol      string
}

func (pm *PortMapping) Apply() error {
    // DNATルールの追加
    dnatCmd := fmt.Sprintf(
        "iptables -t nat -A PREROUTING -p %s --dport %d -j DNAT --to-destination %s:%d",
        pm.Protocol, pm.HostPort, pm.ContainerIP, pm.ContainerPort,
    )
    
    if err := exec.Command("sh", "-c", dnatCmd).Run(); err != nil {
        return fmt.Errorf("failed to add DNAT rule: %w", err)
    }
    
    // FORWARDルールの追加
    forwardCmd := fmt.Sprintf(
        "iptables -A FORWARD -p %s -d %s --dport %d -j ACCEPT",
        pm.Protocol, pm.ContainerIP, pm.ContainerPort,
    )
    
    return exec.Command("sh", "-c", forwardCmd).Run()
}
```

## 3. リバースプロキシの概念

### 3.1 リバースプロキシとは

**定義**
- クライアントからのリクエストを受けて、背後のサーバーに転送
- サーバーの代理として動作
- 負荷分散、SSL終端、キャッシュなどの機能を提供

**フォワードプロキシとの違い**
```
フォワードプロキシ:
Client → Proxy → Server
(クライアント側の代理)

リバースプロキシ:
Client → Proxy → Server
(サーバー側の代理)
```

### 3.2 主要な機能

#### 1. 負荷分散
- Round Robin
- Least Connections
- IP Hash
- Weighted Round Robin

#### 2. SSL終端
- SSL/TLS の暗号化・復号化
- 証明書の一元管理
- バックエンドサーバーの負荷軽減

#### 3. キャッシュ機能
- 静的コンテンツのキャッシュ
- 動的コンテンツの部分キャッシュ
- CDN との統合

## 4. HTTP リバースプロキシの実装

### 4.1 Go での基本実装

```go
package main

import (
    "fmt"
    "net/http"
    "net/http/httputil"
    "net/url"
    "sync"
)

type Backend struct {
    URL    *url.URL
    Alive  bool
    mu     sync.RWMutex
}

func (b *Backend) SetAlive(alive bool) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.Alive = alive
}

func (b *Backend) IsAlive() bool {
    b.mu.RLock()
    defer b.mu.RUnlock()
    return b.Alive
}

type LoadBalancer struct {
    backends []*Backend
    current  int
    mu       sync.RWMutex
}

func (lb *LoadBalancer) NextBackend() *Backend {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    // Round Robin
    for i := 0; i < len(lb.backends); i++ {
        backend := lb.backends[lb.current]
        lb.current = (lb.current + 1) % len(lb.backends)
        
        if backend.IsAlive() {
            return backend
        }
    }
    
    return nil
}

func (lb *LoadBalancer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    backend := lb.NextBackend()
    if backend == nil {
        http.Error(w, "No available backends", http.StatusServiceUnavailable)
        return
    }
    
    proxy := httputil.NewSingleHostReverseProxy(backend.URL)
    proxy.ServeHTTP(w, r)
}
```

### 4.2 ヘルスチェック機能

```go
import (
    "net/http"
    "time"
)

type HealthChecker struct {
    backends []*Backend
    interval time.Duration
}

func (hc *HealthChecker) Start() {
    ticker := time.NewTicker(hc.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            for _, backend := range hc.backends {
                go hc.checkHealth(backend)
            }
        }
    }
}

func (hc *HealthChecker) checkHealth(backend *Backend) {
    healthURL := fmt.Sprintf("%s/health", backend.URL.String())
    
    client := &http.Client{
        Timeout: 5 * time.Second,
    }
    
    resp, err := client.Get(healthURL)
    if err != nil {
        backend.SetAlive(false)
        return
    }
    defer resp.Body.Close()
    
    if resp.StatusCode == http.StatusOK {
        backend.SetAlive(true)
    } else {
        backend.SetAlive(false)
    }
}
```

## 5. 高度な機能の実装

### 5.1 SSL/TLS 終端

```go
import (
    "crypto/tls"
    "net/http"
)

func createTLSConfig() *tls.Config {
    return &tls.Config{
        MinVersion: tls.VersionTLS12,
        CipherSuites: []uint16{
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        },
        PreferServerCipherSuites: true,
        CurvePreferences: []tls.CurveID{
            tls.CurveP256,
            tls.X25519,
        },
    }
}

func startHTTPSServer(handler http.Handler) error {
    server := &http.Server{
        Addr:      ":443",
        Handler:   handler,
        TLSConfig: createTLSConfig(),
    }
    
    return server.ListenAndServeTLS("cert.pem", "key.pem")
}
```

### 5.2 レート制限

```go
import (
    "golang.org/x/time/rate"
    "sync"
)

type RateLimitedProxy struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
}

func (rlp *RateLimitedProxy) getLimiter(clientIP string) *rate.Limiter {
    rlp.mu.RLock()
    limiter, exists := rlp.limiters[clientIP]
    rlp.mu.RUnlock()
    
    if !exists {
        rlp.mu.Lock()
        limiter = rate.NewLimiter(10, 20) // 10 req/sec, burst 20
        rlp.limiters[clientIP] = limiter
        rlp.mu.Unlock()
    }
    
    return limiter
}

func (rlp *RateLimitedProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    clientIP := r.RemoteAddr
    limiter := rlp.getLimiter(clientIP)
    
    if !limiter.Allow() {
        http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
        return
    }
    
    // 通常の処理
    rlp.handler.ServeHTTP(w, r)
}
```

## 6. 監視とログ

### 6.1 メトリクス収集

```go
import (
    "sync/atomic"
    "time"
)

type ProxyMetrics struct {
    RequestCount   uint64
    ResponseTime   time.Duration
    ErrorCount     uint64
    LastUpdate     time.Time
}

func (pm *ProxyMetrics) RecordRequest(duration time.Duration, err error) {
    atomic.AddUint64(&pm.RequestCount, 1)
    
    if err != nil {
        atomic.AddUint64(&pm.ErrorCount, 1)
    }
    
    // 移動平均での応答時間計算
    pm.ResponseTime = (pm.ResponseTime + duration) / 2
    pm.LastUpdate = time.Now()
}

func (pm *ProxyMetrics) GetStats() map[string]interface{} {
    return map[string]interface{}{
        "request_count":  atomic.LoadUint64(&pm.RequestCount),
        "error_count":    atomic.LoadUint64(&pm.ErrorCount),
        "response_time":  pm.ResponseTime,
        "last_update":    pm.LastUpdate,
    }
}
```

### 6.2 アクセスログ

```go
import (
    "log"
    "net/http"
    "time"
)

type LoggingResponseWriter struct {
    http.ResponseWriter
    statusCode int
    size       int
}

func (lrw *LoggingResponseWriter) WriteHeader(code int) {
    lrw.statusCode = code
    lrw.ResponseWriter.WriteHeader(code)
}

func (lrw *LoggingResponseWriter) Write(b []byte) (int, error) {
    size, err := lrw.ResponseWriter.Write(b)
    lrw.size += size
    return size, err
}

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        lrw := &LoggingResponseWriter{
            ResponseWriter: w,
            statusCode:     200,
        }
        
        next.ServeHTTP(lrw, r)
        
        duration := time.Since(start)
        
        log.Printf("%s %s %d %d %v",
            r.Method,
            r.URL.Path,
            lrw.statusCode,
            lrw.size,
            duration,
        )
    })
}
```

## 7. 設定管理とサービスディスカバリ

### 7.1 動的設定更新

```go
import (
    "encoding/json"
    "os"
    "sync"
)

type ProxyConfig struct {
    Backends []BackendConfig `json:"backends"`
    mu       sync.RWMutex
}

type BackendConfig struct {
    Name string `json:"name"`
    URL  string `json:"url"`
    Weight int `json:"weight"`
}

func (pc *ProxyConfig) LoadFromFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()
    
    pc.mu.Lock()
    defer pc.mu.Unlock()
    
    return json.NewDecoder(file).Decode(pc)
}

func (pc *ProxyConfig) GetBackends() []BackendConfig {
    pc.mu.RLock()
    defer pc.mu.RUnlock()
    
    backends := make([]BackendConfig, len(pc.Backends))
    copy(backends, pc.Backends)
    return backends
}
```

### 7.2 サービスディスカバリ統合

```go
import (
    "context"
    "time"
)

type ServiceDiscovery interface {
    Discover(ctx context.Context) ([]string, error)
}

type DNSServiceDiscovery struct {
    serviceName string
    resolver    *net.Resolver
}

func (dsd *DNSServiceDiscovery) Discover(ctx context.Context) ([]string, error) {
    ips, err := dsd.resolver.LookupIPAddr(ctx, dsd.serviceName)
    if err != nil {
        return nil, err
    }
    
    var addresses []string
    for _, ip := range ips {
        addresses = append(addresses, ip.IP.String())
    }
    
    return addresses, nil
}

func (lb *LoadBalancer) UpdateBackendsFromDiscovery(sd ServiceDiscovery) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            addresses, err := sd.Discover(ctx)
            cancel()
            
            if err != nil {
                log.Printf("Service discovery failed: %v", err)
                continue
            }
            
            lb.updateBackends(addresses)
        }
    }
}
```

## 8. セキュリティ考慮事項

### 8.1 認証・認可

```go
import (
    "crypto/subtle"
    "net/http"
    "strings"
)

func BasicAuthMiddleware(username, password string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user, pass, ok := r.BasicAuth()
            if !ok {
                w.Header().Set("WWW-Authenticate", `Basic realm="Restricted"`)
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            
            if subtle.ConstantTimeCompare([]byte(user), []byte(username)) != 1 ||
               subtle.ConstantTimeCompare([]byte(pass), []byte(password)) != 1 {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### 8.2 セキュリティヘッダー

```go
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // HSTS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        
        // XSS Protection
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        
        // Content Type Options
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // Frame Options
        w.Header().Set("X-Frame-Options", "DENY")
        
        // Content Security Policy
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        
        next.ServeHTTP(w, r)
    })
}
```

## 9. トラブルシューティング

### 9.1 一般的な問題と解決策

| 問題 | 原因 | 解決策 |
|------|------|--------|
| 502 Bad Gateway | バックエンドサーバー停止 | ヘルスチェック確認 |
| 504 Gateway Timeout | タイムアウト設定 | タイムアウト値調整 |
| 接続拒否 | ポート競合 | ポート使用状況確認 |
| SSL/TLS エラー | 証明書問題 | 証明書の有効性確認 |

### 9.2 デバッグ手法

```go
import (
    "net/http/httputil"
    "os"
)

func DebugMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if os.Getenv("DEBUG") == "true" {
            dump, err := httputil.DumpRequest(r, true)
            if err != nil {
                log.Printf("Failed to dump request: %v", err)
            } else {
                log.Printf("Request: %s", string(dump))
            }
        }
        
        next.ServeHTTP(w, r)
    })
}
```

## 10. 参考資料

### 10.1 必読ドキュメント

- [iptables man page](https://linux.die.net/man/8/iptables)
- [Netfilter Documentation](https://www.netfilter.org/documentation/)
- [Go HTTP ReverseProxy](https://pkg.go.dev/net/http/httputil#ReverseProxy)

### 10.2 参考実装

- [Nginx](https://nginx.org/en/docs/)
- [HAProxy](https://docs.haproxy.org/)
- [Traefik](https://doc.traefik.io/traefik/)

### 10.3 理解度確認チェックリスト

- [ ] DNAT/SNATの動作を説明できる
- [ ] iptablesでのルール設定ができる
- [ ] 負荷分散アルゴリズムを理解している
- [ ] HTTPリバースプロキシを実装できる
- [ ] ヘルスチェック機能を実装できる
- [ ] セキュリティ機能の実装ができる

これらの知識を身につけることで、Phase 4のポートマッピングとリバースプロキシ実装により深い理解を持って取り組むことができます。

---

## 📖 補助教材

Phase 4をより深く理解するために、以下の補助教材を参照することをお勧めします：

- **[Linux Bridge と veth - 理論的理解](../../resources/03-networking-concepts/bridge-veth-theory.md)**
  - iptablesによるNATの理論的基盤
  - ネットワーク名前空間でのパケット転送
  - 仮想ネットワークデバイスの動作原理

この資料は、Phase 4で実装するポートマッピングとリバースプロキシの基盤となるネットワーク技術を深く理解するのに役立ちます。特に、DNATとSNATの仕組みや、iptablesによるパケット転送制御の理論的背景を学ぶことができます。