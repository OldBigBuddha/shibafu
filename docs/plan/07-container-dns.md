# Phase 6: コンテナ間ネットワークとDNS - 詳細学習計画

## 学習目標
コンテナ間で名前解決可能な内部ネットワークを構築し、複数の独立したブリッジネットワーク、内部DNSサーバー、マルチネットワーク環境におけるサービス発見機能を実装する

## 事前準備

### 環境要件確認
- Phase 5の完了（OCI Registry実装）
- DNS プロトコルの基礎知識
- ネットワークルーティングの理解
- Go言語でのUDPサーバー実装経験

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- DNSプロトコルの基本構造（クエリ、レスポンス、レコードタイプ）
- 権威DNSサーバーと再帰DNSサーバーの違い
- Linux ネットワーク名前空間のDNS設定（/etc/resolv.conf）
- ポリシールーティングとマルチテーブルルーティング
- サービスディスカバリパターン（DNS-SD、Consul、etcd）

## Step 1: DNSプロトコルの実装

### このステップの目的
DNSプロトコルを理解し、基本的なDNSサーバー機能を手動で実装する

### 調査すべき内容
- DNS メッセージフォーマット（ヘッダー、クエリ、回答セクション）
- A、AAAA、CNAME、PTR、SRVレコードの構造
- DNS over UDP とDNS over TCPの使い分け
- DNSキャッシュの仕組み

### 実装課題
以下の最小限のDNSサーバーを実装せよ：

```go
type DNSServer struct {
    records map[string][]DNSRecord
    port    int
    // どのような構造が適切か？
}

type DNSRecord struct {
    Name   string
    Type   uint16  // A=1, AAAA=28, CNAME=5, etc
    TTL    uint32
    Data   []byte
}

func (ds *DNSServer) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
    // DNS クエリの解析
    question := r.Question[0]
    
    // レコードの検索
    answers := ds.lookupRecord(question.Name, question.Qtype)
    
    // レスポンスの構築
    response := &dns.Msg{}
    response.SetReply(r)
    response.Answer = answers
    
    // レスポンスの送信
    w.WriteMsg(response)
}

func (ds *DNSServer) lookupRecord(name string, qtype uint16) []dns.RR {
    // どのような検索アルゴリズムが効率的か？
    return nil
}
```

### 実装課題 - DNSクライアント
```go
func (ds *DNSServer) queryUpstream(name string, qtype uint16) ([]dns.RR, error) {
    // 上位DNSサーバーへのクエリ転送
    // 再帰的な名前解決の実装
    return nil, nil
}
```

### 検証課題
以下のコマンドで実装したDNSサーバーをテストせよ：

```bash
# DNSサーバー起動
go run dns_server.go --port 5353

# 名前解決テスト
dig @localhost -p 5353 test.local A
nslookup test.local localhost:5353

# 期待される結果
# test.local が設定したIPアドレスに解決される
```

### 考察課題
- DNS over UDPの512バイト制限をどう解決するか？
- DNSキャッシュのTTL管理方法は？
- 権威サーバーと再帰サーバーの役割分担は？

## Step 2: 複数ネットワークの実装

### このステップの目的
独立した複数のブリッジネットワークを作成し、ネットワーク間の分離とルーティングを実現する

### 実装課題
1. **ネットワーク管理システム**
   ```go
   type NetworkManager struct {
       networks map[string]*Network
       ipam     *IPAddressManager
       // 他に必要なフィールドは？
   }
   
   type Network struct {
       Name        string
       BridgeName  string
       Subnet      *net.IPNet
       Gateway     net.IP
       DNSServer   net.IP
       Containers  map[string]*ContainerNetworkInfo
   }
   
   func (nm *NetworkManager) CreateNetwork(name, subnet string) (*Network, error) {
       // ブリッジの作成
       // IPアドレス範囲の設定
       // ルーティングテーブルの設定
       return nil, nil
   }
   ```

2. **ネットワーク分離**
   ```go
   func (nm *NetworkManager) IsolateNetworks(net1, net2 string) error {
       // iptables ルールによるトラフィック分離
       // どのようなルールが必要か？
       return nil
   }
   
   func (nm *NetworkManager) ConnectNetworks(net1, net2 string) error {
       // ネットワーク間ルーティングの設定
       return nil
   }
   ```

3. **マルチネットワーク対応コンテナ**
   ```go
   func (nm *NetworkManager) ConnectContainer(containerID string, networks []string) error {
       for _, netName := range networks {
           // 各ネットワークにveth pair作成
           // コンテナ内にインターフェース追加
           // IPアドレス割り当て
       }
       return nil
   }
   ```

### 実装検証
```bash
# 複数ネットワークの作成
./shibafu network create frontend 172.21.0.0/24
./shibafu network create backend 172.22.0.0/24
./shibafu network create database 172.23.0.0/24

# ネットワーク情報の確認
./shibafu network ls
# NAME       SUBNET          GATEWAY      DNS
# frontend   172.21.0.0/24   172.21.0.1   172.21.0.1
# backend    172.22.0.0/24   172.22.0.1   172.22.0.1
# database   172.23.0.0/24   172.23.0.1   172.23.0.1

# ネットワーク分離の確認
./shibafu run --network frontend alpine ping 172.22.0.1
# エラー: ネットワークが分離されているため到達不可
```

### 設計課題
- IPアドレス重複の回避方法は？
- ルーティングテーブルの競合回避は？
- ネットワーク削除時のクリーンアップは？

## Step 3: 内部DNSサーバーの統合

### このステップの目的
各ネットワークに専用DNSサーバーを配置し、コンテナ名による自動名前解決を実現する

### 実装課題
1. **ネットワーク別DNS設定**
   ```go
   type NetworkDNS struct {
       network     *Network
       dnsServer   *DNSServer
       localZone   string  // "frontend.local", "backend.local"
       // 他に必要なフィールドは？
   }
   
   func (nd *NetworkDNS) RegisterContainer(containerID, containerName string, ip net.IP) error {
       // コンテナのDNSレコード追加
       // どのようなレコード形式にするか？
       // example: web.frontend.local -> 172.21.0.5
       return nil
   }
   
   func (nd *NetworkDNS) UnregisterContainer(containerID string) error {
       // コンテナのDNSレコード削除
       return nil
   }
   ```

2. **動的DNS更新**
   ```go
   func (nm *NetworkManager) onContainerStart(containerID string, networks []string) {
       for _, netName := range networks {
           network := nm.networks[netName]
           ip := nm.getContainerIP(containerID, netName)
           containerName := nm.getContainerName(containerID)
           
           // DNS レコードの自動登録
           network.DNS.RegisterContainer(containerID, containerName, ip)
       }
   }
   ```

3. **resolv.conf 管理**
   ```go
   func (nm *NetworkManager) setupContainerDNS(containerID string, networks []string) error {
       // コンテナの /etc/resolv.conf 設定
       dnsServers := []string{}
       searchDomains := []string{}
       
       for _, netName := range networks {
           dnsServers = append(dnsServers, nm.networks[netName].DNSServer.String())
           searchDomains = append(searchDomains, netName+".local")
       }
       
       // どのようにコンテナ内ファイルを設定するか？
       return nil
   }
   ```

### 実装検証
```bash
# コンテナ起動とDNS確認
./shibafu run --name postgres --network backend postgres:14
./shibafu run --name api --network backend --network frontend node-api

# 名前解決テスト
./shibafu exec api nslookup postgres.backend.local
# Answer: postgres.backend.local -> 172.22.0.5

./shibafu exec api ping postgres.backend.local
# PING postgres.backend.local (172.22.0.5): 56 data bytes
```

## Step 4: サービス発見とロードバランシング

### このステップの目的
同一サービスの複数インスタンス間でのロードバランシングとヘルスチェック機能を実装する

### 実装課題
1. **サービス登録・発見システム**
   ```go
   type ServiceRegistry struct {
       services map[string]*Service
       mutex    sync.RWMutex
   }
   
   type Service struct {
       Name        string
       Instances   []*ServiceInstance
       LBPolicy    LoadBalancingPolicy
       HealthCheck HealthCheckConfig
   }
   
   type ServiceInstance struct {
       ContainerID string
       IP          net.IP
       Port        int
       Weight      int
       Healthy     bool
       LastCheck   time.Time
   }
   
   func (sr *ServiceRegistry) RegisterService(containerID, serviceName string, ip net.IP, port int) error {
       // サービスインスタンスの登録
       return nil
   }
   ```

2. **DNS-based ロードバランシング**
   ```go
   func (nd *NetworkDNS) handleServiceQuery(serviceName string) []dns.RR {
       service := nd.registry.GetService(serviceName)
       healthyInstances := service.GetHealthyInstances()
       
       switch service.LBPolicy {
       case RoundRobin:
           // ラウンドロビンでIPアドレス返却
       case WeightedRoundRobin:
           // 重み付きラウンドロビン
       case LeastConnections:
           // 最小接続数ベース
       }
       
       return nil
   }
   ```

3. **ヘルスチェック機能**
   ```go
   func (sr *ServiceRegistry) startHealthChecker() {
       ticker := time.NewTicker(30 * time.Second)
       go func() {
           for range ticker.C {
               for _, service := range sr.services {
                   for _, instance := range service.Instances {
                       healthy := sr.checkHealth(instance)
                       instance.Healthy = healthy
                       instance.LastCheck = time.Now()
                   }
               }
           }
       }()
   }
   
   func (sr *ServiceRegistry) checkHealth(instance *ServiceInstance) bool {
       // TCPコネクションチェック
       // HTTPヘルスチェックエンドポイント
       // カスタムヘルスチェックコマンド
       return false
   }
   ```

### 実装検証
```bash
# 同一サービスの複数インスタンス起動
./shibafu run --name web1 --network frontend --service web nginx
./shibafu run --name web2 --network frontend --service web nginx
./shibafu run --name web3 --network frontend --service web nginx

# サービス発見テスト
./shibafu exec client nslookup web.frontend.local
# 複数のIPアドレスが返される

# ロードバランシング動作確認
for i in {1..10}; do
    ./shibafu exec client curl http://web.frontend.local/
done
# 各webサーバーに分散してアクセスされることを確認
```

## Step 5: 高度なネットワーク機能

### このステップの目的
実用的なコンテナネットワークに必要な高度な機能を実装する

### 実装課題
1. **ネットワークポリシー**
   ```go
   type NetworkPolicy struct {
       Name      string
       Networks  []string
       Rules     []PolicyRule
   }
   
   type PolicyRule struct {
       Action    PolicyAction  // Allow, Deny
       Direction Direction     // Ingress, Egress
       Protocol  string        // tcp, udp, icmp
       Ports     []int
       Sources   []string      // CIDR, service names
   }
   
   func (nm *NetworkManager) ApplyPolicy(policy *NetworkPolicy) error {
       // iptables ルールの生成と適用
       return nil
   }
   ```

2. **SRVレコード対応**
   ```go
   func (nd *NetworkDNS) RegisterServiceWithSRV(serviceName string, instances []SRVInstance) error {
       // SRVレコードによるサービス登録
       // _http._tcp.api.backend.local SRV 10 5 8080 api1.backend.local
       return nil
   }
   ```

3. **外部DNS統合**
   ```go
   func (nd *NetworkDNS) setupForwarders(forwarders []string) {
       // 外部DNSサーバーへのクエリ転送設定
       // 条件付きフォワーディング（*.example.com -> 8.8.8.8）
   }
   ```

4. **IPv6対応**
   ```go
   func (nm *NetworkManager) CreateDualStackNetwork(name, ipv4Subnet, ipv6Subnet string) error {
       // IPv4/IPv6デュアルスタックネットワーク
       return nil
   }
   ```

### セキュリティ機能
```bash
# ネットワークポリシーの適用
./shibafu network policy create \
    --name "database-isolation" \
    --network database \
    --deny-ingress-from frontend \
    --allow-ingress-from backend:5432

# ポリシー確認
./shibafu network policy list
```

## Step 6: パフォーマンス最適化

### このステップの目的
大規模環境での高性能な名前解決とネットワーク処理を実現する

### 実装課題
1. **DNSキャッシュ最適化**
   ```go
   type DNSCache struct {
       cache   sync.Map  // concurrent map
       ttl     time.Duration
       metrics *CacheMetrics
   }
   
   func (dc *DNSCache) Get(key string) ([]dns.RR, bool) {
       // TTL考慮したキャッシュ取得
       return nil, false
   }
   
   func (dc *DNSCache) Set(key string, records []dns.RR, ttl time.Duration) {
       // レコードのキャッシュ保存
   }
   ```

2. **非同期DNS処理**
   ```go
   func (ds *DNSServer) handleQueryAsync(w dns.ResponseWriter, r *dns.Msg) {
       go func() {
           response := ds.processQuery(r)
           w.WriteMsg(response)
       }()
   }
   ```

3. **ネットワーク接続プール**
   ```go
   type ConnectionPool struct {
       pools map[string]chan net.Conn
       maxConns int
   }
   
   func (cp *ConnectionPool) GetConnection(network string) (net.Conn, error) {
       // 接続の再利用
       return nil, nil
   }
   ```

### 監視とメトリクス
```go
type NetworkMetrics struct {
    DNSQueries     *prometheus.CounterVec
    NetworkLatency *prometheus.HistogramVec
    ActiveConns    *prometheus.GaugeVec
}

func (nm *NetworkManager) collectMetrics() {
    // Prometheus メトリクス収集
}
```

## Step 7: 運用機能とデバッグ

### このステップの目的
ネットワーク問題の診断と運用に必要な機能を実装する

### 実装課題
1. **ネットワーク診断ツール**
   ```bash
   # ネットワーク接続性テスト
   ./shibafu network diagnose --from container1 --to container2
   
   # DNSクエリのトレース
   ./shibafu network dns-trace --query api.backend.local
   
   # ルーティングテーブル表示
   ./shibafu network routes --container web1
   ```

2. **ネットワーク可視化**
   ```go
   func (nm *NetworkManager) GenerateNetworkTopology() (*NetworkGraph, error) {
       // ネットワーク構成の可視化データ生成
       return nil, nil
   }
   ```

3. **ログとトレーシング**
   ```go
   func (ds *DNSServer) logQuery(query *dns.Msg, response *dns.Msg, duration time.Duration) {
       // 構造化ログ出力
       // 分散トレーシング対応
   }
   ```

### 設定管理
```yaml
# network-config.yaml
networks:
  frontend:
    subnet: "172.21.0.0/24"
    dns:
      zone: "frontend.local"
      forwarders: ["8.8.8.8", "8.8.4.4"]
  backend:
    subnet: "172.22.0.0/24"
    isolation: strict
    
policies:
  - name: "web-db-isolation"
    rules:
      - deny: {from: "frontend", to: "database"}
      - allow: {from: "backend", to: "database", ports: [5432]}
```

## 発展課題

### 課題1: マルチホストネットワーク
- VXLAN overlay network実装
- etcdクラスター連携
- 分散DNS設定

### 課題2: サービスメッシュ統合
- Envoy proxy連携
- Istio互換機能
- mTLS対応

### 課題3: CNI プラグイン
- Kubernetes CNI仕様対応
- 他のCNIプラグインとの互換性
- ネットワークプロバイダー抽象化

### 課題4: 高可用性
- DNS サーバーの冗長化
- ネットワーク障害時の自動復旧
- 分散レジストリ同期

## 参考資料
- [DNS Protocol RFC 1035](https://tools.ietf.org/html/rfc1035)
- [SRV Record RFC 2782](https://tools.ietf.org/html/rfc2782)
- [Linux Advanced Routing](https://lartc.org/howto/)
- [Container Network Interface (CNI)](https://github.com/containernetworking/cni)

## 完了チェックリスト
- [ ] DNSプロトコルを実装できた
- [ ] 複数ネットワークの分離を実装できた
- [ ] 内部DNSサーバーの統合ができた
- [ ] サービス発見とロードバランシングを実装できた
- [ ] 高度なネットワーク機能を実装できた
- [ ] パフォーマンス最適化ができた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. DNSプロトコルの詳細と実装技術
2. 大規模ネットワーク環境の設計と管理手法
3. サービス発見とロードバランシングの実装
4. ネットワークセキュリティとポリシー管理

次のPhase 7では、これまでのすべての機能を統合し、完全なコンテナランタイムシステムとして動作する統合プラットフォームを完成させます。