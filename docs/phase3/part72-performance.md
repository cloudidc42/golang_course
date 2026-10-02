# Part 72: Performance Optimization

## เป้าหมายการเรียนรู้
- Performance Profiling Methodology
- Go Escape Analysis
- Zero-copy Techniques
- Memory Alignment
- CPU Cache Optimization
- Lock-free Data Structures
- Custom Allocators
- Benchmarking

---

## 1. Performance Profiling Methodology

### Profiling Tools ใน Go

```go
// profiling/main.go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // ลงทะเบียน /debug/pprof endpoints
    "os"
    "runtime/pprof"
    "time"
)

func main() {
    // HTTP profiling endpoint
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // CPU Profile
    f, err := os.Create("cpu.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // Memory Profile
    memFile, _ := os.Create("mem.prof")
    defer func() {
        pprof.WriteHeapProfile(memFile)
        memFile.Close()
    }()
    
    // Run workload
    runWorkload()
}

// Flame graph: go tool pprof -http=:8080 cpu.prof
// Memory:      go tool pprof -http=:8080 mem.prof
// Goroutines:  curl http://localhost:6060/debug/pprof/goroutine?debug=1
// Trace:       curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
//              go tool trace trace.out
```

### Benchmarks

```go
// benchmark_test.go
package main

import (
    "fmt"
    "strings"
    "testing"
)

// เปรียบเทียบ string concatenation methods
func BenchmarkStringConcat(b *testing.B) {
    for n := 0; n < b.N; n++ {
        s := ""
        for i := 0; i < 100; i++ {
            s += fmt.Sprintf("%d", i)
        }
        _ = s
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    for n := 0; n < b.N; n++ {
        var sb strings.Builder
        for i := 0; i < 100; i++ {
            fmt.Fprintf(&sb, "%d", i)
        }
        _ = sb.String()
    }
}

func BenchmarkBytesJoin(b *testing.B) {
    for n := 0; n < b.N; n++ {
        parts := make([]string, 100)
        for i := 0; i < 100; i++ {
            parts[i] = fmt.Sprintf("%d", i)
        }
        _ = strings.Join(parts, "")
    }
}

// Run: go test -bench=. -benchmem -count=5
// Output:
// BenchmarkStringConcat-8    100000    15234 ns/op    5360 B/op    99 allocs/op
// BenchmarkStringBuilder-8  1000000    1523 ns/op     536 B/op     2 allocs/op
// BenchmarkBytesJoin-8       500000    2134 ns/op     536 B/op     3 allocs/op
```

---

## 2. Escape Analysis

Go compiler ตัดสินใจว่า variable จะอยู่บน stack หรือ heap

```go
// escape/analysis.go
package main

// go build -gcflags="-m -m" ./... เพื่อดู escape analysis

// ✅ Stack allocation - เร็วกว่า, ไม่ GC pressure
func stackAlloc() int {
    x := 42 // อยู่บน stack
    return x
}

// ❌ Heap allocation - GC pressure
func heapAlloc() *int {
    x := 42
    return &x // x ต้อง escape ไป heap เพราะ return pointer
}

// ✅ ป้องกัน escape ด้วย sync.Pool
type Buffer struct {
    data []byte
}

var bufferPool = sync.Pool{
    New: func() interface{} {
        return &Buffer{data: make([]byte, 0, 4096)}
    },
}

func processWithPool(input []byte) []byte {
    buf := bufferPool.Get().(*Buffer)
    defer func() {
        buf.data = buf.data[:0] // Reset
        bufferPool.Put(buf)
    }()
    
    buf.data = append(buf.data, input...)
    result := make([]byte, len(buf.data))
    copy(result, buf.data)
    
    return result
}

// Interface causing escape
type Animal interface {
    Sound() string
}

type Dog struct {
    Name string
}

func (d Dog) Sound() string { return "Woof" }

func makeSound(a Animal) string {
    return a.Sound() // a อาจ escape เพราะเป็น interface
}

// ✅ Better: ใช้ concrete type เมื่อทำได้
func makeDogSound(d Dog) string {
    return d.Sound() // ไม่ escape
}
```

### Reducing Allocations

```go
// reduce_allocs.go
package main

import (
    "encoding/json"
    "fmt"
    "sync"
)

// ❌ แบบมี allocations เยอะ
func badJSONEncoder(data interface{}) ([]byte, error) {
    return json.Marshal(data) // สร้าง []byte ใหม่ทุกครั้ง
}

// ✅ แบบ reuse buffer
var encoderPool = sync.Pool{
    New: func() interface{} {
        return &jsonEncoder{
            buf: make([]byte, 0, 1024),
        }
    },
}

type jsonEncoder struct {
    buf []byte
}

func goodJSONEncoder(data interface{}) ([]byte, error) {
    enc := encoderPool.Get().(*jsonEncoder)
    defer func() {
        enc.buf = enc.buf[:0]
        encoderPool.Put(enc)
    }()
    
    b, err := json.Marshal(data)
    if err != nil {
        return nil, err
    }
    
    enc.buf = append(enc.buf, b...)
    
    result := make([]byte, len(enc.buf))
    copy(result, enc.buf)
    
    return result, nil
}

// Pre-allocate slices
func badSlice() []int {
    var s []int
    for i := 0; i < 1000; i++ {
        s = append(s, i) // Re-allocate หลายครั้ง
    }
    return s
}

func goodSlice() []int {
    s := make([]int, 0, 1000) // Pre-allocate
    for i := 0; i < 1000; i++ {
        s = append(s, i) // ไม่ต้อง re-allocate
    }
    return s
}

// Object pooling สำหรับ Request/Response
type Request struct {
    Body   []byte
    Header map[string]string
    ID     string
}

var requestPool = sync.Pool{
    New: func() interface{} {
        return &Request{
            Header: make(map[string]string),
            Body:   make([]byte, 0, 512),
        }
    },
}

func processRequest(rawData []byte) {
    req := requestPool.Get().(*Request)
    defer func() {
        req.Body = req.Body[:0]
        for k := range req.Header {
            delete(req.Header, k)
        }
        req.ID = ""
        requestPool.Put(req)
    }()
    
    // ใช้งาน req...
    req.Body = append(req.Body, rawData...)
}
```

---

## 3. Zero-copy Techniques

```go
// zerocopy/reader.go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net"
    "os"
    "syscall"
    "unsafe"
)

// Zero-copy string to bytes (ระวัง: ห้ามแก้ไข bytes ที่ได้!)
func unsafeStringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(&struct {
        string
        Cap int
    }{s, len(s)}))
}

// Zero-copy bytes to string
func unsafeBytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

// ใช้งานตัวอย่าง
func processStrings(data []string) int {
    total := 0
    for _, s := range data {
        b := unsafeStringToBytes(s) // ไม่ copy
        total += len(b)
    }
    return total
}

// sendfile - zero-copy file transfer
func serveFile(conn net.Conn, filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    
    stat, err := f.Stat()
    if err != nil {
        return err
    }
    
    // ใช้ sendfile syscall ถ้าเป็น TCP connection
    if tcpConn, ok := conn.(*net.TCPConn); ok {
        rawConn, err := tcpConn.SyscallConn()
        if err == nil {
            var sendErr error
            rawConn.Control(func(fd uintptr) {
                // sendfile ไม่ copy data ผ่าน userspace
                _, sendErr = syscall.Sendfile(
                    int(fd),
                    int(f.Fd()),
                    nil,
                    int(stat.Size()),
                )
            })
            if sendErr == nil {
                return nil
            }
        }
    }
    
    // Fallback: copy ปกติ
    _, err = io.Copy(conn, f)
    return err
}

// io.Reader ที่ไม่ copy
type ZeroCopyReader struct {
    data   []byte
    offset int
}

func NewZeroCopyReader(data []byte) *ZeroCopyReader {
    return &ZeroCopyReader{data: data}
}

func (r *ZeroCopyReader) Read(p []byte) (n int, err error) {
    if r.offset >= len(r.data) {
        return 0, io.EOF
    }
    
    n = copy(p, r.data[r.offset:])
    r.offset += n
    return n, nil
}

// Bytes ที่ไม่ copy ด้วย bytes.NewBuffer
func zeroCopyBuffer(data []byte) *bytes.Buffer {
    return bytes.NewBuffer(data) // ไม่ copy, share underlying array
}

// Memory mapping
func mmapFile(filename string) ([]byte, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    
    stat, err := f.Stat()
    if err != nil {
        return nil, err
    }
    
    data, err := syscall.Mmap(
        int(f.Fd()),
        0,
        int(stat.Size()),
        syscall.PROT_READ,
        syscall.MAP_SHARED,
    )
    if err != nil {
        return nil, fmt.Errorf("mmap failed: %w", err)
    }
    
    return data, nil
}
```

---

## 4. Memory Alignment

```go
// alignment/structs.go
package main

import (
    "fmt"
    "unsafe"
)

// ❌ Poor alignment - มี padding เยอะ
type BadStruct struct {
    A bool    // 1 byte
    B int64   // 8 bytes (7 bytes padding ก่อนหน้า)
    C bool    // 1 byte
    D int64   // 8 bytes (7 bytes padding ก่อนหน้า)
    E bool    // 1 byte
}
// Total: 1+7+8+1+7+8+1+7 = 40 bytes!

// ✅ Good alignment - เรียง fields จากใหญ่ไปเล็ก
type GoodStruct struct {
    B int64   // 8 bytes
    D int64   // 8 bytes
    A bool    // 1 byte
    C bool    // 1 byte
    E bool    // 1 byte
}
// Total: 8+8+1+1+1+5(padding) = 24 bytes!

func demonstrateAlignment() {
    bad := BadStruct{}
    good := GoodStruct{}
    
    fmt.Printf("BadStruct size: %d bytes\n", unsafe.Sizeof(bad))
    fmt.Printf("GoodStruct size: %d bytes\n", unsafe.Sizeof(good))
    
    // Offsets
    fmt.Printf("Bad.A offset: %d\n", unsafe.Offsetof(bad.A))
    fmt.Printf("Bad.B offset: %d\n", unsafe.Offsetof(bad.B))
    
    fmt.Printf("Good.B offset: %d\n", unsafe.Offsetof(good.B))
    fmt.Printf("Good.A offset: %d\n", unsafe.Offsetof(good.A))
}

// Cache line alignment (64 bytes)
// ป้องกัน False Sharing ใน concurrent programs
type PaddedCounter struct {
    count int64
    _     [56]byte // padding ให้ครบ 64 bytes (1 cache line)
}

type FalseSharing struct {
    // ❌ counter1 และ counter2 อยู่ cache line เดียวกัน
    counter1 int64
    counter2 int64
}

type NoFalseSharing struct {
    // ✅ แต่ละ counter อยู่คนละ cache line
    counter1 PaddedCounter
    counter2 PaddedCounter
}

func BenchmarkFalseSharing(b *testing.B) {
    var shared FalseSharing
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            atomic.AddInt64(&shared.counter1, 1) // ping-pong ระหว่าง cores
        }
    })
}

func BenchmarkNoFalseSharing(b *testing.B) {
    var padded NoFalseSharing
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            atomic.AddInt64(&padded.counter1.count, 1) // ไม่กระทบ counter2
        }
    })
}
```

---

## 5. Lock-free Data Structures

```go
// lockfree/queue.go
package lockfree

import (
    "sync/atomic"
    "unsafe"
)

// Lock-free Queue (Michael & Scott algorithm)
type node struct {
    value interface{}
    next  unsafe.Pointer
}

type LockFreeQueue struct {
    head unsafe.Pointer
    tail unsafe.Pointer
}

func NewLockFreeQueue() *LockFreeQueue {
    n := &node{}
    ptr := unsafe.Pointer(n)
    return &LockFreeQueue{head: ptr, tail: ptr}
}

func (q *LockFreeQueue) Enqueue(value interface{}) {
    newNode := &node{value: value}
    newPtr := unsafe.Pointer(newNode)
    
    for {
        tail := atomic.LoadPointer(&q.tail)
        tailNode := (*node)(tail)
        next := atomic.LoadPointer(&tailNode.next)
        
        // ตรวจสอบว่า tail ยังเป็น tail อยู่
        if tail == atomic.LoadPointer(&q.tail) {
            if next == nil {
                // CAS: ใส่ node ใหม่
                if atomic.CompareAndSwapPointer(&tailNode.next, nil, newPtr) {
                    // อัปเดต tail (อาจไม่สำเร็จแต่ไม่เป็นไร)
                    atomic.CompareAndSwapPointer(&q.tail, tail, newPtr)
                    return
                }
            } else {
                // Tail ล้าหลัง - อัปเดต
                atomic.CompareAndSwapPointer(&q.tail, tail, next)
            }
        }
    }
}

func (q *LockFreeQueue) Dequeue() (interface{}, bool) {
    for {
        head := atomic.LoadPointer(&q.head)
        tail := atomic.LoadPointer(&q.tail)
        headNode := (*node)(head)
        next := atomic.LoadPointer(&headNode.next)
        
        if head == atomic.LoadPointer(&q.head) {
            if head == tail {
                if next == nil {
                    return nil, false // Empty
                }
                // Tail ล้าหลัง - อัปเดต
                atomic.CompareAndSwapPointer(&q.tail, tail, next)
            } else {
                nextNode := (*node)(next)
                value := nextNode.value
                if atomic.CompareAndSwapPointer(&q.head, head, next) {
                    return value, true
                }
            }
        }
    }
}

// Lock-free Stack
type lfNode struct {
    value interface{}
    next  unsafe.Pointer
}

type LockFreeStack struct {
    top unsafe.Pointer
}

func (s *LockFreeStack) Push(value interface{}) {
    newNode := &lfNode{value: value}
    for {
        top := atomic.LoadPointer(&s.top)
        newNode.next = top
        if atomic.CompareAndSwapPointer(&s.top, top, unsafe.Pointer(newNode)) {
            return
        }
    }
}

func (s *LockFreeStack) Pop() (interface{}, bool) {
    for {
        top := atomic.LoadPointer(&s.top)
        if top == nil {
            return nil, false
        }
        
        topNode := (*lfNode)(top)
        next := topNode.next
        
        if atomic.CompareAndSwapPointer(&s.top, top, next) {
            return topNode.value, true
        }
    }
}

// Atomic operations
func AtomicExample() {
    var counter int64
    
    // Atomic increment
    atomic.AddInt64(&counter, 1)
    
    // CAS (Compare-and-Swap)
    old := atomic.LoadInt64(&counter)
    atomic.CompareAndSwapInt64(&counter, old, old*2)
    
    // Atomic load/store
    val := atomic.LoadInt64(&counter)
    atomic.StoreInt64(&counter, val+100)
}
```

---

## 6. Custom Memory Allocators

```go
// allocator/arena.go
package allocator

import (
    "unsafe"
)

// Arena Allocator - allocate เร็ว, free ทั้งก้อน
type Arena struct {
    memory []byte
    offset int
}

func NewArena(size int) *Arena {
    return &Arena{
        memory: make([]byte, size),
    }
}

// Alloc N bytes จาก arena
func (a *Arena) Alloc(size int) unsafe.Pointer {
    // Align ไปที่ 8 bytes
    aligned := (size + 7) &^ 7
    
    if a.offset+aligned > len(a.memory) {
        panic("arena out of memory")
    }
    
    ptr := unsafe.Pointer(&a.memory[a.offset])
    a.offset += aligned
    
    return ptr
}

// Reset arena - O(1) สำหรับ free ทั้งหมด
func (a *Arena) Reset() {
    a.offset = 0
}

// ใช้งานกับ request processing
type RequestArena struct {
    arena *Arena
}

func (ra *RequestArena) AllocString(s string) *string {
    ptr := ra.arena.Alloc(int(unsafe.Sizeof("") + uintptr(len(s))))
    str := (*string)(ptr)
    *str = s
    return str
}

// Slab Allocator - สำหรับ objects ขนาดเดียวกัน
type SlabAllocator struct {
    pool     sync.Pool
    objSize  int
}

func NewSlabAllocator(objSize int) *SlabAllocator {
    return &SlabAllocator{
        objSize: objSize,
        pool: sync.Pool{
            New: func() interface{} {
                return make([]byte, objSize)
            },
        },
    }
}

func (s *SlabAllocator) Alloc() []byte {
    return s.pool.Get().([]byte)
}

func (s *SlabAllocator) Free(b []byte) {
    // Clear แล้ว return ไป pool
    for i := range b {
        b[i] = 0
    }
    s.pool.Put(b)
}

// Buffer pool สำหรับ HTTP responses
var responsePool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    buf := responsePool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        responsePool.Put(buf)
    }()
    
    // เขียนไปยัง buffer ก่อน
    fmt.Fprintf(buf, `{"status": "ok", "path": "%s"}`, r.URL.Path)
    
    // แล้วค่อย copy ไป response
    w.Header().Set("Content-Type", "application/json")
    w.Write(buf.Bytes())
}
```

---

## 7. CPU Cache Optimization

```go
// cache_friendly.go
package main

import "testing"

// ❌ Column-major access (cache unfriendly)
func sumColumnMajor(matrix [][]int) int {
    n := len(matrix)
    sum := 0
    for col := 0; col < n; col++ {        // outer: column
        for row := 0; row < n; row++ {    // inner: row
            sum += matrix[row][col]        // กระโดด memory
        }
    }
    return sum
}

// ✅ Row-major access (cache friendly)
func sumRowMajor(matrix [][]int) int {
    sum := 0
    for _, row := range matrix { // sequential memory access
        for _, val := range row {
            sum += val
        }
    }
    return sum
}

// ✅ Flat 2D array (better cache locality)
type Matrix struct {
    data []int
    rows int
    cols int
}

func NewMatrix(rows, cols int) *Matrix {
    return &Matrix{
        data: make([]int, rows*cols),
        rows: rows,
        cols: cols,
    }
}

func (m *Matrix) Get(row, col int) int {
    return m.data[row*m.cols+col]
}

func (m *Matrix) Set(row, col, val int) {
    m.data[row*m.cols+col] = val
}

func (m *Matrix) Sum() int {
    sum := 0
    for _, v := range m.data { // Sequential access
        sum += v
    }
    return sum
}

// Benchmark
func BenchmarkColumnMajor(b *testing.B) {
    n := 1024
    matrix := make([][]int, n)
    for i := range matrix {
        matrix[i] = make([]int, n)
    }
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sumColumnMajor(matrix)
    }
}

func BenchmarkRowMajor(b *testing.B) {
    n := 1024
    matrix := make([][]int, n)
    for i := range matrix {
        matrix[i] = make([]int, n)
    }
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sumRowMajor(matrix)
    }
}

func BenchmarkFlatMatrix(b *testing.B) {
    n := 1024
    m := NewMatrix(n, n)
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        m.Sum()
    }
}

// Results (typical):
// BenchmarkColumnMajor-8  100  12345678 ns/op  (slow - cache misses)
// BenchmarkRowMajor-8     500  3456789 ns/op   (faster - better locality)
// BenchmarkFlatMatrix-8  1000  1234567 ns/op   (fastest - sequential)
```

---

## 8. Goroutine Optimization

```go
// goroutine_opt.go
package main

import (
    "runtime"
    "sync"
)

// Worker pool - จำกัดจำนวน goroutines
type WorkerPool struct {
    jobs    chan func()
    wg      sync.WaitGroup
    workers int
}

func NewWorkerPool(workers int, queueSize int) *WorkerPool {
    pool := &WorkerPool{
        jobs:    make(chan func(), queueSize),
        workers: workers,
    }
    
    for i := 0; i < workers; i++ {
        pool.wg.Add(1)
        go func() {
            defer pool.wg.Done()
            for job := range pool.jobs {
                job()
            }
        }()
    }
    
    return pool
}

func (p *WorkerPool) Submit(job func()) {
    p.jobs <- job
}

func (p *WorkerPool) Shutdown() {
    close(p.jobs)
    p.wg.Wait()
}

// GOMAXPROCS tuning
func tuneGOMAXPROCS() {
    // ปกติ Go ตั้งให้เท่ากับ CPU cores
    cpus := runtime.NumCPU()
    runtime.GOMAXPROCS(cpus)
    
    // สำหรับ CPU-intensive: GOMAXPROCS = NumCPU
    // สำหรับ I/O-intensive: GOMAXPROCS = NumCPU * 2
}

// Channel buffering
func optimalChannelBuffer() {
    // ❌ Unbuffered - goroutine block
    unbuffered := make(chan int)
    
    // ✅ Buffered - ลด context switching
    buffered := make(chan int, 100)
    
    // ✅✅ Batch processing ลด overhead
    batched := make(chan []int, 10)
    
    _ = unbuffered
    _ = buffered
    _ = batched
}

// Goroutine leak prevention
func safeGoroutine(ctx context.Context, fn func(ctx context.Context)) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("Goroutine panic: %v", r)
            }
        }()
        
        fn(ctx)
    }()
}
```

---

## Workshop: High-Performance HTTP Server

```go
// workshop/perf_server.go
package main

import (
    "bufio"
    "context"
    "fmt"
    "net"
    "net/http"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

var (
    requestCount  int64
    responseBytes int64
    activeConns   int64
)

type HighPerfServer struct {
    addr    string
    handler http.Handler
    pool    *WorkerPool
    
    // Connection pooling
    bufferPool sync.Pool
}

func NewHighPerfServer(addr string, handler http.Handler) *HighPerfServer {
    return &HighPerfServer{
        addr:    addr,
        handler: handler,
        pool:    NewWorkerPool(runtime.NumCPU()*2, 10000),
        bufferPool: sync.Pool{
            New: func() interface{} {
                return bufio.NewWriterSize(nil, 32*1024)
            },
        },
    }
}

func (s *HighPerfServer) Start() error {
    ln, err := net.Listen("tcp", s.addr)
    if err != nil {
        return err
    }
    
    // TCP optimizations
    tcpLn := ln.(*net.TCPListener)
    
    for {
        conn, err := tcpLn.AcceptTCP()
        if err != nil {
            return err
        }
        
        // TCP_NODELAY - ปิด Nagle's algorithm
        conn.SetNoDelay(true)
        conn.SetKeepAlive(true)
        conn.SetKeepAlivePeriod(60 * time.Second)
        
        atomic.AddInt64(&activeConns, 1)
        
        s.pool.Submit(func() {
            defer func() {
                atomic.AddInt64(&activeConns, -1)
                conn.Close()
            }()
            
            s.handleConn(conn)
        })
    }
}

func (s *HighPerfServer) handleConn(conn net.Conn) {
    // ดึง buffered writer จาก pool
    bw := s.bufferPool.Get().(*bufio.Writer)
    bw.Reset(conn)
    defer func() {
        bw.Flush()
        s.bufferPool.Put(bw)
    }()
    
    // Handle requests...
    atomic.AddInt64(&requestCount, 1)
}

// Benchmark server performance
func BenchmarkHTTPServer(b *testing.B) {
    srv := &http.Server{
        Addr: ":8080",
        Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Content-Type", "application/json")
            fmt.Fprint(w, `{"status":"ok"}`)
        }),
        // Tune timeouts
        ReadTimeout:       1 * time.Second,
        WriteTimeout:      1 * time.Second,
        IdleTimeout:       30 * time.Second,
        ReadHeaderTimeout: 500 * time.Millisecond,
        
        // Connection pooling
        MaxHeaderBytes: 1 << 20, // 1MB
    }
    
    // Run benchmarks...
    _ = srv
    b.ResetTimer()
}

func main() {
    // Performance tips สำหรับ production
    
    // 1. GOGC tuning
    // GOGC=100 (default) - run GC เมื่อ heap size double
    // GOGC=200 - run GC น้อยลง (ใช้ memory เพิ่มขึ้น แต่ GC pause น้อยลง)
    // os.Setenv("GOGC", "200")
    
    // 2. GOMAXPROCS
    runtime.GOMAXPROCS(runtime.NumCPU())
    
    // 3. SetMemoryLimit (Go 1.19+)
    // runtime/debug.SetMemoryLimit(512 * 1024 * 1024) // 512MB
    
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"status":"ok"}`))
    })
    
    mux.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "requests=%d\nactive_conns=%d\n",
            atomic.LoadInt64(&requestCount),
            atomic.LoadInt64(&activeConns),
        )
    })
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", mux)
}
```

---

## Benchmarks Summary

```go
// benchmarks_summary_test.go
package main

import "testing"

// String operations
func BenchmarkStringOps(b *testing.B) {
    b.Run("concat", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := "hello" + " " + "world"
            _ = s
        }
    })
    b.Run("sprintf", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := fmt.Sprintf("%s %s", "hello", "world")
            _ = s
        }
    })
    b.Run("builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            sb.WriteString("hello")
            sb.WriteByte(' ')
            sb.WriteString("world")
            _ = sb.String()
        }
    })
}

// Map vs slice lookup
func BenchmarkMapVsSlice(b *testing.B) {
    m := map[int]string{1: "one", 2: "two", 3: "three"}
    s := []string{"one", "two", "three"}
    
    b.Run("map", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = m[2]
        }
    })
    b.Run("slice", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = s[2]
        }
    })
}

// Interface vs direct call
type Adder interface {
    Add(a, b int) int
}

type ConcreteAdder struct{}

func (ca ConcreteAdder) Add(a, b int) int { return a + b }

func directAdd(a, b int) int { return a + b }

func BenchmarkInterfaceVsDirect(b *testing.B) {
    ca := ConcreteAdder{}
    var adder Adder = ca
    
    b.Run("direct", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = directAdd(1, 2)
        }
    })
    b.Run("interface", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = adder.Add(1, 2)
        }
    })
    b.Run("concrete", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = ca.Add(1, 2)
        }
    })
}
```

---

## สรุป

| เทคนิค | ลด | เพิ่ม |
|--------|-----|------|
| sync.Pool | Allocations, GC pressure | Code complexity |
| Pre-allocate slices | Re-allocations | Memory usage |
| Stack allocation | Heap allocs | - |
| Lock-free | Lock contention | Code complexity |
| Worker pool | Goroutine overhead | - |
| Memory alignment | Cache misses | Code size |
| Zero-copy | Memory copies, CPU | Code complexity |
| Row-major access | Cache misses | - |

### Profiling Process
1. **Measure** - อย่า optimize ก่อน profile
2. **Find bottleneck** - ใช้ pprof, trace
3. **Optimize** - แก้จาก hotspot
4. **Measure again** - ตรวจสอบ improvement

---

*จบ Part 72: Performance Optimization*
