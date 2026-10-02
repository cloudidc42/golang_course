# Part 71: Advanced Caching

## เป้าหมายการเรียนรู้
- Multi-level Caching
- Cache Consistency Patterns
- Write-through vs Write-back
- Cache Stampede Prevention
- Bloom Filters
- Probabilistic Caching
- CDN Integration
- Edge Caching

---

## 1. Multi-level Caching

```
Request → L1 Cache (in-memory) → L2 Cache (Redis) → Database
              (fastest)              (fast)           (slowest)
              ~0.1ms                  ~1ms              ~10ms
```

```go
// cache/multilevel.go
package cache

import (
    "context"
    "encoding/json"
    "fmt"
    "sync"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type CacheEntry struct {
    Value     interface{}
    ExpiresAt time.Time
}

// L1: In-memory LRU cache
type L1Cache struct {
    mu       sync.RWMutex
    data     map[string]*CacheEntry
    maxItems int
    order    []string // สำหรับ LRU eviction
}

func NewL1Cache(maxItems int) *L1Cache {
    return &L1Cache{
        data:     make(map[string]*CacheEntry),
        maxItems: maxItems,
    }
}

func (c *L1Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    entry, ok := c.data[key]
    c.mu.RUnlock()
    
    if !ok {
        return nil, false
    }
    
    if time.Now().After(entry.ExpiresAt) {
        c.Delete(key)
        return nil, false
    }
    
    // LRU: เลื่อน key ไปข้างหน้า
    c.mu.Lock()
    c.moveToFront(key)
    c.mu.Unlock()
    
    return entry.Value, true
}

func (c *L1Cache) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Evict ถ้าเต็ม
    if len(c.data) >= c.maxItems {
        c.evictLRU()
    }
    
    c.data[key] = &CacheEntry{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
    c.order = append([]string{key}, c.order...)
}

func (c *L1Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.data, key)
}

func (c *L1Cache) evictLRU() {
    if len(c.order) == 0 {
        return
    }
    oldest := c.order[len(c.order)-1]
    c.order = c.order[:len(c.order)-1]
    delete(c.data, oldest)
}

func (c *L1Cache) moveToFront(key string) {
    for i, k := range c.order {
        if k == key {
            c.order = append(c.order[:i], c.order[i+1:]...)
            c.order = append([]string{key}, c.order...)
            return
        }
    }
}

// L2: Redis Cache
type L2Cache struct {
    client *redis.Client
    prefix string
}

func NewL2Cache(client *redis.Client, prefix string) *L2Cache {
    return &L2Cache{client: client, prefix: prefix}
}

func (c *L2Cache) Get(ctx context.Context, key string) ([]byte, error) {
    val, err := c.client.Get(ctx, c.prefix+key).Bytes()
    if err == redis.Nil {
        return nil, nil // Cache miss
    }
    return val, err
}

func (c *L2Cache) Set(ctx context.Context, key string, value []byte, ttl time.Duration) error {
    return c.client.Set(ctx, c.prefix+key, value, ttl).Err()
}

func (c *L2Cache) Delete(ctx context.Context, key string) error {
    return c.client.Del(ctx, c.prefix+key).Err()
}

// Multi-level cache manager
type MultiLevelCache struct {
    l1     *L1Cache
    l2     *L2Cache
    l1TTL  time.Duration
    l2TTL  time.Duration
}

func NewMultiLevelCache(l1 *L1Cache, l2 *L2Cache) *MultiLevelCache {
    return &MultiLevelCache{
        l1:    l1,
        l2:    l2,
        l1TTL: 30 * time.Second,
        l2TTL: 5 * time.Minute,
    }
}

func (c *MultiLevelCache) Get(ctx context.Context, key string, dest interface{}) error {
    // ลอง L1 ก่อน
    if val, ok := c.l1.Get(key); ok {
        // Copy value to dest
        data, _ := json.Marshal(val)
        return json.Unmarshal(data, dest)
    }
    
    // ลอง L2
    data, err := c.l2.Get(ctx, key)
    if err != nil {
        return err
    }
    if data == nil {
        return ErrCacheMiss
    }
    
    // Deserialize
    if err := json.Unmarshal(data, dest); err != nil {
        return err
    }
    
    // Populate L1
    c.l1.Set(key, dest, c.l1TTL)
    
    return nil
}

func (c *MultiLevelCache) Set(ctx context.Context, key string, value interface{}) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    // Set ทั้ง L1 และ L2
    c.l1.Set(key, value, c.l1TTL)
    return c.l2.Set(ctx, key, data, c.l2TTL)
}

func (c *MultiLevelCache) Delete(ctx context.Context, key string) error {
    c.l1.Delete(key)
    return c.l2.Delete(ctx, key)
}

var ErrCacheMiss = fmt.Errorf("cache miss")
```

---

## 2. Cache Consistency

### Write-through Cache

```go
// cache/write_through.go
package cache

import (
    "context"
    "database/sql"
    "encoding/json"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type WriteThroughCache struct {
    redis    *redis.Client
    db       *sql.DB
    ttl      time.Duration
}

// Write-through: เขียน cache และ DB พร้อมกัน
func (c *WriteThroughCache) Set(ctx context.Context, key string, value interface{}) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    // เขียน DB ก่อน
    if err := c.saveToDatabase(ctx, key, value); err != nil {
        return fmt.Errorf("database write failed: %w", err)
    }
    
    // เขียน cache
    if err := c.redis.Set(ctx, key, data, c.ttl).Err(); err != nil {
        // Log error แต่ไม่ fail (DB update สำเร็จแล้ว)
        log.Printf("Cache write warning: %v", err)
    }
    
    return nil
}

func (c *WriteThroughCache) Get(ctx context.Context, key string, dest interface{}) error {
    // ลอง cache ก่อน
    data, err := c.redis.Get(ctx, key).Bytes()
    if err == nil {
        return json.Unmarshal(data, dest)
    }
    
    // Cache miss - โหลดจาก DB
    if err := c.loadFromDatabase(ctx, key, dest); err != nil {
        return err
    }
    
    // Populate cache
    if serialized, err := json.Marshal(dest); err == nil {
        c.redis.Set(ctx, key, serialized, c.ttl)
    }
    
    return nil
}
```

### Write-back (Write-behind) Cache

```go
// cache/write_back.go
package cache

import (
    "context"
    "sync"
    "time"
)

type WriteBackCache struct {
    cache      *redis.Client
    db         *sql.DB
    dirty      map[string]interface{} // รอ flush ไป DB
    mu         sync.RWMutex
    flushInterval time.Duration
}

func NewWriteBackCache(cache *redis.Client, db *sql.DB) *WriteBackCache {
    c := &WriteBackCache{
        cache:         cache,
        db:            db,
        dirty:         make(map[string]interface{}),
        flushInterval: 5 * time.Second,
    }
    
    go c.startFlushLoop()
    return c
}

// Write-back: เขียน cache ก่อน, flush ไป DB ทีหลัง
func (c *WriteBackCache) Set(ctx context.Context, key string, value interface{}) error {
    data, _ := json.Marshal(value)
    
    // เขียน cache ทันที
    if err := c.cache.Set(ctx, key, data, 10*time.Minute).Err(); err != nil {
        return err
    }
    
    // Mark as dirty
    c.mu.Lock()
    c.dirty[key] = value
    c.mu.Unlock()
    
    return nil
}

// Flush dirty entries ไป DB
func (c *WriteBackCache) Flush(ctx context.Context) error {
    c.mu.Lock()
    dirtySnapshot := c.dirty
    c.dirty = make(map[string]interface{})
    c.mu.Unlock()
    
    if len(dirtySnapshot) == 0 {
        return nil
    }
    
    tx, err := c.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    for key, value := range dirtySnapshot {
        if err := c.saveToDatabase(ctx, tx, key, value); err != nil {
            // Re-add to dirty if flush fails
            c.mu.Lock()
            c.dirty[key] = value
            c.mu.Unlock()
            return err
        }
    }
    
    return tx.Commit()
}

func (c *WriteBackCache) startFlushLoop() {
    ticker := time.NewTicker(c.flushInterval)
    for range ticker.C {
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        if err := c.Flush(ctx); err != nil {
            log.Printf("Write-back flush error: %v", err)
        }
        cancel()
    }
}
```

---

## 3. Cache Stampede Prevention

Cache stampede เกิดเมื่อ cache หมดอายุพร้อมกัน ทำให้ทุก request พุ่งไป DB พร้อมกัน

```go
// cache/stampede.go
package cache

import (
    "context"
    "sync"
    "time"
    
    "golang.org/x/sync/singleflight"
)

// Solution 1: Singleflight - request เดียวไป DB แล้ว share ผลลัพธ์
type SingleflightCache struct {
    cache *redis.Client
    group singleflight.Group
}

func (c *SingleflightCache) Get(ctx context.Context, key string, fetch func() (interface{}, error)) (interface{}, error) {
    // ลอง cache ก่อน
    if val, err := c.cache.Get(ctx, key).Result(); err == nil {
        return val, nil
    }
    
    // ใช้ singleflight - มีแค่ 1 request ไป DB
    result, err, _ := c.group.Do(key, func() (interface{}, error) {
        // Double-check cache หลัง wait
        if val, err := c.cache.Get(ctx, key).Result(); err == nil {
            return val, nil
        }
        
        // Fetch จาก DB
        value, err := fetch()
        if err != nil {
            return nil, err
        }
        
        // Cache ผลลัพธ์
        data, _ := json.Marshal(value)
        c.cache.Set(ctx, key, data, 5*time.Minute)
        
        return value, nil
    })
    
    return result, err
}

// Solution 2: Probabilistic Early Expiration (XFetch)
// refresh cache ก่อนหมดอายุโดยใช้ probability
type XFetchCache struct {
    cache *redis.Client
    beta  float64 // ค่าปกติ = 1.0
}

type CacheValue struct {
    Data      interface{} `json:"data"`
    ExpiresAt time.Time   `json:"expires_at"`
    Delta     float64     `json:"delta"` // เวลาที่ใช้ fetch (ms)
}

func (c *XFetchCache) Get(ctx context.Context, key string, fetch func() (interface{}, time.Duration, error)) (interface{}, error) {
    var cached CacheValue
    
    data, err := c.cache.Get(ctx, key).Bytes()
    if err == nil {
        json.Unmarshal(data, &cached)
        
        // XFetch: ตัดสินใจว่าจะ refresh หรือไม่
        ttl := time.Until(cached.ExpiresAt)
        
        // คำนวณ probability
        // P = exp(-delta * beta * log(rand)) > TTL
        rand := generateRandomFloat()
        threshold := -cached.Delta * c.beta * math.Log(rand)
        
        if float64(ttl.Seconds()) > threshold {
            return cached.Data, nil // ไม่ต้อง refresh
        }
        
        // Refresh ก่อนหมดอายุ
        log.Printf("XFetch: proactively refreshing key %s", key)
    }
    
    // Fetch ใหม่
    value, ttl, err := fetch()
    if err != nil {
        if cached.Data != nil {
            return cached.Data, nil // Return stale data
        }
        return nil, err
    }
    
    newCached := CacheValue{
        Data:      value,
        ExpiresAt: time.Now().Add(ttl),
        Delta:     float64(time.Since(time.Now()).Milliseconds()),
    }
    
    serialized, _ := json.Marshal(newCached)
    c.cache.Set(ctx, key, serialized, ttl)
    
    return value, nil
}

// Solution 3: Mutex lock บน cache key
type MutexCache struct {
    cache *redis.Client
    locks sync.Map
}

func (c *MutexCache) Get(ctx context.Context, key string, fetch func() (interface{}, error)) (interface{}, error) {
    // ลอง cache ก่อน
    if val, err := c.cache.Get(ctx, key).Bytes(); err == nil {
        var result interface{}
        json.Unmarshal(val, &result)
        return result, nil
    }
    
    // Lock สำหรับ key นี้
    mu := &sync.Mutex{}
    actual, _ := c.locks.LoadOrStore(key, mu)
    mu = actual.(*sync.Mutex)
    
    mu.Lock()
    defer func() {
        mu.Unlock()
        c.locks.Delete(key)
    }()
    
    // Double check หลัง lock
    if val, err := c.cache.Get(ctx, key).Bytes(); err == nil {
        var result interface{}
        json.Unmarshal(val, &result)
        return result, nil
    }
    
    // Fetch
    value, err := fetch()
    if err != nil {
        return nil, err
    }
    
    data, _ := json.Marshal(value)
    c.cache.Set(ctx, key, data, 5*time.Minute)
    
    return value, nil
}
```

---

## 4. Bloom Filters

Bloom filter ใช้ตรวจสอบว่า key มีอยู่หรือไม่ ก่อนที่จะไป DB (ป้องกัน cache penetration)

```go
// cache/bloom_filter.go
package cache

import (
    "hash"
    "hash/fnv"
    "math"
    "sync"
)

type BloomFilter struct {
    bitset []bool
    k      int // จำนวน hash functions
    m      int // ขนาด bit array
    mu     sync.RWMutex
}

// สร้าง Bloom Filter
// n = จำนวน elements ที่คาดว่าจะใส่
// p = false positive rate (0.01 = 1%)
func NewBloomFilter(n int, p float64) *BloomFilter {
    // คำนวณขนาดที่เหมาะสม
    m := int(math.Ceil(-float64(n) * math.Log(p) / (math.Log(2) * math.Log(2))))
    k := int(math.Ceil(float64(m) / float64(n) * math.Log(2)))
    
    return &BloomFilter{
        bitset: make([]bool, m),
        k:      k,
        m:      m,
    }
}

func (bf *BloomFilter) Add(key string) {
    bf.mu.Lock()
    defer bf.mu.Unlock()
    
    for i := 0; i < bf.k; i++ {
        pos := bf.hash(key, i) % bf.m
        bf.bitset[pos] = true
    }
}

// Returns: false = definitely not in set, true = possibly in set
func (bf *BloomFilter) Contains(key string) bool {
    bf.mu.RLock()
    defer bf.mu.RUnlock()
    
    for i := 0; i < bf.k; i++ {
        pos := bf.hash(key, i) % bf.m
        if !bf.bitset[pos] {
            return false // Definitely not in set
        }
    }
    
    return true // Possibly in set
}

func (bf *BloomFilter) hash(key string, seed int) int {
    h1 := fnv.New64a()
    h2 := fnv.New64()
    
    h1.Write([]byte(key))
    h2.Write([]byte(key))
    
    // Double hashing
    return int((h1.Sum64() + uint64(seed)*h2.Sum64()) % uint64(bf.m))
}

// Bloom Filter กับ Cache (ป้องกัน Cache Penetration)
type BloomFilteredCache struct {
    bloom  *BloomFilter
    cache  *redis.Client
    db     *sql.DB
}

func (c *BloomFilteredCache) Get(ctx context.Context, key string) (interface{}, error) {
    // ตรวจสอบ bloom filter ก่อน
    if !c.bloom.Contains(key) {
        // Key ไม่มีอยู่จริงในระบบ - ป้องกัน DB hit
        return nil, ErrNotFound
    }
    
    // ลอง cache
    if val, err := c.cache.Get(ctx, key).Bytes(); err == nil {
        var result interface{}
        json.Unmarshal(val, &result)
        return result, nil
    }
    
    // Fallback ไป DB
    result, err := c.loadFromDB(ctx, key)
    if err != nil {
        return nil, err
    }
    
    // Cache ผลลัพธ์
    data, _ := json.Marshal(result)
    c.cache.Set(ctx, key, data, 10*time.Minute)
    
    return result, nil
}

// Redis-based Bloom Filter (สำหรับ distributed systems)
type RedisBloomFilter struct {
    client *redis.Client
    key    string
}

func (rbf *RedisBloomFilter) Add(ctx context.Context, value string) error {
    return rbf.client.Do(ctx, "BF.ADD", rbf.key, value).Err()
}

func (rbf *RedisBloomFilter) Contains(ctx context.Context, value string) (bool, error) {
    result, err := rbf.client.Do(ctx, "BF.EXISTS", rbf.key, value).Bool()
    return result, err
}
```

---

## 5. Cache-aside Pattern (Lazy Loading)

```go
// cache/cache_aside.go
package cache

import (
    "context"
    "time"
)

type CacheAside[T any] struct {
    cache    *redis.Client
    loader   func(ctx context.Context, key string) (T, error)
    ttl      time.Duration
    prefix   string
}

func NewCacheAside[T any](
    cache *redis.Client,
    loader func(ctx context.Context, key string) (T, error),
    ttl time.Duration,
    prefix string,
) *CacheAside[T] {
    return &CacheAside[T]{
        cache:  cache,
        loader: loader,
        ttl:    ttl,
        prefix: prefix,
    }
}

func (c *CacheAside[T]) Get(ctx context.Context, key string) (T, error) {
    cacheKey := c.prefix + ":" + key
    
    // ลอง cache
    data, err := c.cache.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var result T
        if err := json.Unmarshal(data, &result); err == nil {
            return result, nil
        }
    }
    
    // Load จาก source
    value, err := c.loader(ctx, key)
    if err != nil {
        var zero T
        return zero, err
    }
    
    // Cache ผลลัพธ์
    if serialized, err := json.Marshal(value); err == nil {
        c.cache.Set(ctx, cacheKey, serialized, c.ttl)
    }
    
    return value, nil
}

func (c *CacheAside[T]) Invalidate(ctx context.Context, key string) error {
    return c.cache.Del(ctx, c.prefix+":"+key).Err()
}

// ใช้งาน
type UserRepository struct {
    db    *sql.DB
    cache *CacheAside[*User]
}

func NewUserRepository(db *sql.DB, redis *redis.Client) *UserRepository {
    loader := func(ctx context.Context, key string) (*User, error) {
        var user User
        err := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", key).
            Scan(&user.ID, &user.Name, &user.Email)
        return &user, err
    }
    
    return &UserRepository{
        db: db,
        cache: NewCacheAside(redis, loader, 5*time.Minute, "user"),
    }
}

func (r *UserRepository) GetByID(ctx context.Context, id string) (*User, error) {
    return r.cache.Get(ctx, id)
}

func (r *UserRepository) Update(ctx context.Context, user *User) error {
    _, err := r.db.ExecContext(ctx,
        "UPDATE users SET name = $1, email = $2 WHERE id = $3",
        user.Name, user.Email, user.ID,
    )
    if err != nil {
        return err
    }
    
    // Invalidate cache
    return r.cache.Invalidate(ctx, user.ID)
}
```

---

## 6. CDN Integration

```go
// cdn/cloudfront.go
package cdn

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "fmt"
    "net/url"
    "time"
)

type CloudFrontSigner struct {
    keyID      string
    privateKey []byte
}

// สร้าง signed URL สำหรับ private content
func (s *CloudFrontSigner) SignURL(rawURL string, expiry time.Time) (string, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return "", err
    }
    
    // สร้าง policy
    policy := fmt.Sprintf(`{
        "Statement": [{
            "Resource": "%s",
            "Condition": {
                "DateLessThan": {"AWS:EpochTime": %d}
            }
        }]
    }`, rawURL, expiry.Unix())
    
    // Sign policy
    encodedPolicy := base64.URLEncoding.EncodeToString([]byte(policy))
    
    mac := hmac.New(sha256.New, s.privateKey)
    mac.Write([]byte(encodedPolicy))
    signature := base64.URLEncoding.EncodeToString(mac.Sum(nil))
    
    // เพิ่ม query parameters
    q := u.Query()
    q.Set("Policy", encodedPolicy)
    q.Set("Signature", signature)
    q.Set("Key-Pair-Id", s.keyID)
    u.RawQuery = q.Encode()
    
    return u.String(), nil
}

// Cache-Control headers สำหรับ CDN
func SetCacheHeaders(w http.ResponseWriter, r *http.Request, maxAge time.Duration, private bool) {
    if private {
        w.Header().Set("Cache-Control", "private, no-cache")
        return
    }
    
    cacheControl := fmt.Sprintf("public, max-age=%d, s-maxage=%d",
        int(maxAge.Seconds()),
        int(maxAge.Seconds()*2), // CDN cache 2x นาน
    )
    
    w.Header().Set("Cache-Control", cacheControl)
    w.Header().Set("Vary", "Accept-Encoding, Accept-Language")
    
    // ETag สำหรับ conditional requests
    etag := generateETag(r.URL.Path)
    w.Header().Set("ETag", etag)
    
    // Check If-None-Match
    if r.Header.Get("If-None-Match") == etag {
        w.WriteHeader(http.StatusNotModified)
        return
    }
}

// Cache Invalidation ผ่าน CloudFront API
func InvalidateCDNPath(distributionID, path string) error {
    cfg, _ := config.LoadDefaultConfig(context.Background())
    client := cloudfront.NewFromConfig(cfg)
    
    _, err := client.CreateInvalidation(context.Background(), &cloudfront.CreateInvalidationInput{
        DistributionId: &distributionID,
        InvalidationBatch: &types.InvalidationBatch{
            CallerReference: aws.String(fmt.Sprintf("%d", time.Now().UnixNano())),
            Paths: &types.Paths{
                Quantity: aws.Int32(1),
                Items:    []string{path},
            },
        },
    })
    
    return err
}
```

---

## 7. Edge Caching กับ Stale-while-revalidate

```go
// cache/edge.go
package cache

import (
    "context"
    "sync"
    "time"
)

type EdgeCacheEntry struct {
    Data       interface{}
    ExpiresAt  time.Time
    StaleUntil time.Time
    Updating   bool
}

type EdgeCache struct {
    mu      sync.RWMutex
    entries map[string]*EdgeCacheEntry
    fetcher func(ctx context.Context, key string) (interface{}, error)
    ttl     time.Duration
    staleTTL time.Duration
}

func NewEdgeCache(fetcher func(ctx context.Context, key string) (interface{}, error)) *EdgeCache {
    return &EdgeCache{
        entries:  make(map[string]*EdgeCacheEntry),
        fetcher:  fetcher,
        ttl:      5 * time.Minute,
        staleTTL: 10 * time.Minute,
    }
}

// Stale-while-revalidate: คืน stale data ทันที แล้ว refresh background
func (c *EdgeCache) Get(ctx context.Context, key string) (interface{}, error) {
    c.mu.RLock()
    entry, ok := c.entries[key]
    c.mu.RUnlock()
    
    now := time.Now()
    
    if ok {
        if now.Before(entry.ExpiresAt) {
            // Fresh cache hit
            return entry.Data, nil
        }
        
        if now.Before(entry.StaleUntil) {
            // Stale but within grace period
            if !entry.Updating {
                // Revalidate ใน background
                go c.revalidate(context.Background(), key)
            }
            return entry.Data, nil // Return stale data ทันที
        }
    }
    
    // Cache miss หรือ expired
    return c.fetchAndStore(ctx, key)
}

func (c *EdgeCache) revalidate(ctx context.Context, key string) {
    c.mu.Lock()
    if entry, ok := c.entries[key]; ok {
        entry.Updating = true
    }
    c.mu.Unlock()
    
    defer func() {
        c.mu.Lock()
        if entry, ok := c.entries[key]; ok {
            entry.Updating = false
        }
        c.mu.Unlock()
    }()
    
    c.fetchAndStore(ctx, key)
}

func (c *EdgeCache) fetchAndStore(ctx context.Context, key string) (interface{}, error) {
    data, err := c.fetcher(ctx, key)
    if err != nil {
        return nil, err
    }
    
    now := time.Now()
    
    c.mu.Lock()
    c.entries[key] = &EdgeCacheEntry{
        Data:       data,
        ExpiresAt:  now.Add(c.ttl),
        StaleUntil: now.Add(c.staleTTL),
    }
    c.mu.Unlock()
    
    return data, nil
}
```

---

## 8. Distributed Cache Invalidation

```go
// cache/invalidation.go
package cache

import (
    "context"
    "encoding/json"
    "log"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type InvalidationMessage struct {
    Keys      []string  `json:"keys"`
    Timestamp time.Time `json:"timestamp"`
    Source    string    `json:"source"`
}

// Redis Pub/Sub สำหรับ cache invalidation ใน cluster
type CacheInvalidator struct {
    redis   *redis.Client
    channel string
    l1      *L1Cache
    nodeID  string
}

func NewCacheInvalidator(redis *redis.Client, l1 *L1Cache, nodeID string) *CacheInvalidator {
    ci := &CacheInvalidator{
        redis:   redis,
        channel: "cache:invalidation",
        l1:      l1,
        nodeID:  nodeID,
    }
    
    go ci.listen()
    return ci
}

func (ci *CacheInvalidator) Invalidate(ctx context.Context, keys ...string) error {
    msg := InvalidationMessage{
        Keys:      keys,
        Timestamp: time.Now(),
        Source:    ci.nodeID,
    }
    
    data, _ := json.Marshal(msg)
    
    // Publish ไปทุก nodes
    return ci.redis.Publish(ctx, ci.channel, data).Err()
}

func (ci *CacheInvalidator) listen() {
    ctx := context.Background()
    
    sub := ci.redis.Subscribe(ctx, ci.channel)
    defer sub.Close()
    
    for msg := range sub.Channel() {
        var invalidation InvalidationMessage
        if err := json.Unmarshal([]byte(msg.Payload), &invalidation); err != nil {
            continue
        }
        
        // ไม่ต้อง invalidate ของตัวเอง (ส่งไปแล้ว)
        if invalidation.Source == ci.nodeID {
            continue
        }
        
        // Invalidate L1 cache บน node นี้
        for _, key := range invalidation.Keys {
            ci.l1.Delete(key)
            log.Printf("Invalidated key: %s (from node: %s)", key, invalidation.Source)
        }
    }
}
```

---

## Workshop: Complete Caching Layer

```go
// workshop/cache_layer.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type Product struct {
    ID       string  `json:"id"`
    Name     string  `json:"name"`
    Price    float64 `json:"price"`
    Category string  `json:"category"`
}

type CachedProductRepo struct {
    db          *sql.DB
    multiCache  *MultiLevelCache
    bloom       *BloomFilter
    invalidator *CacheInvalidator
    sfGroup     singleflight.Group
}

func NewCachedProductRepo(db *sql.DB, redisClient *redis.Client) *CachedProductRepo {
    l1 := NewL1Cache(10000)
    l2 := NewL2Cache(redisClient, "product")
    multiCache := NewMultiLevelCache(l1, l2)
    
    // Bloom filter สำหรับ 1M products, 1% false positive rate
    bloom := NewBloomFilter(1000000, 0.01)
    
    invalidator := NewCacheInvalidator(redisClient, l1, "node-1")
    
    return &CachedProductRepo{
        db:          db,
        multiCache:  multiCache,
        bloom:       bloom,
        invalidator: invalidator,
    }
}

func (r *CachedProductRepo) GetProduct(ctx context.Context, id string) (*Product, error) {
    // ตรวจสอบ bloom filter ก่อน
    if !r.bloom.Contains(id) {
        return nil, fmt.Errorf("product not found")
    }
    
    cacheKey := "product:" + id
    
    // ลอง cache ด้วย singleflight
    result, err, _ := r.sfGroup.Do(cacheKey, func() (interface{}, error) {
        var product Product
        
        if err := r.multiCache.Get(ctx, cacheKey, &product); err == nil {
            return &product, nil
        }
        
        // Load จาก DB
        if err := r.db.QueryRowContext(ctx, "SELECT id, name, price, category FROM products WHERE id = $1", id).
            Scan(&product.ID, &product.Name, &product.Price, &product.Category); err != nil {
            return nil, err
        }
        
        r.multiCache.Set(ctx, cacheKey, &product)
        
        return &product, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    return result.(*Product), nil
}

func (r *CachedProductRepo) UpdateProduct(ctx context.Context, product *Product) error {
    _, err := r.db.ExecContext(ctx,
        "UPDATE products SET name = $1, price = $2, category = $3 WHERE id = $4",
        product.Name, product.Price, product.Category, product.ID,
    )
    if err != nil {
        return err
    }
    
    // Invalidate cache บนทุก nodes
    cacheKey := "product:" + product.ID
    return r.invalidator.Invalidate(ctx, cacheKey)
}

// HTTP Handler
func ProductHandler(repo *CachedProductRepo) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.URL.Query().Get("id")
        
        product, err := repo.GetProduct(r.Context(), id)
        if err != nil {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
        
        // Set cache headers
        SetCacheHeaders(w, r, 5*time.Minute, false)
        
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(product)
    }
}

func main() {
    db, _ := sql.Open("postgres", "postgres://user:pass@localhost/db")
    redisClient := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    
    repo := NewCachedProductRepo(db, redisClient)
    
    http.HandleFunc("/products", ProductHandler(repo))
    
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## สรุป

| Pattern | ป้องกัน | ใช้เมื่อ |
|---------|---------|---------|
| Multi-level | N/A | Always |
| Write-through | Inconsistency | Strong consistency needed |
| Write-back | N/A | Write-heavy, eventual OK |
| Singleflight | Cache stampede | High concurrency |
| Bloom filter | Cache penetration | Many invalid key queries |
| XFetch | Cache stampede | Probabilistic |
| Stale-while-revalidate | Latency spikes | High availability |
| Pub/Sub invalidation | Cache inconsistency | Multi-node setup |

### Performance Tips
1. L1 cache ขนาดเล็กแต่เร็วมาก (in-memory)
2. L2 cache (Redis) รองรับ distributed
3. ตั้ง TTL ให้สั้นกว่าที่คิด (ข้อมูลเปลี่ยนบ่อยกว่าที่คิด)
4. Monitor hit rate (ควร > 80%)
5. ระวัง memory leak ใน L1 cache

---

*จบ Part 71: Advanced Caching*
