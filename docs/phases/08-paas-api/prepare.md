# Phase 8 �MfҬ�� - !PaaS API

Snɭ����oPhase 8gRESTful API�X_PaaSPlatform as a Service	�������Y�_�kŁj��X�~h�_�ngY

## 1. PaaS���Ư���

### 1.1 PaaSn�,��

**PaaSn�� **
```
                                     
           PaaS Platform             
                                     $
                               
     API      Build    Deploy  
   Gateway   System    Engine  
                               
                                     $
                               
  Routing   Service    Config  
  & LB      Registry    Mgmt   
                               
                                     $
       Container Runtime             
     (Phases 1-7 Integration)       
                                     
```

**PaaSn��**
- **��������յ���**: ���������\b
- **���Ѥ���**: ������K����ʤ���n��
- **��ƣ�**: ���գïn�������xnM�
- **��ӹz�**: �������n�/�
- **-��**: ��	p������n�

### 1.2 Heroku��������

**Git Push Deployment**
```bash
# �zn������
git add .
git commit -m "Feature: add user authentication"
git push paas main

# PaaS����gn�
# 1. ��������
# 2. ����ï�
# 3. �������
# 4. �������
# 5. ��ƣ���
```

**Buildpack��**
```go
type Buildpack interface {
    Detect(appDir string) (bool, error)
    Build(appDir, cacheDir, launchDir string) error
    GetMetadata() BuildpackMetadata
}

type BuildpackMetadata struct {
    Name     string   `json:"name"`
    Version  string   `json:"version"`
    Language string   `json:"language"`
    Stacks   []string `json:"stacks"`
}

// Node.js Buildpack�
type NodejsBuildpack struct{}

func (nb *NodejsBuildpack) Detect(appDir string) (bool, error) {
    packageJsonPath := filepath.Join(appDir, "package.json")
    _, err := os.Stat(packageJsonPath)
    return err == nil, nil
}

func (nb *NodejsBuildpack) Build(appDir, cacheDir, launchDir string) error {
    // package.jsonn�
    packageJson := nb.parsePackageJson(appDir)
    
    // Node.jsn�����
    if err := nb.installNodejs(cacheDir, packageJson.Engines.Node); err != nil {
        return err
    }
    
    // �X��n�����
    if err := nb.runNpmInstall(appDir, cacheDir); err != nil {
        return err
    }
    
    // wչ����n
    return nb.generateLaunchScript(launchDir, packageJson)
}
```

## 2. RESTful API-

### 2.1 ���-

**API ���ݤ��-**
```
POST   /v1/apps                     # �������\
GET    /v1/apps                     # ������� �
GET    /v1/apps/{app_id}            # �������s0
PUT    /v1/apps/{app_id}            # ���������
DELETE /v1/apps/{app_id}            # �������Jd

POST   /v1/apps/{app_id}/deploys    # �������\
GET    /v1/apps/{app_id}/deploys    # �������et
GET    /v1/deploys/{deploy_id}      # �������s0
POST   /v1/deploys/{deploy_id}/rollback # ����ï

GET    /v1/apps/{app_id}/logs       # ��֗
GET    /v1/apps/{app_id}/metrics    # ��꯹֗

POST   /v1/apps/{app_id}/scale      # �����
GET    /v1/apps/{app_id}/processes  # ���� �

POST   /v1/configs                  # -�\
GET    /v1/apps/{app_id}/configs    # ���-�֗
PUT    /v1/apps/{app_id}/configs    # ���-���
```

**������**
```go
type Application struct {
    ID          string            `json:"id"`
    Name        string            `json:"name"`
    Stack       string            `json:"stack"`
    Buildpacks  []string          `json:"buildpacks"`
    Repository  RepositoryInfo    `json:"repository"`
    Config      map[string]string `json:"config"`
    Domains     []string          `json:"domains"`
    Status      AppStatus         `json:"status"`
    CreatedAt   time.Time         `json:"created_at"`
    UpdatedAt   time.Time         `json:"updated_at"`
}

type Deployment struct {
    ID          string            `json:"id"`
    AppID       string            `json:"app_id"`
    Version     string            `json:"version"`
    Status      DeployStatus      `json:"status"`
    Buildpack   string            `json:"buildpack"`
    SourceURL   string            `json:"source_url"`
    ImageDigest string            `json:"image_digest"`
    Config      map[string]string `json:"config"`
    StartedAt   time.Time         `json:"started_at"`
    FinishedAt  *time.Time        `json:"finished_at,omitempty"`
    LogURL      string            `json:"log_url"`
}

type Process struct {
    ID        string            `json:"id"`
    AppID     string            `json:"app_id"`
    Type      string            `json:"type"` // web, worker, etc.
    Command   string            `json:"command"`
    Status    ProcessStatus     `json:"status"`
    Instances int               `json:"instances"`
    Memory    string            `json:"memory"`
    CPU       string            `json:"cpu"`
}
```

### 2.2 API��ѿ��

**Gin���n������**
```go
import "github.com/gin-gonic/gin"

type PaaSAPI struct {
    appService    ApplicationService
    deployService DeploymentService
    authService   AuthenticationService
}

func (api *PaaSAPI) SetupRoutes() *gin.Engine {
    r := gin.Default()
    
    // ��릧�
    r.Use(api.corsMiddleware())
    r.Use(api.loggingMiddleware())
    r.Use(api.rateLimitMiddleware())
    
    v1 := r.Group("/v1")
    v1.Use(api.authMiddleware())
    {
        // �������
        apps := v1.Group("/apps")
        {
            apps.POST("", api.createApplication)
            apps.GET("", api.listApplications)
            apps.GET("/:app_id", api.getApplication)
            apps.PUT("/:app_id", api.updateApplication)
            apps.DELETE("/:app_id", api.deleteApplication)
            
            // �������
            apps.POST("/:app_id/deploys", api.createDeployment)
            apps.GET("/:app_id/deploys", api.listDeployments)
            
            // �����
            apps.POST("/:app_id/scale", api.scaleApplication)
            
            // ��h��꯹
            apps.GET("/:app_id/logs", api.getApplicationLogs)
            apps.GET("/:app_id/metrics", api.getApplicationMetrics)
        }
        
        // �������s0
        v1.GET("/deploys/:deploy_id", api.getDeployment)
        v1.POST("/deploys/:deploy_id/rollback", api.rollbackDeployment)
    }
    
    return r
}

func (api *PaaSAPI) createApplication(c *gin.Context) {
    var req CreateApplicationRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // �<����n֗
    userID := api.getUserIDFromContext(c)
    
    // �������n\
    app, err := api.appService.CreateApplication(userID, &req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, app)
}
```

## 3. �������Ѥ���

### 3.1 Git�����

**Git Push�**
```go
import "github.com/go-git/go-git/v5"

type GitReceiver struct {
    repoDir       string
    buildQueue    chan *BuildJob
    deployService DeploymentService
}

type BuildJob struct {
    AppID     string
    CommitSHA string
    Branch    string
    SourceDir string
    UserID    string
}

func (gr *GitReceiver) HandlePush(appID, userID string, refs []string) error {
    // ��������1n֗
    app, err := gr.deployService.GetApplication(appID)
    if err != nil {
        return err
    }
    
    // �ݸ��n����/��
    repoPath := filepath.Join(gr.repoDir, appID)
    repo, err := gr.cloneOrUpdateRepo(app.Repository.URL, repoPath)
    if err != nil {
        return err
    }
    
    //  �����n֗
    head, err := repo.Head()
    if err != nil {
        return err
    }
    
    // ��ɸ��n�e
    buildJob := &BuildJob{
        AppID:     appID,
        CommitSHA: head.Hash().String(),
        Branch:    "main",
        SourceDir: repoPath,
        UserID:    userID,
    }
    
    select {
    case gr.buildQueue <- buildJob:
        return nil
    case <-time.After(5 * time.Second):
        return fmt.Errorf("build queue is full")
    }
}

func (gr *GitReceiver) cloneOrUpdateRepo(repoURL, localPath string) (*git.Repository, error) {
    if _, err := os.Stat(localPath); os.IsNotExist(err) {
        // ޯ���
        return git.PlainClone(localPath, false, &git.CloneOptions{
            URL: repoURL,
        })
    } else {
        // �X�ݸ��n��
        repo, err := git.PlainOpen(localPath)
        if err != nil {
            return nil, err
        }
        
        workTree, err := repo.Worktree()
        if err != nil {
            return nil, err
        }
        
        err = workTree.Pull(&git.PullOptions{})
        if err != nil && err != git.NoErrAlreadyUpToDate {
            return nil, err
        }
        
        return repo, nil
    }
}
```

### 3.2 ��ɷ���

**����������**
```go
type BuildSystem struct {
    containerRuntime RuntimeEngine
    buildpacks       []Buildpack
    imageRegistry    RegistryClient
}

func (bs *BuildSystem) ProcessBuildJob(job *BuildJob) (*BuildResult, error) {
    // Buildpackn�
    buildpack, err := bs.detectBuildpack(job.SourceDir)
    if err != nil {
        return nil, fmt.Errorf("buildpack detection failed: %w", err)
    }
    
    // ��ɰ�n��
    buildContext := &BuildContext{
        SourceDir: job.SourceDir,
        CacheDir:  bs.getCacheDir(job.AppID),
        OutputDir: bs.getOutputDir(job.AppID, job.CommitSHA),
    }
    
    // Buildpackk����ɟL
    if err := buildpack.Build(buildContext.SourceDir, buildContext.CacheDir, buildContext.OutputDir); err != nil {
        return nil, fmt.Errorf("build failed: %w", err)
    }
    
    // ���ʤ���n��
    imageTag := fmt.Sprintf("%s:%s", job.AppID, job.CommitSHA[:8])
    imageDigest, err := bs.buildContainerImage(buildContext.OutputDir, imageTag)
    if err != nil {
        return nil, fmt.Errorf("image build failed: %w", err)
    }
    
    // 츹��xn�÷�
    if err := bs.imageRegistry.PushImage(imageTag); err != nil {
        return nil, fmt.Errorf("image push failed: %w", err)
    }
    
    return &BuildResult{
        ImageDigest: imageDigest,
        Buildpack:   buildpack.GetMetadata().Name,
        BuildLog:    bs.getBuildLog(job.AppID, job.CommitSHA),
    }, nil
}

func (bs *BuildSystem) detectBuildpack(sourceDir string) (Buildpack, error) {
    for _, bp := range bs.buildpacks {
        if detected, err := bp.Detect(sourceDir); err == nil && detected {
            return bp, nil
        }
    }
    return nil, fmt.Errorf("no suitable buildpack found")
}

func (bs *BuildSystem) buildContainerImage(buildDir, tag string) (string, error) {
    // Dockerfilen
    dockerfile := bs.generateDockerfile(buildDir)
    
    // ��ɳ�ƭ��n\
    buildContext, err := bs.createBuildContext(buildDir, dockerfile)
    if err != nil {
        return "", err
    }
    
    // �������
    buildReq := &BuildImageRequest{
        Tag:     tag,
        Context: buildContext,
    }
    
    return bs.containerRuntime.BuildImage(buildReq)
}
```

### 3.3 ������ȡ

**Blue-Green�������**
```go
type DeploymentStrategy interface {
    Deploy(app *Application, deployment *Deployment) error
    Rollback(app *Application, targetDeployment *Deployment) error
}

type BlueGreenStrategy struct {
    runtime     RuntimeEngine
    loadBalancer LoadBalancer
}

func (bgs *BlueGreenStrategy) Deploy(app *Application, deployment *Deployment) error {
    // �WD��Green	n\
    greenEnv := fmt.Sprintf("%s-green", app.ID)
    
    // �WD����nw�
    containers, err := bgs.startContainers(greenEnv, deployment)
    if err != nil {
        return fmt.Errorf("failed to start green environment: %w", err)
    }
    
    // ����ï
    if err := bgs.waitForHealthy(containers); err != nil {
        bgs.cleanupContainers(containers)
        return fmt.Errorf("health check failed: %w", err)
    }
    
    // ��գïn��H
    if err := bgs.loadBalancer.SwitchTraffic(app.ID, greenEnv); err != nil {
        bgs.cleanupContainers(containers)
        return fmt.Errorf("traffic switch failed: %w", err)
    }
    
    // 簃Blue	n������
    blueEnv := fmt.Sprintf("%s-blue", app.ID)
    go bgs.cleanupEnvironment(blueEnv)
    
    // ��nM	�Green � Blue	
    return bgs.promoteEnvironment(greenEnv, blueEnv)
}

func (bgs *BlueGreenStrategy) startContainers(envName string, deployment *Deployment) ([]*Container, error) {
    var containers []*Container
    
    for i := 0; i < deployment.Instances; i++ {
        containerName := fmt.Sprintf("%s-%d", envName, i)
        
        container, err := bgs.runtime.CreateContainer(&CreateContainerRequest{
            Name:    containerName,
            Image:   deployment.ImageDigest,
            Env:     deployment.Config,
            Command: deployment.Command,
            Memory:  deployment.Memory,
            CPU:     deployment.CPU,
        })
        
        if err != nil {
            // ��j1Wg��X���ʒ������
            bgs.cleanupContainers(containers)
            return nil, err
        }
        
        containers = append(containers, container)
    }
    
    return containers, nil
}

func (bgs *BlueGreenStrategy) waitForHealthy(containers []*Container) error {
    timeout := time.After(5 * time.Minute)
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-timeout:
            return fmt.Errorf("health check timeout")
        case <-ticker.C:
            allHealthy := true
            for _, container := range containers {
                if !bgs.checkContainerHealth(container) {
                    allHealthy = false
                    break
                }
            }
            
            if allHealthy {
                return nil
            }
        }
    }
}
```

## 4. ��ƣ�h�������

### 4.1 �����������

**HTTP����n��**
```go
type ApplicationRouter struct {
    routes        map[string]*RouteTarget // domain -> target
    loadBalancers map[string]LoadBalancer // app_id -> load_balancer
    mu            sync.RWMutex
}

type RouteTarget struct {
    AppID       string
    Environment string
    Containers  []*Container
    HealthCheck string
}

func (ar *ApplicationRouter) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ar.mu.RLock()
    target, exists := ar.routes[r.Host]
    ar.mu.RUnlock()
    
    if !exists {
        http.NotFound(w, r)
        return
    }
    
    // �������k���Hz�
    lb := ar.loadBalancers[target.AppID]
    container := lb.SelectContainer()
    
    if container == nil {
        http.Error(w, "Service Unavailable", http.StatusServiceUnavailable)
        return
    }
    
    // ��������gꯨ���
    ar.proxyRequest(w, r, container)
}

func (ar *ApplicationRouter) UpdateRoutes(app *Application, containers []*Container) error {
    ar.mu.Lock()
    defer ar.mu.Unlock()
    
    for _, domain := range app.Domains {
        ar.routes[domain] = &RouteTarget{
            AppID:       app.ID,
            Environment: "current",
            Containers:  containers,
            HealthCheck: "/health",
        }
    }
    
    // �������n��
    if lb, exists := ar.loadBalancers[app.ID]; exists {
        lb.UpdateContainers(containers)
    } else {
        ar.loadBalancers[app.ID] = NewRoundRobinBalancer(containers)
    }
    
    return nil
}

func (ar *ApplicationRouter) proxyRequest(w http.ResponseWriter, r *http.Request, container *Container) {
    targetURL := &url.URL{
        Scheme: "http",
        Host:   fmt.Sprintf("%s:%d", container.IP, container.Port),
        Path:   r.URL.Path,
    }
    
    proxy := httputil.NewSingleHostReverseProxy(targetURL)
    
    // ����ǣ쯿�������ji	
    proxy.Director = func(req *http.Request) {
        req.URL = targetURL
        req.Host = targetURL.Host
        
        // X-Forwarded-For����n��
        if req.Header.Get("X-Forwarded-For") == "" {
            req.Header.Set("X-Forwarded-For", r.RemoteAddr)
        }
        
        // ��������	����
        req.Header.Set("X-App-ID", container.AppID)
        req.Header.Set("X-Container-ID", container.ID)
    }
    
    proxy.ServeHTTP(w, r)
}
```

### 4.2 SSL/TLSB�

**��<��**
```go
import "golang.org/x/crypto/acme/autocert"

type TLSManager struct {
    certManager *autocert.Manager
    domains     map[string]string // domain -> app_id
    mu          sync.RWMutex
}

func NewTLSManager(cacheDir string) *TLSManager {
    manager := &autocert.Manager{
        Cache:      autocert.DirCache(cacheDir),
        Prompt:     autocert.AcceptTOS,
        HostPolicy: nil, // �g-�
    }
    
    tm := &TLSManager{
        certManager: manager,
        domains:     make(map[string]string),
    }
    
    // ۹����n-�
    manager.HostPolicy = tm.hostPolicy
    
    return tm
}

func (tm *TLSManager) hostPolicy(ctx context.Context, host string) error {
    tm.mu.RLock()
    _, exists := tm.domains[host]
    tm.mu.RUnlock()
    
    if !exists {
        return fmt.Errorf("domain not registered: %s", host)
    }
    
    return nil
}

func (tm *TLSManager) AddDomain(domain, appID string) error {
    tm.mu.Lock()
    defer tm.mu.Unlock()
    
    tm.domains[domain] = appID
    return nil
}

func (tm *TLSManager) GetTLSConfig() *tls.Config {
    return &tls.Config{
        GetCertificate: tm.certManager.GetCertificate,
        NextProtos:     []string{"h2", "http/1.1"},
    }
}
```

## 5. -�h�����ȡ

### 5.1 ��	p�

**�d-�����**
```go
type ConfigManager struct {
    globalConfig map[string]string
    appConfigs   map[string]map[string]string // app_id -> config
    secrets      map[string]map[string]string // app_id -> secrets
    mu           sync.RWMutex
}

func (cm *ConfigManager) GetAppConfig(appID string) map[string]string {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    
    // �����-�K���
    config := make(map[string]string)
    for k, v := range cm.globalConfig {
        config[k] = v
    }
    
    // ����	-�g
�M
    if appConfig, exists := cm.appConfigs[appID]; exists {
        for k, v := range appConfig {
            config[k] = v
        }
    }
    
    // �����Ȓ����գï��M	
    if secrets, exists := cm.secrets[appID]; exists {
        for k, v := range secrets {
            config[k] = v
        }
    }
    
    return config
}

func (cm *ConfigManager) SetAppConfig(appID string, config map[string]string) error {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    
    if cm.appConfigs[appID] == nil {
        cm.appConfigs[appID] = make(map[string]string)
    }
    
    for k, v := range config {
        cm.appConfigs[appID][k] = v
    }
    
    return nil
}

func (cm *ConfigManager) SetSecret(appID, key, value string) error {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    
    if cm.secrets[appID] == nil {
        cm.secrets[appID] = make(map[string]string)
    }
    
    // ������n��
    encryptedValue, err := cm.encrypt(value)
    if err != nil {
        return err
    }
    
    cm.secrets[appID][key] = encryptedValue
    return nil
}

func (cm *ConfigManager) encrypt(value string) (string, error) {
    // AES��n��
    // ��n��goij������(
    return base64.StdEncoding.EncodeToString([]byte(value)), nil
}
```

### 5.2 Մ-���

**-�	�n� **
```go
type ConfigWatcher struct {
    configManager *ConfigManager
    deployService DeploymentService
    eventBus      EventBus
}

func (cw *ConfigWatcher) HandleConfigChange(appID string, newConfig map[string]string) error {
    // �(n-�hn���ï
    currentConfig := cw.configManager.GetAppConfig(appID)
    if cw.configEqual(currentConfig, newConfig) {
        return nil // 	�jW
    }
    
    // -�n��
    if err := cw.configManager.SetAppConfig(appID, newConfig); err != nil {
        return err
    }
    
    // �L-n����xn� LŁK��ï
    requiresRestart := cw.checkRestartRequired(currentConfig, newConfig)
    
    if requiresRestart {
        // ���������g���ʍw�
        return cw.rollingRestart(appID)
    } else {
        // ���������j-�n��
        return cw.hotReloadConfig(appID, newConfig)
    }
}

func (cw *ConfigWatcher) rollingRestart(appID string) error {
    app, err := cw.deployService.GetApplication(appID)
    if err != nil {
        return err
    }
    
    containers, err := cw.deployService.GetApplicationContainers(appID)
    if err != nil {
        return err
    }
    
    // 1dZd���ʒ�w�
    for _, container := range containers {
        // �WD���ʒw�
        newContainer, err := cw.deployService.StartContainerWithNewConfig(app, container)
        if err != nil {
            return err
        }
        
        // ����ï
        if err := cw.waitForHealthy(newContainer); err != nil {
            cw.deployService.StopContainer(newContainer.ID)
            return err
        }
        
        // �������K����ʒd
        cw.deployService.RemoveFromLoadBalancer(container.ID)
        
        // ����n\b
        cw.deployService.StopContainer(container.ID)
        
        // ����ʒ�������k��
        cw.deployService.AddToLoadBalancer(newContainer.ID)
    }
    
    return nil
}
```

## 6. �h���

### 6.1 ���������

**�������**
```go
type LogManager struct {
    logStreams map[string]*LogStream // app_id -> stream
    storage    LogStorage
    mu         sync.RWMutex
}

type LogEntry struct {
    Timestamp   time.Time         `json:"timestamp"`
    AppID       string            `json:"app_id"`
    ContainerID string            `json:"container_id"`
    Process     string            `json:"process"`
    Level       string            `json:"level"`
    Message     string            `json:"message"`
    Metadata    map[string]interface{} `json:"metadata"`
}

func (lm *LogManager) StreamLogs(appID string, w http.ResponseWriter, r *http.Request) {
    // WebSocket��װ���
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        http.Error(w, "WebSocket upgrade failed", http.StatusBadRequest)
        return
    }
    defer conn.Close()
    
    // �������n֗
    stream := lm.getLogStream(appID)
    if stream == nil {
        conn.WriteMessage(websocket.TextMessage, []byte("Log stream not found"))
        return
    }
    
    // �餢��xn���
    logChan := stream.Subscribe()
    defer stream.Unsubscribe(logChan)
    
    for {
        select {
        case logEntry := <-logChan:
            data, _ := json.Marshal(logEntry)
            if err := conn.WriteMessage(websocket.TextMessage, data); err != nil {
                return
            }
        case <-r.Context().Done():
            return
        }
    }
}

func (lm *LogManager) CollectContainerLogs(containerID string) error {
    // �������K����֗
    logReader, err := lm.runtime.GetContainerLogs(containerID)
    if err != nil {
        return err
    }
    defer logReader.Close()
    
    scanner := bufio.NewScanner(logReader)
    for scanner.Scan() {
        line := scanner.Text()
        
        // ������n�
        entry, err := lm.parseLogLine(containerID, line)
        if err != nil {
            continue // ������o!�
        }
        
        // �����xn�X
        if err := lm.storage.WriteLog(entry); err != nil {
            log.Printf("Failed to write log: %v", err)
        }
        
        // �뿤�����xnM�
        if stream := lm.getLogStreamByContainer(containerID); stream != nil {
            stream.Publish(entry)
        }
    }
    
    return scanner.Err()
}
```

### 6.2 ���������꯹

**��꯹��**
```go
type MetricsCollector struct {
    apps         map[string]*AppMetrics
    containerMgr ContainerManager
    mu           sync.RWMutex
}

type AppMetrics struct {
    AppID             string    `json:"app_id"`
    RequestCount      int64     `json:"request_count"`
    RequestsPerSecond float64   `json:"requests_per_second"`
    ResponseTime      float64   `json:"response_time_ms"`
    ErrorRate         float64   `json:"error_rate"`
    CPUUsage          float64   `json:"cpu_usage"`
    MemoryUsage       int64     `json:"memory_usage"`
    DiskUsage         int64     `json:"disk_usage"`
    ActiveConnections int       `json:"active_connections"`
    Timestamp         time.Time `json:"timestamp"`
}

func (mc *MetricsCollector) CollectMetrics() {
    ticker := time.NewTicker(30 * time.Second)
    go func() {
        for range ticker.C {
            mc.collectAllAppMetrics()
        }
    }()
}

func (mc *MetricsCollector) collectAllAppMetrics() {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    
    for appID := range mc.apps {
        metrics, err := mc.collectAppMetrics(appID)
        if err != nil {
            log.Printf("Failed to collect metrics for app %s: %v", appID, err)
            continue
        }
        
        mc.apps[appID] = metrics
    }
}

func (mc *MetricsCollector) collectAppMetrics(appID string) (*AppMetrics, error) {
    containers, err := mc.containerMgr.GetAppContainers(appID)
    if err != nil {
        return nil, err
    }
    
    var totalCPU, totalMemory, totalDisk float64
    var totalRequests int64
    
    for _, container := range containers {
        stats, err := mc.containerMgr.GetContainerStats(container.ID)
        if err != nil {
            continue
        }
        
        totalCPU += stats.CPUUsage
        totalMemory += float64(stats.MemoryUsage)
        totalDisk += float64(stats.DiskUsage)
        totalRequests += stats.RequestCount
    }
    
    return &AppMetrics{
        AppID:        appID,
        RequestCount: totalRequests,
        CPUUsage:     totalCPU / float64(len(containers)),
        MemoryUsage:  int64(totalMemory),
        DiskUsage:    int64(totalDisk),
        Timestamp:    time.Now(),
    }, nil
}

func (mc *MetricsCollector) GetAppMetrics(appID string, duration time.Duration) ([]*AppMetrics, error) {
    // B�������K�N�n��꯹�֗
    return mc.storage.GetMetricsHistory(appID, time.Now().Add(-duration), time.Now())
}
```

## 7. �Ǚ

### 7.1 PaaS-ѿ��
- [The Twelve-Factor App](https://12factor.net/)
- [Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference)
- [Cloud Foundry Architecture](https://docs.cloudfoundry.org/concepts/architecture/)

### 7.2 ���
- [Dokku](https://github.com/dokku/dokku) - Docker Powered PaaS
- [Flynn](https://github.com/flynn/flynn) - Open Source PaaS
- [Tsuru](https://github.com/tsuru/tsuru) - Extensible PaaS

### 7.3 㦺���ï��

- [ ] PaaS���Ư��nhSϒ�WfD�
- [ ] RESTful API-LgM�
- [ ] �������Ѥ����gM�
- [ ] ��ƣ�h������󰒟�gM�
- [ ] -�h�����ȡ���gM�
- [ ] �h������gM�

S��n�X��kdQ�ShgPhase 8n!PaaS API��k���k֊D�ShLgM~Y