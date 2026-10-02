# Part 13: Goroutines and Concurrency ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- อธิบายความแตกต่างระหว่าง concurrency และ parallelism
- สร้างและใช้งาน goroutines ได้
- ใช้ sync.WaitGroup รอ goroutines
- เข้าใจ race conditions และวิธีตรวจสอบ
- ใช้ sync.Mutex และ sync.RWMutex
- เข้าใจ goroutine lifecycle
- เขียน goroutine patterns ที่ดี
- ตั้งค่า GOMAXPROCS

---

## 13.1 Concurrency vs Parallelism

**Concurrency (การทำงานพร้อมกันเชิงตรรกะ)**
- จัดการหลาย tasks พร้อมกัน แต่อาจไม่ได้รันพร้อมกันจริงๆ
- เกี่ยวกับการออกแบบ structure ของโปรแกรม
- "Dealing with lots of things at once"

**Parallelism (การทำงานขนานกัน)**
- รัน tasks หลายตัวพร้อมกันจริงๆ บน multiple CPU cores
- เกี่ยวกับ execution
- "Doing lots of things at once"

```
Concurrency (1 CPU):
Time: 1  2  3  4  5  6  7  8
T1:   ██    ██    ██          (interleaved)
T2:      ██    ██    ██

Parallelism (2 CPUs):
Time: 1  2  3
T1:   ████████              (CPU1)
T2:   ████████              (CPU2)
```

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func main() {
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())
    fmt.Printf("NumGoroutine: %d\n", runtime.NumGoroutine())
}
```

---

## 13.2 Goroutine Basics

**Goroutine** คือ lightweight thread ที่ Go runtime จัดการ ใช้ `go` keyword นำหน้า function call

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    fmt.Printf("สวัสดี %s!\n", name)
}

func main() {
    // Normal function call - รอให้จบก่อน
    sayHello("สมชาย")
    
    // Goroutine - รันแบบ concurrent
    go sayHello("สมหญิง")
    go sayHello("สมศักดิ์")
    
    // ต้องรอ goroutines ทำงานเสร็จ
    time.Sleep(100 * time.Millisecond)
    
    fmt.Println("main จบแล้ว")
}
```

### Goroutine กับ Anonymous Function

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        i := i // สำคัญมาก! สร้าง copy ของ i
        go func() {
            defer wg.Done()
            fmt.Printf("Goroutine %d กำลังทำงาน\n", i)
        }()
    }
    
    wg.Wait()
    fmt.Println("ทุก goroutine จบแล้ว")
}
```

---

## 13.3 sync.WaitGroup

WaitGroup ใช้รอ goroutines ทั้งหมดให้ทำงานเสร็จ

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func downloadFile(id int, wg *sync.WaitGroup) {
    defer wg.Done() // เรียกเมื่อ function จบ
    
    fmt.Printf("เริ่มดาวน์โหลด file %d\n", id)
    time.Sleep(time.Duration(id*100) * time.Millisecond)
    fmt.Printf("ดาวน์โหลด file %d เสร็จแล้ว\n", id)
}

func main() {
    var wg sync.WaitGroup
    
    start := time.Now()
    
    files := []int{1, 2, 3, 4, 5}
    
    for _, id := range files {
        wg.Add(1)        // เพิ่มนับก่อน start goroutine
        go downloadFile(id, &wg)
    }
    
    wg.Wait() // รอทุก goroutine
    
    elapsed := time.Since(start)
    fmt.Printf("\nดาวน์โหลดทั้งหมดเสร็จใน %.2f วินาที\n", elapsed.Seconds())
    
    // เปรียบเทียบ: แบบ sequential
    start2 := time.Now()
    for _, id := range files {
        downloadFile(id, &wg) // ไม่ใช้ goroutine
        // wg.Done() would be needed but this is just for timing
    }
    _ = start2
}
```

### WaitGroup Pattern

```go
package main

import (
    "fmt"
    "sync"
    "math/rand"
    "time"
)

type Result struct {
    ID    int
    Value string
    Err   error
}

func processItem(id int) Result {
    // Simulate work
    time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
    
    if id%7 == 0 {
        return Result{ID: id, Err: fmt.Errorf("item %d: processing failed", id)}
    }
    
    return Result{ID: id, Value: fmt.Sprintf("processed-item-%d", id)}
}

func processAll(items []int) []Result {
    results := make([]Result, len(items))
    var wg sync.WaitGroup
    
    for i, item := range items {
        wg.Add(1)
        i, item := i, item // capture loop variables
        go func() {
            defer wg.Done()
            results[i] = processItem(item)
        }()
    }
    
    wg.Wait()
    return results
}

func main() {
    items := make([]int, 20)
    for i := range items {
        items[i] = i + 1
    }
    
    start := time.Now()
    results := processAll(items)
    elapsed := time.Since(start)
    
    success := 0
    failed := 0
    
    for _, r := range results {
        if r.Err != nil {
            fmt.Printf("ERROR: %v\n", r.Err)
            failed++
        } else {
            success++
        }
    }
    
    fmt.Printf("\nประมวลผล %d รายการ ใน %.2f วินาที\n", len(items), elapsed.Seconds())
    fmt.Printf("สำเร็จ: %d, ล้มเหลว: %d\n", success, failed)
}
```

---

## 13.4 Race Conditions

**Race condition** เกิดขึ้นเมื่อ goroutines หลายตัวเข้าถึงข้อมูลเดียวกันพร้อมกัน โดยไม่มีการ synchronize

```go
package main

import (
    "fmt"
    "sync"
)

// Race condition - ไม่ดี!
var counter int

func incrementUnsafe(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        counter++ // race condition!
    }
}

// Thread-safe ด้วย Mutex
type SafeCounter struct {
    mu sync.Mutex
    n  int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}

// Thread-safe ด้วย atomic
import "sync/atomic"

var atomicCounter int64

func incrementAtomic(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        atomic.AddInt64(&atomicCounter, 1)
    }
}

func main() {
    var wg sync.WaitGroup
    
    // Unsafe counter (อาจได้ผลลัพธ์ผิด)
    counter = 0
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go incrementUnsafe(&wg)
    }
    wg.Wait()
    fmt.Printf("Unsafe counter: %d (ควรเป็น 10000)\n", counter)
    
    // Safe counter ด้วย Mutex
    safe := &SafeCounter{}
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                safe.Increment()
            }
        }()
    }
    wg.Wait()
    fmt.Printf("Safe counter: %d\n", safe.Value())
    
    // Atomic counter
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go incrementAtomic(&wg)
    }
    wg.Wait()
    fmt.Printf("Atomic counter: %d\n", atomic.LoadInt64(&atomicCounter))
}
```

---

## 13.5 Data Race Detection

Go มี built-in race detector ใช้ `go run -race` หรือ `go test -race`

```go
package main

import (
    "fmt"
    "sync"
)

// ตัวอย่างที่มี data race
type UnsafeCache struct {
    data map[string]string
}

func (c *UnsafeCache) Set(key, value string) {
    c.data[key] = value // RACE!
}

func (c *UnsafeCache) Get(key string) string {
    return c.data[key] // RACE!
}

// Thread-safe cache
type SafeCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewSafeCache() *SafeCache {
    return &SafeCache{data: make(map[string]string)}
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()         // Write lock
    defer c.mu.Unlock()
    c.data[key] = value
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()         // Read lock (หลาย goroutines อ่านพร้อมกันได้)
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *SafeCache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.data, key)
}

func (c *SafeCache) Keys() []string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    keys := make([]string, 0, len(c.data))
    for k := range c.data {
        keys = append(keys, k)
    }
    return keys
}

func main() {
    cache := NewSafeCache()
    var wg sync.WaitGroup
    
    // Writers
    for i := 0; i < 5; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", i)
            cache.Set(key, fmt.Sprintf("value-%d", i))
        }()
    }
    
    // Readers (ในขณะที่กำลัง write)
    for i := 0; i < 10; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", i%5)
            if v, ok := cache.Get(key); ok {
                fmt.Printf("Read %s = %s\n", key, v)
            }
        }()
    }
    
    wg.Wait()
    
    fmt.Println("\nAll keys:", cache.Keys())
}
```

---

## 13.6 Mutex Patterns

### sync.Mutex - Exclusive Lock

```go
package main

import (
    "fmt"
    "sync"
)

type BankAccount struct {
    mu      sync.Mutex
    balance float64
    owner   string
}

func NewBankAccount(owner string, initial float64) *BankAccount {
    return &BankAccount{owner: owner, balance: initial}
}

func (a *BankAccount) Deposit(amount float64) error {
    if amount <= 0 {
        return fmt.Errorf("จำนวนเงินต้องมากกว่า 0")
    }
    a.mu.Lock()
    defer a.mu.Unlock()
    a.balance += amount
    return nil
}

func (a *BankAccount) Withdraw(amount float64) error {
    if amount <= 0 {
        return fmt.Errorf("จำนวนเงินต้องมากกว่า 0")
    }
    a.mu.Lock()
    defer a.mu.Unlock()
    if a.balance < amount {
        return fmt.Errorf("ยอดเงินไม่พอ (มี %.2f ต้องการ %.2f)", a.balance, amount)
    }
    a.balance -= amount
    return nil
}

func (a *BankAccount) Balance() float64 {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.balance
}

func main() {
    account := NewBankAccount("สมชาย", 1000)
    var wg sync.WaitGroup
    
    // Concurrent deposits and withdrawals
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            account.Deposit(100)
        }()
    }
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            err := account.Withdraw(150)
            if err != nil {
                fmt.Printf("Withdraw error: %v\n", err)
            }
        }()
    }
    
    wg.Wait()
    fmt.Printf("ยอดเงินสุดท้าย: %.2f\n", account.Balance())
    // 1000 + (10*100) - (5*150) = 1000 + 1000 - 750 = 1250
}
```

### sync.RWMutex - Read/Write Lock

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Config struct {
    mu     sync.RWMutex
    values map[string]string
}

func NewConfig() *Config {
    return &Config{values: make(map[string]string)}
}

func (c *Config) Get(key string) (string, bool) {
    c.mu.RLock() // หลาย goroutines อ่านพร้อมกันได้
    defer c.mu.RUnlock()
    v, ok := c.values[key]
    return v, ok
}

func (c *Config) Set(key, value string) {
    c.mu.Lock() // exclusive write lock
    defer c.mu.Unlock()
    c.values[key] = value
}

func (c *Config) LoadAll() map[string]string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    result := make(map[string]string, len(c.values))
    for k, v := range c.values {
        result[k] = v
    }
    return result
}

func main() {
    cfg := NewConfig()
    
    // Initial values
    cfg.Set("host", "localhost")
    cfg.Set("port", "8080")
    cfg.Set("debug", "false")
    
    var wg sync.WaitGroup
    
    // 100 concurrent readers
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            host, _ := cfg.Get("host")
            port, _ := cfg.Get("port")
            _ = host
            _ = port
        }()
    }
    
    // 5 occasional writers
    for i := 0; i < 5; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            time.Sleep(time.Duration(i*10) * time.Millisecond)
            cfg.Set("port", fmt.Sprintf("%d", 8080+i))
        }()
    }
    
    wg.Wait()
    
    fmt.Println("Final config:", cfg.LoadAll())
}
```

---

## 13.7 sync.Once

ทำให้ code บางส่วนรันแค่ครั้งเดียว แม้จะเรียกจาก goroutines หลายตัว

```go
package main

import (
    "fmt"
    "sync"
)

type Singleton struct {
    value string
}

var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        fmt.Println("สร้าง singleton instance")
        instance = &Singleton{value: "initialized"}
    })
    return instance
}

// Database connection pool
type DBPool struct {
    connections []string
}

var (
    dbPool     *DBPool
    dbPoolOnce sync.Once
)

func GetDBPool() *DBPool {
    dbPoolOnce.Do(func() {
        fmt.Println("เริ่มต้น database pool...")
        dbPool = &DBPool{
            connections: []string{"conn-1", "conn-2", "conn-3"},
        }
        fmt.Println("Database pool พร้อมแล้ว")
    })
    return dbPool
}

func main() {
    var wg sync.WaitGroup
    
    // เรียก GetInstance จาก goroutines หลายตัว
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            inst := GetInstance()
            _ = inst
        }()
    }
    
    wg.Wait()
    fmt.Printf("Instance value: %s\n", GetInstance().value)
    
    // DB Pool
    fmt.Println()
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            pool := GetDBPool()
            _ = pool
        }()
    }
    
    wg.Wait()
    fmt.Printf("Pool connections: %v\n", GetDBPool().connections)
}
```

---

## 13.8 sync.Map

Thread-safe map ที่ไม่ต้องใช้ mutex เอง

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
    
    // Concurrent writes
    for i := 0; i < 100; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            key := "key-" + strconv.Itoa(i)
            m.Store(key, i*i)
        }()
    }
    
    wg.Wait()
    
    // Read
    for i := 0; i < 5; i++ {
        key := "key-" + strconv.Itoa(i)
        if val, ok := m.Load(key); ok {
            fmt.Printf("%s = %v\n", key, val)
        }
    }
    
    // Range over all entries
    count := 0
    m.Range(func(key, value interface{}) bool {
        count++
        return true // continue iteration
    })
    fmt.Printf("Total entries: %d\n", count)
    
    // Delete
    m.Delete("key-0")
    
    // LoadOrStore
    actual, loaded := m.LoadOrStore("new-key", "new-value")
    fmt.Printf("LoadOrStore: value=%v, loaded=%v\n", actual, loaded)
}
```

---

## 13.9 Goroutine Patterns

### Worker Pool Pattern

```go
package main

import (
    "fmt"
    "sync"
    "time"
    "math/rand"
)

type Job struct {
    ID   int
    Data string
}

type JobResult struct {
    JobID  int
    Result string
    Error  error
}

func worker(id int, jobs <-chan Job, results chan<- JobResult, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for job := range jobs {
        // Process job
        time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
        
        result := JobResult{
            JobID:  job.ID,
            Result: fmt.Sprintf("processed by worker-%d: %s", id, job.Data),
        }
        
        results <- result
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10
    
    jobs := make(chan Job, numJobs)
    results := make(chan JobResult, numJobs)
    
    var wg sync.WaitGroup
    
    // Start workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }
    
    // Send jobs
    for i := 1; i <= numJobs; i++ {
        jobs <- Job{
            ID:   i,
            Data: fmt.Sprintf("data-%d", i),
        }
    }
    close(jobs) // ปิด channel เมื่อส่งงานครบ
    
    // Close results เมื่อ workers ทุกตัวจบ
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    for result := range results {
        if result.Error != nil {
            fmt.Printf("Job %d error: %v\n", result.JobID, result.Error)
        } else {
            fmt.Printf("Job %d: %s\n", result.JobID, result.Result)
        }
    }
}
```

### Fan-out Fan-in Pattern

```go
package main

import (
    "fmt"
    "sync"
)

// Fan-out: แจกงานให้ goroutines หลายตัว
func fanOut(input <-chan int, numWorkers int) []<-chan int {
    channels := make([]<-chan int, numWorkers)
    
    for i := 0; i < numWorkers; i++ {
        ch := make(chan int)
        channels[i] = ch
        
        go func(out chan<- int) {
            for v := range input {
                out <- v * v // ยกกำลังสอง
            }
            close(out)
        }(ch)
    }
    
    return channels
}

// Fan-in: รวม goroutines หลายตัวเป็น channel เดียว
func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int, 100)
    
    output := func(c <-chan int) {
        defer wg.Done()
        for v := range c {
            merged <- v
        }
    }
    
    wg.Add(len(channels))
    for _, c := range channels {
        go output(c)
    }
    
    go func() {
        wg.Wait()
        close(merged)
    }()
    
    return merged
}

func main() {
    // Source data
    source := make(chan int, 10)
    go func() {
        for i := 1; i <= 10; i++ {
            source <- i
        }
        close(source)
    }()
    
    // Fan-out to 3 workers
    workers := fanOut(source, 3)
    
    // Fan-in results
    results := fanIn(workers...)
    
    sum := 0
    for v := range results {
        sum += v
        fmt.Printf("Result: %d\n", v)
    }
    
    fmt.Printf("Sum: %d\n", sum)
}
```

---

## 13.10 Context สำหรับ Cancellation

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func longOperation(ctx context.Context, id int) error {
    select {
    case <-time.After(time.Duration(id*200) * time.Millisecond):
        fmt.Printf("Operation %d completed\n", id)
        return nil
    case <-ctx.Done():
        fmt.Printf("Operation %d cancelled: %v\n", id, ctx.Err())
        return ctx.Err()
    }
}

func fetchWithTimeout(url string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    // Simulate fetch
    done := make(chan string, 1)
    go func() {
        time.Sleep(300 * time.Millisecond) // simulate work
        done <- "fetched: " + url
    }()
    
    select {
    case result := <-done:
        return result, nil
    case <-ctx.Done():
        return "", fmt.Errorf("fetch timeout: %w", ctx.Err())
    }
}

func main() {
    // Context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 350*time.Millisecond)
    defer cancel()
    
    var wg sync.WaitGroup
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            longOperation(ctx, i)
        }()
    }
    
    wg.Wait()
    
    fmt.Println("\n--- Fetch with timeout ---")
    urls := []string{"fast-api.com", "slow-api.com"}
    for _, url := range urls {
        result, err := fetchWithTimeout(url)
        if err != nil {
            fmt.Printf("Error fetching %s: %v\n", url, err)
        } else {
            fmt.Println(result)
        }
    }
}

import "sync"
```

---

## 13.11 runtime.GOMAXPROCS

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func cpuBoundTask(id int) {
    sum := 0
    for i := 0; i < 1000000; i++ {
        sum += i
    }
    _ = sum
}

func benchmark(numProcs int) time.Duration {
    runtime.GOMAXPROCS(numProcs)
    
    var wg sync.WaitGroup
    start := time.Now()
    
    for i := 0; i < 8; i++ {
        wg.Add(1)
        i := i
        go func() {
            defer wg.Done()
            cpuBoundTask(i)
        }()
    }
    
    wg.Wait()
    return time.Since(start)
}

func main() {
    numCPU := runtime.NumCPU()
    fmt.Printf("จำนวน CPU cores: %d\n\n", numCPU)
    
    for _, procs := range []int{1, 2, 4, numCPU} {
        if procs > numCPU {
            continue
        }
        dur := benchmark(procs)
        fmt.Printf("GOMAXPROCS=%d: %.2f ms\n", procs, float64(dur.Microseconds())/1000)
    }
    
    // Reset to default
    runtime.GOMAXPROCS(numCPU)
    fmt.Printf("\nGOMAXPROCS reset to: %d\n", runtime.GOMAXPROCS(0))
    
    // Goroutine stats
    fmt.Printf("Current goroutines: %d\n", runtime.NumGoroutine())
}
```

---

## 13.12 Goroutine Lifecycle

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func goroutineLifecycle() {
    fmt.Printf("Goroutines at start: %d\n", runtime.NumGoroutine())
    
    var wg sync.WaitGroup
    done := make(chan struct{})
    
    // Long-running goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        defer fmt.Println("Long goroutine stopped")
        
        for {
            select {
            case <-done:
                return
            case <-time.After(100 * time.Millisecond):
                // doing work...
            }
        }
    }()
    
    fmt.Printf("Goroutines after start: %d\n", runtime.NumGoroutine())
    
    // Stop goroutine
    time.Sleep(300 * time.Millisecond)
    close(done)
    wg.Wait()
    
    // Give GC time to clean up
    runtime.GC()
    time.Sleep(10 * time.Millisecond)
    
    fmt.Printf("Goroutines after stop: %d\n", runtime.NumGoroutine())
}

// Goroutine leak - ตัวอย่างที่ไม่ดี
func goroutineLeak() {
    ch := make(chan int) // unbuffered channel
    
    go func() {
        // goroutine นี้จะรอตลอดไปถ้าไม่มีใช้อ่าน ch
        for v := range ch {
            fmt.Println(v)
        }
    }()
    
    // ไม่ส่งค่าและไม่ close channel -> goroutine leak!
    fmt.Printf("Goroutines (leaked): %d\n", runtime.NumGoroutine())
}

// Fixed version
func noLeak() {
    ch := make(chan int)
    done := make(chan struct{})
    
    go func() {
        defer fmt.Println("goroutine cleaned up")
        for {
            select {
            case v, ok := <-ch:
                if !ok {
                    return
                }
                fmt.Println(v)
            case <-done:
                return
            }
        }
    }()
    
    ch <- 1
    ch <- 2
    close(done) // signal goroutine to stop
}

func main() {
    goroutineLifecycle()
    fmt.Println()
    goroutineLeak()
    time.Sleep(100 * time.Millisecond)
    fmt.Println()
    noLeak()
    time.Sleep(100 * time.Millisecond)
}
```

---

## Workshop: Parallel File Processor

สร้างโปรแกรมประมวลผลไฟล์แบบ parallel

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "strings"
    "sync"
    "time"
    "math/rand"
)

// ====== Types ======

type FileContent struct {
    Name    string
    Content string
    Size    int
}

type ProcessedFile struct {
    Original  FileContent
    WordCount int
    LineCount int
    CharCount int
    Duration  time.Duration
    Error     error
}

type Stats struct {
    TotalFiles     int
    SuccessFiles   int
    FailedFiles    int
    TotalWords     int
    TotalLines     int
    TotalChars     int
    TotalDuration  time.Duration
    FastestFile    string
    SlowestFile    string
}

// ====== File Generator (mock) ======

func generateFiles(count int) []FileContent {
    templates := []string{
        "สวัสดี Go! นี่คือตัวอย่างไฟล์ที่สร้างขึ้นมา\nมีหลายบรรทัดเพื่อทดสอบ\nการประมวลผลแบบ parallel",
        "Go is an open source programming language\nthat makes it easy to build simple, reliable, and efficient software.\nThis is a test file.",
        "หนึ่ง สอง สาม สี่ ห้า\nหก เจ็ด แปด เก้า สิบ\nการนับแบบง่ายๆ",
        "package main\n\nimport \"fmt\"\n\nfunc main() {\n\tfmt.Println(\"Hello, World!\")\n}",
    }
    
    files := make([]FileContent, count)
    for i := range files {
        content := templates[i%len(templates)]
        files[i] = FileContent{
            Name:    fmt.Sprintf("file_%03d.txt", i+1),
            Content: content,
            Size:    len(content),
        }
    }
    return files
}

// ====== Processor ======

func processFile(ctx context.Context, file FileContent) ProcessedFile {
    start := time.Now()
    
    // Check context
    select {
    case <-ctx.Done():
        return ProcessedFile{
            Original: file,
            Error:    fmt.Errorf("cancelled: %w", ctx.Err()),
            Duration: time.Since(start),
        }
    default:
    }
    
    // Simulate processing time
    delay := time.Duration(rand.Intn(300)+50) * time.Millisecond
    
    select {
    case <-time.After(delay):
        // Process completed
    case <-ctx.Done():
        return ProcessedFile{
            Original: file,
            Error:    fmt.Errorf("timeout: %w", ctx.Err()),
            Duration: time.Since(start),
        }
    }
    
    // Simulate occasional errors
    if rand.Float32() < 0.1 { // 10% error rate
        return ProcessedFile{
            Original: file,
            Error:    fmt.Errorf("failed to process %s", file.Name),
            Duration: time.Since(start),
        }
    }
    
    // Count statistics
    lines := strings.Split(file.Content, "\n")
    words := strings.Fields(file.Content)
    
    return ProcessedFile{
        Original:  file,
        WordCount: len(words),
        LineCount: len(lines),
        CharCount: len(file.Content),
        Duration:  time.Since(start),
    }
}

// ====== File Processor with Worker Pool ======

type FileProcessor struct {
    numWorkers int
    timeout    time.Duration
}

func NewFileProcessor(workers int, timeout time.Duration) *FileProcessor {
    return &FileProcessor{
        numWorkers: workers,
        timeout:    timeout,
    }
}

func (fp *FileProcessor) ProcessAll(ctx context.Context, files []FileContent) ([]ProcessedFile, Stats) {
    ctx, cancel := context.WithTimeout(ctx, fp.timeout)
    defer cancel()
    
    jobs := make(chan FileContent, len(files))
    results := make(chan ProcessedFile, len(files))
    
    var wg sync.WaitGroup
    
    // Start workers
    for i := 0; i < fp.numWorkers; i++ {
        wg.Add(1)
        workerID := i + 1
        go func() {
            defer wg.Done()
            fmt.Printf("Worker %d started\n", workerID)
            
            for file := range jobs {
                result := processFile(ctx, file)
                results <- result
            }
            
            fmt.Printf("Worker %d stopped\n", workerID)
        }()
    }
    
    // Send all files
    for _, f := range files {
        jobs <- f
    }
    close(jobs)
    
    // Wait and close results
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    processed := make([]ProcessedFile, 0, len(files))
    for result := range results {
        processed = append(processed, result)
    }
    
    // Calculate stats
    stats := calculateStats(processed)
    return processed, stats
}

func calculateStats(results []ProcessedFile) Stats {
    stats := Stats{TotalFiles: len(results)}
    
    var fastestDur time.Duration = -1
    var slowestDur time.Duration = 0
    
    for _, r := range results {
        if r.Error != nil {
            stats.FailedFiles++
        } else {
            stats.SuccessFiles++
            stats.TotalWords += r.WordCount
            stats.TotalLines += r.LineCount
            stats.TotalChars += r.CharCount
        }
        
        stats.TotalDuration += r.Duration
        
        if fastestDur < 0 || r.Duration < fastestDur {
            fastestDur = r.Duration
            stats.FastestFile = r.Original.Name
        }
        if r.Duration > slowestDur {
            slowestDur = r.Duration
            stats.SlowestFile = r.Original.Name
        }
    }
    
    return stats
}

func main() {
    fmt.Println("=== Parallel File Processor ===\n")
    
    // Generate test files
    files := generateFiles(20)
    fmt.Printf("ไฟล์ทั้งหมด: %d ไฟล์\n\n", len(files))
    
    // Process with different worker counts
    workerCounts := []int{1, 2, 4, 8}
    
    for _, workers := range workerCounts {
        processor := NewFileProcessor(workers, 10*time.Second)
        
        start := time.Now()
        results, stats := processor.ProcessAll(context.Background(), files)
        elapsed := time.Since(start)
        
        fmt.Printf("\n--- Workers: %d ---\n", workers)
        fmt.Printf("เวลารวม: %.2f วินาที\n", elapsed.Seconds())
        fmt.Printf("สำเร็จ: %d, ล้มเหลว: %d\n", stats.SuccessFiles, stats.FailedFiles)
        fmt.Printf("คำทั้งหมด: %d\n", stats.TotalWords)
        fmt.Printf("ไฟล์เร็วสุด: %s\n", stats.FastestFile)
        fmt.Printf("ไฟล์ช้าสุด: %s\n", stats.SlowestFile)
        
        // Show some errors
        errorCount := 0
        for _, r := range results {
            if r.Error != nil && !errors.Is(r.Error, context.Canceled) {
                errorCount++
                if errorCount <= 3 {
                    fmt.Printf("Error: %v\n", r.Error)
                }
            }
        }
    }
    
    fmt.Println("\n=== Done ===")
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| Goroutine | Lightweight concurrent function ด้วย `go` |
| WaitGroup | รอ goroutines ทั้งหมดให้จบ |
| Mutex | Lock สำหรับ exclusive access |
| RWMutex | Lock ที่ยอมให้ read พร้อมกันหลายตัว |
| sync.Once | รันแค่ครั้งเดียวแม้เรียกหลาย goroutines |
| sync.Map | Thread-safe map |
| Race Condition | ปัญหาเมื่อ goroutines เข้าถึงข้อมูลพร้อมกัน |
| `-race` flag | ตรวจหา data races |
| Worker Pool | Pattern จัดการงานด้วย goroutines pool |
| GOMAXPROCS | กำหนดจำนวน OS threads |

## Resources

- [Go Tour: Goroutines](https://go.dev/tour/concurrency/1)
- [Go Blog: Concurrency patterns](https://go.dev/blog/pipelines)
- [Go Memory Model](https://go.dev/ref/mem)
- [Concurrency in Go (book)](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/)
- [go -race detector](https://go.dev/blog/race-detector)

---

*Part 13 จบแล้ว! ต่อไป Part 14: Channels*
