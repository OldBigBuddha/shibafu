# Phase 7 事前学習ガイド - 統合コンテナランタイム

このドキュメントは、Phase 7で統合コンテナランタイムを実装するために必要な基礎知識をまとめたものです。

## 1. 統合ランタイムの概念

### 1.1 統合ランタイムとは

**定義と目的**
- 複数のコンテナ技術を統合した包括的なランタイム
- ネットワーク、ストレージ、DNS、レジストリの統合管理
- 単一インターフェースによる運用の簡素化

**主要な機能**
- コンテナのライフサイクル管理
- リソース管理と制限
- ネットワーク設定と管理
- ストレージボリュームの管理
- サービスディスカバリとDNS
- 監視と日誌機能

### 1.2 アーキテクチャ設計

**レイヤー構造**
```
┌─────────────────────────────────────┐
│         Management API              │
├─────────────────────────────────────┤
│    Container    │    Network        │
│    Manager      │    Manager        │
├─────────────────────────────────────┤
│    Storage      │    DNS/Service    │
│    Manager      │    Discovery      │
├─────────────────────────────────────┤
│         Resource Manager            │
├─────────────────────────────────────┤
│         Event System                │
└─────────────────────────────────────┘
```

**コンポーネント間の関係**
```
API Server ←→ Container Manager ←→ Network Manager
     ↓              ↓                    ↓
Resource Manager ←→ Storage Manager ←→ DNS Manager
```

## 2. コンテナランタイムの統合実装

### 2.1 統合ランタイムエンジン

```go
type RuntimeEngine struct {
    containerManager *ContainerManager
    networkManager   *NetworkManager
    storageManager   *StorageManager
    dnsManager       *DNSManager
    resourceManager  *ResourceManager
    eventBus         *EventBus
    
    mu sync.RWMutex
}

func NewRuntimeEngine(config *RuntimeConfig) *RuntimeEngine {
    eventBus := NewEventBus()
    
    re := &RuntimeEngine{
        eventBus: eventBus,
    }
    
    // 各マネージャーの初期化
    re.containerManager = NewContainerManager(eventBus)
    re.networkManager = NewNetworkManager(eventBus)
    re.storageManager = NewStorageManager(eventBus)
    re.dnsManager = NewDNSManager(eventBus)
    re.resourceManager = NewResourceManager(eventBus)
    
    // イベントハンドラーの設定
    re.setupEventHandlers()
    
    return re
}

func (re *RuntimeEngine) setupEventHandlers() {
    // コンテナ作成時のイベントハンドラー
    re.eventBus.Subscribe("container.created", re.handleContainerCreated)
    re.eventBus.Subscribe("container.destroyed", re.handleContainerDestroyed)
    
    // ネットワーク関連のイベントハンドラー
    re.eventBus.Subscribe("network.created", re.handleNetworkCreated)
    re.eventBus.Subscribe("network.destroyed", re.handleNetworkDestroyed)
    
    // ストレージ関連のイベントハンドラー
    re.eventBus.Subscribe("volume.mounted", re.handleVolumeMounted)
    re.eventBus.Subscribe("volume.unmounted", re.handleVolumeUnmounted)
}
```

### 2.2 コンテナマネージャー

```go
type ContainerManager struct {
    containers map[string]*Container
    factory    libcontainer.Factory
    eventBus   *EventBus
    mu         sync.RWMutex
}

type Container struct {
    ID          string
    Name        string
    Image       string
    Config      *ContainerConfig
    State       ContainerState
    Process     libcontainer.Process
    CreatedAt   time.Time
    StartedAt   time.Time
    FinishedAt  time.Time
    ExitCode    int
    Networks    []string
    Volumes     []string
    Resources   *ResourceConfig
}

type ContainerConfig struct {
    Image         string            `json:"image"`
    Command       []string          `json:"command"`
    Args          []string          `json:"args"`
    Env           []string          `json:"env"`
    WorkingDir    string            `json:"working_dir"`
    User          string            `json:"user"`
    Volumes       []VolumeMount     `json:"volumes"`
    Networks      []NetworkConfig   `json:"networks"`
    Resources     *ResourceConfig   `json:"resources"`
    Labels        map[string]string `json:"labels"`
    Annotations   map[string]string `json:"annotations"`
}

func (cm *ContainerManager) CreateContainer(req *CreateContainerRequest) (*Container, error) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    
    containerID := generateContainerID()
    
    container := &Container{
        ID:        containerID,
        Name:      req.Name,
        Image:     req.Config.Image,
        Config:    req.Config,
        State:     ContainerStateCreated,
        CreatedAt: time.Now(),
        Networks:  req.Config.Networks,
        Volumes:   req.Config.Volumes,
        Resources: req.Config.Resources,
    }
    
    // libcontainer設定の作成
    libcontainerConfig, err := cm.buildLibcontainerConfig(container)
    if err != nil {
        return nil, fmt.Errorf("failed to build libcontainer config: %w", err)
    }
    
    // コンテナの作成
    libcontainerContainer, err := cm.factory.Create(containerID, libcontainerConfig)
    if err != nil {
        return nil, fmt.Errorf("failed to create libcontainer: %w", err)
    }
    
    container.libcontainer = libcontainerContainer
    cm.containers[containerID] = container
    
    // イベントの発行
    cm.eventBus.Publish("container.created", ContainerEvent{
        ContainerID: containerID,
        Container:   container,
    })
    
    return container, nil
}

func (cm *ContainerManager) StartContainer(containerID string) error {
    cm.mu.Lock()
    container, exists := cm.containers[containerID]
    cm.mu.Unlock()
    
    if !exists {
        return fmt.Errorf("container not found: %s", containerID)
    }
    
    if container.State != ContainerStateCreated {
        return fmt.Errorf("container is not in created state: %s", container.State)
    }
    
    // プロセスの設定
    process := &libcontainer.Process{
        Args:   append(container.Config.Command, container.Config.Args...),
        Env:    container.Config.Env,
        User:   container.Config.User,
        Cwd:    container.Config.WorkingDir,
        Stdin:  os.Stdin,
        Stdout: os.Stdout,
        Stderr: os.Stderr,
    }
    
    err := container.libcontainer.Run(process)
    if err != nil {
        return fmt.Errorf("failed to start container: %w", err)
    }
    
    container.State = ContainerStateRunning
    container.StartedAt = time.Now()
    container.Process = process
    
    // イベントの発行
    cm.eventBus.Publish("container.started", ContainerEvent{
        ContainerID: containerID,
        Container:   container,
    })
    
    return nil
}

func (cm *ContainerManager) StopContainer(containerID string) error {
    cm.mu.Lock()
    container, exists := cm.containers[containerID]
    cm.mu.Unlock()
    
    if !exists {
        return fmt.Errorf("container not found: %s", containerID)
    }
    
    if container.State != ContainerStateRunning {
        return fmt.Errorf("container is not running: %s", container.State)
    }
    
    // SIGTERM送信
    err := container.libcontainer.Signal(syscall.SIGTERM)
    if err != nil {
        return fmt.Errorf("failed to send SIGTERM: %w", err)
    }
    
    // グレースフルシャットダウンの待機
    done := make(chan error, 1)
    go func() {
        _, err := container.Process.Wait()
        done <- err
    }()
    
    select {
    case <-done:
        // 正常終了
    case <-time.After(30 * time.Second):
        // タイムアウト時はSIGKILL
        container.libcontainer.Signal(syscall.SIGKILL)
        <-done
    }
    
    container.State = ContainerStateStopped
    container.FinishedAt = time.Now()
    
    // イベントの発行
    cm.eventBus.Publish("container.stopped", ContainerEvent{
        ContainerID: containerID,
        Container:   container,
    })
    
    return nil
}
```

### 2.3 リソース管理

```go
type ResourceManager struct {
    allocations map[string]*ResourceAllocation
    limits      *ResourceLimits
    eventBus    *EventBus
    mu          sync.RWMutex
}

type ResourceAllocation struct {
    ContainerID string
    CPU         int64  // CPU時間（ミリ秒）
    Memory      int64  // メモリ（バイト）
    Disk        int64  // ディスク容量（バイト）
    Network     int64  // ネットワーク帯域（bps）
}

type ResourceLimits struct {
    MaxCPU     int64
    MaxMemory  int64
    MaxDisk    int64
    MaxNetwork int64
}

func (rm *ResourceManager) AllocateResources(containerID string, req *ResourceRequest) error {
    rm.mu.Lock()
    defer rm.mu.Unlock()
    
    // リソース可用性の確認
    if !rm.canAllocate(req) {
        return fmt.Errorf("insufficient resources")
    }
    
    allocation := &ResourceAllocation{
        ContainerID: containerID,
        CPU:         req.CPU,
        Memory:      req.Memory,
        Disk:        req.Disk,
        Network:     req.Network,
    }
    
    rm.allocations[containerID] = allocation
    
    // cgroupsでのリソース制限適用
    return rm.applyCgroupLimits(containerID, req)
}

func (rm *ResourceManager) canAllocate(req *ResourceRequest) bool {
    totalCPU := req.CPU
    totalMemory := req.Memory
    totalDisk := req.Disk
    
    for _, alloc := range rm.allocations {
        totalCPU += alloc.CPU
        totalMemory += alloc.Memory
        totalDisk += alloc.Disk
    }
    
    return totalCPU <= rm.limits.MaxCPU &&
           totalMemory <= rm.limits.MaxMemory &&
           totalDisk <= rm.limits.MaxDisk
}

func (rm *ResourceManager) applyCgroupLimits(containerID string, req *ResourceRequest) error {
    cgroupPath := fmt.Sprintf("/sys/fs/cgroup/memory/containers/%s", containerID)
    
    if err := os.MkdirAll(cgroupPath, 0755); err != nil {
        return err
    }
    
    // メモリ制限の設定
    memoryLimit := filepath.Join(cgroupPath, "memory.limit_in_bytes")
    if err := ioutil.WriteFile(memoryLimit, []byte(fmt.Sprintf("%d", req.Memory)), 0644); err != nil {
        return err
    }
    
    // CPU制限の設定
    cpuPath := fmt.Sprintf("/sys/fs/cgroup/cpu/containers/%s", containerID)
    if err := os.MkdirAll(cpuPath, 0755); err != nil {
        return err
    }
    
    cpuQuota := filepath.Join(cpuPath, "cpu.cfs_quota_us")
    if err := ioutil.WriteFile(cpuQuota, []byte(fmt.Sprintf("%d", req.CPU*1000)), 0644); err != nil {
        return err
    }
    
    return nil
}
```

## 3. 統合監視システム

### 3.1 メトリクス収集

```go
type MetricsCollector struct {
    metrics map[string]*Metric
    mu      sync.RWMutex
}

type Metric struct {
    Name      string                 `json:"name"`
    Type      MetricType             `json:"type"`
    Value     float64                `json:"value"`
    Labels    map[string]string      `json:"labels"`
    Timestamp time.Time              `json:"timestamp"`
}

type MetricType string

const (
    CounterMetric   MetricType = "counter"
    GaugeMetric     MetricType = "gauge"
    HistogramMetric MetricType = "histogram"
)

func (mc *MetricsCollector) CollectContainerMetrics(containerID string) error {
    // CPU使用率の収集
    cpuUsage, err := mc.getCPUUsage(containerID)
    if err != nil {
        return err
    }
    
    mc.recordMetric(&Metric{
        Name:      "container_cpu_usage",
        Type:      GaugeMetric,
        Value:     cpuUsage,
        Labels:    map[string]string{"container_id": containerID},
        Timestamp: time.Now(),
    })
    
    // メモリ使用量の収集
    memoryUsage, err := mc.getMemoryUsage(containerID)
    if err != nil {
        return err
    }
    
    mc.recordMetric(&Metric{
        Name:      "container_memory_usage",
        Type:      GaugeMetric,
        Value:     float64(memoryUsage),
        Labels:    map[string]string{"container_id": containerID},
        Timestamp: time.Now(),
    })
    
    // ネットワーク統計の収集
    networkStats, err := mc.getNetworkStats(containerID)
    if err != nil {
        return err
    }
    
    mc.recordMetric(&Metric{
        Name:      "container_network_rx_bytes",
        Type:      CounterMetric,
        Value:     float64(networkStats.RxBytes),
        Labels:    map[string]string{"container_id": containerID},
        Timestamp: time.Now(),
    })
    
    mc.recordMetric(&Metric{
        Name:      "container_network_tx_bytes",
        Type:      CounterMetric,
        Value:     float64(networkStats.TxBytes),
        Labels:    map[string]string{"container_id": containerID},
        Timestamp: time.Now(),
    })
    
    return nil
}

func (mc *MetricsCollector) getCPUUsage(containerID string) (float64, error) {
    cpuacctPath := fmt.Sprintf("/sys/fs/cgroup/cpuacct/containers/%s/cpuacct.usage", containerID)
    
    data, err := ioutil.ReadFile(cpuacctPath)
    if err != nil {
        return 0, err
    }
    
    usage, err := strconv.ParseInt(strings.TrimSpace(string(data)), 10, 64)
    if err != nil {
        return 0, err
    }
    
    // ナノ秒から秒に変換
    return float64(usage) / 1e9, nil
}

func (mc *MetricsCollector) getMemoryUsage(containerID string) (int64, error) {
    memoryPath := fmt.Sprintf("/sys/fs/cgroup/memory/containers/%s/memory.usage_in_bytes", containerID)
    
    data, err := ioutil.ReadFile(memoryPath)
    if err != nil {
        return 0, err
    }
    
    usage, err := strconv.ParseInt(strings.TrimSpace(string(data)), 10, 64)
    if err != nil {
        return 0, err
    }
    
    return usage, nil
}
```

### 3.2 ログ管理

```go
type LogManager struct {
    logWriters map[string]*LogWriter
    mu         sync.RWMutex
}

type LogWriter struct {
    containerID string
    file        *os.File
    encoder     *json.Encoder
    mu          sync.Mutex
}

type LogEntry struct {
    ContainerID string    `json:"container_id"`
    Timestamp   time.Time `json:"timestamp"`
    Stream      string    `json:"stream"`
    Message     string    `json:"message"`
    Level       string    `json:"level"`
}

func (lm *LogManager) CreateLogWriter(containerID string) (*LogWriter, error) {
    lm.mu.Lock()
    defer lm.mu.Unlock()
    
    logPath := fmt.Sprintf("/var/log/containers/%s.log", containerID)
    
    if err := os.MkdirAll(filepath.Dir(logPath), 0755); err != nil {
        return nil, err
    }
    
    file, err := os.OpenFile(logPath, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
    if err != nil {
        return nil, err
    }
    
    logWriter := &LogWriter{
        containerID: containerID,
        file:        file,
        encoder:     json.NewEncoder(file),
    }
    
    lm.logWriters[containerID] = logWriter
    
    return logWriter, nil
}

func (lw *LogWriter) WriteLog(stream, message, level string) error {
    lw.mu.Lock()
    defer lw.mu.Unlock()
    
    entry := LogEntry{
        ContainerID: lw.containerID,
        Timestamp:   time.Now(),
        Stream:      stream,
        Message:     message,
        Level:       level,
    }
    
    return lw.encoder.Encode(entry)
}

func (lm *LogManager) GetLogs(containerID string, since time.Time, follow bool) (io.ReadCloser, error) {
    lm.mu.RLock()
    defer lm.mu.RUnlock()
    
    logPath := fmt.Sprintf("/var/log/containers/%s.log", containerID)
    
    if follow {
        return lm.followLogs(logPath, since)
    }
    
    return lm.readLogs(logPath, since)
}

func (lm *LogManager) followLogs(logPath string, since time.Time) (io.ReadCloser, error) {
    cmd := exec.Command("tail", "-f", logPath)
    
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return nil, err
    }
    
    err = cmd.Start()
    if err != nil {
        return nil, err
    }
    
    return stdout, nil
}
```

## 4. 統合API設計

### 4.1 REST API実装

```go
type APIServer struct {
    runtimeEngine *RuntimeEngine
    router        *mux.Router
    server        *http.Server
}

func NewAPIServer(runtimeEngine *RuntimeEngine) *APIServer {
    api := &APIServer{
        runtimeEngine: runtimeEngine,
        router:        mux.NewRouter(),
    }
    
    api.setupRoutes()
    
    api.server = &http.Server{
        Addr:    ":8080",
        Handler: api.router,
    }
    
    return api
}

func (api *APIServer) setupRoutes() {
    // コンテナ管理API
    api.router.HandleFunc("/containers", api.listContainers).Methods("GET")
    api.router.HandleFunc("/containers", api.createContainer).Methods("POST")
    api.router.HandleFunc("/containers/{id}", api.getContainer).Methods("GET")
    api.router.HandleFunc("/containers/{id}/start", api.startContainer).Methods("POST")
    api.router.HandleFunc("/containers/{id}/stop", api.stopContainer).Methods("POST")
    api.router.HandleFunc("/containers/{id}", api.deleteContainer).Methods("DELETE")
    
    // ログ管理API
    api.router.HandleFunc("/containers/{id}/logs", api.getContainerLogs).Methods("GET")
    
    // メトリクス API
    api.router.HandleFunc("/metrics", api.getMetrics).Methods("GET")
    
    // ネットワーク管理API
    api.router.HandleFunc("/networks", api.listNetworks).Methods("GET")
    api.router.HandleFunc("/networks", api.createNetwork).Methods("POST")
    api.router.HandleFunc("/networks/{id}", api.deleteNetwork).Methods("DELETE")
    
    // ボリューム管理API
    api.router.HandleFunc("/volumes", api.listVolumes).Methods("GET")
    api.router.HandleFunc("/volumes", api.createVolume).Methods("POST")
    api.router.HandleFunc("/volumes/{id}", api.deleteVolume).Methods("DELETE")
}

func (api *APIServer) createContainer(w http.ResponseWriter, r *http.Request) {
    var req CreateContainerRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    container, err := api.runtimeEngine.containerManager.CreateContainer(&req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(container)
}

func (api *APIServer) startContainer(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    containerID := vars["id"]
    
    err := api.runtimeEngine.containerManager.StartContainer(containerID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusNoContent)
}

func (api *APIServer) getContainerLogs(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    containerID := vars["id"]
    
    // クエリパラメータの解析
    since := time.Time{}
    if sinceStr := r.URL.Query().Get("since"); sinceStr != "" {
        var err error
        since, err = time.Parse(time.RFC3339, sinceStr)
        if err != nil {
            http.Error(w, "Invalid since parameter", http.StatusBadRequest)
            return
        }
    }
    
    follow := r.URL.Query().Get("follow") == "true"
    
    logReader, err := api.runtimeEngine.logManager.GetLogs(containerID, since, follow)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer logReader.Close()
    
    w.Header().Set("Content-Type", "application/json")
    
    if follow {
        // WebSocketやServer-Sent Eventsでストリーミング
        api.streamLogs(w, logReader)
    } else {
        io.Copy(w, logReader)
    }
}
```

### 4.2 WebSocket による リアルタイム通信

```go
type WebSocketManager struct {
    upgrader    websocket.Upgrader
    connections map[string]*websocket.Conn
    mu          sync.RWMutex
}

func NewWebSocketManager() *WebSocketManager {
    return &WebSocketManager{
        upgrader: websocket.Upgrader{
            CheckOrigin: func(r *http.Request) bool {
                return true // 本番環境では適切な検証を行う
            },
        },
        connections: make(map[string]*websocket.Conn),
    }
}

func (wsm *WebSocketManager) HandleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := wsm.upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket upgrade failed: %v", err)
        return
    }
    defer conn.Close()
    
    clientID := generateClientID()
    
    wsm.mu.Lock()
    wsm.connections[clientID] = conn
    wsm.mu.Unlock()
    
    defer func() {
        wsm.mu.Lock()
        delete(wsm.connections, clientID)
        wsm.mu.Unlock()
    }()
    
    // クライアントからのメッセージを処理
    for {
        var msg WebSocketMessage
        err := conn.ReadJSON(&msg)
        if err != nil {
            log.Printf("WebSocket read error: %v", err)
            break
        }
        
        wsm.handleMessage(clientID, &msg)
    }
}

func (wsm *WebSocketManager) BroadcastEvent(event *Event) {
    wsm.mu.RLock()
    defer wsm.mu.RUnlock()
    
    for clientID, conn := range wsm.connections {
        err := conn.WriteJSON(event)
        if err != nil {
            log.Printf("WebSocket write error for client %s: %v", clientID, err)
            // 接続を削除
            delete(wsm.connections, clientID)
        }
    }
}
```

## 5. 設定管理

### 5.1 設定ファイル構造

```go
type RuntimeConfig struct {
    Runtime     RuntimeSettings     `yaml:"runtime"`
    Network     NetworkSettings     `yaml:"network"`
    Storage     StorageSettings     `yaml:"storage"`
    DNS         DNSSettings         `yaml:"dns"`
    Logging     LoggingSettings     `yaml:"logging"`
    Metrics     MetricsSettings     `yaml:"metrics"`
    API         APISettings         `yaml:"api"`
}

type RuntimeSettings struct {
    StateDir       string `yaml:"state_dir"`
    CgroupDriver   string `yaml:"cgroup_driver"`
    DefaultRuntime string `yaml:"default_runtime"`
}

type NetworkSettings struct {
    DefaultBridge string   `yaml:"default_bridge"`
    BridgeIP      string   `yaml:"bridge_ip"`
    DNSServers    []string `yaml:"dns_servers"`
}

type StorageSettings struct {
    Driver   string `yaml:"driver"`
    RootDir  string `yaml:"root_dir"`
    Options  map[string]string `yaml:"options"`
}

type DNSSettings struct {
    Enabled    bool   `yaml:"enabled"`
    ListenAddr string `yaml:"listen_addr"`
    Domain     string `yaml:"domain"`
}

type LoggingSettings struct {
    Level    string `yaml:"level"`
    Format   string `yaml:"format"`
    Output   string `yaml:"output"`
    Rotation LogRotationSettings `yaml:"rotation"`
}

type MetricsSettings struct {
    Enabled    bool   `yaml:"enabled"`
    ListenAddr string `yaml:"listen_addr"`
    Path       string `yaml:"path"`
}

type APISettings struct {
    ListenAddr string `yaml:"listen_addr"`
    TLS        TLSSettings `yaml:"tls"`
}

func LoadConfig(configPath string) (*RuntimeConfig, error) {
    data, err := ioutil.ReadFile(configPath)
    if err != nil {
        return nil, err
    }
    
    var config RuntimeConfig
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, err
    }
    
    // デフォルト値の設定
    if config.Runtime.StateDir == "" {
        config.Runtime.StateDir = "/var/lib/runtime"
    }
    
    if config.Network.DefaultBridge == "" {
        config.Network.DefaultBridge = "runtime0"
    }
    
    return &config, nil
}
```

### 5.2 動的設定更新

```go
type ConfigManager struct {
    config     *RuntimeConfig
    configFile string
    mu         sync.RWMutex
    callbacks  []func(*RuntimeConfig)
}

func NewConfigManager(configFile string) *ConfigManager {
    return &ConfigManager{
        configFile: configFile,
        callbacks:  make([]func(*RuntimeConfig), 0),
    }
}

func (cm *ConfigManager) LoadConfig() error {
    config, err := LoadConfig(cm.configFile)
    if err != nil {
        return err
    }
    
    cm.mu.Lock()
    cm.config = config
    cm.mu.Unlock()
    
    // 設定変更のコールバック実行
    for _, callback := range cm.callbacks {
        callback(config)
    }
    
    return nil
}

func (cm *ConfigManager) WatchConfig() {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        log.Printf("Failed to create config watcher: %v", err)
        return
    }
    defer watcher.Close()
    
    err = watcher.Add(cm.configFile)
    if err != nil {
        log.Printf("Failed to watch config file: %v", err)
        return
    }
    
    for {
        select {
        case event := <-watcher.Events:
            if event.Op&fsnotify.Write == fsnotify.Write {
                log.Printf("Config file changed: %s", event.Name)
                cm.LoadConfig()
            }
        case err := <-watcher.Errors:
            log.Printf("Config watcher error: %v", err)
        }
    }
}

func (cm *ConfigManager) OnConfigChange(callback func(*RuntimeConfig)) {
    cm.callbacks = append(cm.callbacks, callback)
}
```

## 6. 参考資料

### 6.1 必読ドキュメント

- [libcontainer Documentation](https://github.com/opencontainers/runc/tree/main/libcontainer)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [cgroups Documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)

### 6.2 参考実装

- [containerd](https://github.com/containerd/containerd)
- [Docker Engine](https://github.com/moby/moby)
- [Podman](https://github.com/containers/podman)

### 6.3 理解度確認チェックリスト

- [ ] 統合ランタイムの設計を理解している
- [ ] コンテナライフサイクル管理を実装できる
- [ ] リソース管理システムを実装できる
- [ ] 監視システムの実装ができる
- [ ] REST APIの設計と実装ができる
- [ ] 設定管理システムを実装できる

これらの知識を身につけることで、Phase 7の統合コンテナランタイム実装に効果的に取り組むことができます。

---

## 補助教材

Phase 7をより深く理解するために、以下の補助教材を参照することをお勧めします：

- **[Linux Namespaces - 理論的理解](../../resources/01-linux-kernel-features/namespaces-theory.md)**
  - 統合ランタイムでの namespace 管理
  - プロセス分離の実装原理
  - 階層構造と継承関係

- **[Control Groups - 理論的理解](../../resources/01-linux-kernel-features/cgroups-principles.md)**
  - リソース制限の理論的基盤
  - cgroups v2での統合階層
  - 動的リソース管理

- **[OCI Specifications - 理論的理解](../../resources/02-container-fundamentals/oci-specifications.md)**
  - Runtime Specification準拠
  - 標準化された実装
  - 相互運用性の確保

- **[Linux Bridge と veth - 理論的理解](../../resources/03-networking-concepts/bridge-veth-theory.md)**
  - 統合ネットワーク管理
  - 仮想ネットワークの構築
  - パケット転送制御

- **[OverlayFS - 理論的理解](../../resources/04-storage-systems/overlayfs-theory.md)**
  - 統合ストレージ管理
  - レイヤー構造の最適化
  - 効率的なイメージ管理

- **[Service Discovery Patterns - 理論的理解](../../resources/05-distributed-systems/service-discovery-patterns.md)**
  - 統合サービス発見
  - 動的な構成管理
  - 高可用性システム設計

これらの資料は、Phase 7で実装する統合コンテナランタイムの理論的基盤を深く理解するのに役立ちます。特に、複数のサブシステムを統合する際の設計原則と、システム全体の一貫性を保つ方法を学ぶことができます。