# Part 82: Edge Computing with Go

## เป้าหมายของบทเรียน
- Edge computing concepts
- WebAssembly at the edge
- Cloudflare Workers with Go/WASM
- Fastly Compute@Edge
- Edge caching strategies

---

## 1. Edge Computing Concepts

Edge computing คือการประมวลผลข้อมูลใกล้กับ source มากขึ้น แทนที่จะส่งทุกอย่างไป central datacenter

### ประโยชน์
- Latency ต่ำลง
- Bandwidth ลดลง
- Privacy (ข้อมูลไม่ต้องออกนอก region)
- Offline capability

```go
// edge/concepts.go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

// EdgeNode แสดง edge node
type EdgeNode struct {
    ID       string
    Location string
    Lat      float64
    Lon      float64
    Capacity int
}

// EdgeRequest แสดง request ที่ส่งมา
type EdgeRequest struct {
    ClientID string
    ClientLat float64
    ClientLon float64
    Path     string
    Method   string
}

// EdgeRouter เลือก edge node ที่เหมาะสม
type EdgeRouter struct {
    nodes []*EdgeNode
}

// NewEdgeRouter สร้าง router ใหม่
func NewEdgeRouter() *EdgeRouter {
    return &EdgeRouter{
        nodes: []*EdgeNode{
            {ID: "edge-th-bkk", Location: "Bangkok", Lat: 13.756, Lon: 100.502, Capacity: 1000},
            {ID: "edge-sg", Location: "Singapore", Lat: 1.352, Lon: 103.820, Capacity: 2000},
            {ID: "edge-jp-tky", Location: "Tokyo", Lat: 35.689, Lon: 139.692, Capacity: 1500},
            {ID: "edge-us-lax", Location: "Los Angeles", Lat: 34.052, Lon: -118.244, Capacity: 3000},
            {ID: "edge-eu-ams", Location: "Amsterdam", Lat: 52.370, Lon: 4.895, Capacity: 2500},
        },
    }
}

// distance คำนวณ distance ระหว่าง 2 จุด (simplified)
func distance(lat1, lon1, lat2, lon2 float64) float64 {
    dLat := lat2 - lat1
    dLon := lon2 - lon1
    return dLat*dLat + dLon*dLon
}

// Route เลือก edge node ที่ใกล้ที่สุด
func (r *EdgeRouter) Route(req EdgeRequest) *EdgeNode {
    var closest *EdgeNode
    minDist := float64(1<<62)
    
    for _, node := range r.nodes {
        d := distance(req.ClientLat, req.ClientLon, node.Lat, node.Lon)
        if d < minDist {
            minDist = d
            closest = node
        }
    }
    
    return closest
}

// EdgeCache cache ที่ edge
type EdgeCache struct {
    data     map[string]*CacheEntry
    maxItems int
}

// CacheEntry ข้อมูลใน cache
type CacheEntry struct {
    Value     []byte
    ExpiresAt time.Time
    HitCount  int
}

// NewEdgeCache สร้าง cache ใหม่
func NewEdgeCache(maxItems int) *EdgeCache {
    return &EdgeCache{
        data:     make(map[string]*CacheEntry),
        maxItems: maxItems,
    }
}

// Get ดึงข้อมูลจาก cache
func (c *EdgeCache) Get(key string) ([]byte, bool) {
    entry, ok := c.data[key]
    if !ok {
        return nil, false
    }
    if time.Now().After(entry.ExpiresAt) {
        delete(c.data, key)
        return nil, false
    }
    entry.HitCount++
    return entry.Value, true
}

// Set เก็บข้อมูลใน cache
func (c *EdgeCache) Set(key string, value []byte, ttl time.Duration) {
    if len(c.data) >= c.maxItems {
        c.evict()
    }
    c.data[key] = &CacheEntry{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
}

// evict ลบ entry ที่ hit น้อยที่สุด
func (c *EdgeCache) evict() {
    var minKey string
    minHits := int(^uint(0) >> 1)
    
    for k, v := range c.data {
        if v.HitCount < minHits {
            minHits = v.HitCount
            minKey = k
        }
    }
    
    if minKey != "" {
        delete(c.data, minKey)
    }
}

// EdgeFunction function ที่รันที่ edge
type EdgeFunction func(req *EdgeHTTPRequest) *EdgeHTTPResponse

// EdgeHTTPRequest แสดง HTTP request ที่ edge
type EdgeHTTPRequest struct {
    Method  string
    URL     string
    Headers map[string]string
    Body    []byte
}

// EdgeHTTPResponse แสดง HTTP response จาก edge
type EdgeHTTPResponse struct {
    Status  int
    Headers map[string]string
    Body    []byte
}

// EdgeRuntime runtime สำหรับ edge functions
type EdgeRuntime struct {
    cache     *EdgeCache
    functions map[string]EdgeFunction
}

// NewEdgeRuntime สร้าง runtime ใหม่
func NewEdgeRuntime() *EdgeRuntime {
    return &EdgeRuntime{
        cache:     NewEdgeCache(1000),
        functions: make(map[string]EdgeFunction),
    }
}

// Register ลงทะเบียน function
func (rt *EdgeRuntime) Register(path string, fn EdgeFunction) {
    rt.functions[path] = fn
}

// HandleRequest จัดการ request
func (rt *EdgeRuntime) HandleRequest(req *EdgeHTTPRequest) *EdgeHTTPResponse {
    // ตรวจสอบ cache ก่อน
    cacheKey := req.Method + ":" + req.URL
    if req.Method == "GET" {
        if cached, ok := rt.cache.Get(cacheKey); ok {
            return &EdgeHTTPResponse{
                Status:  200,
                Headers: map[string]string{"X-Cache": "HIT", "Content-Type": "application/json"},
                Body:    cached,
            }
        }
    }
    
    // หา function handler
    fn, ok := rt.functions[req.URL]
    if !ok {
        return &EdgeHTTPResponse{
            Status: 404,
            Body:   []byte(`{"error": "Not Found"}`),
        }
    }
    
    resp := fn(req)
    
    // Cache GET responses
    if req.Method == "GET" && resp.Status == 200 {
        rt.cache.Set(cacheKey, resp.Body, 60*time.Second)
    }
    
    return resp
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    // Edge routing demo
    router := NewEdgeRouter()
    
    clients := []EdgeRequest{
        {ClientID: "th-user", ClientLat: 13.756, ClientLon: 100.502, Path: "/api/data"},
        {ClientID: "jp-user", ClientLat: 35.689, ClientLon: 139.692, Path: "/api/data"},
        {ClientID: "us-user", ClientLat: 37.774, ClientLon: -122.419, Path: "/api/data"},
    }
    
    fmt.Println("=== Edge Routing Demo ===")
    for _, client := range clients {
        node := router.Route(client)
        fmt.Printf("Client: %s -> Edge Node: %s (%s)\n", 
            client.ClientID, node.ID, node.Location)
    }
    
    // Edge Runtime demo
    fmt.Println("\n=== Edge Function Runtime Demo ===")
    runtime := NewEdgeRuntime()
    
    // Register edge functions
    runtime.Register("/api/hello", func(req *EdgeHTTPRequest) *EdgeHTTPResponse {
        return &EdgeHTTPResponse{
            Status:  200,
            Headers: map[string]string{"Content-Type": "application/json"},
            Body:    []byte(`{"message": "Hello from Edge!"}`),
        }
    })
    
    runtime.Register("/api/geo", func(req *EdgeHTTPRequest) *EdgeHTTPResponse {
        country := req.Headers["CF-IPCountry"]
        if country == "" {
            country = "TH"
        }
        body := fmt.Sprintf(`{"country": "%s", "edge": "Bangkok"}`, country)
        return &EdgeHTTPResponse{
            Status:  200,
            Headers: map[string]string{"Content-Type": "application/json"},
            Body:    []byte(body),
        }
    })
    
    // Handle requests
    requests := []*EdgeHTTPRequest{
        {Method: "GET", URL: "/api/hello", Headers: map[string]string{}},
        {Method: "GET", URL: "/api/hello", Headers: map[string]string{}}, // Cache hit
        {Method: "GET", URL: "/api/geo", Headers: map[string]string{"CF-IPCountry": "TH"}},
        {Method: "GET", URL: "/api/not-found"},
    }
    
    for _, req := range requests {
        resp := runtime.HandleRequest(req)
        cacheStatus := resp.Headers["X-Cache"]
        if cacheStatus == "" {
            cacheStatus = "MISS"
        }
        fmt.Printf("%s %s -> %d [Cache: %s]: %s\n",
            req.Method, req.URL, resp.Status, cacheStatus, string(resp.Body))
    }
}
```

---

## 2. WebAssembly at the Edge

```go
// wasm/edge.go
package main

import (
    "fmt"
    "time"
)

// WASMModule แสดง WebAssembly module
type WASMModule struct {
    Name     string
    Size     int64
    Exports  []string
    Memory   int // pages (64KB each)
}

// WASMRuntime runtime สำหรับรัน WASM ที่ edge
type WASMRuntime struct {
    modules  map[string]*WASMModule
    sandbox  bool
}

// NewWASMRuntime สร้าง runtime ใหม่
func NewWASMRuntime(sandbox bool) *WASMRuntime {
    return &WASMRuntime{
        modules: make(map[string]*WASMModule),
        sandbox: sandbox,
    }
}

// LoadModule โหลด WASM module
func (rt *WASMRuntime) LoadModule(name string, wasmBytes []byte) error {
    // จำลองการ compile WASM
    start := time.Now()
    time.Sleep(5 * time.Millisecond) // จำลอง compilation time
    
    module := &WASMModule{
        Name:    name,
        Size:    int64(len(wasmBytes)),
        Exports: []string{"handle_request", "process_data"},
        Memory:  4, // 256KB
    }
    
    rt.modules[name] = module
    fmt.Printf("[WASM] Loaded module '%s' in %v\n", name, time.Since(start))
    return nil
}

// Execute รัน exported function
func (rt *WASMRuntime) Execute(moduleName, funcName string, args ...interface{}) (interface{}, error) {
    module, ok := rt.modules[moduleName]
    if !ok {
        return nil, fmt.Errorf("module not found: %s", moduleName)
    }
    
    // ตรวจสอบ export
    hasExport := false
    for _, exp := range module.Exports {
        if exp == funcName {
            hasExport = true
            break
        }
    }
    
    if !hasExport {
        return nil, fmt.Errorf("function not exported: %s", funcName)
    }
    
    // จำลองการรัน WASM function
    start := time.Now()
    time.Sleep(1 * time.Millisecond)
    
    result := map[string]interface{}{
        "success":   true,
        "duration":  time.Since(start).String(),
        "module":    moduleName,
        "function":  funcName,
    }
    
    return result, nil
}

// EdgeWorker จำลอง Cloudflare Worker
type EdgeWorker struct {
    name    string
    script  string
    runtime *WASMRuntime
}

// NewEdgeWorker สร้าง worker ใหม่
func NewEdgeWorker(name, script string) *EdgeWorker {
    return &EdgeWorker{
        name:    name,
        script:  script,
        runtime: NewWASMRuntime(true),
    }
}

// HandleFetch จัดการ fetch request
func (w *EdgeWorker) HandleFetch(url, method string, headers map[string]string) map[string]interface{} {
    start := time.Now()
    
    // Logic ที่รันที่ edge
    response := map[string]interface{}{
        "status":    200,
        "worker":    w.name,
        "url":       url,
        "method":    method,
        "duration":  time.Since(start).String(),
        "timestamp": time.Now().Unix(),
    }
    
    // Transform response ตาม headers
    if headers["Accept-Language"] == "th-TH" {
        response["message"] = "สวัสดีจาก Edge!"
    } else {
        response["message"] = "Hello from Edge!"
    }
    
    return response
}

// EdgeCachingStrategies กลยุทธ์การ cache ที่ edge
type CachingStrategy string

const (
    CacheFirst    CachingStrategy = "cache-first"
    NetworkFirst  CachingStrategy = "network-first"
    StaleWhileRevalidate CachingStrategy = "stale-while-revalidate"
    CacheOnly     CachingStrategy = "cache-only"
)

// CachePolicy นโยบาย cache
type CachePolicy struct {
    Strategy    CachingStrategy
    TTL         time.Duration
    MaxAge      time.Duration
    StaleAge    time.Duration
    VaryHeaders []string
}

// CacheKeyGenerator สร้าง cache key
type CacheKeyGenerator struct {
    includeQuery  bool
    varyHeaders   []string
}

// GenerateKey สร้าง key สำหรับ cache
func (g *CacheKeyGenerator) GenerateKey(url, method string, headers map[string]string) string {
    key := method + ":" + url
    
    for _, header := range g.varyHeaders {
        if val, ok := headers[header]; ok {
            key += ":" + header + "=" + val
        }
    }
    
    return key
}

func main() {
    // WASM Runtime demo
    fmt.Println("=== WASM at Edge Demo ===\n")
    
    runtime := NewWASMRuntime(true)
    
    // โหลด modules
    runtime.LoadModule("image-processor", make([]byte, 1024*100)) // 100KB WASM
    runtime.LoadModule("auth-validator", make([]byte, 1024*50))   // 50KB WASM
    
    // รัน functions
    result, err := runtime.Execute("image-processor", "handle_request", "image.jpg")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Result: %v\n", result)
    }
    
    // Edge Worker demo
    fmt.Println("\n=== Cloudflare Worker Demo ===\n")
    
    worker := NewEdgeWorker("my-edge-worker", `
        addEventListener('fetch', event => {
            event.respondWith(handleRequest(event.request))
        })
    `)
    
    requests := []struct {
        url     string
        method  string
        headers map[string]string
    }{
        {"/api/data", "GET", map[string]string{"Accept-Language": "th-TH"}},
        {"/api/data", "GET", map[string]string{"Accept-Language": "en-US"}},
        {"/api/health", "GET", map[string]string{}},
    }
    
    for _, req := range requests {
        resp := worker.HandleFetch(req.url, req.method, req.headers)
        fmt.Printf("Request: %s %s -> %v\n", req.method, req.url, resp["message"])
    }
    
    // Cache strategies
    fmt.Println("\n=== Cache Strategies ===")
    
    policies := map[string]CachePolicy{
        "static-assets": {
            Strategy: CacheFirst,
            TTL:      24 * time.Hour,
            MaxAge:   365 * 24 * time.Hour,
        },
        "api-data": {
            Strategy:  StaleWhileRevalidate,
            TTL:       60 * time.Second,
            StaleAge:  5 * time.Minute,
            VaryHeaders: []string{"Accept-Language", "Authorization"},
        },
        "user-data": {
            Strategy: NetworkFirst,
            TTL:      10 * time.Second,
            VaryHeaders: []string{"Authorization"},
        },
    }
    
    gen := &CacheKeyGenerator{
        includeQuery: true,
        varyHeaders:  []string{"Accept-Language"},
    }
    
    for name, policy := range policies {
        key := gen.GenerateKey("/api/"+name, "GET", map[string]string{"Accept-Language": "th-TH"})
        fmt.Printf("Policy: %-15s Strategy: %-25s TTL: %v\n  CacheKey: %s\n",
            name, policy.Strategy, policy.TTL, key)
    }
}
```

---

## 3. Edge Caching Strategies

```go
// caching/edge.go
package main

import (
    "fmt"
    "sync"
    "time"
)

// CacheLayer แสดง layer ของ cache
type CacheLayer struct {
    Name     string
    TTL      time.Duration
    MaxItems int
    data     map[string]*CachedItem
    mu       sync.RWMutex
    hits     int64
    misses   int64
}

// CachedItem item ใน cache
type CachedItem struct {
    Value     []byte
    ExpiresAt time.Time
    Tags      []string
}

// NewCacheLayer สร้าง layer ใหม่
func NewCacheLayer(name string, ttl time.Duration, maxItems int) *CacheLayer {
    return &CacheLayer{
        Name:     name,
        TTL:      ttl,
        MaxItems: maxItems,
        data:     make(map[string]*CachedItem),
    }
}

// Get ดึงข้อมูล
func (cl *CacheLayer) Get(key string) ([]byte, bool) {
    cl.mu.RLock()
    item, ok := cl.data[key]
    cl.mu.RUnlock()
    
    if !ok || time.Now().After(item.ExpiresAt) {
        cl.misses++
        if ok {
            cl.mu.Lock()
            delete(cl.data, key)
            cl.mu.Unlock()
        }
        return nil, false
    }
    
    cl.hits++
    return item.Value, true
}

// Set เก็บข้อมูล
func (cl *CacheLayer) Set(key string, value []byte, tags ...string) {
    cl.mu.Lock()
    defer cl.mu.Unlock()
    
    cl.data[key] = &CachedItem{
        Value:     value,
        ExpiresAt: time.Now().Add(cl.TTL),
        Tags:      tags,
    }
}

// InvalidateByTag ลบ cache items ตาม tag
func (cl *CacheLayer) InvalidateByTag(tag string) int {
    cl.mu.Lock()
    defer cl.mu.Unlock()
    
    count := 0
    for key, item := range cl.data {
        for _, t := range item.Tags {
            if t == tag {
                delete(cl.data, key)
                count++
                break
            }
        }
    }
    return count
}

// HitRate คืน hit rate
func (cl *CacheLayer) HitRate() float64 {
    total := cl.hits + cl.misses
    if total == 0 {
        return 0
    }
    return float64(cl.hits) / float64(total)
}

// MultiLayerCache cache หลาย layers
type MultiLayerCache struct {
    layers []*CacheLayer
}

// NewMultiLayerCache สร้าง cache ใหม่
func NewMultiLayerCache(layers ...*CacheLayer) *MultiLayerCache {
    return &MultiLayerCache{layers: layers}
}

// Get ดึงข้อมูลจาก layer ที่ใกล้ที่สุด
func (mlc *MultiLayerCache) Get(key string) ([]byte, string, bool) {
    for _, layer := range mlc.layers {
        if value, ok := layer.Get(key); ok {
            return value, layer.Name, true
        }
    }
    return nil, "", false
}

// Set เก็บข้อมูลใน layers ที่กำหนด
func (mlc *MultiLayerCache) Set(key string, value []byte, layers ...string) {
    for _, layer := range mlc.layers {
        if len(layers) == 0 {
            layer.Set(key, value)
            continue
        }
        for _, l := range layers {
            if l == layer.Name {
                layer.Set(key, value)
                break
            }
        }
    }
}

func main() {
    // สร้าง multi-layer cache
    l1 := NewCacheLayer("L1-Memory", 30*time.Second, 100)      // Edge memory
    l2 := NewCacheLayer("L2-Disk", 5*time.Minute, 1000)         // Edge disk
    l3 := NewCacheLayer("L3-CDN", 1*time.Hour, 100000)           // CDN/Origin
    
    cache := NewMultiLayerCache(l1, l2, l3)
    
    fmt.Println("=== Multi-Layer Edge Cache Demo ===\n")
    
    // เก็บข้อมูล
    cache.Set("/api/products", []byte(`[{"id": 1, "name": "สินค้า A"}]`))
    cache.Set("/static/logo.png", []byte("... binary data ..."), "L3-CDN")
    
    // ดึงข้อมูล
    keys := []string{"/api/products", "/static/logo.png", "/api/not-found"}
    
    for _, key := range keys {
        value, layer, ok := cache.Get(key)
        if ok {
            fmt.Printf("GET %s -> FOUND in %s: %s\n", key, layer, string(value[:min(len(value), 50)]))
        } else {
            fmt.Printf("GET %s -> MISS (all layers)\n", key)
        }
    }
    
    // แสดง cache statistics
    fmt.Println("\n=== Cache Statistics ===")
    for _, layer := range []*CacheLayer{l1, l2, l3} {
        fmt.Printf("Layer: %-10s Hit Rate: %.0f%%\n",
            layer.Name, layer.HitRate()*100)
    }
    
    // Tag-based invalidation
    l1.Set("user:1:profile", []byte(`{"name": "สมชาย"}`), "user:1")
    l1.Set("user:1:orders", []byte(`[{"id": "ord-1"}]`), "user:1")
    l1.Set("product:1", []byte(`{"name": "สินค้า"}`), "products")
    
    count := l1.InvalidateByTag("user:1")
    fmt.Printf("\nInvalidated %d items with tag 'user:1'\n", count)
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Edge Computing** - แนวคิดและประโยชน์ของการประมวลผลที่ edge
2. **WASM at Edge** - การรัน WebAssembly ที่ edge nodes
3. **Edge Functions** - Cloudflare Workers pattern
4. **Caching Strategies** - Cache-first, network-first, stale-while-revalidate
5. **Multi-layer Cache** - L1/L2/L3 cache hierarchy

### Key Takeaways

- **Edge = Low Latency**: ลด round-trip time ด้วยการประมวลผลใกล้ user
- **Cache at Edge**: Static assets และ read-heavy data ควร cache ที่ edge
- **Tag-based Invalidation**: ใช้ tags เพื่อ invalidate related cache entries
- **WASM Security**: WASM รันใน sandbox ทำให้ปลอดภัยกว่าการรัน native code
