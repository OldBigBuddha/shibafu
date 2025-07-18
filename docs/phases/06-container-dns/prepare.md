# Phase 6 事前学習ガイド - コンテナ DNS とサービスディスカバリ

このドキュメントは、Phase 6でコンテナ専用のDNSサーバーとサービスディスカバリシステムを実装するために必要な基礎知識をまとめたものです。

## 1. DNS の基礎概念

### 1.1 DNSとは

**定義と役割**
- Domain Name System（ドメイン名システム）
- 人間が理解しやすい名前を IPアドレスに変換
- 分散型のデータベースシステム
- インターネットの基盤技術

**DNSレコードの種類**
| レコード型 | 用途 | 例 |
|-----------|------|-----|
| A | IPv4アドレス | example.com → 192.168.1.1 |
| AAAA | IPv6アドレス | example.com → 2001:db8::1 |
| CNAME | 正規名への別名 | www.example.com → example.com |
| MX | メールサーバー | example.com → mail.example.com |
| TXT | テキスト情報 | example.com → "v=spf1 include:_spf.google.com ~all" |
| SRV | サービス情報 | _http._tcp.example.com → server:80 |

### 1.2 DNSクエリの流れ

```
1. クライアント → リゾルバー: "example.com のIPアドレスは？"
2. リゾルバー → ルートサーバー: ".com の権威サーバーは？"
3. ルートサーバー → リゾルバー: "com の権威サーバーは X.X.X.X"
4. リゾルバー → com権威サーバー: "example.com の権威サーバーは？"
5. com権威サーバー → リゾルバー: "example.com の権威サーバーは Y.Y.Y.Y"
6. リゾルバー → example.com権威サーバー: "example.com のIPアドレスは？"
7. example.com権威サーバー → リゾルバー: "192.168.1.1"
8. リゾルバー → クライアント: "192.168.1.1"
```

## 2. Go言語でのDNSサーバー実装

### 2.1 基本的なDNSサーバー

```go
package main

import (
    "fmt"
    "net"
    "strings"
    "sync"
    
    "github.com/miekg/dns"
)

type DNSServer struct {
    records map[string][]dns.RR
    mu      sync.RWMutex
    server  *dns.Server
}

func NewDNSServer(addr string) *DNSServer {
    ds := &DNSServer{
        records: make(map[string][]dns.RR),
    }
    
    mux := dns.NewServeMux()
    mux.HandleFunc(".", ds.handleDNSRequest)
    
    ds.server = &dns.Server{
        Addr:    addr,
        Net:     "udp",
        Handler: mux,
    }
    
    return ds
}

func (ds *DNSServer) handleDNSRequest(w dns.ResponseWriter, r *dns.Msg) {
    msg := new(dns.Msg)
    msg.SetReply(r)
    
    for _, question := range r.Question {
        answers := ds.resolve(question.Name, question.Qtype)
        msg.Answer = append(msg.Answer, answers...)
    }
    
    w.WriteMsg(msg)
}

func (ds *DNSServer) resolve(name string, qtype uint16) []dns.RR {
    ds.mu.RLock()
    defer ds.mu.RUnlock()
    
    key := strings.ToLower(name)
    records, exists := ds.records[key]
    if !exists {
        return nil
    }
    
    var result []dns.RR
    for _, record := range records {
        if record.Header().Rrtype == qtype {
            result = append(result, record)
        }
    }
    
    return result
}
```

### 2.2 レコードの動的管理

```go
func (ds *DNSServer) AddRecord(name string, rtype uint16, value string, ttl uint32) error {
    ds.mu.Lock()
    defer ds.mu.Unlock()
    
    var record dns.RR
    var err error
    
    switch rtype {
    case dns.TypeA:
        record, err = dns.NewRR(fmt.Sprintf("%s %d IN A %s", name, ttl, value))
    case dns.TypeAAAA:
        record, err = dns.NewRR(fmt.Sprintf("%s %d IN AAAA %s", name, ttl, value))
    case dns.TypeCNAME:
        record, err = dns.NewRR(fmt.Sprintf("%s %d IN CNAME %s", name, ttl, value))
    case dns.TypeTXT:
        record, err = dns.NewRR(fmt.Sprintf("%s %d IN TXT \"%s\"", name, ttl, value))
    case dns.TypeSRV:
        record, err = dns.NewRR(fmt.Sprintf("%s %d IN SRV %s", name, ttl, value))
    default:
        return fmt.Errorf("unsupported record type: %d", rtype)
    }
    
    if err != nil {
        return err
    }
    
    key := strings.ToLower(name)
    ds.records[key] = append(ds.records[key], record)
    
    return nil
}

func (ds *DNSServer) RemoveRecord(name string, rtype uint16) {
    ds.mu.Lock()
    defer ds.mu.Unlock()
    
    key := strings.ToLower(name)
    records, exists := ds.records[key]
    if !exists {
        return
    }
    
    var filtered []dns.RR
    for _, record := range records {
        if record.Header().Rrtype != rtype {
            filtered = append(filtered, record)
        }
    }
    
    if len(filtered) == 0 {
        delete(ds.records, key)
    } else {
        ds.records[key] = filtered
    }
}
```

## 3. サービスディスカバリの実装

### 3.1 サービス登録管理

```go
type Service struct {
    Name      string            `json:"name"`
    ID        string            `json:"id"`
    Address   string            `json:"address"`
    Port      int               `json:"port"`
    Tags      []string          `json:"tags"`
    Meta      map[string]string `json:"meta"`
    Health    string            `json:"health"`
    TTL       int               `json:"ttl"`
    CreatedAt time.Time         `json:"created_at"`
    UpdatedAt time.Time         `json:"updated_at"`
}

type ServiceRegistry struct {
    services map[string]*Service
    mu       sync.RWMutex
    dnsServer *DNSServer
    healthChecker *HealthChecker
}

func NewServiceRegistry(dnsServer *DNSServer) *ServiceRegistry {
    sr := &ServiceRegistry{
        services:  make(map[string]*Service),
        dnsServer: dnsServer,
    }
    
    sr.healthChecker = NewHealthChecker(sr)
    return sr
}

func (sr *ServiceRegistry) RegisterService(service *Service) error {
    sr.mu.Lock()
    defer sr.mu.Unlock()
    
    service.ID = generateServiceID(service.Name, service.Address, service.Port)
    service.CreatedAt = time.Now()
    service.UpdatedAt = time.Now()
    
    sr.services[service.ID] = service
    
    // DNS レコードの追加
    fqdn := fmt.Sprintf("%s.service.local.", service.Name)
    err := sr.dnsServer.AddRecord(fqdn, dns.TypeA, service.Address, uint32(service.TTL))
    if err != nil {
        return err
    }
    
    // SRV レコードの追加
    srvRecord := fmt.Sprintf("0 0 %d %s", service.Port, service.Address)
    srvFQDN := fmt.Sprintf("_%s._tcp.service.local.", service.Name)
    err = sr.dnsServer.AddRecord(srvFQDN, dns.TypeSRV, srvRecord, uint32(service.TTL))
    if err != nil {
        return err
    }
    
    return nil
}

func (sr *ServiceRegistry) DeregisterService(serviceID string) error {
    sr.mu.Lock()
    defer sr.mu.Unlock()
    
    service, exists := sr.services[serviceID]
    if !exists {
        return fmt.Errorf("service not found: %s", serviceID)
    }
    
    delete(sr.services, serviceID)
    
    // DNS レコードの削除
    fqdn := fmt.Sprintf("%s.service.local.", service.Name)
    sr.dnsServer.RemoveRecord(fqdn, dns.TypeA)
    
    srvFQDN := fmt.Sprintf("_%s._tcp.service.local.", service.Name)
    sr.dnsServer.RemoveRecord(srvFQDN, dns.TypeSRV)
    
    return nil
}

func (sr *ServiceRegistry) GetService(serviceID string) (*Service, error) {
    sr.mu.RLock()
    defer sr.mu.RUnlock()
    
    service, exists := sr.services[serviceID]
    if !exists {
        return nil, fmt.Errorf("service not found: %s", serviceID)
    }
    
    return service, nil
}

func (sr *ServiceRegistry) ListServices() []*Service {
    sr.mu.RLock()
    defer sr.mu.RUnlock()
    
    var services []*Service
    for _, service := range sr.services {
        services = append(services, service)
    }
    
    return services
}
```

### 3.2 ヘルスチェック機能

```go
type HealthChecker struct {
    registry     *ServiceRegistry
    checkInterval time.Duration
    timeout      time.Duration
    quit         chan bool
}

func NewHealthChecker(registry *ServiceRegistry) *HealthChecker {
    return &HealthChecker{
        registry:     registry,
        checkInterval: 30 * time.Second,
        timeout:      5 * time.Second,
        quit:         make(chan bool),
    }
}

func (hc *HealthChecker) Start() {
    ticker := time.NewTicker(hc.checkInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            hc.performHealthChecks()
        case <-hc.quit:
            return
        }
    }
}

func (hc *HealthChecker) Stop() {
    close(hc.quit)
}

func (hc *HealthChecker) performHealthChecks() {
    services := hc.registry.ListServices()
    
    for _, service := range services {
        go hc.checkService(service)
    }
}

func (hc *HealthChecker) checkService(service *Service) {
    client := &http.Client{
        Timeout: hc.timeout,
    }
    
    healthURL := fmt.Sprintf("http://%s:%d/health", service.Address, service.Port)
    
    resp, err := client.Get(healthURL)
    if err != nil {
        hc.markServiceUnhealthy(service.ID)
        return
    }
    defer resp.Body.Close()
    
    if resp.StatusCode == http.StatusOK {
        hc.markServiceHealthy(service.ID)
    } else {
        hc.markServiceUnhealthy(service.ID)
    }
}

func (hc *HealthChecker) markServiceHealthy(serviceID string) {
    service, err := hc.registry.GetService(serviceID)
    if err != nil {
        return
    }
    
    if service.Health != "healthy" {
        hc.registry.mu.Lock()
        service.Health = "healthy"
        service.UpdatedAt = time.Now()
        hc.registry.mu.Unlock()
        
        // DNS レコードを再追加
        fqdn := fmt.Sprintf("%s.service.local.", service.Name)
        hc.registry.dnsServer.AddRecord(fqdn, dns.TypeA, service.Address, uint32(service.TTL))
    }
}

func (hc *HealthChecker) markServiceUnhealthy(serviceID string) {
    service, err := hc.registry.GetService(serviceID)
    if err != nil {
        return
    }
    
    if service.Health != "unhealthy" {
        hc.registry.mu.Lock()
        service.Health = "unhealthy"
        service.UpdatedAt = time.Now()
        hc.registry.mu.Unlock()
        
        // DNS レコードを削除
        fqdn := fmt.Sprintf("%s.service.local.", service.Name)
        hc.registry.dnsServer.RemoveRecord(fqdn, dns.TypeA)
    }
}
```

## 4. マルチネットワーク対応

### 4.1 ネットワーク別DNS

```go
type MultiNetworkDNS struct {
    networks map[string]*DNSServer
    router   *DNSRouter
    mu       sync.RWMutex
}

func (mnd *MultiNetworkDNS) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
    // クライアントのIPアドレスからネットワークを判定
    clientIP := w.RemoteAddr().(*net.UDPAddr).IP
    networkName := mnd.getNetworkByIP(clientIP)
    
    if networkName == "" {
        // デフォルトネットワーク
        networkName = "default"
    }
    
    mnd.mu.RLock()
    dnsServer, exists := mnd.networks[networkName]
    mnd.mu.RUnlock()
    
    if exists {
        dnsServer.ServeDNS(w, r)
    } else {
        // エラー応答
        m := new(dns.Msg)
        m.SetRcode(r, dns.RcodeServerFailure)
        w.WriteMsg(m)
    }
}

func (mnd *MultiNetworkDNS) getNetworkByIP(ip net.IP) string {
    // IPアドレスからネットワークを判定
    for networkName, cidr := range mnd.networkCIDRs {
        if cidr.Contains(ip) {
            return networkName
        }
    }
    return ""
}
```

### 4.2 ネットワーク分離

```go
type NetworkConfig struct {
    Name     string   `json:"name"`
    CIDR     string   `json:"cidr"`
    Gateway  string   `json:"gateway"`
    DNSRules []string `json:"dns_rules"`
}

type DNSRouter struct {
    configs map[string]*NetworkConfig
    mu      sync.RWMutex
}

func (dr *DNSRouter) AddNetwork(config *NetworkConfig) error {
    dr.mu.Lock()
    defer dr.mu.Unlock()
    
    _, _, err := net.ParseCIDR(config.CIDR)
    if err != nil {
        return fmt.Errorf("invalid CIDR: %w", err)
    }
    
    dr.configs[config.Name] = config
    return nil
}

func (dr *DNSRouter) RemoveNetwork(name string) {
    dr.mu.Lock()
    defer dr.mu.Unlock()
    
    delete(dr.configs, name)
}

func (dr *DNSRouter) GetNetworkConfig(name string) (*NetworkConfig, bool) {
    dr.mu.RLock()
    defer dr.mu.RUnlock()
    
    config, exists := dr.configs[name]
    return config, exists
}
```

## 5. キャッシュとパフォーマンス最適化

### 5.1 DNSキャッシュ

```go
type DNSCache struct {
    cache map[string]*CacheEntry
    mu    sync.RWMutex
}

type CacheEntry struct {
    Records   []dns.RR
    Timestamp time.Time
    TTL       time.Duration
}

func NewDNSCache() *DNSCache {
    dc := &DNSCache{
        cache: make(map[string]*CacheEntry),
    }
    
    // キャッシュクリーンアップの開始
    go dc.cleanupRoutine()
    
    return dc
}

func (dc *DNSCache) Get(key string) ([]dns.RR, bool) {
    dc.mu.RLock()
    defer dc.mu.RUnlock()
    
    entry, exists := dc.cache[key]
    if !exists {
        return nil, false
    }
    
    // TTL チェック
    if time.Since(entry.Timestamp) > entry.TTL {
        return nil, false
    }
    
    return entry.Records, true
}

func (dc *DNSCache) Set(key string, records []dns.RR, ttl time.Duration) {
    dc.mu.Lock()
    defer dc.mu.Unlock()
    
    dc.cache[key] = &CacheEntry{
        Records:   records,
        Timestamp: time.Now(),
        TTL:       ttl,
    }
}

func (dc *DNSCache) cleanupRoutine() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            dc.cleanup()
        }
    }
}

func (dc *DNSCache) cleanup() {
    dc.mu.Lock()
    defer dc.mu.Unlock()
    
    now := time.Now()
    for key, entry := range dc.cache {
        if now.Sub(entry.Timestamp) > entry.TTL {
            delete(dc.cache, key)
        }
    }
}
```

### 5.2 負荷分散

```go
type LoadBalancedDNS struct {
    servers []*DNSServer
    current int
    mu      sync.Mutex
}

func (lbd *LoadBalancedDNS) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
    server := lbd.nextServer()
    if server != nil {
        server.ServeDNS(w, r)
    } else {
        // エラー応答
        m := new(dns.Msg)
        m.SetRcode(r, dns.RcodeServerFailure)
        w.WriteMsg(m)
    }
}

func (lbd *LoadBalancedDNS) nextServer() *DNSServer {
    lbd.mu.Lock()
    defer lbd.mu.Unlock()
    
    if len(lbd.servers) == 0 {
        return nil
    }
    
    server := lbd.servers[lbd.current]
    lbd.current = (lbd.current + 1) % len(lbd.servers)
    
    return server
}
```

## 6. HTTP API インターフェース

### 6.1 サービス管理API

```go
type DNSAPIServer struct {
    registry *ServiceRegistry
    router   *mux.Router
}

func NewDNSAPIServer(registry *ServiceRegistry) *DNSAPIServer {
    api := &DNSAPIServer{
        registry: registry,
        router:   mux.NewRouter(),
    }
    
    api.setupRoutes()
    return api
}

func (api *DNSAPIServer) setupRoutes() {
    api.router.HandleFunc("/services", api.listServices).Methods("GET")
    api.router.HandleFunc("/services", api.registerService).Methods("POST")
    api.router.HandleFunc("/services/{id}", api.getService).Methods("GET")
    api.router.HandleFunc("/services/{id}", api.updateService).Methods("PUT")
    api.router.HandleFunc("/services/{id}", api.deregisterService).Methods("DELETE")
}

func (api *DNSAPIServer) registerService(w http.ResponseWriter, r *http.Request) {
    var service Service
    if err := json.NewDecoder(r.Body).Decode(&service); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    if err := api.registry.RegisterService(&service); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(service)
}

func (api *DNSAPIServer) listServices(w http.ResponseWriter, r *http.Request) {
    services := api.registry.ListServices()
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(services)
}

func (api *DNSAPIServer) getService(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    serviceID := vars["id"]
    
    service, err := api.registry.GetService(serviceID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(service)
}

func (api *DNSAPIServer) deregisterService(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    serviceID := vars["id"]
    
    if err := api.registry.DeregisterService(serviceID); err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    w.WriteHeader(http.StatusNoContent)
}
```

### 6.2 DNS レコード管理API

```go
type DNSRecord struct {
    Name  string `json:"name"`
    Type  string `json:"type"`
    Value string `json:"value"`
    TTL   uint32 `json:"ttl"`
}

func (api *DNSAPIServer) addRecord(w http.ResponseWriter, r *http.Request) {
    var record DNSRecord
    if err := json.NewDecoder(r.Body).Decode(&record); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    recordType := dns.StringToType[record.Type]
    if recordType == 0 {
        http.Error(w, "Invalid record type", http.StatusBadRequest)
        return
    }
    
    err := api.registry.dnsServer.AddRecord(record.Name, recordType, record.Value, record.TTL)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusCreated)
}

func (api *DNSAPIServer) removeRecord(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    name := vars["name"]
    recordType := vars["type"]
    
    rtype := dns.StringToType[recordType]
    if rtype == 0 {
        http.Error(w, "Invalid record type", http.StatusBadRequest)
        return
    }
    
    api.registry.dnsServer.RemoveRecord(name, rtype)
    w.WriteHeader(http.StatusNoContent)
}
```

## 7. 統合とテスト

### 7.1 統合テスト

```go
func TestDNSIntegration(t *testing.T) {
    // DNSサーバーの起動
    dnsServer := NewDNSServer(":5353")
    go dnsServer.Start()
    defer dnsServer.Stop()
    
    // サービスレジストリの設定
    registry := NewServiceRegistry(dnsServer)
    
    // テストサービスの登録
    service := &Service{
        Name:    "test-service",
        Address: "192.168.1.100",
        Port:    8080,
        TTL:     300,
    }
    
    err := registry.RegisterService(service)
    if err != nil {
        t.Fatalf("Failed to register service: %v", err)
    }
    
    // DNS クエリのテスト
    c := new(dns.Client)
    m := new(dns.Msg)
    m.SetQuestion(dns.Fqdn("test-service.service.local"), dns.TypeA)
    
    r, _, err := c.Exchange(m, "127.0.0.1:5353")
    if err != nil {
        t.Fatalf("DNS query failed: %v", err)
    }
    
    if len(r.Answer) == 0 {
        t.Fatal("No DNS answers received")
    }
    
    aRecord := r.Answer[0].(*dns.A)
    if aRecord.A.String() != "192.168.1.100" {
        t.Errorf("Expected 192.168.1.100, got %s", aRecord.A.String())
    }
}
```

### 7.2 負荷テスト

```go
func BenchmarkDNSServer(b *testing.B) {
    dnsServer := NewDNSServer(":5353")
    go dnsServer.Start()
    defer dnsServer.Stop()
    
    // テストレコードを追加
    dnsServer.AddRecord("test.local.", dns.TypeA, "192.168.1.1", 300)
    
    c := new(dns.Client)
    m := new(dns.Msg)
    m.SetQuestion(dns.Fqdn("test.local"), dns.TypeA)
    
    b.ResetTimer()
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, _, err := c.Exchange(m, "127.0.0.1:5353")
            if err != nil {
                b.Error(err)
            }
        }
    })
}
```

## 8. 参考資料

### 8.1 必読ドキュメント

- [RFC 1035 - DNS Implementation and Specification](https://tools.ietf.org/html/rfc1035)
- [RFC 2782 - SRV Records](https://tools.ietf.org/html/rfc2782)
- [miekg/dns - Go DNS library](https://github.com/miekg/dns)

### 8.2 参考実装

- [CoreDNS](https://github.com/coredns/coredns)
- [Consul](https://github.com/hashicorp/consul)
- [etcd](https://github.com/etcd-io/etcd)

### 8.3 理解度確認チェックリスト

- [ ] DNS プロトコルの動作原理を理解している
- [ ] DNS レコードの種類と用途を説明できる
- [ ] サービス登録の仕組みを実装できる
- [ ] ヘルスチェック機能を実装できる
- [ ] マルチネットワーク対応を実装できる
- [ ] DNS キャッシュの実装ができる

これらの知識を身につけることで、Phase 6のコンテナ DNS とサービスディスカバリ実装に効果的に取り組むことができます。

---

## 補助教材

Phase 6をより深く理解するために、以下の補助教材を参照することをお勧めします：

- **[Service Discovery Patterns - 理論的理解](../../resources/05-distributed-systems/service-discovery-patterns.md)**
  - サービスディスカバリの基本概念
  - Client-Side/Server-Side Discovery パターン
  - Service Registry の設計原理
  - 高可用性とパフォーマンス最適化

- **[Linux Bridge と veth - 理論的理解](../../resources/03-networking-concepts/bridge-veth-theory.md)**
  - マルチネットワーク環境での DNS
  - ネットワーク分離とルーティング
  - 仮想ネットワーク間の通信制御

これらの資料は、Phase 6で実装するコンテナ DNS とサービスディスカバリの理論的基盤を深く理解するのに役立ちます。特に、分散システムにおけるサービス発見の重要性と、ネットワーク分離環境での名前解決の複雑性を学ぶことができます。