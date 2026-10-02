# Part 55: Circuit Breaker Pattern

## เป้าหมายการเรียนรู้
- เข้าใจ Circuit Breaker pattern
- States: Closed, Open, Half-Open
- ใช้ gobreaker library
- Fallback strategies
- Bulkhead pattern
- Timeout patterns

---

## 1. Circuit Breaker Concept

Circuit Breaker ป้องกัน cascading failures ใน distributed systems เมื่อ service หนึ่งล้มเหลว circuit จะ "เปิด" และหยุดส่ง request ไปยัง service นั้น

### States
- **Closed**: ปกติ - ส่ง requests ผ่านได้
- **Open**: เปิด - หยุดส่ง requests ทันที (fail fast)
- **Half-Open**: ทดสอบ - ส่ง request จำนวนน้อยเพื่อตรวจว่า service กลับมาแล้วหรือยัง

```
[Closed] --too many failures--> [Open] --timeout--> [Half-Open]
   ^                                                      |
   |------ success ----------------------------------------|
                                      |------ still failing --> [Open]
```

---

## 2. Circuit Breaker Implementation

```go
// circuitbreaker/breaker.go
package circuitbreaker

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

// State ของ circuit breaker
type State int

const (
	StateClosed   State = iota // ปกติ ส่ง requests ผ่านได้
	StateOpen                  // เปิด บล็อก requests
	StateHalfOpen              // ทดสอบ
)

func (s State) String() string {
	switch s {
	case StateClosed:
		return "CLOSED"
	case StateOpen:
		return "OPEN"
	case StateHalfOpen:
		return "HALF-OPEN"
	default:
		return "UNKNOWN"
	}
}

var (
	ErrCircuitOpen    = errors.New("circuit breaker is open")
	ErrTooManyRequests = errors.New("too many requests in half-open state")
)

// Counts เก็บ statistics ของ requests
type Counts struct {
	Requests             uint32
	TotalSuccesses       uint32
	TotalFailures        uint32
	ConsecutiveSuccesses uint32
	ConsecutiveFailures  uint32
}

func (c *Counts) onRequest() {
	c.Requests++
}

func (c *Counts) onSuccess() {
	c.TotalSuccesses++
	c.ConsecutiveSuccesses++
	c.ConsecutiveFailures = 0
}

func (c *Counts) onFailure() {
	c.TotalFailures++
	c.ConsecutiveFailures++
	c.ConsecutiveSuccesses = 0
}

func (c *Counts) clear() {
	*c = Counts{}
}

// Settings configuration สำหรับ circuit breaker
type Settings struct {
	Name          string
	MaxRequests   uint32        // max requests ใน half-open state
	Interval      time.Duration // ช่วงเวลาตรวจ (0 = ไม่ reset)
	Timeout       time.Duration // เวลา open -> half-open
	ReadyToTrip   func(counts Counts) bool
	OnStateChange func(name string, from State, to State)
	IsSuccessful  func(err error) bool
}

// CircuitBreaker implementation
type CircuitBreaker struct {
	name         string
	maxRequests  uint32
	interval     time.Duration
	timeout      time.Duration
	readyToTrip  func(counts Counts) bool
	isSuccessful func(err error) bool
	onStateChange func(name string, from State, to State)

	mu         sync.Mutex
	state      State
	generation uint64
	counts     Counts
	expiry     time.Time
}

// New สร้าง CircuitBreaker ใหม่
func New(st Settings) *CircuitBreaker {
	cb := &CircuitBreaker{
		name:        st.Name,
		maxRequests: st.MaxRequests,
		interval:    st.Interval,
		timeout:     st.Timeout,
		onStateChange: st.OnStateChange,
	}

	if st.MaxRequests == 0 {
		cb.maxRequests = 1
	}
	if st.Timeout == 0 {
		cb.timeout = 60 * time.Second
	}
	if st.ReadyToTrip == nil {
		cb.readyToTrip = func(counts Counts) bool {
			return counts.ConsecutiveFailures > 5
		}
	} else {
		cb.readyToTrip = st.ReadyToTrip
	}
	if st.IsSuccessful == nil {
		cb.isSuccessful = func(err error) bool {
			return err == nil
		}
	} else {
		cb.isSuccessful = st.IsSuccessful
	}

	return cb
}

// Execute รัน function ผ่าน circuit breaker
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
	generation, err := cb.beforeRequest()
	if err != nil {
		return nil, err
	}

	defer func() {
		if r := recover(); r != nil {
			cb.afterRequest(generation, false)
			panic(r)
		}
	}()

	result, err := req()
	cb.afterRequest(generation, cb.isSuccessful(err))
	return result, err
}

func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	state, generation := cb.currentState(time.Now())

	if state == StateOpen {
		return generation, ErrCircuitOpen
	} else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests {
		return generation, ErrTooManyRequests
	}

	cb.counts.onRequest()
	return generation, nil
}

func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	now := time.Now()
	state, generation := cb.currentState(now)
	if generation != before {
		return
	}

	if success {
		cb.onSuccess(state, now)
	} else {
		cb.onFailure(state, now)
	}
}

func (cb *CircuitBreaker) onSuccess(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onSuccess()
	case StateHalfOpen:
		cb.counts.onSuccess()
		if cb.counts.ConsecutiveSuccesses >= cb.maxRequests {
			cb.setState(StateClosed, now)
		}
	}
}

func (cb *CircuitBreaker) onFailure(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onFailure()
		if cb.readyToTrip(cb.counts) {
			cb.setState(StateOpen, now)
		}
	case StateHalfOpen:
		cb.setState(StateOpen, now)
	}
}

func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
	switch cb.state {
	case StateClosed:
		if !cb.expiry.IsZero() && cb.expiry.Before(now) {
			cb.toNewGeneration(now)
		}
	case StateOpen:
		if cb.expiry.Before(now) {
			cb.setState(StateHalfOpen, now)
		}
	}
	return cb.state, cb.generation
}

func (cb *CircuitBreaker) setState(state State, now time.Time) {
	if cb.state == state {
		return
	}

	prev := cb.state
	cb.state = state
	cb.toNewGeneration(now)

	if cb.onStateChange != nil {
		cb.onStateChange(cb.name, prev, state)
	}
}

func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
	cb.generation++
	cb.counts.clear()

	var zero time.Time
	switch cb.state {
	case StateClosed:
		if cb.interval == 0 {
			cb.expiry = zero
		} else {
			cb.expiry = now.Add(cb.interval)
		}
	case StateOpen:
		cb.expiry = now.Add(cb.timeout)
	default: // StateHalfOpen
		cb.expiry = zero
	}
}

// State returns current state
func (cb *CircuitBreaker) State() State {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	state, _ := cb.currentState(time.Now())
	return state
}

// Counts returns current counts
func (cb *CircuitBreaker) Counts() Counts {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	return cb.counts
}
```

---

## 3. Circuit Breaker Usage Examples

```go
// examples/usage.go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"
)

// สร้าง circuit breaker สำหรับ HTTP client
type HTTPClient struct {
	client  *http.Client
	breaker *CircuitBreaker
}

func NewHTTPClient(name string) *HTTPClient {
	cb := New(Settings{
		Name:        name,
		MaxRequests: 3,
		Interval:    10 * time.Second,
		Timeout:     30 * time.Second,
		ReadyToTrip: func(counts Counts) bool {
			failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
			return counts.Requests >= 10 && failureRatio >= 0.6
		},
		OnStateChange: func(name string, from, to State) {
			log.Printf("Circuit breaker %s: %s -> %s", name, from, to)
		},
	})

	return &HTTPClient{
		client:  &http.Client{Timeout: 10 * time.Second},
		breaker: cb,
	}
}

func (c *HTTPClient) Get(ctx context.Context, url string) ([]byte, error) {
	result, err := c.breaker.Execute(func() (interface{}, error) {
		req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
		if err != nil {
			return nil, err
		}

		resp, err := c.client.Do(req)
		if err != nil {
			return nil, err
		}
		defer resp.Body.Close()

		if resp.StatusCode >= 500 {
			return nil, fmt.Errorf("server error: %d", resp.StatusCode)
		}
		return []byte("response body"), nil
	})

	if err != nil {
		if errors.Is(err, ErrCircuitOpen) {
			return nil, fmt.Errorf("circuit is open, service unavailable: %w", err)
		}
		return nil, err
	}

	return result.([]byte), nil
}

// ตัวอย่างการ simulate failures
func simulateService(failRate float64) func() (interface{}, error) {
	return func() (interface{}, error) {
		if rand.Float64() < failRate {
			return nil, errors.New("service unavailable")
		}
		return "success", nil
	}
}

func demoCircuitBreaker() {
	cb := New(Settings{
		Name:        "demo-service",
		MaxRequests: 2,
		Timeout:     5 * time.Second,
		ReadyToTrip: func(counts Counts) bool {
			return counts.ConsecutiveFailures >= 3
		},
		OnStateChange: func(name string, from, to State) {
			fmt.Printf("[%s] State: %s -> %s\n", name, from, to)
		},
	})

	fmt.Println("=== Simulating service failures ===")
	
	// Phase 1: Normal operation (20% failure)
	fmt.Println("\nPhase 1: Normal (20% failure rate)")
	for i := 0; i < 5; i++ {
		_, err := cb.Execute(simulateService(0.2))
		fmt.Printf("Request %d: %v (State: %s)\n", i+1, err, cb.State())
		time.Sleep(100 * time.Millisecond)
	}

	// Phase 2: High failure rate
	fmt.Println("\nPhase 2: High failure (100% failure rate)")
	for i := 0; i < 5; i++ {
		_, err := cb.Execute(simulateService(1.0))
		fmt.Printf("Request %d: %v (State: %s)\n", i+1, err, cb.State())
		time.Sleep(100 * time.Millisecond)
	}

	// Phase 3: Circuit is open, requests fail fast
	fmt.Println("\nPhase 3: Circuit open (fail fast)")
	for i := 0; i < 3; i++ {
		_, err := cb.Execute(simulateService(0.0)) // ไม่ได้ถูก execute
		fmt.Printf("Request %d: %v (State: %s)\n", i+1, err, cb.State())
	}

	// Wait for timeout
	fmt.Printf("\nWaiting 5 seconds for circuit to go half-open...\n")
	time.Sleep(6 * time.Second)

	// Phase 4: Service recovered
	fmt.Println("\nPhase 4: Service recovered")
	for i := 0; i < 5; i++ {
		_, err := cb.Execute(simulateService(0.0)) // 0% failure
		fmt.Printf("Request %d: %v (State: %s)\n", i+1, err, cb.State())
		time.Sleep(100 * time.Millisecond)
	}
}
```

---

## 4. Fallback Strategies

```go
// fallback/strategies.go
package fallback

import (
	"context"
	"encoding/json"
	"errors"
	"log"
	"sync"
	"time"
)

// FallbackFunc type
type FallbackFunc func(ctx context.Context, err error) (interface{}, error)

// WithFallback wraps circuit breaker with fallback
type WithFallback struct {
	breaker  *CircuitBreaker
	fallback FallbackFunc
}

func New(cb *CircuitBreaker, fallback FallbackFunc) *WithFallback {
	return &WithFallback{breaker: cb, fallback: fallback}
}

func (w *WithFallback) Execute(ctx context.Context, req func() (interface{}, error)) (interface{}, error) {
	result, err := w.breaker.Execute(req)
	if err != nil {
		log.Printf("Primary failed: %v, using fallback", err)
		return w.fallback(ctx, err)
	}
	return result, nil
}

// Cache Fallback - ใช้ cached response เมื่อ service ล้ม
type CacheFallback struct {
	mu    sync.RWMutex
	cache map[string]cachedItem
	ttl   time.Duration
}

type cachedItem struct {
	data      []byte
	expiresAt time.Time
}

func NewCacheFallback(ttl time.Duration) *CacheFallback {
	return &CacheFallback{
		cache: make(map[string]cachedItem),
		ttl:   ttl,
	}
}

func (c *CacheFallback) Set(key string, data []byte) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.cache[key] = cachedItem{
		data:      data,
		expiresAt: time.Now().Add(c.ttl),
	}
}

func (c *CacheFallback) Get(key string) ([]byte, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	item, ok := c.cache[key]
	if !ok || time.Now().After(item.expiresAt) {
		return nil, false
	}
	return item.data, true
}

func (c *CacheFallback) CreateFallback(key string) FallbackFunc {
	return func(ctx context.Context, err error) (interface{}, error) {
		data, ok := c.Get(key)
		if !ok {
			return nil, errors.New("no cached response available")
		}
		log.Printf("Returning cached response for key: %s", key)
		return data, nil
	}
}

// Default Response Fallback
func DefaultResponseFallback(defaultData interface{}) FallbackFunc {
	return func(ctx context.Context, err error) (interface{}, error) {
		log.Printf("Returning default response due to: %v", err)
		return defaultData, nil
	}
}

// Multi-level Fallback
type MultiLevelFallback struct {
	levels []FallbackFunc
}

func NewMultiLevelFallback(levels ...FallbackFunc) *MultiLevelFallback {
	return &MultiLevelFallback{levels: levels}
}

func (m *MultiLevelFallback) Execute(ctx context.Context, err error) (interface{}, error) {
	var lastErr error
	for i, level := range m.levels {
		result, err := level(ctx, lastErr)
		if err == nil {
			return result, nil
		}
		log.Printf("Fallback level %d failed: %v", i+1, err)
		lastErr = err
	}
	return nil, fmt.Errorf("all fallback levels failed: %w", lastErr)
}

// ตัวอย่างใช้งาน Fallback
func ExampleFallback() {
	// Cache fallback
	cache := NewCacheFallback(5 * time.Minute)
	
	// Pre-populate cache
	defaultProducts := []map[string]interface{}{
		{"id": "1", "name": "Product 1", "price": 99.99},
	}
	data, _ := json.Marshal(defaultProducts)
	cache.Set("products", data)

	cb := New(Settings{
		Name:        "product-service",
		MaxRequests: 1,
		Timeout:     10 * time.Second,
		ReadyToTrip: func(counts Counts) bool {
			return counts.ConsecutiveFailures >= 2
		},
	})

	withFallback := WithFallback{
		breaker: cb,
		fallback: MultiLevelFallback{
			levels: []FallbackFunc{
				cache.CreateFallback("products"),
				DefaultResponseFallback([]interface{}{}),
			},
		}.Execute,
	}

	ctx := context.Background()

	// Simulate requests
	for i := 0; i < 5; i++ {
		result, err := withFallback.Execute(ctx, func() (interface{}, error) {
			return nil, errors.New("service down") // simulate failure
		})
		if err != nil {
			log.Printf("Request %d failed: %v", i+1, err)
		} else {
			log.Printf("Request %d succeeded with: %v", i+1, result)
		}
	}
}
```

---

## 5. Bulkhead Pattern

```go
// bulkhead/bulkhead.go
package bulkhead

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"time"
)

var ErrBulkheadFull = errors.New("bulkhead is at capacity")

// Bulkhead จำกัด concurrent requests ไปยัง service
type Bulkhead struct {
	name      string
	maxConns  int
	queue     chan struct{}
	semaphore chan struct{}
	mu        sync.Mutex
	metrics   BulkheadMetrics
}

type BulkheadMetrics struct {
	Accepted  int64
	Rejected  int64
	Executing int64
	Queued    int64
}

// NewBulkhead สร้าง Bulkhead ใหม่
func NewBulkhead(name string, maxConns int, maxQueue int) *Bulkhead {
	return &Bulkhead{
		name:      name,
		maxConns:  maxConns,
		semaphore: make(chan struct{}, maxConns),
		queue:     make(chan struct{}, maxQueue),
	}
}

// Execute รัน function ผ่าน bulkhead
func (b *Bulkhead) Execute(ctx context.Context, fn func() (interface{}, error)) (interface{}, error) {
	// Try to acquire semaphore immediately
	select {
	case b.semaphore <- struct{}{}:
		defer func() { <-b.semaphore }()
		b.mu.Lock()
		b.metrics.Accepted++
		b.metrics.Executing++
		b.mu.Unlock()
		defer func() {
			b.mu.Lock()
			b.metrics.Executing--
			b.mu.Unlock()
		}()
		return fn()
	default:
	}

	// Try to queue
	select {
	case b.queue <- struct{}{}:
	default:
		b.mu.Lock()
		b.metrics.Rejected++
		b.mu.Unlock()
		return nil, ErrBulkheadFull
	}

	b.mu.Lock()
	b.metrics.Queued++
	b.mu.Unlock()

	// Wait for semaphore or context cancellation
	select {
	case <-ctx.Done():
		<-b.queue
		b.mu.Lock()
		b.metrics.Queued--
		b.mu.Unlock()
		return nil, ctx.Err()

	case b.semaphore <- struct{}{}:
		<-b.queue
		defer func() { <-b.semaphore }()
		b.mu.Lock()
		b.metrics.Queued--
		b.metrics.Accepted++
		b.metrics.Executing++
		b.mu.Unlock()
		defer func() {
			b.mu.Lock()
			b.metrics.Executing--
			b.mu.Unlock()
		}()
		return fn()
	}
}

// Metrics returns current metrics
func (b *Bulkhead) Metrics() BulkheadMetrics {
	b.mu.Lock()
	defer b.mu.Unlock()
	return b.metrics
}

// ThreadPoolBulkhead ใช้ goroutine pool
type ThreadPoolBulkhead struct {
	name    string
	workers chan chan func()
	wg      sync.WaitGroup
}

func NewThreadPoolBulkhead(name string, poolSize int) *ThreadPoolBulkhead {
	tp := &ThreadPoolBulkhead{
		name:    name,
		workers: make(chan chan func(), poolSize),
	}
	tp.start(poolSize)
	return tp
}

func (tp *ThreadPoolBulkhead) start(size int) {
	for i := 0; i < size; i++ {
		tp.wg.Add(1)
		jobCh := make(chan func(), 1)
		tp.workers <- jobCh

		go func(jobs chan func()) {
			defer tp.wg.Done()
			for job := range jobs {
				job()
				tp.workers <- jobs
			}
		}(jobCh)
	}
}

func (tp *ThreadPoolBulkhead) Execute(ctx context.Context, fn func() (interface{}, error)) (interface{}, error) {
	resultCh := make(chan struct {
		val interface{}
		err error
	}, 1)

	job := func() {
		val, err := fn()
		resultCh <- struct {
			val interface{}
			err error
		}{val, err}
	}

	select {
	case <-ctx.Done():
		return nil, ctx.Err()
	case jobCh := <-tp.workers:
		jobCh <- job
	default:
		return nil, ErrBulkheadFull
	}

	select {
	case <-ctx.Done():
		return nil, ctx.Err()
	case result := <-resultCh:
		return result.val, result.err
	}
}
```

---

## 6. Timeout Patterns

```go
// timeout/patterns.go
package timeout

import (
	"context"
	"errors"
	"fmt"
	"time"
)

var ErrTimeout = errors.New("operation timed out")

// SimpleTimeout wraps function call with timeout
func WithTimeout(ctx context.Context, timeout time.Duration, fn func(ctx context.Context) (interface{}, error)) (interface{}, error) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	type result struct {
		val interface{}
		err error
	}

	ch := make(chan result, 1)
	go func() {
		val, err := fn(ctx)
		ch <- result{val, err}
	}()

	select {
	case <-ctx.Done():
		return nil, fmt.Errorf("%w: %v", ErrTimeout, ctx.Err())
	case r := <-ch:
		return r.val, r.err
	}
}

// AdaptiveTimeout ปรับ timeout ตาม response time history
type AdaptiveTimeout struct {
	measurements []time.Duration
	maxSamples   int
	multiplier   float64
	minTimeout   time.Duration
	maxTimeout   time.Duration
	mu           interface{ Lock(); Unlock(); RLock(); RUnlock() }
}

func NewAdaptiveTimeout(minTimeout, maxTimeout time.Duration, multiplier float64) *AdaptiveTimeout {
	return &AdaptiveTimeout{
		maxSamples: 100,
		multiplier: multiplier,
		minTimeout: minTimeout,
		maxTimeout: maxTimeout,
	}
}

func (at *AdaptiveTimeout) Record(duration time.Duration) {
	if len(at.measurements) >= at.maxSamples {
		at.measurements = at.measurements[1:]
	}
	at.measurements = append(at.measurements, duration)
}

func (at *AdaptiveTimeout) CurrentTimeout() time.Duration {
	if len(at.measurements) == 0 {
		return at.maxTimeout
	}

	// Calculate 95th percentile
	var total time.Duration
	for _, m := range at.measurements {
		total += m
	}
	avg := total / time.Duration(len(at.measurements))

	timeout := time.Duration(float64(avg) * at.multiplier)
	
	if timeout < at.minTimeout {
		return at.minTimeout
	}
	if timeout > at.maxTimeout {
		return at.maxTimeout
	}
	return timeout
}

// Retry with timeout
type RetryConfig struct {
	MaxAttempts  int
	InitialDelay time.Duration
	MaxDelay     time.Duration
	Multiplier   float64
	Timeout      time.Duration
}

func WithRetry(ctx context.Context, cfg RetryConfig, fn func(ctx context.Context) (interface{}, error)) (interface{}, error) {
	var lastErr error
	delay := cfg.InitialDelay

	for attempt := 1; attempt <= cfg.MaxAttempts; attempt++ {
		// Apply per-attempt timeout
		attemptCtx := ctx
		if cfg.Timeout > 0 {
			var cancel context.CancelFunc
			attemptCtx, cancel = context.WithTimeout(ctx, cfg.Timeout)
			defer cancel()
		}

		result, err := fn(attemptCtx)
		if err == nil {
			return result, nil
		}

		lastErr = err
		
		// Don't retry if context is cancelled
		if ctx.Err() != nil {
			return nil, ctx.Err()
		}

		// Don't retry on last attempt
		if attempt == cfg.MaxAttempts {
			break
		}

		// Exponential backoff
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(delay):
		}

		delay = time.Duration(float64(delay) * cfg.Multiplier)
		if delay > cfg.MaxDelay {
			delay = cfg.MaxDelay
		}
	}

	return nil, fmt.Errorf("all %d attempts failed: %w", cfg.MaxAttempts, lastErr)
}
```

---

## 7. Complete Resilience Pattern

```go
// resilience/pattern.go
package resilience

import (
	"context"
	"fmt"
	"log"
	"time"
)

// ResiliencePattern รวม Circuit Breaker + Bulkhead + Timeout + Retry
type ResiliencePattern struct {
	cb       *CircuitBreaker
	bulkhead *Bulkhead
	timeout  time.Duration
	retry    *RetryConfig
	fallback func(error) (interface{}, error)
}

type Config struct {
	Name              string
	CircuitBreaker    CBConfig
	Bulkhead          BulkheadConfig
	Timeout           time.Duration
	Retry             RetryConfig
	Fallback          func(error) (interface{}, error)
}

type CBConfig struct {
	MaxRequests         uint32
	Interval            time.Duration
	Timeout             time.Duration
	ConsecutiveFailures uint32
}

type BulkheadConfig struct {
	MaxConcurrent int
	MaxQueue      int
}

type RetryConfig struct {
	MaxAttempts  int
	InitialDelay time.Duration
	MaxDelay     time.Duration
	Multiplier   float64
}

func NewResiliencePattern(cfg Config) *ResiliencePattern {
	cb := NewCircuitBreaker(cfg.CircuitBreaker)
	bh := NewBulkhead(cfg.Name, cfg.Bulkhead.MaxConcurrent, cfg.Bulkhead.MaxQueue)
	
	return &ResiliencePattern{
		cb:       cb,
		bulkhead: bh,
		timeout:  cfg.Timeout,
		retry:    &cfg.Retry,
		fallback: cfg.Fallback,
	}
}

func NewCircuitBreaker(cfg CBConfig) *CircuitBreaker {
	return New(Settings{
		MaxRequests: cfg.MaxRequests,
		Interval:    cfg.Interval,
		Timeout:     cfg.Timeout,
		ReadyToTrip: func(counts Counts) bool {
			return counts.ConsecutiveFailures >= cfg.ConsecutiveFailures
		},
		OnStateChange: func(name string, from, to State) {
			log.Printf("[%s] Circuit: %s -> %s", name, from, to)
		},
	})
}

func NewBulkhead(name string, maxConcurrent, maxQueue int) *Bulkhead {
	return &Bulkhead{
		name:      name,
		maxConns:  maxConcurrent,
		semaphore: make(chan struct{}, maxConcurrent),
		queue:     make(chan struct{}, maxQueue),
	}
}

func (p *ResiliencePattern) Execute(ctx context.Context, fn func(ctx context.Context) (interface{}, error)) (interface{}, error) {
	// Apply timeout
	if p.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, p.timeout)
		defer cancel()
	}

	var result interface{}
	var err error

	// Apply bulkhead
	_, bulkheadErr := p.bulkhead.Execute(ctx, func() (interface{}, error) {
		// Apply circuit breaker
		cbResult, cbErr := p.cb.Execute(func() (interface{}, error) {
			// Apply retry
			if p.retry != nil {
				result, err = withRetry(ctx, *p.retry, fn)
			} else {
				result, err = fn(ctx)
			}
			return result, err
		})
		return cbResult, cbErr
	})

	if bulkheadErr != nil {
		err = bulkheadErr
	}

	// Apply fallback
	if err != nil && p.fallback != nil {
		return p.fallback(err)
	}

	return result, err
}

func withRetry(ctx context.Context, cfg RetryConfig, fn func(context.Context) (interface{}, error)) (interface{}, error) {
	delay := cfg.InitialDelay
	var lastErr error

	for i := 0; i < cfg.MaxAttempts; i++ {
		if i > 0 {
			select {
			case <-ctx.Done():
				return nil, ctx.Err()
			case <-time.After(delay):
			}
			delay = time.Duration(float64(delay) * cfg.Multiplier)
			if delay > cfg.MaxDelay {
				delay = cfg.MaxDelay
			}
		}

		result, err := fn(ctx)
		if err == nil {
			return result, nil
		}
		lastErr = err
	}
	return nil, fmt.Errorf("retry exhausted: %w", lastErr)
}

// Example usage
func ExampleResilience() {
	pattern := NewResiliencePattern(Config{
		Name: "payment-service",
		CircuitBreaker: CBConfig{
			MaxRequests:         3,
			Interval:            10 * time.Second,
			Timeout:             30 * time.Second,
			ConsecutiveFailures: 5,
		},
		Bulkhead: BulkheadConfig{
			MaxConcurrent: 10,
			MaxQueue:      20,
		},
		Timeout: 5 * time.Second,
		Retry: RetryConfig{
			MaxAttempts:  3,
			InitialDelay: 100 * time.Millisecond,
			MaxDelay:     2 * time.Second,
			Multiplier:   2.0,
		},
		Fallback: func(err error) (interface{}, error) {
			return map[string]string{
				"status":  "pending",
				"message": "Payment will be processed later",
			}, nil
		},
	})

	ctx := context.Background()
	result, err := pattern.Execute(ctx, func(ctx context.Context) (interface{}, error) {
		// Call payment service
		return map[string]string{"status": "success"}, nil
	})

	if err != nil {
		log.Printf("Error: %v", err)
	} else {
		log.Printf("Result: %v", result)
	}
}
```

---

## สรุป

| Pattern | จุดประสงค์ | สถานการณ์ |
|---------|-----------|----------|
| Circuit Breaker | ป้องกัน cascading failures | Service หยุดทำงาน |
| Bulkhead | จำกัด resource usage | Service ช้าหรือ overload |
| Timeout | ป้องกัน request รอนานเกินไป | Service ไม่ตอบ |
| Retry | ลอง request ซ้ำ | Transient failures |
| Fallback | ตอบ default response | เมื่อทุกอย่างล้มเหลว |

---

**ต่อไป**: Part 56 - Event-Driven Architecture
