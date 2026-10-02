# Part 75: Production Readiness

## เป้าหมายการเรียนรู้
- Production Readiness Checklist
- Graceful Shutdown
- Health Endpoints
- Blue-Green Deployments
- Canary Releases
- Feature Flags
- Rollback Strategies
- Post-mortem Analysis
- SRE Practices

---

## 1. Graceful Shutdown

```go
// shutdown/graceful.go
package shutdown

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type GracefulServer struct {
    server          *http.Server
    shutdownTimeout time.Duration
    onShutdown      []func(ctx context.Context) error
    mu              sync.Mutex
}

func NewGracefulServer(addr string, handler http.Handler) *GracefulServer {
    return &GracefulServer{
        server: &http.Server{
            Addr:              addr,
            Handler:           handler,
            ReadTimeout:       5 * time.Second,
            WriteTimeout:      10 * time.Second,
            IdleTimeout:       120 * time.Second,
            ReadHeaderTimeout: 2 * time.Second,
        },
        shutdownTimeout: 30 * time.Second,
    }
}

// Register shutdown hooks
func (s *GracefulServer) OnShutdown(fn func(ctx context.Context) error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.onShutdown = append(s.onShutdown, fn)
}

func (s *GracefulServer) Start() error {
    // Channel สำหรับรับ OS signals
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)
    
    // Start server ใน goroutine
    serverErr := make(chan error, 1)
    go func() {
        log.Printf("Server starting on %s", s.server.Addr)
        if err := s.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            serverErr <- err
        }
    }()
    
    // รอ signal หรือ error
    select {
    case sig := <-sigChan:
        log.Printf("Received signal: %v, starting graceful shutdown...", sig)
        return s.Shutdown()
    case err := <-serverErr:
        return fmt.Errorf("server error: %w", err)
    }
}

func (s *GracefulServer) Shutdown() error {
    ctx, cancel := context.WithTimeout(context.Background(), s.shutdownTimeout)
    defer cancel()
    
    // 1. หยุดรับ request ใหม่
    if err := s.server.Shutdown(ctx); err != nil {
        log.Printf("HTTP server shutdown error: %v", err)
    }
    log.Println("HTTP server stopped accepting new requests")
    
    // 2. รัน shutdown hooks (ปิด DB connections, Kafka, etc.)
    var wg sync.WaitGroup
    errors := make(chan error, len(s.onShutdown))
    
    for _, fn := range s.onShutdown {
        wg.Add(1)
        go func(f func(ctx context.Context) error) {
            defer wg.Done()
            if err := f(ctx); err != nil {
                errors <- err
            }
        }(fn)
    }
    
    // รอทุก hooks เสร็จ
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()
    
    select {
    case <-ctx.Done():
        log.Println("Shutdown timeout reached")
    case <-done:
        log.Println("All shutdown hooks completed")
    }
    
    close(errors)
    
    var errs []error
    for err := range errors {
        errs = append(errs, err)
    }
    
    if len(errs) > 0 {
        return fmt.Errorf("shutdown errors: %v", errs)
    }
    
    log.Println("Graceful shutdown complete")
    return nil
}

// ตัวอย่างการใช้งาน
func main() {
    db, _ := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    redisClient := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    kafkaProducer, _ := NewKafkaProducer([]string{"localhost:9092"}, "events")
    
    mux := http.NewServeMux()
    mux.HandleFunc("/", apiHandler)
    
    server := NewGracefulServer(":8080", mux)
    
    // Register cleanup hooks
    server.OnShutdown(func(ctx context.Context) error {
        log.Println("Closing database...")
        return db.Close()
    })
    
    server.OnShutdown(func(ctx context.Context) error {
        log.Println("Closing Redis...")
        return redisClient.Close()
    })
    
    server.OnShutdown(func(ctx context.Context) error {
        log.Println("Closing Kafka producer...")
        return kafkaProducer.producer.Close()
    })
    
    if err := server.Start(); err != nil {
        log.Fatalf("Server error: %v", err)
    }
}
```

---

## 2. Health Endpoints

```go
// health/endpoints.go
package health

import (
    "context"
    "database/sql"
    "encoding/json"
    "net/http"
    "sync"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type HealthStatus string

const (
    StatusHealthy   HealthStatus = "healthy"
    StatusUnhealthy HealthStatus = "unhealthy"
    StatusDegraded  HealthStatus = "degraded"
)

type CheckResult struct {
    Name    string       `json:"name"`
    Status  HealthStatus `json:"status"`
    Message string       `json:"message,omitempty"`
    Latency string       `json:"latency"`
}

type HealthResponse struct {
    Status  HealthStatus  `json:"status"`
    Checks  []CheckResult `json:"checks"`
    Version string        `json:"version"`
    Uptime  string        `json:"uptime"`
}

type HealthChecker struct {
    checks    []Check
    startTime time.Time
    version   string
}

type Check interface {
    Name() string
    Check(ctx context.Context) CheckResult
}

func NewHealthChecker(version string, checks ...Check) *HealthChecker {
    return &HealthChecker{
        checks:    checks,
        startTime: time.Now(),
        version:   version,
    }
}

// Liveness probe: ตรวจสอบว่า process ยังทำงานอยู่
func (hc *HealthChecker) Liveness(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "alive"})
}

// Readiness probe: ตรวจสอบว่า service พร้อมรับ traffic
func (hc *HealthChecker) Readiness(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    var (
        wg      sync.WaitGroup
        mu      sync.Mutex
        checks  []CheckResult
        overall = StatusHealthy
    )
    
    for _, check := range hc.checks {
        wg.Add(1)
        go func(c Check) {
            defer wg.Done()
            result := c.Check(ctx)
            mu.Lock()
            checks = append(checks, result)
            if result.Status == StatusUnhealthy {
                overall = StatusUnhealthy
            } else if result.Status == StatusDegraded && overall == StatusHealthy {
                overall = StatusDegraded
            }
            mu.Unlock()
        }(check)
    }
    
    wg.Wait()
    
    resp := HealthResponse{
        Status:  overall,
        Checks:  checks,
        Version: hc.version,
        Uptime:  time.Since(hc.startTime).Round(time.Second).String(),
    }
    
    statusCode := http.StatusOK
    if overall == StatusUnhealthy {
        statusCode = http.StatusServiceUnavailable
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(resp)
}

// Database health check
type DatabaseCheck struct {
    db   *sql.DB
    name string
}

func NewDatabaseCheck(db *sql.DB) *DatabaseCheck {
    return &DatabaseCheck{db: db, name: "database"}
}

func (c *DatabaseCheck) Name() string { return c.name }

func (c *DatabaseCheck) Check(ctx context.Context) CheckResult {
    start := time.Now()
    
    err := c.db.PingContext(ctx)
    latency := time.Since(start)
    
    if err != nil {
        return CheckResult{
            Name:    c.name,
            Status:  StatusUnhealthy,
            Message: err.Error(),
            Latency: latency.String(),
        }
    }
    
    status := StatusHealthy
    if latency > 100*time.Millisecond {
        status = StatusDegraded
    }
    
    return CheckResult{
        Name:    c.name,
        Status:  status,
        Latency: latency.String(),
    }
}

// Redis health check
type RedisCheck struct {
    client *redis.Client
}

func NewRedisCheck(client *redis.Client) *RedisCheck {
    return &RedisCheck{client: client}
}

func (c *RedisCheck) Name() string { return "redis" }

func (c *RedisCheck) Check(ctx context.Context) CheckResult {
    start := time.Now()
    
    err := c.client.Ping(ctx).Err()
    latency := time.Since(start)
    
    if err != nil {
        return CheckResult{
            Name:    "redis",
            Status:  StatusUnhealthy,
            Message: err.Error(),
            Latency: latency.String(),
        }
    }
    
    return CheckResult{
        Name:    "redis",
        Status:  StatusHealthy,
        Latency: latency.String(),
    }
}

// External Service check
type HTTPCheck struct {
    name    string
    url     string
    timeout time.Duration
}

func (c *HTTPCheck) Name() string { return c.name }

func (c *HTTPCheck) Check(ctx context.Context) CheckResult {
    checkCtx, cancel := context.WithTimeout(ctx, c.timeout)
    defer cancel()
    
    start := time.Now()
    
    req, _ := http.NewRequestWithContext(checkCtx, "GET", c.url, nil)
    resp, err := http.DefaultClient.Do(req)
    latency := time.Since(start)
    
    if err != nil {
        return CheckResult{
            Name:    c.name,
            Status:  StatusUnhealthy,
            Message: err.Error(),
            Latency: latency.String(),
        }
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 500 {
        return CheckResult{
            Name:    c.name,
            Status:  StatusUnhealthy,
            Message: fmt.Sprintf("HTTP %d", resp.StatusCode),
            Latency: latency.String(),
        }
    }
    
    return CheckResult{
        Name:    c.name,
        Status:  StatusHealthy,
        Latency: latency.String(),
    }
}
```

---

## 3. Feature Flags

```go
// featureflags/flags.go
package featureflags

import (
    "context"
    "hash/fnv"
    "sync"
    "time"
)

type FlagValue struct {
    Enabled    bool
    Percentage int // 0-100, ใช้สำหรับ gradual rollout
    Users      []string // specific user IDs ที่ enable
    Groups     []string // specific groups ที่ enable
}

type FlagStore interface {
    Get(ctx context.Context, name string) (*FlagValue, error)
    Set(ctx context.Context, name string, value *FlagValue) error
}

type FeatureFlags struct {
    store    FlagStore
    cache    map[string]*cachedFlag
    mu       sync.RWMutex
    cacheTTL time.Duration
}

type cachedFlag struct {
    value     *FlagValue
    expiresAt time.Time
}

func NewFeatureFlags(store FlagStore) *FeatureFlags {
    return &FeatureFlags{
        store:    store,
        cache:    make(map[string]*cachedFlag),
        cacheTTL: 30 * time.Second,
    }
}

// IsEnabled ตรวจสอบว่า feature เปิดอยู่หรือไม่สำหรับ user/request
func (ff *FeatureFlags) IsEnabled(ctx context.Context, flagName string, userID string) bool {
    flag := ff.getFlag(ctx, flagName)
    if flag == nil {
        return false // Default: disabled
    }
    
    // Global disable
    if !flag.Enabled {
        return false
    }
    
    // Specific user override
    for _, uid := range flag.Users {
        if uid == userID {
            return true
        }
    }
    
    // Percentage rollout
    if flag.Percentage < 100 {
        return ff.isInPercentage(userID, flagName, flag.Percentage)
    }
    
    return true
}

// Gradual rollout โดยใช้ consistent hashing
func (ff *FeatureFlags) isInPercentage(userID, flagName string, percentage int) bool {
    h := fnv.New32a()
    h.Write([]byte(flagName + ":" + userID))
    hash := h.Sum32()
    
    bucket := int(hash % 100)
    return bucket < percentage
}

func (ff *FeatureFlags) getFlag(ctx context.Context, name string) *FlagValue {
    // ตรวจสอบ cache
    ff.mu.RLock()
    if cached, ok := ff.cache[name]; ok && time.Now().Before(cached.expiresAt) {
        ff.mu.RUnlock()
        return cached.value
    }
    ff.mu.RUnlock()
    
    // Fetch จาก store
    value, err := ff.store.Get(ctx, name)
    if err != nil {
        return nil
    }
    
    ff.mu.Lock()
    ff.cache[name] = &cachedFlag{
        value:     value,
        expiresAt: time.Now().Add(ff.cacheTTL),
    }
    ff.mu.Unlock()
    
    return value
}

// Redis Flag Store
type RedisFlagStore struct {
    client *redis.Client
    prefix string
}

func (s *RedisFlagStore) Get(ctx context.Context, name string) (*FlagValue, error) {
    data, err := s.client.Get(ctx, s.prefix+name).Bytes()
    if err == redis.Nil {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    
    var flag FlagValue
    if err := json.Unmarshal(data, &flag); err != nil {
        return nil, err
    }
    
    return &flag, nil
}

func (s *RedisFlagStore) Set(ctx context.Context, name string, value *FlagValue) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    return s.client.Set(ctx, s.prefix+name, data, 0).Err()
}

// Middleware ที่ inject feature flags ใน context
func FeatureFlagMiddleware(ff *FeatureFlags) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), "feature_flags", ff)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func GetFlags(ctx context.Context) *FeatureFlags {
    ff, _ := ctx.Value("feature_flags").(*FeatureFlags)
    return ff
}

// ตัวอย่างการใช้ใน handler
func productHandler(w http.ResponseWriter, r *http.Request) {
    userID := r.Context().Value("user_id").(string)
    flags := GetFlags(r.Context())
    
    if flags.IsEnabled(r.Context(), "new-product-ui", userID) {
        // แสดง UI ใหม่
        renderNewProductUI(w, r)
    } else {
        // แสดง UI เก่า
        renderOldProductUI(w, r)
    }
}
```

---

## 4. Blue-Green Deployment

```yaml
# deployment/blue-green.yaml
# Blue Deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10

---
# Green Deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0

---
# Service - point to blue initially
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # เปลี่ยนเป็น green เมื่อ deploy
  ports:
  - port: 80
    targetPort: 8080
```

```go
// deployment/blue_green.go
package deployment

import (
    "context"
    "fmt"
    "log"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type BlueGreenDeployer struct {
    k8s       *kubernetes.Clientset
    namespace string
    service   string
}

func (d *BlueGreenDeployer) Deploy(ctx context.Context, newVersion, image string) error {
    // 1. Deploy green
    log.Printf("Deploying green version: %s", image)
    if err := d.createGreenDeployment(ctx, image); err != nil {
        return fmt.Errorf("failed to create green deployment: %w", err)
    }
    
    // 2. รอ green ready
    log.Println("Waiting for green deployment to be ready...")
    if err := d.waitForReady(ctx, "myapp-green", 5*time.Minute); err != nil {
        d.deleteGreenDeployment(ctx)
        return fmt.Errorf("green deployment not ready: %w", err)
    }
    
    // 3. Run smoke tests บน green
    log.Println("Running smoke tests on green...")
    if err := d.runSmokeTests(ctx, "green"); err != nil {
        d.deleteGreenDeployment(ctx)
        return fmt.Errorf("smoke tests failed: %w", err)
    }
    
    // 4. Switch traffic ไป green
    log.Println("Switching traffic to green...")
    if err := d.switchTraffic(ctx, "green"); err != nil {
        return fmt.Errorf("traffic switch failed: %w", err)
    }
    
    // 5. Monitor ระยะเวลาหนึ่ง
    log.Println("Monitoring green deployment...")
    if err := d.monitor(ctx, 5*time.Minute); err != nil {
        log.Printf("Issues detected, rolling back: %v", err)
        d.switchTraffic(ctx, "blue")
        return fmt.Errorf("rollback triggered: %w", err)
    }
    
    // 6. Delete blue
    log.Println("Deleting blue deployment...")
    d.deleteBlueDeployment(ctx)
    
    // 7. Rename green to blue (สำหรับ next deployment)
    d.renameGreenToBlue(ctx)
    
    log.Println("Blue-green deployment complete!")
    return nil
}

func (d *BlueGreenDeployer) switchTraffic(ctx context.Context, version string) error {
    svc, err := d.k8s.CoreV1().Services(d.namespace).Get(ctx, d.service, metav1.GetOptions{})
    if err != nil {
        return err
    }
    
    svc.Spec.Selector["version"] = version
    
    _, err = d.k8s.CoreV1().Services(d.namespace).Update(ctx, svc, metav1.UpdateOptions{})
    return err
}

func (d *BlueGreenDeployer) monitor(ctx context.Context, duration time.Duration) error {
    deadline := time.Now().Add(duration)
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // ตรวจสอบ error rate
            errorRate := d.getErrorRate("green")
            if errorRate > 0.05 { // 5% error rate
                return fmt.Errorf("error rate too high: %.2f%%", errorRate*100)
            }
            
            if time.Now().After(deadline) {
                return nil
            }
            
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

---

## 5. Canary Releases

```yaml
# canary/virtual-service.yaml - Istio canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10   # เริ่มที่ 10%
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
```

```go
// canary/controller.go
package canary

import (
    "context"
    "fmt"
    "log"
    "time"
)

type CanaryController struct {
    deployer     Deployer
    metrics      MetricsClient
    trafficSplit TrafficSplitter
}

type CanaryConfig struct {
    InitialPercentage int           // เริ่มต้น (เช่น 5%)
    MaxPercentage     int           // สูงสุด (เช่น 100%)
    StepSize          int           // เพิ่มทีละ (เช่น 10%)
    StepInterval      time.Duration // ความถี่ที่เพิ่ม
    ErrorThreshold    float64       // ถ้า error rate เกินนี้ = rollback
    LatencyThreshold  float64       // ถ้า p99 latency เกินนี้ = rollback
}

func (c *CanaryController) Deploy(ctx context.Context, image string, cfg CanaryConfig) error {
    log.Printf("Starting canary deployment: %s", image)
    
    // Deploy canary version
    if err := c.deployer.DeployCanary(ctx, image); err != nil {
        return err
    }
    
    currentPercentage := cfg.InitialPercentage
    
    for currentPercentage <= cfg.MaxPercentage {
        log.Printf("Setting canary traffic to %d%%", currentPercentage)
        
        if err := c.trafficSplit.SetCanaryWeight(ctx, currentPercentage); err != nil {
            c.rollback(ctx)
            return fmt.Errorf("traffic split failed: %w", err)
        }
        
        // Monitor
        time.Sleep(cfg.StepInterval)
        
        metrics, err := c.metrics.GetCanaryMetrics(ctx)
        if err != nil {
            log.Printf("Warning: failed to get metrics: %v", err)
        } else {
            if metrics.ErrorRate > cfg.ErrorThreshold {
                log.Printf("Error rate too high (%.2f%%), rolling back", metrics.ErrorRate*100)
                c.rollback(ctx)
                return fmt.Errorf("canary rollback: error rate %.2f%%", metrics.ErrorRate*100)
            }
            
            if metrics.P99Latency > cfg.LatencyThreshold {
                log.Printf("Latency too high (%.2fms), rolling back", metrics.P99Latency)
                c.rollback(ctx)
                return fmt.Errorf("canary rollback: latency %.2fms", metrics.P99Latency)
            }
        }
        
        if currentPercentage == 100 {
            break
        }
        
        currentPercentage = min(currentPercentage+cfg.StepSize, 100)
    }
    
    // Full rollout สำเร็จ
    log.Println("Canary deployment complete - promoting to stable")
    return c.deployer.PromoteCanary(ctx)
}

func (c *CanaryController) rollback(ctx context.Context) {
    log.Println("Rolling back canary deployment...")
    
    c.trafficSplit.SetCanaryWeight(ctx, 0)
    c.deployer.DeleteCanary(ctx)
    
    log.Println("Rollback complete")
}
```

---

## 6. SRE Practices

```go
// sre/slo.go
package sre

import (
    "context"
    "fmt"
    "time"
)

// SLO (Service Level Objective) tracking
type SLO struct {
    Name       string
    Target     float64 // 99.9% = 0.999
    Window     time.Duration
    Calculator SLOCalculator
}

type SLOCalculator interface {
    Calculate(ctx context.Context, start, end time.Time) (float64, error)
}

type ErrorBudget struct {
    slo        *SLO
    prometheus PrometheusClient
}

func (eb *ErrorBudget) Remaining(ctx context.Context) (time.Duration, error) {
    now := time.Now()
    start := now.Add(-eb.slo.Window)
    
    current, err := eb.slo.Calculator.Calculate(ctx, start, now)
    if err != nil {
        return 0, err
    }
    
    // Error budget consumed
    consumed := 1 - current
    allowed := 1 - eb.slo.Target
    
    if consumed > allowed {
        return 0, nil // Error budget exhausted
    }
    
    remainingFraction := (allowed - consumed) / allowed
    remaining := time.Duration(float64(eb.slo.Window) * remainingFraction)
    
    return remaining, nil
}

// Alerting based on error budget burn rate
type BurnRateAlert struct {
    slo        *SLO
    threshold  float64 // burn rate ที่ alert (เช่น 2 = 2x normal rate)
    window     time.Duration
}

func (a *BurnRateAlert) ShouldAlert(ctx context.Context) (bool, string, error) {
    burnRate := a.calculateBurnRate(ctx)
    
    if burnRate > a.threshold {
        return true, fmt.Sprintf(
            "High error budget burn rate: %.1fx (threshold: %.1fx)", 
            burnRate, a.threshold,
        ), nil
    }
    
    return false, "", nil
}

// SLI Measurement
type AvailabilitySLI struct {
    successQuery string
    totalQuery   string
    prometheus   PrometheusClient
}

func (s *AvailabilitySLI) Calculate(ctx context.Context, start, end time.Time) (float64, error) {
    successes, err := s.prometheus.QueryRange(ctx, s.successQuery, start, end)
    if err != nil {
        return 0, err
    }
    
    total, err := s.prometheus.QueryRange(ctx, s.totalQuery, start, end)
    if err != nil {
        return 0, err
    }
    
    if total == 0 {
        return 1.0, nil // ไม่มี traffic = 100% available
    }
    
    return successes / total, nil
}

// Runbook automation
type Runbook struct {
    Name     string
    Trigger  string
    Steps    []RunbookStep
}

type RunbookStep struct {
    Name        string
    Description string
    Action      func(ctx context.Context) error
    Rollback    func(ctx context.Context) error
}

func (rb *Runbook) Execute(ctx context.Context) error {
    log.Printf("Executing runbook: %s", rb.Name)
    
    var completedSteps []*RunbookStep
    
    for i, step := range rb.Steps {
        log.Printf("Step %d: %s", i+1, step.Name)
        
        if err := step.Action(ctx); err != nil {
            log.Printf("Step %d failed: %v", i+1, err)
            
            // Rollback completed steps ย้อนกลับ
            for j := len(completedSteps) - 1; j >= 0; j-- {
                if completedSteps[j].Rollback != nil {
                    if rbErr := completedSteps[j].Rollback(ctx); rbErr != nil {
                        log.Printf("Rollback step %d failed: %v", j+1, rbErr)
                    }
                }
            }
            
            return fmt.Errorf("runbook failed at step %d: %w", i+1, err)
        }
        
        completedSteps = append(completedSteps, &rb.Steps[i])
    }
    
    log.Printf("Runbook %s completed successfully", rb.Name)
    return nil
}
```

---

## 7. Production Readiness Checklist

```go
// checklist/production.go
package checklist

// Production Readiness Checklist สำหรับ Go services

/*
## Reliability
[ ] Graceful shutdown implemented
[ ] Health endpoints /live, /ready, /health
[ ] Retry with exponential backoff
[ ] Circuit breakers on external calls
[ ] Timeouts on all external calls
[ ] Connection pooling configured
[ ] Resource limits set (CPU, memory)

## Observability
[ ] Structured logging (JSON)
[ ] Distributed tracing (OpenTelemetry)
[ ] Metrics (Prometheus)
[ ] Alerts configured (Grafana)
[ ] Log aggregation (ELK/Loki)
[ ] Error tracking (Sentry)

## Security
[ ] TLS 1.3 on all endpoints
[ ] Secrets in Vault/K8s secrets
[ ] Input validation on all endpoints
[ ] Rate limiting implemented
[ ] CORS configured properly
[ ] Security headers set
[ ] Container runs as non-root
[ ] Read-only filesystem

## Performance
[ ] Load tested
[ ] Database queries optimized
[ ] N+1 queries eliminated
[ ] Caching implemented
[ ] Connection pools tuned
[ ] Memory profiling done

## Deployment
[ ] Docker image optimized (multi-stage)
[ ] Kubernetes resources defined
[ ] Horizontal Pod Autoscaler configured
[ ] Pod Disruption Budget set
[ ] Liveness/Readiness probes
[ ] Rolling update strategy

## Data
[ ] Database migrations automated
[ ] Backup strategy defined
[ ] Restore tested
[ ] Data retention policy

## Documentation
[ ] API documentation (OpenAPI)
[ ] Runbook for common issues
[ ] Architecture diagram
[ ] On-call rotation

## Testing
[ ] Unit tests >80% coverage
[ ] Integration tests
[ ] Load tests
[ ] Chaos engineering
*/

type ProductionCheck struct {
    Name       string
    Category   string
    CheckFn    func() (bool, string)
    Critical   bool
}

var checks = []ProductionCheck{
    {
        Name:     "Graceful shutdown",
        Category: "Reliability",
        CheckFn: func() (bool, string) {
            // ตรวจสอบว่ามี SIGTERM handler
            return true, "OK"
        },
        Critical: true,
    },
    {
        Name:     "Health endpoints",
        Category: "Reliability",
        CheckFn: func() (bool, string) {
            resp, err := http.Get("http://localhost:8080/health")
            if err != nil {
                return false, err.Error()
            }
            return resp.StatusCode == 200, fmt.Sprintf("status: %d", resp.StatusCode)
        },
        Critical: true,
    },
    {
        Name:     "Metrics endpoint",
        Category: "Observability",
        CheckFn: func() (bool, string) {
            resp, err := http.Get("http://localhost:8080/metrics")
            if err != nil {
                return false, err.Error()
            }
            return resp.StatusCode == 200, "Prometheus metrics available"
        },
        Critical: false,
    },
}

func RunChecklist() {
    fmt.Println("Running Production Readiness Checklist")
    fmt.Println("=========================================")
    
    passed := 0
    failed := 0
    
    for _, check := range checks {
        ok, msg := check.CheckFn()
        
        status := "✓"
        if !ok {
            status = "✗"
            failed++
            if check.Critical {
                fmt.Printf("[CRITICAL] %s [%s]: %s\n", status, check.Name, msg)
            } else {
                fmt.Printf("[WARNING]  %s [%s]: %s\n", status, check.Name, msg)
            }
        } else {
            passed++
            fmt.Printf("[OK]       %s [%s]: %s\n", status, check.Name, msg)
        }
    }
    
    fmt.Printf("\nResults: %d passed, %d failed\n", passed, failed)
    
    if failed > 0 {
        fmt.Println("NOT PRODUCTION READY")
    } else {
        fmt.Println("PRODUCTION READY!")
    }
}
```

---

## 8. Post-mortem Analysis

```markdown
# Post-mortem Template

## Incident: [Title]

### Summary
Brief description ของ incident ที่เกิดขึ้น

### Timeline
| Time | Event |
|------|-------|
| 10:30 | Alert fired: p99 latency > 1s |
| 10:35 | On-call engineer paged |
| 10:42 | Issue identified: database connection pool exhausted |
| 11:00 | Temporary fix: increased pool size |
| 11:30 | Root cause identified |
| 12:00 | Permanent fix deployed |
| 12:15 | Incident resolved |

### Impact
- Duration: 90 minutes
- Error rate: 15%
- Affected users: ~5,000
- Revenue impact: ~$500

### Root Cause
Connection pool ตั้งค่า MaxOpenConns=10 แต่ traffic เพิ่มขึ้น 10x จาก marketing campaign

### Contributing Factors
1. ไม่มี monitoring สำหรับ connection pool metrics
2. Load test ไม่ครอบคลุม traffic spike scenario
3. Alert threshold สูงเกินไป (ควร alert เร็วกว่านี้)

### What Went Well
- On-call engineer respond เร็ว
- Rollback mechanism ทำงานได้
- Communication ภายในทีมดี

### Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|---------|
| เพิ่ม monitoring สำหรับ DB pool | DevOps | 2024-01-15 | High |
| Update load test scenarios | QA | 2024-01-20 | High |
| ปรับ alert thresholds | SRE | 2024-01-12 | Medium |
| Auto-scaling ตาม DB connections | DevOps | 2024-02-01 | Medium |

### Lessons Learned
1. Connection pool exhaustion เกิดได้เร็วมาก - ต้องมี buffer เพียงพอ
2. Marketing campaigns ควร notify DevOps ล่วงหน้า
3. ต้องมี runbook สำหรับ common issues
```

---

## Workshop: Production-ready Service

```go
// workshop/production_service/main.go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log/slog"
    "net/http"
    _ "net/http/pprof"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "github.com/prometheus/client_golang/prometheus/promhttp"
    _ "github.com/lib/pq"
    "github.com/redis/go-redis/v9"
)

func main() {
    // Structured logging
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)
    
    // Load config
    cfg := loadConfig()
    
    // Connect to databases
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        slog.Error("Failed to open database", "error", err)
        os.Exit(1)
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)
    
    redisClient := redis.NewClient(&redis.Options{
        Addr:     cfg.RedisAddr,
        Password: cfg.RedisPassword,
        DB:       0,
    })
    
    // Health checks
    healthChecker := health.NewHealthChecker(
        cfg.Version,
        health.NewDatabaseCheck(db),
        health.NewRedisCheck(redisClient),
    )
    
    // Feature flags
    flagStore := &RedisFlagStore{client: redisClient, prefix: "flags:"}
    flags := featureflags.NewFeatureFlags(flagStore)
    
    // Setup routes
    mux := http.NewServeMux()
    
    // Business endpoints
    mux.HandleFunc("/api/users", usersHandler(db, redisClient, flags))
    mux.HandleFunc("/api/orders", ordersHandler(db, flags))
    
    // Health endpoints
    mux.HandleFunc("/live", healthChecker.Liveness)
    mux.HandleFunc("/ready", healthChecker.Readiness)
    mux.HandleFunc("/health", healthChecker.Readiness)
    
    // Metrics
    mux.Handle("/metrics", promhttp.Handler())
    
    // Debug (ควร restrict ใน production)
    if cfg.Debug {
        mux.HandleFunc("/debug/pprof/", http.DefaultServeMux.ServeHTTP)
    }
    
    // Middleware
    handler := chain(mux,
        requestIDMiddleware,
        loggingMiddleware,
        recoveryMiddleware,
        metricsMiddleware,
        corsMiddleware,
    )
    
    // Graceful server
    server := NewGracefulServer(":"+cfg.Port, handler)
    
    // Register cleanup
    server.OnShutdown(func(ctx context.Context) error {
        return db.Close()
    })
    server.OnShutdown(func(ctx context.Context) error {
        return redisClient.Close()
    })
    
    slog.Info("Service starting", 
        "port", cfg.Port, 
        "version", cfg.Version,
        "environment", cfg.Environment,
    )
    
    if err := server.Start(); err != nil {
        slog.Error("Service stopped", "error", err)
        os.Exit(1)
    }
}

// Middleware chain
func chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Recovery middleware
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                slog.Error("Panic recovered",
                    "error", err,
                    "path", r.URL.Path,
                    "method", r.Method,
                )
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Request ID middleware
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := r.Header.Get("X-Request-ID")
        if reqID == "" {
            reqID = generateID()
        }
        
        w.Header().Set("X-Request-ID", reqID)
        ctx := context.WithValue(r.Context(), "request_id", reqID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

type Config struct {
    Port        string
    Version     string
    Environment string
    Debug       bool
    DatabaseURL string
    RedisAddr   string
    RedisPassword string
}

func loadConfig() Config {
    return Config{
        Port:        getEnv("PORT", "8080"),
        Version:     getEnv("VERSION", "development"),
        Environment: getEnv("ENVIRONMENT", "development"),
        Debug:       getEnv("DEBUG", "false") == "true",
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/myapp"),
        RedisAddr:   getEnv("REDIS_ADDR", "localhost:6379"),
    }
}

func getEnv(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}
```

---

## สรุป

### Production Readiness Pyramid

```
        ┌─────────────────────┐
        │   Business Goals    │ ← SLO, SLA, Error Budget
        ├─────────────────────┤
        │   Observability     │ ← Metrics, Tracing, Logging
        ├─────────────────────┤
        │   Reliability       │ ← Health checks, Circuit breakers
        ├─────────────────────┤
        │   Security          │ ← TLS, Auth, Secrets
        ├─────────────────────┤
        │   Infrastructure    │ ← K8s, Auto-scaling, Backups
        └─────────────────────┘
```

### สิ่งที่สำคัญที่สุด
1. **Know your SLOs** - รู้ว่า service ต้องทำงานได้แค่ไหน
2. **Automate everything** - deployment, testing, rollback
3. **Monitor everything** - ถ้าไม่ได้ monitor = ไม่รู้ว่ามีปัญหา
4. **Plan for failure** - assume ทุกอย่างจะพัง
5. **Practice regularly** - chaos engineering, runbook drills
6. **Blameless culture** - post-mortem ไม่ใช่การโทษคน

---

*จบ Part 75: Production Readiness*

---

## ภาพรวม Phase 3 (Parts 66-75)

| Part | หัวข้อ | ความสำคัญ |
|------|--------|----------|
| 66 | Service Mesh | Infrastructure |
| 67 | Advanced Database | Data Layer |
| 68 | Full-Text Search | Features |
| 69 | Security Advanced | Must-have |
| 70 | OAuth 2.0 / OIDC | Identity |
| 71 | Advanced Caching | Performance |
| 72 | Performance | Scale |
| 73 | Real-time | Features |
| 74 | Data Pipeline | Data |
| 75 | Production Ready | All |

ยินดีด้วยที่ผ่านมาถึง Part 75! คุณได้เรียนรู้ Go อย่างครบถ้วนตั้งแต่พื้นฐานจนถึง production-grade systems
