# Part 14: Channels ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- สร้างและใช้งาน channels ได้
- เข้าใจความแตกต่างระหว่าง buffered และ unbuffered channels
- ใช้ channel directions เพื่อความปลอดภัย
- ใช้ select statement กับ channels
- ปิด channels และ range over channels
- ใช้ concurrency patterns: Fan-out, Fan-in, Pipeline
- ใช้ Done channel pattern
- จัดการ timeouts ด้วย channels

---

## 14.1 Channel คืออะไร?

**Channel** คือ "ท่อ" ที่ goroutines ใช้สื่อสารกัน ส่งข้อมูลระหว่างกัน

> "Don't communicate by sharing memory; share memory by communicating."
> — Rob Pike

```
Goroutine A ──── channel ────> Goroutine B
               (ส่งข้อมูล)    (รับข้อมูล)
```

```go
package main

import "fmt"

func sum(nums []int, ch chan int) {
    total := 0
    for _, n := range nums {
        total += n
    }
    ch <- total // ส่งผลลัพธ์ผ่าน channel
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    ch := make(chan int) // สร้าง channel
    
    mid := len(nums) / 2
    go sum(nums[:mid], ch) // ครึ่งแรก
    go sum(nums[mid:], ch) // ครึ่งหลัง
    
    // รับค่าสองครั้ง
    x, y := <-ch, <-ch
    
    fmt.Printf("ผลรวมครึ่งแรก + ครึ่งหลัง = %d + %d = %d\n", x, y, x+y)
}
```

---

## 14.2 Unbuffered Channels

Unbuffered channel ต้องมีทั้ง sender และ receiver พร้อมกัน (synchronous)

```go
package main

import (
    "fmt"
    "time"
)

func producer(ch chan<- string) {
    messages := []string{"ข้อความ 1", "ข้อความ 2", "ข้อความ 3"}
    for _, msg := range messages {
        fmt.Printf("ส่ง: %s\n", msg)
        ch <- msg // บล็อกจนกว่า receiver จะรับ
        fmt.Printf("ส่งแล้ว: %s\n", msg)
        time.Sleep(100 * time.Millisecond)
    }
    close(ch)
}

func consumer(ch <-chan string) {
    for msg := range ch {
        fmt.Printf("รับ: %s\n", msg)
        time.Sleep(200 * time.Millisecond) // ช้ากว่า producer
    }
}

func main() {
    ch := make(chan string) // unbuffered
    
    go producer(ch)
    consumer(ch)
    
    fmt.Println("เสร็จสิ้น")
}
```

### Deadlock ที่ต้องระวัง

```go
package main

import "fmt"

func main() {
    ch := make(chan int)
    
    // DEADLOCK! ไม่มี goroutine รับ
    // ch <- 1 // บล็อกตลอดไป -> panic: deadlock!
    
    // ต้องมี goroutine รับก่อน
    go func() {
        val := <-ch
        fmt.Println("Received:", val)
    }()
    
    ch <- 42 // ตอนนี้ไม่ deadlock
    fmt.Println("Sent")
}
```

---

## 14.3 Buffered Channels

Buffered channel มี buffer รองรับข้อมูล sender ไม่ต้องรอ receiver (ถ้า buffer ยังไม่เต็ม)

```go
package main

import "fmt"

func main() {
    // Buffered channel ขนาด 3
    ch := make(chan int, 3)
    
    // ส่งข้อมูลโดยไม่ต้องรอ receiver
    ch <- 1
    ch <- 2
    ch <- 3
    // ch <- 4 // นี่จะบล็อกเพราะ buffer เต็ม
    
    fmt.Printf("Buffer: %d/%d\n", len(ch), cap(ch))
    
    // รับข้อมูล
    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
    
    fmt.Printf("Buffer after: %d/%d\n", len(ch), cap(ch))
}
```

### เปรียบเทียบ Unbuffered vs Buffered

```go
package main

import (
    "fmt"
    "time"
)

func testUnbuffered() {
    ch := make(chan int)
    done := make(chan bool)
    
    go func() {
        for i := range ch {
            fmt.Printf("  Received %d\n", i)
        }
        done <- true
    }()
    
    start := time.Now()
    for i := 1; i <= 3; i++ {
        fmt.Printf("  Sending %d...\n", i)
        ch <- i
        fmt.Printf("  Sent %d (%.0fms)\n", i, float64(time.Since(start).Milliseconds()))
    }
    close(ch)
    <-done
}

func testBuffered(bufSize int) {
    ch := make(chan int, bufSize)
    done := make(chan bool)
    
    go func() {
        time.Sleep(500 * time.Millisecond) // delayed receiver
        for i := range ch {
            fmt.Printf("  Received %d\n", i)
        }
        done <- true
    }()
    
    start := time.Now()
    for i := 1; i <= 3; i++ {
        fmt.Printf("  Sending %d...\n", i)
        ch <- i
        fmt.Printf("  Sent %d (%.0fms)\n", i, float64(time.Since(start).Milliseconds()))
    }
    close(ch)
    <-done
}

func main() {
    fmt.Println("=== Unbuffered ===")
    testUnbuffered()
    
    fmt.Println("\n=== Buffered (size=3) ===")
    testBuffered(3)
}
```

---

## 14.4 Channel Directions

ระบุ direction ของ channel ใน function signature เพื่อความปลอดภัย

```go
package main

import "fmt"

// send-only channel
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// receive-only channel เข้า, send-only channel ออก
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// receive-only channel
func printer(in <-chan int) {
    for v := range in {
        fmt.Printf("  %d\n", v)
    }
}

func main() {
    // Pipeline: generator -> square -> printer
    nums := generator(1, 2, 3, 4, 5)
    squared := square(nums)
    printer(squared)
}
```

---

## 14.5 Select Statement

`select` ให้ goroutine รอหลาย channel operations พร้อมกัน

```go
package main

import (
    "fmt"
    "time"
)

func fibonacci(n int, ch chan int, quit chan struct{}) {
    x, y := 0, 1
    for {
        select {
        case ch <- x:
            x, y = y, x+y
            if x > n {
                return
            }
        case <-quit:
            fmt.Println("quit signal received")
            return
        }
    }
}

func main() {
    ch := make(chan int)
    quit := make(chan struct{})
    
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Printf("%d ", <-ch)
        }
        fmt.Println()
        close(quit)
    }()
    
    fibonacci(1000, ch, quit)
}
```

### Select กับ default

```go
package main

import (
    "fmt"
    "time"
)

func tryReceive(ch chan int) {
    select {
    case v := <-ch:
        fmt.Printf("Received: %d\n", v)
    default:
        fmt.Println("No value ready")
    }
}

func trySend(ch chan int, val int) bool {
    select {
    case ch <- val:
        return true
    default:
        return false
    }
}

func main() {
    ch := make(chan int, 1)
    
    // ลอง receive จาก empty channel
    tryReceive(ch) // "No value ready"
    
    // ส่งค่า
    ch <- 42
    
    // ลอง receive อีกครั้ง
    tryReceive(ch) // "Received: 42"
    
    // ลอง send ไปยัง full buffer
    ch <- 1
    sent := trySend(ch, 2) // buffer เต็ม
    fmt.Printf("Sent: %v\n", sent) // false
    
    // Non-blocking select with timeout
    ch2 := make(chan string)
    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "data"
    }()
    
    select {
    case v := <-ch2:
        fmt.Println("Got:", v)
    case <-time.After(100 * time.Millisecond):
        fmt.Println("Timeout! No data received")
    }
}
```

---

## 14.6 Channel Closing

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 5)
    
    // ส่งข้อมูล
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // ปิด channel
    
    // รับข้อมูลจาก closed channel
    // รูปแบบที่ 1: comma-ok
    for {
        v, ok := <-ch
        if !ok {
            fmt.Println("Channel closed!")
            break
        }
        fmt.Println("Received:", v)
    }
    
    // รับจาก closed + empty channel
    v, ok := <-ch
    fmt.Printf("After close: v=%d, ok=%v\n", v, ok) // v=0 (zero value), ok=false
    
    // รูปแบบที่ 2: range (แนะนำ)
    ch2 := make(chan string, 3)
    ch2 <- "a"
    ch2 <- "b"
    ch2 <- "c"
    close(ch2)
    
    for s := range ch2 { // หยุดเมื่อ channel closed
        fmt.Println(s)
    }
}
```

### ข้อควรระวังเรื่อง close

```go
package main

import (
    "fmt"
    "sync"
)

// ต้องปิด channel แค่ครั้งเดียว
// ถ้าปิดสองครั้ง -> panic!

func safeClose(ch chan int) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("channel already closed: %v", r)
        }
    }()
    close(ch)
    return nil
}

// ใช้ sync.Once เพื่อปิด channel แค่ครั้งเดียว
type SafeChannel struct {
    ch   chan int
    once sync.Once
}

func (sc *SafeChannel) Close() {
    sc.once.Do(func() {
        close(sc.ch)
    })
}

func (sc *SafeChannel) Send(v int) {
    sc.ch <- v
}

func (sc *SafeChannel) Receive() (int, bool) {
    v, ok := <-sc.ch
    return v, ok
}

func main() {
    ch := make(chan int)
    close(ch)
    
    err := safeClose(ch) // ปิด 2 ครั้ง - recover แทน panic
    fmt.Printf("Safe close error: %v\n", err)
    
    // SafeChannel
    sc := &SafeChannel{ch: make(chan int, 3)}
    sc.Send(1)
    sc.Send(2)
    sc.Close()
    sc.Close() // ไม่ panic
    
    for {
        v, ok := sc.Receive()
        if !ok {
            break
        }
        fmt.Println("Received:", v)
    }
}
```

---

## 14.7 Fan-out Pattern

แจกงานไปยัง workers หลายตัว

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Task struct {
    ID   int
    Data string
}

type Result struct {
    TaskID   int
    Output   string
    Duration time.Duration
}

func worker(id int, tasks <-chan Task, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for task := range tasks {
        start := time.Now()
        // simulate work
        time.Sleep(50 * time.Millisecond)
        
        results <- Result{
            TaskID:   task.ID,
            Output:   fmt.Sprintf("[Worker %d] processed: %s", id, task.Data),
            Duration: time.Since(start),
        }
    }
}

func fanOut(tasks []Task, numWorkers int) []Result {
    taskCh := make(chan Task, len(tasks))
    resultCh := make(chan Result, len(tasks))
    
    var wg sync.WaitGroup
    
    // Start workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, taskCh, resultCh, &wg)
    }
    
    // Send tasks
    for _, t := range tasks {
        taskCh <- t
    }
    close(taskCh)
    
    // Wait and collect
    go func() {
        wg.Wait()
        close(resultCh)
    }()
    
    var results []Result
    for r := range resultCh {
        results = append(results, r)
    }
    
    return results
}

func main() {
    tasks := make([]Task, 15)
    for i := range tasks {
        tasks[i] = Task{ID: i + 1, Data: fmt.Sprintf("task-data-%d", i+1)}
    }
    
    start := time.Now()
    results := fanOut(tasks, 4)
    elapsed := time.Since(start)
    
    for _, r := range results {
        fmt.Printf("Task %d: %s (%.0fms)\n",
            r.TaskID, r.Output, float64(r.Duration.Milliseconds()))
    }
    
    fmt.Printf("\nProcessed %d tasks in %.2fs\n", len(results), elapsed.Seconds())
}
```

---

## 14.8 Fan-in Pattern

รวม channels หลายตัวเป็น channel เดียว

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func generateNumbers(start, end int, delay time.Duration) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := start; i <= end; i++ {
            time.Sleep(delay)
            ch <- i
        }
    }()
    return ch
}

func merge(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int, 100)
    
    output := func(c <-chan int) {
        defer wg.Done()
        for v := range c {
            merged <- v
        }
    }
    
    wg.Add(len(channels))
    for _, ch := range channels {
        go output(ch)
    }
    
    go func() {
        wg.Wait()
        close(merged)
    }()
    
    return merged
}

func main() {
    // สามแหล่งข้อมูล ส่งข้อมูลด้วยความเร็วต่างกัน
    ch1 := generateNumbers(1, 5, 100*time.Millisecond)   // ช้า
    ch2 := generateNumbers(10, 15, 50*time.Millisecond)  // ปานกลาง
    ch3 := generateNumbers(100, 105, 30*time.Millisecond) // เร็ว
    
    // Merge channels
    merged := merge(ch1, ch2, ch3)
    
    sum := 0
    count := 0
    for v := range merged {
        fmt.Printf("Received: %d\n", v)
        sum += v
        count++
    }
    
    fmt.Printf("\nTotal: %d values, Sum: %d\n", count, sum)
}
```

---

## 14.9 Pipeline Pattern

เชื่อม stages หลายๆ stage เข้าด้วยกัน

```go
package main

import (
    "fmt"
    "strings"
)

// Stage 1: Generate
func generate(words ...string) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for _, w := range words {
            out <- w
        }
    }()
    return out
}

// Stage 2: To uppercase
func toUpper(in <-chan string) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for s := range in {
            out <- strings.ToUpper(s)
        }
    }()
    return out
}

// Stage 3: Filter (ความยาว > n)
func filterLong(in <-chan string, minLen int) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for s := range in {
            if len(s) > minLen {
                out <- s
            }
        }
    }()
    return out
}

// Stage 4: Add prefix
func addPrefix(in <-chan string, prefix string) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for s := range in {
            out <- prefix + s
        }
    }()
    return out
}

// Stage 5: Collect
func collect(in <-chan string) []string {
    var result []string
    for s := range in {
        result = append(result, s)
    }
    return result
}

func main() {
    words := []string{
        "go", "python", "java", "rust", "javascript",
        "c", "cpp", "swift", "kotlin", "typescript",
    }
    
    // Pipeline: generate -> toUpper -> filterLong -> addPrefix -> collect
    result := collect(
        addPrefix(
            filterLong(
                toUpper(
                    generate(words...),
                ),
                4, // ความยาวมากกว่า 4
            ),
            "LANG: ",
        ),
    )
    
    fmt.Println("Pipeline result:")
    for _, s := range result {
        fmt.Printf("  %s\n", s)
    }
}
```

---

## 14.10 Done Channel Pattern

ใช้ channel สัญญาณ "done" เพื่อหยุด goroutines

```go
package main

import (
    "fmt"
    "time"
)

// Generator ที่หยุดได้เมื่อได้รับสัญญาณ done
func counter(done <-chan struct{}) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        n := 0
        for {
            select {
            case out <- n:
                n++
            case <-done:
                fmt.Println("Counter stopped")
                return
            }
        }
    }()
    return out
}

func printer(done <-chan struct{}, in <-chan int) <-chan struct{} {
    finished := make(chan struct{})
    go func() {
        defer close(finished)
        for {
            select {
            case v, ok := <-in:
                if !ok {
                    return
                }
                fmt.Printf("Value: %d\n", v)
            case <-done:
                fmt.Println("Printer stopped")
                return
            }
        }
    }()
    return finished
}

func main() {
    done := make(chan struct{})
    
    nums := counter(done)
    finished := printer(done, nums)
    
    // รัน 500ms แล้วหยุด
    time.Sleep(500 * time.Millisecond)
    close(done) // ส่งสัญญาณหยุด
    
    <-finished // รอให้ printer หยุด
    fmt.Println("All stopped")
}
```

---

## 14.11 Timeouts ด้วย Channels

```go
package main

import (
    "fmt"
    "time"
)

// Timeout สำหรับ operation หนึ่งครั้ง
func operationWithTimeout(work func() string, timeout time.Duration) (string, error) {
    result := make(chan string, 1)
    
    go func() {
        result <- work()
    }()
    
    select {
    case r := <-result:
        return r, nil
    case <-time.After(timeout):
        return "", fmt.Errorf("operation timed out after %v", timeout)
    }
}

// Timeout แบบ context
type Request struct {
    ID   int
    Data string
}

type Response struct {
    RequestID int
    Result    string
    Error     error
}

func processWithTimeout(req Request, timeout time.Duration) Response {
    done := make(chan Response, 1)
    
    go func() {
        // Simulate variable processing time
        processingTime := time.Duration(req.ID*100) * time.Millisecond
        time.Sleep(processingTime)
        done <- Response{
            RequestID: req.ID,
            Result:    fmt.Sprintf("processed: %s", req.Data),
        }
    }()
    
    select {
    case resp := <-done:
        return resp
    case <-time.After(timeout):
        return Response{
            RequestID: req.ID,
            Error:     fmt.Errorf("request %d timed out", req.ID),
        }
    }
}

// Heartbeat pattern
func heartbeat(done <-chan struct{}, interval time.Duration) <-chan time.Time {
    ch := make(chan time.Time)
    go func() {
        defer close(ch)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        
        for {
            select {
            case t := <-ticker.C:
                select {
                case ch <- t:
                default: // drop if no one is reading
                }
            case <-done:
                return
            }
        }
    }()
    return ch
}

func main() {
    // Simple timeout
    fast := func() string {
        time.Sleep(100 * time.Millisecond)
        return "fast result"
    }
    
    slow := func() string {
        time.Sleep(600 * time.Millisecond)
        return "slow result"
    }
    
    if result, err := operationWithTimeout(fast, 500*time.Millisecond); err != nil {
        fmt.Println("Fast error:", err)
    } else {
        fmt.Println("Fast result:", result)
    }
    
    if result, err := operationWithTimeout(slow, 500*time.Millisecond); err != nil {
        fmt.Println("Slow error:", err)
    } else {
        fmt.Println("Slow result:", result)
    }
    
    fmt.Println()
    
    // Process multiple requests with timeout
    requests := []Request{
        {1, "fast-data"},
        {3, "medium-data"},
        {7, "slow-data"},
    }
    
    for _, req := range requests {
        resp := processWithTimeout(req, 500*time.Millisecond)
        if resp.Error != nil {
            fmt.Printf("Request %d: ERROR - %v\n", resp.RequestID, resp.Error)
        } else {
            fmt.Printf("Request %d: %s\n", resp.RequestID, resp.Result)
        }
    }
    
    fmt.Println()
    
    // Heartbeat
    done := make(chan struct{})
    beats := heartbeat(done, 100*time.Millisecond)
    
    count := 0
    for beat := range beats {
        fmt.Printf("Beat at: %s\n", beat.Format("15:04:05.000"))
        count++
        if count >= 3 {
            close(done)
            break
        }
    }
}
```

---

## 14.12 Or-Done Channel

```go
package main

import (
    "fmt"
    "time"
)

// orDone - wraps channel รับสัญญาณ done
func orDone(done <-chan struct{}, c <-chan int) <-chan int {
    valStream := make(chan int)
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if !ok {
                    return
                }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}

func main() {
    done := make(chan struct{})
    nums := make(chan int)
    
    go func() {
        defer close(nums)
        for i := 0; ; i++ {
            nums <- i
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    // Stop after 200ms
    time.AfterFunc(200*time.Millisecond, func() {
        close(done)
    })
    
    for n := range orDone(done, nums) {
        fmt.Printf("Received: %d\n", n)
    }
    
    fmt.Println("Done")
}
```

---

## 14.13 Bridge Channel Pattern

รับ channel-of-channels และ "flatten" เป็น channel เดียว

```go
package main

import "fmt"

func bridge(done <-chan struct{}, chanStream <-chan <-chan int) <-chan int {
    valStream := make(chan int)
    go func() {
        defer close(valStream)
        for {
            var stream <-chan int
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }
            
            for val := range stream {
                select {
                case valStream <- val:
                case <-done:
                }
            }
        }
    }()
    return valStream
}

func genVals(done <-chan struct{}) <-chan (<-chan int) {
    chanStream := make(chan (<-chan int))
    go func() {
        defer close(chanStream)
        for i := 0; i < 5; i++ {
            stream := make(chan int, 3)
            stream <- i*10
            stream <- i*10 + 1
            stream <- i*10 + 2
            close(stream)
            select {
            case chanStream <- stream:
            case <-done:
                return
            }
        }
    }()
    return chanStream
}

func main() {
    done := make(chan struct{})
    defer close(done)
    
    for v := range bridge(done, genVals(done)) {
        fmt.Printf("%d ", v)
    }
    fmt.Println()
}
```

---

## 14.14 Semaphore Pattern

จำกัดจำนวน goroutines ที่ทำงานพร้อมกัน

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Semaphore chan struct{}

func NewSemaphore(n int) Semaphore {
    return make(Semaphore, n)
}

func (s Semaphore) Acquire() {
    s <- struct{}{} // บล็อกถ้าเต็ม
}

func (s Semaphore) Release() {
    <-s // คืน slot
}

func processWithLimit(tasks []int, maxConcurrent int) {
    sem := NewSemaphore(maxConcurrent)
    var wg sync.WaitGroup
    
    for _, task := range tasks {
        wg.Add(1)
        task := task
        go func() {
            defer wg.Done()
            sem.Acquire()
            defer sem.Release()
            
            // Simulate work
            fmt.Printf("Processing task %d...\n", task)
            time.Sleep(100 * time.Millisecond)
            fmt.Printf("Done task %d\n", task)
        }()
    }
    
    wg.Wait()
}

func main() {
    tasks := make([]int, 10)
    for i := range tasks {
        tasks[i] = i + 1
    }
    
    start := time.Now()
    fmt.Println("Processing with max 3 concurrent:")
    processWithLimit(tasks, 3)
    fmt.Printf("Done in %.2fs\n", time.Since(start).Seconds())
}
```

---

## 14.15 Event Bus ด้วย Channels

```go
package main

import (
    "fmt"
    "sync"
)

type Event struct {
    Type    string
    Payload interface{}
}

type EventBus struct {
    mu          sync.RWMutex
    subscribers map[string][]chan Event
}

func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string][]chan Event),
    }
}

func (eb *EventBus) Subscribe(eventType string) <-chan Event {
    ch := make(chan Event, 10)
    
    eb.mu.Lock()
    eb.subscribers[eventType] = append(eb.subscribers[eventType], ch)
    eb.mu.Unlock()
    
    return ch
}

func (eb *EventBus) Publish(event Event) {
    eb.mu.RLock()
    subs := eb.subscribers[event.Type]
    eb.mu.RUnlock()
    
    for _, ch := range subs {
        select {
        case ch <- event:
        default:
            // Skip if subscriber is not reading
        }
    }
}

func (eb *EventBus) Close() {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    
    for _, subs := range eb.subscribers {
        for _, ch := range subs {
            close(ch)
        }
    }
}

func main() {
    bus := NewEventBus()
    var wg sync.WaitGroup
    
    // Subscribe
    userEvents := bus.Subscribe("user.created")
    orderEvents := bus.Subscribe("order.placed")
    allEvents := bus.Subscribe("user.created") // another subscriber
    
    // Start listeners
    wg.Add(3)
    
    go func() {
        defer wg.Done()
        for e := range userEvents {
            fmt.Printf("[UserService] Event: %s, Data: %v\n", e.Type, e.Payload)
        }
    }()
    
    go func() {
        defer wg.Done()
        for e := range orderEvents {
            fmt.Printf("[OrderService] Event: %s, Data: %v\n", e.Type, e.Payload)
        }
    }()
    
    go func() {
        defer wg.Done()
        for e := range allEvents {
            fmt.Printf("[Analytics] Event: %s, Data: %v\n", e.Type, e.Payload)
        }
    }()
    
    // Publish events
    bus.Publish(Event{Type: "user.created", Payload: map[string]string{"id": "1", "name": "สมชาย"}})
    bus.Publish(Event{Type: "order.placed", Payload: map[string]interface{}{"id": "ORD-001", "total": 1500}})
    bus.Publish(Event{Type: "user.created", Payload: map[string]string{"id": "2", "name": "สมหญิง"}})
    
    // Give goroutines time to process
    import "time"
    time.Sleep(100 * time.Millisecond)
    
    bus.Close()
    wg.Wait()
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| `make(chan T)` | สร้าง unbuffered channel |
| `make(chan T, n)` | สร้าง buffered channel ขนาด n |
| `ch <- v` | ส่งค่าเข้า channel |
| `v := <-ch` | รับค่าจาก channel |
| `close(ch)` | ปิด channel (ส่งได้อีกไม่ได้) |
| `for v := range ch` | รับค่าจน channel ถูกปิด |
| `select` | รอหลาย channel operations |
| `<-ch, ok` | ตรวจสอบว่า channel ถูกปิดหรือไม่ |
| `<-chan T` | Receive-only channel |
| `chan<- T` | Send-only channel |
| Fan-out | แจกงานให้ workers หลายตัว |
| Fan-in | รวม channels หลายตัว |
| Pipeline | เชื่อม stages |
| Done channel | สัญญาณหยุด goroutines |

## Resources

- [Go Tour: Channels](https://go.dev/tour/concurrency/2)
- [Go Blog: Pipelines and cancellation](https://go.dev/blog/pipelines)
- [Go by Example: Channels](https://gobyexample.com/channels)
- [Concurrency in Go (book by Katherine Cox-Buday)](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/)

---

*Part 14 จบแล้ว! ต่อไป Part 15: Packages and Modules*
