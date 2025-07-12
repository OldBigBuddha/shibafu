# Phase 4: ポートマッピングとサービス公開 - 詳細学習計画

## 学習目標
コンテナ内のサービスをホスト経由で外部に公開し、iptables DNATを使用した動的ポート割り当て、HTTPリバースプロキシによるL7ルーティング、およびロードバランシング機能を実装する

## 事前準備

### 環境要件確認
- Phase 3の完了（OverlayFSレイヤー管理）
- iptables管理権限
- HTTP/TCPトラフィック テスト環境
- Go HTTP ライブラリの理解

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- DNAT（Destination NAT）とSNAT（Source NAT）の違い
- L4（Transport Layer）とL7（Application Layer）ロードバランシング
- リバースプロキシの動作原理
- HTTP Hostヘッダーとバーチャルホスト
- UNIX socketとTCP socketの使い分け

## Step 1: iptables DNATの理解と実装

### このステップの目的
iptablesのNATテーブルを使用してパケット転送ルールを作成し、ホストポートからコンテナポートへの転送を実現する

### 調査すべき内容
- iptablesのNATテーブル構造（PREROUTING, POSTROUTING, OUTPUT）
- DNATとSNATの使い分け
- conntrackとNATセッション管理
- iptablesルールの優先順位と評価順序

### 実験課題
以下の手順で基本的なポート転送を手動で構築せよ：

```bash
# 1. コンテナ内でWebサーバーを起動
./shibafu run --name webtest alpine
# コンテナ内で: python3 -m http.server 8080

# 2. コンテナのIPアドレスを確認
CONTAINER_IP=$(./shibafu inspect webtest | jq -r '.NetworkSettings.IPAddress')

# 3. DNATルールの追加
sudo iptables -t nat -A PREROUTING -p tcp --dport ? -j DNAT --to-destination ${CONTAINER_IP}:8080

# 4. SNATルールの追加（必要に応じて）
sudo iptables -t nat -A POSTROUTING -p tcp -d ${CONTAINER_IP} --dport 8080 -j ?

# 5. 動作確認
curl http://localhost:?
```

### 設計課題
- ホストポートの自動割り当て範囲は？（30000-32767？）
- 複数コンテナでの重複ポート回避方法は？
- ルールの削除タイミングは？

### 分析課題
以下のコマンドで設定されたルールを分析せよ：
```bash
# NATテーブルの確認
sudo iptables -t nat -L -n --line-numbers

# conntrackエントリの確認  
sudo conntrack -L | grep ${CONTAINER_IP}

# パケットフローの追跡
sudo tcpdump -i any host ${CONTAINER_IP}
```

## Step 2: 動的ポート管理システム

### このステップの目的
Goプログラムから動的なポート割り当てとiptablesルール管理を自動化する

### 実装課題
以下の機能を含むポートマネージャーを実装せよ：

```go
type PortManager struct {
    allocatedPorts map[int]string  // port -> containerID
    portRange      PortRange
    iptablesClient IptablesClient
    // 他に必要なフィールドは？
}

type PortMapping struct {
    ContainerID   string
    ContainerIP   net.IP
    ContainerPort int
    HostPort      int
    Protocol      string  // tcp, udp
}

func (pm *PortManager) AllocatePort(containerID string, containerIP net.IP, containerPort int) (*PortMapping, error) {
    // 利用可能なホストポートを検索
    hostPort := pm.findAvailablePort()
    
    // DNATルールの追加
    err := pm.addNATRule(hostPort, containerIP, containerPort)
    
    // 割り当て情報の記録
    mapping := &PortMapping{ /* ... */ }
    return mapping, err
}

func (pm *PortManager) ReleasePort(containerID string, hostPort int) error {
    // iptablesルールの削除
    // 割り当て情報の削除
    return nil
}
```

### 実装課題 - iptables操作
```go
// どのライブラリを使用するか？
// github.com/coreos/go-iptables?
// exec.Command("iptables")?

func (pm *PortManager) addNATRule(hostPort int, containerIP net.IP, containerPort int) error {
    // PREROUTING DNATルールの追加
    rule := []string{
        "-t", "nat",
        "-A", "PREROUTING", 
        "-p", "tcp", 
        "--dport", strconv.Itoa(hostPort),
        "-j", "DNAT",
        "--to-destination", fmt.Sprintf("%s:%d", containerIP, containerPort),
    }
    
    // どのように実行するか？
    return nil
}
```

### 設計課題
- ポート割り当て情報の永続化方法は？
- 競合状態（race condition）の回避方法は？
- 障害復旧時のiptablesルール復元方法は？

### エラーハンドリング課題
以下の状況への対処法を実装せよ：
- iptablesコマンド実行失敗
- ポート範囲の枯渇
- コンテナ削除時のルールクリーンアップ失敗

## Step 3: libcontainerとの統合

### このステップの目的
Phase 1-3で作成したコンテナランタイムにポートマッピング機能を統合する

### 実装課題
1. **コンテナ作成時のポート処理**
   ```go
   func (cm *ContainerManager) CreateContainerWithPorts(
       containerID string,
       config *configs.Config,
       portMappings []PortSpec) error {
       
       // コンテナの作成（Phase 1-3の機能）
       container, err := cm.createContainer(containerID, config)
       
       // コンテナ起動後にIPアドレス取得
       containerIP := cm.getContainerIP(containerID)
       
       // ポートマッピングの設定
       for _, portSpec := range portMappings {
           mapping, err := cm.portManager.AllocatePort(
               containerID, containerIP, portSpec.ContainerPort)
           // エラーハンドリング
       }
       
       return nil
   }
   ```

2. **CLIインターフェースの拡張**
   ```bash
   # 自動ポート割り当て
   ./shibafu run --publish 80 nginx
   # Output: Container started, accessible at localhost:30001
   
   # 手動ポート指定
   ./shibafu run --publish 8080:80 nginx
   
   # 複数ポートの公開
   ./shibafu run --publish 80 --publish 443 nginx
   
   # UDPポートの公開
   ./shibafu run --publish 53:53/udp dns-server
   ```

3. **コンテナ情報表示の拡張**
   ```bash
   ./shibafu ps
   # ID     NAME    PORTS                    STATUS
   # abc123 web1    0.0.0.0:8080->80/tcp    running
   # def456 web2    0.0.0.0:8081->80/tcp    running
   ```

### 設計課題
- コンテナ削除時のポートクリーンアップ確実性
- 複数プロトコル（TCP/UDP）への対応
- IPv6環境への対応

## Step 4: HTTPリバースプロキシの実装

### このステップの目的
L7レイヤーでのHTTPトラフィック処理とコンテナ名ベースのルーティングを実装する

### 実装課題
以下の機能を含むリバースプロキシサーバーを実装せよ：

```go
type ReverseProxy struct {
    routes     map[string]ProxyTarget  // hostname -> target
    server     *http.Server
    // 他に必要なフィールドは？
}

type ProxyTarget struct {
    ContainerID string
    BackendURL  *url.URL
    HealthCheck HealthCheckConfig
}

func (rp *ReverseProxy) AddRoute(hostname string, containerID string, backendURL string) error {
    // ルーティングルールの追加
    target := ProxyTarget{
        ContainerID: containerID,
        BackendURL:  mustParseURL(backendURL),
    }
    rp.routes[hostname] = target
    return nil
}

func (rp *ReverseProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Hostヘッダーによるルーティング
    host := r.Host
    target, exists := rp.routes[host]
    if !exists {
        http.NotFound(w, r)
        return
    }
    
    // バックエンドへのプロキシ処理
    // どのように実装するか？
}
```

### 実装課題 - プロキシ処理
```go
func (rp *ReverseProxy) proxyRequest(w http.ResponseWriter, r *http.Request, target *url.URL) {
    // httputil.ReverseProxyを使用？
    // カスタム実装？
    
    // リクエストの書き換え
    r.URL.Scheme = target.Scheme
    r.URL.Host = target.Host
    
    // レスポンスのストリーミング
    // エラーハンドリング
}
```

### 高度な機能実装
1. **ロードバランシング**
   ```go
   type LoadBalancer interface {
       SelectBackend(backends []ProxyTarget) *ProxyTarget
   }
   
   type RoundRobinLB struct {
       counter int64
   }
   
   func (lb *RoundRobinLB) SelectBackend(backends []ProxyTarget) *ProxyTarget {
       // ラウンドロビン選択の実装
   }
   ```

2. **ヘルスチェック**
   ```go
   func (rp *ReverseProxy) startHealthChecker() {
       ticker := time.NewTicker(30 * time.Second)
       go func() {
           for range ticker.C {
               for hostname, target := range rp.routes {
                   if !rp.isHealthy(target.BackendURL) {
                       // 不健全なバックエンドの処理
                   }
               }
           }
       }()
   }
   ```

3. **Sticky Session**
   ```go
   func (rp *ReverseProxy) getSessionBackend(r *http.Request) *ProxyTarget {
       // Cookieやセッション情報によるバックエンド選択
   }
   ```

## Step 5: サービスディスカバリの実装

### このステップの目的
コンテナの動的な追加・削除に対応するサービス発見機能を実装する

### 実装課題
1. **サービス登録機能**
   ```go
   type ServiceRegistry struct {
       services map[string][]ServiceEndpoint
       mutex    sync.RWMutex
   }
   
   type ServiceEndpoint struct {
       ContainerID string
       IP          net.IP
       Port        int
       Weight      int
       Health      HealthStatus
   }
   
   func (sr *ServiceRegistry) RegisterService(serviceName string, endpoint ServiceEndpoint) {
       // サービスエンドポイントの登録
   }
   
   func (sr *ServiceRegistry) DeregisterService(serviceName string, containerID string) {
       // サービスエンドポイントの削除
   }
   ```

2. **動的ルート更新**
   ```go
   func (rp *ReverseProxy) WatchServiceChanges(registry *ServiceRegistry) {
       // サービス変更の監視
       // ルーティングテーブルの動的更新
   }
   ```

3. **サービス名ベースルーティング**
   ```bash
   # サービス名でのアクセス
   curl http://webapp.local:8080
   curl http://api.local:8080
   curl http://database.local:8080
   ```

### 実装課題 - DNS統合
```go
// 簡易DNSサーバーの実装（オプション）
func (sr *ServiceRegistry) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
    // サービス名をIPアドレスに解決
    // A レコードの動的生成
}
```

## Step 6: 高度なネットワーク機能

### このステップの目的
本格的なコンテナプラットフォームに必要な高度なネットワーク機能を実装する

### 実装課題
1. **SSL終端**
   ```go
   func (rp *ReverseProxy) setupSSL(certFile, keyFile string) error {
       // TLS証明書の設定
       // HTTPS -> HTTPプロキシ
   }
   ```

2. **レート制限**
   ```go
   type RateLimiter struct {
       limits map[string]*rate.Limiter  // client IP -> limiter
   }
   
   func (rl *RateLimiter) Allow(clientIP string) bool {
       // IPアドレス別レート制限
   }
   ```

3. **アクセスログ**
   ```go
   func (rp *ReverseProxy) logRequest(r *http.Request, status int, duration time.Duration) {
       // 構造化ログの出力
       // メトリクス収集
   }
   ```

4. **WebSocket対応**
   ```go
   func (rp *ReverseProxy) handleWebSocket(w http.ResponseWriter, r *http.Request, target *url.URL) {
       // WebSocketプロキシの実装
   }
   ```

### 運用機能
```bash
# リアルタイム監視
./shibafu proxy stats
# Requests/sec: 1205
# Active connections: 45
# Backend health: api=healthy, web=degraded

# 設定の動的更新
./shibafu proxy add-route webapp.local container123
./shibafu proxy remove-route old-service.local
```

## Step 7: パフォーマンス最適化とモニタリング

### このステップの目的
高負荷環境での安定動作とパフォーマンス最適化を実現する

### 実装課題
1. **接続プーリング**
   ```go
   type ConnectionPool struct {
       pools map[string]*sync.Pool  // backend URL -> connection pool
   }
   
   func (cp *ConnectionPool) Get(backendURL string) *http.Client {
       // 接続の再利用
   }
   ```

2. **非同期処理**
   ```go
   func (rp *ReverseProxy) handleAsync(w http.ResponseWriter, r *http.Request) {
       // Goroutineベースの並行処理
       // チャンネルを使用したバックプレッシャー制御
   }
   ```

3. **メトリクス収集**
   ```go
   type Metrics struct {
       requestCount   *prometheus.CounterVec
       requestLatency *prometheus.HistogramVec
       activeConns    *prometheus.GaugeVec
   }
   
   func (m *Metrics) RecordRequest(method, path string, duration time.Duration) {
       // Prometheusメトリクスの記録
   }
   ```

### ベンチマークとテスト
```bash
# 負荷テスト
ab -n 10000 -c 100 http://localhost:8080/

# パフォーマンス分析
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30

# メモリ使用量監視
go tool pprof http://localhost:8080/debug/pprof/heap
```

## 発展課題

### 課題1: マイクロサービス対応
- API Gateway機能の実装
- OAuth/JWT認証の統合
- API versioning対応

### 課題2: グローバルロードバランシング
- 地理的分散バックエンド対応
- レイテンシベースルーティング
- 災害復旧対応

### 課題3: 高可用性
- プロキシサーバーの冗長化
- 設定の分散同期
- 自動障害検出と復旧

### 課題4: セキュリティ強化
- WAF（Web Application Firewall）機能
- DDoS攻撃対策
- 異常トラフィック検出

## 参考資料
- [iptables tutorial](https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO.html)
- [Go reverse proxy implementation](https://golang.org/pkg/net/http/httputil/#ReverseProxy)
- [HAProxy configuration guide](http://www.haproxy.org/download/1.8/doc/configuration.txt)
- [Nginx reverse proxy module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)

## 完了チェックリスト
- [ ] iptables DNATによるポート転送を実装できた
- [ ] 動的ポート管理システムを実装できた
- [ ] libcontainerとの統合ができた
- [ ] HTTPリバースプロキシを実装できた
- [ ] サービスディスカバリを実装できた
- [ ] 高度なネットワーク機能を実装できた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. L4/L7ロードバランシングの設計と実装
2. iptablesによる高度なネットワーク制御
3. 動的なサービス管理とルーティング
4. 高性能プロキシサーバーの実装技術

次のPhase 5では、OCI Distribution Specification準拠のコンテナレジストリを実装し、イメージの保存・配布・管理機能を構築します。