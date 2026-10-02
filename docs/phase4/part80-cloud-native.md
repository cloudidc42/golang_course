# Part 80: Cloud-Native Development with Go

## เป้าหมายของบทเรียน
- เข้าใจ Cloud-Native principles
- รู้จัก CNCF landscape
- Implement serverless patterns
- สร้าง FaaS ด้วย AWS Lambda และ GCP Functions
- ทำ container orchestration
- ใช้ GitOps
- Infrastructure as Code

---

## 1. Cloud-Native Principles

The Twelve-Factor App methodology สำหรับ cloud-native applications:

```go
// twelve-factor/app.go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

// Config แสดง 12-factor config pattern
// Factor III: Config - เก็บ config ใน environment variables
type Config struct {
    Port         string
    DatabaseURL  string
    RedisURL     string
    LogLevel     string
    AppName      string
    Version      string
    Environment  string
}

// LoadConfig โหลด config จาก environment
func LoadConfig() Config {
    return Config{
        Port:        getEnv("PORT", "8080"),
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/app"),
        RedisURL:    getEnv("REDIS_URL", "redis://localhost:6379"),
        LogLevel:    getEnv("LOG_LEVEL", "info"),
        AppName:     getEnv("APP_NAME", "my-service"),
        Version:     getEnv("APP_VERSION", "1.0.0"),
        Environment: getEnv("ENVIRONMENT", "development"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

// CloudNativeApp แสดง cloud-native application
type CloudNativeApp struct {
    config  Config
    server  *http.Server
    health  HealthChecker
    ready   bool
}

// HealthChecker interface
type HealthChecker interface {
    IsHealthy() bool
    IsReady() bool
}

// DefaultHealthChecker ตรวจสอบ health
type DefaultHealthChecker struct {
    ready bool
}

func (h *DefaultHealthChecker) IsHealthy() bool { return true }
func (h *DefaultHealthChecker) IsReady() bool   { return h.ready }

// NewCloudNativeApp สร้าง app ใหม่
func NewCloudNativeApp(config Config) *CloudNativeApp {
    app := &CloudNativeApp{
        config: config,
        health: &DefaultHealthChecker{ready: false},
    }
    return app
}

// setupRoutes ตั้งค่า routes
func (app *CloudNativeApp) setupRoutes() *http.ServeMux {
    mux := http.NewServeMux()
    
    // Factor XI: Logs - treat logs as event streams
    mux.HandleFunc("/", app.handleRoot)
    
    // Health endpoints (required for Kubernetes)
    mux.HandleFunc("/healthz", app.handleLiveness)
    mux.HandleFunc("/readyz", app.handleReadiness)
    mux.HandleFunc("/metrics", app.handleMetrics)
    
    return mux
}

func (app *CloudNativeApp) handleRoot(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, `{"service": "%s", "version": "%s", "env": "%s"}`,
        app.config.AppName, app.config.Version, app.config.Environment)
}

func (app *CloudNativeApp) handleLiveness(w http.ResponseWriter, r *http.Request) {
    if app.health.IsHealthy() {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, `{"status": "healthy"}`)
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
        fmt.Fprintf(w, `{"status": "unhealthy"}`)
    }
}

func (app *CloudNativeApp) handleReadiness(w http.ResponseWriter, r *http.Request) {
    if app.health.IsReady() {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, `{"status": "ready"}`)
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
        fmt.Fprintf(w, `{"status": "not ready"}`)
    }
}

func (app *CloudNativeApp) handleMetrics(w http.ResponseWriter, r *http.Request) {
    // Prometheus format metrics
    fmt.Fprintf(w, "# Cloud-native metrics\n")
}

// Start เริ่ม application
// Factor IX: Disposability - fast startup and graceful shutdown
func (app *CloudNativeApp) Start(ctx context.Context) error {
    mux := app.setupRoutes()
    
    app.server = &http.Server{
        Addr:         ":" + app.config.Port,
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // Signal handling สำหรับ graceful shutdown
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
    
    errCh := make(chan error, 1)
    
    go func() {
        fmt.Printf("Starting %s on port %s (env: %s)\n",
            app.config.AppName, app.config.Port, app.config.Environment)
        
        // Mark as ready after startup
        time.AfterFunc(100*time.Millisecond, func() {
            if hc, ok := app.health.(*DefaultHealthChecker); ok {
                hc.ready = true
                fmt.Println("Service is ready to accept traffic")
            }
        })
        
        if err := app.server.ListenAndServe(); err != http.ErrServerClosed {
            errCh <- err
        }
    }()
    
    select {
    case <-ctx.Done():
    case sig := <-sigCh:
        fmt.Printf("Received signal: %v\n", sig)
    case err := <-errCh:
        return err
    }
    
    // Graceful shutdown
    fmt.Println("Shutting down gracefully...")
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    return app.server.Shutdown(shutdownCtx)
}

func main() {
    config := LoadConfig()
    app := NewCloudNativeApp(config)
    
    ctx := context.Background()
    if err := app.Start(ctx); err != nil {
        fmt.Fprintf(os.Stderr, "Server error: %v\n", err)
        os.Exit(1)
    }
}
```

---

## 2. Serverless Patterns in Go

```go
// serverless/patterns.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
)

// Event ข้อมูล event จาก event source
type Event struct {
    ID        string                 `json:"id"`
    Source    string                 `json:"source"`
    Type      string                 `json:"type"`
    Data      map[string]interface{} `json:"data"`
    Timestamp time.Time              `json:"timestamp"`
}

// Response ผลลัพธ์จาก function
type Response struct {
    StatusCode int
    Headers    map[string]string
    Body       string
}

// Handler interface สำหรับ serverless function
type Handler interface {
    Handle(ctx context.Context, event Event) (*Response, error)
}

// FunctionChain chain ของ functions
type FunctionChain struct {
    handlers []Handler
}

// Add เพิ่ม handler
func (fc *FunctionChain) Add(h Handler) *FunctionChain {
    fc.handlers = append(fc.handlers, h)
    return fc
}

// Execute รัน chain
func (fc *FunctionChain) Execute(ctx context.Context, event Event) (*Response, error) {
    var resp *Response
    var err error
    
    for _, h := range fc.handlers {
        resp, err = h.Handle(ctx, event)
        if err != nil {
            return nil, err
        }
        
        // อัปเดต event data ด้วย response
        if resp != nil && resp.Body != "" {
            var newData map[string]interface{}
            if jsonErr := json.Unmarshal([]byte(resp.Body), &newData); jsonErr == nil {
                for k, v := range newData {
                    event.Data[k] = v
                }
            }
        }
    }
    
    return resp, nil
}

// ValidateHandler ตรวจสอบ input
type ValidateHandler struct{}

func (h *ValidateHandler) Handle(ctx context.Context, event Event) (*Response, error) {
    fmt.Printf("[Validate] Event: %s\n", event.Type)
    
    if event.Data == nil {
        return &Response{StatusCode: 400, Body: `{"error": "no data"}`}, nil
    }
    
    return &Response{StatusCode: 200, Body: `{"validated": true}`}, nil
}

// ProcessHandler ประมวลผล
type ProcessHandler struct{}

func (h *ProcessHandler) Handle(ctx context.Context, event Event) (*Response, error) {
    fmt.Printf("[Process] Processing event: %s\n", event.ID)
    
    result := map[string]interface{}{
        "processed": true,
        "event_id":  event.ID,
    }
    
    data, _ := json.Marshal(result)
    return &Response{StatusCode: 200, Body: string(data)}, nil
}

// NotifyHandler ส่ง notification
type NotifyHandler struct{}

func (h *NotifyHandler) Handle(ctx context.Context, event Event) (*Response, error) {
    fmt.Printf("[Notify] Sending notification for: %s\n", event.ID)
    return &Response{StatusCode: 200, Body: `{"notified": true}`}, nil
}

// FanOut pattern: ส่ง event ไปหลาย functions พร้อมกัน
type FanOutPattern struct {
    handlers []Handler
}

// Execute รัน handlers พร้อมกัน
func (f *FanOutPattern) Execute(ctx context.Context, event Event) ([]Response, []error) {
    type result struct {
        resp *Response
        err  error
        idx  int
    }
    
    resultCh := make(chan result, len(f.handlers))
    
    for i, h := range f.handlers {
        go func(idx int, handler Handler) {
            resp, err := handler.Handle(ctx, event)
            resultCh <- result{resp: resp, err: err, idx: idx}
        }(i, h)
    }
    
    responses := make([]Response, len(f.handlers))
    errors := make([]error, len(f.handlers))
    
    for i := 0; i < len(f.handlers); i++ {
        r := <-resultCh
        if r.resp != nil {
            responses[r.idx] = *r.resp
        }
        errors[r.idx] = r.err
    }
    
    return responses, errors
}

func main() {
    ctx := context.Background()
    
    event := Event{
        ID:        "evt-001",
        Source:    "order-service",
        Type:      "order.created",
        Data:      map[string]interface{}{"order_id": "ord-123", "amount": 500.00},
        Timestamp: time.Now(),
    }
    
    // Function Chain pattern
    fmt.Println("=== Function Chain Pattern ===")
    chain := &FunctionChain{}
    chain.Add(&ValidateHandler{})
    chain.Add(&ProcessHandler{})
    chain.Add(&NotifyHandler{})
    
    resp, err := chain.Execute(ctx, event)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Result: %s\n\n", resp.Body)
    
    // Fan-Out pattern
    fmt.Println("=== Fan-Out Pattern ===")
    fanOut := &FanOutPattern{
        handlers: []Handler{
            &NotifyHandler{},
            &ProcessHandler{},
            &ValidateHandler{},
        },
    }
    
    responses, errs := fanOut.Execute(ctx, event)
    for i, r := range responses {
        if errs[i] != nil {
            fmt.Printf("Handler %d error: %v\n", i, errs[i])
        } else {
            fmt.Printf("Handler %d response: %s\n", i, r.Body)
        }
    }
}
```

---

## 3. AWS Lambda with Go

```go
// lambda/handler.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
)

// LambdaRequest แสดง API Gateway Lambda request
type LambdaRequest struct {
    HTTPMethod            string            `json:"httpMethod"`
    Path                  string            `json:"path"`
    QueryStringParameters map[string]string `json:"queryStringParameters"`
    Headers               map[string]string `json:"headers"`
    Body                  string            `json:"body"`
    IsBase64Encoded       bool              `json:"isBase64Encoded"`
}

// LambdaResponse แสดง API Gateway Lambda response
type LambdaResponse struct {
    StatusCode      int               `json:"statusCode"`
    Headers         map[string]string `json:"headers"`
    Body            string            `json:"body"`
    IsBase64Encoded bool              `json:"isBase64Encoded"`
}

// LambdaContext context ของ Lambda function
type LambdaContext struct {
    FunctionName    string
    FunctionVersion string
    MemoryLimitInMB int
    RemainingTimeMs int
}

// LambdaHandler interface
type LambdaHandler interface {
    Handle(ctx context.Context, req LambdaRequest) (LambdaResponse, error)
}

// HealthCheckHandler handler สำหรับ health check
type HealthCheckHandler struct{}

func (h *HealthCheckHandler) Handle(ctx context.Context, req LambdaRequest) (LambdaResponse, error) {
    body := map[string]interface{}{
        "status":    "healthy",
        "timestamp": time.Now().Unix(),
    }
    
    data, _ := json.Marshal(body)
    return LambdaResponse{
        StatusCode: 200,
        Headers: map[string]string{
            "Content-Type": "application/json",
        },
        Body: string(data),
    }, nil
}

// UserHandler handler สำหรับ user operations
type UserHandler struct {
    users map[string]User
}

// User แสดง user
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// NewUserHandler สร้าง user handler
func NewUserHandler() *UserHandler {
    return &UserHandler{
        users: map[string]User{
            "1": {ID: "1", Name: "สมชาย", Email: "somchai@example.com"},
            "2": {ID: "2", Name: "สมหญิง", Email: "somying@example.com"},
        },
    }
}

func (h *UserHandler) Handle(ctx context.Context, req LambdaRequest) (LambdaResponse, error) {
    switch req.HTTPMethod {
    case "GET":
        return h.handleGet(req)
    case "POST":
        return h.handleCreate(req)
    default:
        return LambdaResponse{
            StatusCode: 405,
            Body:       `{"error": "Method Not Allowed"}`,
        }, nil
    }
}

func (h *UserHandler) handleGet(req LambdaRequest) (LambdaResponse, error) {
    id, ok := req.QueryStringParameters["id"]
    
    if !ok {
        // คืน all users
        users := make([]User, 0, len(h.users))
        for _, u := range h.users {
            users = append(users, u)
        }
        data, _ := json.Marshal(users)
        return LambdaResponse{
            StatusCode: 200,
            Headers:    map[string]string{"Content-Type": "application/json"},
            Body:       string(data),
        }, nil
    }
    
    user, exists := h.users[id]
    if !exists {
        return LambdaResponse{
            StatusCode: 404,
            Body:       `{"error": "User not found"}`,
        }, nil
    }
    
    data, _ := json.Marshal(user)
    return LambdaResponse{
        StatusCode: 200,
        Headers:    map[string]string{"Content-Type": "application/json"},
        Body:       string(data),
    }, nil
}

func (h *UserHandler) handleCreate(req LambdaRequest) (LambdaResponse, error) {
    var user User
    if err := json.Unmarshal([]byte(req.Body), &user); err != nil {
        return LambdaResponse{
            StatusCode: 400,
            Body:       `{"error": "Invalid request body"}`,
        }, nil
    }
    
    user.ID = fmt.Sprintf("%d", len(h.users)+1)
    h.users[user.ID] = user
    
    data, _ := json.Marshal(user)
    return LambdaResponse{
        StatusCode: 201,
        Headers:    map[string]string{"Content-Type": "application/json"},
        Body:       string(data),
    }, nil
}

// Router สำหรับ Lambda
type LambdaRouter struct {
    routes map[string]LambdaHandler
}

// NewLambdaRouter สร้าง router ใหม่
func NewLambdaRouter() *LambdaRouter {
    return &LambdaRouter{
        routes: make(map[string]LambdaHandler),
    }
}

// Add เพิ่ม route
func (r *LambdaRouter) Add(path string, handler LambdaHandler) {
    r.routes[path] = handler
}

// Handle จัดการ request
func (r *LambdaRouter) Handle(ctx context.Context, req LambdaRequest) (LambdaResponse, error) {
    handler, ok := r.routes[req.Path]
    if !ok {
        return LambdaResponse{
            StatusCode: 404,
            Body:       `{"error": "Not Found"}`,
        }, nil
    }
    return handler.Handle(ctx, req)
}

// Middleware สำหรับ Lambda
type CORSMiddleware struct {
    next LambdaHandler
}

func (m *CORSMiddleware) Handle(ctx context.Context, req LambdaRequest) (LambdaResponse, error) {
    resp, err := m.next.Handle(ctx, req)
    if err != nil {
        return resp, err
    }
    
    if resp.Headers == nil {
        resp.Headers = make(map[string]string)
    }
    resp.Headers["Access-Control-Allow-Origin"] = "*"
    resp.Headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
    
    return resp, nil
}

// Lambda Cold Start Optimization
type WarmPool struct {
    initialized bool
    db          string // จำลอง database connection
    cache       string // จำลอง cache connection
}

var warmPool *WarmPool

// init รัน ณ cold start (ครั้งแรก)
func init() {
    log.Println("Lambda cold start: initializing connections...")
    
    warmPool = &WarmPool{
        db:    os.Getenv("DATABASE_URL"),
        cache: os.Getenv("REDIS_URL"),
    }
    
    // จำลอง connection initialization
    time.Sleep(10 * time.Millisecond)
    warmPool.initialized = true
    
    log.Println("Connections initialized")
}

// LambdaEntry จุดเริ่มต้นของ Lambda function
func LambdaEntry(ctx context.Context, req LambdaRequest) (LambdaResponse, error) {
    router := NewLambdaRouter()
    router.Add("/health", &HealthCheckHandler{})
    router.Add("/users", NewUserHandler())
    
    // Wrap with CORS middleware
    handler := &CORSMiddleware{next: router}
    
    return handler.Handle(ctx, req)
}

func main() {
    // ทดสอบ Lambda handler locally
    ctx := context.Background()
    
    testRequests := []LambdaRequest{
        {HTTPMethod: "GET", Path: "/health"},
        {HTTPMethod: "GET", Path: "/users"},
        {HTTPMethod: "GET", Path: "/users", QueryStringParameters: map[string]string{"id": "1"}},
        {HTTPMethod: "POST", Path: "/users", Body: `{"name": "สมศักดิ์", "email": "somsak@example.com"}`},
        {HTTPMethod: "GET", Path: "/not-found"},
    }
    
    fmt.Println("=== Lambda Handler Tests ===\n")
    
    for _, req := range testRequests {
        resp, err := LambdaEntry(ctx, req)
        if err != nil {
            fmt.Printf("%s %s -> ERROR: %v\n", req.HTTPMethod, req.Path, err)
        } else {
            fmt.Printf("%s %s -> %d: %s\n", req.HTTPMethod, req.Path, resp.StatusCode, resp.Body)
        }
    }
    
    _ = http.StatusOK
}
```

---

## 4. GitOps with Go

```go
// gitops/reconciler.go
package main

import (
    "fmt"
    "time"
    "crypto/sha256"
)

// GitOpsState แสดง state ใน GitOps
type GitOpsState struct {
    Repository  string
    Branch      string
    CommitHash  string
    LastSynced  time.Time
    Resources   map[string]ResourceState
}

// ResourceState แสดง state ของ resource
type ResourceState struct {
    Kind       string
    Name       string
    Namespace  string
    Hash       string // hash ของ spec
    Applied    bool
    LastApplied time.Time
}

// GitRepository จำลอง Git repository
type GitRepository struct {
    URL        string
    Branch     string
    CommitHash string
    Files      map[string]string // path -> content
}

// GitOpsReconciler reconcile state จาก Git
type GitOpsReconciler struct {
    repo         *GitRepository
    currentState *GitOpsState
    interval     time.Duration
}

// NewGitOpsReconciler สร้าง reconciler ใหม่
func NewGitOpsReconciler(repo *GitRepository, interval time.Duration) *GitOpsReconciler {
    return &GitOpsReconciler{
        repo:     repo,
        interval: interval,
        currentState: &GitOpsState{
            Resources: make(map[string]ResourceState),
        },
    }
}

// hashContent คำนวณ hash ของ content
func hashContent(content string) string {
    h := sha256.New()
    h.Write([]byte(content))
    return fmt.Sprintf("%x", h.Sum(nil))[:8]
}

// Reconcile reconcile state
func (r *GitOpsReconciler) Reconcile() {
    fmt.Printf("[GitOps] Reconciling from repo: %s (branch: %s)\n",
        r.repo.URL, r.repo.Branch)
    
    // จำลองการอ่าน manifests จาก git
    for path, content := range r.repo.Files {
        hash := hashContent(content)
        key := path
        
        if existing, exists := r.currentState.Resources[key]; exists {
            if existing.Hash == hash {
                fmt.Printf("[GitOps] No changes: %s\n", path)
                continue
            }
            fmt.Printf("[GitOps] Updating: %s (hash: %s -> %s)\n", path, existing.Hash, hash)
        } else {
            fmt.Printf("[GitOps] Creating: %s\n", path)
        }
        
        // Apply resource
        r.applyResource(path, content, hash)
    }
    
    // ลบ resources ที่ถูกลบจาก git
    for key := range r.currentState.Resources {
        if _, exists := r.repo.Files[key]; !exists {
            fmt.Printf("[GitOps] Deleting: %s\n", key)
            delete(r.currentState.Resources, key)
        }
    }
    
    r.currentState.LastSynced = time.Now()
    r.currentState.CommitHash = r.repo.CommitHash
}

func (r *GitOpsReconciler) applyResource(path, content, hash string) {
    r.currentState.Resources[path] = ResourceState{
        Name:        path,
        Hash:        hash,
        Applied:     true,
        LastApplied: time.Now(),
    }
    
    // จำลอง kubectl apply
    fmt.Printf("[GitOps]   Applied: %s\n", path)
}

// Start เริ่ม reconcile loop
func (r *GitOpsReconciler) Start() {
    ticker := time.NewTicker(r.interval)
    defer ticker.Stop()
    
    // รัน reconcile ครั้งแรก
    r.Reconcile()
    
    for range ticker.C {
        r.Reconcile()
    }
}

func main() {
    // จำลอง Git repository
    repo := &GitRepository{
        URL:        "https://github.com/myorg/k8s-manifests",
        Branch:     "main",
        CommitHash: "abc123",
        Files: map[string]string{
            "apps/web-server/deployment.yaml": `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  image: nginx:1.21`,
            "apps/web-server/service.yaml": `
apiVersion: v1
kind: Service
metadata:
  name: web-server
spec:
  type: ClusterIP
  port: 80`,
            "apps/api/deployment.yaml": `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  image: myapi:v1.0`,
        },
    }
    
    reconciler := NewGitOpsReconciler(repo, 30*time.Second)
    
    fmt.Println("=== GitOps Reconciler Demo ===\n")
    
    // Initial reconcile
    reconciler.Reconcile()
    
    fmt.Println("\n--- Simulating git push (updated replicas) ---")
    repo.Files["apps/web-server/deployment.yaml"] = `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 5
  image: nginx:1.22`
    repo.CommitHash = "def456"
    
    reconciler.Reconcile()
    
    fmt.Println("\n--- Simulating resource deletion ---")
    delete(repo.Files, "apps/api/deployment.yaml")
    
    reconciler.Reconcile()
    
    fmt.Printf("\nLast synced: %v\n", reconciler.currentState.LastSynced.Format(time.RFC3339))
    fmt.Printf("Commit: %s\n", reconciler.currentState.CommitHash)
    fmt.Printf("Resources managed: %d\n", len(reconciler.currentState.Resources))
}
```

---

## 5. Infrastructure as Code

```go
// iac/terraform.go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

// TerraformResource แสดง Terraform resource
type TerraformResource struct {
    Type       string                 `json:"type"`
    Name       string                 `json:"name"`
    Properties map[string]interface{} `json:"properties"`
}

// TerraformModule แสดง Terraform module
type TerraformModule struct {
    Name      string
    Provider  string
    Resources []TerraformResource
    Variables map[string]TFVariable
    Outputs   map[string]TFOutput
}

// TFVariable Terraform variable
type TFVariable struct {
    Type        string
    Description string
    Default     interface{}
}

// TFOutput Terraform output
type TFOutput struct {
    Value       string
    Description string
    Sensitive   bool
}

// InfrastructureBuilder สร้าง infrastructure configurations
type InfrastructureBuilder struct {
    modules []*TerraformModule
}

// NewInfrastructureBuilder สร้าง builder ใหม่
func NewInfrastructureBuilder() *InfrastructureBuilder {
    return &InfrastructureBuilder{
        modules: make([]*TerraformModule, 0),
    }
}

// AddModule เพิ่ม module
func (b *InfrastructureBuilder) AddModule(module *TerraformModule) {
    b.modules = append(b.modules, module)
}

// GenerateHCL สร้าง HCL configuration
func (b *InfrastructureBuilder) GenerateHCL() string {
    var sb strings.Builder
    
    for _, module := range b.modules {
        sb.WriteString(fmt.Sprintf("# Module: %s\n\n", module.Name))
        
        // Variables
        for name, v := range module.Variables {
            sb.WriteString(fmt.Sprintf("variable \"%s\" {\n", name))
            if v.Type != "" {
                sb.WriteString(fmt.Sprintf("  type = %s\n", v.Type))
            }
            if v.Description != "" {
                sb.WriteString(fmt.Sprintf("  description = \"%s\"\n", v.Description))
            }
            if v.Default != nil {
                data, _ := json.Marshal(v.Default)
                sb.WriteString(fmt.Sprintf("  default = %s\n", string(data)))
            }
            sb.WriteString("}\n\n")
        }
        
        // Resources
        for _, r := range module.Resources {
            sb.WriteString(fmt.Sprintf("resource \"%s\" \"%s\" {\n", r.Type, r.Name))
            for k, v := range r.Properties {
                data, _ := json.Marshal(v)
                sb.WriteString(fmt.Sprintf("  %s = %s\n", k, string(data)))
            }
            sb.WriteString("}\n\n")
        }
        
        // Outputs
        for name, o := range module.Outputs {
            sb.WriteString(fmt.Sprintf("output \"%s\" {\n", name))
            sb.WriteString(fmt.Sprintf("  value = %s\n", o.Value))
            if o.Description != "" {
                sb.WriteString(fmt.Sprintf("  description = \"%s\"\n", o.Description))
            }
            if o.Sensitive {
                sb.WriteString("  sensitive = true\n")
            }
            sb.WriteString("}\n\n")
        }
    }
    
    return sb.String()
}

// EKSClusterModule สร้าง EKS cluster module
func EKSClusterModule(clusterName, region string, nodeCount int) *TerraformModule {
    return &TerraformModule{
        Name:     "eks-cluster",
        Provider: "aws",
        Variables: map[string]TFVariable{
            "cluster_name": {
                Type:        "string",
                Description: "Name of the EKS cluster",
                Default:     clusterName,
            },
            "region": {
                Type:        "string",
                Description: "AWS region",
                Default:     region,
            },
            "node_count": {
                Type:        "number",
                Description: "Number of worker nodes",
                Default:     nodeCount,
            },
        },
        Resources: []TerraformResource{
            {
                Type: "aws_eks_cluster",
                Name: "main",
                Properties: map[string]interface{}{
                    "name": "${var.cluster_name}",
                    "role_arn": "${aws_iam_role.eks_cluster.arn}",
                },
            },
            {
                Type: "aws_eks_node_group",
                Name: "main",
                Properties: map[string]interface{}{
                    "cluster_name": "${aws_eks_cluster.main.name}",
                    "node_group_name": "main",
                    "scaling_config": map[string]interface{}{
                        "desired_size": "${var.node_count}",
                        "max_size":     "${var.node_count * 2}",
                        "min_size":     1,
                    },
                },
            },
        },
        Outputs: map[string]TFOutput{
            "cluster_endpoint": {
                Value:       "${aws_eks_cluster.main.endpoint}",
                Description: "EKS cluster endpoint",
            },
            "cluster_name": {
                Value:       "${aws_eks_cluster.main.name}",
                Description: "EKS cluster name",
            },
        },
    }
}

func main() {
    builder := NewInfrastructureBuilder()
    
    // เพิ่ม EKS cluster module
    builder.AddModule(EKSClusterModule("my-cluster", "ap-southeast-1", 3))
    
    // Generate HCL
    hcl := builder.GenerateHCL()
    
    fmt.Println("=== Generated Terraform HCL ===")
    fmt.Println(hcl)
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cloud-Native Principles** - 12-factor app methodology
2. **Serverless Patterns** - Function chains, fan-out
3. **AWS Lambda** - Handler implementation, cold start optimization
4. **GitOps** - Reconciliation loop, declarative configuration
5. **Infrastructure as Code** - Terraform generation

### Key Takeaways

- **Stateless services** ทำให้ scale ง่ายขึ้น
- **Graceful shutdown** สำคัญสำหรับ zero-downtime deployments
- **GitOps** = Git เป็น single source of truth
- **IaC** ทำให้ infrastructure reproducible และ auditable
