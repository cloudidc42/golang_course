# Part 48: Benchmarking และ Profiling ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เขียน benchmarks ด้วย go test -bench
- วัด memory allocations
- ใช้ pprof สำหรับ CPU profiling
- ใช้ pprof สำหรับ memory profiling
- อ่าน flame graphs
- ใช้ go tool trace
- Optimize code based on profiling

---

## 1. Basic Benchmarks

```go
// ตัวอย่าง 1: Writing benchmarks

package benchmarks

import (
	"fmt"
	"strings"
	"testing"
)

// ฟังก์ชันที่จะ benchmark
func concatWithPlus(strs []string) string {
	result := ""
	for _, s := range strs {
		result += s
	}
	return result
}

func concatWithBuilder(strs []string) string {
	var b strings.Builder
	for _, s := range strs {
		b.WriteString(s)
	}
	return b.String()
}

func concatWithJoin(strs []string) string {
	return strings.Join(strs, "")
}

func concatWithSprintf(strs []string) string {
	result := ""
	for _, s := range strs {
		result = fmt.Sprintf("%s%s", result, s)
	}
	return result
}

// Benchmark functions
func BenchmarkConcatWithPlus(b *testing.B) {
	strs := makeStrings(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		concatWithPlus(strs)
	}
}

func BenchmarkConcatWithBuilder(b *testing.B) {
	strs := makeStrings(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		concatWithBuilder(strs)
	}
}

func BenchmarkConcatWithJoin(b *testing.B) {
	strs := makeStrings(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		concatWithJoin(strs)
	}
}

func BenchmarkConcatWithSprintf(b *testing.B) {
	strs := makeStrings(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		concatWithSprintf(strs)
	}
}

func makeStrings(n int) []string {
	strs := make([]string, n)
	for i := range strs {
		strs[i] = fmt.Sprintf("item%d", i)
	}
	return strs
}
```

```bash
# Run benchmarks
go test -bench=. ./...

# Run specific benchmark
go test -bench=BenchmarkConcat -benchtime=5s ./...

# With memory stats
go test -bench=. -benchmem ./...

# Run N times
go test -bench=. -count=3 ./...

# Output:
# BenchmarkConcatWithPlus-8      51374   23234 ns/op   56344 B/op   99 allocs/op
# BenchmarkConcatWithBuilder-8  1000000    1021 ns/op    1152 B/op    6 allocs/op
# BenchmarkConcatWithJoin-8     1442474     829 ns/op     896 B/op    1 allocs/op
# BenchmarkConcatWithSprintf-8    45978   25982 ns/op   74248 B/op  199 allocs/op
```

---

## 2. Benchmark with Different Sizes

```go
// ตัวอย่าง 2: Benchmarks with different input sizes

package benchmarks

func BenchmarkConcat(b *testing.B) {
	sizes := []int{10, 100, 1000, 10000}
	
	for _, size := range sizes {
		strs := makeStrings(size)
		
		b.Run(fmt.Sprintf("Plus_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				concatWithPlus(strs)
			}
		})
		
		b.Run(fmt.Sprintf("Builder_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				concatWithBuilder(strs)
			}
		})
		
		b.Run(fmt.Sprintf("Join_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				concatWithJoin(strs)
			}
		})
	}
}
```

---

## 3. Memory Benchmarks

```go
// ตัวอย่าง 3: Memory allocation benchmarks

package benchmarks

import (
	"sync"
	"testing"
)

// Without pool
func allocateWithoutPool() []byte {
	return make([]byte, 1024)
}

// With sync.Pool
var bytePool = sync.Pool{
	New: func() interface{} {
		return make([]byte, 1024)
	},
}

func allocateWithPool() []byte {
	b := bytePool.Get().([]byte)
	return b
}

func releaseToPool(b []byte) {
	// Reset and return to pool
	for i := range b {
		b[i] = 0
	}
	bytePool.Put(b)
}

func BenchmarkAllocateWithoutPool(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		buf := allocateWithoutPool()
		_ = buf
	}
}

func BenchmarkAllocateWithPool(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		buf := allocateWithPool()
		releaseToPool(buf)
	}
}

// ตัวอย่าง 4: Measuring allocations manually

func BenchmarkMapVsSlice(b *testing.B) {
	b.Run("map lookup", func(b *testing.B) {
		m := map[int]string{1: "a", 2: "b", 3: "c"}
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			_ = m[2]
		}
	})
	
	b.Run("slice lookup", func(b *testing.B) {
		s := []string{"a", "b", "c"}
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			_ = s[1]
		}
	})
}

// Custom allocation counter
func BenchmarkWithCustomAllocs(b *testing.B) {
	b.ResetTimer()
	
	var totalAllocs uint64
	
	for i := 0; i < b.N; i++ {
		var ms runtime.MemStats
		runtime.ReadMemStats(&ms)
		before := ms.Mallocs
		
		// Run code being measured
		result := processData([]int{1, 2, 3, 4, 5})
		_ = result
		
		runtime.ReadMemStats(&ms)
		totalAllocs += ms.Mallocs - before
	}
	
	b.ReportMetric(float64(totalAllocs)/float64(b.N), "allocs/op")
}

func processData(data []int) []int {
	result := make([]int, len(data))
	for i, v := range data {
		result[i] = v * 2
	}
	return result
}
```

---

## 4. CPU Profiling with pprof

```go
// ตัวอย่าง 5: Enable pprof in HTTP server

package main

import (
	"net/http"
	_ "net/http/pprof"  // Import for side effects
	"log"
)

func main() {
	// pprof endpoints available at /debug/pprof/
	go func() {
		log.Println("pprof server on :6060")
		log.Println(http.ListenAndServe(":6060", nil))
	}()
	
	// Main server
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// Your app code
	})
	
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

```bash
# ตัวอย่าง 6: CPU profiling commands

# Collect 30 seconds of CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Or save to file
curl -o cpu.prof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof cpu.prof

# Interactive pprof commands:
# top10       - top 10 CPU consumers
# top -cum    - sorted by cumulative time
# web         - open flame graph in browser (needs graphviz)
# list <func> - show source code with annotations
# weblist <func> - show in browser

# Memory profile
curl -o mem.prof http://localhost:6060/debug/pprof/heap
go tool pprof mem.prof

# Goroutine profile
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Block profile (blocking operations)
go tool pprof http://localhost:6060/debug/pprof/block

# Mutex profile
go tool pprof http://localhost:6060/debug/pprof/mutex
```

---

## 5. CPU Profiling in Tests

```go
// ตัวอย่าง 7: CPU profile in benchmark

package benchmarks

import (
	"os"
	"runtime/pprof"
	"testing"
)

func BenchmarkWithCPUProfile(b *testing.B) {
	// Create CPU profile file
	f, err := os.Create("cpu.prof")
	if err != nil {
		b.Fatal(err)
	}
	defer f.Close()
	
	// Start profiling
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()
	
	// Run benchmark
	for i := 0; i < b.N; i++ {
		// Code to profile
		complexCalculation(1000)
	}
}

func complexCalculation(n int) int {
	result := 0
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			result += i * j
		}
	}
	return result
}
```

```bash
# Run benchmark and collect profile
go test -bench=BenchmarkWithCPUProfile -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof

# Memory profile
go test -bench=. -memprofile=mem.prof ./...
go tool pprof mem.prof
```

---

## 6. Flame Graphs

```bash
# ตัวอย่าง 8: Generate flame graph

# Using go tool pprof web
go tool pprof -http=:8888 cpu.prof
# Opens browser with flame graph at http://localhost:8888

# Using FlameGraph scripts (Brendan Gregg)
go tool pprof -raw -output=cpu.txt cpu.prof
stackcollapse-go.pl cpu.txt | flamegraph.pl > cpu.svg
open cpu.svg

# Continuous profiling with pyroscope
# Install: go install github.com/grafana/pyroscope/...
```

---

## 7. Memory Profiling

```go
// ตัวอย่าง 9: Memory profiling

package main

import (
	"os"
	"runtime"
	"runtime/pprof"
)

func captureHeapProfile(filename string) error {
	f, err := os.Create(filename)
	if err != nil {
		return err
	}
	defer f.Close()
	
	runtime.GC() // Get up-to-date stats
	return pprof.WriteHeapProfile(f)
}

func printMemStats() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	
	fmt.Printf("Memory Stats:\n")
	fmt.Printf("  Alloc      = %7.2f MB (currently allocated)\n",
		float64(m.Alloc)/1024/1024)
	fmt.Printf("  TotalAlloc = %7.2f MB (total cumulative)\n",
		float64(m.TotalAlloc)/1024/1024)
	fmt.Printf("  Sys        = %7.2f MB (from OS)\n",
		float64(m.Sys)/1024/1024)
	fmt.Printf("  HeapAlloc  = %7.2f MB\n",
		float64(m.HeapAlloc)/1024/1024)
	fmt.Printf("  HeapSys    = %7.2f MB\n",
		float64(m.HeapSys)/1024/1024)
	fmt.Printf("  HeapInuse  = %7.2f MB\n",
		float64(m.HeapInuse)/1024/1024)
	fmt.Printf("  NumGC      = %d\n", m.NumGC)
	fmt.Printf("  Mallocs    = %d\n", m.Mallocs)
	fmt.Printf("  Frees      = %d\n", m.Frees)
}

func memoryLeakDemo() {
	fmt.Println("Before allocation:")
	printMemStats()
	
	// Allocate 100MB
	data := make([][]byte, 100)
	for i := range data {
		data[i] = make([]byte, 1024*1024) // 1MB each
	}
	
	fmt.Println("\nAfter 100MB allocation:")
	printMemStats()
	
	// Capture profile before GC
	captureHeapProfile("heap_before_gc.prof")
	
	// GC
	data = nil
	runtime.GC()
	
	fmt.Println("\nAfter GC:")
	printMemStats()
}

func main() {
	memoryLeakDemo()
}
```

---

## 8. go tool trace

```go
// ตัวอย่าง 10: Runtime tracing

package main

import (
	"os"
	"runtime/trace"
)

func main() {
	// Create trace file
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()
	
	// Start tracing
	if err := trace.Start(f); err != nil {
		panic(err)
	}
	defer trace.Stop()
	
	// Your code here
	doWork()
}

func doWork() {
	// Simulate concurrent work
	done := make(chan struct{})
	
	for i := 0; i < 10; i++ {
		go func(id int) {
			defer func() { done <- struct{}{} }()
			
			// Task annotations visible in trace
			ctx, task := trace.NewTask(context.Background(),
				fmt.Sprintf("worker-%d", id))
			defer task.End()
			
			trace.Log(ctx, "status", "starting")
			time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
			trace.Log(ctx, "status", "done")
		}(i)
	}
	
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

```bash
# Run and view trace
go run main.go
go tool trace trace.out
# Opens browser at http://localhost:PORT
# View: goroutine analysis, scheduler latency, network blocking, etc.
```

---

## 9. benchstat - Compare Benchmarks

```bash
# ตัวอย่าง 11: Comparing benchmark results

# Install benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# Run benchmark before change
go test -bench=. -count=5 ./... > old.txt

# Make code changes...

# Run after change
go test -bench=. -count=5 ./... > new.txt

# Compare
benchstat old.txt new.txt

# Output:
# name              old time/op    new time/op    delta
# ConcatBuilder-8   1.02µs ± 3%    0.85µs ± 2%  -16.67%  (p=0.008 n=5+5)
# ConcatJoin-8       829ns ± 1%     750ns ± 2%   -9.53%  (p=0.016 n=5+5)
#
# name              old alloc/op   new alloc/op   delta
# ConcatBuilder-8   1.15kB ± 0%    0.90kB ± 0%  -21.74%  (p=0.008 n=5+5)
```

---

## 10. Optimization Examples

```go
// ตัวอย่าง 12: Before/after optimization

package benchmarks

import (
	"testing"
	"sort"
)

// BEFORE: Naive deduplication
func deduplicateNaive(items []int) []int {
	result := []int{}
	for _, item := range items {
		found := false
		for _, r := range result {
			if r == item {
				found = true
				break
			}
		}
		if !found {
			result = append(result, item)
		}
	}
	return result
}

// AFTER: Using map for O(1) lookup
func deduplicateWithMap(items []int) []int {
	seen := make(map[int]bool, len(items))
	result := make([]int, 0, len(items))
	for _, item := range items {
		if !seen[item] {
			seen[item] = true
			result = append(result, item)
		}
	}
	return result
}

// ALTERNATIVE: Sort + unique (no map)
func deduplicateSorted(items []int) []int {
	if len(items) == 0 {
		return items
	}
	sorted := make([]int, len(items))
	copy(sorted, items)
	sort.Ints(sorted)
	
	result := sorted[:1]
	for _, v := range sorted[1:] {
		if v != result[len(result)-1] {
			result = append(result, v)
		}
	}
	return result
}

func makeDuplicateInts(n int) []int {
	items := make([]int, n)
	for i := range items {
		items[i] = i % (n / 2) // 50% duplicates
	}
	return items
}

func BenchmarkDeduplicate(b *testing.B) {
	sizes := []int{100, 1000, 10000}
	
	for _, size := range sizes {
		items := makeDuplicateInts(size)
		
		b.Run(fmt.Sprintf("Naive_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				deduplicateNaive(items)
			}
		})
		
		b.Run(fmt.Sprintf("Map_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				deduplicateWithMap(items)
			}
		})
		
		b.Run(fmt.Sprintf("Sorted_%d", size), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				deduplicateSorted(items)
			}
		})
	}
}
```

---

## 11. Escape Analysis

```go
// ตัวอย่าง 13: Understanding escape analysis

package benchmarks

// Go escapes to heap when:
// 1. Variable address taken and returned
// 2. Interface conversion
// 3. Too large for stack
// 4. Closure captures variable

// Does NOT escape (stays on stack)
func noEscape(n int) int {
	x := n * 2 // x stays on stack
	return x
}

// ESCAPES to heap
func escapes(n int) *int {
	x := n * 2 // x escapes because we return its address
	return &x
}

// Interface causes escape
func interfaceEscapes(n int) interface{} {
	x := n
	return x // escapes because interface{}
}

// Check escape analysis:
// go build -gcflags="-m" ./...
// go build -gcflags="-m=2" ./...  // more verbose

// Output example:
// ./main.go:20:2: moved to heap: x
// ./main.go:26:9: n escapes to heap

func BenchmarkStackVsHeap(b *testing.B) {
	b.Run("stack", func(b *testing.B) {
		sum := 0
		for i := 0; i < b.N; i++ {
			sum += noEscape(i)
		}
		_ = sum
	})
	
	b.Run("heap", func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			p := escapes(i)
			_ = p
		}
	})
}
```

---

## 12. Continuous Profiling

```go
// ตัวอย่าง 14: Auto-profiling middleware

package middleware

import (
	"net/http"
	"os"
	"runtime/pprof"
	"time"
)

// ProfilerMiddleware captures profiles periodically
type ProfilerMiddleware struct {
	outputDir string
	interval  time.Duration
}

func NewProfilerMiddleware(outputDir string, interval time.Duration) *ProfilerMiddleware {
	os.MkdirAll(outputDir, 0755)
	return &ProfilerMiddleware{outputDir: outputDir, interval: interval}
}

func (p *ProfilerMiddleware) Start() {
	go func() {
		ticker := time.NewTicker(p.interval)
		defer ticker.Stop()
		
		for t := range ticker.C {
			p.captureProfile(t)
		}
	}()
}

func (p *ProfilerMiddleware) captureProfile(t time.Time) {
	// CPU profile
	cpuFile := fmt.Sprintf("%s/cpu_%s.prof", p.outputDir,
		t.Format("20060102_150405"))
	f, err := os.Create(cpuFile)
	if err != nil {
		return
	}
	defer f.Close()
	
	pprof.StartCPUProfile(f)
	time.Sleep(10 * time.Second) // Capture 10 seconds
	pprof.StopCPUProfile()
}

// ตัวอย่าง 15: Request profiling

func RequestProfileHandler(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Query().Get("profile") == "1" {
			// Profile this specific request
			f, _ := os.CreateTemp("", "request-profile-*.prof")
			pprof.StartCPUProfile(f)
			defer func() {
				pprof.StopCPUProfile()
				f.Close()
				fmt.Printf("Profile saved: %s\n", f.Name())
			}()
		}
		
		next.ServeHTTP(w, r)
	})
}
```

---

## สรุป

ใน Part 48 เราได้เรียนรู้:

1. **Benchmarks**: go test -bench, การเขียน benchmark ที่ถูกต้อง
2. **Memory Benchmarks**: วัด allocations, -benchmem
3. **Sub-benchmarks**: ทดสอบหลาย input sizes
4. **pprof HTTP**: /debug/pprof endpoints
5. **CPU Profiling**: ค้นหา CPU bottlenecks
6. **Memory Profiling**: ค้นหา memory issues
7. **Flame Graphs**: visualize CPU usage
8. **go tool trace**: runtime event tracing
9. **benchstat**: เปรียบเทียบ benchmark results
10. **Escape Analysis**: stack vs heap allocations
11. **Optimization**: ตัวอย่างการ optimize จาก profiling

---

## Resources

- [pprof](https://pkg.go.dev/runtime/pprof)
- [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat)
- [Profiling Go Programs](https://go.dev/blog/pprof)
- [go tool trace](https://pkg.go.dev/runtime/trace)
- [Flame Graph](https://www.brendangregg.com/flamegraphs.html)
