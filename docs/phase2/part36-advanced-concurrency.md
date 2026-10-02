# Part 36: Advanced Concurrency Patterns ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ Concurrency Patterns ขั้นสูงใน Go
- ใช้ Pipeline pattern สำหรับการประมวลผลข้อมูลแบบ stream
- ใช้ Fan-out/Fan-in pattern สำหรับการประมวลผลแบบขนาน
- สร้าง Generator pattern สำหรับการสร้างข้อมูล
- จัดการ Cancellation ด้วย context
- สร้าง Semaphore และ Rate Limiter ด้วย goroutines
- ออกแบบ Worker Pool pattern

---

## 1. ทบทวน Goroutines และ Channels

ก่อนเข้าสู่ patterns ขั้นสูง มาทบทวนพื้นฐานกัน

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 1: Goroutine พื้นฐาน
func basicGoroutine() {
	var wg sync.WaitGroup
	
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			fmt.Printf("Goroutine %d is running\n", id)
			time.Sleep(time.Millisecond * 100)
			fmt.Printf("Goroutine %d is done\n", id)
		}(i)
	}
	
	wg.Wait()
	fmt.Println("All goroutines completed")
}

// ตัวอย่าง 2: Buffered vs Unbuffered Channel
func channelComparison() {
	// Unbuffered channel - ต้อง sync กัน
	unbuffered := make(chan int)
	go func() {
		unbuffered <- 42 // blocks until receiver is ready
	}()
	val := <-unbuffered
	fmt.Printf("Unbuffered received: %d\n", val)
	
	// Buffered channel - ไม่ต้อง sync กัน
	buffered := make(chan int, 5)
	for i := 0; i < 5; i++ {
		buffered <- i // doesn't block
	}
	close(buffered)
	for v := range buffered {
		fmt.Printf("Buffered received: %d\n", v)
	}
}

func main() {
	fmt.Println("=== Basic Goroutine ===")
	basicGoroutine()
	
	fmt.Println("\n=== Channel Comparison ===")
	channelComparison()
}
```

---

## 2. Pipeline Pattern

Pipeline pattern คือการเชื่อมต่อ stages หลายๆ stages เข้าด้วยกัน โดยแต่ละ stage รับข้อมูลจาก channel และส่งออกผ่าน channel

```go
package main

import (
	"fmt"
	"math"
)

// ตัวอย่าง 3: Pipeline - Generate numbers
func generate(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

// ตัวอย่าง 4: Pipeline - Square numbers
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

// ตัวอย่าง 5: Pipeline - Filter numbers
func filter(in <-chan int, predicate func(int) bool) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			if predicate(n) {
				out <- n
			}
		}
	}()
	return out
}

// ตัวอย่าง 6: Pipeline - Transform numbers
func transform(in <-chan int, fn func(int) int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- fn(n)
		}
	}()
	return out
}

// ตัวอย่าง 7: Complex pipeline
func complexPipeline() {
	// สร้าง pipeline: generate -> square -> filter(>50) -> transform(sqrt)
	nums := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
	squared := square(nums)
	filtered := filter(squared, func(n int) bool { return n > 50 })
	result := transform(filtered, func(n int) int {
		return int(math.Sqrt(float64(n)))
	})
	
	fmt.Println("Pipeline results (numbers where square > 50, then sqrt back):")
	for n := range result {
		fmt.Printf("  %d\n", n)
	}
}

func main() {
	complexPipeline()
}
```

---

## 3. Fan-out / Fan-in Pattern

Fan-out: แจกงานให้ goroutines หลายตัวทำพร้อมกัน
Fan-in: รวมผลลัพธ์จาก goroutines หลายตัวเข้าด้วยกัน

```go
package main

import (
	"fmt"
	"sync"
	"time"
	"math/rand"
)

// ตัวอย่าง 8: Fan-out
func fanOut(in <-chan int, numWorkers int) []<-chan int {
	outputs := make([]<-chan int, numWorkers)
	
	for i := 0; i < numWorkers; i++ {
		out := make(chan int)
		outputs[i] = out
		
		go func(ch chan<- int, workerID int) {
			defer close(ch)
			for n := range in {
				// Simulate work
				time.Sleep(time.Millisecond * time.Duration(rand.Intn(100)))
				result := n * n // process
				fmt.Printf("Worker %d processed %d -> %d\n", workerID, n, result)
				ch <- result
			}
		}(out, i)
	}
	
	return outputs
}

// ตัวอย่าง 9: Fan-in (merge)
func fanIn(channels ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	merged := make(chan int)
	
	// Start output goroutine for each input channel
	output := func(ch <-chan int) {
		defer wg.Done()
		for n := range ch {
			merged <- n
		}
	}
	
	wg.Add(len(channels))
	for _, ch := range channels {
		go output(ch)
	}
	
	// Start goroutine to close merged channel once all inputs are done
	go func() {
		wg.Wait()
		close(merged)
	}()
	
	return merged
}

// ตัวอย่าง 10: Fan-out/Fan-in combined
func fanOutFanIn() {
	// Source
	source := make(chan int)
	go func() {
		defer close(source)
		for i := 1; i <= 10; i++ {
			source <- i
		}
	}()
	
	// Fan-out to 3 workers
	// Note: Simple fan-out distributes to channels, but Go channel doesn't support multicast
	// We need a distributor
	workers := make([]chan int, 3)
	for i := range workers {
		workers[i] = make(chan int, 5)
	}
	
	// Distributor goroutine
	go func() {
		defer func() {
			for _, w := range workers {
				close(w)
			}
		}()
		i := 0
		for n := range source {
			workers[i%len(workers)] <- n
			i++
		}
	}()
	
	// Workers process and output
	workerOutputs := make([]<-chan int, len(workers))
	for i, w := range workers {
		out := make(chan int)
		workerOutputs[i] = out
		go func(in <-chan int, out chan<- int, id int) {
			defer close(out)
			for n := range in {
				time.Sleep(time.Millisecond * time.Duration(rand.Intn(50)))
				result := n * 2
				out <- result
			}
		}(w, out, i)
	}
	
	// Fan-in all worker outputs
	merged := fanIn(workerOutputs...)
	
	fmt.Println("Fan-out/Fan-in results:")
	for result := range merged {
		fmt.Printf("  result: %d\n", result)
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())
	fanOutFanIn()
}
```

---

## 4. Generator Pattern

Generator pattern สร้าง sequence ของค่าต่างๆ ผ่าน channel

```go
package main

import (
	"fmt"
	"context"
	"math/rand"
	"time"
)

// ตัวอย่าง 11: Integer generator
func integers(ctx context.Context, start int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		for i := start; ; i++ {
			select {
			case <-ctx.Done():
				return
			case ch <- i:
			}
		}
	}()
	return ch
}

// ตัวอย่าง 12: Fibonacci generator
func fibonacci(ctx context.Context) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		a, b := 0, 1
		for {
			select {
			case <-ctx.Done():
				return
			case ch <- a:
				a, b = b, a+b
			}
		}
	}()
	return ch
}

// ตัวอย่าง 13: Random number generator
func randomNumbers(ctx context.Context, min, max int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		for {
			select {
			case <-ctx.Done():
				return
			case ch <- min + rand.Intn(max-min+1):
			}
		}
	}()
	return ch
}

// ตัวอย่าง 14: Take N from generator
func take(ctx context.Context, ch <-chan int, n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		count := 0
		for {
			select {
			case <-ctx.Done():
				return
			case v, ok := <-ch:
				if !ok {
					return
				}
				out <- v
				count++
				if count >= n {
					return
				}
			}
		}
	}()
	return out
}

// ตัวอย่าง 15: Repeat generator
func repeat(ctx context.Context, values ...interface{}) <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		defer close(ch)
		for {
			for _, v := range values {
				select {
				case <-ctx.Done():
					return
				case ch <- v:
				}
			}
		}
	}()
	return ch
}

func main() {
	rand.Seed(time.Now().UnixNano())
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	
	fmt.Println("=== First 10 integers from 5 ===")
	for n := range take(ctx, integers(ctx, 5), 10) {
		fmt.Printf("%d ", n)
	}
	fmt.Println()
	
	ctx2, cancel2 := context.WithCancel(context.Background())
	defer cancel2()
	
	fmt.Println("\n=== First 10 Fibonacci numbers ===")
	for n := range take(ctx2, fibonacci(ctx2), 10) {
		fmt.Printf("%d ", n)
	}
	fmt.Println()
	
	fmt.Println("\n=== 5 random numbers (1-100) ===")
	ctx3, cancel3 := context.WithCancel(context.Background())
	defer cancel3()
	for n := range take(ctx3, randomNumbers(ctx3, 1, 100), 5) {
		fmt.Printf("%d ", n)
	}
	fmt.Println()
}
```

---

## 5. Cancellation Pattern

Cancellation pattern ใช้ context.Context ในการยกเลิกการทำงาน

```go
package main

import (
	"context"
	"fmt"
	"time"
	"sync"
)

// ตัวอย่าง 16: Context cancellation
func longRunningTask(ctx context.Context, id int) error {
	for i := 0; i < 10; i++ {
		select {
		case <-ctx.Done():
			fmt.Printf("Task %d cancelled at step %d: %v\n", id, i, ctx.Err())
			return ctx.Err()
		default:
			fmt.Printf("Task %d: step %d\n", id, i)
			time.Sleep(200 * time.Millisecond)
		}
	}
	fmt.Printf("Task %d completed\n", id)
	return nil
}

// ตัวอย่าง 17: Context with deadline
func withDeadline() {
	deadline := time.Now().Add(1 * time.Second)
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()
	
	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			longRunningTask(ctx, id)
		}(i)
	}
	
	wg.Wait()
}

// ตัวอย่าง 18: Context with timeout
func withTimeout() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	
	done := make(chan struct{})
	go func() {
		defer close(done)
		select {
		case <-ctx.Done():
			fmt.Println("Operation timed out:", ctx.Err())
		case <-time.After(2 * time.Second):
			fmt.Println("Operation completed")
		}
	}()
	
	<-done
}

// ตัวอย่าง 19: Context value passing
type requestIDKey struct{}

func withValue() {
	ctx := context.WithValue(context.Background(), requestIDKey{}, "req-123")
	
	processRequest(ctx)
}

func processRequest(ctx context.Context) {
	reqID, ok := ctx.Value(requestIDKey{}).(string)
	if !ok {
		fmt.Println("No request ID found")
		return
	}
	fmt.Printf("Processing request: %s\n", reqID)
}

// ตัวอย่าง 20: Cascaded cancellation
func cascadedCancellation() {
	parent, parentCancel := context.WithCancel(context.Background())
	defer parentCancel()
	
	child1, _ := context.WithCancel(parent)
	child2, _ := context.WithTimeout(parent, 5*time.Second)
	
	var wg sync.WaitGroup
	
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case <-child1.Done():
			fmt.Println("Child1 cancelled:", child1.Err())
		case <-time.After(10 * time.Second):
			fmt.Println("Child1 completed normally")
		}
	}()
	
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case <-child2.Done():
			fmt.Println("Child2 cancelled:", child2.Err())
		case <-time.After(10 * time.Second):
			fmt.Println("Child2 completed normally")
		}
	}()
	
	// Cancel parent after 300ms - this cancels all children
	time.Sleep(300 * time.Millisecond)
	fmt.Println("Cancelling parent...")
	parentCancel()
	
	wg.Wait()
}

func main() {
	fmt.Println("=== Context with Deadline ===")
	withDeadline()
	
	fmt.Println("\n=== Context with Timeout ===")
	withTimeout()
	
	fmt.Println("\n=== Context with Value ===")
	withValue()
	
	fmt.Println("\n=== Cascaded Cancellation ===")
	cascadedCancellation()
}
```

---

## 6. Semaphore Pattern

Semaphore จำกัดจำนวน goroutines ที่ทำงานพร้อมกัน

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 21: Semaphore implementation
type Semaphore struct {
	ch chan struct{}
}

func NewSemaphore(maxConcurrency int) *Semaphore {
	return &Semaphore{
		ch: make(chan struct{}, maxConcurrency),
	}
}

func (s *Semaphore) Acquire() {
	s.ch <- struct{}{}
}

func (s *Semaphore) AcquireContext(ctx context.Context) error {
	select {
	case s.ch <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (s *Semaphore) Release() {
	<-s.ch
}

func (s *Semaphore) TryAcquire() bool {
	select {
	case s.ch <- struct{}{}:
		return true
	default:
		return false
	}
}

// ตัวอย่าง 22: Using semaphore for DB connections
func simulateDatabaseQuery(id int, duration time.Duration) string {
	time.Sleep(duration)
	return fmt.Sprintf("Result from query %d", id)
}

func semaphoreDemo() {
	sem := NewSemaphore(3) // max 3 concurrent database connections
	var wg sync.WaitGroup
	
	for i := 1; i <= 10; i++ {
		wg.Add(1)
		go func(queryID int) {
			defer wg.Done()
			
			sem.Acquire()
			defer sem.Release()
			
			fmt.Printf("Query %d: starting\n", queryID)
			result := simulateDatabaseQuery(queryID, 200*time.Millisecond)
			fmt.Printf("Query %d: completed - %s\n", queryID, result)
		}(i)
	}
	
	wg.Wait()
	fmt.Println("All queries completed")
}

// ตัวอย่าง 23: Weighted semaphore
type WeightedSemaphore struct {
	mu       sync.Mutex
	capacity int
	current  int
	cond     *sync.Cond
}

func NewWeightedSemaphore(capacity int) *WeightedSemaphore {
	ws := &WeightedSemaphore{capacity: capacity}
	ws.cond = sync.NewCond(&ws.mu)
	return ws
}

func (ws *WeightedSemaphore) Acquire(weight int) {
	ws.mu.Lock()
	defer ws.mu.Unlock()
	
	for ws.current+weight > ws.capacity {
		ws.cond.Wait()
	}
	ws.current += weight
}

func (ws *WeightedSemaphore) Release(weight int) {
	ws.mu.Lock()
	defer ws.mu.Unlock()
	
	ws.current -= weight
	ws.cond.Broadcast()
}

func weightedSemaphoreDemo() {
	ws := NewWeightedSemaphore(10)
	var wg sync.WaitGroup
	
	tasks := []struct {
		id     int
		weight int
	}{
		{1, 3}, {2, 5}, {3, 2}, {4, 4}, {5, 1},
		{6, 6}, {7, 2}, {8, 3}, {9, 1}, {10, 5},
	}
	
	for _, task := range tasks {
		wg.Add(1)
		go func(id, weight int) {
			defer wg.Done()
			
			ws.Acquire(weight)
			defer ws.Release(weight)
			
			fmt.Printf("Task %d (weight=%d): executing\n", id, weight)
			time.Sleep(300 * time.Millisecond)
			fmt.Printf("Task %d (weight=%d): done\n", id, weight)
		}(task.id, task.weight)
	}
	
	wg.Wait()
}

func main() {
	fmt.Println("=== Semaphore Demo ===")
	semaphoreDemo()
	
	fmt.Println("\n=== Weighted Semaphore Demo ===")
	weightedSemaphoreDemo()
}
```

---

## 7. Rate Limiter with Goroutines

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 24: Simple rate limiter using channel
type RateLimiter struct {
	tokens chan struct{}
	ticker *time.Ticker
	quit   chan struct{}
}

func NewRateLimiter(rate int, per time.Duration) *RateLimiter {
	rl := &RateLimiter{
		tokens: make(chan struct{}, rate),
		quit:   make(chan struct{}),
	}
	
	// Fill initial tokens
	for i := 0; i < rate; i++ {
		rl.tokens <- struct{}{}
	}
	
	// Refill tokens periodically
	rl.ticker = time.NewTicker(per / time.Duration(rate))
	go func() {
		for {
			select {
			case <-rl.ticker.C:
				select {
				case rl.tokens <- struct{}{}:
				default:
					// Token bucket is full
				}
			case <-rl.quit:
				return
			}
		}
	}()
	
	return rl
}

func (rl *RateLimiter) Allow() bool {
	select {
	case <-rl.tokens:
		return true
	default:
		return false
	}
}

func (rl *RateLimiter) Wait() {
	<-rl.tokens
}

func (rl *RateLimiter) Stop() {
	rl.ticker.Stop()
	close(rl.quit)
}

func rateLimiterDemo() {
	// Allow 5 requests per second
	limiter := NewRateLimiter(5, time.Second)
	defer limiter.Stop()
	
	var wg sync.WaitGroup
	results := make([]string, 20)
	
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			start := time.Now()
			limiter.Wait()
			elapsed := time.Since(start)
			results[id] = fmt.Sprintf("Request %2d: waited %v", id, elapsed.Round(time.Millisecond))
		}(i)
	}
	
	wg.Wait()
	for _, r := range results {
		fmt.Println(r)
	}
}

// ตัวอย่าง 25: Adaptive rate limiter
type AdaptiveRateLimiter struct {
	mu           sync.Mutex
	currentRate  float64
	minRate      float64
	maxRate      float64
	errorCount   int
	successCount int
	window       time.Duration
	lastAdjust   time.Time
	limiter      *time.Ticker
}

func NewAdaptiveRateLimiter(initialRate, minRate, maxRate float64) *AdaptiveRateLimiter {
	arl := &AdaptiveRateLimiter{
		currentRate: initialRate,
		minRate:     minRate,
		maxRate:     maxRate,
		window:      10 * time.Second,
		lastAdjust:  time.Now(),
		limiter:     time.NewTicker(time.Duration(float64(time.Second) / initialRate)),
	}
	return arl
}

func (arl *AdaptiveRateLimiter) Wait() {
	<-arl.limiter.C
}

func (arl *AdaptiveRateLimiter) RecordSuccess() {
	arl.mu.Lock()
	defer arl.mu.Unlock()
	arl.successCount++
	arl.maybeAdjust()
}

func (arl *AdaptiveRateLimiter) RecordError() {
	arl.mu.Lock()
	defer arl.mu.Unlock()
	arl.errorCount++
	arl.maybeAdjust()
}

func (arl *AdaptiveRateLimiter) maybeAdjust() {
	if time.Since(arl.lastAdjust) < arl.window {
		return
	}
	
	total := arl.successCount + arl.errorCount
	if total == 0 {
		return
	}
	
	errorRate := float64(arl.errorCount) / float64(total)
	
	if errorRate > 0.1 { // >10% errors, slow down
		arl.currentRate = arl.currentRate * 0.8
		if arl.currentRate < arl.minRate {
			arl.currentRate = arl.minRate
		}
	} else if errorRate < 0.05 { // <5% errors, speed up
		arl.currentRate = arl.currentRate * 1.2
		if arl.currentRate > arl.maxRate {
			arl.currentRate = arl.maxRate
		}
	}
	
	arl.limiter.Reset(time.Duration(float64(time.Second) / arl.currentRate))
	arl.errorCount = 0
	arl.successCount = 0
	arl.lastAdjust = time.Now()
	
	fmt.Printf("Rate adjusted to %.2f req/s\n", arl.currentRate)
}

func main() {
	fmt.Println("=== Rate Limiter Demo ===")
	rateLimiterDemo()
}
```

---

## 8. Worker Pool Pattern

```go
package main

import (
	"fmt"
	"sync"
	"time"
	"math/rand"
)

// ตัวอย่าง 26: Basic Worker Pool
type Job struct {
	ID   int
	Data interface{}
}

type Result struct {
	JobID  int
	Output interface{}
	Error  error
}

type WorkerPool struct {
	numWorkers int
	jobs       chan Job
	results    chan Result
	wg         sync.WaitGroup
}

func NewWorkerPool(numWorkers, jobBuffer int) *WorkerPool {
	return &WorkerPool{
		numWorkers: numWorkers,
		jobs:       make(chan Job, jobBuffer),
		results:    make(chan Result, jobBuffer),
	}
}

func (wp *WorkerPool) Start(processor func(Job) Result) {
	for i := 0; i < wp.numWorkers; i++ {
		wp.wg.Add(1)
		go func(workerID int) {
			defer wp.wg.Done()
			for job := range wp.jobs {
				result := processor(job)
				wp.results <- result
			}
		}(i)
	}
	
	// Close results when all workers are done
	go func() {
		wp.wg.Wait()
		close(wp.results)
	}()
}

func (wp *WorkerPool) Submit(job Job) {
	wp.jobs <- job
}

func (wp *WorkerPool) Close() {
	close(wp.jobs)
}

func (wp *WorkerPool) Results() <-chan Result {
	return wp.results
}

func workerPoolDemo() {
	rand.Seed(time.Now().UnixNano())
	
	pool := NewWorkerPool(3, 100)
	
	// Processor function
	processor := func(job Job) Result {
		// Simulate work
		time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
		data := job.Data.(int)
		return Result{
			JobID:  job.ID,
			Output: data * data,
		}
	}
	
	pool.Start(processor)
	
	// Submit jobs
	for i := 1; i <= 20; i++ {
		pool.Submit(Job{ID: i, Data: i})
	}
	pool.Close()
	
	// Collect results
	var results []Result
	for result := range pool.Results() {
		results = append(results, result)
	}
	
	fmt.Printf("Processed %d jobs\n", len(results))
	for _, r := range results[:5] {
		fmt.Printf("  Job %d: %d^2 = %v\n", r.JobID, r.JobID, r.Output)
	}
}

func main() {
	fmt.Println("=== Worker Pool Demo ===")
	workerPoolDemo()
}
```

---

## 9. Or-Done Pattern

```go
package main

import (
	"fmt"
	"time"
)

// ตัวอย่าง 27: orDone - read from channel until done
func orDone(done, ch <-chan interface{}) <-chan interface{} {
	valStream := make(chan interface{})
	go func() {
		defer close(valStream)
		for {
			select {
			case <-done:
				return
			case v, ok := <-ch:
				if !ok {
					return
				}
				select {
				case valStream <- v:
				case <-done:
					return
				}
			}
		}
	}()
	return valStream
}

func orDoneDemo() {
	done := make(chan interface{})
	data := make(chan interface{})
	
	// Producer
	go func() {
		for i := 0; i < 100; i++ {
			select {
			case <-done:
				return
			case data <- i:
			}
		}
		close(data)
	}()
	
	// Consumer - stop after 5 items
	count := 0
	for v := range orDone(done, data) {
		fmt.Printf("Received: %v\n", v)
		count++
		if count >= 5 {
			close(done)
			break
		}
	}
}

// ตัวอย่าง 28: Bridge pattern - flatten channel of channels
func bridge(done <-chan interface{}, chanStream <-chan <-chan interface{}) <-chan interface{} {
	valStream := make(chan interface{})
	go func() {
		defer close(valStream)
		for {
			var stream <-chan interface{}
			select {
			case <-done:
				return
			case maybeStream, ok := <-chanStream:
				if !ok {
					return
				}
				stream = maybeStream
			}
			
			for val := range orDone(done, stream) {
				select {
				case valStream <- val:
				case <-done:
					return
				}
			}
		}
	}()
	return valStream
}

func bridgeDemo() {
	genVals := func() <-chan <-chan interface{} {
		chanStream := make(chan (<-chan interface{}))
		go func() {
			defer close(chanStream)
			for i := 0; i < 3; i++ {
				stream := make(chan interface{}, 5)
				for j := 0; j < 5; j++ {
					stream <- fmt.Sprintf("stream %d, val %d", i, j)
				}
				close(stream)
				chanStream <- stream
			}
		}()
		return chanStream
	}
	
	done := make(chan interface{})
	defer close(done)
	
	fmt.Println("Bridge pattern output:")
	for v := range bridge(done, genVals()) {
		fmt.Printf("  %v\n", v)
	}
}

func main() {
	fmt.Println("=== Or-Done Pattern ===")
	orDoneDemo()
	
	fmt.Println("\n=== Bridge Pattern ===")
	bridgeDemo()
	
	_ = time.Second // keep import
}
```

---

## 10. Tee Channel Pattern

```go
package main

import (
	"fmt"
	"sync"
)

// ตัวอย่าง 29: Tee - duplicate a channel into two
func tee(done <-chan interface{}, in <-chan interface{}) (<-chan interface{}, <-chan interface{}) {
	out1 := make(chan interface{})
	out2 := make(chan interface{})
	
	go func() {
		defer close(out1)
		defer close(out2)
		
		for val := range in {
			// Create local copies to avoid closure issues
			var out1Copy, out2Copy = out1, out2
			
			// Send to both channels
			for i := 0; i < 2; i++ {
				select {
				case <-done:
					return
				case out1Copy <- val:
					out1Copy = nil
				case out2Copy <- val:
					out2Copy = nil
				}
			}
		}
	}()
	
	return out1, out2
}

func teeDemo() {
	done := make(chan interface{})
	defer close(done)
	
	// Source
	source := make(chan interface{})
	go func() {
		defer close(source)
		for i := 1; i <= 5; i++ {
			select {
			case <-done:
				return
			case source <- i:
			}
		}
	}()
	
	// Split into two channels
	out1, out2 := tee(done, source)
	
	var wg sync.WaitGroup
	
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range out1 {
			fmt.Printf("Consumer 1 received: %v\n", v)
		}
	}()
	
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range out2 {
			fmt.Printf("Consumer 2 received: %v\n", v)
		}
	}()
	
	wg.Wait()
}

// ตัวอย่าง 30: Queue/Batch pattern
func batch(done <-chan interface{}, in <-chan int, batchSize int) <-chan []int {
	out := make(chan []int)
	go func() {
		defer close(out)
		var batch []int
		for {
			select {
			case <-done:
				if len(batch) > 0 {
					out <- batch
				}
				return
			case v, ok := <-in:
				if !ok {
					if len(batch) > 0 {
						out <- batch
					}
					return
				}
				batch = append(batch, v)
				if len(batch) >= batchSize {
					out <- batch
					batch = nil
				}
			}
		}
	}()
	return out
}

func batchDemo() {
	done := make(chan interface{})
	defer close(done)
	
	source := make(chan int)
	go func() {
		defer close(source)
		for i := 1; i <= 23; i++ {
			source <- i
		}
	}()
	
	fmt.Println("Batched output (batch size 5):")
	for b := range batch(done, source, 5) {
		fmt.Printf("  Batch: %v\n", b)
	}
}

func main() {
	fmt.Println("=== Tee Pattern ===")
	teeDemo()
	
	fmt.Println("\n=== Batch Pattern ===")
	batchDemo()
}
```

---

## 11. Error Handling in Concurrent Code

```go
package main

import (
	"errors"
	"fmt"
	"sync"
)

// ตัวอย่าง 31: Error group pattern
type Result struct {
	Value int
	Err   error
}

func processWithError(id int) Result {
	if id%3 == 0 {
		return Result{Err: fmt.Errorf("error processing item %d", id)}
	}
	return Result{Value: id * id}
}

func concurrentWithErrors() {
	jobs := make(chan int, 20)
	results := make(chan Result, 20)
	
	var wg sync.WaitGroup
	
	// Start workers
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobs {
				results <- processWithError(job)
			}
		}()
	}
	
	// Send jobs
	for i := 1; i <= 10; i++ {
		jobs <- i
	}
	close(jobs)
	
	// Close results when done
	go func() {
		wg.Wait()
		close(results)
	}()
	
	// Collect results and errors
	var errs []error
	var values []int
	
	for r := range results {
		if r.Err != nil {
			errs = append(errs, r.Err)
		} else {
			values = append(values, r.Value)
		}
	}
	
	fmt.Printf("Successful results: %v\n", values)
	fmt.Printf("Errors: %v\n", errs)
}

// ตัวอย่าง 32: First error wins pattern
func firstErrorWins(tasks []func() error) error {
	errCh := make(chan error, len(tasks))
	var wg sync.WaitGroup
	
	for _, task := range tasks {
		wg.Add(1)
		go func(t func() error) {
			defer wg.Done()
			if err := t(); err != nil {
				errCh <- err
			}
		}(task)
	}
	
	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()
	
	select {
	case err := <-errCh:
		return err
	case <-done:
		return nil
	}
}

func main() {
	fmt.Println("=== Concurrent Error Handling ===")
	concurrentWithErrors()
	
	fmt.Println("\n=== First Error Wins ===")
	tasks := []func() error{
		func() error { return nil },
		func() error { return errors.New("task 2 failed") },
		func() error { return nil },
	}
	
	if err := firstErrorWins(tasks); err != nil {
		fmt.Printf("Got error: %v\n", err)
	}
}
```

---

## Workshop: Building a Concurrent Data Processing Pipeline

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Workshop: Data processing pipeline with error handling and cancellation

type DataRecord struct {
	ID    int
	Value float64
}

type ProcessedRecord struct {
	ID       int
	Original float64
	Result   float64
	Worker   int
}

// Stage 1: Data source
func dataSource(ctx context.Context, count int) <-chan DataRecord {
	out := make(chan DataRecord)
	go func() {
		defer close(out)
		for i := 0; i < count; i++ {
			record := DataRecord{
				ID:    i + 1,
				Value: rand.Float64() * 100,
			}
			select {
			case <-ctx.Done():
				return
			case out <- record:
			}
		}
	}()
	return out
}

// Stage 2: Validate records
func validateRecords(ctx context.Context, in <-chan DataRecord) (<-chan DataRecord, <-chan error) {
	out := make(chan DataRecord)
	errs := make(chan error, 10)
	
	go func() {
		defer close(out)
		defer close(errs)
		
		for record := range in {
			select {
			case <-ctx.Done():
				return
			default:
			}
			
			if record.Value < 0 {
				errs <- fmt.Errorf("invalid record %d: negative value", record.ID)
				continue
			}
			
			select {
			case out <- record:
			case <-ctx.Done():
				return
			}
		}
	}()
	
	return out, errs
}

// Stage 3: Process records (fan-out to multiple workers)
func processRecords(ctx context.Context, in <-chan DataRecord, numWorkers int) <-chan ProcessedRecord {
	out := make(chan ProcessedRecord)
	var wg sync.WaitGroup
	
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for record := range in {
				select {
				case <-ctx.Done():
					return
				default:
				}
				
				// Simulate processing
				time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond)
				result := ProcessedRecord{
					ID:       record.ID,
					Original: record.Value,
					Result:   record.Value * 2.5, // some transformation
					Worker:   workerID,
				}
				
				select {
				case out <- result:
				case <-ctx.Done():
					return
				}
			}
		}(i + 1)
	}
	
	go func() {
		wg.Wait()
		close(out)
	}()
	
	return out
}

// Stage 4: Aggregate results
func aggregateResults(ctx context.Context, in <-chan ProcessedRecord) <-chan map[string]float64 {
	out := make(chan map[string]float64, 1)
	
	go func() {
		defer close(out)
		
		stats := map[string]float64{
			"count": 0,
			"sum":   0,
			"min":   1e18,
			"max":   -1e18,
		}
		
		for record := range in {
			select {
			case <-ctx.Done():
				return
			default:
			}
			
			stats["count"]++
			stats["sum"] += record.Result
			if record.Result < stats["min"] {
				stats["min"] = record.Result
			}
			if record.Result > stats["max"] {
				stats["max"] = record.Result
			}
		}
		
		if stats["count"] > 0 {
			stats["avg"] = stats["sum"] / stats["count"]
		}
		
		out <- stats
	}()
	
	return out
}

func main() {
	rand.Seed(time.Now().UnixNano())
	
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	
	fmt.Println("=== Data Processing Pipeline Workshop ===")
	
	// Build pipeline
	records := dataSource(ctx, 100)
	validated, errs := validateRecords(ctx, records)
	processed := processRecords(ctx, validated, 4)
	aggregated := aggregateResults(ctx, processed)
	
	// Handle errors in background
	go func() {
		for err := range errs {
			fmt.Printf("Validation error: %v\n", err)
		}
	}()
	
	// Get final stats
	if stats, ok := <-aggregated; ok {
		fmt.Printf("\nPipeline Statistics:\n")
		fmt.Printf("  Records processed: %.0f\n", stats["count"])
		fmt.Printf("  Sum: %.2f\n", stats["sum"])
		fmt.Printf("  Average: %.2f\n", stats["avg"])
		fmt.Printf("  Min: %.2f\n", stats["min"])
		fmt.Printf("  Max: %.2f\n", stats["max"])
	}
}
```

---

## สรุป

ใน Part 36 เราได้เรียนรู้:

1. **Pipeline Pattern**: การเชื่อมต่อ stages ผ่าน channels สำหรับการประมวลผลข้อมูลแบบ stream
2. **Fan-out/Fan-in**: การกระจายงานและรวบรวมผลลัพธ์จาก goroutines หลายตัว
3. **Generator Pattern**: การสร้าง sequence ของข้อมูลผ่าน channel
4. **Cancellation Pattern**: การใช้ context.Context เพื่อยกเลิกการทำงาน
5. **Semaphore Pattern**: การจำกัดจำนวน goroutines ที่ทำงานพร้อมกัน
6. **Rate Limiter**: การควบคุมอัตราการทำงาน
7. **Worker Pool**: การจัดการ pool ของ goroutines

---

## Resources

- [Go Concurrency Patterns](https://go.dev/blog/pipelines)
- [Advanced Go Concurrency Patterns](https://go.dev/blog/advanced-go-concurrency-patterns)
- [Go by Example: Goroutines](https://gobyexample.com/goroutines)
- [Go by Example: Channels](https://gobyexample.com/channels)
- [Concurrency in Go (Book)](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/)
