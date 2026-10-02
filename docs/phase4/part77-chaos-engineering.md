# Part 77: Chaos Engineering in Go

## เป้าหมายของบทเรียน
- เข้าใจหลักการ Chaos Engineering
- เรียนรู้จาก Netflix Chaos Monkey
- ใช้ Chaos Mesh สำหรับ Kubernetes
- Implement Go fault injection library
- ออกแบบ game days
- ทำ hypothesis-driven testing
- จัดการ blast radius

---

## 1. Chaos Engineering Principles

Chaos Engineering คือการทดลองกับระบบในสภาพแวดล้อม production เพื่อสร้างความมั่นใจในความสามารถในการทนต่อ turbulent conditions

### หลักการ 5 ข้อของ Chaos Engineering

1. **Build a Hypothesis around Steady State Behavior** - กำหนด normal behavior ก่อน
2. **Vary Real-world Events** - ทดสอบด้วย events ที่เกิดในชีวิตจริง
3. **Run Experiments in Production** - ทดสอบใน production (แต่ระวัง!)
4. **Automate Experiments** - ทำ automation เพื่อรัน continuous
5. **Minimize Blast Radius** - จำกัดผลกระทบ

```go
// chaos/principles.go
package chaos

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

// SteadyState กำหนด normal behavior ของระบบ
type SteadyState struct {
    Name        string
    Description string
    Probe       func() (float64, error) // คืน metric value
    Threshold   ThresholdConfig
}

// ThresholdConfig กำหนด acceptable range
type ThresholdConfig struct {
    Min float64
    Max float64
}

// IsWithinThreshold ตรวจสอบว่า value อยู่ใน threshold
func (t ThresholdConfig) IsWithinThreshold(value float64) bool {
    return value >= t.Min && value <= t.Max
}

// Hypothesis การทดสอบ
type Hypothesis struct {
    Name         string
    Description  string
    SteadyState  SteadyState
    Experiment   Experiment
    BlastRadius  BlastRadius
}

// Experiment การทดลอง chaos
type Experiment struct {
    Name     string
    Method   func(ctx context.Context) error
    Duration time.Duration
}

// BlastRadius กำหนด scope ของผลกระทบ
type BlastRadius struct {
    Scope      string  // "host", "service", "region"
    Percentage float64 // เปอร์เซ็นต์ที่ได้รับผลกระทบ
    MaxTargets int     // จำนวน targets สูงสุด
}

// ChaosResult ผลลัพธ์ของ chaos experiment
type ChaosResult struct {
    Hypothesis    string
    StartTime     time.Time
    EndTime       time.Time
    SteadyStateBefore float64
    SteadyStateAfter  float64
    Succeeded     bool
    Observations  []Observation
}

// Observation การสังเกตระหว่าง experiment
type Observation struct {
    Timestamp time.Time
    Metric    string
    Value     float64
    Note      string
}

// ChaosEngine engine สำหรับรัน experiments
type ChaosEngine struct {
    hypotheses []*Hypothesis
    results    []ChaosResult
    enabled    bool
    dryRun     bool
}

// NewChaosEngine สร้าง engine ใหม่
func NewChaosEngine(dryRun bool) *ChaosEngine {
    return &ChaosEngine{
        hypotheses: make([]*Hypothesis, 0),
        results:    make([]ChaosResult, 0),
        enabled:    true,
        dryRun:     dryRun,
    }
}

// AddHypothesis เพิ่ม hypothesis
func (ce *ChaosEngine) AddHypothesis(h *Hypothesis) {
    ce.hypotheses = append(ce.hypotheses, h)
}

// RunExperiment รัน chaos experiment
func (ce *ChaosEngine) RunExperiment(ctx context.Context, h *Hypothesis) (*ChaosResult, error) {
    if !ce.enabled {
        return nil, fmt.Errorf("chaos engine is disabled")
    }
    
    result := &ChaosResult{
        Hypothesis: h.Name,
        StartTime:  time.Now(),
        Observations: make([]Observation, 0),
    }
    
    // วัด steady state ก่อน
    fmt.Printf("[Chaos] Measuring steady state for: %s\n", h.SteadyState.Name)
    before, err := h.SteadyState.Probe()
    if err != nil {
        return nil, fmt.Errorf("failed to measure steady state: %w", err)
    }
    result.SteadyStateBefore = before
    
    if !h.SteadyState.Threshold.IsWithinThreshold(before) {
        return nil, fmt.Errorf("system not in steady state before experiment (value: %.2f)", before)
    }
    
    fmt.Printf("[Chaos] Steady state: %.2f (acceptable: %.2f-%.2f)\n", 
        before, h.SteadyState.Threshold.Min, h.SteadyState.Threshold.Max)
    
    if ce.dryRun {
        fmt.Printf("[Chaos] DRY RUN: Would run experiment '%s'\n", h.Experiment.Name)
        result.EndTime = time.Now()
        result.Succeeded = true
        return result, nil
    }
    
    // รัน experiment
    fmt.Printf("[Chaos] Starting experiment: %s\n", h.Experiment.Name)
    expCtx, cancel := context.WithTimeout(ctx, h.Experiment.Duration)
    defer cancel()
    
    // รัน monitoring goroutine
    go func() {
        ticker := time.NewTicker(1 * time.Second)
        defer ticker.Stop()
        
        for {
            select {
            case <-expCtx.Done():
                return
            case t := <-ticker.C:
                if value, err := h.SteadyState.Probe(); err == nil {
                    result.Observations = append(result.Observations, Observation{
                        Timestamp: t,
                        Metric:    h.SteadyState.Name,
                        Value:     value,
                    })
                }
            }
        }
    }()
    
    // รัน chaos
    if err := h.Experiment.Method(expCtx); err != nil && err != context.DeadlineExceeded {
        fmt.Printf("[Chaos] Experiment error: %v\n", err)
    }
    
    // รอให้ experiment เสร็จ
    <-expCtx.Done()
    
    // วัด steady state หลัง experiment
    fmt.Printf("[Chaos] Measuring steady state after experiment\n")
    
    // รอให้ระบบ stabilize
    time.Sleep(2 * time.Second)
    
    after, err := h.SteadyState.Probe()
    if err != nil {
        result.EndTime = time.Now()
        result.Succeeded = false
        return result, nil
    }
    
    result.SteadyStateAfter = after
    result.EndTime = time.Now()
    result.Succeeded = h.SteadyState.Threshold.IsWithinThreshold(after)
    
    if result.Succeeded {
        fmt.Printf("[Chaos] ✓ Hypothesis CONFIRMED: System maintained steady state\n")
    } else {
        fmt.Printf("[Chaos] ✗ Hypothesis REJECTED: System fell outside steady state (%.2f)\n", after)
    }
    
    ce.results = append(ce.results, *result)
    return result, nil
}

// PrintReport พิมพ์รายงาน
func (ce *ChaosEngine) PrintReport() {
    fmt.Printf("\n=== Chaos Engineering Report ===\n")
    fmt.Printf("Total experiments: %d\n\n", len(ce.results))
    
    for _, r := range ce.results {
        status := "✓ PASSED"
        if !r.Succeeded {
            status = "✗ FAILED"
        }
        fmt.Printf("Hypothesis: %s\n", r.Hypothesis)
        fmt.Printf("Status: %s\n", status)
        fmt.Printf("Duration: %v\n", r.EndTime.Sub(r.StartTime).Round(time.Second))
        fmt.Printf("Steady State: %.2f -> %.2f\n", r.SteadyStateBefore, r.SteadyStateAfter)
        fmt.Printf("Observations: %d data points\n\n", len(r.Observations))
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    ctx := context.Background()
    
    // สร้าง engine (dry run สำหรับ demo)
    engine := NewChaosEngine(false)
    
    // กำหนด steady state
    requestSuccessRate := 0.95 // 95% success rate
    
    // สร้าง hypothesis
    h := &Hypothesis{
        Name:        "Service survives single node failure",
        Description: "เมื่อ node หนึ่งใน cluster fail ระบบยังคง serve requests ได้ด้วย success rate >=90%",
        SteadyState: SteadyState{
            Name:        "request-success-rate",
            Description: "เปอร์เซ็นต์ของ requests ที่สำเร็จ",
            Probe: func() (float64, error) {
                // จำลอง metric probe
                return requestSuccessRate + rand.Float64()*0.02 - 0.01, nil
            },
            Threshold: ThresholdConfig{Min: 0.90, Max: 1.0},
        },
        Experiment: Experiment{
            Name:     "Kill random node",
            Duration: 5 * time.Second,
            Method: func(ctx context.Context) error {
                fmt.Println("[Chaos] Killing node-2...")
                // จำลอง: ลด success rate ชั่วคราว
                requestSuccessRate = 0.85
                
                <-ctx.Done()
                
                // Recovery
                requestSuccessRate = 0.95
                fmt.Println("[Chaos] Node-2 restarted")
                return nil
            },
        },
        BlastRadius: BlastRadius{
            Scope:      "service",
            Percentage: 33.3, // 1 จาก 3 nodes
            MaxTargets: 1,
        },
    }
    
    engine.AddHypothesis(h)
    
    result, err := engine.RunExperiment(ctx, h)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    _ = result
    engine.PrintReport()
}
```

---

## 2. Netflix Chaos Monkey Concepts

```go
// chaos/monkey.go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// ChaosMonkeyConfig การตั้งค่าของ Chaos Monkey
type ChaosMonkeyConfig struct {
    Enabled         bool
    Frequency       time.Duration // ความถี่ในการ attack
    KillProbability float64       // ความน่าจะเป็นที่จะ kill instance
    Schedule        string        // cron expression
    ExcludedGroups  []string      // groups ที่ไม่ kill
    MaxKillsPerHour int           // จำกัดจำนวน kills
}

// Instance แสดง service instance
type Instance struct {
    ID      string
    Group   string
    Host    string
    Port    int
    Running bool
    mu      sync.Mutex
}

// Kill จำลองการ kill instance
func (inst *Instance) Kill() {
    inst.mu.Lock()
    defer inst.mu.Unlock()
    inst.Running = false
    fmt.Printf("[ChaosMonkey] Killed instance %s in group %s\n", inst.ID, inst.Group)
}

// Restart จำลองการ restart instance
func (inst *Instance) Restart() {
    inst.mu.Lock()
    defer inst.mu.Unlock()
    inst.Running = true
    fmt.Printf("[ChaosMonkey] Restarted instance %s in group %s\n", inst.ID, inst.Group)
}

// ChaosMonkey จำลอง Netflix Chaos Monkey
type ChaosMonkey struct {
    mu        sync.Mutex
    config    ChaosMonkeyConfig
    instances []*Instance
    killCount int
    killHistory []KillRecord
}

// KillRecord บันทึกการ kill
type KillRecord struct {
    Timestamp  time.Time
    InstanceID string
    Group      string
}

// NewChaosMonkey สร้าง Chaos Monkey ใหม่
func NewChaosMonkey(config ChaosMonkeyConfig) *ChaosMonkey {
    return &ChaosMonkey{
        config:      config,
        instances:   make([]*Instance, 0),
        killHistory: make([]KillRecord, 0),
    }
}

// RegisterInstance ลงทะเบียน instance
func (cm *ChaosMonkey) RegisterInstance(inst *Instance) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    cm.instances = append(cm.instances, inst)
}

// isExcluded ตรวจสอบว่า group ถูก exclude หรือไม่
func (cm *ChaosMonkey) isExcluded(group string) bool {
    for _, excluded := range cm.config.ExcludedGroups {
        if excluded == group {
            return true
        }
    }
    return false
}

// getKillsInLastHour คืนจำนวน kills ใน 1 ชั่วโมงที่ผ่านมา
func (cm *ChaosMonkey) getKillsInLastHour() int {
    oneHourAgo := time.Now().Add(-1 * time.Hour)
    count := 0
    for _, record := range cm.killHistory {
        if record.Timestamp.After(oneHourAgo) {
            count++
        }
    }
    return count
}

// Attack เลือก instance แบบสุ่มและ kill
func (cm *ChaosMonkey) Attack() {
    if !cm.config.Enabled {
        return
    }
    
    cm.mu.Lock()
    defer cm.mu.Unlock()
    
    // ตรวจสอบ rate limit
    if cm.getKillsInLastHour() >= cm.config.MaxKillsPerHour {
        fmt.Println("[ChaosMonkey] Rate limit reached, skipping attack")
        return
    }
    
    // หา running instances ที่ไม่ได้ excluded
    candidates := make([]*Instance, 0)
    for _, inst := range cm.instances {
        inst.mu.Lock()
        running := inst.Running
        inst.mu.Unlock()
        
        if running && !cm.isExcluded(inst.Group) {
            candidates = append(candidates, inst)
        }
    }
    
    if len(candidates) == 0 {
        fmt.Println("[ChaosMonkey] No eligible instances to kill")
        return
    }
    
    // ตัดสินใจว่าจะ kill หรือไม่
    if rand.Float64() > cm.config.KillProbability {
        return
    }
    
    // เลือก instance แบบสุ่ม
    target := candidates[rand.Intn(len(candidates))]
    target.Kill()
    
    cm.killCount++
    cm.killHistory = append(cm.killHistory, KillRecord{
        Timestamp:  time.Now(),
        InstanceID: target.ID,
        Group:      target.Group,
    })
    
    // Auto-restart หลังจากสักครู่ (จำลอง auto-scaling)
    go func() {
        delay := time.Duration(rand.Intn(10)+5) * time.Second
        time.Sleep(delay)
        target.Restart()
    }()
}

// Start เริ่มรัน Chaos Monkey
func (cm *ChaosMonkey) Start(ctx context.Context) {
    ticker := time.NewTicker(cm.config.Frequency)
    defer ticker.Stop()
    
    fmt.Printf("[ChaosMonkey] Started with frequency: %v, kill probability: %.0f%%\n",
        cm.config.Frequency, cm.config.KillProbability*100)
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("[ChaosMonkey] Stopped. Total kills: %d\n", cm.killCount)
            return
        case <-ticker.C:
            cm.Attack()
        }
    }
}

// GetStats คืน statistics
func (cm *ChaosMonkey) GetStats() map[string]interface{} {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    
    return map[string]interface{}{
        "total_kills":        cm.killCount,
        "kills_last_hour":    cm.getKillsInLastHour(),
        "registered_instances": len(cm.instances),
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    config := ChaosMonkeyConfig{
        Enabled:         true,
        Frequency:       2 * time.Second,
        KillProbability: 0.3,
        MaxKillsPerHour: 10,
        ExcludedGroups:  []string{"critical-service"},
    }
    
    monkey := NewChaosMonkey(config)
    
    // สร้าง instances
    services := []struct {
        group string
        count int
    }{
        {"web-server", 5},
        {"api-service", 3},
        {"worker", 4},
        {"critical-service", 2}, // excluded
    }
    
    for _, svc := range services {
        for i := 0; i < svc.count; i++ {
            monkey.RegisterInstance(&Instance{
                ID:      fmt.Sprintf("%s-%d", svc.group, i+1),
                Group:   svc.group,
                Host:    fmt.Sprintf("10.0.%d.%d", i, i+1),
                Port:    8080,
                Running: true,
            })
        }
    }
    
    // รัน Chaos Monkey เป็นเวลา 15 วินาที
    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    
    go monkey.Start(ctx)
    
    // แสดง stats ทุก 5 วินาที
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            stats := monkey.GetStats()
            fmt.Printf("\n=== Final Stats ===\n")
            for k, v := range stats {
                fmt.Printf("%s: %v\n", k, v)
            }
            return
        case <-ticker.C:
            stats := monkey.GetStats()
            fmt.Printf("Stats: kills=%v, last_hour=%v\n",
                stats["total_kills"], stats["kills_last_hour"])
        }
    }
}
```

---

## 3. Go Fault Injection Library

```go
// faultinjection/fault.go
package main

import (
    "context"
    "errors"
    "fmt"
    "math/rand"
    "net/http"
    "time"
)

// FaultType ประเภทของ fault
type FaultType string

const (
    FaultLatency    FaultType = "latency"
    FaultError      FaultType = "error"
    FaultPanic      FaultType = "panic"
    FaultTimeout    FaultType = "timeout"
    FaultCorruption FaultType = "corruption"
)

// FaultRule กฎสำหรับ fault injection
type FaultRule struct {
    ID          string
    Type        FaultType
    Enabled     bool
    Probability float64
    
    // สำหรับ latency fault
    MinLatency time.Duration
    MaxLatency time.Duration
    
    // สำหรับ error fault
    Error      error
    StatusCode int
    
    // Filter: เงื่อนไขว่าจะ apply กับ request ไหน
    PathFilter   string
    MethodFilter string
    HeaderFilter map[string]string
}

// FaultRegistry เก็บ fault rules
type FaultRegistry struct {
    rules map[string]*FaultRule
}

// NewFaultRegistry สร้าง registry ใหม่
func NewFaultRegistry() *FaultRegistry {
    return &FaultRegistry{
        rules: make(map[string]*FaultRule),
    }
}

// AddRule เพิ่ม rule
func (fr *FaultRegistry) AddRule(rule *FaultRule) {
    fr.rules[rule.ID] = rule
}

// RemoveRule ลบ rule
func (fr *FaultRegistry) RemoveRule(id string) {
    delete(fr.rules, id)
}

// GetActiveRules คืน rules ที่ active
func (fr *FaultRegistry) GetActiveRules() []*FaultRule {
    active := make([]*FaultRule, 0)
    for _, rule := range fr.rules {
        if rule.Enabled {
            active = append(active, rule)
        }
    }
    return active
}

// FaultMiddleware HTTP middleware สำหรับ fault injection
func FaultMiddleware(registry *FaultRegistry) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            rules := registry.GetActiveRules()
            
            for _, rule := range rules {
                if !shouldApplyRule(rule, r) {
                    continue
                }
                
                if rand.Float64() > rule.Probability {
                    continue
                }
                
                switch rule.Type {
                case FaultLatency:
                    latency := rule.MinLatency + time.Duration(rand.Int63n(int64(rule.MaxLatency-rule.MinLatency)))
                    fmt.Printf("[FaultInjection] Injecting %v latency\n", latency)
                    time.Sleep(latency)
                    
                case FaultError:
                    statusCode := rule.StatusCode
                    if statusCode == 0 {
                        statusCode = http.StatusInternalServerError
                    }
                    fmt.Printf("[FaultInjection] Injecting HTTP %d error\n", statusCode)
                    http.Error(w, "Injected fault", statusCode)
                    return
                    
                case FaultTimeout:
                    fmt.Println("[FaultInjection] Injecting timeout")
                    time.Sleep(30 * time.Second) // จำลอง timeout
                    return
                }
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// shouldApplyRule ตรวจสอบว่าควร apply rule กับ request นี้หรือไม่
func shouldApplyRule(rule *FaultRule, r *http.Request) bool {
    if rule.PathFilter != "" && r.URL.Path != rule.PathFilter {
        return false
    }
    if rule.MethodFilter != "" && r.Method != rule.MethodFilter {
        return false
    }
    for key, value := range rule.HeaderFilter {
        if r.Header.Get(key) != value {
            return false
        }
    }
    return true
}

// FaultClient HTTP client ที่รองรับ fault injection
type FaultClient struct {
    registry *FaultRegistry
    base     *http.Client
}

// NewFaultClient สร้าง fault client ใหม่
func NewFaultClient(registry *FaultRegistry) *FaultClient {
    return &FaultClient{
        registry: registry,
        base:     &http.Client{},
    }
}

// Do ส่ง HTTP request พร้อม fault injection
func (fc *FaultClient) Do(req *http.Request) (*http.Response, error) {
    rules := fc.registry.GetActiveRules()
    
    for _, rule := range rules {
        if !shouldApplyRule(rule, req) {
            continue
        }
        
        if rand.Float64() > rule.Probability {
            continue
        }
        
        switch rule.Type {
        case FaultLatency:
            latency := rule.MinLatency + time.Duration(rand.Int63n(int64(rule.MaxLatency-rule.MinLatency)))
            fmt.Printf("[FaultClient] Injecting %v latency\n", latency)
            
            select {
            case <-time.After(latency):
            case <-req.Context().Done():
                return nil, req.Context().Err()
            }
            
        case FaultError:
            fmt.Println("[FaultClient] Injecting error")
            if rule.Error != nil {
                return nil, rule.Error
            }
            return nil, errors.New("injected fault error")
            
        case FaultTimeout:
            fmt.Println("[FaultClient] Injecting timeout")
            return nil, context.DeadlineExceeded
        }
    }
    
    return fc.base.Do(req)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    registry := NewFaultRegistry()
    
    // เพิ่ม fault rules
    registry.AddRule(&FaultRule{
        ID:          "latency-rule",
        Type:        FaultLatency,
        Enabled:     true,
        Probability: 0.3,
        MinLatency:  100 * time.Millisecond,
        MaxLatency:  500 * time.Millisecond,
    })
    
    registry.AddRule(&FaultRule{
        ID:          "error-rule",
        Type:        FaultError,
        Enabled:     true,
        Probability: 0.1,
        StatusCode:  http.StatusServiceUnavailable,
        PathFilter:  "/api/users",
    })
    
    // สร้าง test handler
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, `{"users": []}`)
    })
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, `{"status": "ok"}`)
    })
    
    // Wrap with fault middleware
    handler := FaultMiddleware(registry)(mux)
    
    fmt.Println("Server starting on :8080 with fault injection enabled")
    fmt.Println("Active fault rules:")
    for _, rule := range registry.GetActiveRules() {
        fmt.Printf("  - %s (type: %s, probability: %.0f%%)\n", 
            rule.ID, rule.Type, rule.Probability*100)
    }
    
    // จำลองการทำงาน (ไม่ start server จริงๆ เพื่อ demo)
    _ = handler
    fmt.Println("\nFault injection middleware configured successfully")
    
    // ทดสอบ fault injection โดยตรง
    fmt.Println("\n=== Testing Fault Injection ===")
    client := NewFaultClient(registry)
    
    for i := 0; i < 10; i++ {
        req, _ := http.NewRequest("GET", "http://localhost:8080/api/users", nil)
        start := time.Now()
        _, err := client.Do(req)
        elapsed := time.Since(start)
        
        if err != nil {
            fmt.Printf("Request %d: ERROR (%v) - %v\n", i+1, elapsed.Round(time.Millisecond), err)
        } else {
            fmt.Printf("Request %d: OK (%v)\n", i+1, elapsed.Round(time.Millisecond))
        }
    }
}
```

---

## 4. Game Days

```go
// gameday/runner.go
package main

import (
    "context"
    "fmt"
    "strings"
    "time"
)

// GameDayScenario สถานการณ์สำหรับ game day
type GameDayScenario struct {
    ID          string
    Name        string
    Description string
    Severity    SeverityLevel
    Steps       []GameDayStep
    RunbookURL  string
    Owner       string
}

// SeverityLevel ระดับความรุนแรง
type SeverityLevel int

const (
    SeverityLow      SeverityLevel = 1
    SeverityMedium   SeverityLevel = 2
    SeverityHigh     SeverityLevel = 3
    SeverityCritical SeverityLevel = 4
)

func (s SeverityLevel) String() string {
    switch s {
    case SeverityLow:
        return "LOW"
    case SeverityMedium:
        return "MEDIUM"
    case SeverityHigh:
        return "HIGH"
    case SeverityCritical:
        return "CRITICAL"
    default:
        return "UNKNOWN"
    }
}

// GameDayStep ขั้นตอนใน game day
type GameDayStep struct {
    Order       int
    Name        string
    Description string
    Execute     func(ctx context.Context) error
    Verify      func(ctx context.Context) error
    Rollback    func(ctx context.Context) error
    Timeout     time.Duration
}

// GameDayResult ผลลัพธ์ของ game day
type GameDayResult struct {
    ScenarioID   string
    StartTime    time.Time
    EndTime      time.Time
    StepResults  []StepResult
    Overall      bool
    Observations []string
    Improvements []string
}

// StepResult ผลลัพธ์ของแต่ละ step
type StepResult struct {
    StepName  string
    Success   bool
    Duration  time.Duration
    Error     error
    Notes     string
}

// GameDayRunner รัน game day
type GameDayRunner struct {
    scenarios []*GameDayScenario
    results   []*GameDayResult
    logger    func(string)
}

// NewGameDayRunner สร้าง runner ใหม่
func NewGameDayRunner() *GameDayRunner {
    return &GameDayRunner{
        scenarios: make([]*GameDayScenario, 0),
        results:   make([]*GameDayResult, 0),
        logger: func(msg string) {
            fmt.Printf("[GameDay] %s\n", msg)
        },
    }
}

// AddScenario เพิ่ม scenario
func (gdr *GameDayRunner) AddScenario(scenario *GameDayScenario) {
    gdr.scenarios = append(gdr.scenarios, scenario)
}

// Run รัน game day scenario
func (gdr *GameDayRunner) Run(ctx context.Context, scenarioID string) (*GameDayResult, error) {
    var scenario *GameDayScenario
    for _, s := range gdr.scenarios {
        if s.ID == scenarioID {
            scenario = s
            break
        }
    }
    
    if scenario == nil {
        return nil, fmt.Errorf("scenario not found: %s", scenarioID)
    }
    
    result := &GameDayResult{
        ScenarioID:   scenario.ID,
        StartTime:    time.Now(),
        StepResults:  make([]StepResult, 0),
        Observations: make([]string, 0),
        Improvements: make([]string, 0),
    }
    
    gdr.logger(fmt.Sprintf("Starting Game Day: %s (Severity: %s)", scenario.Name, scenario.Severity))
    gdr.logger(fmt.Sprintf("Description: %s", scenario.Description))
    fmt.Println(strings.Repeat("=", 60))
    
    for _, step := range scenario.Steps {
        gdr.logger(fmt.Sprintf("Step %d: %s", step.Order, step.Name))
        
        stepResult := StepResult{StepName: step.Name}
        stepStart := time.Now()
        
        stepCtx, cancel := context.WithTimeout(ctx, step.Timeout)
        
        // Execute
        if step.Execute != nil {
            if err := step.Execute(stepCtx); err != nil {
                stepResult.Success = false
                stepResult.Error = err
                gdr.logger(fmt.Sprintf("  ✗ Execute failed: %v", err))
                
                // Try rollback
                if step.Rollback != nil {
                    gdr.logger("  Running rollback...")
                    if rbErr := step.Rollback(stepCtx); rbErr != nil {
                        gdr.logger(fmt.Sprintf("  ✗ Rollback failed: %v", rbErr))
                    } else {
                        gdr.logger("  ✓ Rollback successful")
                    }
                }
                
                cancel()
                stepResult.Duration = time.Since(stepStart)
                result.StepResults = append(result.StepResults, stepResult)
                continue
            }
        }
        
        // Verify
        if step.Verify != nil {
            if err := step.Verify(stepCtx); err != nil {
                stepResult.Success = false
                stepResult.Error = fmt.Errorf("verification failed: %w", err)
                gdr.logger(fmt.Sprintf("  ✗ Verification failed: %v", err))
            } else {
                stepResult.Success = true
                gdr.logger("  ✓ Step completed successfully")
            }
        } else {
            stepResult.Success = true
            gdr.logger("  ✓ Step completed")
        }
        
        cancel()
        stepResult.Duration = time.Since(stepStart)
        result.StepResults = append(result.StepResults, stepResult)
    }
    
    result.EndTime = time.Now()
    
    // คำนวณ overall result
    result.Overall = true
    for _, sr := range result.StepResults {
        if !sr.Success {
            result.Overall = false
            break
        }
    }
    
    gdr.results = append(gdr.results, result)
    gdr.printResult(result)
    
    return result, nil
}

func (gdr *GameDayRunner) printResult(result *GameDayResult) {
    fmt.Println(strings.Repeat("=", 60))
    fmt.Printf("Game Day Results for: %s\n", result.ScenarioID)
    fmt.Printf("Duration: %v\n", result.EndTime.Sub(result.StartTime).Round(time.Second))
    fmt.Printf("Overall: %s\n", map[bool]string{true: "✓ PASSED", false: "✗ FAILED"}[result.Overall])
    fmt.Println("\nStep Results:")
    
    for _, sr := range result.StepResults {
        status := "✓"
        if !sr.Success {
            status = "✗"
        }
        fmt.Printf("  %s %s (%v)\n", status, sr.StepName, sr.Duration.Round(time.Millisecond))
        if sr.Error != nil {
            fmt.Printf("    Error: %v\n", sr.Error)
        }
    }
}

func main() {
    runner := NewGameDayRunner()
    
    // สร้าง scenario: Database failover
    dbFailoverScenario := &GameDayScenario{
        ID:          "db-failover-001",
        Name:        "Database Primary Failover",
        Description: "ทดสอบว่าระบบ handle ได้เมื่อ database primary ล่มและต้อง failover ไป replica",
        Severity:    SeverityHigh,
        Owner:       "Platform Team",
        Steps: []GameDayStep{
            {
                Order:       1,
                Name:        "Verify system health",
                Description: "ตรวจสอบว่าระบบอยู่ใน steady state",
                Execute: func(ctx context.Context) error {
                    fmt.Println("    Checking system health...")
                    time.Sleep(100 * time.Millisecond)
                    return nil
                },
                Verify: func(ctx context.Context) error {
                    fmt.Println("    All systems healthy")
                    return nil
                },
                Timeout: 30 * time.Second,
            },
            {
                Order:       2,
                Name:        "Kill database primary",
                Description: "ฆ่า primary database instance",
                Execute: func(ctx context.Context) error {
                    fmt.Println("    Stopping primary database...")
                    time.Sleep(200 * time.Millisecond)
                    fmt.Println("    Primary database stopped")
                    return nil
                },
                Rollback: func(ctx context.Context) error {
                    fmt.Println("    Restarting primary database...")
                    time.Sleep(200 * time.Millisecond)
                    return nil
                },
                Timeout: 1 * time.Minute,
            },
            {
                Order:       3,
                Name:        "Verify failover",
                Description: "ตรวจสอบว่า replica ถูก promote เป็น primary",
                Execute: func(ctx context.Context) error {
                    fmt.Println("    Waiting for failover...")
                    time.Sleep(300 * time.Millisecond)
                    return nil
                },
                Verify: func(ctx context.Context) error {
                    fmt.Println("    Replica promoted to primary")
                    return nil
                },
                Timeout: 2 * time.Minute,
            },
            {
                Order:       4,
                Name:        "Test application connectivity",
                Description: "ทดสอบว่า application เชื่อมต่อกับ new primary ได้",
                Execute: func(ctx context.Context) error {
                    fmt.Println("    Testing application connectivity...")
                    time.Sleep(100 * time.Millisecond)
                    return nil
                },
                Verify: func(ctx context.Context) error {
                    fmt.Println("    Application connected to new primary")
                    return nil
                },
                Timeout: 30 * time.Second,
            },
            {
                Order:       5,
                Name:        "Cleanup",
                Description: "ทำความสะอาดหลัง experiment",
                Execute: func(ctx context.Context) error {
                    fmt.Println("    Restoring original setup...")
                    time.Sleep(100 * time.Millisecond)
                    return nil
                },
                Timeout: 5 * time.Minute,
            },
        },
    }
    
    runner.AddScenario(dbFailoverScenario)
    
    ctx := context.Background()
    _, err := runner.Run(ctx, "db-failover-001")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

---

## 5. Hypothesis-Driven Testing

```go
// hypothesis/testing.go
package main

import (
    "context"
    "fmt"
    "math"
    "time"
)

// MetricCollector เก็บ metrics ระหว่าง experiment
type MetricCollector struct {
    samples []float64
}

// AddSample เพิ่ม sample
func (mc *MetricCollector) AddSample(value float64) {
    mc.samples = append(mc.samples, value)
}

// Mean คืนค่าเฉลี่ย
func (mc *MetricCollector) Mean() float64 {
    if len(mc.samples) == 0 {
        return 0
    }
    sum := 0.0
    for _, s := range mc.samples {
        sum += s
    }
    return sum / float64(len(mc.samples))
}

// StdDev คืน standard deviation
func (mc *MetricCollector) StdDev() float64 {
    if len(mc.samples) < 2 {
        return 0
    }
    mean := mc.Mean()
    sumSq := 0.0
    for _, s := range mc.samples {
        diff := s - mean
        sumSq += diff * diff
    }
    return math.Sqrt(sumSq / float64(len(mc.samples)-1))
}

// Percentile คืน percentile
func (mc *MetricCollector) Percentile(p float64) float64 {
    if len(mc.samples) == 0 {
        return 0
    }
    
    sorted := make([]float64, len(mc.samples))
    copy(sorted, mc.samples)
    
    // Simple sort
    for i := 0; i < len(sorted); i++ {
        for j := i + 1; j < len(sorted); j++ {
            if sorted[j] < sorted[i] {
                sorted[i], sorted[j] = sorted[j], sorted[i]
            }
        }
    }
    
    idx := int(math.Ceil(p/100.0*float64(len(sorted)))) - 1
    if idx < 0 {
        idx = 0
    }
    return sorted[idx]
}

// HypothesisTest การทดสอบ hypothesis
type HypothesisTest struct {
    Name           string
    NullHypothesis string   // H0: สมมติฐานที่จะพิสูจน์หักล้าง
    AltHypothesis  string   // H1: สมมติฐานทางเลือก
    Alpha          float64  // significance level (เช่น 0.05)
    
    Control        *MetricCollector // กลุ่ม control (ปกติ)
    Treatment      *MetricCollector // กลุ่ม treatment (chaos)
}

// NewHypothesisTest สร้าง test ใหม่
func NewHypothesisTest(name, h0, h1 string, alpha float64) *HypothesisTest {
    return &HypothesisTest{
        Name:           name,
        NullHypothesis: h0,
        AltHypothesis:  h1,
        Alpha:          alpha,
        Control:        &MetricCollector{},
        Treatment:      &MetricCollector{},
    }
}

// TTestResult ผลลัพธ์ของ t-test
type TTestResult struct {
    TStatistic    float64
    PValue        float64
    RejectH0      bool
    ControlMean   float64
    TreatmentMean float64
    Difference    float64
    DiffPercent   float64
}

// RunTTest รัน two-sample t-test (simplified)
func (ht *HypothesisTest) RunTTest() *TTestResult {
    controlMean := ht.Control.Mean()
    treatmentMean := ht.Treatment.Mean()
    
    controlStd := ht.Control.StdDev()
    treatmentStd := ht.Treatment.StdDev()
    
    n1 := float64(len(ht.Control.samples))
    n2 := float64(len(ht.Treatment.samples))
    
    if n1 == 0 || n2 == 0 {
        return nil
    }
    
    // Welch's t-test
    se := math.Sqrt(controlStd*controlStd/n1 + treatmentStd*treatmentStd/n2)
    if se == 0 {
        return &TTestResult{
            ControlMean:   controlMean,
            TreatmentMean: treatmentMean,
        }
    }
    
    tStat := (treatmentMean - controlMean) / se
    
    // Approximate p-value (simplified)
    pValue := 2 * (1 - normalCDF(math.Abs(tStat)))
    
    diff := treatmentMean - controlMean
    diffPercent := 0.0
    if controlMean != 0 {
        diffPercent = (diff / controlMean) * 100
    }
    
    return &TTestResult{
        TStatistic:    tStat,
        PValue:        pValue,
        RejectH0:      pValue < ht.Alpha,
        ControlMean:   controlMean,
        TreatmentMean: treatmentMean,
        Difference:    diff,
        DiffPercent:   diffPercent,
    }
}

// normalCDF approximation of normal CDF
func normalCDF(x float64) float64 {
    return 0.5 * math.Erfc(-x/math.Sqrt2)
}

func main() {
    // ทดสอบ: "การเพิ่ม latency 100ms จะไม่กระทบ error rate อย่างมีนัยสำคัญ"
    test := NewHypothesisTest(
        "Latency Impact on Error Rate",
        "H0: การเพิ่ม latency ไม่กระทบ error rate (mean_control = mean_treatment)",
        "H1: การเพิ่ม latency กระทบ error rate (mean_control ≠ mean_treatment)",
        0.05,
    )
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    // เก็บ samples จาก control group (ปกติ)
    go func() {
        for i := 0; i < 100; i++ {
            select {
            case <-ctx.Done():
                return
            default:
                // Error rate ปกติ ~2%
                errorRate := 0.02 + float64(i%3)*0.005
                test.Control.AddSample(errorRate)
                time.Sleep(50 * time.Millisecond)
            }
        }
    }()
    
    // เก็บ samples จาก treatment group (chaos)
    go func() {
        for i := 0; i < 100; i++ {
            select {
            case <-ctx.Done():
                return
            default:
                // Error rate สูงขึ้นเล็กน้อย ~3%
                errorRate := 0.03 + float64(i%5)*0.003
                test.Treatment.AddSample(errorRate)
                time.Sleep(50 * time.Millisecond)
            }
        }
    }()
    
    time.Sleep(6 * time.Second)
    cancel()
    
    result := test.RunTTest()
    if result == nil {
        fmt.Println("Insufficient data")
        return
    }
    
    fmt.Printf("=== Hypothesis Test: %s ===\n", test.Name)
    fmt.Printf("H0: %s\n", test.NullHypothesis)
    fmt.Printf("H1: %s\n", test.AltHypothesis)
    fmt.Printf("Alpha: %.2f\n\n", test.Alpha)
    fmt.Printf("Control Mean: %.4f (%.2f%%)\n", result.ControlMean, result.ControlMean*100)
    fmt.Printf("Treatment Mean: %.4f (%.2f%%)\n", result.TreatmentMean, result.TreatmentMean*100)
    fmt.Printf("Difference: %.4f (%.2f%%)\n", result.Difference, result.DiffPercent)
    fmt.Printf("T-Statistic: %.4f\n", result.TStatistic)
    fmt.Printf("P-Value: %.4f\n", result.PValue)
    
    if result.RejectH0 {
        fmt.Printf("\nConclusion: REJECT H0 (p=%.4f < α=%.2f)\n", result.PValue, test.Alpha)
        fmt.Println("=> Chaos significantly affected the error rate!")
    } else {
        fmt.Printf("\nConclusion: FAIL TO REJECT H0 (p=%.4f >= α=%.2f)\n", result.PValue, test.Alpha)
        fmt.Println("=> No significant impact detected")
    }
}
```

---

## 6. Blast Radius Management

```go
// blastradius/manager.go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

// BlastRadiusScope scope ของ blast radius
type BlastRadiusScope string

const (
    ScopeHost      BlastRadiusScope = "host"
    ScopeService   BlastRadiusScope = "service"
    ScopeDatacenter BlastRadiusScope = "datacenter"
    ScopeRegion    BlastRadiusScope = "region"
)

// Target เป้าหมายที่จะได้รับผลกระทบ
type Target struct {
    ID       string
    Scope    BlastRadiusScope
    Critical bool
}

// BlastRadiusController จัดการ blast radius
type BlastRadiusController struct {
    maxPercent    float64
    maxTargets    int
    criticalSafe  bool // ป้องกัน critical targets
    targets       []Target
}

// NewBlastRadiusController สร้าง controller ใหม่
func NewBlastRadiusController(maxPercent float64, maxTargets int, criticalSafe bool) *BlastRadiusController {
    return &BlastRadiusController{
        maxPercent:   maxPercent,
        maxTargets:   maxTargets,
        criticalSafe: criticalSafe,
        targets:      make([]Target, 0),
    }
}

// RegisterTarget ลงทะเบียน target
func (brc *BlastRadiusController) RegisterTarget(target Target) {
    brc.targets = append(brc.targets, target)
}

// SelectTargets เลือก targets ที่จะได้รับผลกระทบ
func (brc *BlastRadiusController) SelectTargets() []Target {
    eligible := make([]Target, 0)
    
    for _, t := range brc.targets {
        if brc.criticalSafe && t.Critical {
            continue
        }
        eligible = append(eligible, t)
    }
    
    if len(eligible) == 0 {
        return nil
    }
    
    // คำนวณจำนวน targets สูงสุด
    maxByPercent := int(float64(len(eligible)) * brc.maxPercent / 100)
    maxCount := maxByPercent
    if brc.maxTargets > 0 && brc.maxTargets < maxCount {
        maxCount = brc.maxTargets
    }
    
    if maxCount <= 0 {
        maxCount = 1
    }
    
    // เลือก targets แบบสุ่ม
    rand.Shuffle(len(eligible), func(i, j int) {
        eligible[i], eligible[j] = eligible[j], eligible[i]
    })
    
    if maxCount > len(eligible) {
        maxCount = len(eligible)
    }
    
    return eligible[:maxCount]
}

// GetBlastImpact คำนวณผลกระทบ
func (brc *BlastRadiusController) GetBlastImpact(selected []Target) map[string]interface{} {
    totalTargets := len(brc.targets)
    affectedTargets := len(selected)
    
    criticalAffected := 0
    for _, t := range selected {
        if t.Critical {
            criticalAffected++
        }
    }
    
    return map[string]interface{}{
        "total_targets":    totalTargets,
        "affected_targets": affectedTargets,
        "affected_percent": float64(affectedTargets) / float64(totalTargets) * 100,
        "critical_affected": criticalAffected,
        "is_safe":          criticalAffected == 0,
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    // สร้าง controller: สูงสุด 30% ของ hosts, ไม่เกิน 5 targets
    controller := NewBlastRadiusController(30.0, 5, true)
    
    // ลงทะเบียน targets (จำลอง production hosts)
    for i := 1; i <= 20; i++ {
        critical := i <= 3 // hosts 1-3 เป็น critical
        controller.RegisterTarget(Target{
            ID:       fmt.Sprintf("host-%02d", i),
            Scope:    ScopeHost,
            Critical: critical,
        })
    }
    
    fmt.Println("=== Blast Radius Management ===")
    fmt.Printf("Total targets: 20 (3 critical, 17 non-critical)\n")
    fmt.Printf("Max percentage: %.0f%%\n", controller.maxPercent)
    fmt.Printf("Max targets: %d\n", controller.maxTargets)
    fmt.Printf("Critical safe: %v\n\n", controller.criticalSafe)
    
    // เลือก targets
    selected := controller.SelectTargets()
    impact := controller.GetBlastImpact(selected)
    
    fmt.Printf("Selected targets for chaos experiment:\n")
    for _, t := range selected {
        critical := ""
        if t.Critical {
            critical = " (CRITICAL)"
        }
        fmt.Printf("  - %s%s\n", t.ID, critical)
    }
    
    fmt.Printf("\nImpact Analysis:\n")
    fmt.Printf("  Affected: %v/%v (%.1f%%)\n",
        impact["affected_targets"],
        impact["total_targets"],
        impact["affected_percent"])
    fmt.Printf("  Critical affected: %v\n", impact["critical_affected"])
    fmt.Printf("  Is safe: %v\n", impact["is_safe"])
}
```

---

## Workshop: Complete Chaos Engineering Setup

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// ChaosTestSuite ชุดทดสอบ chaos ที่สมบูรณ์
type ChaosTestSuite struct {
    mu          sync.Mutex
    tests       []ChaosTest
    results     []ChaosTestResult
    failFast    bool
}

// ChaosTest ชุดทดสอบ
type ChaosTest struct {
    Name     string
    Setup    func() error
    Chaos    func(ctx context.Context) error
    Verify   func() (bool, string)
    Cleanup  func() error
    Timeout  time.Duration
}

// ChaosTestResult ผลลัพธ์
type ChaosTestResult struct {
    Name     string
    Passed   bool
    Message  string
    Duration time.Duration
}

// NewChaosTestSuite สร้าง test suite ใหม่
func NewChaosTestSuite(failFast bool) *ChaosTestSuite {
    return &ChaosTestSuite{
        tests:    make([]ChaosTest, 0),
        results:  make([]ChaosTestResult, 0),
        failFast: failFast,
    }
}

// Add เพิ่ม test
func (cts *ChaosTestSuite) Add(test ChaosTest) {
    cts.mu.Lock()
    defer cts.mu.Unlock()
    cts.tests = append(cts.tests, test)
}

// Run รัน tests ทั้งหมด
func (cts *ChaosTestSuite) Run(ctx context.Context) {
    fmt.Println("=== Chaos Test Suite ===")
    
    for _, test := range cts.tests {
        result := cts.runTest(ctx, test)
        cts.results = append(cts.results, result)
        
        status := "✓ PASS"
        if !result.Passed {
            status = "✗ FAIL"
        }
        fmt.Printf("[%s] %s (%v): %s\n", status, result.Name, result.Duration.Round(time.Millisecond), result.Message)
        
        if cts.failFast && !result.Passed {
            fmt.Println("Stopping due to failFast=true")
            break
        }
    }
    
    cts.printSummary()
}

func (cts *ChaosTestSuite) runTest(ctx context.Context, test ChaosTest) ChaosTestResult {
    start := time.Now()
    
    // Setup
    if test.Setup != nil {
        if err := test.Setup(); err != nil {
            return ChaosTestResult{
                Name:     test.Name,
                Passed:   false,
                Message:  fmt.Sprintf("Setup failed: %v", err),
                Duration: time.Since(start),
            }
        }
    }
    
    // Run chaos
    timeout := test.Timeout
    if timeout == 0 {
        timeout = 30 * time.Second
    }
    
    chaosCtx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()
    
    chaosErr := test.Chaos(chaosCtx)
    
    // Verify
    passed := true
    message := "OK"
    
    if test.Verify != nil {
        var ok bool
        ok, message = test.Verify()
        passed = ok
    } else if chaosErr != nil && chaosErr != context.DeadlineExceeded {
        passed = false
        message = fmt.Sprintf("Chaos error: %v", chaosErr)
    }
    
    // Cleanup
    if test.Cleanup != nil {
        test.Cleanup()
    }
    
    return ChaosTestResult{
        Name:     test.Name,
        Passed:   passed,
        Message:  message,
        Duration: time.Since(start),
    }
}

func (cts *ChaosTestSuite) printSummary() {
    passed := 0
    failed := 0
    
    for _, r := range cts.results {
        if r.Passed {
            passed++
        } else {
            failed++
        }
    }
    
    fmt.Printf("\n=== Summary: %d passed, %d failed ===\n", passed, failed)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    suite := NewChaosTestSuite(false)
    
    // Test 1: Network partition simulation
    serviceAvailable := true
    suite.Add(ChaosTest{
        Name: "Service survives network partition",
        Setup: func() error {
            serviceAvailable = true
            return nil
        },
        Chaos: func(ctx context.Context) error {
            // จำลอง network partition เป็นเวลา 2 วินาที
            serviceAvailable = false
            select {
            case <-time.After(2 * time.Second):
                serviceAvailable = true
                return nil
            case <-ctx.Done():
                serviceAvailable = true
                return ctx.Err()
            }
        },
        Verify: func() (bool, string) {
            if serviceAvailable {
                return true, "Service recovered after partition"
            }
            return false, "Service still unavailable"
        },
        Timeout: 10 * time.Second,
    })
    
    // Test 2: Memory pressure
    suite.Add(ChaosTest{
        Name: "System handles memory pressure",
        Chaos: func(ctx context.Context) error {
            // จำลอง memory allocation
            allocated := make([][]byte, 0)
            for i := 0; i < 10; i++ {
                select {
                case <-ctx.Done():
                    return nil
                default:
                    allocated = append(allocated, make([]byte, 1024*1024)) // 1MB
                    time.Sleep(100 * time.Millisecond)
                }
            }
            _ = allocated
            return nil
        },
        Verify: func() (bool, string) {
            return true, "System handled memory pressure"
        },
        Timeout: 5 * time.Second,
    })
    
    // Test 3: High CPU load
    suite.Add(ChaosTest{
        Name: "Service responsive under CPU load",
        Chaos: func(ctx context.Context) error {
            // จำลอง CPU load
            done := make(chan struct{})
            go func() {
                defer close(done)
                for {
                    select {
                    case <-ctx.Done():
                        return
                    default:
                        // Busy loop
                        _ = rand.Float64() * rand.Float64()
                    }
                }
            }()
            
            select {
            case <-ctx.Done():
            case <-done:
            }
            return nil
        },
        Verify: func() (bool, string) {
            return true, "Service remained responsive under load"
        },
        Timeout: 3 * time.Second,
    })
    
    ctx := context.Background()
    suite.Run(ctx)
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Chaos Engineering Principles** - 5 หลักการสำคัญ
2. **Netflix Chaos Monkey** - แนวคิดและการ implement
3. **Go Fault Injection** - Library สำหรับ inject faults
4. **Game Days** - การซ้อมรับมือกับ incidents
5. **Hypothesis-Driven Testing** - การทดสอบด้วย hypothesis ทางสถิติ
6. **Blast Radius Management** - การจำกัดผลกระทบ

### Key Takeaways

- **เริ่มต้นเล็กๆ**: เริ่มจาก dev/staging environment ก่อน production
- **สร้าง observability ก่อน**: ต้องมี metrics/logs ที่ดีก่อนจะทำ chaos
- **ทำงานเป็นทีม**: Game days ต้องการความร่วมมือจากทุกคน
- **Document ทุกอย่าง**: บันทึกผลและข้อค้นพบ
- **Automate**: ทำ chaos experiments เป็น automated tests
