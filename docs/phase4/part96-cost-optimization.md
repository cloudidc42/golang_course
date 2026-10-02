# Part 96: Cloud Cost Optimization

## เป้าหมายของบทเรียน
- Cloud cost optimization strategies
- Right-sizing instances
- Spot/Preemptible instances
- Go application cost optimization
- Cost monitoring และ alerting

---

## 1. Cost Analysis Framework

```go
// cost_analyzer.go - วิเคราะห์ cloud costs
package main

import (
    "fmt"
    "sort"
    "time"
)

// ResourceType ประเภท resource
type ResourceType string

const (
    EC2Instance   ResourceType = "EC2"
    RDSInstance   ResourceType = "RDS"
    S3Storage     ResourceType = "S3"
    DataTransfer  ResourceType = "DataTransfer"
    LambdaFunc    ResourceType = "Lambda"
)

// CostEntry รายการค่าใช้จ่าย
type CostEntry struct {
    Service      string
    ResourceType ResourceType
    ResourceID   string
    Region       string
    DailyCost    float64
    MonthlyCost  float64
    Tags         map[string]string
    Utilization  float64 // percentage (0-100)
}

// CostReport รายงานค่าใช้จ่าย
type CostReport struct {
    Period     string
    TotalCost  float64
    Entries    []*CostEntry
    Anomalies  []*CostAnomaly
}

// CostAnomaly ความผิดปกติด้านค่าใช้จ่าย
type CostAnomaly struct {
    ResourceID   string
    Expected     float64
    Actual       float64
    Difference   float64
    Percentage   float64
    DetectedAt   time.Time
}

// Recommendations คำแนะนำประหยัดค่าใช้จ่าย
type Recommendation struct {
    ResourceID      string
    Type            string
    Description     string
    EstimatedSaving float64
    Effort          string // Low, Medium, High
    Priority        string // Critical, High, Medium, Low
}

// CostOptimizer วิเคราะห์และแนะนำ optimization
type CostOptimizer struct {
    entries []*CostEntry
}

func NewCostOptimizer(entries []*CostEntry) *CostOptimizer {
    return &CostOptimizer{entries: entries}
}

// FindUnderutilized หา resources ที่ใช้งานน้อย
func (o *CostOptimizer) FindUnderutilized(threshold float64) []*CostEntry {
    var result []*CostEntry
    for _, e := range o.entries {
        if e.Utilization < threshold && e.Utilization > 0 {
            result = append(result, e)
        }
    }
    return result
}

// GenerateRecommendations สร้างคำแนะนำ
func (o *CostOptimizer) GenerateRecommendations() []Recommendation {
    var recs []Recommendation
    
    for _, entry := range o.entries {
        // Right-sizing recommendation
        if entry.ResourceType == EC2Instance && entry.Utilization < 20 {
            recs = append(recs, Recommendation{
                ResourceID:      entry.ResourceID,
                Type:            "Right-sizing",
                Description:     fmt.Sprintf("Downsize %s - only %.0f%% utilization", entry.ResourceID, entry.Utilization),
                EstimatedSaving: entry.MonthlyCost * 0.5,
                Effort:          "Low",
                Priority:        "High",
            })
        }
        
        // Spot instance recommendation
        if entry.ResourceType == EC2Instance && entry.Utilization > 0 {
            savings := entry.MonthlyCost * 0.7
            if savings > 50 {
                recs = append(recs, Recommendation{
                    ResourceID:      entry.ResourceID,
                    Type:            "Spot Instance",
                    Description:     fmt.Sprintf("Convert %s to Spot (save up to 70%%)", entry.ResourceID),
                    EstimatedSaving: savings,
                    Effort:          "Medium",
                    Priority:        "Medium",
                })
            }
        }
        
        // Reserved instance recommendation
        if entry.ResourceType == RDSInstance && entry.Utilization > 80 {
            recs = append(recs, Recommendation{
                ResourceID:      entry.ResourceID,
                Type:            "Reserved Instance",
                Description:     fmt.Sprintf("Convert %s to 1-year reserved (save ~40%%)", entry.ResourceID),
                EstimatedSaving: entry.MonthlyCost * 0.4,
                Effort:          "Low",
                Priority:        "High",
            })
        }
        
        // Idle resource
        if entry.Utilization == 0 {
            recs = append(recs, Recommendation{
                ResourceID:      entry.ResourceID,
                Type:            "Delete Idle Resource",
                Description:     fmt.Sprintf("Remove idle %s %s", entry.ResourceType, entry.ResourceID),
                EstimatedSaving: entry.MonthlyCost,
                Effort:          "Low",
                Priority:        "Critical",
            })
        }
    }
    
    // Sort by savings
    sort.Slice(recs, func(i, j int) bool {
        return recs[i].EstimatedSaving > recs[j].EstimatedSaving
    })
    
    return recs
}

// TotalMonthlyCost คำนวณยอดรวม
func (o *CostOptimizer) TotalMonthlyCost() float64 {
    total := 0.0
    for _, e := range o.entries {
        total += e.MonthlyCost
    }
    return total
}

func main() {
    fmt.Println("=== Cloud Cost Optimization Demo ===\n")
    
    // Mock data
    entries := []*CostEntry{
        {
            Service:      "EC2",
            ResourceType: EC2Instance,
            ResourceID:   "i-0123456789abcdef0",
            Region:       "us-east-1",
            DailyCost:    10.50,
            MonthlyCost:  315.0,
            Utilization:  8.5,
            Tags:         map[string]string{"env": "prod", "app": "api"},
        },
        {
            Service:      "EC2",
            ResourceType: EC2Instance,
            ResourceID:   "i-0987654321fedcba0",
            Region:       "us-east-1",
            DailyCost:    5.25,
            MonthlyCost:  157.5,
            Utilization:  0, // IDLE!
            Tags:         map[string]string{"env": "dev"},
        },
        {
            Service:      "RDS",
            ResourceType: RDSInstance,
            ResourceID:   "db-prod-postgres-01",
            Region:       "us-east-1",
            DailyCost:    25.0,
            MonthlyCost:  750.0,
            Utilization:  85.0,
            Tags:         map[string]string{"env": "prod"},
        },
        {
            Service:      "Lambda",
            ResourceType: LambdaFunc,
            ResourceID:   "my-function",
            Region:       "us-east-1",
            DailyCost:    0.50,
            MonthlyCost:  15.0,
            Utilization:  60.0,
        },
    }
    
    optimizer := NewCostOptimizer(entries)
    
    fmt.Printf("Total Monthly Cost: $%.2f\n\n", optimizer.TotalMonthlyCost())
    
    // Underutilized resources
    fmt.Println("Underutilized Resources (<30% utilization):")
    for _, e := range optimizer.FindUnderutilized(30) {
        fmt.Printf("  %s: %.0f%% utilization ($%.2f/month)\n",
            e.ResourceID, e.Utilization, e.MonthlyCost)
    }
    
    // Recommendations
    fmt.Println("\nOptimization Recommendations:")
    recs := optimizer.GenerateRecommendations()
    
    totalSavings := 0.0
    for i, rec := range recs {
        fmt.Printf("\n%d. [%s] %s\n", i+1, rec.Priority, rec.Type)
        fmt.Printf("   Resource: %s\n", rec.ResourceID)
        fmt.Printf("   Action: %s\n", rec.Description)
        fmt.Printf("   Estimated Saving: $%.2f/month\n", rec.EstimatedSaving)
        fmt.Printf("   Effort: %s\n", rec.Effort)
        totalSavings += rec.EstimatedSaving
    }
    
    fmt.Printf("\nTotal Potential Savings: $%.2f/month (%.0f%% of current cost)\n",
        totalSavings,
        totalSavings/optimizer.TotalMonthlyCost()*100)
}
```

---

## 2. Go Application Performance Optimization

```go
// app_optimization.go - Optimize Go app สำหรับลดค่าใช้จ่าย
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "sync"
    "time"
)

/*
Go Application Cost Optimization Strategies:

1. Memory optimization
   - ลด allocations
   - ใช้ object pools
   - ลด GC pressure
   
2. CPU optimization
   - ลด unnecessary work
   - Cache results
   - Use goroutines efficiently
   
3. I/O optimization
   - Connection pooling
   - Request batching
   - Caching
   
4. Startup optimization
   - Lazy initialization
   - ลด cold start time
*/

// ObjectPool ลด heap allocations
type ObjectPool[T any] struct {
    pool sync.Pool
}

func NewObjectPool[T any](factory func() T) *ObjectPool[T] {
    return &ObjectPool[T]{
        pool: sync.Pool{
            New: func() interface{} {
                v := factory()
                return &v
            },
        },
    }
}

func (p *ObjectPool[T]) Get() *T {
    return p.pool.Get().(*T)
}

func (p *ObjectPool[T]) Put(v *T) {
    p.pool.Put(v)
}

// RequestBatcher รวม requests เพื่อลด DB calls
type RequestBatcher[K comparable, V any] struct {
    mu      sync.Mutex
    pending map[K][]chan V
    fn      func(keys []K) map[K]V
    delay   time.Duration
    timer   *time.Timer
}

func NewRequestBatcher[K comparable, V any](
    fn func([]K) map[K]V,
    delay time.Duration,
) *RequestBatcher[K, V] {
    return &RequestBatcher[K, V]{
        pending: make(map[K][]chan V),
        fn:      fn,
        delay:   delay,
    }
}

func (b *RequestBatcher[K, V]) Load(key K) V {
    b.mu.Lock()
    
    ch := make(chan V, 1)
    b.pending[key] = append(b.pending[key], ch)
    
    if b.timer == nil {
        b.timer = time.AfterFunc(b.delay, b.flush)
    }
    
    b.mu.Unlock()
    
    return <-ch
}

func (b *RequestBatcher[K, V]) flush() {
    b.mu.Lock()
    pending := b.pending
    b.pending = make(map[K][]chan V)
    b.timer = nil
    b.mu.Unlock()
    
    keys := make([]K, 0, len(pending))
    for k := range pending {
        keys = append(keys, k)
    }
    
    results := b.fn(keys)
    
    for key, channels := range pending {
        val := results[key]
        for _, ch := range channels {
            ch <- val
        }
    }
}

// MemoryOptimizationDemo แสดงเทคนิค memory optimization
func MemoryOptimizationDemo() {
    fmt.Println("=== Memory Optimization ===\n")
    
    // BEFORE: allocation ทุก request
    before := func() []byte {
        return make([]byte, 1024)
    }
    
    // AFTER: reuse buffers
    bufPool := &sync.Pool{
        New: func() interface{} {
            return make([]byte, 1024)
        },
    }
    
    after := func() []byte {
        buf := bufPool.Get().([]byte)
        return buf
    }
    
    returnBuf := func(buf []byte) {
        buf = buf[:0]
        bufPool.Put(buf)
    }
    
    // Benchmark
    runtime.GC()
    var ms1 runtime.MemStats
    runtime.ReadMemStats(&ms1)
    
    for i := 0; i < 100000; i++ {
        _ = before()
    }
    
    var ms2 runtime.MemStats
    runtime.ReadMemStats(&ms2)
    
    runtime.GC()
    var ms3 runtime.MemStats
    runtime.ReadMemStats(&ms3)
    
    for i := 0; i < 100000; i++ {
        buf := after()
        returnBuf(buf)
    }
    
    var ms4 runtime.MemStats
    runtime.ReadMemStats(&ms4)
    
    fmt.Printf("Without pool: %d allocations\n", ms2.Mallocs-ms1.Mallocs)
    fmt.Printf("With pool:    %d allocations\n", ms4.Mallocs-ms3.Mallocs)
}

// GCTuningDemo แสดงการ tune GC
func GCTuningDemo() {
    fmt.Println("\n=== GC Tuning for Cost ===\n")
    
    // Default GOGC=100 (GC when heap doubles)
    // Higher GOGC = fewer GC runs = lower CPU = lower cost
    // But higher memory usage
    
    // For Lambda/serverless: lower GOGC to reduce memory
    // For long-running services: higher GOGC to reduce CPU
    
    currentGOGC := debug.SetGCPercent(-1)
    fmt.Printf("Current GOGC: %d\n", currentGOGC)
    debug.SetGCPercent(currentGOGC)
    
    strategies := []struct {
        env     string
        gogc    int
        reason  string
        savings string
    }{
        {
            env:     "Lambda (128MB)",
            gogc:    50,
            reason:  "Reduce memory usage, accept more GC CPU",
            savings: "Lower memory tier = lower cost",
        },
        {
            env:     "Long-running API server",
            gogc:    200,
            reason:  "Reduce GC frequency, use more memory",
            savings: "Lower CPU usage = smaller instance",
        },
        {
            env:     "Batch processing",
            gogc:    500,
            reason:  "Maximize throughput, GC at end",
            savings: "Faster completion = less compute time",
        },
    }
    
    fmt.Println("GC Tuning Strategies:")
    for _, s := range strategies {
        fmt.Printf("\n  Environment: %s\n", s.env)
        fmt.Printf("  GOGC=%d\n", s.gogc)
        fmt.Printf("  Reason: %s\n", s.reason)
        fmt.Printf("  Savings: %s\n", s.savings)
    }
}

func main() {
    fmt.Println("=== Go App Cost Optimization ===\n")
    
    MemoryOptimizationDemo()
    GCTuningDemo()
    
    fmt.Println("\n=== Cost Optimization Checklist ===\n")
    
    checklist := []struct {
        category string
        items    []string
    }{
        {
            category: "Instance Right-sizing",
            items: []string{
                "Review CPU/memory utilization weekly",
                "Use smallest instance that meets SLA",
                "Consider ARM (Graviton) - 20% cheaper",
                "Use auto-scaling to match demand",
            },
        },
        {
            category: "Spot/Preemptible Instances",
            items: []string{
                "Use Spot for non-critical batch jobs",
                "Implement graceful shutdown for preemption",
                "Mix On-Demand + Spot for resilience",
                "Use Spot Fleet for automatic diversification",
            },
        },
        {
            category: "Go App Optimization",
            items: []string{
                "Profile CPU and memory (pprof)",
                "Use sync.Pool for frequent allocations",
                "Tune GOGC based on workload",
                "Batch database/API calls",
                "Cache expensive computations",
            },
        },
        {
            category: "Data Transfer",
            items: []string{
                "Use same region for services",
                "Compress data (gzip/zstd)",
                "CDN for static assets",
                "Reduce unnecessary API calls",
            },
        },
    }
    
    for _, section := range checklist {
        fmt.Printf("%s:\n", section.category)
        for _, item := range section.items {
            fmt.Printf("  [ ] %s\n", item)
        }
        fmt.Println()
    }
}
```

---

## สรุป

บทนี้ครอบคลุม Cloud Cost Optimization:

1. **Cost Analysis** - วิเคราะห์ค่าใช้จ่าย resource ต่างๆ
2. **Right-sizing** - ปรับขนาด instance ให้เหมาะสม
3. **Spot Instances** - ประหยัดค่าใช้จ่ายสูงสุด 90%
4. **Go App Optimization** - ลด CPU/memory footprint
5. **GC Tuning** - ปรับ GC สำหรับ cost vs performance

### Key Takeaways

- Idle resources = 100% waste, ต้องลบหรือ schedule
- Under-utilized instances ควร downsize
- Spot instances เหมาะกับ batch, dev/test, stateless services
- Go apps สามารถ optimize memory ด้วย sync.Pool และ GC tuning
- วัดก่อน optimize, อย่า premature optimization
