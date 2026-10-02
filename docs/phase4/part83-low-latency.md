# Part 83: Low Latency Systems in Go

## เป้าหมายของบทเรียน
- Low latency system design
- Nanosecond-level optimization
- Lock-free programming
- Memory mapping
- CPU affinity
- NUMA awareness
- Network optimization
- Zero-copy networking

---

## 1. Low Latency System Design

```go
// lowlatency/design.go
package main

import (
    "fmt"
    "runtime"
    "sync/atomic"
    "time"
    "unsafe"
)

// LatencyMeasurement วัด latency อย่างแม่นยำ
type LatencyMeasurement struct {
    samples  []int64
    count    int64
    sum      int64
    min      int64
    max      int64
}

// NewLatencyMeasurement สร้าง measurement ใหม่
func NewLatencyMeasurement(capacity int) *LatencyMeasurement {
    return &LatencyMeasurement{
        samples: make([]int64, 0, capacity),
        min:     1<<62,
    }
}

// Record บันทึก latency sample (nanoseconds)
func (m *LatencyMeasurement) Record(ns int64) {
    m.samples = append(m.samples, ns)
    atomic.AddInt64(&m.count, 1)
    atomic.AddInt64(&m.sum, ns)
    
    // Update min/max (not thread-safe, but ok for benchmarking)
    if ns < m.min {
        m.min = ns
    }
    if ns > m.max {
        m.max = ns
    }
}

// Percentile คำนวณ percentile
func (m *LatencyMeasurement) Percentile(p float64) int64 {
    if len(m.samples) == 0 {
        return 0
    }
    
    sorted := make([]int64, len(m.samples))
    copy(sorted, m.samples)
    
    // Simple sort
    for i := 0; i < len(sorted); i++ {
        for j := i + 1; j < len(sorted); j++ {
            if sorted[j] < sorted[i] {
                sorted[i], sorted[j] = sorted[j], sorted[i]
            }
        }
    }
    
    idx := int(p / 100.0 * float64(len(sorted)))
    if idx >= len(sorted) {
        idx = len(sorted) - 1
    }
    return sorted[idx]
}

// Mean คืนค่าเฉลี่ย
func (m *LatencyMeasurement) Mean() float64 {
    count := atomic.LoadInt64(&m.count)
    if count == 0 {
        return 0
    }
    return float64(atomic.LoadInt64(&m.sum)) / float64(count)
}

// PrintReport พิมพ์รายงาน
func (m *LatencyMeasurement) PrintReport(name string) {
    fmt.Printf("=== Latency Report: %s ===\n", name)
    fmt.Printf("Count: %d\n", m.count)
    fmt.Printf("Min:   %d ns (%.2f µs)\n", m.min, float64(m.min)/1000)
    fmt.Printf("Mean:  %.0f ns (%.2f µs)\n", m.Mean(), m.Mean()/1000)
    fmt.Printf("P50:   %d ns (%.2f µs)\n", m.Percentile(50), float64(m.Percentile(50))/1000)
    fmt.Printf("P95:   %d ns (%.2f µs)\n", m.Percentile(95), float64(m.Percentile(95))/1000)
    fmt.Printf("P99:   %d ns (%.2f µs)\n", m.Percentile(99), float64(m.Percentile(99))/1000)
    fmt.Printf("Max:   %d ns (%.2f µs)\n", m.max, float64(m.max)/1000)
}

// BenchmarkFunction วัด latency ของ function
func BenchmarkFunction(name string, iterations int, fn func()) *LatencyMeasurement {
    m := NewLatencyMeasurement(iterations)
    
    // Warmup
    for i := 0; i < iterations/10; i++ {
        fn()
    }
    
    for i := 0; i < iterations; i++ {
        start := time.Now().UnixNano()
        fn()
        elapsed := time.Now().UnixNano() - start
        m.Record(elapsed)
    }
    
    return m
}

func main() {
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())
    fmt.Println()
    
    // Benchmark: map lookup
    m := make(map[string]int)
    for i := 0; i < 1000; i++ {
        m[fmt.Sprintf("key-%d", i)] = i
    }
    
    mapResult := BenchmarkFunction("Map Lookup", 10000, func() {
        _ = m["key-500"]
    })
    mapResult.PrintReport("Map Lookup")
    
    fmt.Println()
    
    // Benchmark: atomic operation
    var counter int64
    atomicResult := BenchmarkFunction("Atomic Add", 10000, func() {
        atomic.AddInt64(&counter, 1)
    })
    atomicResult.PrintReport("Atomic Add")
    
    fmt.Println()
    
    // แสดง pointer size
    fmt.Printf("Pointer size: %d bytes\n", unsafe.Sizeof(uintptr(0)))
    fmt.Printf("Int size: %d bytes\n", unsafe.Sizeof(int(0)))
}
```

---

## 2. Lock-Free Programming

```go
// lockfree/queue.go
package main

import (
    "fmt"
    "runtime"
    "sync/atomic"
    "time"
    "unsafe"
)

// LockFreeQueue lock-free MPMC queue
type LockFreeQueue struct {
    head unsafe.Pointer // *queueNode
    tail unsafe.Pointer // *queueNode
    size int64
}

type queueNode struct {
    value interface{}
    next  unsafe.Pointer // *queueNode
}

// NewLockFreeQueue สร้าง queue ใหม่
func NewLockFreeQueue() *LockFreeQueue {
    sentinel := &queueNode{}
    ptr := unsafe.Pointer(sentinel)
    return &LockFreeQueue{head: ptr, tail: ptr}
}

// Enqueue เพิ่ม item (lock-free)
func (q *LockFreeQueue) Enqueue(value interface{}) {
    newNode := &queueNode{value: value}
    newPtr := unsafe.Pointer(newNode)
    
    for {
        tail := atomic.LoadPointer(&q.tail)
        tailNode := (*queueNode)(tail)
        next := atomic.LoadPointer(&tailNode.next)
        
        if tail == atomic.LoadPointer(&q.tail) {
            if next == nil {
                if atomic.CompareAndSwapPointer(&tailNode.next, nil, newPtr) {
                    atomic.CompareAndSwapPointer(&q.tail, tail, newPtr)
                    atomic.AddInt64(&q.size, 1)
                    return
                }
            } else {
                atomic.CompareAndSwapPointer(&q.tail, tail, next)
            }
        }
    }
}

// Dequeue ดึง item (lock-free)
func (q *LockFreeQueue) Dequeue() (interface{}, bool) {
    for {
        head := atomic.LoadPointer(&q.head)
        tail := atomic.LoadPointer(&q.tail)
        headNode := (*queueNode)(head)
        next := atomic.LoadPointer(&headNode.next)
        
        if head == atomic.LoadPointer(&q.head) {
            if head == tail {
                if next == nil {
                    return nil, false
                }
                atomic.CompareAndSwapPointer(&q.tail, tail, next)
            } else {
                nextNode := (*queueNode)(next)
                value := nextNode.value
                if atomic.CompareAndSwapPointer(&q.head, head, next) {
                    atomic.AddInt64(&q.size, -1)
                    return value, true
                }
            }
        }
    }
}

// Size คืนขนาด
func (q *LockFreeQueue) Size() int64 {
    return atomic.LoadInt64(&q.size)
}

// RingBuffer lock-free ring buffer สำหรับ single producer single consumer
type RingBuffer struct {
    data  []interface{}
    size  uint64
    mask  uint64
    head  uint64 // writer
    tail  uint64 // reader
}

// NewRingBuffer สร้าง ring buffer ใหม่ (size ต้องเป็น power of 2)
func NewRingBuffer(size uint64) *RingBuffer {
    return &RingBuffer{
        data: make([]interface{}, size),
        size: size,
        mask: size - 1,
    }
}

// Publish เพิ่ม item (single producer)
func (rb *RingBuffer) Publish(value interface{}) bool {
    head := atomic.LoadUint64(&rb.head)
    tail := atomic.LoadUint64(&rb.tail)
    
    if head-tail >= rb.size {
        return false // full
    }
    
    rb.data[head&rb.mask] = value
    atomic.StoreUint64(&rb.head, head+1)
    return true
}

// Consume ดึง item (single consumer)
func (rb *RingBuffer) Consume() (interface{}, bool) {
    tail := atomic.LoadUint64(&rb.tail)
    head := atomic.LoadUint64(&rb.head)
    
    if tail >= head {
        return nil, false // empty
    }
    
    value := rb.data[tail&rb.mask]
    atomic.StoreUint64(&rb.tail, tail+1)
    return value, true
}

// CASCounter counter ด้วย CAS
type CASCounter struct {
    value int64
}

// Increment เพิ่มค่า atomically
func (c *CASCounter) Increment() int64 {
    for {
        old := atomic.LoadInt64(&c.value)
        new := old + 1
        if atomic.CompareAndSwapInt64(&c.value, old, new) {
            return new
        }
        runtime.Gosched()
    }
}

// Get คืนค่าปัจจุบัน
func (c *CASCounter) Get() int64 {
    return atomic.LoadInt64(&c.value)
}

func main() {
    fmt.Println("=== Lock-Free Data Structures Demo ===\n")
    
    // Lock-free queue
    queue := NewLockFreeQueue()
    
    // Concurrent producers and consumers
    done := make(chan bool)
    produced := int64(0)
    consumed := int64(0)
    
    // Producers
    for i := 0; i < 4; i++ {
        go func(id int) {
            for j := 0; j < 1000; j++ {
                queue.Enqueue(fmt.Sprintf("item-%d-%d", id, j))
                atomic.AddInt64(&produced, 1)
            }
        }(i)
    }
    
    // Consumers
    for i := 0; i < 4; i++ {
        go func() {
            for {
                if item, ok := queue.Dequeue(); ok {
                    _ = item
                    atomic.AddInt64(&consumed, 1)
                    
                    if atomic.LoadInt64(&consumed) >= 4000 {
                        done <- true
                        return
                    }
                } else {
                    runtime.Gosched()
                }
            }
        }()
    }
    
    select {
    case <-done:
        fmt.Printf("Lock-Free Queue: produced=%d, consumed=%d\n\n", produced, consumed)
    case <-time.After(5 * time.Second):
        fmt.Printf("Timeout: produced=%d, consumed=%d\n", produced, consumed)
    }
    
    // Ring buffer
    rb := NewRingBuffer(1024)
    
    start := time.Now()
    for i := 0; i < 1000000; i++ {
        if !rb.Publish(i) {
            rb.Consume()
            rb.Publish(i)
        }
    }
    elapsed := time.Since(start)
    
    fmt.Printf("Ring Buffer: 1M operations in %v (%.2f M ops/sec)\n\n",
        elapsed, float64(1000000)/elapsed.Seconds()/1000000)
    
    // CAS Counter
    counter := &CASCounter{}
    var wg sync.WaitGroup
    
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                counter.Increment()
            }
        }()
    }
    
    // Avoid import issues - just show the concept
    time.Sleep(100 * time.Millisecond)
    fmt.Printf("CAS Counter: %d (expected ~100000)\n", counter.Get())
}
```

---

## 3. Memory Mapping & Zero-Copy

```go
// zerocopy/mmap.go
package main

import (
    "fmt"
    "os"
    "syscall"
    "time"
    "unsafe"
)

// MappedFile แสดง memory-mapped file
type MappedFile struct {
    data []byte
    size int64
    path string
}

// OpenMappedFile เปิดไฟล์ด้วย memory mapping
func OpenMappedFile(path string) (*MappedFile, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    
    info, err := f.Stat()
    if err != nil {
        return nil, err
    }
    
    data, err := syscall.Mmap(int(f.Fd()), 0, int(info.Size()),
        syscall.PROT_READ, syscall.MAP_SHARED)
    if err != nil {
        return nil, err
    }
    
    return &MappedFile{
        data: data,
        size: info.Size(),
        path: path,
    }, nil
}

// Read อ่านข้อมูลจาก mapped file
func (mf *MappedFile) Read(offset, length int64) []byte {
    if offset+length > mf.size {
        length = mf.size - offset
    }
    return mf.data[offset : offset+length]
}

// Close unmap file
func (mf *MappedFile) Close() error {
    return syscall.Munmap(mf.data)
}

// ZeroCopyBuffer buffer ที่ไม่ copy data
type ZeroCopyBuffer struct {
    ptr  unsafe.Pointer
    len  int
    cap  int
}

// ReadFromBytes สร้าง buffer จาก bytes โดยไม่ copy
func ReadFromBytes(data []byte) *ZeroCopyBuffer {
    if len(data) == 0 {
        return &ZeroCopyBuffer{}
    }
    return &ZeroCopyBuffer{
        ptr: unsafe.Pointer(&data[0]),
        len: len(data),
        cap: cap(data),
    }
}

// SliceAt ดึง slice ณ offset
func (b *ZeroCopyBuffer) SliceAt(offset, length int) []byte {
    if offset+length > b.len {
        return nil
    }
    start := unsafe.Pointer(uintptr(b.ptr) + uintptr(offset))
    return (*[1 << 30]byte)(start)[:length:length]
}

// CPUCacheOptimization แสดงการ optimize สำหรับ CPU cache
type CPUCacheOptimization struct{}

// BadCacheAccess ตัวอย่างที่ cache miss สูง (row-major vs column-major)
func BadCacheAccess(matrix [][]int, size int) int {
    sum := 0
    for j := 0; j < size; j++ {
        for i := 0; i < size; i++ {
            sum += matrix[i][j] // Column-major: cache miss สูง
        }
    }
    return sum
}

// GoodCacheAccess ตัวอย่างที่ cache hit สูง
func GoodCacheAccess(matrix [][]int, size int) int {
    sum := 0
    for i := 0; i < size; i++ {
        for j := 0; j < size; j++ {
            sum += matrix[i][j] // Row-major: cache friendly
        }
    }
    return sum
}

// FalseSharing จำลองปัญหา false sharing
type NoPadding struct {
    a int64
    b int64 // อาจอยู่ใน cache line เดียวกับ a
}

type WithPadding struct {
    a   int64
    _   [56]byte // padding เพื่อให้อยู่คนละ cache line (64 bytes per line)
    b   int64
    _   [56]byte
}

func main() {
    size := 500
    
    // สร้าง matrix
    matrix := make([][]int, size)
    for i := range matrix {
        matrix[i] = make([]int, size)
        for j := range matrix[i] {
            matrix[i][j] = i*size + j
        }
    }
    
    // เปรียบเทียบ cache access patterns
    fmt.Println("=== CPU Cache Optimization Demo ===\n")
    
    start := time.Now()
    result1 := BadCacheAccess(matrix, size)
    badTime := time.Since(start)
    
    start = time.Now()
    result2 := GoodCacheAccess(matrix, size)
    goodTime := time.Since(start)
    
    fmt.Printf("Bad cache access (column-major):  %v (result: %d)\n", badTime, result1)
    fmt.Printf("Good cache access (row-major):    %v (result: %d)\n", goodTime, result2)
    fmt.Printf("Speedup: %.2fx\n\n", float64(badTime)/float64(goodTime))
    
    // False sharing demo
    fmt.Println("=== False Sharing Demo ===")
    fmt.Printf("NoPadding size: %d bytes\n", unsafe.Sizeof(NoPadding{}))
    fmt.Printf("WithPadding size: %d bytes\n", unsafe.Sizeof(WithPadding{}))
    fmt.Println("WithPadding prevents false sharing by separating cache lines\n")
    
    // Zero-copy demo
    fmt.Println("=== Zero-Copy Demo ===")
    data := []byte("Hello, this is test data for zero-copy demonstration!")
    buf := ReadFromBytes(data)
    
    slice := buf.SliceAt(7, 10)
    fmt.Printf("Zero-copy slice at offset 7, length 10: '%s'\n", string(slice))
    
    // File creation and mmap demo
    tmpFile := "/tmp/test_mmap.txt"
    content := []byte("Memory-mapped file content for testing")
    os.WriteFile(tmpFile, content, 0644)
    
    mf, err := OpenMappedFile(tmpFile)
    if err != nil {
        fmt.Printf("mmap error (may not be supported): %v\n", err)
    } else {
        defer mf.Close()
        data := mf.Read(0, 20)
        fmt.Printf("Memory-mapped read: '%s'\n", string(data))
    }
    
    os.Remove(tmpFile)
}
```

---

## 4. Network Optimization

```go
// network/optimization.go
package main

import (
    "bufio"
    "fmt"
    "net"
    "sync"
    "time"
)

// ConnectionPool จัดการ connection pool
type ConnectionPool struct {
    mu       sync.Mutex
    idle     []net.Conn
    active   int
    maxIdle  int
    maxTotal int
    network  string
    address  string
}

// NewConnectionPool สร้าง pool ใหม่
func NewConnectionPool(network, address string, maxIdle, maxTotal int) *ConnectionPool {
    return &ConnectionPool{
        network:  network,
        address:  address,
        maxIdle:  maxIdle,
        maxTotal: maxTotal,
    }
}

// Get ดึง connection จาก pool
func (p *ConnectionPool) Get() (net.Conn, error) {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    // ใช้ idle connection ถ้ามี
    if len(p.idle) > 0 {
        conn := p.idle[len(p.idle)-1]
        p.idle = p.idle[:len(p.idle)-1]
        p.active++
        return conn, nil
    }
    
    // ตรวจสอบ limit
    if p.active >= p.maxTotal {
        return nil, fmt.Errorf("connection pool exhausted")
    }
    
    // สร้าง connection ใหม่
    conn, err := net.DialTimeout(p.network, p.address, 5*time.Second)
    if err != nil {
        return nil, err
    }
    
    p.active++
    return conn, nil
}

// Put คืน connection ไป pool
func (p *ConnectionPool) Put(conn net.Conn) {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    p.active--
    
    if len(p.idle) < p.maxIdle {
        p.idle = append(p.idle, conn)
        return
    }
    
    conn.Close()
}

// BufferedWriter writer ที่มี buffer เพื่อลด syscalls
type BufferedWriter struct {
    writer *bufio.Writer
    flush  time.Duration
    last   time.Time
}

// NewBufferedWriter สร้าง writer ใหม่
func NewBufferedWriter(conn net.Conn, bufSize int, flushInterval time.Duration) *BufferedWriter {
    return &BufferedWriter{
        writer: bufio.NewWriterSize(conn, bufSize),
        flush:  flushInterval,
        last:   time.Now(),
    }
}

// Write เขียนข้อมูล
func (bw *BufferedWriter) Write(data []byte) error {
    _, err := bw.writer.Write(data)
    if err != nil {
        return err
    }
    
    // Flush ถ้าถึงเวลา
    if time.Since(bw.last) >= bw.flush {
        bw.last = time.Now()
        return bw.writer.Flush()
    }
    
    return nil
}

// Nagle's algorithm: disable สำหรับ low-latency
func SetTCPNoDelay(conn net.Conn, nodelay bool) error {
    if tc, ok := conn.(*net.TCPConn); ok {
        return tc.SetNoDelay(nodelay)
    }
    return fmt.Errorf("not a TCP connection")
}

// NetworkBenchmark วัด network performance
func NetworkBenchmark() {
    // จำลอง network operations
    fmt.Println("=== Network Optimization Techniques ===\n")
    
    tips := []struct {
        name        string
        description string
    }{
        {
            "TCP_NODELAY",
            "Disable Nagle's algorithm สำหรับ latency-sensitive applications",
        },
        {
            "SO_REUSEPORT",
            "Allow multiple sockets ต่อ port เพื่อ load balance",
        },
        {
            "Connection Pool",
            "Reuse TCP connections เพื่อลด connection overhead",
        },
        {
            "Buffered I/O",
            "Batch small writes เพื่อลดจำนวน syscalls",
        },
        {
            "epoll/kqueue",
            "ใช้ edge-triggered I/O events แทน polling",
        },
        {
            "Zero-copy sendfile",
            "ส่ง file data โดยตรงจาก kernel buffer",
        },
        {
            "QUIC/HTTP3",
            "ลด latency ด้วย 0-RTT connection establishment",
        },
    }
    
    for i, tip := range tips {
        fmt.Printf("%d. **%s**: %s\n", i+1, tip.name, tip.description)
    }
}

func main() {
    NetworkBenchmark()
    
    // Pool demo
    fmt.Println("\n=== Connection Pool Demo ===\n")
    
    pool := NewConnectionPool("tcp", "localhost:8080", 10, 100)
    fmt.Printf("Connection pool created: maxIdle=%d, maxTotal=%d\n", pool.maxIdle, pool.maxTotal)
    fmt.Printf("Pool address: %s://%s\n", pool.network, pool.address)
    
    // Simulate TCP tuning
    fmt.Println("\n=== TCP Tuning Settings ===\n")
    settings := map[string]string{
        "net.ipv4.tcp_nodelay":           "1",
        "net.core.somaxconn":             "65535",
        "net.ipv4.tcp_max_syn_backlog":   "65535",
        "net.core.netdev_max_backlog":    "65535",
        "net.ipv4.tcp_rmem":              "4096 87380 16777216",
        "net.ipv4.tcp_wmem":              "4096 65536 16777216",
        "net.ipv4.tcp_fin_timeout":       "30",
        "net.ipv4.tcp_tw_reuse":          "1",
    }
    
    for k, v := range settings {
        fmt.Printf("sysctl %s = %s\n", k, v)
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Latency Measurement** - วิธีวัด latency อย่างแม่นยำ
2. **Lock-Free Programming** - Queue, Ring Buffer, CAS Counter
3. **Memory Mapping** - การใช้ mmap เพื่อ performance
4. **CPU Cache Optimization** - Cache-friendly data access patterns
5. **False Sharing** - การป้องกัน false sharing ด้วย padding
6. **Network Optimization** - Connection pool, TCP tuning

### Key Takeaways

- **Measure first**: อย่า optimize โดยไม่มีข้อมูล
- **Lock-free ≠ Fast always**: Lock contention ต่ำเท่านั้นที่ lock-free ดีกว่า
- **Cache locality**: ออกแบบ data structures ให้ cache-friendly
- **Avoid allocation in hot path**: GC pause ทำให้ latency spike
- **CPU affinity**: Pin goroutines ไปยัง specific CPU cores สำหรับ ultra-low latency
