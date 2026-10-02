# Part 81: Serverless Go

## เป้าหมายของบทเรียน
- Serverless overview
- AWS Lambda with Go
- GCP Cloud Functions with Go
- Azure Functions with Go
- Cold starts optimization
- Serverless patterns
- Cost optimization

---

## 1. Serverless Overview

Serverless คือ execution model ที่ cloud provider จัดการ server infrastructure เอง เราเพียงแค่ deploy code

### ข้อดี
- ไม่ต้องจัดการ servers
- Auto-scaling อัตโนมัติ
- จ่ายตามการใช้งานจริง
- High availability built-in

### ข้อเสีย
- Cold start latency
- Execution time limits
- Stateless
- Vendor lock-in

```go
// serverless/concepts.go
package main

import (
    "context"
    "fmt"
    "time"
)

// ServerlessFunction interface
type ServerlessFunction interface {
    Invoke(ctx context.Context, payload []byte) ([]byte, error)
    Name() string
    MemoryMB() int
    TimeoutSec() int
}

// FunctionMetrics metrics ของ function
type FunctionMetrics struct {
    Invocations    int64
    Errors         int64
    ColdStarts     int64
    AvgDurationMs  float64
    MaxDurationMs  float64
    BilledDuration float64
}

// LambdaFunction จำลอง AWS Lambda function
type LambdaFunction struct {
    name       string
    memoryMB   int
    timeoutSec int
    handler    func(ctx context.Context, payload []byte) ([]byte, error)
    metrics    FunctionMetrics
    initialized bool
    initTime    time.Time
}

// NewLambdaFunction สร้าง function ใหม่
func NewLambdaFunction(name string, memMB, timeoutSec int, 
    handler func(ctx context.Context, payload []byte) ([]byte, error)) *LambdaFunction {
    return &LambdaFunction{
        name:       name,
        memoryMB:   memMB,
        timeoutSec: timeoutSec,
        handler:    handler,
    }
}

func (f *LambdaFunction) Name() string     { return f.name }
func (f *LambdaFunction) MemoryMB() int    { return f.memoryMB }
func (f *LambdaFunction) TimeoutSec() int  { return f.timeoutSec }

// Invoke รัน function
func (f *LambdaFunction) Invoke(ctx context.Context, payload []byte) ([]byte, error) {
    start := time.Now()
    
    // จำลอง cold start
    if !f.initialized {
        f.metrics.ColdStarts++
        coldStartTime := 200 * time.Millisecond
        fmt.Printf("[Lambda] Cold start for %s (initializing...)\n", f.name)
        time.Sleep(coldStartTime)
        f.initialized = true
        f.initTime = time.Now()
    }
    
    f.metrics.Invocations++
    
    // รัน handler
    timeoutCtx, cancel := context.WithTimeout(ctx, time.Duration(f.timeoutSec)*time.Second)
    defer cancel()
    
    result, err := f.handler(timeoutCtx, payload)
    
    duration := time.Since(start)
    
    if err != nil {
        f.metrics.Errors++
    }
    
    // อัปเดต metrics
    ms := float64(duration.Milliseconds())
    f.metrics.AvgDurationMs = (f.metrics.AvgDurationMs*float64(f.metrics.Invocations-1) + ms) / float64(f.metrics.Invocations)
    if ms > f.metrics.MaxDurationMs {
        f.metrics.MaxDurationMs = ms
    }
    // Billed in 1ms increments
    f.metrics.BilledDuration += ms
    
    return result, err
}

// GetMetrics คืน metrics
func (f *LambdaFunction) GetMetrics() FunctionMetrics {
    return f.metrics
}

// CostCalculator คำนวณ cost
type CostCalculator struct {
    pricePerGB float64 // price per GB-second
    pricePerReq float64 // price per request
    freeGB     float64  // free tier GB-seconds
    freeReqs   int64    // free tier requests
}

// NewAWSLambdaCostCalculator สร้าง calculator สำหรับ AWS Lambda
func NewAWSLambdaCostCalculator() *CostCalculator {
    return &CostCalculator{
        pricePerGB:  0.0000166667, // $0.0000166667 per GB-second
        pricePerReq: 0.0000002,    // $0.0000002 per request
        freeGB:      400000,       // 400,000 GB-seconds per month
        freeReqs:    1000000,      // 1 million requests per month
    }
}

// Calculate คำนวณ monthly cost
func (c *CostCalculator) Calculate(memoryMB int, invocations, avgDurationMs int64) float64 {
    gbSecs := float64(memoryMB) / 1024 * float64(avgDurationMs) / 1000 * float64(invocations)
    
    billableGB := gbSecs - c.freeGB
    if billableGB < 0 {
        billableGB = 0
    }
    
    billableReqs := invocations - c.freeReqs
    if billableReqs < 0 {
        billableReqs = 0
    }
    
    computeCost := billableGB * c.pricePerGB
    requestCost := float64(billableReqs) * c.pricePerReq
    
    return computeCost + requestCost
}

func main() {
    // สร้าง Lambda functions
    apiHandler := NewLambdaFunction("api-handler", 512, 30, func(ctx context.Context, payload []byte) ([]byte, error) {
        time.Sleep(10 * time.Millisecond)
        return []byte(`{"status": "ok", "data": []}`), nil
    })
    
    workerHandler := NewLambdaFunction("worker", 1024, 60, func(ctx context.Context, payload []byte) ([]byte, error) {
        time.Sleep(100 * time.Millisecond)
        return []byte(`{"processed": true}`), nil
    })
    
    ctx := context.Background()
    
    fmt.Println("=== Serverless Lambda Demo ===\n")
    
    // Cold start invocation
    apiHandler.Invoke(ctx, []byte(`{"path": "/users"}`))
    
    // Warm invocations
    for i := 0; i < 5; i++ {
        result, err := apiHandler.Invoke(ctx, []byte(fmt.Sprintf(`{"path": "/item/%d"}`, i)))
        if err != nil {
            fmt.Printf("Error: %v\n", err)
        } else {
            fmt.Printf("Invocation %d: %s\n", i+1, string(result))
        }
    }
    
    // Worker cold start
    workerHandler.Invoke(ctx, []byte(`{"job": "process-report"}`))
    
    // Metrics
    apiMetrics := apiHandler.GetMetrics()
    fmt.Printf("\n=== API Handler Metrics ===\n")
    fmt.Printf("Invocations: %d\n", apiMetrics.Invocations)
    fmt.Printf("Errors: %d\n", apiMetrics.Errors)
    fmt.Printf("Cold Starts: %d\n", apiMetrics.ColdStarts)
    fmt.Printf("Avg Duration: %.2f ms\n", apiMetrics.AvgDurationMs)
    
    // Cost calculation
    calculator := NewAWSLambdaCostCalculator()
    monthlyCost := calculator.Calculate(512, 1000000, 50) // 1M invocations, 50ms avg
    fmt.Printf("\n=== Monthly Cost Estimate ===\n")
    fmt.Printf("API Handler (1M invocations, 512MB, 50ms avg): $%.4f\n", monthlyCost)
}
```

---

## 2. GCP Cloud Functions

```go
// gcp/function.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

// CloudFunctionRequest แสดง GCP Cloud Function HTTP request
type CloudFunctionRequest struct {
    Method  string
    Path    string
    Headers map[string][]string
    Body    []byte
}

// CloudFunctionResponse แสดง response
type CloudFunctionResponse struct {
    StatusCode int
    Headers    map[string]string
    Body       interface{}
}

// GCPFunction interface
type GCPFunction interface {
    ServeHTTP(w http.ResponseWriter, r *http.Request)
}

// UserService GCP Cloud Function สำหรับ user service
type UserService struct {
    users []User
}

// User แสดง user
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func NewUserService() *UserService {
    return &UserService{
        users: []User{
            {ID: 1, Name: "สมชาย", Email: "somchai@example.com"},
            {ID: 2, Name: "สมหญิง", Email: "somying@example.com"},
        },
    }
}

func (s *UserService) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    switch r.Method {
    case "GET":
        json.NewEncoder(w).Encode(s.users)
    case "POST":
        var user User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        user.ID = len(s.users) + 1
        s.users = append(s.users, user)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(user)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

// CloudEventHandler จัดการ Cloud Events (Pub/Sub trigger)
type CloudEvent struct {
    ID          string                 `json:"id"`
    Source      string                 `json:"source"`
    SpecVersion string                 `json:"specversion"`
    Type        string                 `json:"type"`
    Data        map[string]interface{} `json:"data"`
    Time        time.Time              `json:"time"`
}

// PubSubMessage แสดง Pub/Sub message
type PubSubMessage struct {
    Data        []byte            `json:"data"`
    MessageID   string            `json:"messageId"`
    Attributes  map[string]string `json:"attributes"`
    PublishTime time.Time         `json:"publishTime"`
}

// ProcessPubSubMessage ประมวลผล Pub/Sub message
func ProcessPubSubMessage(ctx context.Context, msg PubSubMessage) error {
    fmt.Printf("[CloudFunction] Processing message: %s\n", msg.MessageID)
    fmt.Printf("[CloudFunction] Data: %s\n", string(msg.Data))
    
    // ประมวลผล...
    time.Sleep(50 * time.Millisecond)
    
    return nil
}

// ScheduledFunction function ที่รันตาม schedule
func ScheduledFunction(ctx context.Context) error {
    fmt.Printf("[CloudFunction] Running scheduled task at %v\n", time.Now())
    
    // งานที่ต้องทำตาม schedule...
    fmt.Println("[CloudFunction] Cleaning up old records...")
    time.Sleep(100 * time.Millisecond)
    fmt.Println("[CloudFunction] Cleanup complete")
    
    return nil
}

// ConnectionPool จำลอง connection pool สำหรับ warm instances
var (
    dbPool  *ConnectionPool
    poolErr error
)

type ConnectionPool struct {
    dsn string
}

// เริ่มต้น connection pool ณ cold start
func init() {
    fmt.Println("[CloudFunction] Initializing connection pool...")
    dbPool = &ConnectionPool{dsn: "postgres://localhost/db"}
    fmt.Println("[CloudFunction] Connection pool ready")
}

func main() {
    ctx := context.Background()
    
    // ทดสอบ Pub/Sub handler
    msg := PubSubMessage{
        MessageID: "msg-001",
        Data:      []byte(`{"event": "user.created", "user_id": 123}`),
        Attributes: map[string]string{
            "source": "user-service",
        },
        PublishTime: time.Now(),
    }
    
    fmt.Println("=== GCP Cloud Functions Demo ===\n")
    
    if err := ProcessPubSubMessage(ctx, msg); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
    
    // ทดสอบ scheduled function
    ScheduledFunction(ctx)
    
    // ทดสอบ HTTP function
    svc := NewUserService()
    fmt.Printf("\nUser service ready: %T\n", svc)
    
    _ = dbPool
}
```

---

## 3. Cold Start Optimization

```go
// coldstart/optimization.go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// ColdStartOptimizer เทคนิคการ optimize cold start
type ColdStartOptimizer struct {
    resources map[string]interface{}
    mu        sync.RWMutex
    initOnce  sync.Once
}

// NewColdStartOptimizer สร้าง optimizer ใหม่
func NewColdStartOptimizer() *ColdStartOptimizer {
    return &ColdStartOptimizer{
        resources: make(map[string]interface{}),
    }
}

// Technique 1: Lazy initialization - initialize ณ first request
func (o *ColdStartOptimizer) LazyInit(name string, initFn func() (interface{}, error)) (interface{}, error) {
    o.mu.RLock()
    if res, ok := o.resources[name]; ok {
        o.mu.RUnlock()
        return res, nil
    }
    o.mu.RUnlock()
    
    o.mu.Lock()
    defer o.mu.Unlock()
    
    // Double-check
    if res, ok := o.resources[name]; ok {
        return res, nil
    }
    
    res, err := initFn()
    if err != nil {
        return nil, err
    }
    o.resources[name] = res
    return res, nil
}

// Technique 2: Parallel initialization
func (o *ColdStartOptimizer) ParallelInit(inits map[string]func() (interface{}, error)) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(inits))
    
    for name, initFn := range inits {
        wg.Add(1)
        go func(n string, fn func() (interface{}, error)) {
            defer wg.Done()
            res, err := fn()
            if err != nil {
                errCh <- fmt.Errorf("init %s: %w", n, err)
                return
            }
            o.mu.Lock()
            o.resources[n] = res
            o.mu.Unlock()
            fmt.Printf("[ColdStart] Initialized: %s\n", n)
        }(name, initFn)
    }
    
    wg.Wait()
    close(errCh)
    
    for err := range errCh {
        return err
    }
    return nil
}

// Technique 3: Provisioned Concurrency - keep instances warm
type WarmInstancePool struct {
    instances   []*WarmInstance
    mu          sync.Mutex
    minWarm     int
    maxInstances int
    factory     func() *WarmInstance
}

type WarmInstance struct {
    ID          string
    CreatedAt   time.Time
    LastUsed    time.Time
    Available   bool
}

func NewWarmInstancePool(min, max int, factory func() *WarmInstance) *WarmInstancePool {
    pool := &WarmInstancePool{
        instances:    make([]*WarmInstance, 0),
        minWarm:     min,
        maxInstances: max,
        factory:     factory,
    }
    
    // Pre-warm instances
    for i := 0; i < min; i++ {
        pool.instances = append(pool.instances, factory())
    }
    
    return pool
}

// Acquire ขอใช้ instance
func (p *WarmInstancePool) Acquire() *WarmInstance {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    for _, inst := range p.instances {
        if inst.Available {
            inst.Available = false
            inst.LastUsed = time.Now()
            return inst
        }
    }
    
    // สร้าง instance ใหม่
    if len(p.instances) < p.maxInstances {
        inst := p.factory()
        inst.Available = false
        p.instances = append(p.instances, inst)
        return inst
    }
    
    return nil
}

// Release คืน instance
func (p *WarmInstancePool) Release(inst *WarmInstance) {
    p.mu.Lock()
    defer p.mu.Unlock()
    inst.Available = true
}

// Technique 4: Reduce binary size สำหรับ faster load
// ใช้: go build -ldflags="-s -w" -trimpath
// ใช้ TinyGo สำหรับ WebAssembly
// หลีกเลี่ยง imports ที่ไม่จำเป็น

// BinaryOptimizer แสดงวิธี optimize binary
type BinaryOptimizer struct{}

func (b *BinaryOptimizer) Suggestions() []string {
    return []string{
        "1. ใช้ -ldflags='-s -w' เพื่อ strip debug symbols",
        "2. ใช้ -trimpath เพื่อลบ absolute paths",
        "3. ใช้ upx หรือ compress tool เพื่อ compress binary",
        "4. หลีกเลี่ยง CGO ถ้าไม่จำเป็น (CGO_ENABLED=0)",
        "5. ใช้ lazy loading สำหรับ large dependencies",
        "6. แยก initialization code ออกจาก hot path",
    }
}

func main() {
    fmt.Println("=== Cold Start Optimization Demo ===\n")
    
    optimizer := NewColdStartOptimizer()
    
    // Parallel initialization
    start := time.Now()
    err := optimizer.ParallelInit(map[string]func() (interface{}, error){
        "database": func() (interface{}, error) {
            time.Sleep(50 * time.Millisecond) // จำลอง db connection
            return "db-connection", nil
        },
        "cache": func() (interface{}, error) {
            time.Sleep(30 * time.Millisecond) // จำลอง cache connection
            return "cache-connection", nil
        },
        "config": func() (interface{}, error) {
            time.Sleep(10 * time.Millisecond) // จำลอง config load
            return map[string]string{"env": "production"}, nil
        },
    })
    
    if err != nil {
        fmt.Printf("Init error: %v\n", err)
        return
    }
    fmt.Printf("Parallel init completed in: %v\n\n", time.Since(start))
    
    // Lazy initialization
    fmt.Println("Lazy initialization:")
    for i := 0; i < 3; i++ {
        start = time.Now()
        res, _ := optimizer.LazyInit("expensive-resource", func() (interface{}, error) {
            fmt.Println("  Initializing expensive resource...")
            time.Sleep(100 * time.Millisecond)
            return "expensive-result", nil
        })
        fmt.Printf("  Attempt %d: %v (%v)\n", i+1, res, time.Since(start))
    }
    
    // Warm instance pool
    fmt.Println("\nWarm Instance Pool:")
    instanceCount := 0
    pool := NewWarmInstancePool(2, 5, func() *WarmInstance {
        instanceCount++
        return &WarmInstance{
            ID:        fmt.Sprintf("inst-%d", instanceCount),
            CreatedAt: time.Now(),
            Available: true,
        }
    })
    
    inst := pool.Acquire()
    fmt.Printf("Acquired: %s\n", inst.ID)
    pool.Release(inst)
    
    // Binary optimization suggestions
    bo := &BinaryOptimizer{}
    fmt.Println("\nBinary Optimization Suggestions:")
    for _, s := range bo.Suggestions() {
        fmt.Printf("  %s\n", s)
    }
    
    _ = context.Background()
}
```

---

## 4. Serverless Cost Optimization

```go
// cost/optimizer.go
package main

import (
    "fmt"
    "math"
    "time"
)

// ExecutionProfile โปรไฟล์การทำงานของ function
type ExecutionProfile struct {
    Name           string
    InvocationsPerMonth int64
    AvgDurationMs  float64
    MemoryMB       int
    ColdStartRatio float64 // เปอร์เซ็นต์ที่เป็น cold start
}

// CostModel cost model
type CostModel struct {
    PricePerGBSecond float64
    PricePerRequest  float64
    FreeGBSeconds    float64
    FreeRequests     int64
}

// AWSLambdaCost คำนวณ cost
func AWSLambdaCost(profile ExecutionProfile) float64 {
    model := CostModel{
        PricePerGBSecond: 0.0000166667,
        PricePerRequest:  0.0000002,
        FreeGBSeconds:    400000,
        FreeRequests:     1000000,
    }
    
    gbSec := float64(profile.MemoryMB) / 1024 * profile.AvgDurationMs / 1000 * float64(profile.InvocationsPerMonth)
    
    billableGB := math.Max(0, gbSec-model.FreeGBSeconds)
    billableReqs := math.Max(0, float64(profile.InvocationsPerMonth-model.FreeRequests))
    
    return billableGB*model.PricePerGBSecond + billableReqs*model.PricePerRequest
}

// OptimizationRecommendation คำแนะนำการ optimize
type OptimizationRecommendation struct {
    Issue       string
    Solution    string
    Savings     float64
    Complexity  string
}

// Analyze วิเคราะห์และแนะนำการ optimize
func Analyze(profile ExecutionProfile) []OptimizationRecommendation {
    recommendations := make([]OptimizationRecommendation, 0)
    
    currentCost := AWSLambdaCost(profile)
    
    // ตรวจสอบ memory
    if profile.MemoryMB > 512 && profile.AvgDurationMs < 100 {
        newProfile := profile
        newProfile.MemoryMB = 256
        newCost := AWSLambdaCost(newProfile)
        savings := currentCost - newCost
        
        recommendations = append(recommendations, OptimizationRecommendation{
            Issue:      fmt.Sprintf("Memory %dMB อาจสูงเกินไปสำหรับ duration %.0fms", profile.MemoryMB, profile.AvgDurationMs),
            Solution:   "ลด memory เหลือ 256MB",
            Savings:    savings,
            Complexity: "LOW",
        })
    }
    
    // ตรวจสอบ cold start ratio
    if profile.ColdStartRatio > 0.1 {
        recommendations = append(recommendations, OptimizationRecommendation{
            Issue:      fmt.Sprintf("Cold start ratio สูง: %.0f%%", profile.ColdStartRatio*100),
            Solution:   "ใช้ Provisioned Concurrency หรือ keep-warm mechanism",
            Savings:    currentCost * 0.1,
            Complexity: "MEDIUM",
        })
    }
    
    // ตรวจสอบ invocations
    if profile.InvocationsPerMonth > 10000000 {
        recommendations = append(recommendations, OptimizationRecommendation{
            Issue:      "Invocations สูงมาก อาจถูกกว่าด้วย EC2",
            Solution:   "พิจารณา migration ไปยัง container บน ECS/EKS",
            Savings:    currentCost * 0.3,
            Complexity: "HIGH",
        })
    }
    
    return recommendations
}

func main() {
    profiles := []ExecutionProfile{
        {
            Name:                "api-handler",
            InvocationsPerMonth: 5000000,
            AvgDurationMs:       50,
            MemoryMB:            512,
            ColdStartRatio:      0.05,
        },
        {
            Name:                "report-generator",
            InvocationsPerMonth: 100000,
            AvgDurationMs:       5000,
            MemoryMB:            1024,
            ColdStartRatio:      0.3,
        },
        {
            Name:                "image-processor",
            InvocationsPerMonth: 50000000,
            AvgDurationMs:       200,
            MemoryMB:            1024,
            ColdStartRatio:      0.02,
        },
    }
    
    fmt.Println("=== Serverless Cost Analysis ===\n")
    
    totalCost := 0.0
    
    for _, profile := range profiles {
        cost := AWSLambdaCost(profile)
        totalCost += cost
        
        fmt.Printf("Function: %s\n", profile.Name)
        fmt.Printf("  Invocations: %d/month\n", profile.InvocationsPerMonth)
        fmt.Printf("  Memory: %dMB\n", profile.MemoryMB)
        fmt.Printf("  Avg Duration: %.0fms\n", profile.AvgDurationMs)
        fmt.Printf("  Monthly Cost: $%.4f\n", cost)
        
        recommendations := Analyze(profile)
        if len(recommendations) > 0 {
            fmt.Println("  Recommendations:")
            for _, r := range recommendations {
                fmt.Printf("    ⚠ %s\n", r.Issue)
                fmt.Printf("      Solution: %s (saves ~$%.4f/month, complexity: %s)\n",
                    r.Solution, r.Savings, r.Complexity)
            }
        }
        fmt.Println()
    }
    
    fmt.Printf("Total Monthly Cost: $%.4f\n", totalCost)
    fmt.Printf("Annual Cost: $%.2f\n", totalCost*12)
    
    _ = time.Now()
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Serverless Overview** - ข้อดี ข้อเสีย และ use cases
2. **AWS Lambda** - Handler implementation, routing, middleware
3. **GCP Cloud Functions** - HTTP triggers, Pub/Sub triggers, scheduled
4. **Cold Start Optimization** - Parallel init, lazy init, warm pools
5. **Cost Optimization** - การวิเคราะห์และลด cost

### Key Takeaways

- **Keep binaries small**: ใช้ `-ldflags="-s -w"` และหลีกเลี่ยง dependencies ที่ไม่จำเป็น
- **Initialize outside handler**: ทำ database connections ใน `init()` หรือ global variables
- **Use context cancellation**: ตรวจสอบ `ctx.Done()` เสมอ
- **Monitor cold starts**: Cold starts มีผลต่อ latency และ user experience
- **Right-size memory**: เพิ่ม memory ไม่เสมอหมายถึง เร็วกว่า หรือ ถูกกว่า
