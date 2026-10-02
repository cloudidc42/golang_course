# Part 49: Memory Management ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ Go memory model
- เข้าใจ stack vs heap
- ทำ escape analysis
- เข้าใจ Garbage Collector
- ใช้ GOGC และ GOMEMLIMIT
- หา memory leaks
- ใช้ sync.Pool
- ลด allocations

---

## 1. Go Memory Model Overview

```go
// ตัวอย่าง 1: Go memory model basics

package main

import (
	"fmt"
	"runtime"
)

func memoryModelOverview() {
	// Stack: local variables, function frames
	// Fast allocation/deallocation (move stack pointer)
	// Size limit (default 1MB goroutine stack, grows to 1GB)
	
	// Heap: dynamically allocated objects
	// Managed by Garbage Collector
	// Slower than stack
	
	// Example: stack allocation
	x := 42           // stays on stack
	arr := [5]int{}   // fixed-size array, stays on stack
	
	// Example: heap allocation
	p := new(int)      // always heap
	s := make([]int, 100) // backing array on heap
	m := make(map[string]int) // on heap
	
	fmt.Printf("x=%d, p=%p, len(s)=%d, len(m)=%d, arr=%v\n",
		x, p, len(s), len(m), arr)
	
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	fmt.Printf("Heap: %.2f MB\n", float64(ms.HeapAlloc)/1024/1024)
}

func main() {
	memoryModelOverview()
}
```

---

## 2. Stack vs Heap

```go
// ตัวอย่าง 2: Understanding when variables escape to heap

package main

import (
	"fmt"
)

// STACK: local value, does not escape
func stackExample() int {
	x := 42
	return x // copy of value returned
}

// HEAP: pointer escapes
func heapExample() *int {
	x := 42
	return &x // pointer to x escapes to heap
}

// HEAP: large allocation
func largeAllocation() {
	// Large arrays go to heap
	bigArray := make([]byte, 10*1024*1024) // 10MB -> heap
	_ = bigArray
}

// HEAP: interface conversion
type Stringer interface {
	String() string
}

type MyVal struct {
	n int
}

func (m MyVal) String() string { return fmt.Sprintf("%d", m.n) }

func interfaceEscape() Stringer {
	v := MyVal{42}
	return v // value boxed in interface -> heap
}

// HEAP: closure captures
func closureCapture() func() int {
	x := 0
	return func() int {
		x++ // x captured by closure -> heap
		return x
	}
}

// STACK: no escape because not returned
func noEscape() {
	s := make([]int, 100) // might stay on stack if compiler sees no escape
	for i := range s {
		s[i] = i
	}
	_ = s
}

// Demonstrate
func escapeDemo() {
	fmt.Println("=== Escape Analysis Demo ===")
	fmt.Println("Check with: go build -gcflags='-m' ./...")
	fmt.Println()
	
	// Stack allocation (fast)
	v := stackExample()
	fmt.Printf("Stack: %d\n", v)
	
	// Heap allocation (GC managed)
	p := heapExample()
	fmt.Printf("Heap: %d\n", *p)
	
	// Closure with captured variable
	counter := closureCapture()
	fmt.Printf("Counter: %d, %d, %d\n", counter(), counter(), counter())
}

func main() {
	escapeDemo()
}
```

---

## 3. Garbage Collector

```go
// ตัวอย่าง 3: Understanding the GC

package main

import (
	"fmt"
	"runtime"
	"time"
)

func gcDemo() {
	fmt.Println("=== Garbage Collector Demo ===")
	
	// Force GC
	runtime.GC()
	
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	fmt.Printf("Before: HeapAlloc=%.2f MB, NumGC=%d\n",
		float64(ms.HeapAlloc)/1024/1024, ms.NumGC)
	
	// Allocate lots of short-lived objects
	for i := 0; i < 1000; i++ {
		data := make([]byte, 1024) // 1KB each
		_ = data
	}
	
	runtime.ReadMemStats(&ms)
	fmt.Printf("During: HeapAlloc=%.2f MB\n",
		float64(ms.HeapAlloc)/1024/1024)
	
	runtime.GC()
	runtime.ReadMemStats(&ms)
	fmt.Printf("After GC: HeapAlloc=%.2f MB, NumGC=%d\n",
		float64(ms.HeapAlloc)/1024/1024, ms.NumGC)
}

// ตัวอย่าง 4: GC stats monitoring

type GCMonitor struct {
	ticker   *time.Ticker
	done     chan struct{}
	lastNumGC uint32
}

func NewGCMonitor(interval time.Duration) *GCMonitor {
	m := &GCMonitor{
		ticker: time.NewTicker(interval),
		done:   make(chan struct{}),
	}
	go m.run()
	return m
}

func (m *GCMonitor) run() {
	for {
		select {
		case <-m.done:
			return
		case <-m.ticker.C:
			var ms runtime.MemStats
			runtime.ReadMemStats(&ms)
			
			newGCs := ms.NumGC - m.lastNumGC
			m.lastNumGC = ms.NumGC
			
			if newGCs > 0 {
				fmt.Printf("[GC] count=+%d total=%d heap=%.1fMB gc_pause=%.2fms\n",
					newGCs, ms.NumGC,
					float64(ms.HeapAlloc)/1024/1024,
					float64(ms.PauseNs[(ms.NumGC+255)%256])/1e6)
			}
		}
	}
}

func (m *GCMonitor) Stop() {
	m.ticker.Stop()
	close(m.done)
}

func main() {
	gcDemo()
	
	monitor := NewGCMonitor(100 * time.Millisecond)
	defer monitor.Stop()
	
	// Simulate workload
	for i := 0; i < 10; i++ {
		data := make([][]byte, 1000)
		for j := range data {
			data[j] = make([]byte, 1024)
		}
		time.Sleep(50 * time.Millisecond)
		data = nil
		runtime.GC()
	}
}
```

---

## 4. GOGC and GOMEMLIMIT

```bash
# ตัวอย่าง 5: GC tuning environment variables

# GOGC: GC target percentage (default 100)
# When heap doubles since last GC, trigger new GC
GOGC=100 ./app   # default: GC when heap doubles
GOGC=200 ./app   # GC when heap triples (less frequent, more memory)
GOGC=50  ./app   # GC when heap grows 50% (more frequent, less memory)
GOGC=off ./app   # Disable GC (careful!)

# GOMEMLIMIT: Hard memory limit (Go 1.19+)
GOMEMLIMIT=512MiB ./app  # Limit to 512MB
GOMEMLIMIT=1GiB  ./app  # Limit to 1GB

# Combination
GOGC=off GOMEMLIMIT=1GiB ./app  # Let GC manage based on memory limit only
```

```go
// ตัวอย่าง 6: Setting GC parameters programmatically

package main

import (
	"fmt"
	"runtime/debug"
)

func gcTuning() {
	// Set GOGC = 200 (GC less frequently)
	oldPercent := debug.SetGCPercent(200)
	fmt.Printf("Old GOGC: %d\n", oldPercent)
	
	// Set memory limit (Go 1.19+)
	oldLimit := debug.SetMemoryLimit(512 * 1024 * 1024) // 512MB
	fmt.Printf("Old memory limit: %d\n", oldLimit)
	
	// Get GC stats
	var stats debug.GCStats
	debug.ReadGCStats(&stats)
	fmt.Printf("GC count: %d\n", stats.NumGC)
	fmt.Printf("Last GC: %v\n", stats.LastGC)
	
	// Print GC tree (allocation sites)
	// debug.PrintStack()
	
	// Force GC
	debug.FreeOSMemory() // GC + return memory to OS
}

func main() {
	gcTuning()
}
```

---

## 5. Memory Leaks

```go
// ตัวอย่าง 7: Common memory leaks in Go

package main

import (
	"fmt"
	"net/http"
	"time"
)

// LEAK 1: Goroutine leak
func goroutineLeak() {
	ch := make(chan int)
	
	go func() {
		// This goroutine blocks forever if ch is never read
		ch <- 42 // LEAK: goroutine stuck here
	}()
	
	// ch never read -> goroutine never terminates
	fmt.Println("Leaked goroutine")
}

// FIX: Use buffered channel or context
func goroutineFixed() {
	ch := make(chan int, 1) // Buffered
	
	go func() {
		ch <- 42 // Non-blocking now
	}()
	
	select {
	case v := <-ch:
		fmt.Printf("Got: %d\n", v)
	case <-time.After(time.Second):
		fmt.Println("Timeout")
	}
}

// LEAK 2: Slice memory leak (holding large backing array)
func sliceLeak() []int {
	bigSlice := make([]int, 1000000)
	for i := range bigSlice {
		bigSlice[i] = i
	}
	// Returns slice of 10 elements but holds 1M backing array
	return bigSlice[:10] // LEAK
}

// FIX: Copy only what you need
func sliceFixed() []int {
	bigSlice := make([]int, 1000000)
	for i := range bigSlice {
		bigSlice[i] = i
	}
	
	result := make([]int, 10)
	copy(result, bigSlice[:10])
	return result // bigSlice can now be GCed
}

// LEAK 3: Map not cleaned up
type Cache struct {
	data map[string][]byte
}

func (c *Cache) addLeak(key string, value []byte) {
	c.data[key] = value // Never removed -> grows forever
}

// FIX: TTL or size limit
type TTLCache struct {
	data    map[string]cacheEntry
	maxSize int
}

type cacheEntry struct {
	value     []byte
	expiresAt time.Time
}

func (c *TTLCache) Set(key string, value []byte, ttl time.Duration) {
	if len(c.data) >= c.maxSize {
		c.evict()
	}
	c.data[key] = cacheEntry{
		value:     value,
		expiresAt: time.Now().Add(ttl),
	}
}

func (c *TTLCache) evict() {
	now := time.Now()
	for k, v := range c.data {
		if now.After(v.expiresAt) {
			delete(c.data, k)
		}
	}
}

// LEAK 4: HTTP response body not closed
func httpLeak() {
	resp, err := http.Get("http://example.com")
	if err != nil {
		return
	}
	// LEAK: body not closed!
	fmt.Println(resp.StatusCode)
}

// FIX: Always close body
func httpFixed() {
	resp, err := http.Get("http://example.com")
	if err != nil {
		return
	}
	defer resp.Body.Close() // Always close
	
	fmt.Println(resp.StatusCode)
}

// LEAK 5: Timer not stopped
func timerLeak() {
	timer := time.NewTimer(time.Hour)
	// Use timer once but don't stop it
	<-timer.C
	// Timer should be stopped but isn't
}

// FIX: Always stop timer
func timerFixed() {
	timer := time.NewTimer(time.Hour)
	defer timer.Stop() // Always stop
	
	select {
	case <-timer.C:
		fmt.Println("Timer fired")
	case <-time.After(time.Second):
		fmt.Println("Cancelled")
	}
}

func main() {
	goroutineFixed()
	
	s1 := sliceLeak()
	s2 := sliceFixed()
	fmt.Printf("Slice lengths: %d, %d\n", len(s1), len(s2))
	
	fmt.Println("Memory leak examples demonstrated")
}
```

---

## 6. sync.Pool

```go
// ตัวอย่าง 8: sync.Pool for reducing allocations

package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"sync"
	"testing"
)

// Pool for bytes.Buffer
var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
}

func getBuffer() *bytes.Buffer {
	return bufPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
	buf.Reset()
	bufPool.Put(buf)
}

// JSON marshal using pool
func marshalJSON(v interface{}) ([]byte, error) {
	buf := getBuffer()
	defer putBuffer(buf)
	
	enc := json.NewEncoder(buf)
	if err := enc.Encode(v); err != nil {
		return nil, err
	}
	
	// Copy result before returning buffer to pool
	result := make([]byte, buf.Len())
	copy(result, buf.Bytes())
	return result, nil
}

// ตัวอย่าง 9: Pool for large objects

type WorkItem struct {
	Data    []byte
	Results []byte
}

var workPool = sync.Pool{
	New: func() interface{} {
		return &WorkItem{
			Data:    make([]byte, 0, 65536),
			Results: make([]byte, 0, 65536),
		}
	},
}

func processWork(data []byte) []byte {
	item := workPool.Get().(*WorkItem)
	defer func() {
		item.Data = item.Data[:0]
		item.Results = item.Results[:0]
		workPool.Put(item)
	}()
	
	// Process using pre-allocated slices
	item.Data = append(item.Data, data...)
	
	for _, b := range item.Data {
		item.Results = append(item.Results, b^0xFF) // XOR transform
	}
	
	result := make([]byte, len(item.Results))
	copy(result, item.Results)
	return result
}

// Benchmark pool vs no pool
func BenchmarkWithPool(b *testing.B) {
	data := make([]byte, 1024)
	b.ReportAllocs()
	
	for i := 0; i < b.N; i++ {
		result := processWork(data)
		_ = result
	}
}

func BenchmarkWithoutPool(b *testing.B) {
	data := make([]byte, 1024)
	b.ReportAllocs()
	
	for i := 0; i < b.N; i++ {
		results := make([]byte, len(data))
		for j, v := range data {
			results[j] = v ^ 0xFF
		}
		_ = results
	}
}

func main() {
	// Demo pool
	data := map[string]string{"key": "value"}
	encoded, err := marshalJSON(data)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	fmt.Printf("Encoded: %s", encoded)
}
```

---

## 7. Reducing Allocations

```go
// ตัวอย่าง 10: Techniques to reduce allocations

package main

import (
	"fmt"
	"strings"
)

// Technique 1: Pre-allocate slices
func appendNaive(n int) []int {
	var result []int // starts with nil, multiple reallocations
	for i := 0; i < n; i++ {
		result = append(result, i)
	}
	return result
}

func appendPreallocated(n int) []int {
	result := make([]int, 0, n) // pre-allocate capacity
	for i := 0; i < n; i++ {
		result = append(result, i)
	}
	return result
}

// Technique 2: Reuse slices
type Processor struct {
	buffer []byte
}

func (p *Processor) Process(data []byte) []byte {
	// Reuse buffer if large enough
	if cap(p.buffer) < len(data) {
		p.buffer = make([]byte, len(data))
	} else {
		p.buffer = p.buffer[:len(data)]
	}
	
	for i, b := range data {
		p.buffer[i] = b * 2
	}
	return p.buffer
}

// Technique 3: Avoid interface boxing
type Counter interface {
	Inc()
	Value() int
}

type IntCounter struct {
	n int
}

func (c *IntCounter) Inc() { c.n++ }
func (c *IntCounter) Value() int { return c.n }

// Avoid: stores Counter interface (causes allocation for small types)
func incrementInterface(c Counter) {
	c.Inc()
}

// Better: concrete type
func incrementConcrete(c *IntCounter) {
	c.Inc()
}

// Technique 4: String builder
func buildString(parts []string) string {
	var b strings.Builder
	b.Grow(estimateSize(parts)) // Hint size
	for _, p := range parts {
		b.WriteString(p)
	}
	return b.String()
}

func estimateSize(parts []string) int {
	n := 0
	for _, p := range parts {
		n += len(p)
	}
	return n
}

// Technique 5: Avoid unnecessary conversions
func unnecessaryConversion(data []byte) {
	s := string(data)         // allocation!
	_ = len(s)
}

func noConversion(data []byte) {
	_ = len(data)              // no allocation
}

// For map lookups with string([]byte)
var lookup = map[string]int{"key": 1}

func lookupWithAlloc(data []byte) int {
	return lookup[string(data)] // Go optimizes this to avoid allocation
}

// Technique 6: Value types vs pointer types
type SmallStruct struct {
	A, B int
}

// Passing by value is fine for small structs
func processSmall(s SmallStruct) int {
	return s.A + s.B
}

type LargeStruct struct {
	Data [1024]byte
}

// Passing by pointer avoids copying 1KB
func processLarge(s *LargeStruct) int {
	return int(s.Data[0])
}

func main() {
	fmt.Println("=== Reducing Allocations ===")
	
	// Pre-allocate
	s1 := appendNaive(1000)
	s2 := appendPreallocated(1000)
	fmt.Printf("Slices: %d, %d\n", len(s1), len(s2))
	
	// Processor with reused buffer
	p := &Processor{}
	result := p.Process([]byte{1, 2, 3, 4, 5})
	fmt.Printf("Processed: %v\n", result)
	
	// Counter
	c := &IntCounter{}
	incrementConcrete(c)
	incrementConcrete(c)
	fmt.Printf("Counter: %d\n", c.Value())
}
```

---

## 8. Memory Profiling Tools

```go
// ตัวอย่าง 11: Finding memory leaks with pprof

package main

import (
	"net/http"
	_ "net/http/pprof"
	"os"
	"runtime/pprof"
	"time"
)

func detectLeaks() {
	// Enable pprof server
	go http.ListenAndServe(":6060", nil)
	
	// Take baseline snapshot
	f1, _ := os.Create("heap-before.prof")
	pprof.WriteHeapProfile(f1)
	f1.Close()
	
	// Run workload that might leak
	simulateWorkload()
	
	time.Sleep(time.Second)
	runtime.GC()
	
	// Take snapshot after
	f2, _ := os.Create("heap-after.prof")
	pprof.WriteHeapProfile(f2)
	f2.Close()
	
	fmt.Println("Profiles saved. Compare with:")
	fmt.Println("  go tool pprof -base heap-before.prof heap-after.prof")
}

func simulateWorkload() {
	// Simulate work that causes allocations
	for i := 0; i < 1000; i++ {
		data := make([]byte, 1024)
		_ = data
	}
}

// ตัวอย่าง 12: Goroutine leak detection

func detectGoroutineLeaks() {
	before := runtime.NumGoroutine()
	fmt.Printf("Goroutines before: %d\n", before)
	
	// Simulate potential leak
	for i := 0; i < 10; i++ {
		go func() {
			time.Sleep(time.Hour) // stuck goroutines
		}()
	}
	
	time.Sleep(100 * time.Millisecond)
	
	after := runtime.NumGoroutine()
	fmt.Printf("Goroutines after: %d (leaked: %d)\n", after, after-before)
	
	// In tests, use goleak package:
	// goleak.VerifyNone(t) // fails if goroutines are still running
}

func main() {
	detectLeaks()
}
```

---

## 9. Arena Allocator Pattern

```go
// ตัวอย่าง 13: Arena pattern for bulk allocation

package main

import (
	"fmt"
	"unsafe"
)

// Arena: allocate many small objects together
type Arena struct {
	data   []byte
	offset int
}

func NewArena(size int) *Arena {
	return &Arena{
		data: make([]byte, size),
	}
}

func (a *Arena) Alloc(size int) []byte {
	if a.offset+size > len(a.data) {
		// Grow arena
		newData := make([]byte, len(a.data)*2)
		copy(newData, a.data)
		a.data = newData
	}
	
	slice := a.data[a.offset : a.offset+size]
	a.offset += size
	return slice
}

func (a *Arena) Reset() {
	a.offset = 0 // Reuse all memory at once
}

func (a *Arena) Used() int  { return a.offset }
func (a *Arena) Total() int { return len(a.data) }

type Node struct {
	Value int
	Next  *Node
}

func buildListArena(arena *Arena, n int) *Node {
	nodeSize := int(unsafe.Sizeof(Node{}))
	
	head := (*Node)(unsafe.Pointer(&arena.Alloc(nodeSize)[0]))
	head.Value = 0
	
	current := head
	for i := 1; i < n; i++ {
		next := (*Node)(unsafe.Pointer(&arena.Alloc(nodeSize)[0]))
		next.Value = i
		current.Next = next
		current = next
	}
	
	return head
}

func arenaDemo() {
	arena := NewArena(1024 * 1024) // 1MB arena
	
	// Build linked list without individual allocations
	head := buildListArena(arena, 100)
	
	// Count nodes
	count := 0
	for n := head; n != nil; n = n.Next {
		count++
	}
	
	fmt.Printf("List: %d nodes, Arena used: %d bytes\n",
		count, arena.Used())
	
	// Reset arena - free all at once
	arena.Reset()
	fmt.Printf("After reset, Arena used: %d bytes\n", arena.Used())
}

func main() {
	arenaDemo()
}
```

---

## 10. Memory Layout Optimization

```go
// ตัวอย่าง 14: Struct field alignment

package main

import (
	"fmt"
	"unsafe"
)

// BAD: wastes memory due to padding
type BadStruct struct {
	A bool    // 1 byte + 7 bytes padding
	B int64   // 8 bytes
	C bool    // 1 byte + 7 bytes padding
	D int64   // 8 bytes
	E bool    // 1 byte + 7 bytes padding
}

// GOOD: packed efficiently
type GoodStruct struct {
	B int64  // 8 bytes
	D int64  // 8 bytes
	A bool   // 1 byte
	C bool   // 1 byte
	E bool   // 1 byte + 5 bytes padding
}

func alignmentDemo() {
	bad := BadStruct{}
	good := GoodStruct{}
	
	fmt.Printf("BadStruct size:  %d bytes\n", unsafe.Sizeof(bad))
	fmt.Printf("GoodStruct size: %d bytes\n", unsafe.Sizeof(good))
	
	// Use fieldalignment tool to check
	// go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
	// fieldalignment ./...
}

// ตัวอย่าง 15: Cache-friendly data structures

type Point struct {
	X, Y float64
}

// Structure of Arrays (SoA) - cache friendly for processing all X's
type PointsSoA struct {
	X []float64
	Y []float64
}

// Array of Structures (AoS) - cache friendly for accessing single points
type PointsAoS []Point

func sumXAoS(points PointsAoS) float64 {
	sum := 0.0
	for _, p := range points {
		sum += p.X // X and Y loaded together, but only X used
	}
	return sum
}

func sumXSoA(points PointsSoA) float64 {
	sum := 0.0
	for _, x := range points.X {
		sum += x // Only X loaded - cache efficient
	}
	return sum
}

func main() {
	alignmentDemo()
}
```

---

## สรุป

ใน Part 49 เราได้เรียนรู้:

1. **Memory Model**: Stack vs Heap ใน Go
2. **Escape Analysis**: เมื่อไหร่ตัวแปรหนีไป heap
3. **Garbage Collector**: ทำงานอย่างไร
4. **GOGC/GOMEMLIMIT**: tuning GC behavior
5. **Memory Leaks**: goroutine leaks, slice leaks, map leaks
6. **sync.Pool**: reuse objects เพื่อลด allocations
7. **Reducing Allocations**: pre-allocate, reuse buffers, avoid boxing
8. **Profiling**: heap profiles, goroutine leak detection
9. **Arena Allocator**: bulk allocation pattern
10. **Memory Layout**: struct field ordering สำหรับ alignment

---

## Resources

- [Go Memory Model](https://go.dev/ref/mem)
- [runtime/debug package](https://pkg.go.dev/runtime/debug)
- [sync.Pool](https://pkg.go.dev/sync#Pool)
- [fieldalignment](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment)
- [goleak](https://github.com/uber-go/goleak)
- [The Go Garbage Collector](https://tip.golang.org/doc/gc-guide)
