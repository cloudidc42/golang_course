# Part 37: Worker Pools ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- สร้าง Worker Pool ที่มีประสิทธิภาพ
- จัดการ Job Queue และ Result Aggregation
- จัดการ Error Handling ใน Worker Pool
- ปรับขนาด Pool แบบ Dynamic
- Shutdown Pool อย่าง Graceful
- Monitor Pool Metrics
- สร้าง Image Processing Pool (Workshop)

---

## 1. ทำความเข้าใจ Worker Pool

Worker Pool คือ pattern ที่ใช้ goroutines จำนวนคงที่ ประมวลผลงานที่เข้ามาในคิว แทนที่จะสร้าง goroutine ใหม่สำหรับทุก task ซึ่งช่วยควบคุม resource consumption

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 1: Basic Worker Pool
type Task struct {
	ID   int
	Data int
}

type TaskResult struct {
	TaskID int
	Result int
	Error  error
}

func basicWorkerPool() {
	numWorkers := 3
	numJobs := 15
	
	jobs := make(chan Task, numJobs)
	results := make(chan TaskResult, numJobs)
	
	// Start workers
	var wg sync.WaitGroup
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for task := range jobs {
				// Simulate work
				time.Sleep(50 * time.Millisecond)
				result := TaskResult{
					TaskID: task.ID,
					Result: task.Data * task.Data,
				}
				results <- result
				fmt.Printf("Worker %d processed task %d\n", workerID, task.ID)
			}
		}(w)
	}
	
	// Send jobs
	for i := 1; i <= numJobs; i++ {
		jobs <- Task{ID: i, Data: i}
	}
	close(jobs)
	
	// Close results when all workers done
	go func() {
		wg.Wait()
		close(results)
	}()
	
	// Collect results
	var allResults []TaskResult
	for r := range results {
		allResults = append(allResults, r)
	}
	
	fmt.Printf("\nProcessed %d tasks\n", len(allResults))
}

func main() {
	basicWorkerPool()
}
```

---

## 2. Advanced Worker Pool Implementation

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// ตัวอย่าง 2: Full-featured Worker Pool

type Job interface {
	Execute() (interface{}, error)
}

type JobResult struct {
	Output interface{}
	Error  error
}

type WorkerPool struct {
	ctx        context.Context
	cancel     context.CancelFunc
	numWorkers int
	jobQueue   chan Job
	results    chan JobResult
	wg         sync.WaitGroup
	mu         sync.RWMutex
	
	// Metrics
	totalJobs     int64
	completedJobs int64
	failedJobs    int64
	activeWorkers int64
	
	started bool
	stopped bool
}

func NewWorkerPool(ctx context.Context, numWorkers, queueSize int) *WorkerPool {
	poolCtx, cancel := context.WithCancel(ctx)
	return &WorkerPool{
		ctx:        poolCtx,
		cancel:     cancel,
		numWorkers: numWorkers,
		jobQueue:   make(chan Job, queueSize),
		results:    make(chan JobResult, queueSize),
	}
}

func (wp *WorkerPool) Start() {
	wp.mu.Lock()
	defer wp.mu.Unlock()
	
	if wp.started {
		return
	}
	wp.started = true
	
	for i := 0; i < wp.numWorkers; i++ {
		wp.wg.Add(1)
		go wp.worker(i + 1)
	}
	
	go func() {
		wp.wg.Wait()
		close(wp.results)
	}()
}

func (wp *WorkerPool) worker(id int) {
	defer wp.wg.Done()
	atomic.AddInt64(&wp.activeWorkers, 1)
	defer atomic.AddInt64(&wp.activeWorkers, -1)
	
	for {
		select {
		case <-wp.ctx.Done():
			return
		case job, ok := <-wp.jobQueue:
			if !ok {
				return
			}
			
			output, err := job.Execute()
			result := JobResult{Output: output, Error: err}
			
			if err != nil {
				atomic.AddInt64(&wp.failedJobs, 1)
			} else {
				atomic.AddInt64(&wp.completedJobs, 1)
			}
			
			select {
			case wp.results <- result:
			case <-wp.ctx.Done():
				return
			}
		}
	}
}

func (wp *WorkerPool) Submit(job Job) bool {
	atomic.AddInt64(&wp.totalJobs, 1)
	select {
	case wp.jobQueue <- job:
		return true
	case <-wp.ctx.Done():
		return false
	}
}

func (wp *WorkerPool) Stop() {
	wp.mu.Lock()
	defer wp.mu.Unlock()
	
	if wp.stopped {
		return
	}
	wp.stopped = true
	close(wp.jobQueue)
}

func (wp *WorkerPool) Cancel() {
	wp.cancel()
}

func (wp *WorkerPool) Results() <-chan JobResult {
	return wp.results
}

func (wp *WorkerPool) Metrics() map[string]int64 {
	return map[string]int64{
		"total":     atomic.LoadInt64(&wp.totalJobs),
		"completed": atomic.LoadInt64(&wp.completedJobs),
		"failed":    atomic.LoadInt64(&wp.failedJobs),
		"active":    atomic.LoadInt64(&wp.activeWorkers),
	}
}

// ตัวอย่าง 3: Sample job implementations
type MathJob struct {
	A, B int
	Op   string
}

func (j MathJob) Execute() (interface{}, error) {
	time.Sleep(10 * time.Millisecond) // simulate work
	switch j.Op {
	case "add":
		return j.A + j.B, nil
	case "multiply":
		return j.A * j.B, nil
	case "divide":
		if j.B == 0 {
			return nil, fmt.Errorf("division by zero")
		}
		return j.A / j.B, nil
	default:
		return nil, fmt.Errorf("unknown operation: %s", j.Op)
	}
}

func advancedWorkerPoolDemo() {
	ctx := context.Background()
	pool := NewWorkerPool(ctx, 5, 100)
	pool.Start()
	
	// Submit various jobs
	jobs := []MathJob{
		{A: 10, B: 2, Op: "add"},
		{A: 10, B: 2, Op: "multiply"},
		{A: 10, B: 2, Op: "divide"},
		{A: 10, B: 0, Op: "divide"}, // will fail
		{A: 5, B: 5, Op: "add"},
		{A: 3, B: 4, Op: "multiply"},
		{A: 20, B: 4, Op: "divide"},
	}
	
	for _, job := range jobs {
		pool.Submit(job)
	}
	pool.Stop()
	
	// Collect results
	for result := range pool.Results() {
		if result.Error != nil {
			fmt.Printf("Error: %v\n", result.Error)
		} else {
			fmt.Printf("Result: %v\n", result.Output)
		}
	}
	
	metrics := pool.Metrics()
	fmt.Printf("\nMetrics: total=%d, completed=%d, failed=%d\n",
		metrics["total"], metrics["completed"], metrics["failed"])
}

func main() {
	advancedWorkerPoolDemo()
}
```

---

## 3. Job Queue with Priority

```go
package main

import (
	"container/heap"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 4: Priority Job Queue

type PriorityJob struct {
	ID       int
	Priority int // higher = more important
	Data     string
	index    int // used by heap interface
}

// PriorityQueue implements heap.Interface
type PriorityQueue []*PriorityJob

func (pq PriorityQueue) Len() int { return len(pq) }
func (pq PriorityQueue) Less(i, j int) bool {
	return pq[i].Priority > pq[j].Priority // max-heap
}
func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}
func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*PriorityJob)
	item.index = n
	*pq = append(*pq, item)
}
func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	old[n-1] = nil
	item.index = -1
	*pq = old[:n-1]
	return item
}

type PriorityWorkerPool struct {
	mu      sync.Mutex
	cond    *sync.Cond
	queue   PriorityQueue
	workers int
	quit    chan struct{}
	closed  bool
}

func NewPriorityWorkerPool(workers int) *PriorityWorkerPool {
	p := &PriorityWorkerPool{
		workers: workers,
		quit:    make(chan struct{}),
	}
	p.cond = sync.NewCond(&p.mu)
	heap.Init(&p.queue)
	return p
}

func (p *PriorityWorkerPool) Submit(job *PriorityJob) {
	p.mu.Lock()
	defer p.mu.Unlock()
	heap.Push(&p.queue, job)
	p.cond.Signal()
}

func (p *PriorityWorkerPool) Start() {
	var wg sync.WaitGroup
	for i := 0; i < p.workers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for {
				p.mu.Lock()
				for p.queue.Len() == 0 && !p.closed {
					p.cond.Wait()
				}
				
				if p.closed && p.queue.Len() == 0 {
					p.mu.Unlock()
					return
				}
				
				job := heap.Pop(&p.queue).(*PriorityJob)
				p.mu.Unlock()
				
				// Process job
				fmt.Printf("Worker %d processing job %d (priority=%d): %s\n",
					workerID, job.ID, job.Priority, job.Data)
				time.Sleep(50 * time.Millisecond)
			}
		}(i + 1)
	}
	wg.Wait()
}

func (p *PriorityWorkerPool) Stop() {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.closed = true
	p.cond.Broadcast()
}

func priorityQueueDemo() {
	pool := NewPriorityWorkerPool(2)
	
	go func() {
		jobs := []*PriorityJob{
			{ID: 1, Priority: 1, Data: "low priority task 1"},
			{ID: 2, Priority: 5, Data: "high priority task 1"},
			{ID: 3, Priority: 3, Data: "medium priority task 1"},
			{ID: 4, Priority: 5, Data: "high priority task 2"},
			{ID: 5, Priority: 1, Data: "low priority task 2"},
			{ID: 6, Priority: 10, Data: "critical task"},
		}
		
		for _, job := range jobs {
			pool.Submit(job)
			time.Sleep(10 * time.Millisecond) // simulate job arrival
		}
		
		time.Sleep(200 * time.Millisecond)
		pool.Stop()
	}()
	
	pool.Start()
}

func main() {
	fmt.Println("=== Priority Worker Pool ===")
	priorityQueueDemo()
}
```

---

## 4. Result Aggregation

```go
package main

import (
	"fmt"
	"sync"
	"time"
	"math/rand"
)

// ตัวอย่าง 5: Result aggregation with statistics

type ProcessResult struct {
	WorkerID  int
	JobID     int
	Value     float64
	Duration  time.Duration
	Success   bool
	Error     error
}

type AggregatedStats struct {
	mu sync.RWMutex
	
	TotalJobs      int
	SuccessfulJobs int
	FailedJobs     int
	
	TotalDuration    time.Duration
	MinDuration      time.Duration
	MaxDuration      time.Duration
	AverageDuration  time.Duration
	
	TotalValue float64
	MinValue   float64
	MaxValue   float64
	AvgValue   float64
	
	ResultsByWorker map[int]int
}

func NewAggregatedStats() *AggregatedStats {
	return &AggregatedStats{
		ResultsByWorker: make(map[int]int),
		MinDuration:     time.Duration(1<<63 - 1),
		MinValue:        1e18,
		MaxValue:        -1e18,
	}
}

func (s *AggregatedStats) Add(result ProcessResult) {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	s.TotalJobs++
	if result.Success {
		s.SuccessfulJobs++
		s.TotalValue += result.Value
		if result.Value < s.MinValue {
			s.MinValue = result.Value
		}
		if result.Value > s.MaxValue {
			s.MaxValue = result.Value
		}
	} else {
		s.FailedJobs++
	}
	
	s.TotalDuration += result.Duration
	if result.Duration < s.MinDuration {
		s.MinDuration = result.Duration
	}
	if result.Duration > s.MaxDuration {
		s.MaxDuration = result.Duration
	}
	
	s.ResultsByWorker[result.WorkerID]++
}

func (s *AggregatedStats) Finalize() {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	if s.TotalJobs > 0 {
		s.AverageDuration = s.TotalDuration / time.Duration(s.TotalJobs)
	}
	if s.SuccessfulJobs > 0 {
		s.AvgValue = s.TotalValue / float64(s.SuccessfulJobs)
	}
}

func (s *AggregatedStats) Print() {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	fmt.Printf("=== Aggregated Statistics ===\n")
	fmt.Printf("Total Jobs:      %d\n", s.TotalJobs)
	fmt.Printf("Successful:      %d\n", s.SuccessfulJobs)
	fmt.Printf("Failed:          %d\n", s.FailedJobs)
	fmt.Printf("Success Rate:    %.1f%%\n", float64(s.SuccessfulJobs)/float64(s.TotalJobs)*100)
	fmt.Printf("\nDuration Stats:\n")
	fmt.Printf("  Average:       %v\n", s.AverageDuration)
	fmt.Printf("  Min:           %v\n", s.MinDuration)
	fmt.Printf("  Max:           %v\n", s.MaxDuration)
	fmt.Printf("\nValue Stats:\n")
	fmt.Printf("  Average:       %.2f\n", s.AvgValue)
	fmt.Printf("  Min:           %.2f\n", s.MinValue)
	fmt.Printf("  Max:           %.2f\n", s.MaxValue)
	fmt.Printf("\nJobs per Worker:\n")
	for workerID, count := range s.ResultsByWorker {
		fmt.Printf("  Worker %d: %d jobs\n", workerID, count)
	}
}

func aggregationDemo() {
	rand.Seed(time.Now().UnixNano())
	
	jobs := make(chan int, 100)
	results := make(chan ProcessResult, 100)
	stats := NewAggregatedStats()
	
	var wg sync.WaitGroup
	
	// Start 4 workers
	for i := 1; i <= 4; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for jobID := range jobs {
				start := time.Now()
				
				// Simulate variable work
				duration := time.Duration(rand.Intn(100)+10) * time.Millisecond
				time.Sleep(duration)
				
				// Random failures (10% chance)
				success := rand.Float64() > 0.1
				var value float64
				var err error
				if success {
					value = rand.Float64() * 1000
				} else {
					err = fmt.Errorf("worker %d failed on job %d", workerID, jobID)
				}
				
				results <- ProcessResult{
					WorkerID: workerID,
					JobID:    jobID,
					Value:    value,
					Duration: time.Since(start),
					Success:  success,
					Error:    err,
				}
			}
		}(i)
	}
	
	// Send 50 jobs
	for i := 1; i <= 50; i++ {
		jobs <- i
	}
	close(jobs)
	
	// Close results when all workers done
	go func() {
		wg.Wait()
		close(results)
	}()
	
	// Aggregate results
	for result := range results {
		stats.Add(result)
		if result.Error != nil {
			fmt.Printf("Error: %v\n", result.Error)
		}
	}
	
	stats.Finalize()
	stats.Print()
}

func main() {
	aggregationDemo()
}
```

---

## 5. Error Handling in Worker Pools

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
	"context"
)

// ตัวอย่าง 6: Error handling strategies

type RetryableJob struct {
	ID       int
	MaxRetry int
	fn       func() error
}

func (j RetryableJob) Execute() error {
	return j.fn()
}

type RetryConfig struct {
	MaxRetries int
	Delay      time.Duration
	MaxDelay   time.Duration
	Multiplier float64
}

func withRetry(job func() error, config RetryConfig) error {
	delay := config.Delay
	
	for attempt := 0; attempt <= config.MaxRetries; attempt++ {
		err := job()
		if err == nil {
			return nil
		}
		
		// Check if error is retryable
		var nonRetryable *NonRetryableError
		if errors.As(err, &nonRetryable) {
			return err
		}
		
		if attempt < config.MaxRetries {
			fmt.Printf("  Attempt %d failed: %v, retrying in %v...\n", attempt+1, err, delay)
			time.Sleep(delay)
			delay = time.Duration(float64(delay) * config.Multiplier)
			if delay > config.MaxDelay {
				delay = config.MaxDelay
			}
		} else {
			return fmt.Errorf("all %d attempts failed: %w", config.MaxRetries+1, err)
		}
	}
	return nil
}

type NonRetryableError struct {
	msg string
}

func (e *NonRetryableError) Error() string {
	return e.msg
}

// ตัวอย่าง 7: Worker pool with retry
func retryWorkerPool() {
	ctx := context.Background()
	jobs := make(chan RetryableJob, 20)
	
	retryConfig := RetryConfig{
		MaxRetries: 3,
		Delay:      10 * time.Millisecond,
		MaxDelay:   100 * time.Millisecond,
		Multiplier: 2.0,
	}
	
	var wg sync.WaitGroup
	errors_ch := make(chan error, 20)
	
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for job := range jobs {
				select {
				case <-ctx.Done():
					return
				default:
				}
				
				err := withRetry(func() error {
					return job.Execute()
				}, retryConfig)
				
				if err != nil {
					errors_ch <- fmt.Errorf("job %d: %w", job.ID, err)
				} else {
					fmt.Printf("Worker %d: job %d succeeded\n", workerID, job.ID)
				}
			}
		}(i + 1)
	}
	
	// Simulate jobs with varying reliability
	failCount := make(map[int]int)
	mu := &sync.Mutex{}
	
	for i := 1; i <= 10; i++ {
		jobID := i
		jobs <- RetryableJob{
			ID: jobID,
			fn: func() error {
				mu.Lock()
				failCount[jobID]++
				count := failCount[jobID]
				mu.Unlock()
				
				// Fail first 2 attempts for even job IDs
				if jobID%2 == 0 && count <= 2 {
					return fmt.Errorf("transient error")
				}
				// Never succeed for job 10
				if jobID == 10 {
					return fmt.Errorf("persistent error")
				}
				return nil
			},
		}
	}
	close(jobs)
	
	go func() {
		wg.Wait()
		close(errors_ch)
	}()
	
	fmt.Println("\nErrors:")
	for err := range errors_ch {
		fmt.Printf("  %v\n", err)
	}
}

func main() {
	fmt.Println("=== Retry Worker Pool ===")
	retryWorkerPool()
}
```

---

## 6. Dynamic Pool Sizing

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// ตัวอย่าง 8: Dynamic worker pool that scales up/down

type DynamicPool struct {
	ctx        context.Context
	cancel     context.CancelFunc
	
	minWorkers int
	maxWorkers int
	
	jobQueue     chan func()
	workerCount  int64
	activeCount  int64
	
	scaleUpThreshold   float64 // scale up if active/total > threshold
	scaleDownThreshold float64 // scale down if active/total < threshold
	
	mu      sync.Mutex
	workers map[int64]context.CancelFunc
	nextID  int64
}

func NewDynamicPool(ctx context.Context, min, max int) *DynamicPool {
	poolCtx, cancel := context.WithCancel(ctx)
	p := &DynamicPool{
		ctx:                poolCtx,
		cancel:             cancel,
		minWorkers:         min,
		maxWorkers:         max,
		jobQueue:           make(chan func(), 1000),
		scaleUpThreshold:   0.8,
		scaleDownThreshold: 0.3,
		workers:            make(map[int64]context.CancelFunc),
	}
	
	// Start minimum workers
	for i := 0; i < min; i++ {
		p.addWorker()
	}
	
	// Start monitor
	go p.monitor()
	
	return p
}

func (p *DynamicPool) addWorker() {
	workerCtx, workerCancel := context.WithCancel(p.ctx)
	
	p.mu.Lock()
	id := p.nextID
	p.nextID++
	p.workers[id] = workerCancel
	p.mu.Unlock()
	
	atomic.AddInt64(&p.workerCount, 1)
	
	go func() {
		defer func() {
			p.mu.Lock()
			delete(p.workers, id)
			p.mu.Unlock()
			atomic.AddInt64(&p.workerCount, -1)
		}()
		
		for {
			select {
			case <-workerCtx.Done():
				return
			case job, ok := <-p.jobQueue:
				if !ok {
					return
				}
				atomic.AddInt64(&p.activeCount, 1)
				job()
				atomic.AddInt64(&p.activeCount, -1)
			}
		}
	}()
}

func (p *DynamicPool) removeWorker() {
	p.mu.Lock()
	defer p.mu.Unlock()
	
	currentWorkers := int(atomic.LoadInt64(&p.workerCount))
	if currentWorkers <= p.minWorkers {
		return
	}
	
	// Cancel one worker
	for id, cancel := range p.workers {
		cancel()
		delete(p.workers, id)
		break
	}
}

func (p *DynamicPool) monitor() {
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()
	
	for {
		select {
		case <-p.ctx.Done():
			return
		case <-ticker.C:
			workers := int(atomic.LoadInt64(&p.workerCount))
			active := int(atomic.LoadInt64(&p.activeCount))
			
			if workers == 0 {
				continue
			}
			
			utilization := float64(active) / float64(workers)
			queueLen := len(p.jobQueue)
			
			// Scale up
			if (utilization > p.scaleUpThreshold || queueLen > workers*2) && workers < p.maxWorkers {
				fmt.Printf("[Monitor] Scaling up: workers=%d, active=%d, queue=%d\n",
					workers, active, queueLen)
				p.addWorker()
			}
			
			// Scale down
			if utilization < p.scaleDownThreshold && workers > p.minWorkers && queueLen == 0 {
				fmt.Printf("[Monitor] Scaling down: workers=%d, active=%d\n",
					workers, active)
				p.removeWorker()
			}
		}
	}
}

func (p *DynamicPool) Submit(job func()) bool {
	select {
	case p.jobQueue <- job:
		return true
	case <-p.ctx.Done():
		return false
	}
}

func (p *DynamicPool) Stop() {
	p.cancel()
}

func dynamicPoolDemo() {
	ctx := context.Background()
	pool := NewDynamicPool(ctx, 2, 10)
	defer pool.Stop()
	
	var wg sync.WaitGroup
	
	// Simulate burst of work
	fmt.Println("Phase 1: Heavy load (50 jobs)")
	for i := 0; i < 50; i++ {
		wg.Add(1)
		jobID := i
		pool.Submit(func() {
			defer wg.Done()
			time.Sleep(100 * time.Millisecond)
			_ = jobID
		})
	}
	wg.Wait()
	
	// Let it scale down
	time.Sleep(1 * time.Second)
	fmt.Printf("\nAfter scale down: workers=%d\n",
		atomic.LoadInt64(&pool.workerCount))
	
	// Another burst
	fmt.Println("\nPhase 2: Another burst (20 jobs)")
	for i := 0; i < 20; i++ {
		wg.Add(1)
		pool.Submit(func() {
			defer wg.Done()
			time.Sleep(50 * time.Millisecond)
		})
	}
	wg.Wait()
	
	fmt.Printf("\nFinal worker count: %d\n",
		atomic.LoadInt64(&pool.workerCount))
}

func main() {
	dynamicPoolDemo()
}
```

---

## 7. Graceful Shutdown

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// ตัวอย่าง 9: Graceful shutdown with drain

type GracefulPool struct {
	ctx         context.Context
	cancel      context.CancelFunc
	shutdownCtx context.Context
	shutdown    context.CancelFunc
	
	jobs    chan func() error
	wg      sync.WaitGroup
	results chan error
	
	drainTimeout time.Duration
}

func NewGracefulPool(numWorkers int, drainTimeout time.Duration) *GracefulPool {
	ctx, cancel := context.WithCancel(context.Background())
	shutdownCtx, shutdown := context.WithCancel(context.Background())
	
	p := &GracefulPool{
		ctx:          ctx,
		cancel:       cancel,
		shutdownCtx:  shutdownCtx,
		shutdown:     shutdown,
		jobs:         make(chan func() error, 1000),
		results:      make(chan error, 1000),
		drainTimeout: drainTimeout,
	}
	
	for i := 0; i < numWorkers; i++ {
		p.wg.Add(1)
		go p.worker(i + 1)
	}
	
	go func() {
		p.wg.Wait()
		close(p.results)
	}()
	
	return p
}

func (p *GracefulPool) worker(id int) {
	defer p.wg.Done()
	
	for {
		select {
		case job, ok := <-p.jobs:
			if !ok {
				fmt.Printf("Worker %d: shutting down (channel closed)\n", id)
				return
			}
			if err := job(); err != nil {
				p.results <- err
			}
		case <-p.ctx.Done():
			// Drain remaining jobs
			fmt.Printf("Worker %d: received cancellation, draining...\n", id)
			for {
				select {
				case job, ok := <-p.jobs:
					if !ok || job == nil {
						fmt.Printf("Worker %d: drain complete\n", id)
						return
					}
					if err := job(); err != nil {
						p.results <- err
					}
				default:
					fmt.Printf("Worker %d: nothing to drain, exiting\n", id)
					return
				}
			}
		}
	}
}

func (p *GracefulPool) Submit(job func() error) bool {
	select {
	case <-p.shutdownCtx.Done():
		return false
	case p.jobs <- job:
		return true
	}
}

func (p *GracefulPool) GracefulStop() error {
	fmt.Println("Initiating graceful shutdown...")
	
	// Stop accepting new jobs
	p.shutdown()
	
	// Signal workers to drain
	p.cancel()
	
	// Wait for drain with timeout
	done := make(chan struct{})
	go func() {
		p.wg.Wait()
		close(done)
	}()
	
	select {
	case <-done:
		fmt.Println("All workers drained successfully")
		return nil
	case <-time.After(p.drainTimeout):
		return fmt.Errorf("graceful shutdown timed out after %v", p.drainTimeout)
	}
}

func (p *GracefulPool) Results() <-chan error {
	return p.results
}

func gracefulShutdownDemo() {
	pool := NewGracefulPool(3, 5*time.Second)
	
	// Handle OS signals
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	
	var wg sync.WaitGroup
	
	// Submit jobs
	go func() {
		for i := 0; i < 20; i++ {
			jobID := i + 1
			ok := pool.Submit(func() error {
				time.Sleep(200 * time.Millisecond)
				fmt.Printf("Job %d completed\n", jobID)
				return nil
			})
			if !ok {
				fmt.Printf("Job %d: pool is shutting down, not submitting\n", jobID)
				break
			}
			time.Sleep(50 * time.Millisecond)
		}
		close(sigCh) // simulate shutdown after jobs submitted
	}()
	
	// Wait for signal
	wg.Add(1)
	go func() {
		defer wg.Done()
		<-sigCh
		fmt.Println("\nShutdown signal received")
		if err := pool.GracefulStop(); err != nil {
			fmt.Printf("Shutdown error: %v\n", err)
		}
	}()
	
	// Collect errors
	go func() {
		for err := range pool.Results() {
			fmt.Printf("Error: %v\n", err)
		}
	}()
	
	wg.Wait()
	fmt.Println("Server stopped")
}

func main() {
	gracefulShutdownDemo()
}
```

---

## 8. Monitoring Pool Metrics

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// ตัวอย่าง 10: Pool with comprehensive metrics

type PoolMetrics struct {
	mu sync.RWMutex
	
	// Counters
	TotalSubmitted  int64
	TotalCompleted  int64
	TotalFailed     int64
	TotalDropped    int64
	
	// Gauges
	ActiveWorkers   int64
	QueueDepth      int64
	
	// Histograms (simplified)
	DurationBuckets map[string]int64
	
	// Rates (jobs per second)
	startTime       time.Time
}

func NewPoolMetrics() *PoolMetrics {
	return &PoolMetrics{
		DurationBuckets: map[string]int64{
			"<10ms":  0,
			"<50ms":  0,
			"<100ms": 0,
			"<500ms": 0,
			">=500ms": 0,
		},
		startTime: time.Now(),
	}
}

func (m *PoolMetrics) RecordDuration(d time.Duration) {
	m.mu.Lock()
	defer m.mu.Unlock()
	
	switch {
	case d < 10*time.Millisecond:
		m.DurationBuckets["<10ms"]++
	case d < 50*time.Millisecond:
		m.DurationBuckets["<50ms"]++
	case d < 100*time.Millisecond:
		m.DurationBuckets["<100ms"]++
	case d < 500*time.Millisecond:
		m.DurationBuckets["<500ms"]++
	default:
		m.DurationBuckets[">=500ms"]++
	}
}

func (m *PoolMetrics) Print() {
	m.mu.RLock()
	defer m.mu.RUnlock()
	
	elapsed := time.Since(m.startTime).Seconds()
	
	fmt.Printf("\n=== Pool Metrics ===\n")
	fmt.Printf("Uptime:         %.1fs\n", elapsed)
	fmt.Printf("Total Jobs:     %d\n", atomic.LoadInt64(&m.TotalSubmitted))
	fmt.Printf("Completed:      %d\n", atomic.LoadInt64(&m.TotalCompleted))
	fmt.Printf("Failed:         %d\n", atomic.LoadInt64(&m.TotalFailed))
	fmt.Printf("Dropped:        %d\n", atomic.LoadInt64(&m.TotalDropped))
	
	total := atomic.LoadInt64(&m.TotalCompleted) + atomic.LoadInt64(&m.TotalFailed)
	if elapsed > 0 {
		fmt.Printf("Throughput:     %.1f jobs/s\n", float64(total)/elapsed)
	}
	
	fmt.Printf("\nDuration Distribution:\n")
	for bucket, count := range m.DurationBuckets {
		pct := float64(0)
		if total > 0 {
			pct = float64(count) / float64(total) * 100
		}
		fmt.Printf("  %s: %d (%.1f%%)\n", bucket, count, pct)
	}
}

type InstrumentedPool struct {
	metrics    *PoolMetrics
	jobs       chan func() error
	numWorkers int
	wg         sync.WaitGroup
}

func NewInstrumentedPool(numWorkers, queueSize int) *InstrumentedPool {
	p := &InstrumentedPool{
		metrics:    NewPoolMetrics(),
		jobs:       make(chan func() error, queueSize),
		numWorkers: numWorkers,
	}
	
	atomic.StoreInt64(&p.metrics.ActiveWorkers, int64(numWorkers))
	
	for i := 0; i < numWorkers; i++ {
		p.wg.Add(1)
		go func() {
			defer p.wg.Done()
			for job := range p.jobs {
				atomic.AddInt64(&p.metrics.QueueDepth, -1)
				
				start := time.Now()
				err := job()
				duration := time.Since(start)
				
				p.metrics.RecordDuration(duration)
				
				if err != nil {
					atomic.AddInt64(&p.metrics.TotalFailed, 1)
				} else {
					atomic.AddInt64(&p.metrics.TotalCompleted, 1)
				}
			}
		}()
	}
	
	return p
}

func (p *InstrumentedPool) Submit(job func() error) bool {
	select {
	case p.jobs <- job:
		atomic.AddInt64(&p.metrics.TotalSubmitted, 1)
		atomic.AddInt64(&p.metrics.QueueDepth, 1)
		return true
	default:
		atomic.AddInt64(&p.metrics.TotalDropped, 1)
		return false
	}
}

func (p *InstrumentedPool) Stop() {
	close(p.jobs)
	p.wg.Wait()
}

func (p *InstrumentedPool) Metrics() *PoolMetrics {
	return p.metrics
}

func metricsDemo() {
	pool := NewInstrumentedPool(5, 100)
	
	// Periodic metrics display
	go func() {
		ticker := time.NewTicker(500 * time.Millisecond)
		for range ticker.C {
			pool.Metrics().Print()
		}
	}()
	
	// Submit jobs
	for i := 0; i < 100; i++ {
		pool.Submit(func() error {
			time.Sleep(time.Duration(10+i%50) * time.Millisecond)
			return nil
		})
		time.Sleep(5 * time.Millisecond)
	}
	
	pool.Stop()
	pool.Metrics().Print()
}

func main() {
	metricsDemo()
}
```

---

## Workshop: Image Processing Pool

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"image"
	"image/color"
	"image/png"
	"math"
	"sync"
	"sync/atomic"
	"time"
)

// Workshop: Image Processing Worker Pool
// Simulates processing a batch of images with multiple filters

// ImageData represents an image to process
type ImageData struct {
	ID     int
	Name   string
	Width  int
	Height int
	Pixels [][]color.RGBA
}

// ProcessedImage is the result of processing
type ProcessedImage struct {
	ID       int
	Name     string
	Original *ImageData
	Result   *ImageData
	Duration time.Duration
	Error    error
}

// Create a simple test image
func createTestImage(id, width, height int) *ImageData {
	pixels := make([][]color.RGBA, height)
	for y := 0; y < height; y++ {
		pixels[y] = make([]color.RGBA, width)
		for x := 0; x < width; x++ {
			// Create gradient pattern
			r := uint8((x * 255) / width)
			g := uint8((y * 255) / height)
			b := uint8(128)
			pixels[y][x] = color.RGBA{R: r, G: g, B: b, A: 255}
		}
	}
	return &ImageData{
		ID:     id,
		Name:   fmt.Sprintf("image_%d.png", id),
		Width:  width,
		Height: height,
		Pixels: pixels,
	}
}

// Filter: Grayscale
func applyGrayscale(img *ImageData) *ImageData {
	result := &ImageData{
		ID:     img.ID,
		Name:   "gray_" + img.Name,
		Width:  img.Width,
		Height: img.Height,
		Pixels: make([][]color.RGBA, img.Height),
	}
	
	for y := 0; y < img.Height; y++ {
		result.Pixels[y] = make([]color.RGBA, img.Width)
		for x := 0; x < img.Width; x++ {
			p := img.Pixels[y][x]
			gray := uint8(0.299*float64(p.R) + 0.587*float64(p.G) + 0.114*float64(p.B))
			result.Pixels[y][x] = color.RGBA{R: gray, G: gray, B: gray, A: p.A}
		}
	}
	return result
}

// Filter: Blur (simple box blur)
func applyBlur(img *ImageData, radius int) *ImageData {
	result := &ImageData{
		ID:     img.ID,
		Name:   "blur_" + img.Name,
		Width:  img.Width,
		Height: img.Height,
		Pixels: make([][]color.RGBA, img.Height),
	}
	
	for y := 0; y < img.Height; y++ {
		result.Pixels[y] = make([]color.RGBA, img.Width)
		for x := 0; x < img.Width; x++ {
			var rSum, gSum, bSum, count float64
			
			for dy := -radius; dy <= radius; dy++ {
				for dx := -radius; dx <= radius; dx++ {
					ny, nx := y+dy, x+dx
					if ny >= 0 && ny < img.Height && nx >= 0 && nx < img.Width {
						p := img.Pixels[ny][nx]
						rSum += float64(p.R)
						gSum += float64(p.G)
						bSum += float64(p.B)
						count++
					}
				}
			}
			
			result.Pixels[y][x] = color.RGBA{
				R: uint8(rSum / count),
				G: uint8(gSum / count),
				B: uint8(bSum / count),
				A: img.Pixels[y][x].A,
			}
		}
	}
	return result
}

// Filter: Edge detection (Sobel)
func applyEdgeDetection(img *ImageData) *ImageData {
	// Convert to grayscale first
	gray := applyGrayscale(img)
	
	result := &ImageData{
		ID:     img.ID,
		Name:   "edge_" + img.Name,
		Width:  img.Width,
		Height: img.Height,
		Pixels: make([][]color.RGBA, img.Height),
	}
	
	sobelX := [3][3]float64{
		{-1, 0, 1},
		{-2, 0, 2},
		{-1, 0, 1},
	}
	sobelY := [3][3]float64{
		{-1, -2, -1},
		{0, 0, 0},
		{1, 2, 1},
	}
	
	for y := 0; y < img.Height; y++ {
		result.Pixels[y] = make([]color.RGBA, img.Width)
		for x := 0; x < img.Width; x++ {
			var gx, gy float64
			
			for dy := -1; dy <= 1; dy++ {
				for dx := -1; dx <= 1; dx++ {
					ny, nx := y+dy, x+dx
					if ny >= 0 && ny < img.Height && nx >= 0 && nx < img.Width {
						intensity := float64(gray.Pixels[ny][nx].R)
						gx += intensity * sobelX[dy+1][dx+1]
						gy += intensity * sobelY[dy+1][dx+1]
					}
				}
			}
			
			magnitude := math.Sqrt(gx*gx + gy*gy)
			if magnitude > 255 {
				magnitude = 255
			}
			edge := uint8(magnitude)
			result.Pixels[y][x] = color.RGBA{R: edge, G: edge, B: edge, A: 255}
		}
	}
	return result
}

// Convert ImageData to PNG bytes (for output/saving)
func toPNGBytes(img *ImageData) ([]byte, error) {
	rgba := image.NewRGBA(image.Rect(0, 0, img.Width, img.Height))
	for y := 0; y < img.Height; y++ {
		for x := 0; x < img.Width; x++ {
			rgba.Set(x, y, img.Pixels[y][x])
		}
	}
	
	var buf bytes.Buffer
	if err := png.Encode(&buf, rgba); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}

// ImageProcessor handles image processing pipeline
type ImageProcessor struct {
	ctx     context.Context
	cancel  context.CancelFunc
	
	inputQueue  chan *ImageData
	outputQueue chan ProcessedImage
	
	filters []func(*ImageData) *ImageData
	
	numWorkers    int
	wg            sync.WaitGroup
	
	// Metrics
	processed int64
	errors    int64
}

func NewImageProcessor(ctx context.Context, numWorkers int) *ImageProcessor {
	procCtx, cancel := context.WithCancel(ctx)
	
	p := &ImageProcessor{
		ctx:         procCtx,
		cancel:      cancel,
		inputQueue:  make(chan *ImageData, numWorkers*2),
		outputQueue: make(chan ProcessedImage, numWorkers*2),
		numWorkers:  numWorkers,
		filters: []func(*ImageData) *ImageData{
			func(img *ImageData) *ImageData { return applyBlur(img, 2) },
			applyGrayscale,
			applyEdgeDetection,
		},
	}
	
	return p
}

func (p *ImageProcessor) Start() {
	for i := 0; i < p.numWorkers; i++ {
		p.wg.Add(1)
		go func(workerID int) {
			defer p.wg.Done()
			
			for {
				select {
				case <-p.ctx.Done():
					return
				case img, ok := <-p.inputQueue:
					if !ok {
						return
					}
					
					start := time.Now()
					
					// Apply all filters in sequence
					result := img
					var err error
					for _, filter := range p.filters {
						result = filter(result)
					}
					
					// Save to PNG
					pngBytes, saveErr := toPNGBytes(result)
					if saveErr != nil {
						err = saveErr
						atomic.AddInt64(&p.errors, 1)
					} else {
						_ = pngBytes // In real code, save to disk
						atomic.AddInt64(&p.processed, 1)
					}
					
					p.outputQueue <- ProcessedImage{
						ID:       img.ID,
						Name:     img.Name,
						Original: img,
						Result:   result,
						Duration: time.Since(start),
						Error:    err,
					}
				}
			}
		}(i + 1)
	}
	
	go func() {
		p.wg.Wait()
		close(p.outputQueue)
	}()
}

func (p *ImageProcessor) Submit(img *ImageData) {
	select {
	case p.inputQueue <- img:
	case <-p.ctx.Done():
	}
}

func (p *ImageProcessor) Stop() {
	close(p.inputQueue)
}

func (p *ImageProcessor) Results() <-chan ProcessedImage {
	return p.outputQueue
}

func imageProcessingWorkshop() {
	ctx := context.Background()
	processor := NewImageProcessor(ctx, 4)
	processor.Start()
	
	fmt.Println("=== Image Processing Workshop ===")
	fmt.Printf("Workers: %d, Filters: %d\n", processor.numWorkers, len(processor.filters))
	
	// Create test images of various sizes
	imageSizes := []struct{ w, h int }{
		{64, 64}, {128, 128}, {256, 256},
		{64, 64}, {128, 128}, {256, 256},
		{64, 64}, {128, 128}, {256, 256},
		{512, 512},
	}
	
	startTime := time.Now()
	
	// Submit images
	for i, size := range imageSizes {
		img := createTestImage(i+1, size.w, size.h)
		processor.Submit(img)
	}
	processor.Stop()
	
	// Collect results
	var totalDuration time.Duration
	var processedCount int
	var errorCount int
	
	for result := range processor.Results() {
		processedCount++
		if result.Error != nil {
			fmt.Printf("Error processing %s: %v\n", result.Name, result.Error)
			errorCount++
		} else {
			totalDuration += result.Duration
			fmt.Printf("Processed: %-20s (%dx%d) in %v\n",
				result.Name,
				result.Original.Width,
				result.Original.Height,
				result.Duration.Round(time.Millisecond))
		}
	}
	
	wallTime := time.Since(startTime)
	
	fmt.Printf("\n=== Summary ===\n")
	fmt.Printf("Total images:    %d\n", processedCount)
	fmt.Printf("Successful:      %d\n", processedCount-errorCount)
	fmt.Printf("Errors:          %d\n", errorCount)
	fmt.Printf("Total CPU time:  %v\n", totalDuration.Round(time.Millisecond))
	fmt.Printf("Wall time:       %v\n", wallTime.Round(time.Millisecond))
	fmt.Printf("Speedup:         %.1fx\n", float64(totalDuration)/float64(wallTime))
}

func main() {
	imageProcessingWorkshop()
}
```

---

## สรุป

ใน Part 37 เราได้เรียนรู้:

1. **Worker Pool พื้นฐาน**: การใช้ goroutines จำนวนคงที่ประมวลผลงาน
2. **Priority Queue**: การจัดลำดับความสำคัญของงาน
3. **Result Aggregation**: การรวบรวมและวิเคราะห์ผลลัพธ์
4. **Error Handling**: การจัดการ errors รวมถึง retry logic
5. **Dynamic Pool Sizing**: การปรับขนาด pool ตาม load
6. **Graceful Shutdown**: การหยุดทำงานอย่างปลอดภัย
7. **Metrics & Monitoring**: การติดตามประสิทธิภาพ
8. **Workshop**: Image Processing Pool ที่ใช้งานจริง

---

## Resources

- [Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines)
- [Worker Pool Pattern in Go](https://gobyexample.com/worker-pools)
- [Effective Go: Goroutines](https://go.dev/doc/effective_go#goroutines)
- [Go by Example: WaitGroups](https://gobyexample.com/waitgroups)
