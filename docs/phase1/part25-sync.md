# Part 25: Synchronization ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `sync.Mutex` และ `sync.RWMutex` ป้องกัน race conditions
- ใช้ `sync.WaitGroup` รอ goroutines
- ใช้ `sync.Once` สำหรับ one-time initialization
- ใช้ `sync.Map` สำหรับ concurrent map
- ใช้ `sync.Pool` สำหรับ object reuse
- ใช้ `sync/atomic` สำหรับ atomic operations
- ใช้ `sync.Cond` สำหรับ condition variables
- ตรวจจับและป้องกัน race conditions และ deadlocks

---

## 25.1 sync.Mutex

### 25.1.1 Mutex พื้นฐาน

```go
package main

import (
    "fmt"
    "sync"
)

// Counter ที่ไม่ thread-safe
type UnsafeCounter struct {
    count int
}

func (c *UnsafeCounter) Inc() {
    c.count++ // race condition!
}

func (c *UnsafeCounter) Value() int {
    return c.count
}

// Counter ที่ thread-safe ด้วย Mutex
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func (c *SafeCounter) Add(n int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count += n
}

func (c *SafeCounter) Reset() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count = 0
}

func main() {
    counter := &SafeCounter{}
    
    var wg sync.WaitGroup
    
    // เพิ่ม counter จาก 1000 goroutines
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Inc()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Final count: %d (expected 1000)\n", counter.Value())
    
    // เพิ่มหลายค่าพร้อมกัน
    for i := 0; i < 10; i++ {
        wg.Add(1)
        n := i
        go func() {
            defer wg.Done()
            counter.Add(n)
        }()
    }
    
    wg.Wait()
    fmt.Printf("After adding 0-9: %d (expected 1045)\n", counter.Value())
}
```

### 25.1.2 RWMutex

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Cache ที่อ่านบ่อยกว่าเขียน
type Cache struct {
    mu    sync.RWMutex
    store map[string]string
    hits  int
    misses int
}

func NewCache() *Cache {
    return &Cache{
        store: make(map[string]string),
    }
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock() // หลาย goroutine อ่านพร้อมกันได้
    defer c.mu.RUnlock()
    
    val, ok := c.store[key]
    if ok {
        c.hits++
    } else {
        c.misses++
    }
    return val, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock() // เขียน exclusive lock
    defer c.mu.Unlock()
    
    c.store[key] = value
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    delete(c.store, key)
}

func (c *Cache) Stats() (hits, misses int) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.hits, c.misses
}

func (c *Cache) Len() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.store)
}

func main() {
    cache := NewCache()
    
    // เติมข้อมูล
    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("key%d", i)
        val := fmt.Sprintf("value%d", i)
        cache.Set(key, val)
    }
    
    var wg sync.WaitGroup
    
    // Readers (10 goroutines)
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 50; j++ {
                key := fmt.Sprintf("key%d", j)
                if val, ok := cache.Get(key); ok {
                    _ = val
                }
                time.Sleep(time.Millisecond)
            }
            fmt.Printf("Reader %d done\n", id)
        }(i)
    }
    
    // Writers (2 goroutines)
    for i := 0; i < 2; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 10; j++ {
                key := fmt.Sprintf("new-key%d-%d", id, j)
                cache.Set(key, fmt.Sprintf("new-value%d-%d", id, j))
                time.Sleep(5 * time.Millisecond)
            }
            fmt.Printf("Writer %d done\n", id)
        }(i)
    }
    
    wg.Wait()
    
    hits, misses := cache.Stats()
    fmt.Printf("\nCache stats:\n")
    fmt.Printf("  Size: %d\n", cache.Len())
    fmt.Printf("  Hits: %d, Misses: %d\n", hits, misses)
}
```

---

## 25.2 sync.WaitGroup

### 25.2.1 WaitGroup พื้นฐาน

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Duration(id*100) * time.Millisecond)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }
    
    wg.Wait()
    fmt.Println("All workers completed")
    
    // ตัวอย่าง: batch processing
    fmt.Println("\n=== Batch Processing ===")
    
    items := []string{"a", "b", "c", "d", "e", "f", "g", "h"}
    results := make([]string, len(items))
    
    var mu sync.Mutex
    
    for i, item := range items {
        wg.Add(1)
        i, item := i, item
        
        go func() {
            defer wg.Done()
            
            // simulate processing
            processed := fmt.Sprintf("processed_%s", item)
            
            mu.Lock()
            results[i] = processed
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    fmt.Println("Results:", results)
}
```

### 25.2.2 WaitGroup กับ Error Handling

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Task struct {
    ID   int
    Name string
}

type Result struct {
    TaskID int
    Output string
    Error  error
}

func processTask(ctx context.Context, task Task) Result {
    // ตรวจ context
    select {
    case <-ctx.Done():
        return Result{TaskID: task.ID, Error: ctx.Err()}
    default:
    }
    
    // simulate work
    time.Sleep(100 * time.Millisecond)
    
    if task.ID%3 == 0 {
        return Result{TaskID: task.ID, Error: fmt.Errorf("task %d failed", task.ID)}
    }
    
    return Result{
        TaskID: task.ID,
        Output: fmt.Sprintf("output from %s", task.Name),
    }
}

func runTasks(ctx context.Context, tasks []Task, concurrency int) []Result {
    results := make([]Result, len(tasks))
    
    // Semaphore channel สำหรับ limit concurrency
    sem := make(chan struct{}, concurrency)
    
    var wg sync.WaitGroup
    
    for i, task := range tasks {
        wg.Add(1)
        i, task := i, task
        
        go func() {
            defer wg.Done()
            
            sem <- struct{}{}        // acquire
            defer func() { <-sem }() // release
            
            results[i] = processTask(ctx, task)
        }()
    }
    
    wg.Wait()
    return results
}

func main() {
    tasks := make([]Task, 10)
    for i := range tasks {
        tasks[i] = Task{ID: i + 1, Name: fmt.Sprintf("task-%d", i+1)}
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    fmt.Println("Running 10 tasks with concurrency 3...")
    start := time.Now()
    
    results := runTasks(ctx, tasks, 3)
    
    fmt.Printf("Completed in: %v\n\n", time.Since(start))
    
    var successes, failures int
    for _, r := range results {
        if r.Error != nil {
            fmt.Printf("Task %d: ERROR - %v\n", r.TaskID, r.Error)
            failures++
        } else {
            fmt.Printf("Task %d: %s\n", r.TaskID, r.Output)
            successes++
        }
    }
    
    fmt.Printf("\nSummary: %d succeeded, %d failed\n", successes, failures)
}
```

---

## 25.3 sync.Once

### 25.3.1 One-time Initialization

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Singleton pattern ด้วย sync.Once
type Database struct {
    connString string
    connected  bool
}

func (db *Database) Query(sql string) string {
    return fmt.Sprintf("Result of: %s", sql)
}

var (
    dbInstance *Database
    dbOnce     sync.Once
)

func GetDB() *Database {
    dbOnce.Do(func() {
        fmt.Println("Initializing database connection...")
        time.Sleep(100 * time.Millisecond) // simulate connection setup
        dbInstance = &Database{
            connString: "postgres://localhost/mydb",
            connected:  true,
        }
        fmt.Println("Database initialized!")
    })
    return dbInstance
}

// Config Singleton
type Config struct {
    Host    string
    Port    int
    Debug   bool
    Timeout time.Duration
}

type ConfigLoader struct {
    once   sync.Once
    config *Config
    err    error
}

func (cl *ConfigLoader) Load() (*Config, error) {
    cl.once.Do(func() {
        fmt.Println("Loading config...")
        // simulate loading from file/env
        cl.config = &Config{
            Host:    "localhost",
            Port:    8080,
            Debug:   true,
            Timeout: 30 * time.Second,
        }
    })
    return cl.config, cl.err
}

func main() {
    var wg sync.WaitGroup
    
    // หลาย goroutines เรียก GetDB พร้อมกัน
    fmt.Println("Starting 5 goroutines all calling GetDB...")
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            db := GetDB()
            result := db.Query(fmt.Sprintf("SELECT * FROM users WHERE id=%d", id))
            fmt.Printf("Goroutine %d: %s\n", id, result)
        }(i)
    }
    
    wg.Wait()
    
    // ตรวจสอบว่าเป็น instance เดียวกัน
    db1 := GetDB()
    db2 := GetDB()
    fmt.Printf("\nSame instance: %v\n", db1 == db2)
    
    // Config loader
    fmt.Println("\n=== Config Loader ===")
    loader := &ConfigLoader{}
    
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cfg, _ := loader.Load()
            fmt.Printf("Goroutine %d: host=%s, port=%d\n", id, cfg.Host, cfg.Port)
        }(i)
    }
    
    wg.Wait()
}
```

---

## 25.4 sync.Map

### 25.4.1 Concurrent Map

```go
package main

import (
    "fmt"
    "sync"
    "strconv"
)

func main() {
    var m sync.Map
    
    var wg sync.WaitGroup
    
    // เขียนพร้อมกัน
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := strconv.Itoa(id)
            m.Store(key, id*id)
        }(i)
    }
    
    wg.Wait()
    
    // อ่านพร้อมกัน
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := strconv.Itoa(id)
            if val, ok := m.Load(key); ok {
                fmt.Printf("key=%s, val=%v\n", key, val)
            }
        }(i)
    }
    
    wg.Wait()
    
    // LoadOrStore
    fmt.Println("\n=== LoadOrStore ===")
    actual, loaded := m.LoadOrStore("100", 999)
    fmt.Printf("loaded=%v, actual=%v\n", loaded, actual) // false, 999 (stored)
    
    actual2, loaded2 := m.LoadOrStore("0", 999) // key "0" already exists
    fmt.Printf("loaded=%v, actual=%v\n", loaded2, actual2) // true, 0
    
    // LoadAndDelete
    val, ok := m.LoadAndDelete("0")
    fmt.Printf("\nLoadAndDelete '0': val=%v, ok=%v\n", val, ok)
    
    _, ok2 := m.Load("0")
    fmt.Printf("Load '0' after delete: ok=%v\n", ok2)
    
    // Range
    fmt.Println("\n=== All values ===")
    m.Range(func(key, value any) bool {
        fmt.Printf("  %v: %v\n", key, value)
        return true // return false หยุด iteration
    })
    
    // ตัวอย่างจริง: Concurrent request cache
    fmt.Println("\n=== Request Counter ===")
    var requestCounts sync.Map
    
    // Simulate concurrent requests
    endpoints := []string{"/api/users", "/api/orders", "/api/products", "/api/users", "/api/orders"}
    
    for _, ep := range endpoints {
        wg.Add(1)
        endpoint := ep
        go func() {
            defer wg.Done()
            for {
                actual, loaded := requestCounts.LoadOrStore(endpoint, 1)
                if loaded {
                    // อัปเดต counter (ไม่ atomic แต่ใช้สำหรับ demo)
                    requestCounts.Store(endpoint, actual.(int)+1)
                    return
                }
                return
            }
        }()
    }
    
    wg.Wait()
    
    requestCounts.Range(func(key, value any) bool {
        fmt.Printf("  %s: %d requests\n", key, value)
        return true
    })
}
```

---

## 25.5 sync.Pool

### 25.5.1 Object Pool

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
    "time"
)

// Buffer Pool - ลด GC pressure
var bufferPool = sync.Pool{
    New: func() any {
        fmt.Println("  [pool] Creating new buffer")
        return new(bytes.Buffer)
    },
}

func getBuffer() *bytes.Buffer {
    return bufferPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
    buf.Reset() // ล้างข้อมูลก่อนคืน
    bufferPool.Put(buf)
}

func processRequest(id int) string {
    buf := getBuffer()
    defer putBuffer(buf)
    
    fmt.Fprintf(buf, "Request %d processed at %s", id, time.Now().Format("15:04:05.000"))
    return buf.String()
}

// Custom object pool
type Worker struct {
    ID       int
    buffer   []byte
    reusable bool
}

func (w *Worker) Reset() {
    w.ID = 0
    w.buffer = w.buffer[:0]
}

var workerPool = sync.Pool{
    New: func() any {
        return &Worker{
            buffer: make([]byte, 0, 4096),
        }
    },
}

func main() {
    fmt.Println("=== Buffer Pool ===")
    
    var wg sync.WaitGroup
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        id := i
        go func() {
            defer wg.Done()
            result := processRequest(id)
            fmt.Printf("  Result: %s\n", result)
        }()
    }
    
    wg.Wait()
    
    // ทดสอบ reuse
    fmt.Println("\n=== Reuse Demo ===")
    
    buf1 := getBuffer()
    buf1.WriteString("hello")
    addr1 := fmt.Sprintf("%p", buf1)
    putBuffer(buf1)
    
    buf2 := getBuffer()
    addr2 := fmt.Sprintf("%p", buf2)
    fmt.Printf("buf1 addr: %s\n", addr1)
    fmt.Printf("buf2 addr: %s (reused: %v)\n", addr2, addr1 == addr2)
    fmt.Printf("buf2 empty: %v\n", buf2.Len() == 0) // should be empty after Reset
    putBuffer(buf2)
    
    // Pool กับ benchmark context
    fmt.Println("\n=== Throughput Test ===")
    
    start := time.Now()
    for i := 0; i < 10000; i++ {
        buf := getBuffer()
        fmt.Fprintf(buf, "item %d", i)
        _ = buf.String()
        putBuffer(buf)
    }
    withPool := time.Since(start)
    
    start = time.Now()
    for i := 0; i < 10000; i++ {
        var buf bytes.Buffer
        fmt.Fprintf(&buf, "item %d", i)
        _ = buf.String()
    }
    withoutPool := time.Since(start)
    
    fmt.Printf("With pool:    %v\n", withPool)
    fmt.Printf("Without pool: %v\n", withoutPool)
    fmt.Printf("Speedup: %.2fx\n", float64(withoutPool)/float64(withPool))
}
```

---

## 25.6 sync/atomic

### 25.6.1 Atomic Operations

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

func main() {
    // Atomic counter
    var counter int64
    
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }
    
    wg.Wait()
    fmt.Printf("Counter: %d (expected 1000)\n", atomic.LoadInt64(&counter))
    
    // Compare and Swap
    var state int32 = 0 // 0=idle, 1=running
    
    tryStart := func() bool {
        return atomic.CompareAndSwapInt32(&state, 0, 1)
    }
    
    stop := func() {
        atomic.StoreInt32(&state, 0)
    }
    
    if tryStart() {
        fmt.Println("\nStarted successfully")
        time.Sleep(10 * time.Millisecond)
        
        if tryStart() {
            fmt.Println("Started again (shouldn't happen)")
        } else {
            fmt.Println("Already running (expected)")
        }
        
        stop()
        fmt.Println("Stopped")
        
        if tryStart() {
            fmt.Println("Started again after stop")
            stop()
        }
    }
    
    // Atomic Value
    var config atomic.Value
    
    type AppConfig struct {
        Debug   bool
        Version string
        MaxConn int
    }
    
    config.Store(AppConfig{Debug: false, Version: "1.0", MaxConn: 100})
    
    // Reader goroutines
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cfg := config.Load().(AppConfig)
            fmt.Printf("Reader %d: version=%s, debug=%v\n", id, cfg.Version, cfg.Debug)
        }(i)
    }
    
    // Hot reload
    wg.Add(1)
    go func() {
        defer wg.Done()
        time.Sleep(5 * time.Millisecond)
        config.Store(AppConfig{Debug: true, Version: "1.1", MaxConn: 200})
        fmt.Println("Config updated!")
    }()
    
    wg.Wait()
    
    finalCfg := config.Load().(AppConfig)
    fmt.Printf("\nFinal config: %+v\n", finalCfg)
    
    // Atomic operations reference
    var val int64 = 10
    
    fmt.Println("\n=== Atomic Operations ===")
    fmt.Printf("Initial: %d\n", atomic.LoadInt64(&val))
    
    old := atomic.SwapInt64(&val, 20)
    fmt.Printf("Swap 20: old=%d, new=%d\n", old, atomic.LoadInt64(&val))
    
    atomic.AddInt64(&val, 5)
    fmt.Printf("Add 5: %d\n", atomic.LoadInt64(&val))
    
    swapped := atomic.CompareAndSwapInt64(&val, 25, 100)
    fmt.Printf("CAS(25→100): swapped=%v, val=%d\n", swapped, atomic.LoadInt64(&val))
    
    swapped2 := atomic.CompareAndSwapInt64(&val, 999, 200) // val is 100, not 999
    fmt.Printf("CAS(999→200): swapped=%v, val=%d\n", swapped2, atomic.LoadInt64(&val))
}
```

---

## 25.7 sync.Cond

### 25.7.1 Condition Variables

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Queue ที่รอข้อมูลด้วย Cond
type BlockingQueue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []interface{}
    cap   int
}

func NewBlockingQueue(capacity int) *BlockingQueue {
    q := &BlockingQueue{
        items: make([]interface{}, 0, capacity),
        cap:   capacity,
    }
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *BlockingQueue) Push(item interface{}) {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    // รอจนกว่า queue จะมีที่ว่าง
    for len(q.items) >= q.cap {
        fmt.Printf("  Queue full (cap=%d), waiting...\n", q.cap)
        q.cond.Wait() // unlock mutex and wait, then re-lock
    }
    
    q.items = append(q.items, item)
    fmt.Printf("  Pushed: %v (size=%d)\n", item, len(q.items))
    q.cond.Signal() // wake one waiter
}

func (q *BlockingQueue) Pop() interface{} {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    // รอจนกว่าจะมีข้อมูล
    for len(q.items) == 0 {
        fmt.Println("  Queue empty, waiting...")
        q.cond.Wait()
    }
    
    item := q.items[0]
    q.items = q.items[1:]
    fmt.Printf("  Popped: %v (size=%d)\n", item, len(q.items))
    q.cond.Signal() // wake one waiter
    return item
}

func (q *BlockingQueue) Size() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    return len(q.items)
}

func main() {
    queue := NewBlockingQueue(3) // capacity = 3
    
    var wg sync.WaitGroup
    
    // Producer: push 5 items
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 1; i <= 5; i++ {
            fmt.Printf("\nProducer pushing %d\n", i)
            queue.Push(i)
            time.Sleep(100 * time.Millisecond)
        }
        fmt.Println("\nProducer done")
    }()
    
    // Consumer: pop 5 items (slower than producer)
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 5; i++ {
            time.Sleep(300 * time.Millisecond) // consume slower
            fmt.Printf("\nConsumer popping\n")
            val := queue.Pop()
            fmt.Printf("Consumer got: %v\n", val)
        }
        fmt.Println("\nConsumer done")
    }()
    
    wg.Wait()
    fmt.Printf("\nFinal queue size: %d\n", queue.Size())
    
    // Broadcast example
    fmt.Println("\n=== Broadcast ===")
    
    var mu sync.Mutex
    cond := sync.NewCond(&mu)
    ready := false
    
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            mu.Lock()
            for !ready {
                cond.Wait()
            }
            mu.Unlock()
            fmt.Printf("Worker %d started!\n", id)
        }(i)
    }
    
    time.Sleep(100 * time.Millisecond)
    fmt.Println("Broadcasting start signal...")
    
    mu.Lock()
    ready = true
    cond.Broadcast() // wake ALL waiters
    mu.Unlock()
    
    wg.Wait()
}
```

---

## 25.8 Race Conditions และ Deadlocks

### 25.8.1 Race Condition Examples

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// ตัวอย่าง Race Condition (อย่าทำแบบนี้!)
type BankAccount struct {
    balance float64
    mu      sync.Mutex
}

func (a *BankAccount) Deposit(amount float64) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.balance += amount
}

func (a *BankAccount) Withdraw(amount float64) error {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    if a.balance < amount {
        return fmt.Errorf("insufficient funds: have %.2f, need %.2f", a.balance, amount)
    }
    a.balance -= amount
    return nil
}

func (a *BankAccount) Balance() float64 {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.balance
}

// Transfer ที่ต้อง lock 2 accounts พร้อมกัน
func Transfer(from, to *BankAccount, amount float64) error {
    // วิธีที่ถูกต้อง: lock ตามลำดับที่กำหนดไว้ เพื่อป้องกัน deadlock
    // (ในชีวิตจริงใช้ ID หรือ pointer address เพื่อกำหนดลำดับ)
    
    // ใช้ pointer address เพื่อกำหนด lock order
    if uintptr(fmt.Sprintf("%p", from)[0]) < uintptr(fmt.Sprintf("%p", to)[0]) {
        from.mu.Lock()
        defer from.mu.Unlock()
        to.mu.Lock()
        defer to.mu.Unlock()
    } else {
        to.mu.Lock()
        defer to.mu.Unlock()
        from.mu.Lock()
        defer from.mu.Unlock()
    }
    
    if from.balance < amount {
        return fmt.Errorf("insufficient funds")
    }
    
    from.balance -= amount
    to.balance += amount
    return nil
}

// Data race ตัวอย่าง และวิธีแก้ไข
type RaceCounter struct {
    // BAD: no protection
    badCount int
    
    // GOOD: with mutex
    mu       sync.Mutex
    goodCount int
}

func (c *RaceCounter) BadIncrement() {
    c.badCount++ // RACE CONDITION!
}

func (c *RaceCounter) GoodIncrement() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.goodCount++
}

func main() {
    // Bank transfer example
    alice := &BankAccount{balance: 1000}
    bob := &BankAccount{balance: 500}
    
    var wg sync.WaitGroup
    
    // Concurrent transfers
    for i := 0; i < 10; i++ {
        wg.Add(2)
        
        go func() {
            defer wg.Done()
            Transfer(alice, bob, 50)
        }()
        
        go func() {
            defer wg.Done()
            Transfer(bob, alice, 30)
        }()
    }
    
    wg.Wait()
    
    total := alice.Balance() + bob.Balance()
    fmt.Printf("Alice: %.2f\n", alice.Balance())
    fmt.Printf("Bob: %.2f\n", bob.Balance())
    fmt.Printf("Total: %.2f (should be 1500)\n", total)
    
    // Timer-based detection
    fmt.Println("\n=== Stale Read Detection ===")
    
    var mu sync.Mutex
    data := make(map[string]int)
    
    set := func(key string, val int) {
        mu.Lock()
        defer mu.Unlock()
        data[key] = val
    }
    
    get := func(key string) int {
        mu.Lock()
        defer mu.Unlock()
        return data[key]
    }
    
    set("counter", 0)
    
    start := time.Now()
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            current := get("counter")
            set("counter", current+1)
        }()
    }
    
    wg.Wait()
    
    fmt.Printf("Counter: %d (may not be 100 due to TOCTOU)\n", get("counter"))
    fmt.Printf("Time: %v\n", time.Since(start))
    
    // Correct atomic increment
    var atomicCounter sync.Mutex
    atomicData := 0
    
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomicCounter.Lock()
            atomicData++
            atomicCounter.Unlock()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Correct counter: %d\n", atomicData)
}
```

---

## 25.9 Workshop: Thread-safe Cache

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

// Cache entry
type entry struct {
    value     interface{}
    expiresAt time.Time
    hits      int64
}

func (e *entry) isExpired() bool {
    return !e.expiresAt.IsZero() && time.Now().After(e.expiresAt)
}

// Thread-safe Cache with TTL and stats
type Cache struct {
    mu      sync.RWMutex
    data    map[string]*entry
    
    // Stats (atomic)
    hits    int64
    misses  int64
    sets    int64
    evicts  int64
    
    // Config
    maxSize    int
    defaultTTL time.Duration
    
    // Cleanup
    cleanupOnce sync.Once
    stopCleanup chan struct{}
}

type CacheConfig struct {
    MaxSize    int
    DefaultTTL time.Duration
}

func NewCache(cfg CacheConfig) *Cache {
    if cfg.MaxSize <= 0 {
        cfg.MaxSize = 1000
    }
    if cfg.DefaultTTL <= 0 {
        cfg.DefaultTTL = 5 * time.Minute
    }
    
    c := &Cache{
        data:        make(map[string]*entry),
        maxSize:     cfg.MaxSize,
        defaultTTL:  cfg.DefaultTTL,
        stopCleanup: make(chan struct{}),
    }
    
    // เริ่ม background cleanup goroutine
    c.cleanupOnce.Do(func() {
        go c.cleanupLoop()
    })
    
    return c
}

func (c *Cache) cleanupLoop() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            c.evictExpired()
        case <-c.stopCleanup:
            return
        }
    }
}

func (c *Cache) evictExpired() {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    var evicted int
    for key, e := range c.data {
        if e.isExpired() {
            delete(c.data, key)
            evicted++
        }
    }
    
    if evicted > 0 {
        atomic.AddInt64(&c.evicts, int64(evicted))
    }
}

func (c *Cache) Set(key string, value interface{}, ttl ...time.Duration) {
    expiry := c.defaultTTL
    if len(ttl) > 0 && ttl[0] > 0 {
        expiry = ttl[0]
    }
    
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Evict if full
    if len(c.data) >= c.maxSize {
        c.evictOne()
    }
    
    c.data[key] = &entry{
        value:     value,
        expiresAt: time.Now().Add(expiry),
    }
    
    atomic.AddInt64(&c.sets, 1)
}

func (c *Cache) evictOne() {
    // Simple eviction: remove first expired, or first found
    for key, e := range c.data {
        if e.isExpired() {
            delete(c.data, key)
            atomic.AddInt64(&c.evicts, 1)
            return
        }
    }
    // No expired found, remove arbitrary
    for key := range c.data {
        delete(c.data, key)
        atomic.AddInt64(&c.evicts, 1)
        return
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    e, ok := c.data[key]
    c.mu.RUnlock()
    
    if !ok {
        atomic.AddInt64(&c.misses, 1)
        return nil, false
    }
    
    if e.isExpired() {
        // Delete expired entry
        c.mu.Lock()
        delete(c.data, key)
        c.mu.Unlock()
        
        atomic.AddInt64(&c.misses, 1)
        atomic.AddInt64(&c.evicts, 1)
        return nil, false
    }
    
    atomic.AddInt64(&c.hits, 1)
    atomic.AddInt64(&e.hits, 1)
    return e.value, true
}

// GetOrSet: get หากมี, ไม่มีก็ set และ return
func (c *Cache) GetOrSet(key string, fn func() (interface{}, error), ttl ...time.Duration) (interface{}, error) {
    if val, ok := c.Get(key); ok {
        return val, nil
    }
    
    val, err := fn()
    if err != nil {
        return nil, err
    }
    
    c.Set(key, val, ttl...)
    return val, nil
}

func (c *Cache) Delete(key string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    _, ok := c.data[key]
    if ok {
        delete(c.data, key)
    }
    return ok
}

func (c *Cache) Clear() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data = make(map[string]*entry)
}

func (c *Cache) Keys() []string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    keys := make([]string, 0, len(c.data))
    for k, e := range c.data {
        if !e.isExpired() {
            keys = append(keys, k)
        }
    }
    return keys
}

type CacheStats struct {
    Size   int
    Hits   int64
    Misses int64
    Sets   int64
    Evicts int64
    HitRate float64
}

func (c *Cache) Stats() CacheStats {
    c.mu.RLock()
    size := len(c.data)
    c.mu.RUnlock()
    
    hits := atomic.LoadInt64(&c.hits)
    misses := atomic.LoadInt64(&c.misses)
    total := hits + misses
    
    hitRate := 0.0
    if total > 0 {
        hitRate = float64(hits) / float64(total) * 100
    }
    
    return CacheStats{
        Size:    size,
        Hits:    hits,
        Misses:  misses,
        Sets:    atomic.LoadInt64(&c.sets),
        Evicts:  atomic.LoadInt64(&c.evicts),
        HitRate: hitRate,
    }
}

func (c *Cache) Close() {
    close(c.stopCleanup)
}

// Application usage
type UserService struct {
    cache *Cache
    db    map[int]string // simulate DB
    mu    sync.RWMutex
}

func NewUserService() *UserService {
    return &UserService{
        cache: NewCache(CacheConfig{
            MaxSize:    500,
            DefaultTTL: 10 * time.Minute,
        }),
        db: map[int]string{
            1: "Alice",
            2: "Bob",
            3: "Charlie",
            4: "Diana",
            5: "Eve",
        },
    }
}

func (s *UserService) GetUser(ctx context.Context, id int) (string, error) {
    cacheKey := fmt.Sprintf("user:%d", id)
    
    // ลอง cache ก่อน
    val, err := s.cache.GetOrSet(cacheKey, func() (interface{}, error) {
        // Cache miss: query DB
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(10 * time.Millisecond): // simulate DB latency
        }
        
        s.mu.RLock()
        name, ok := s.db[id]
        s.mu.RUnlock()
        
        if !ok {
            return nil, fmt.Errorf("user %d not found", id)
        }
        
        fmt.Printf("  [DB] Fetched user %d: %s\n", id, name)
        return name, nil
    }, 30*time.Second)
    
    if err != nil {
        return "", err
    }
    
    return val.(string), nil
}

func main() {
    fmt.Println("=== Thread-safe Cache Demo ===\n")
    
    // Basic operations
    cache := NewCache(CacheConfig{MaxSize: 100, DefaultTTL: 5 * time.Second})
    defer cache.Close()
    
    cache.Set("name", "Alice")
    cache.Set("age", 30)
    cache.Set("city", "Bangkok", 2*time.Second) // short TTL
    
    if val, ok := cache.Get("name"); ok {
        fmt.Printf("name: %v\n", val)
    }
    
    fmt.Printf("Keys: %v\n", cache.Keys())
    
    // Wait for short TTL to expire
    time.Sleep(3 * time.Second)
    
    if _, ok := cache.Get("city"); !ok {
        fmt.Println("city expired (expected)")
    }
    
    // Concurrent access test
    fmt.Println("\n=== Concurrent Access ===")
    
    svc := NewUserService()
    ctx := context.Background()
    
    var wg sync.WaitGroup
    
    // 20 concurrent requests for 5 users
    for i := 0; i < 20; i++ {
        wg.Add(1)
        userID := (i % 5) + 1
        go func(id int) {
            defer wg.Done()
            name, err := svc.GetUser(ctx, id)
            if err != nil {
                fmt.Printf("  Error: %v\n", err)
            } else {
                _ = name
            }
        }(userID)
    }
    
    wg.Wait()
    
    stats := svc.cache.Stats()
    fmt.Printf("\nCache Stats:\n")
    fmt.Printf("  Size:     %d\n", stats.Size)
    fmt.Printf("  Hits:     %d\n", stats.Hits)
    fmt.Printf("  Misses:   %d\n", stats.Misses)
    fmt.Printf("  Sets:     %d\n", stats.Sets)
    fmt.Printf("  Evicts:   %d\n", stats.Evicts)
    fmt.Printf("  Hit Rate: %.1f%%\n", stats.HitRate)
    
    // Stress test
    fmt.Println("\n=== Stress Test ===")
    
    stressCache := NewCache(CacheConfig{MaxSize: 50, DefaultTTL: 1 * time.Second})
    defer stressCache.Close()
    
    start := time.Now()
    var ops int64
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                key := fmt.Sprintf("key%d", j%100)
                stressCache.Set(key, j)
                stressCache.Get(key)
                atomic.AddInt64(&ops, 2)
            }
        }(i)
    }
    
    wg.Wait()
    elapsed := time.Since(start)
    
    stressStats := stressCache.Stats()
    totalOps := atomic.LoadInt64(&ops)
    
    fmt.Printf("Operations: %d\n", totalOps)
    fmt.Printf("Duration: %v\n", elapsed)
    fmt.Printf("Throughput: %.0f ops/sec\n", float64(totalOps)/elapsed.Seconds())
    fmt.Printf("Hit Rate: %.1f%%\n", stressStats.HitRate)
    
    fmt.Println("\nAll tests completed!")
}
```

---

## สรุป

| Primitive | การใช้งาน |
|----------|----------|
| `sync.Mutex` | Exclusive access ทั่วไป |
| `sync.RWMutex` | Read-heavy workload |
| `sync.WaitGroup` | รอ goroutines ให้เสร็จ |
| `sync.Once` | Initialize ครั้งเดียว |
| `sync.Map` | Concurrent map |
| `sync.Pool` | Reuse objects, ลด GC |
| `sync.Cond` | Wait/Signal บน condition |
| `atomic` | Simple counter, flag, pointer |

### Checklist

- ใช้ `-race` flag เมื่อ test: `go test -race ./...`
- ไม่ copy Mutex (ใช้ pointer receiver)
- `defer mu.Unlock()` ทันทีหลัง `mu.Lock()`
- Lock order ต้องสม่ำเสมอเพื่อป้องกัน deadlock
- Prefer channels สำหรับ communication, Mutex สำหรับ state

## Resources

- [sync package](https://pkg.go.dev/sync)
- [sync/atomic package](https://pkg.go.dev/sync/atomic)
- [Go blog: Share Memory by Communicating](https://go.dev/blog/codelab-share)
- [Go Data Race Detector](https://go.dev/doc/articles/race_detector)
- [The Go Memory Model](https://go.dev/ref/mem)
