# Part 76: High Availability Systems in Go

## เป้าหมายของบทเรียน
- เข้าใจหลักการออกแบบระบบ High Availability (HA)
- สามารถออกแบบและ implement redundancy strategies
- เข้าใจ failover mechanisms และ data replication
- สามารถออกแบบ multi-region deployments
- เข้าใจ disaster recovery และ RTO/RPO objectives
- เรียนรู้ chaos engineering basics

---

## 1. High Availability System Design

High Availability (HA) คือความสามารถของระบบในการให้บริการได้อย่างต่อเนื่อง แม้ว่าบางส่วนจะเกิดความผิดพลาด โดยทั่วไปวัดเป็น "nines":

| Availability | Downtime per Year | Downtime per Month |
|---|---|---|
| 99% (2 nines) | 87.6 hours | 7.2 hours |
| 99.9% (3 nines) | 8.76 hours | 43.8 minutes |
| 99.99% (4 nines) | 52.6 minutes | 4.4 minutes |
| 99.999% (5 nines) | 5.26 minutes | 26.3 seconds |

### หลักการพื้นฐาน

```go
// package main - HA System Design Concepts

package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

// HealthStatus แสดงสถานะของ service
type HealthStatus struct {
    Healthy   bool
    LastCheck time.Time
    Message   string
}

// Service interface สำหรับทุก service ใน cluster
type Service interface {
    Health() HealthStatus
    Start(ctx context.Context) error
    Stop() error
    Name() string
}

// HACluster จัดการ cluster ของ services
type HACluster struct {
    mu       sync.RWMutex
    services []Service
    primary  Service
    replicas []Service
}

// NewHACluster สร้าง cluster ใหม่
func NewHACluster() *HACluster {
    return &HACluster{
        services: make([]Service, 0),
        replicas: make([]Service, 0),
    }
}

// AddService เพิ่ม service เข้า cluster
func (c *HACluster) AddService(svc Service, isPrimary bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.services = append(c.services, svc)
    if isPrimary {
        c.primary = svc
    } else {
        c.replicas = append(c.replicas, svc)
    }
}

// GetHealthy คืน service ที่ healthy
func (c *HACluster) GetHealthy() Service {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    // ลอง primary ก่อน
    if c.primary != nil {
        if status := c.primary.Health(); status.Healthy {
            return c.primary
        }
    }
    
    // ลอง replicas
    for _, replica := range c.replicas {
        if status := replica.Health(); status.Healthy {
            return replica
        }
    }
    
    return nil
}

func main() {
    cluster := NewHACluster()
    fmt.Println("HA Cluster initialized")
    _ = cluster
}
```

---

## 2. Redundancy Strategies

### Active-Active vs Active-Passive

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

// RedundancyMode กำหนด mode ของ redundancy
type RedundancyMode int

const (
    ActiveActive  RedundancyMode = iota // ทุก node active พร้อมกัน
    ActivePassive                       // มี standby node
    NPlus1                              // N active + 1 standby
)

// Node แสดง server node
type Node struct {
    ID       string
    Address  string
    Active   bool
    Weight   int
    mu       sync.RWMutex
    requests int64
}

// IncrementRequests เพิ่ม request counter
func (n *Node) IncrementRequests() {
    atomic.AddInt64(&n.requests, 1)
}

// GetRequests คืนจำนวน requests
func (n *Node) GetRequests() int64 {
    return atomic.LoadInt64(&n.requests)
}

// LoadBalancer จัดการ load balancing
type LoadBalancer struct {
    mu      sync.RWMutex
    nodes   []*Node
    mode    RedundancyMode
    current int
}

// NewLoadBalancer สร้าง load balancer ใหม่
func NewLoadBalancer(mode RedundancyMode) *LoadBalancer {
    return &LoadBalancer{
        nodes: make([]*Node, 0),
        mode:  mode,
    }
}

// AddNode เพิ่ม node เข้า balancer
func (lb *LoadBalancer) AddNode(node *Node) {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    lb.nodes = append(lb.nodes, node)
}

// RoundRobin เลือก node แบบ round-robin
func (lb *LoadBalancer) RoundRobin() *Node {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    activeNodes := lb.getActiveNodes()
    if len(activeNodes) == 0 {
        return nil
    }
    
    node := activeNodes[lb.current%len(activeNodes)]
    lb.current++
    return node
}

// LeastConnections เลือก node ที่มี connections น้อยสุด
func (lb *LoadBalancer) LeastConnections() *Node {
    lb.mu.RLock()
    defer lb.mu.RUnlock()
    
    activeNodes := lb.getActiveNodes()
    if len(activeNodes) == 0 {
        return nil
    }
    
    var selected *Node
    var minRequests int64 = -1
    
    for _, node := range activeNodes {
        reqs := node.GetRequests()
        if minRequests == -1 || reqs < minRequests {
            minRequests = reqs
            selected = node
        }
    }
    
    return selected
}

// WeightedRoundRobin เลือก node ตาม weight
func (lb *LoadBalancer) WeightedRoundRobin() *Node {
    lb.mu.RLock()
    defer lb.mu.RUnlock()
    
    activeNodes := lb.getActiveNodes()
    if len(activeNodes) == 0 {
        return nil
    }
    
    totalWeight := 0
    for _, node := range activeNodes {
        totalWeight += node.Weight
    }
    
    if totalWeight == 0 {
        return activeNodes[0]
    }
    
    // Simple weighted selection
    target := lb.current % totalWeight
    lb.current++
    
    cumulative := 0
    for _, node := range activeNodes {
        cumulative += node.Weight
        if target < cumulative {
            return node
        }
    }
    
    return activeNodes[0]
}

// getActiveNodes คืน nodes ที่ active (ไม่ lock)
func (lb *LoadBalancer) getActiveNodes() []*Node {
    active := make([]*Node, 0)
    for _, node := range lb.nodes {
        node.mu.RLock()
        if node.Active {
            active = append(active, node)
        }
        node.mu.RUnlock()
    }
    return active
}

// FailNode จำลองการ fail ของ node
func (lb *LoadBalancer) FailNode(nodeID string) {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    for _, node := range lb.nodes {
        if node.ID == nodeID {
            node.mu.Lock()
            node.Active = false
            node.mu.Unlock()
            fmt.Printf("Node %s marked as failed\n", nodeID)
            return
        }
    }
}

// RecoverNode จำลองการ recover ของ node
func (lb *LoadBalancer) RecoverNode(nodeID string) {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    for _, node := range lb.nodes {
        if node.ID == nodeID {
            node.mu.Lock()
            node.Active = true
            node.mu.Unlock()
            fmt.Printf("Node %s recovered\n", nodeID)
            return
        }
    }
}

func main() {
    lb := NewLoadBalancer(ActiveActive)
    
    // เพิ่ม nodes
    lb.AddNode(&Node{ID: "node-1", Address: "10.0.0.1:8080", Active: true, Weight: 3})
    lb.AddNode(&Node{ID: "node-2", Address: "10.0.0.2:8080", Active: true, Weight: 2})
    lb.AddNode(&Node{ID: "node-3", Address: "10.0.0.3:8080", Active: true, Weight: 1})
    
    // จำลอง requests
    for i := 0; i < 10; i++ {
        node := lb.RoundRobin()
        if node != nil {
            fmt.Printf("Request %d -> Node %s\n", i+1, node.ID)
            node.IncrementRequests()
        }
    }
    
    // จำลอง node failure
    fmt.Println("\n--- Node 1 fails ---")
    lb.FailNode("node-1")
    
    for i := 0; i < 5; i++ {
        node := lb.RoundRobin()
        if node != nil {
            fmt.Printf("Request %d -> Node %s\n", i+1, node.ID)
        }
    }
    
    // จำลอง recovery
    fmt.Println("\n--- Node 1 recovers ---")
    lb.RecoverNode("node-1")
    
    // ทดสอบ least connections
    fmt.Println("\n--- Least Connections ---")
    for i := 0; i < 5; i++ {
        node := lb.LeastConnections()
        if node != nil {
            fmt.Printf("Request %d -> Node %s (requests: %d)\n", i+1, node.ID, node.GetRequests())
        }
    }
    
    _ = context.Background()
    _ = time.Now()
}
```

---

## 3. Failover Mechanisms

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "sync"
    "time"
)

// ErrNoHealthyNode เกิดเมื่อไม่มี node ที่ healthy
var ErrNoHealthyNode = errors.New("no healthy node available")

// HealthChecker ตรวจสอบสุขภาพของ service
type HealthChecker struct {
    mu       sync.RWMutex
    checks   map[string]func() bool
    statuses map[string]bool
    interval time.Duration
}

// NewHealthChecker สร้าง health checker ใหม่
func NewHealthChecker(interval time.Duration) *HealthChecker {
    return &HealthChecker{
        checks:   make(map[string]func() bool),
        statuses: make(map[string]bool),
        interval: interval,
    }
}

// Register ลงทะเบียน health check function
func (hc *HealthChecker) Register(name string, check func() bool) {
    hc.mu.Lock()
    defer hc.mu.Unlock()
    hc.checks[name] = check
    hc.statuses[name] = true // assume healthy initially
}

// IsHealthy คืนสถานะของ service
func (hc *HealthChecker) IsHealthy(name string) bool {
    hc.mu.RLock()
    defer hc.mu.RUnlock()
    return hc.statuses[name]
}

// Start เริ่มการตรวจสอบ health อย่างต่อเนื่อง
func (hc *HealthChecker) Start(ctx context.Context) {
    ticker := time.NewTicker(hc.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            hc.runChecks()
        }
    }
}

func (hc *HealthChecker) runChecks() {
    hc.mu.Lock()
    defer hc.mu.Unlock()
    
    for name, check := range hc.checks {
        healthy := check()
        if hc.statuses[name] != healthy {
            if healthy {
                log.Printf("[HealthChecker] %s recovered", name)
            } else {
                log.Printf("[HealthChecker] %s is unhealthy", name)
            }
            hc.statuses[name] = healthy
        }
    }
}

// CircuitBreaker ป้องกัน cascade failures
type CircuitBreakerState int

const (
    StateClosed   CircuitBreakerState = iota // ปกติ
    StateOpen                                // ปิดกั้น requests
    StateHalfOpen                            // ทดสอบ recovery
)

type CircuitBreaker struct {
    mu            sync.Mutex
    state         CircuitBreakerState
    failures      int
    successes     int
    lastFailure   time.Time
    threshold     int
    timeout       time.Duration
    halfOpenLimit int
}

// NewCircuitBreaker สร้าง circuit breaker ใหม่
func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:         StateClosed,
        threshold:     threshold,
        timeout:       timeout,
        halfOpenLimit: 3,
    }
}

// Execute รัน function ผ่าน circuit breaker
func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    
    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            log.Println("[CircuitBreaker] Transitioning to half-open")
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    case StateHalfOpen:
        // อนุญาตให้ผ่านในจำนวนจำกัด
    }
    
    cb.mu.Unlock()
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        
        if cb.state == StateHalfOpen || cb.failures >= cb.threshold {
            cb.state = StateOpen
            log.Printf("[CircuitBreaker] Opened after %d failures", cb.failures)
        }
        return err
    }
    
    // Success
    if cb.state == StateHalfOpen {
        cb.successes++
        if cb.successes >= cb.halfOpenLimit {
            cb.state = StateClosed
            cb.failures = 0
            log.Println("[CircuitBreaker] Closed after recovery")
        }
    } else {
        cb.failures = 0
    }
    
    return nil
}

// GetState คืน state ปัจจุบัน
func (cb *CircuitBreaker) GetState() CircuitBreakerState {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    return cb.state
}

// Retry ลอง execute อีกครั้งเมื่อ fail
type RetryConfig struct {
    MaxAttempts int
    Backoff     time.Duration
    MaxBackoff  time.Duration
    Multiplier  float64
}

// WithRetry execute function พร้อม retry logic
func WithRetry(ctx context.Context, cfg RetryConfig, fn func() error) error {
    var lastErr error
    backoff := cfg.Backoff
    
    for attempt := 1; attempt <= cfg.MaxAttempts; attempt++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        
        err := fn()
        if err == nil {
            return nil
        }
        
        lastErr = err
        log.Printf("[Retry] Attempt %d/%d failed: %v", attempt, cfg.MaxAttempts, err)
        
        if attempt < cfg.MaxAttempts {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(backoff):
            }
            
            backoff = time.Duration(float64(backoff) * cfg.Multiplier)
            if backoff > cfg.MaxBackoff {
                backoff = cfg.MaxBackoff
            }
        }
    }
    
    return fmt.Errorf("all %d attempts failed: %w", cfg.MaxAttempts, lastErr)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Health Checker example
    hc := NewHealthChecker(5 * time.Second)
    
    dbHealthy := true
    hc.Register("database", func() bool {
        return dbHealthy
    })
    
    go hc.Start(ctx)
    
    // Circuit Breaker example
    cb := NewCircuitBreaker(3, 10*time.Second)
    
    failCount := 0
    for i := 0; i < 10; i++ {
        err := cb.Execute(func() error {
            failCount++
            if failCount <= 5 {
                return errors.New("service unavailable")
            }
            return nil
        })
        
        if err != nil {
            fmt.Printf("Request %d: FAILED - %v (state: %d)\n", i+1, err, cb.GetState())
        } else {
            fmt.Printf("Request %d: SUCCESS\n", i+1)
        }
        time.Sleep(100 * time.Millisecond)
    }
    
    // Retry example
    fmt.Println("\n--- Retry Example ---")
    attempts := 0
    err := WithRetry(ctx, RetryConfig{
        MaxAttempts: 5,
        Backoff:     100 * time.Millisecond,
        MaxBackoff:  1 * time.Second,
        Multiplier:  2.0,
    }, func() error {
        attempts++
        if attempts < 3 {
            return fmt.Errorf("temporary error (attempt %d)", attempts)
        }
        return nil
    })
    
    if err != nil {
        fmt.Printf("Final error: %v\n", err)
    } else {
        fmt.Printf("Succeeded after %d attempts\n", attempts)
    }
}
```

---

## 4. Data Replication

```go
package main

import (
    "fmt"
    "log"
    "sync"
    "time"
)

// ReplicationMode กำหนด mode ของ replication
type ReplicationMode int

const (
    SynchronousReplication  ReplicationMode = iota // รอให้ replica ยืนยันก่อน
    AsynchronousReplication                         // ไม่รอ replica
    SemiSynchronousReplication                      // รอ replica อย่างน้อย 1 ตัว
)

// Record ข้อมูลที่จะ replicate
type Record struct {
    Key       string
    Value     interface{}
    Version   int64
    Timestamp time.Time
}

// ReplicaNode แสดง replica node
type ReplicaNode struct {
    mu   sync.RWMutex
    ID   string
    data map[string]*Record
    lag  time.Duration
}

// NewReplicaNode สร้าง replica node ใหม่
func NewReplicaNode(id string, lag time.Duration) *ReplicaNode {
    return &ReplicaNode{
        ID:   id,
        data: make(map[string]*Record),
        lag:  lag,
    }
}

// Apply ใช้ record กับ replica (จำลอง network lag)
func (rn *ReplicaNode) Apply(record *Record) error {
    time.Sleep(rn.lag) // จำลอง network delay
    
    rn.mu.Lock()
    defer rn.mu.Unlock()
    
    existing, exists := rn.data[record.Key]
    if exists && existing.Version >= record.Version {
        return nil // ไม่ต้อง update เพราะมี version ใหม่กว่า
    }
    
    rn.data[record.Key] = record
    log.Printf("[Replica %s] Applied record: key=%s, version=%d", rn.ID, record.Key, record.Version)
    return nil
}

// Get คืนข้อมูลจาก replica
func (rn *ReplicaNode) Get(key string) (*Record, bool) {
    rn.mu.RLock()
    defer rn.mu.RUnlock()
    record, ok := rn.data[key]
    return record, ok
}

// PrimaryNode แสดง primary node
type PrimaryNode struct {
    mu       sync.RWMutex
    data     map[string]*Record
    version  int64
    replicas []*ReplicaNode
    mode     ReplicationMode
}

// NewPrimaryNode สร้าง primary node ใหม่
func NewPrimaryNode(mode ReplicationMode) *PrimaryNode {
    return &PrimaryNode{
        data:     make(map[string]*Record),
        replicas: make([]*ReplicaNode, 0),
        mode:     mode,
    }
}

// AddReplica เพิ่ม replica
func (pn *PrimaryNode) AddReplica(replica *ReplicaNode) {
    pn.mu.Lock()
    defer pn.mu.Unlock()
    pn.replicas = append(pn.replicas, replica)
}

// Write เขียนข้อมูลและ replicate
func (pn *PrimaryNode) Write(key string, value interface{}) error {
    pn.mu.Lock()
    
    pn.version++
    record := &Record{
        Key:       key,
        Value:     value,
        Version:   pn.version,
        Timestamp: time.Now(),
    }
    pn.data[key] = record
    replicas := make([]*ReplicaNode, len(pn.replicas))
    copy(replicas, pn.replicas)
    
    pn.mu.Unlock()
    
    return pn.replicate(record, replicas)
}

// replicate ส่งข้อมูลไป replicas
func (pn *PrimaryNode) replicate(record *Record, replicas []*ReplicaNode) error {
    switch pn.mode {
    case SynchronousReplication:
        return pn.syncReplicate(record, replicas)
    case AsynchronousReplication:
        go pn.asyncReplicate(record, replicas)
        return nil
    case SemiSynchronousReplication:
        return pn.semiSyncReplicate(record, replicas)
    }
    return nil
}

// syncReplicate รอให้ทุก replica ยืนยัน
func (pn *PrimaryNode) syncReplicate(record *Record, replicas []*ReplicaNode) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(replicas))
    
    for _, replica := range replicas {
        wg.Add(1)
        go func(r *ReplicaNode) {
            defer wg.Done()
            if err := r.Apply(record); err != nil {
                errCh <- fmt.Errorf("replica %s: %w", r.ID, err)
            }
        }(replica)
    }
    
    wg.Wait()
    close(errCh)
    
    for err := range errCh {
        return err // คืน error แรก
    }
    return nil
}

// asyncReplicate ส่งข้อมูลแบบ async
func (pn *PrimaryNode) asyncReplicate(record *Record, replicas []*ReplicaNode) {
    for _, replica := range replicas {
        go func(r *ReplicaNode) {
            if err := r.Apply(record); err != nil {
                log.Printf("[Primary] Async replication error to %s: %v", r.ID, err)
            }
        }(replica)
    }
}

// semiSyncReplicate รอให้อย่างน้อย 1 replica ยืนยัน
func (pn *PrimaryNode) semiSyncReplicate(record *Record, replicas []*ReplicaNode) error {
    if len(replicas) == 0 {
        return nil
    }
    
    doneCh := make(chan bool, len(replicas))
    
    for _, replica := range replicas {
        go func(r *ReplicaNode) {
            err := r.Apply(record)
            doneCh <- (err == nil)
        }(replica)
    }
    
    // รอให้อย่างน้อย 1 replica สำเร็จ
    select {
    case success := <-doneCh:
        if !success {
            // ลอง replica อื่น
            select {
            case s := <-doneCh:
                if !s {
                    return fmt.Errorf("all replicas failed")
                }
            case <-time.After(5 * time.Second):
                return fmt.Errorf("timeout waiting for replica")
            }
        }
    case <-time.After(5 * time.Second):
        return fmt.Errorf("timeout waiting for replica")
    }
    
    // ส่ง async ไปยัง replicas ที่เหลือ
    go func() {
        for i := 0; i < len(replicas)-1; i++ {
            <-doneCh
        }
    }()
    
    return nil
}

// Read อ่านข้อมูลจาก primary
func (pn *PrimaryNode) Read(key string) (*Record, bool) {
    pn.mu.RLock()
    defer pn.mu.RUnlock()
    record, ok := pn.data[key]
    return record, ok
}

func main() {
    fmt.Println("=== Data Replication Demo ===")
    
    // สร้าง primary node แบบ sync
    primary := NewPrimaryNode(SynchronousReplication)
    
    // เพิ่ม replicas พร้อม lag ต่างกัน
    replica1 := NewReplicaNode("replica-1", 10*time.Millisecond)
    replica2 := NewReplicaNode("replica-2", 20*time.Millisecond)
    replica3 := NewReplicaNode("replica-3", 50*time.Millisecond)
    
    primary.AddReplica(replica1)
    primary.AddReplica(replica2)
    primary.AddReplica(replica3)
    
    // เขียนข้อมูล
    start := time.Now()
    if err := primary.Write("user:1", map[string]string{"name": "สมชาย", "email": "somchai@example.com"}); err != nil {
        log.Fatalf("Write failed: %v", err)
    }
    fmt.Printf("Synchronous write completed in %v\n", time.Since(start))
    
    // ตรวจสอบว่า replicas มีข้อมูล
    for _, replica := range []*ReplicaNode{replica1, replica2, replica3} {
        if record, ok := replica.Get("user:1"); ok {
            fmt.Printf("Replica %s has data: version=%d\n", replica.ID, record.Version)
        }
    }
    
    // ทดสอบ async replication
    fmt.Println("\n--- Async Replication ---")
    asyncPrimary := NewPrimaryNode(AsynchronousReplication)
    asyncReplica := NewReplicaNode("async-replica", 100*time.Millisecond)
    asyncPrimary.AddReplica(asyncReplica)
    
    start = time.Now()
    asyncPrimary.Write("key:1", "value1")
    fmt.Printf("Async write returned in %v\n", time.Since(start))
    
    // รอ async replication
    time.Sleep(200 * time.Millisecond)
    if record, ok := asyncReplica.Get("key:1"); ok {
        fmt.Printf("Async replica received data: version=%d\n", record.Version)
    }
}
```

---

## 5. Multi-Region Deployments

```go
package main

import (
    "fmt"
    "math"
    "sync"
    "time"
)

// Region แสดง geographic region
type Region struct {
    Name     string
    Location string
    Latency  time.Duration // latency ไปยัง region นี้
}

// RegionalCluster จัดการ cluster ใน region
type RegionalCluster struct {
    mu      sync.RWMutex
    Region  Region
    Nodes   []*Node
    Primary bool
}

// MultiRegionManager จัดการ multi-region deployment
type MultiRegionManager struct {
    mu       sync.RWMutex
    clusters map[string]*RegionalCluster
    routing  RoutingStrategy
}

// RoutingStrategy กำหนดวิธีการ routing
type RoutingStrategy interface {
    SelectRegion(clusters map[string]*RegionalCluster, clientLocation string) *RegionalCluster
}

// LatencyBasedRouting เลือก region ตาม latency
type LatencyBasedRouting struct{}

func (r *LatencyBasedRouting) SelectRegion(clusters map[string]*RegionalCluster, clientLocation string) *RegionalCluster {
    var selected *RegionalCluster
    var minLatency time.Duration = math.MaxInt64
    
    for _, cluster := range clusters {
        if len(cluster.Nodes) > 0 && cluster.Region.Latency < minLatency {
            minLatency = cluster.Region.Latency
            selected = cluster
        }
    }
    return selected
}

// GeographicRouting เลือก region ตาม geography
type GeographicRouting struct {
    regionMap map[string]string // client location -> preferred region
}

func NewGeographicRouting() *GeographicRouting {
    return &GeographicRouting{
        regionMap: map[string]string{
            "TH": "ap-southeast-1", // Thailand -> Singapore
            "JP": "ap-northeast-1", // Japan -> Tokyo
            "US": "us-east-1",      // US -> US East
            "EU": "eu-west-1",      // Europe -> Ireland
        },
    }
}

func (r *GeographicRouting) SelectRegion(clusters map[string]*RegionalCluster, clientLocation string) *RegionalCluster {
    if preferredRegion, ok := r.regionMap[clientLocation]; ok {
        if cluster, ok := clusters[preferredRegion]; ok {
            return cluster
        }
    }
    // Fallback: latency based
    latencyRouting := &LatencyBasedRouting{}
    return latencyRouting.SelectRegion(clusters, clientLocation)
}

// NewMultiRegionManager สร้าง manager ใหม่
func NewMultiRegionManager(routing RoutingStrategy) *MultiRegionManager {
    return &MultiRegionManager{
        clusters: make(map[string]*RegionalCluster),
        routing:  routing,
    }
}

// AddRegion เพิ่ม region
func (m *MultiRegionManager) AddRegion(region Region, isPrimary bool) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    m.clusters[region.Name] = &RegionalCluster{
        Region:  region,
        Nodes:   make([]*Node, 0),
        Primary: isPrimary,
    }
}

// Route เลือก region ที่เหมาะสม
func (m *MultiRegionManager) Route(clientLocation string) *RegionalCluster {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.routing.SelectRegion(m.clusters, clientLocation)
}

// GetPrimaryRegion คืน primary region
func (m *MultiRegionManager) GetPrimaryRegion() *RegionalCluster {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    for _, cluster := range m.clusters {
        if cluster.Primary {
            return cluster
        }
    }
    return nil
}

// Node struct สำหรับ example นี้
type Node struct {
    ID      string
    Address string
    Active  bool
}

func main() {
    // ใช้ Geographic routing
    routing := NewGeographicRouting()
    manager := NewMultiRegionManager(routing)
    
    // เพิ่ม regions
    manager.AddRegion(Region{
        Name:     "ap-southeast-1",
        Location: "Singapore",
        Latency:  30 * time.Millisecond,
    }, true)
    
    manager.AddRegion(Region{
        Name:     "ap-northeast-1",
        Location: "Tokyo",
        Latency:  50 * time.Millisecond,
    }, false)
    
    manager.AddRegion(Region{
        Name:     "us-east-1",
        Location: "N. Virginia",
        Latency:  200 * time.Millisecond,
    }, false)
    
    manager.AddRegion(Region{
        Name:     "eu-west-1",
        Location: "Ireland",
        Latency:  250 * time.Millisecond,
    }, false)
    
    // ทดสอบ routing
    clients := []string{"TH", "JP", "US", "EU", "AU"}
    
    fmt.Println("=== Multi-Region Routing ===")
    for _, client := range clients {
        cluster := manager.Route(client)
        if cluster != nil {
            fmt.Printf("Client from %s -> Region: %s (%s)\n", 
                client, cluster.Region.Name, cluster.Region.Location)
        } else {
            fmt.Printf("Client from %s -> No region available\n", client)
        }
    }
    
    primary := manager.GetPrimaryRegion()
    if primary != nil {
        fmt.Printf("\nPrimary Region: %s\n", primary.Region.Name)
    }
}
```

---

## 6. Disaster Recovery

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "time"
)

// RPO Recovery Point Objective - ข้อมูลสูงสุดที่ยอมสูญเสีย
// RTO Recovery Time Objective - เวลาสูงสุดในการ recovery

// BackupType ประเภทของ backup
type BackupType string

const (
    FullBackup        BackupType = "full"
    IncrementalBackup BackupType = "incremental"
    DifferentialBackup BackupType = "differential"
)

// BackupMetadata ข้อมูล metadata ของ backup
type BackupMetadata struct {
    ID          string     `json:"id"`
    Type        BackupType `json:"type"`
    CreatedAt   time.Time  `json:"created_at"`
    SizeBytes   int64      `json:"size_bytes"`
    Checksum    string     `json:"checksum"`
    ParentID    string     `json:"parent_id,omitempty"` // สำหรับ incremental/differential
    Location    string     `json:"location"`
}

// DisasterRecoveryPlan แผน DR
type DisasterRecoveryPlan struct {
    Name           string        `json:"name"`
    RPO            time.Duration `json:"rpo"`
    RTO            time.Duration `json:"rto"`
    BackupSchedule string        `json:"backup_schedule"` // cron expression
    BackupRetention int          `json:"backup_retention_days"`
    PrimaryRegion  string        `json:"primary_region"`
    DRRegion       string        `json:"dr_region"`
    Runbook        []RunbookStep `json:"runbook"`
}

// RunbookStep ขั้นตอนใน runbook
type RunbookStep struct {
    Order       int    `json:"order"`
    Name        string `json:"name"`
    Description string `json:"description"`
    Command     string `json:"command,omitempty"`
    Timeout     string `json:"timeout"`
    Owner       string `json:"owner"`
}

// BackupManager จัดการ backups
type BackupManager struct {
    backups  []BackupMetadata
    plan     *DisasterRecoveryPlan
    location string
}

// NewBackupManager สร้าง backup manager ใหม่
func NewBackupManager(plan *DisasterRecoveryPlan, location string) *BackupManager {
    return &BackupManager{
        backups:  make([]BackupMetadata, 0),
        plan:     plan,
        location: location,
    }
}

// CreateBackup สร้าง backup ใหม่
func (bm *BackupManager) CreateBackup(bType BackupType) (*BackupMetadata, error) {
    var parentID string
    
    // สำหรับ incremental ต้องหา parent backup
    if bType == IncrementalBackup || bType == DifferentialBackup {
        lastFull := bm.getLastFullBackup()
        if lastFull == nil {
            // ถ้าไม่มี full backup ให้ทำ full แทน
            bType = FullBackup
        } else if bType == IncrementalBackup {
            // สำหรับ incremental ใช้ backup ล่าสุด
            lastBackup := bm.getLastBackup()
            if lastBackup != nil {
                parentID = lastBackup.ID
            }
        } else {
            // สำหรับ differential ใช้ full backup ล่าสุด
            parentID = lastFull.ID
        }
    }
    
    backup := &BackupMetadata{
        ID:        fmt.Sprintf("backup-%d", time.Now().Unix()),
        Type:      bType,
        CreatedAt: time.Now(),
        SizeBytes: bm.estimateSize(bType),
        Checksum:  fmt.Sprintf("sha256:%d", time.Now().UnixNano()),
        ParentID:  parentID,
        Location:  bm.location,
    }
    
    bm.backups = append(bm.backups, *backup)
    fmt.Printf("[Backup] Created %s backup: %s\n", bType, backup.ID)
    return backup, nil
}

// estimateSize ประเมินขนาด backup
func (bm *BackupManager) estimateSize(bType BackupType) int64 {
    switch bType {
    case FullBackup:
        return 10 * 1024 * 1024 * 1024 // 10 GB
    case IncrementalBackup:
        return 100 * 1024 * 1024 // 100 MB
    case DifferentialBackup:
        return 500 * 1024 * 1024 // 500 MB
    }
    return 0
}

// getLastFullBackup คืน full backup ล่าสุด
func (bm *BackupManager) getLastFullBackup() *BackupMetadata {
    for i := len(bm.backups) - 1; i >= 0; i-- {
        if bm.backups[i].Type == FullBackup {
            return &bm.backups[i]
        }
    }
    return nil
}

// getLastBackup คืน backup ล่าสุด
func (bm *BackupManager) getLastBackup() *BackupMetadata {
    if len(bm.backups) == 0 {
        return nil
    }
    return &bm.backups[len(bm.backups)-1]
}

// CalculateRecoveryChain คำนวณ chain ที่ต้องใช้ใน recovery
func (bm *BackupManager) CalculateRecoveryChain() []BackupMetadata {
    chain := make([]BackupMetadata, 0)
    
    lastBackup := bm.getLastBackup()
    if lastBackup == nil {
        return chain
    }
    
    // สร้าง map สำหรับการค้นหา
    backupMap := make(map[string]*BackupMetadata)
    for i := range bm.backups {
        backupMap[bm.backups[i].ID] = &bm.backups[i]
    }
    
    // ย้อนกลับจาก backup ล่าสุด
    current := lastBackup
    for current != nil {
        chain = append([]BackupMetadata{*current}, chain...)
        if current.ParentID == "" {
            break
        }
        current = backupMap[current.ParentID]
    }
    
    return chain
}

// PrintDRPlan แสดง DR plan
func PrintDRPlan(plan *DisasterRecoveryPlan) {
    data, _ := json.MarshalIndent(plan, "", "  ")
    fmt.Println(string(data))
}

func main() {
    // สร้าง DR plan
    plan := &DisasterRecoveryPlan{
        Name:            "Production DR Plan",
        RPO:             1 * time.Hour,
        RTO:             4 * time.Hour,
        BackupSchedule:  "0 */6 * * *", // ทุก 6 ชั่วโมง
        BackupRetention: 30,
        PrimaryRegion:   "ap-southeast-1",
        DRRegion:        "ap-northeast-1",
        Runbook: []RunbookStep{
            {
                Order:       1,
                Name:        "Assess Situation",
                Description: "ประเมินสถานการณ์และตัดสินใจว่าจะ failover หรือไม่",
                Timeout:     "30m",
                Owner:       "On-Call Engineer",
            },
            {
                Order:       2,
                Name:        "Notify Stakeholders",
                Description: "แจ้ง stakeholders เกี่ยวกับ incident",
                Command:     "notify-stakeholders --severity high",
                Timeout:     "15m",
                Owner:       "Incident Commander",
            },
            {
                Order:       3,
                Name:        "Activate DR Region",
                Description: "เปิดใช้งาน DR region",
                Command:     "terraform apply -var region=ap-northeast-1",
                Timeout:     "2h",
                Owner:       "Infrastructure Team",
            },
            {
                Order:       4,
                Name:        "Restore from Backup",
                Description: "restore ข้อมูลจาก backup ล่าสุด",
                Command:     "restore-from-backup --backup-id latest",
                Timeout:     "1h",
                Owner:       "Database Team",
            },
            {
                Order:       5,
                Name:        "Update DNS",
                Description: "อัปเดต DNS records ให้ชี้ไปที่ DR region",
                Command:     "update-dns --region ap-northeast-1",
                Timeout:     "30m",
                Owner:       "Network Team",
            },
            {
                Order:       6,
                Name:        "Verify Services",
                Description: "ตรวจสอบว่า services ทำงานได้ปกติ",
                Command:     "run-smoke-tests",
                Timeout:     "30m",
                Owner:       "QA Team",
            },
        },
    }
    
    fmt.Println("=== Disaster Recovery Plan ===")
    PrintDRPlan(plan)
    
    // จำลอง backup schedule
    fmt.Println("\n=== Backup Schedule Simulation ===")
    bm := NewBackupManager(plan, "s3://backup-bucket/production")
    
    // สร้าง backup sequence
    bm.CreateBackup(FullBackup)
    bm.CreateBackup(IncrementalBackup)
    bm.CreateBackup(IncrementalBackup)
    bm.CreateBackup(DifferentialBackup)
    bm.CreateBackup(IncrementalBackup)
    
    // คำนวณ recovery chain
    chain := bm.CalculateRecoveryChain()
    fmt.Printf("\nRecovery chain (%d backups):\n", len(chain))
    for i, backup := range chain {
        fmt.Printf("  %d. %s (%s) - %.2f GB\n", 
            i+1, backup.ID, backup.Type, float64(backup.SizeBytes)/(1024*1024*1024))
    }
    
    // ตรวจสอบว่า RPO ถูกต้อง
    fmt.Printf("\nRPO: %v\n", plan.RPO)
    fmt.Printf("RTO: %v\n", plan.RTO)
    
    _ = os.Getenv("BACKUP_LOCATION")
}
```

---

## 7. Chaos Engineering Basics

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// ChaosExperiment แสดง chaos experiment
type ChaosExperiment struct {
    Name        string
    Description string
    Hypothesis  string        // สิ่งที่เราคาดว่าจะเกิดขึ้น
    BlastRadius string        // scope ของผลกระทบ
    Duration    time.Duration
    Rollback    func() error
}

// FaultInjector ฉีด faults เข้าระบบ
type FaultInjector struct {
    mu       sync.Mutex
    active   bool
    faults   []FaultConfig
}

// FaultConfig กำหนด fault ที่จะฉีด
type FaultConfig struct {
    Name        string
    Type        FaultType
    Probability float64 // 0.0 - 1.0
    Latency     time.Duration
    ErrorRate   float64
}

// FaultType ประเภทของ fault
type FaultType int

const (
    LatencyFault FaultType = iota
    ErrorFault
    PacketLossFault
    MemoryFault
    CPUFault
)

// NewFaultInjector สร้าง fault injector ใหม่
func NewFaultInjector() *FaultInjector {
    return &FaultInjector{
        faults: make([]FaultConfig, 0),
    }
}

// AddFault เพิ่ม fault configuration
func (fi *FaultInjector) AddFault(fault FaultConfig) {
    fi.mu.Lock()
    defer fi.mu.Unlock()
    fi.faults = append(fi.faults, fault)
}

// Start เปิดใช้งาน fault injection
func (fi *FaultInjector) Start() {
    fi.mu.Lock()
    defer fi.mu.Unlock()
    fi.active = true
    fmt.Println("[ChaosEngine] Fault injection ENABLED")
}

// Stop ปิดใช้งาน fault injection
func (fi *FaultInjector) Stop() {
    fi.mu.Lock()
    defer fi.mu.Unlock()
    fi.active = false
    fmt.Println("[ChaosEngine] Fault injection DISABLED")
}

// Intercept ดัก request และฉีด fault
func (fi *FaultInjector) Intercept(ctx context.Context, fn func() error) error {
    fi.mu.Lock()
    active := fi.active
    faults := make([]FaultConfig, len(fi.faults))
    copy(faults, fi.faults)
    fi.mu.Unlock()
    
    if !active {
        return fn()
    }
    
    // ลองแต่ละ fault
    for _, fault := range faults {
        if rand.Float64() < fault.Probability {
            switch fault.Type {
            case LatencyFault:
                fmt.Printf("[Chaos] Injecting %v latency\n", fault.Latency)
                select {
                case <-time.After(fault.Latency):
                case <-ctx.Done():
                    return ctx.Err()
                }
            case ErrorFault:
                if rand.Float64() < fault.ErrorRate {
                    fmt.Printf("[Chaos] Injecting error\n")
                    return fmt.Errorf("chaos: injected error from fault '%s'", fault.Name)
                }
            case PacketLossFault:
                if rand.Float64() < fault.Probability {
                    fmt.Printf("[Chaos] Simulating packet loss\n")
                    return fmt.Errorf("chaos: packet loss")
                }
            }
        }
    }
    
    return fn()
}

// ChaosRunner รัน chaos experiments
type ChaosRunner struct {
    experiments []*ChaosExperiment
    injector    *FaultInjector
}

// NewChaosRunner สร้าง chaos runner ใหม่
func NewChaosRunner(injector *FaultInjector) *ChaosRunner {
    return &ChaosRunner{
        experiments: make([]*ChaosExperiment, 0),
        injector:    injector,
    }
}

// AddExperiment เพิ่ม experiment
func (cr *ChaosRunner) AddExperiment(exp *ChaosExperiment) {
    cr.experiments = append(cr.experiments, exp)
}

// Run รัน experiment
func (cr *ChaosRunner) Run(ctx context.Context, exp *ChaosExperiment) error {
    fmt.Printf("\n=== Starting Chaos Experiment: %s ===\n", exp.Name)
    fmt.Printf("Hypothesis: %s\n", exp.Hypothesis)
    fmt.Printf("Blast Radius: %s\n", exp.BlastRadius)
    fmt.Printf("Duration: %v\n", exp.Duration)
    
    cr.injector.Start()
    
    // สร้าง context ที่หมดเวลาตาม experiment duration
    expCtx, cancel := context.WithTimeout(ctx, exp.Duration)
    defer cancel()
    
    // รัน experiment
    <-expCtx.Done()
    
    cr.injector.Stop()
    
    // รัน rollback ถ้ามี
    if exp.Rollback != nil {
        fmt.Println("Running rollback...")
        if err := exp.Rollback(); err != nil {
            return fmt.Errorf("rollback failed: %w", err)
        }
        fmt.Println("Rollback completed")
    }
    
    fmt.Printf("=== Experiment '%s' Completed ===\n", exp.Name)
    return nil
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    // สร้าง fault injector
    injector := NewFaultInjector()
    injector.AddFault(FaultConfig{
        Name:        "network-latency",
        Type:        LatencyFault,
        Probability: 0.3,
        Latency:     500 * time.Millisecond,
    })
    injector.AddFault(FaultConfig{
        Name:        "service-error",
        Type:        ErrorFault,
        Probability: 0.2,
        ErrorRate:   0.5,
    })
    
    // สร้าง chaos runner
    runner := NewChaosRunner(injector)
    
    // สร้าง experiment
    exp := &ChaosExperiment{
        Name:        "Database Latency Experiment",
        Description: "ทดสอบว่าระบบทำงานได้เมื่อ database มี latency สูง",
        Hypothesis:  "ระบบควรยังคง serve requests ได้ แต่อาจมี timeout บางส่วน",
        BlastRadius: "10% ของ production requests",
        Duration:    2 * time.Second,
        Rollback: func() error {
            fmt.Println("Restoring normal database latency")
            return nil
        },
    }
    
    runner.AddExperiment(exp)
    
    ctx := context.Background()
    
    // จำลองการรับ requests ระหว่าง experiment
    go func() {
        for i := 0; i < 20; i++ {
            time.Sleep(100 * time.Millisecond)
            
            err := injector.Intercept(ctx, func() error {
                // จำลอง database call
                return nil
            })
            
            if err != nil {
                fmt.Printf("Request %d: FAILED - %v\n", i+1, err)
            } else {
                fmt.Printf("Request %d: SUCCESS\n", i+1)
            }
        }
    }()
    
    // รัน experiment
    if err := runner.Run(ctx, exp); err != nil {
        fmt.Printf("Experiment failed: %v\n", err)
    }
}
```

---

## 8. Workshop: HA System Implementation

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "sync/atomic"
    "time"
)

// HAConfig การตั้งค่าของระบบ HA
type HAConfig struct {
    HealthCheckInterval time.Duration
    FailoverThreshold   int
    RecoveryTimeout     time.Duration
    MaxRetries          int
}

// DefaultHAConfig ค่า default สำหรับ HA
func DefaultHAConfig() HAConfig {
    return HAConfig{
        HealthCheckInterval: 5 * time.Second,
        FailoverThreshold:   3,
        RecoveryTimeout:     30 * time.Second,
        MaxRetries:          3,
    }
}

// ServiceState สถานะของ service
type ServiceState int32

const (
    StateRunning  ServiceState = iota
    StateDegraded              // ทำงานได้แต่ไม่สมบูรณ์
    StateFailed
    StateRecovering
)

// HAService service ที่รองรับ HA
type HAService struct {
    mu            sync.RWMutex
    name          string
    state         int32 // ServiceState
    failureCount  int
    lastFailure   time.Time
    config        HAConfig
    onFailover    func()
    onRecovery    func()
    metrics       ServiceMetrics
}

// ServiceMetrics metrics ของ service
type ServiceMetrics struct {
    TotalRequests  int64
    SuccessCount   int64
    FailureCount   int64
    AvgLatencyMs   float64
    P99LatencyMs   float64
    Uptime         time.Duration
    StartTime      time.Time
}

// NewHAService สร้าง HA service ใหม่
func NewHAService(name string, config HAConfig) *HAService {
    return &HAService{
        name:    name,
        state:   int32(StateRunning),
        config:  config,
        metrics: ServiceMetrics{StartTime: time.Now()},
    }
}

// OnFailover กำหนด callback เมื่อ failover
func (s *HAService) OnFailover(fn func()) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.onFailover = fn
}

// OnRecovery กำหนด callback เมื่อ recover
func (s *HAService) OnRecovery(fn func()) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.onRecovery = fn
}

// Execute รัน operation พร้อม HA
func (s *HAService) Execute(ctx context.Context, op func() error) error {
    start := time.Now()
    atomic.AddInt64(&s.metrics.TotalRequests, 1)
    
    err := op()
    latency := time.Since(start).Milliseconds()
    
    if err != nil {
        atomic.AddInt64(&s.metrics.FailureCount, 1)
        s.handleFailure(err)
        return err
    }
    
    atomic.AddInt64(&s.metrics.SuccessCount, 1)
    s.updateLatency(float64(latency))
    s.handleSuccess()
    return nil
}

func (s *HAService) handleFailure(err error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    s.failureCount++
    s.lastFailure = time.Now()
    
    if s.failureCount >= s.config.FailoverThreshold {
        if ServiceState(atomic.LoadInt32(&s.state)) != StateFailed {
            atomic.StoreInt32(&s.state, int32(StateFailed))
            log.Printf("[HA] Service %s FAILED after %d failures: %v", s.name, s.failureCount, err)
            
            if s.onFailover != nil {
                go s.onFailover()
            }
            
            // เริ่ม recovery goroutine
            go s.startRecovery()
        }
    }
}

func (s *HAService) handleSuccess() {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if s.failureCount > 0 {
        s.failureCount--
    }
}

func (s *HAService) startRecovery() {
    atomic.StoreInt32(&s.state, int32(StateRecovering))
    log.Printf("[HA] Starting recovery for service %s", s.name)
    
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    timeout := time.After(s.config.RecoveryTimeout)
    
    for {
        select {
        case <-timeout:
            log.Printf("[HA] Recovery timeout for service %s", s.name)
            atomic.StoreInt32(&s.state, int32(StateFailed))
            return
        case <-ticker.C:
            // ลอง health check
            if s.healthCheck() {
                s.mu.Lock()
                s.failureCount = 0
                s.mu.Unlock()
                
                atomic.StoreInt32(&s.state, int32(StateRunning))
                log.Printf("[HA] Service %s recovered", s.name)
                
                if s.onRecovery != nil {
                    go s.onRecovery()
                }
                return
            }
        }
    }
}

func (s *HAService) healthCheck() bool {
    // จำลอง health check
    return time.Since(s.lastFailure) > 10*time.Second
}

func (s *HAService) updateLatency(latencyMs float64) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    total := s.metrics.TotalRequests
    if total > 0 {
        s.metrics.AvgLatencyMs = (s.metrics.AvgLatencyMs*float64(total-1) + latencyMs) / float64(total)
    }
}

// GetMetrics คืน metrics ปัจจุบัน
func (s *HAService) GetMetrics() ServiceMetrics {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    metrics := s.metrics
    metrics.Uptime = time.Since(metrics.StartTime)
    return metrics
}

// GetState คืน state ปัจจุบัน
func (s *HAService) GetState() ServiceState {
    return ServiceState(atomic.LoadInt32(&s.state))
}

func main() {
    config := DefaultHAConfig()
    config.FailoverThreshold = 3
    config.RecoveryTimeout = 15 * time.Second
    
    svc := NewHAService("payment-service", config)
    
    var failoverCount int
    var recoveryCount int
    
    svc.OnFailover(func() {
        failoverCount++
        fmt.Printf("[Alert] FAILOVER triggered! Count: %d\n", failoverCount)
        // ในระบบจริง: ส่ง alert, redirect traffic ไป backup
    })
    
    svc.OnRecovery(func() {
        recoveryCount++
        fmt.Printf("[Alert] Service RECOVERED! Count: %d\n", recoveryCount)
        // ในระบบจริง: update DNS, redirect traffic กลับ
    })
    
    ctx := context.Background()
    
    // จำลอง requests ที่มีทั้ง success และ failure
    fmt.Println("=== HA Service Demo ===")
    
    for i := 0; i < 20; i++ {
        reqNum := i + 1
        err := svc.Execute(ctx, func() error {
            // จำลอง operation ที่ fail บางครั้ง
            if reqNum >= 5 && reqNum <= 8 {
                return fmt.Errorf("service temporarily unavailable")
            }
            time.Sleep(10 * time.Millisecond)
            return nil
        })
        
        state := svc.GetState()
        if err != nil {
            fmt.Printf("Request %d: FAILED (state: %d) - %v\n", reqNum, state, err)
        } else {
            fmt.Printf("Request %d: SUCCESS (state: %d)\n", reqNum, state)
        }
        
        time.Sleep(500 * time.Millisecond)
    }
    
    // รอให้ recovery เกิดขึ้น
    time.Sleep(5 * time.Second)
    
    metrics := svc.GetMetrics()
    fmt.Printf("\n=== Service Metrics ===\n")
    fmt.Printf("Total Requests: %d\n", metrics.TotalRequests)
    fmt.Printf("Success: %d\n", metrics.SuccessCount)
    fmt.Printf("Failures: %d\n", metrics.FailureCount)
    fmt.Printf("Avg Latency: %.2f ms\n", metrics.AvgLatencyMs)
    fmt.Printf("Uptime: %v\n", metrics.Uptime.Round(time.Second))
    fmt.Printf("Failovers: %d\n", failoverCount)
    fmt.Printf("Recoveries: %d\n", recoveryCount)
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HA System Design** - หลักการออกแบบระบบที่มี availability สูง
2. **Redundancy Strategies** - Active-Active, Active-Passive, load balancing algorithms
3. **Failover Mechanisms** - Health checking, circuit breaker, retry with backoff
4. **Data Replication** - Synchronous, asynchronous, semi-synchronous replication
5. **Multi-Region Deployments** - Geographic routing, latency-based routing
6. **Disaster Recovery** - RPO/RTO objectives, backup strategies, DR runbooks
7. **Chaos Engineering** - Fault injection, blast radius management

### Key Takeaways

- **Design for failure**: สมมติว่าทุกอย่างจะ fail และออกแบบให้ระบบ handle ได้
- **Measure availability**: ใช้ SLI/SLO ในการวัดและตั้ง target
- **Test your DR plan**: DR plan ที่ไม่เคย test คือ plan ที่ใช้ไม่ได้
- **Automate failover**: Manual failover ช้าเกินไปสำหรับ 99.99% SLA
- **Practice chaos**: ใช้ chaos engineering เพื่อค้นหาจุดอ่อนก่อน production incident
