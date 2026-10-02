# Part 39: Caching ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ใช้ In-memory caching ด้วย sync.Map และ custom LRU
- ใช้ groupcache, bigcache และ freecache
- เข้าใจ Cache patterns: Cache-aside, Write-through, Write-behind
- จัดการ TTL และ Eviction policies
- สร้าง Cache warming strategy
- ออกแบบ Cache invalidation

---

## 1. In-Memory Caching ด้วย sync.Map

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 1: Simple sync.Map cache
type SimpleCache struct {
	data sync.Map
}

func (c *SimpleCache) Set(key string, value interface{}) {
	c.data.Store(key, value)
}

func (c *SimpleCache) Get(key string) (interface{}, bool) {
	return c.data.Load(key)
}

func (c *SimpleCache) Delete(key string) {
	c.data.Delete(key)
}

func (c *SimpleCache) GetOrSet(key string, loader func() (interface{}, error)) (interface{}, error) {
	if v, ok := c.data.Load(key); ok {
		return v, nil
	}
	
	value, err := loader()
	if err != nil {
		return nil, err
	}
	
	c.data.Store(key, value)
	return value, nil
}

// ตัวอย่าง 2: Cache with TTL
type TTLCache struct {
	mu    sync.RWMutex
	items map[string]*cacheItem
}

type cacheItem struct {
	value     interface{}
	expiresAt time.Time
}

func NewTTLCache() *TTLCache {
	c := &TTLCache{
		items: make(map[string]*cacheItem),
	}
	go c.cleanup()
	return c
}

func (c *TTLCache) Set(key string, value interface{}, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	c.items[key] = &cacheItem{
		value:     value,
		expiresAt: time.Now().Add(ttl),
	}
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	
	item, ok := c.items[key]
	if !ok {
		return nil, false
	}
	
	if time.Now().After(item.expiresAt) {
		return nil, false // expired
	}
	
	return item.value, true
}

func (c *TTLCache) Delete(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	delete(c.items, key)
}

func (c *TTLCache) cleanup() {
	ticker := time.NewTicker(time.Minute)
	defer ticker.Stop()
	
	for range ticker.C {
		c.mu.Lock()
		now := time.Now()
		for key, item := range c.items {
			if now.After(item.expiresAt) {
				delete(c.items, key)
			}
		}
		c.mu.Unlock()
	}
}

func (c *TTLCache) Stats() map[string]int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	
	active := 0
	expired := 0
	now := time.Now()
	
	for _, item := range c.items {
		if now.After(item.expiresAt) {
			expired++
		} else {
			active++
		}
	}
	
	return map[string]int{
		"total":   len(c.items),
		"active":  active,
		"expired": expired,
	}
}

func cacheTTLDemo() {
	cache := NewTTLCache()
	
	fmt.Println("=== TTL Cache Demo ===")
	
	cache.Set("user:1", map[string]string{"name": "Alice", "email": "alice@example.com"}, 2*time.Second)
	cache.Set("user:2", map[string]string{"name": "Bob", "email": "bob@example.com"}, 500*time.Millisecond)
	
	// Immediate get
	if v, ok := cache.Get("user:1"); ok {
		fmt.Printf("Got user:1 immediately: %v\n", v)
	}
	
	// Wait for user:2 to expire
	time.Sleep(600 * time.Millisecond)
	
	if _, ok := cache.Get("user:2"); !ok {
		fmt.Println("user:2 expired as expected")
	}
	
	if v, ok := cache.Get("user:1"); ok {
		fmt.Printf("user:1 still valid: %v\n", v)
	}
	
	fmt.Printf("Stats: %v\n", cache.Stats())
}

func main() {
	cacheTTLDemo()
}
```

---

## 2. Custom LRU Cache

```go
package main

import (
	"container/list"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 3: LRU Cache Implementation

type LRUCache struct {
	mu       sync.Mutex
	capacity int
	items    map[string]*list.Element
	order    *list.List
}

type lruEntry struct {
	key   string
	value interface{}
}

func NewLRUCache(capacity int) *LRUCache {
	return &LRUCache{
		capacity: capacity,
		items:    make(map[string]*list.Element),
		order:    list.New(),
	}
}

func (c *LRUCache) Get(key string) (interface{}, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	if elem, ok := c.items[key]; ok {
		c.order.MoveToFront(elem)
		return elem.Value.(*lruEntry).value, true
	}
	return nil, false
}

func (c *LRUCache) Set(key string, value interface{}) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	if elem, ok := c.items[key]; ok {
		c.order.MoveToFront(elem)
		elem.Value.(*lruEntry).value = value
		return
	}
	
	// Add new entry
	entry := &lruEntry{key: key, value: value}
	elem := c.order.PushFront(entry)
	c.items[key] = elem
	
	// Evict if over capacity
	for c.order.Len() > c.capacity {
		oldest := c.order.Back()
		if oldest != nil {
			c.order.Remove(oldest)
			delete(c.items, oldest.Value.(*lruEntry).key)
		}
	}
}

func (c *LRUCache) Delete(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	if elem, ok := c.items[key]; ok {
		c.order.Remove(elem)
		delete(c.items, key)
	}
}

func (c *LRUCache) Len() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.order.Len()
}

func (c *LRUCache) Keys() []string {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	keys := make([]string, 0, c.order.Len())
	for key := range c.items {
		keys = append(keys, key)
	}
	return keys
}

// ตัวอย่าง 4: LRU with TTL
type LRUTTLCache struct {
	mu       sync.Mutex
	capacity int
	items    map[string]*list.Element
	order    *list.List
}

type lruTTLEntry struct {
	key       string
	value     interface{}
	expiresAt time.Time
}

func NewLRUTTLCache(capacity int) *LRUTTLCache {
	c := &LRUTTLCache{
		capacity: capacity,
		items:    make(map[string]*list.Element),
		order:    list.New(),
	}
	go c.cleanup()
	return c
}

func (c *LRUTTLCache) Get(key string) (interface{}, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	if elem, ok := c.items[key]; ok {
		entry := elem.Value.(*lruTTLEntry)
		if time.Now().After(entry.expiresAt) {
			// Expired
			c.order.Remove(elem)
			delete(c.items, key)
			return nil, false
		}
		c.order.MoveToFront(elem)
		return entry.value, true
	}
	return nil, false
}

func (c *LRUTTLCache) Set(key string, value interface{}, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	expiresAt := time.Now().Add(ttl)
	
	if elem, ok := c.items[key]; ok {
		c.order.MoveToFront(elem)
		entry := elem.Value.(*lruTTLEntry)
		entry.value = value
		entry.expiresAt = expiresAt
		return
	}
	
	entry := &lruTTLEntry{key: key, value: value, expiresAt: expiresAt}
	elem := c.order.PushFront(entry)
	c.items[key] = elem
	
	for c.order.Len() > c.capacity {
		oldest := c.order.Back()
		if oldest != nil {
			c.order.Remove(oldest)
			delete(c.items, oldest.Value.(*lruTTLEntry).key)
		}
	}
}

func (c *LRUTTLCache) cleanup() {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()
	
	for range ticker.C {
		c.mu.Lock()
		now := time.Now()
		for elem := c.order.Back(); elem != nil; {
			entry := elem.Value.(*lruTTLEntry)
			prev := elem.Prev()
			if now.After(entry.expiresAt) {
				c.order.Remove(elem)
				delete(c.items, entry.key)
			}
			elem = prev
		}
		c.mu.Unlock()
	}
}

func lruCacheDemo() {
	lru := NewLRUCache(3)
	
	fmt.Println("=== LRU Cache Demo ===")
	
	lru.Set("a", 1)
	lru.Set("b", 2)
	lru.Set("c", 3)
	fmt.Printf("After set a,b,c: len=%d, keys=%v\n", lru.Len(), lru.Keys())
	
	// Access 'a' to make it recently used
	lru.Get("a")
	
	// Add 'd' - should evict 'b' (least recently used)
	lru.Set("d", 4)
	fmt.Printf("After set d (evicts b): len=%d\n", lru.Len())
	
	if _, ok := lru.Get("b"); !ok {
		fmt.Println("'b' was evicted correctly")
	}
	
	if v, ok := lru.Get("a"); ok {
		fmt.Printf("'a' still present: %v\n", v)
	}
}

func main() {
	lruCacheDemo()
}
```

---

## 3. Cache Patterns

### 3.1 Cache-Aside Pattern

```go
package main

import (
	"context"
	"fmt"
	"time"
	"errors"
)

// ตัวอย่าง 5: Cache-Aside Pattern
// Application code manages cache explicitly

type User struct {
	ID    int
	Name  string
	Email string
}

type UserRepository interface {
	FindByID(ctx context.Context, id int) (*User, error)
	Save(ctx context.Context, user *User) error
}

type Cache interface {
	Get(key string) (interface{}, bool)
	Set(key string, value interface{}, ttl time.Duration)
	Delete(key string)
}

// Cache-Aside: Read
// 1. Check cache first
// 2. If miss, load from DB
// 3. Store in cache
// 4. Return data

type CachedUserService struct {
	repo  UserRepository
	cache Cache
	ttl   time.Duration
}

func NewCachedUserService(repo UserRepository, cache Cache, ttl time.Duration) *CachedUserService {
	return &CachedUserService{repo: repo, cache: cache, ttl: ttl}
}

func (s *CachedUserService) GetUser(ctx context.Context, id int) (*User, error) {
	cacheKey := fmt.Sprintf("user:%d", id)
	
	// Step 1: Check cache
	if cached, ok := s.cache.Get(cacheKey); ok {
		fmt.Printf("Cache HIT for user %d\n", id)
		return cached.(*User), nil
	}
	
	fmt.Printf("Cache MISS for user %d, loading from DB\n", id)
	
	// Step 2: Load from DB
	user, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return nil, err
	}
	
	// Step 3: Store in cache
	s.cache.Set(cacheKey, user, s.ttl)
	
	return user, nil
}

func (s *CachedUserService) UpdateUser(ctx context.Context, user *User) error {
	// Update in DB
	if err := s.repo.Save(ctx, user); err != nil {
		return err
	}
	
	// Invalidate cache (delete)
	cacheKey := fmt.Sprintf("user:%d", user.ID)
	s.cache.Delete(cacheKey)
	fmt.Printf("Cache invalidated for user %d\n", user.ID)
	
	return nil
}

// ตัวอย่าง 6: Write-Through Pattern
// Write to cache and DB simultaneously

type WriteThroughUserService struct {
	repo  UserRepository
	cache Cache
	ttl   time.Duration
}

func (s *WriteThroughUserService) UpdateUser(ctx context.Context, user *User) error {
	// Step 1: Update cache
	cacheKey := fmt.Sprintf("user:%d", user.ID)
	s.cache.Set(cacheKey, user, s.ttl)
	
	// Step 2: Update DB
	if err := s.repo.Save(ctx, user); err != nil {
		// Rollback cache on DB failure
		s.cache.Delete(cacheKey)
		return fmt.Errorf("db write failed, cache rolled back: %w", err)
	}
	
	fmt.Printf("Write-through: updated cache and DB for user %d\n", user.ID)
	return nil
}

// ตัวอย่าง 7: Write-Behind (Write-Back) Pattern
// Write to cache immediately, DB asynchronously

type WriteBehindCache struct {
	cache     Cache
	repo      UserRepository
	writeQueue chan *User
	batchSize int
	flushInterval time.Duration
}

func NewWriteBehindCache(repo UserRepository, cache Cache) *WriteBehindCache {
	wbc := &WriteBehindCache{
		cache:         cache,
		repo:          repo,
		writeQueue:    make(chan *User, 1000),
		batchSize:     50,
		flushInterval: 5 * time.Second,
	}
	go wbc.flushWorker()
	return wbc
}

func (wbc *WriteBehindCache) UpdateUser(ctx context.Context, user *User) error {
	// Step 1: Update cache immediately
	cacheKey := fmt.Sprintf("user:%d", user.ID)
	wbc.cache.Set(cacheKey, user, 10*time.Minute)
	
	// Step 2: Queue DB write (non-blocking)
	select {
	case wbc.writeQueue <- user:
		fmt.Printf("Write-behind: user %d queued for DB write\n", user.ID)
	default:
		// Queue full, write synchronously
		return wbc.repo.Save(ctx, user)
	}
	
	return nil
}

func (wbc *WriteBehindCache) flushWorker() {
	ticker := time.NewTicker(wbc.flushInterval)
	defer ticker.Stop()
	
	var batch []*User
	
	for {
		select {
		case user := <-wbc.writeQueue:
			batch = append(batch, user)
			if len(batch) >= wbc.batchSize {
				wbc.flush(batch)
				batch = batch[:0]
			}
		case <-ticker.C:
			if len(batch) > 0 {
				wbc.flush(batch)
				batch = batch[:0]
			}
		}
	}
}

func (wbc *WriteBehindCache) flush(users []*User) {
	fmt.Printf("Flushing %d users to DB\n", len(users))
	for _, user := range users {
		ctx := context.Background()
		if err := wbc.repo.Save(ctx, user); err != nil {
			fmt.Printf("Error saving user %d: %v\n", user.ID, err)
		}
	}
}

// Mock implementations
type MockUserRepo struct {
	users map[int]*User
}

func NewMockUserRepo() *MockUserRepo {
	return &MockUserRepo{
		users: map[int]*User{
			1: {ID: 1, Name: "Alice", Email: "alice@example.com"},
			2: {ID: 2, Name: "Bob", Email: "bob@example.com"},
			3: {ID: 3, Name: "Charlie", Email: "charlie@example.com"},
		},
	}
}

func (r *MockUserRepo) FindByID(ctx context.Context, id int) (*User, error) {
	time.Sleep(10 * time.Millisecond) // simulate DB latency
	user, ok := r.users[id]
	if !ok {
		return nil, errors.New("user not found")
	}
	return user, nil
}

func (r *MockUserRepo) Save(ctx context.Context, user *User) error {
	time.Sleep(20 * time.Millisecond) // simulate DB write latency
	r.users[user.ID] = user
	return nil
}

type InMemoryCache struct {
	data sync.Map
	ttls sync.Map
}

func (c *InMemoryCache) Get(key string) (interface{}, bool) {
	ttlVal, ok := c.ttls.Load(key)
	if ok {
		if time.Now().After(ttlVal.(time.Time)) {
			c.data.Delete(key)
			c.ttls.Delete(key)
			return nil, false
		}
	}
	return c.data.Load(key)
}

func (c *InMemoryCache) Set(key string, value interface{}, ttl time.Duration) {
	c.data.Store(key, value)
	c.ttls.Store(key, time.Now().Add(ttl))
}

func (c *InMemoryCache) Delete(key string) {
	c.data.Delete(key)
	c.ttls.Delete(key)
}

import "sync"

func cachePatternsDemo() {
	repo := NewMockUserRepo()
	cache := &InMemoryCache{}
	
	// Cache-Aside demo
	service := NewCachedUserService(repo, cache, 5*time.Minute)
	ctx := context.Background()
	
	fmt.Println("=== Cache-Aside Pattern Demo ===")
	
	// First call - cache miss
	user, _ := service.GetUser(ctx, 1)
	fmt.Printf("Got: %+v\n", user)
	
	// Second call - cache hit
	user, _ = service.GetUser(ctx, 1)
	fmt.Printf("Got: %+v\n", user)
	
	// Update - invalidate cache
	user.Name = "Alice Updated"
	service.UpdateUser(ctx, user)
	
	// Next call - cache miss again (invalidated)
	user, _ = service.GetUser(ctx, 1)
	fmt.Printf("After update: %+v\n", user)
}

func main() {
	cachePatternsDemo()
}
```

---

## 4. Cache Warming Strategy

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 8: Cache warming on startup

type CacheWarmer struct {
	cache     *TTLCache
	loaders   []WarmupLoader
	batchSize int
	workers   int
}

type WarmupLoader func(ctx context.Context) (map[string]interface{}, error)

func NewCacheWarmer(cache *TTLCache, batchSize, workers int) *CacheWarmer {
	return &CacheWarmer{
		cache:     cache,
		batchSize: batchSize,
		workers:   workers,
	}
}

func (cw *CacheWarmer) AddLoader(loader WarmupLoader) {
	cw.loaders = append(cw.loaders, loader)
}

func (cw *CacheWarmer) Warm(ctx context.Context, ttl time.Duration) error {
	fmt.Println("Starting cache warming...")
	start := time.Now()
	
	var wg sync.WaitGroup
	errs := make(chan error, len(cw.loaders))
	
	for _, loader := range cw.loaders {
		wg.Add(1)
		go func(l WarmupLoader) {
			defer wg.Done()
			
			data, err := l(ctx)
			if err != nil {
				errs <- err
				return
			}
			
			for key, value := range data {
				cw.cache.Set(key, value, ttl)
			}
			
			fmt.Printf("Warmed %d items\n", len(data))
		}(loader)
	}
	
	wg.Wait()
	close(errs)
	
	var errList []error
	for err := range errs {
		errList = append(errList, err)
	}
	
	fmt.Printf("Cache warming completed in %v, %d errors\n",
		time.Since(start).Round(time.Millisecond), len(errList))
	
	if len(errList) > 0 {
		return fmt.Errorf("%d loaders failed", len(errList))
	}
	return nil
}

// ตัวอย่าง 9: Lazy loading with singleflight
type LazyCache struct {
	cache  *TTLCache
	group  singleflight.Group
	loader func(ctx context.Context, key string) (interface{}, error)
	ttl    time.Duration
}

// Note: using sync/singleflight
import "golang.org/x/sync/singleflight"

func NewLazyCache(loader func(ctx context.Context, key string) (interface{}, error), ttl time.Duration) *LazyCache {
	return &LazyCache{
		cache:  NewTTLCache(),
		loader: loader,
		ttl:    ttl,
	}
}

func (lc *LazyCache) Get(ctx context.Context, key string) (interface{}, error) {
	if v, ok := lc.cache.Get(key); ok {
		return v, nil
	}
	
	// Use singleflight to prevent cache stampede
	value, err, _ := lc.group.Do(key, func() (interface{}, error) {
		// Double-check cache after acquiring lock
		if v, ok := lc.cache.Get(key); ok {
			return v, nil
		}
		
		// Load from source
		v, err := lc.loader(ctx, key)
		if err != nil {
			return nil, err
		}
		
		lc.cache.Set(key, v, lc.ttl)
		return v, nil
	})
	
	if err != nil {
		return nil, err
	}
	return value, nil
}

func cacheWarmingDemo() {
	cache := NewTTLCache()
	warmer := NewCacheWarmer(cache, 100, 4)
	
	// Add data loaders
	warmer.AddLoader(func(ctx context.Context) (map[string]interface{}, error) {
		// Simulate loading users
		data := make(map[string]interface{})
		for i := 1; i <= 100; i++ {
			data[fmt.Sprintf("user:%d", i)] = map[string]string{
				"name":  fmt.Sprintf("User%d", i),
				"email": fmt.Sprintf("user%d@example.com", i),
			}
		}
		return data, nil
	})
	
	warmer.AddLoader(func(ctx context.Context) (map[string]interface{}, error) {
		// Simulate loading products
		data := make(map[string]interface{})
		for i := 1; i <= 50; i++ {
			data[fmt.Sprintf("product:%d", i)] = map[string]interface{}{
				"name":  fmt.Sprintf("Product%d", i),
				"price": float64(i) * 9.99,
			}
		}
		return data, nil
	})
	
	ctx := context.Background()
	if err := warmer.Warm(ctx, 30*time.Minute); err != nil {
		fmt.Printf("Warning: cache warming had errors: %v\n", err)
	}
	
	// Test cache
	if v, ok := cache.Get("user:42"); ok {
		fmt.Printf("user:42 = %v\n", v)
	}
	
	fmt.Printf("Cache stats: %v\n", cache.Stats())
}

func main() {
	cacheWarmingDemo()
}
```

---

## 5. Cache Invalidation Strategies

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 10: Event-based cache invalidation

type CacheEvent struct {
	Type     string // "set", "delete", "expire"
	Key      string
	Value    interface{}
	ExpireAt time.Time
}

type InvalidationCache struct {
	mu          sync.RWMutex
	data        map[string]*cacheEntry
	subscribers []chan CacheEvent
}

type cacheEntry struct {
	value     interface{}
	expiresAt time.Time
	version   int64
}

func NewInvalidationCache() *InvalidationCache {
	c := &InvalidationCache{
		data: make(map[string]*cacheEntry),
	}
	go c.expireWorker()
	return c
}

func (c *InvalidationCache) Subscribe() <-chan CacheEvent {
	ch := make(chan CacheEvent, 100)
	c.mu.Lock()
	c.subscribers = append(c.subscribers, ch)
	c.mu.Unlock()
	return ch
}

func (c *InvalidationCache) publish(event CacheEvent) {
	for _, sub := range c.subscribers {
		select {
		case sub <- event:
		default:
			// Subscriber slow, skip
		}
	}
}

func (c *InvalidationCache) Set(key string, value interface{}, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	entry := &cacheEntry{
		value:     value,
		expiresAt: time.Now().Add(ttl),
		version:   time.Now().UnixNano(),
	}
	
	c.data[key] = entry
	c.publish(CacheEvent{
		Type:     "set",
		Key:      key,
		Value:    value,
		ExpireAt: entry.expiresAt,
	})
}

func (c *InvalidationCache) Get(key string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	
	entry, ok := c.data[key]
	if !ok {
		return nil, false
	}
	
	if time.Now().After(entry.expiresAt) {
		return nil, false
	}
	
	return entry.value, true
}

func (c *InvalidationCache) Invalidate(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	if _, ok := c.data[key]; ok {
		delete(c.data, key)
		c.publish(CacheEvent{Type: "delete", Key: key})
	}
}

func (c *InvalidationCache) InvalidateByPrefix(prefix string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	for key := range c.data {
		if len(key) >= len(prefix) && key[:len(prefix)] == prefix {
			delete(c.data, key)
			c.publish(CacheEvent{Type: "delete", Key: key})
		}
	}
}

func (c *InvalidationCache) expireWorker() {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()
	
	for range ticker.C {
		c.mu.Lock()
		now := time.Now()
		for key, entry := range c.data {
			if now.After(entry.expiresAt) {
				delete(c.data, key)
				c.publish(CacheEvent{Type: "expire", Key: key})
			}
		}
		c.mu.Unlock()
	}
}

// ตัวอย่าง 11: Tag-based invalidation
type TaggedCache struct {
	mu       sync.RWMutex
	data     map[string]*taggedEntry
	tagIndex map[string][]string // tag -> keys
}

type taggedEntry struct {
	value     interface{}
	expiresAt time.Time
	tags      []string
}

func NewTaggedCache() *TaggedCache {
	return &TaggedCache{
		data:     make(map[string]*taggedEntry),
		tagIndex: make(map[string][]string),
	}
}

func (c *TaggedCache) Set(key string, value interface{}, ttl time.Duration, tags ...string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	// Remove old tags for this key
	if old, ok := c.data[key]; ok {
		for _, tag := range old.tags {
			c.removeFromTagIndex(tag, key)
		}
	}
	
	c.data[key] = &taggedEntry{
		value:     value,
		expiresAt: time.Now().Add(ttl),
		tags:      tags,
	}
	
	// Update tag index
	for _, tag := range tags {
		c.tagIndex[tag] = append(c.tagIndex[tag], key)
	}
}

func (c *TaggedCache) Get(key string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	
	entry, ok := c.data[key]
	if !ok || time.Now().After(entry.expiresAt) {
		return nil, false
	}
	
	return entry.value, true
}

func (c *TaggedCache) InvalidateByTag(tag string) int {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	keys, ok := c.tagIndex[tag]
	if !ok {
		return 0
	}
	
	count := 0
	for _, key := range keys {
		if _, ok := c.data[key]; ok {
			delete(c.data, key)
			count++
		}
	}
	delete(c.tagIndex, tag)
	
	return count
}

func (c *TaggedCache) removeFromTagIndex(tag, key string) {
	keys := c.tagIndex[tag]
	for i, k := range keys {
		if k == key {
			c.tagIndex[tag] = append(keys[:i], keys[i+1:]...)
			break
		}
	}
}

func cacheInvalidationDemo() {
	fmt.Println("=== Tag-based Cache Invalidation ===")
	
	tc := NewTaggedCache()
	
	// Store user-related data with tags
	tc.Set("user:1", map[string]string{"name": "Alice"}, 10*time.Minute, "user", "user:1")
	tc.Set("user:1:profile", map[string]string{"avatar": "alice.jpg"}, 10*time.Minute, "user", "user:1")
	tc.Set("user:1:settings", map[string]string{"theme": "dark"}, 10*time.Minute, "user", "user:1")
	tc.Set("user:2", map[string]string{"name": "Bob"}, 10*time.Minute, "user", "user:2")
	tc.Set("product:1", map[string]string{"name": "Widget"}, 10*time.Minute, "product")
	
	// Invalidate all user:1 data
	count := tc.InvalidateByTag("user:1")
	fmt.Printf("Invalidated %d entries for user:1\n", count)
	
	if _, ok := tc.Get("user:1"); !ok {
		fmt.Println("user:1 data cleared")
	}
	if _, ok := tc.Get("user:1:profile"); !ok {
		fmt.Println("user:1:profile cleared")
	}
	if v, ok := tc.Get("user:2"); ok {
		fmt.Printf("user:2 still exists: %v\n", v)
	}
	if v, ok := tc.Get("product:1"); ok {
		fmt.Printf("product:1 still exists: %v\n", v)
	}
}

func main() {
	cacheInvalidationDemo()
}
```

---

## 6. BigCache / FreeCache Integration

```go
package main

import (
	"fmt"
	"time"
	
	"github.com/allegro/bigcache/v3"
	"github.com/coocood/freecache"
)

// ตัวอย่าง 12: BigCache - designed for storing large amounts of data

func bigCacheDemo() {
	// BigCache configuration
	config := bigcache.Config{
		Shards:           1024,                // number of cache shards
		LifeWindow:       10 * time.Minute,    // time after which entry can be evicted
		CleanWindow:      5 * time.Minute,     // interval between removing expired entries
		MaxEntriesInWindow: 1000 * 10 * 60,   // max number of entries in given window
		MaxEntrySize:     500,                  // max size of entry in bytes
		HardMaxCacheSize: 512,                  // max size of cache in MB
	}
	
	cache, err := bigcache.New(context.Background(), config)
	if err != nil {
		panic(err)
	}
	defer cache.Close()
	
	// Store data
	cache.Set("user:1", []byte(`{"name":"Alice","email":"alice@example.com"}`))
	cache.Set("user:2", []byte(`{"name":"Bob","email":"bob@example.com"}`))
	
	// Retrieve data
	if entry, err := cache.Get("user:1"); err == nil {
		fmt.Printf("BigCache get: %s\n", entry)
	}
	
	// Delete
	cache.Delete("user:2")
	
	// Stats
	stats := cache.Stats()
	fmt.Printf("BigCache stats: hits=%d, misses=%d\n", stats.Hits, stats.Misses)
	
	fmt.Printf("BigCache entries: %d, capacity: %.1fMB\n",
		cache.Len(), float64(cache.Capacity())/1024/1024)
}

// ตัวอย่าง 13: FreeCache - offheap cache to reduce GC pressure

func freeCacheDemo() {
	// 100MB cache
	cacheSize := 100 * 1024 * 1024
	cache := freecache.NewCache(cacheSize)
	
	// Set with TTL in seconds
	key := []byte("session:abc123")
	value := []byte(`{"user_id":1,"role":"admin","expires":"2025-01-01"}`)
	ttl := 3600 // 1 hour
	
	err := cache.Set(key, value, ttl)
	if err != nil {
		fmt.Printf("FreeCache set error: %v\n", err)
		return
	}
	
	// Get
	got, err := cache.Get(key)
	if err != nil {
		fmt.Printf("FreeCache get error: %v\n", err)
	} else {
		fmt.Printf("FreeCache get: %s\n", got)
	}
	
	// TTL remaining
	remaining, err := cache.TTL(key)
	if err == nil {
		fmt.Printf("TTL remaining: %d seconds\n", remaining)
	}
	
	// Stats
	fmt.Printf("FreeCache: entries=%d, hit_rate=%.1f%%\n",
		cache.EntryCount(),
		float64(cache.HitCount())/float64(cache.LookupCount()+1)*100)
	
	cache.Del(key)
	fmt.Println("Session deleted")
}

// ตัวอย่าง 14: Choosing the right cache library

func cacheComparison() {
	fmt.Println("=== Cache Library Comparison ===")
	fmt.Println()
	fmt.Println("sync.Map:")
	fmt.Println("  ✓ Built-in, no dependencies")
	fmt.Println("  ✓ Thread-safe")
	fmt.Println("  ✗ No TTL support")
	fmt.Println("  ✗ No eviction policy")
	fmt.Println("  Best for: Small, long-lived caches")
	fmt.Println()
	fmt.Println("Custom LRU:")
	fmt.Println("  ✓ Full control")
	fmt.Println("  ✓ LRU eviction")
	fmt.Println("  ✗ More code to maintain")
	fmt.Println("  Best for: When you need specific behavior")
	fmt.Println()
	fmt.Println("BigCache:")
	fmt.Println("  ✓ Very fast (lock-free design)")
	fmt.Println("  ✓ Low GC pressure")
	fmt.Println("  ✓ High throughput")
	fmt.Println("  ✗ Keys must be strings")
	fmt.Println("  ✗ Values must be bytes")
	fmt.Println("  Best for: High-throughput HTTP caching")
	fmt.Println()
	fmt.Println("FreeCache:")
	fmt.Println("  ✓ Offheap storage (reduces GC)")
	fmt.Println("  ✓ Built-in TTL")
	fmt.Println("  ✓ Memory efficient")
	fmt.Println("  ✗ Fixed size")
	fmt.Println("  Best for: Session storage, token caching")
}

func main() {
	// Note: These require external packages
	// In practice you'd import and use them
	fmt.Println("=== Cache Library Demos ===")
	fmt.Println("(Requires: github.com/allegro/bigcache/v3, github.com/coocood/freecache)")
	cacheComparison()
}
```

---

## 7. Groupcache

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"
	
	"github.com/golang/groupcache"
)

// ตัวอย่าง 15: Groupcache - distributed caching

func setupGroupcache() {
	// Create a peer pool (in production, list all nodes)
	pool := groupcache.NewHTTPPool("http://localhost:8080")
	
	// Start peer pool server in background
	go func() {
		log.Fatal(http.ListenAndServe(":8080", pool))
	}()
	
	// Create a cache group
	var thumbnails = groupcache.NewGroup("thumbnails", 64<<20, groupcache.GetterFunc(
		func(ctx context.Context, key string, dest groupcache.Sink) error {
			// This function is called on cache miss
			fmt.Printf("Loading thumbnail for: %s\n", key)
			
			// Simulate loading from storage
			time.Sleep(100 * time.Millisecond)
			data := []byte(fmt.Sprintf("thumbnail_data_for_%s", key))
			
			return dest.SetBytes(data, time.Now().Add(time.Hour))
		},
	))
	
	// Use the cache
	var data []byte
	ctx := context.Background()
	
	// First call - cache miss, loads data
	err := thumbnails.Get(ctx, "user_123", groupcache.AllocatingByteSliceSink(&data))
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Got thumbnail: %s\n", data)
	
	// Second call - cache hit
	err = thumbnails.Get(ctx, "user_123", groupcache.AllocatingByteSliceSink(&data))
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Got thumbnail (cached): %s\n", data)
	
	// Get statistics
	stats := thumbnails.Stats
	fmt.Printf("Cache stats: gets=%d, cache_hits=%d\n",
		stats.Gets.Get(), stats.CacheHits.Get())
}

func main() {
	fmt.Println("=== Groupcache Demo ===")
	fmt.Println("(Requires: github.com/golang/groupcache)")
	fmt.Println("Groupcache key features:")
	fmt.Println("  - Consistent hashing")
	fmt.Println("  - Duplicate suppression (single flight)")
	fmt.Println("  - LRU eviction")
	fmt.Println("  - Peer-to-peer distribution")
}
```

---

## 8. Cache Metrics และ Monitoring

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

// ตัวอย่าง 16: Cache with built-in metrics

type MetricsCache struct {
	inner *TTLCache
	
	hits    int64
	misses  int64
	sets    int64
	deletes int64
	evicts  int64
	
	hitDuration  int64 // nanoseconds
	missDuration int64 // nanoseconds
}

func NewMetricsCache() *MetricsCache {
	return &MetricsCache{inner: NewTTLCache()}
}

func (mc *MetricsCache) Get(key string) (interface{}, bool) {
	start := time.Now()
	v, ok := mc.inner.Get(key)
	duration := time.Since(start).Nanoseconds()
	
	if ok {
		atomic.AddInt64(&mc.hits, 1)
		atomic.AddInt64(&mc.hitDuration, duration)
	} else {
		atomic.AddInt64(&mc.misses, 1)
		atomic.AddInt64(&mc.missDuration, duration)
	}
	
	return v, ok
}

func (mc *MetricsCache) Set(key string, value interface{}, ttl time.Duration) {
	atomic.AddInt64(&mc.sets, 1)
	mc.inner.Set(key, value, ttl)
}

func (mc *MetricsCache) Delete(key string) {
	atomic.AddInt64(&mc.deletes, 1)
	mc.inner.Delete(key)
}

func (mc *MetricsCache) HitRate() float64 {
	hits := atomic.LoadInt64(&mc.hits)
	misses := atomic.LoadInt64(&mc.misses)
	total := hits + misses
	if total == 0 {
		return 0
	}
	return float64(hits) / float64(total)
}

func (mc *MetricsCache) AvgHitLatency() time.Duration {
	hits := atomic.LoadInt64(&mc.hits)
	if hits == 0 {
		return 0
	}
	return time.Duration(atomic.LoadInt64(&mc.hitDuration) / hits)
}

func (mc *MetricsCache) AvgMissLatency() time.Duration {
	misses := atomic.LoadInt64(&mc.misses)
	if misses == 0 {
		return 0
	}
	return time.Duration(atomic.LoadInt64(&mc.missDuration) / misses)
}

func (mc *MetricsCache) PrintStats() {
	fmt.Printf("=== Cache Metrics ===\n")
	fmt.Printf("Hits:          %d\n", atomic.LoadInt64(&mc.hits))
	fmt.Printf("Misses:        %d\n", atomic.LoadInt64(&mc.misses))
	fmt.Printf("Hit Rate:      %.1f%%\n", mc.HitRate()*100)
	fmt.Printf("Sets:          %d\n", atomic.LoadInt64(&mc.sets))
	fmt.Printf("Deletes:       %d\n", atomic.LoadInt64(&mc.deletes))
	fmt.Printf("Avg Hit Lat:   %v\n", mc.AvgHitLatency())
	fmt.Printf("Avg Miss Lat:  %v\n", mc.AvgMissLatency())
}

func metricsDemo() {
	mc := NewMetricsCache()
	
	// Populate cache
	for i := 1; i <= 20; i++ {
		mc.Set(fmt.Sprintf("key:%d", i), fmt.Sprintf("value:%d", i), 5*time.Minute)
	}
	
	// Mix of hits and misses
	for i := 1; i <= 30; i++ {
		mc.Get(fmt.Sprintf("key:%d", i)) // 20 hits, 10 misses
	}
	
	mc.PrintStats()
}

func main() {
	metricsDemo()
}
```

---

## สรุป

ใน Part 39 เราได้เรียนรู้:

1. **sync.Map**: built-in thread-safe map, ง่ายแต่ไม่มี TTL
2. **Custom LRU**: ควบคุมการ evict ด้วย least-recently-used policy
3. **Cache-Aside**: application จัดการ cache เอง
4. **Write-Through**: เขียน cache และ DB พร้อมกัน
5. **Write-Behind**: เขียน cache ก่อน, DB asynchronously
6. **Cache Warming**: เติม cache ก่อนเปิดให้บริการ
7. **Tag-based Invalidation**: invalidate หลาย keys ด้วย tag เดียว
8. **BigCache/FreeCache**: high-performance external libraries
9. **Groupcache**: distributed caching
10. **Metrics**: ติดตาม cache performance

---

## Resources

- [BigCache](https://github.com/allegro/bigcache)
- [FreeCache](https://github.com/coocood/freecache)
- [GroupCache](https://github.com/golang/groupcache)
- [Caching Best Practices](https://aws.amazon.com/caching/best-practices/)
