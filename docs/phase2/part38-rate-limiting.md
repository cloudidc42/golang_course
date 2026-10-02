# Part 38: Rate Limiting ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ Rate Limiting algorithms ต่างๆ
- ใช้ golang.org/x/time/rate ได้
- สร้าง Redis-based Rate Limiting
- สร้าง Per-user Rate Limiting
- ใช้ Rate Limiting ใน Gin middleware
- สร้าง Distributed Rate Limiting

---

## 1. Rate Limiting Algorithms

### 1.1 Token Bucket Algorithm

Token Bucket เป็น algorithm ที่ใช้กันแพร่หลายมากที่สุด ทำงานดังนี้:
- มี bucket ที่เก็บ tokens
- tokens จะถูกเพิ่มในอัตราคงที่
- แต่ละ request ต้องใช้ 1 token
- ถ้า bucket ว่าง request จะถูก reject

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 1: Token Bucket Implementation
type TokenBucket struct {
	mu         sync.Mutex
	capacity   float64 // maximum tokens
	tokens     float64 // current tokens
	refillRate float64 // tokens per second
	lastRefill time.Time
}

func NewTokenBucket(capacity float64, refillRate float64) *TokenBucket {
	return &TokenBucket{
		capacity:   capacity,
		tokens:     capacity, // start full
		refillRate: refillRate,
		lastRefill: time.Now(),
	}
}

func (tb *TokenBucket) refill() {
	now := time.Now()
	elapsed := now.Sub(tb.lastRefill).Seconds()
	tb.tokens += elapsed * tb.refillRate
	if tb.tokens > tb.capacity {
		tb.tokens = tb.capacity
	}
	tb.lastRefill = now
}

func (tb *TokenBucket) Allow() bool {
	return tb.AllowN(1)
}

func (tb *TokenBucket) AllowN(n float64) bool {
	tb.mu.Lock()
	defer tb.mu.Unlock()
	
	tb.refill()
	
	if tb.tokens >= n {
		tb.tokens -= n
		return true
	}
	return false
}

func (tb *TokenBucket) Wait() {
	tb.WaitN(1)
}

func (tb *TokenBucket) WaitN(n float64) {
	for {
		tb.mu.Lock()
		tb.refill()
		
		if tb.tokens >= n {
			tb.tokens -= n
			tb.mu.Unlock()
			return
		}
		
		// Calculate wait time
		needed := n - tb.tokens
		waitTime := time.Duration(needed/tb.refillRate * float64(time.Second))
		tb.mu.Unlock()
		
		time.Sleep(waitTime)
	}
}

func (tb *TokenBucket) Tokens() float64 {
	tb.mu.Lock()
	defer tb.mu.Unlock()
	tb.refill()
	return tb.tokens
}

func tokenBucketDemo() {
	// Allow 10 requests per second, burst of 5
	tb := NewTokenBucket(5, 10)
	
	fmt.Println("=== Token Bucket Demo ===")
	
	// Simulate burst traffic
	for i := 1; i <= 10; i++ {
		allowed := tb.Allow()
		fmt.Printf("Request %2d: allowed=%v, tokens=%.1f\n", i, allowed, tb.Tokens())
		time.Sleep(50 * time.Millisecond)
	}
}

func main() {
	tokenBucketDemo()
}
```

### 1.2 Leaky Bucket Algorithm

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 2: Leaky Bucket Implementation
// ออกจาก bucket ในอัตราคงที่ ไม่ว่า input จะมากแค่ไหน
type LeakyBucket struct {
	mu       sync.Mutex
	capacity int
	queue    []time.Time
	rate     time.Duration // time between requests
}

func NewLeakyBucket(capacity int, rate time.Duration) *LeakyBucket {
	return &LeakyBucket{
		capacity: capacity,
		queue:    make([]time.Time, 0, capacity),
		rate:     rate,
	}
}

func (lb *LeakyBucket) Allow() bool {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	
	now := time.Now()
	
	// Remove expired times from queue
	for len(lb.queue) > 0 {
		if now.Sub(lb.queue[0]) >= lb.rate*time.Duration(len(lb.queue)) {
			lb.queue = lb.queue[1:]
		} else {
			break
		}
	}
	
	if len(lb.queue) >= lb.capacity {
		return false
	}
	
	// Calculate when this request will be processed
	var processTime time.Time
	if len(lb.queue) == 0 {
		processTime = now
	} else {
		processTime = lb.queue[len(lb.queue)-1].Add(lb.rate)
	}
	
	lb.queue = append(lb.queue, processTime)
	return true
}

func leakyBucketDemo() {
	// Process 1 request per 100ms, max queue of 5
	lb := NewLeakyBucket(5, 100*time.Millisecond)
	
	fmt.Println("=== Leaky Bucket Demo ===")
	
	// Send 10 requests quickly
	for i := 1; i <= 10; i++ {
		allowed := lb.Allow()
		fmt.Printf("Request %2d: allowed=%v\n", i, allowed)
	}
	
	fmt.Println("\nWaiting 300ms then trying again...")
	time.Sleep(300 * time.Millisecond)
	
	for i := 11; i <= 15; i++ {
		allowed := lb.Allow()
		fmt.Printf("Request %2d: allowed=%v\n", i, allowed)
	}
}

func main() {
	leakyBucketDemo()
}
```

### 1.3 Fixed Window Counter

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 3: Fixed Window Counter
type FixedWindowCounter struct {
	mu          sync.Mutex
	limit       int
	windowSize  time.Duration
	count       int
	windowStart time.Time
}

func NewFixedWindowCounter(limit int, windowSize time.Duration) *FixedWindowCounter {
	return &FixedWindowCounter{
		limit:       limit,
		windowSize:  windowSize,
		windowStart: time.Now(),
	}
}

func (fc *FixedWindowCounter) Allow() bool {
	fc.mu.Lock()
	defer fc.mu.Unlock()
	
	now := time.Now()
	
	// Reset window if expired
	if now.Sub(fc.windowStart) >= fc.windowSize {
		fc.count = 0
		fc.windowStart = now
	}
	
	if fc.count < fc.limit {
		fc.count++
		return true
	}
	return false
}

func (fc *FixedWindowCounter) RemainingRequests() int {
	fc.mu.Lock()
	defer fc.mu.Unlock()
	return fc.limit - fc.count
}

func (fc *FixedWindowCounter) WindowResetTime() time.Time {
	fc.mu.Lock()
	defer fc.mu.Unlock()
	return fc.windowStart.Add(fc.windowSize)
}

func fixedWindowDemo() {
	// 5 requests per second
	fc := NewFixedWindowCounter(5, time.Second)
	
	fmt.Println("=== Fixed Window Counter Demo ===")
	
	for i := 1; i <= 8; i++ {
		allowed := fc.Allow()
		fmt.Printf("Request %d: allowed=%v, remaining=%d\n",
			i, allowed, fc.RemainingRequests())
		time.Sleep(100 * time.Millisecond)
	}
	
	fmt.Println("\nWindow reset, trying again...")
	time.Sleep(time.Second)
	
	for i := 9; i <= 13; i++ {
		allowed := fc.Allow()
		fmt.Printf("Request %d: allowed=%v\n", i, allowed)
	}
}

func main() {
	fixedWindowDemo()
}
```

### 1.4 Sliding Window Log

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 4: Sliding Window Log
type SlidingWindowLog struct {
	mu         sync.Mutex
	limit      int
	windowSize time.Duration
	timestamps []time.Time
}

func NewSlidingWindowLog(limit int, windowSize time.Duration) *SlidingWindowLog {
	return &SlidingWindowLog{
		limit:      limit,
		windowSize: windowSize,
	}
}

func (sw *SlidingWindowLog) Allow() bool {
	sw.mu.Lock()
	defer sw.mu.Unlock()
	
	now := time.Now()
	cutoff := now.Add(-sw.windowSize)
	
	// Remove old timestamps
	valid := sw.timestamps[:0]
	for _, t := range sw.timestamps {
		if t.After(cutoff) {
			valid = append(valid, t)
		}
	}
	sw.timestamps = valid
	
	if len(sw.timestamps) < sw.limit {
		sw.timestamps = append(sw.timestamps, now)
		return true
	}
	return false
}

func (sw *SlidingWindowLog) Count() int {
	sw.mu.Lock()
	defer sw.mu.Unlock()
	
	now := time.Now()
	cutoff := now.Add(-sw.windowSize)
	
	count := 0
	for _, t := range sw.timestamps {
		if t.After(cutoff) {
			count++
		}
	}
	return count
}

// ตัวอย่าง 5: Sliding Window Counter (more efficient)
type SlidingWindowCounter struct {
	mu         sync.Mutex
	limit      int
	windowSize time.Duration
	bucketSize time.Duration
	buckets    map[int64]int
}

func NewSlidingWindowCounter(limit int, windowSize, bucketSize time.Duration) *SlidingWindowCounter {
	return &SlidingWindowCounter{
		limit:      limit,
		windowSize: windowSize,
		bucketSize: bucketSize,
		buckets:    make(map[int64]int),
	}
}

func (sc *SlidingWindowCounter) getBucketKey(t time.Time) int64 {
	return t.UnixNano() / int64(sc.bucketSize)
}

func (sc *SlidingWindowCounter) Allow() bool {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	
	now := time.Now()
	currentBucket := sc.getBucketKey(now)
	cutoffBucket := sc.getBucketKey(now.Add(-sc.windowSize))
	
	// Clean old buckets
	for key := range sc.buckets {
		if key <= cutoffBucket {
			delete(sc.buckets, key)
		}
	}
	
	// Count total requests in window
	total := 0
	for _, count := range sc.buckets {
		total += count
	}
	
	if total < sc.limit {
		sc.buckets[currentBucket]++
		return true
	}
	return false
}

func slidingWindowDemo() {
	sw := NewSlidingWindowLog(5, time.Second)
	
	fmt.Println("=== Sliding Window Log Demo ===")
	
	// First burst
	for i := 1; i <= 7; i++ {
		allowed := sw.Allow()
		fmt.Printf("Request %d: allowed=%v, count=%d\n", i, allowed, sw.Count())
		time.Sleep(100 * time.Millisecond)
	}
	
	// Wait for some to expire
	time.Sleep(600 * time.Millisecond)
	fmt.Printf("\nAfter 600ms wait, count=%d\n", sw.Count())
	
	for i := 8; i <= 12; i++ {
		allowed := sw.Allow()
		fmt.Printf("Request %d: allowed=%v\n", i, allowed)
	}
}

func main() {
	slidingWindowDemo()
}
```

---

## 2. golang.org/x/time/rate

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"golang.org/x/time/rate"
)

// ตัวอย่าง 6: Using golang.org/x/time/rate
func standardRateLimiter() {
	// Create limiter: 5 events per second, burst of 10
	limiter := rate.NewLimiter(rate.Limit(5), 10)
	
	fmt.Println("=== Standard Rate Limiter ===")
	
	// Allow() - non-blocking
	for i := 1; i <= 15; i++ {
		if limiter.Allow() {
			fmt.Printf("Request %2d: allowed (tokens: %.1f)\n", i, limiter.Tokens())
		} else {
			fmt.Printf("Request %2d: rejected\n", i)
		}
		time.Sleep(50 * time.Millisecond)
	}
}

// ตัวอย่าง 7: Wait with context
func waitWithContext() {
	limiter := rate.NewLimiter(rate.Limit(2), 1) // 2 per second, burst 1
	
	fmt.Println("\n=== Wait with Context ===")
	
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	
	for i := 1; i <= 6; i++ {
		start := time.Now()
		err := limiter.Wait(ctx)
		elapsed := time.Since(start)
		
		if err != nil {
			fmt.Printf("Request %d: context cancelled: %v\n", i, err)
			break
		}
		fmt.Printf("Request %2d: allowed after %v\n", i, elapsed.Round(time.Millisecond))
	}
}

// ตัวอย่าง 8: Reserve - check without consuming
func reserveDemo() {
	limiter := rate.NewLimiter(rate.Limit(5), 5)
	
	fmt.Println("\n=== Reserve Demo ===")
	
	for i := 1; i <= 8; i++ {
		r := limiter.Reserve()
		if !r.OK() {
			fmt.Printf("Request %d: would never be allowed\n", i)
			continue
		}
		
		delay := r.Delay()
		if delay == 0 {
			fmt.Printf("Request %2d: immediate\n", i)
		} else {
			fmt.Printf("Request %2d: wait %v\n", i, delay.Round(time.Millisecond))
		}
		time.Sleep(80 * time.Millisecond)
	}
}

// ตัวอย่าง 9: Adaptive rate limiting based on errors
func adaptiveRateLimiter() {
	fmt.Println("\n=== Adaptive Rate Limiter ===")
	
	// Start with 10 rps
	currentRate := rate.Limit(10)
	limiter := rate.NewLimiter(currentRate, int(currentRate))
	
	// Simulate calls with some failures
	failures := 0
	successes := 0
	
	for i := 1; i <= 20; i++ {
		ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
		err := limiter.Wait(ctx)
		cancel()
		
		if err != nil {
			fmt.Printf("Request %d: rate limited\n", i)
			continue
		}
		
		// Simulate 30% failure rate
		if i%3 == 0 {
			failures++
			// Reduce rate on failure
			if float64(failures)/float64(failures+successes) > 0.2 {
				newRate := currentRate * 0.8
				if newRate < 1 {
					newRate = 1
				}
				if newRate != currentRate {
					currentRate = newRate
					limiter.SetLimit(currentRate)
					fmt.Printf("  Rate reduced to %.1f rps\n", float64(currentRate))
				}
			}
		} else {
			successes++
			// Increase rate on success
			if float64(failures)/float64(failures+successes) < 0.1 && successes%5 == 0 {
				newRate := currentRate * 1.1
				if newRate > 20 {
					newRate = 20
				}
				if newRate != currentRate {
					currentRate = newRate
					limiter.SetLimit(currentRate)
					fmt.Printf("  Rate increased to %.1f rps\n", float64(currentRate))
				}
			}
		}
		
		fmt.Printf("Request %2d: ok (rate=%.1f, failures=%d/%d)\n",
			i, float64(currentRate), failures, failures+successes)
	}
}

func main() {
	standardRateLimiter()
	waitWithContext()
	reserveDemo()
	adaptiveRateLimiter()
}
```

---

## 3. Per-User Rate Limiting

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
	
	"golang.org/x/time/rate"
)

// ตัวอย่าง 10: Per-user rate limiter

type UserRateLimiter struct {
	mu       sync.RWMutex
	limiters map[string]*userLimiter
	rate     rate.Limit
	burst    int
	
	// Cleanup
	cleanupInterval time.Duration
	ttl             time.Duration
}

type userLimiter struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

func NewUserRateLimiter(r rate.Limit, burst int) *UserRateLimiter {
	ul := &UserRateLimiter{
		limiters:        make(map[string]*userLimiter),
		rate:            r,
		burst:           burst,
		cleanupInterval: time.Minute,
		ttl:             3 * time.Minute,
	}
	
	go ul.cleanup()
	return ul
}

func (ul *UserRateLimiter) getLimiter(userID string) *rate.Limiter {
	ul.mu.Lock()
	defer ul.mu.Unlock()
	
	if v, exists := ul.limiters[userID]; exists {
		v.lastSeen = time.Now()
		return v.limiter
	}
	
	limiter := rate.NewLimiter(ul.rate, ul.burst)
	ul.limiters[userID] = &userLimiter{
		limiter:  limiter,
		lastSeen: time.Now(),
	}
	return limiter
}

func (ul *UserRateLimiter) Allow(userID string) bool {
	return ul.getLimiter(userID).Allow()
}

func (ul *UserRateLimiter) Wait(ctx context.Context, userID string) error {
	return ul.getLimiter(userID).Wait(ctx)
}

func (ul *UserRateLimiter) cleanup() {
	ticker := time.NewTicker(ul.cleanupInterval)
	defer ticker.Stop()
	
	for range ticker.C {
		ul.mu.Lock()
		for id, v := range ul.limiters {
			if time.Since(v.lastSeen) > ul.ttl {
				delete(ul.limiters, id)
			}
		}
		ul.mu.Unlock()
	}
}

func (ul *UserRateLimiter) Stats() map[string]int {
	ul.mu.RLock()
	defer ul.mu.RUnlock()
	
	return map[string]int{
		"active_users": len(ul.limiters),
	}
}

// ตัวอย่าง 11: Tiered rate limiting (different limits for different user tiers)
type TieredRateLimiter struct {
	limiters map[string]*UserRateLimiter
}

type UserTier string

const (
	TierFree       UserTier = "free"
	TierBasic      UserTier = "basic"
	TierPremium    UserTier = "premium"
	TierEnterprise UserTier = "enterprise"
)

func NewTieredRateLimiter() *TieredRateLimiter {
	return &TieredRateLimiter{
		limiters: map[string]*UserRateLimiter{
			string(TierFree):       NewUserRateLimiter(10, 20),    // 10 rps, burst 20
			string(TierBasic):      NewUserRateLimiter(50, 100),   // 50 rps, burst 100
			string(TierPremium):    NewUserRateLimiter(200, 400),  // 200 rps, burst 400
			string(TierEnterprise): NewUserRateLimiter(1000, 2000), // 1000 rps, burst 2000
		},
	}
}

func (trl *TieredRateLimiter) Allow(userID string, tier UserTier) bool {
	limiter, ok := trl.limiters[string(tier)]
	if !ok {
		return false
	}
	return limiter.Allow(userID)
}

func tieredRateLimiterDemo() {
	trl := NewTieredRateLimiter()
	
	// Simulate different users with different tiers
	users := []struct {
		id   string
		tier UserTier
	}{
		{"user1", TierFree},
		{"user2", TierBasic},
		{"user3", TierPremium},
	}
	
	fmt.Println("=== Tiered Rate Limiter Demo ===")
	
	// Burst 15 requests for each user
	for _, user := range users {
		allowed := 0
		for i := 0; i < 15; i++ {
			if trl.Allow(user.id, user.tier) {
				allowed++
			}
		}
		fmt.Printf("User %-8s (%-10s): %d/15 allowed\n",
			user.id, user.tier, allowed)
	}
}

func main() {
	// Per-user demo
	ul := NewUserRateLimiter(3, 5) // 3 per second, burst 5
	
	users := []string{"alice", "bob", "alice", "charlie", "alice", "bob"}
	
	fmt.Println("=== Per-User Rate Limiter ===")
	for i, userID := range users {
		allowed := ul.Allow(userID)
		fmt.Printf("Request %d by %-10s: allowed=%v\n", i+1, userID, allowed)
	}
	
	fmt.Printf("\nStats: %v\n", ul.Stats())
	
	// Tiered demo
	tieredRateLimiterDemo()
}
```

---

## 4. Rate Limiting in Gin Middleware

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"sync"
	"time"
	
	"github.com/gin-gonic/gin"
	"golang.org/x/time/rate"
)

// ตัวอย่าง 12: Gin Rate Limiting Middleware

type IPRateLimiter struct {
	ips  map[string]*rate.Limiter
	mu   sync.RWMutex
	r    rate.Limit
	b    int
}

func NewIPRateLimiter(r rate.Limit, b int) *IPRateLimiter {
	return &IPRateLimiter{
		ips: make(map[string]*rate.Limiter),
		r:   r,
		b:   b,
	}
}

func (i *IPRateLimiter) AddIP(ip string) *rate.Limiter {
	i.mu.Lock()
	defer i.mu.Unlock()
	
	limiter := rate.NewLimiter(i.r, i.b)
	i.ips[ip] = limiter
	return limiter
}

func (i *IPRateLimiter) GetLimiter(ip string) *rate.Limiter {
	i.mu.Lock()
	limiter, exists := i.ips[ip]
	i.mu.Unlock()
	
	if !exists {
		return i.AddIP(ip)
	}
	return limiter
}

// Middleware function
func RateLimitMiddleware(limiter *IPRateLimiter) gin.HandlerFunc {
	return func(c *gin.Context) {
		ip := c.ClientIP()
		lim := limiter.GetLimiter(ip)
		
		ctx, cancel := context.WithTimeout(c.Request.Context(), 100*time.Millisecond)
		defer cancel()
		
		if err := lim.Wait(ctx); err != nil {
			c.JSON(http.StatusTooManyRequests, gin.H{
				"error":   "rate limit exceeded",
				"message": fmt.Sprintf("too many requests from %s", ip),
			})
			c.Abort()
			return
		}
		
		c.Next()
	}
}

// ตัวอย่าง 13: Advanced middleware with headers
func AdvancedRateLimitMiddleware(limiter *IPRateLimiter) gin.HandlerFunc {
	return func(c *gin.Context) {
		ip := c.ClientIP()
		lim := limiter.GetLimiter(ip)
		
		// Add rate limit headers
		c.Header("X-RateLimit-Limit", fmt.Sprintf("%v", lim.Limit()))
		c.Header("X-RateLimit-Remaining", fmt.Sprintf("%.0f", lim.Tokens()))
		
		if !lim.Allow() {
			// Calculate retry-after
			r := lim.Reserve()
			retryAfter := r.Delay().Seconds()
			r.Cancel() // Cancel the reservation
			
			c.Header("Retry-After", fmt.Sprintf("%.0f", retryAfter))
			c.JSON(http.StatusTooManyRequests, gin.H{
				"error":       "rate_limit_exceeded",
				"retry_after": retryAfter,
			})
			c.Abort()
			return
		}
		
		c.Next()
	}
}

// ตัวอย่าง 14: Route-specific rate limiting
func setupGinServer() {
	r := gin.New()
	r.Use(gin.Logger())
	r.Use(gin.Recovery())
	
	// Global rate limiter: 100 rps per IP
	globalLimiter := NewIPRateLimiter(100, 200)
	r.Use(RateLimitMiddleware(globalLimiter))
	
	// Strict rate limiter for auth: 5 per minute
	authLimiter := NewIPRateLimiter(rate.Every(time.Minute/5), 5)
	
	// Regular API endpoints
	api := r.Group("/api")
	{
		api.GET("/data", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"data": "some data"})
		})
		
		api.GET("/health", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"status": "ok"})
		})
	}
	
	// Auth endpoints with strict limiting
	auth := r.Group("/auth")
	auth.Use(AdvancedRateLimitMiddleware(authLimiter))
	{
		auth.POST("/login", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"token": "fake-token"})
		})
		
		auth.POST("/register", func(c *gin.Context) {
			c.JSON(http.StatusCreated, gin.H{"message": "registered"})
		})
	}
	
	fmt.Println("Server would start on :8080")
	// r.Run(":8080")
}

func main() {
	setupGinServer()
	
	fmt.Println("=== Rate Limiting Gin Middleware Setup Complete ===")
}
```

---

## 5. Redis-Based Distributed Rate Limiting

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

// ตัวอย่าง 15: Redis-based sliding window rate limiter

type RedisRateLimiter struct {
	client     *redis.Client
	windowSize time.Duration
	limit      int
}

func NewRedisRateLimiter(client *redis.Client, windowSize time.Duration, limit int) *RedisRateLimiter {
	return &RedisRateLimiter{
		client:     client,
		windowSize: windowSize,
		limit:      limit,
	}
}

// Lua script for atomic sliding window check
const slidingWindowScript = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local unique = ARGV[4]

-- Remove entries outside window
redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)

-- Count current entries
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add new entry
    redis.call('ZADD', key, now, unique)
    -- Set expiry
    redis.call('PEXPIRE', key, window)
    return {1, count + 1, limit}
else
    return {0, count, limit}
end
`

func (rrl *RedisRateLimiter) Allow(ctx context.Context, identifier string) (bool, error) {
	now := time.Now().UnixMilli()
	windowMs := rrl.windowSize.Milliseconds()
	unique := fmt.Sprintf("%s:%d", identifier, now)
	
	result, err := rrl.client.Eval(ctx, slidingWindowScript,
		[]string{fmt.Sprintf("ratelimit:%s", identifier)},
		now, windowMs, rrl.limit, unique,
	).Int64Slice()
	
	if err != nil {
		return false, fmt.Errorf("redis rate limit check failed: %w", err)
	}
	
	return result[0] == 1, nil
}

// ตัวอย่าง 16: Redis token bucket
const tokenBucketScript = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local lastRefill = tonumber(data[2]) or now

-- Calculate new tokens
local elapsed = (now - lastRefill) / 1000
local newTokens = math.min(capacity, tokens + elapsed * refillRate)

if newTokens >= requested then
    -- Consume tokens
    redis.call('HMSET', key, 'tokens', newTokens - requested, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return {1, newTokens - requested}
else
    -- Update last refill time but don't consume
    redis.call('HMSET', key, 'tokens', newTokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return {0, newTokens}
end
`

type RedisTokenBucket struct {
	client     *redis.Client
	capacity   int
	refillRate int // tokens per second
}

func NewRedisTokenBucket(client *redis.Client, capacity, refillRate int) *RedisTokenBucket {
	return &RedisTokenBucket{
		client:     client,
		capacity:   capacity,
		refillRate: refillRate,
	}
}

func (rtb *RedisTokenBucket) Allow(ctx context.Context, identifier string) (bool, float64, error) {
	now := time.Now().UnixMilli()
	
	result, err := rtb.client.Eval(ctx, tokenBucketScript,
		[]string{fmt.Sprintf("tokenbucket:%s", identifier)},
		rtb.capacity, rtb.refillRate, now, 1,
	).Int64Slice()
	
	if err != nil {
		return false, 0, err
	}
	
	return result[0] == 1, float64(result[1]), nil
}

func redisRateLimiterDemo() {
	// This is a demo showing how it would work with Redis
	fmt.Println("=== Redis Rate Limiter Demo ===")
	fmt.Println("(Requires Redis connection - showing code structure)")
	
	// In real usage:
	// client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	// limiter := NewRedisRateLimiter(client, time.Second, 10)
	// allowed, err := limiter.Allow(context.Background(), "user:123")
	
	fmt.Println("RedisRateLimiter: Sliding window using Redis ZADD/ZREMRANGEBYSCORE")
	fmt.Println("RedisTokenBucket: Token bucket using Redis HMSET with Lua script")
	fmt.Println("Both provide atomic operations for distributed systems")
}

func main() {
	redisRateLimiterDemo()
}
```

---

## 6. Distributed Rate Limiting

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 17: Distributed rate limiter with gossip protocol simulation

// In a distributed system, rate limiting requires coordination between nodes.
// Approaches:
// 1. Centralized (Redis/database) - simple but single point of failure
// 2. Distributed consensus - complex but highly available
// 3. Local + periodic sync - eventual consistency

type DistributedCounter struct {
	mu         sync.Mutex
	nodeID     string
	localCount int
	peerCounts map[string]int
	limit      int
	window     time.Duration
	lastSync   time.Time
}

func NewDistributedCounter(nodeID string, limit int, window time.Duration) *DistributedCounter {
	return &DistributedCounter{
		nodeID:     nodeID,
		peerCounts: make(map[string]int),
		limit:      limit,
		window:     window,
		lastSync:   time.Now(),
	}
}

func (dc *DistributedCounter) Allow() bool {
	dc.mu.Lock()
	defer dc.mu.Unlock()
	
	// Total count across all nodes
	total := dc.localCount
	for _, count := range dc.peerCounts {
		total += count
	}
	
	if total < dc.limit {
		dc.localCount++
		return true
	}
	return false
}

// Simulates receiving count from a peer node
func (dc *DistributedCounter) ReceivePeerCount(nodeID string, count int) {
	dc.mu.Lock()
	defer dc.mu.Unlock()
	dc.peerCounts[nodeID] = count
}

// Simulates broadcasting local count to peers
func (dc *DistributedCounter) GetLocalCount() (string, int) {
	dc.mu.Lock()
	defer dc.mu.Unlock()
	return dc.nodeID, dc.localCount
}

func (dc *DistributedCounter) Reset() {
	dc.mu.Lock()
	defer dc.mu.Unlock()
	dc.localCount = 0
	dc.peerCounts = make(map[string]int)
}

// ตัวอย่าง 18: Simulating distributed rate limiting across nodes
func simulateDistributedRateLimit() {
	fmt.Println("=== Distributed Rate Limiting Simulation ===")
	
	limit := 10 // 10 total requests allowed across all nodes
	window := time.Second
	
	// Create 3 nodes
	node1 := NewDistributedCounter("node1", limit, window)
	node2 := NewDistributedCounter("node2", limit, window)
	node3 := NewDistributedCounter("node3", limit, window)
	
	nodes := []*DistributedCounter{node1, node2, node3}
	
	// Simulate sync between nodes
	sync := func() {
		for i, sender := range nodes {
			nodeID, count := sender.GetLocalCount()
			for j, receiver := range nodes {
				if i != j {
					receiver.ReceivePeerCount(nodeID, count)
				}
			}
		}
	}
	
	var wg sync.WaitGroup
	results := make([]bool, 15)
	mu := sync.Mutex{}
	
	// Simulate 15 concurrent requests across 3 nodes
	for i := 0; i < 15; i++ {
		wg.Add(1)
		go func(reqID, nodeIndex int) {
			defer wg.Done()
			
			node := nodes[nodeIndex]
			allowed := node.Allow()
			
			mu.Lock()
			results[reqID] = allowed
			mu.Unlock()
			
			// Sync after each request
			sync()
		}(i, i%3)
	}
	
	wg.Wait()
	
	allowed := 0
	for _, r := range results {
		if r {
			allowed++
		}
	}
	
	fmt.Printf("Total requests: 15, Allowed: %d, Limit: %d\n", allowed, limit)
	fmt.Println("Note: Exact count may vary due to race conditions in distributed systems")
	fmt.Println("In production, use Redis or similar for atomic distributed counting")
	
	_ = window // suppress unused warning
}

// ตัวอย่าง 19: Rate limiter with backpressure
type BackpressureLimiter struct {
	mu          sync.Mutex
	queue       []chan struct{}
	limit       int
	windowSize  time.Duration
	timestamps  []time.Time
}

func NewBackpressureLimiter(limit int, window time.Duration) *BackpressureLimiter {
	return &BackpressureLimiter{
		limit:      limit,
		windowSize: window,
	}
}

func (bl *BackpressureLimiter) Acquire() {
	ch := make(chan struct{})
	
	bl.mu.Lock()
	bl.queue = append(bl.queue, ch)
	bl.mu.Unlock()
	
	// Process queue
	go bl.processQueue()
	
	<-ch
}

func (bl *BackpressureLimiter) processQueue() {
	bl.mu.Lock()
	defer bl.mu.Unlock()
	
	if len(bl.queue) == 0 {
		return
	}
	
	now := time.Now()
	cutoff := now.Add(-bl.windowSize)
	
	// Remove old timestamps
	valid := bl.timestamps[:0]
	for _, t := range bl.timestamps {
		if t.After(cutoff) {
			valid = append(valid, t)
		}
	}
	bl.timestamps = valid
	
	if len(bl.timestamps) < bl.limit {
		bl.timestamps = append(bl.timestamps, now)
		ch := bl.queue[0]
		bl.queue = bl.queue[1:]
		close(ch)
	}
}

// ตัวอย่าง 20: Circuit breaker pattern (complements rate limiting)
type CircuitState int

const (
	StateClosed   CircuitState = iota // Normal operation
	StateOpen                         // Failing, reject all
	StateHalfOpen                     // Testing recovery
)

type CircuitBreaker struct {
	mu           sync.Mutex
	state        CircuitState
	failures     int
	maxFailures  int
	timeout      time.Duration
	lastFailTime time.Time
	
	successThreshold int
	successCount     int
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
	return &CircuitBreaker{
		state:            StateClosed,
		maxFailures:      maxFailures,
		timeout:          timeout,
		successThreshold: 2,
	}
}

func (cb *CircuitBreaker) Call(fn func() error) error {
	cb.mu.Lock()
	
	switch cb.state {
	case StateOpen:
		if time.Since(cb.lastFailTime) > cb.timeout {
			cb.state = StateHalfOpen
			cb.successCount = 0
			cb.mu.Unlock()
		} else {
			cb.mu.Unlock()
			return fmt.Errorf("circuit breaker is open")
		}
		
	case StateHalfOpen:
		cb.mu.Unlock()
		
	case StateClosed:
		cb.mu.Unlock()
	}
	
	err := fn()
	
	cb.mu.Lock()
	defer cb.mu.Unlock()
	
	if err != nil {
		cb.failures++
		cb.lastFailTime = time.Now()
		
		if cb.state == StateHalfOpen || cb.failures >= cb.maxFailures {
			cb.state = StateOpen
			fmt.Printf("Circuit breaker opened (failures=%d)\n", cb.failures)
		}
		return err
	}
	
	// Success
	if cb.state == StateHalfOpen {
		cb.successCount++
		if cb.successCount >= cb.successThreshold {
			cb.state = StateClosed
			cb.failures = 0
			fmt.Println("Circuit breaker closed (recovered)")
		}
	} else {
		cb.failures = 0
	}
	
	return nil
}

func circuitBreakerDemo() {
	cb := NewCircuitBreaker(3, 500*time.Millisecond)
	
	fmt.Println("=== Circuit Breaker Demo ===")
	
	// Simulate service calls
	callCount := 0
	for i := 0; i < 15; i++ {
		callCount++
		err := cb.Call(func() error {
			// Fail for calls 3-8
			if callCount >= 3 && callCount <= 8 {
				return fmt.Errorf("service unavailable")
			}
			return nil
		})
		
		if err != nil {
			fmt.Printf("Call %2d: FAILED - %v\n", i+1, err)
		} else {
			fmt.Printf("Call %2d: SUCCESS\n", i+1)
		}
		
		time.Sleep(100 * time.Millisecond)
	}
}

func main() {
	simulateDistributedRateLimit()
	fmt.Println()
	circuitBreakerDemo()
}
```

---

## สรุป

ใน Part 38 เราได้เรียนรู้:

1. **Token Bucket**: algorithm ที่ยืดหยุ่นสำหรับ burst traffic
2. **Leaky Bucket**: ควบคุม output rate ให้สม่ำเสมอ
3. **Fixed Window Counter**: นับ requests ในช่วงเวลาคงที่
4. **Sliding Window Log**: ติดตาม requests แม่นยำแต่ใช้ memory มากกว่า
5. **golang.org/x/time/rate**: standard library สำหรับ rate limiting
6. **Per-User Rate Limiting**: จำกัดแต่ละ user แยกกัน
7. **Tiered Rate Limiting**: ระดับ limit ตาม subscription tier
8. **Gin Middleware**: integration กับ HTTP framework
9. **Redis-Based**: distributed rate limiting
10. **Circuit Breaker**: pattern ที่ทำงานร่วมกับ rate limiting

---

## Resources

- [golang.org/x/time/rate](https://pkg.go.dev/golang.org/x/time/rate)
- [Rate Limiting Patterns](https://stripe.com/blog/rate-limiters)
- [Redis Rate Limiting](https://redis.io/docs/manual/patterns/rate-limiting/)
- [Gin Framework](https://gin-gonic.com/docs/)
