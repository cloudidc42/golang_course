# Part 87: Go Runtime Internals

## เป้าหมายของบทเรียน
- เข้าใจ Go runtime architecture
- GMP scheduler model (Goroutine, Machine, Processor)
- Memory allocator
- Garbage collector algorithms
- Stack management
- Runtime hooks และ monitoring

---

## 1. GMP Scheduler Model

```go
// gmp_model.go - แสดงการทำงานของ GMP scheduler
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

/*
GMP Model:
- G (Goroutine): lightweight thread, stack เริ่มต้น 2-8KB
- M (Machine): OS thread ที่รัน goroutines
- P (Processor): logical processor, ควบคุม run queue

Relationships:
  G --> P (run queue)
  P --> M (bound)
  M --> OS Thread

Local Run Queue (LRQ): P มี queue ของ G ที่รอรัน
Global Run Queue (GRQ): G ที่ไม่ได้อยู่ใน LRQ ใด
Work Stealing: P ว่างสามารถขโมย G จาก P อื่น
*/

func demonstrateGMP() {
    fmt.Printf("=== GMP Scheduler Information ===\n\n")
    
    // จำนวน logical processors
    fmt.Printf("GOMAXPROCS (P count): %d\n", runtime.GOMAXPROCS(0))
    
    // จำนวน goroutines ปัจจุบัน
    fmt.Printf("Current goroutines: %d\n", runtime.NumGoroutine())
    
    // CPU count
    fmt.Printf("CPU count: %d\n", runtime.NumCPU())
    
    // OS threads (approx M count)
    // ไม่มี API ตรงๆ แต่สามารถดูจาก runtime stats
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    fmt.Printf("GC cycles completed: %d\n", ms.NumGC)
}

// goroutineScheduling แสดงการ schedule goroutines
func goroutineScheduling() {
    fmt.Println("\n=== Goroutine Scheduling Demo ===")
    
    var wg sync.WaitGroup
    results := make([]int, 5)
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            // Goroutine สามารถถูก preempt ระหว่างนี้
            work := 0
            for j := 0; j < 1000000; j++ {
                work += j
            }
            results[id] = work
            
        }(i)
    }
    
    wg.Wait()
    fmt.Printf("All %d goroutines completed\n", len(results))
}

// workStealing แสดง work stealing concept
func workStealing() {
    fmt.Println("\n=== Work Stealing Simulation ===")
    
    // สร้าง goroutines จำนวนมาก
    numTasks := 100
    tasks := make(chan int, numTasks)
    
    for i := 0; i < numTasks; i++ {
        tasks <- i
    }
    close(tasks)
    
    var wg sync.WaitGroup
    var mu sync.Mutex
    processorWork := make(map[int]int)
    
    // Workers จำลอง P
    numWorkers := runtime.GOMAXPROCS(0)
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            count := 0
            for range tasks {
                count++
                time.Sleep(time.Microsecond)
            }
            mu.Lock()
            processorWork[workerID] = count
            mu.Unlock()
        }(w)
    }
    
    wg.Wait()
    
    fmt.Printf("Work distribution across %d processors:\n", numWorkers)
    for pid, count := range processorWork {
        fmt.Printf("  P%d: %d tasks\n", pid, count)
    }
}

func main() {
    demonstrateGMP()
    goroutineScheduling()
    workStealing()
}
```

---

## 2. Memory Allocator

```go
// memory_allocator.go - Go memory allocator internals
package main

import (
    "fmt"
    "runtime"
    "unsafe"
)

/*
Go Memory Allocator Architecture:

Size Classes:
- Tiny allocator: < 16 bytes (ใช้ร่วมกัน)
- Small allocator: 16 bytes - 32KB (mspan)
- Large allocator: > 32KB (direct from heap)

Memory Hierarchy:
- mcache: per-P cache (ไม่ต้อง lock)
- mcentral: per-size-class global cache
- mheap: global heap
- OS: memory pages

mspan: ชุดของ pages สำหรับ size class เดียวกัน
*/

func memoryAllocationDemo() {
    fmt.Println("=== Memory Allocation Demo ===\n")
    
    // ดู memory stats ก่อน
    var before runtime.MemStats
    runtime.GC()
    runtime.ReadMemStats(&before)
    
    // allocate ขนาดต่างๆ
    allocations := make([]interface{}, 0)
    
    // Tiny: < 16 bytes
    for i := 0; i < 1000; i++ {
        b := new(int8)  // 1 byte
        allocations = append(allocations, b)
    }
    
    // Small: 16 bytes - 32KB
    for i := 0; i < 1000; i++ {
        b := make([]byte, 64)  // 64 bytes
        allocations = append(allocations, b)
    }
    
    // Medium
    for i := 0; i < 100; i++ {
        b := make([]byte, 4096)  // 4KB
        allocations = append(allocations, b)
    }
    
    // Large: > 32KB
    for i := 0; i < 10; i++ {
        b := make([]byte, 64*1024)  // 64KB
        allocations = append(allocations, b)
    }
    
    var after runtime.MemStats
    runtime.ReadMemStats(&after)
    
    fmt.Printf("Allocations:\n")
    fmt.Printf("  HeapAlloc:   %d bytes\n", after.HeapAlloc)
    fmt.Printf("  HeapInuse:   %d bytes\n", after.HeapInuse)
    fmt.Printf("  HeapObjects: %d\n", after.HeapObjects)
    fmt.Printf("  Mallocs:     %d\n", after.Mallocs-before.Mallocs)
    fmt.Printf("  Frees:       %d\n", after.Frees-before.Frees)
    
    _ = allocations
}

// stackVsHeap แสดงความแตกต่าง stack vs heap
func stackVsHeap() {
    fmt.Println("\n=== Stack vs Heap ===")
    
    // Stack allocation - เร็วกว่า, auto reclaim
    stackVar := 42
    fmt.Printf("Stack var: %d (addr: %p)\n", stackVar, &stackVar)
    
    // Heap allocation - ช้ากว่า, GC จัดการ
    heapVar := new(int)
    *heapVar = 42
    fmt.Printf("Heap var: %d (addr: %p)\n", *heapVar, heapVar)
    
    // Escape analysis ตัดสินใจ
    // ถ้า var ถูก return หรือส่ง pointer ออกไป -> escape to heap
    ptr := escapesToHeap()
    fmt.Printf("Escaped to heap: %d (addr: %p)\n", *ptr, ptr)
}

// escapesToHeap - local variable escape to heap
func escapesToHeap() *int {
    // x จะ escape to heap เพราะเรา return pointer ออกไป
    x := 100
    return &x
}

// sizeClassDemo แสดง size classes
func sizeClassDemo() {
    fmt.Println("\n=== Size Class Examples ===")
    
    // Go มี 68 size classes
    sizes := []int{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 
                   144, 160, 176, 192, 208, 224, 240, 256,
                   288, 320, 352, 384, 416, 448, 480, 512,
                   576, 640, 704, 768, 896, 1024, 1152, 1280,
                   1408, 1536, 1792, 2048, 2304, 2688, 3072,
                   3200, 3456, 4096, 4864, 5376, 6144, 6528,
                   6784, 6912, 8192, 9472, 9728, 10240, 10880,
                   12288, 13568, 14336, 16384, 18432, 19072,
                   20480, 21760, 24576, 27264, 28672, 32768}
    
    fmt.Printf("Go has %d size classes\n", len(sizes))
    fmt.Printf("Sizes (bytes): 0 to 32768\n")
    
    // แสดง overhead ของ size class
    testSizes := []int{1, 9, 17, 33, 65, 129, 257, 513, 1025, 4097}
    fmt.Printf("\n%-10s %-15s %-10s\n", "Requested", "Allocated", "Waste%")
    for _, req := range testSizes {
        // หา size class ที่เล็กที่สุดที่ใหญ่พอ
        allocated := findSizeClass(req, sizes)
        waste := float64(allocated-req) / float64(allocated) * 100
        fmt.Printf("%-10d %-15d %.1f%%\n", req, allocated, waste)
    }
    
    _ = unsafe.Sizeof(0) // just to use unsafe
}

func findSizeClass(size int, classes []int) int {
    for _, c := range classes {
        if c >= size {
            return c
        }
    }
    return size
}

func main() {
    memoryAllocationDemo()
    stackVsHeap()
    sizeClassDemo()
}
```

---

## 3. Garbage Collector

```go
// garbage_collector.go - GC internals
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "time"
)

/*
Go GC Algorithm: Tri-color Mark-and-Sweep (Concurrent)

3 Colors:
- White: ยังไม่ได้ scan (อาจ collect ได้)
- Gray: พบแล้ว แต่ยังไม่ได้ scan references
- Black: scan แล้ว (ยังใช้อยู่)

GC Phases:
1. Mark Setup (STW): enable write barriers
2. Mark: concurrent marking
3. Mark Termination (STW): flush write barriers
4. Sweep: concurrent sweeping

Write Barrier:
- ป้องกัน mutator จากการทำลาย GC invariant
- Dijkstra write barrier / Yuasa write barrier
*/

func gcInternalsDemo() {
    fmt.Println("=== GC Internals Demo ===\n")
    
    // ดู GC stats
    var stats debug.GCStats
    debug.ReadGCStats(&stats)
    
    fmt.Printf("GC runs so far: %d\n", stats.NumGC)
    if stats.NumGC > 0 {
        fmt.Printf("Last GC pause: %v\n", stats.PauseTotal/time.Duration(stats.NumGC))
    }
    
    // Memory stats
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    
    fmt.Printf("\nMemory Stats:\n")
    fmt.Printf("  Alloc:        %d KB\n", ms.Alloc/1024)
    fmt.Printf("  TotalAlloc:   %d KB\n", ms.TotalAlloc/1024)
    fmt.Printf("  Sys:          %d KB\n", ms.Sys/1024)
    fmt.Printf("  NumGC:        %d\n", ms.NumGC)
    fmt.Printf("  GCCPUFraction: %.4f\n", ms.GCCPUFraction)
    fmt.Printf("  NextGC:       %d KB\n", ms.NextGC/1024)
    fmt.Printf("  HeapIdle:     %d KB\n", ms.HeapIdle/1024)
    fmt.Printf("  HeapReleased: %d KB\n", ms.HeapReleased/1024)
}

// gcTrigger แสดงการ trigger GC
func gcTrigger() {
    fmt.Println("\n=== GC Trigger Demo ===")
    
    var before, after runtime.MemStats
    runtime.ReadMemStats(&before)
    
    // Allocate จำนวนมาก
    var data [][]byte
    for i := 0; i < 1000; i++ {
        data = append(data, make([]byte, 1024))
    }
    
    runtime.ReadMemStats(&after)
    fmt.Printf("After alloc: HeapAlloc=%d KB, NumGC=%d\n",
        after.HeapAlloc/1024, after.NumGC)
    
    // ล้าง reference
    data = nil
    
    // Force GC
    runtime.GC()
    
    var final runtime.MemStats
    runtime.ReadMemStats(&final)
    fmt.Printf("After GC:    HeapAlloc=%d KB, NumGC=%d\n",
        final.HeapAlloc/1024, final.NumGC)
    
    _ = data
}

// gcTuning แสดงการ tune GC
func gcTuning() {
    fmt.Println("\n=== GC Tuning ===")
    
    // GOGC environment variable
    // Default: 100 (heap doubles before GC triggers)
    // Lower = more GC, less memory
    // Higher = less GC, more memory
    // -1 = disable GC
    
    // ดู current GOGC
    currentGOGC := debug.SetGCPercent(-1)
    fmt.Printf("Current GOGC: %d\n", currentGOGC)
    debug.SetGCPercent(currentGOGC) // restore
    
    // GOMEMLIMIT (Go 1.19+)
    // กำหนด soft memory limit
    // ช่วยป้องกัน OOM
    
    fmt.Println("\nGC Tuning Options:")
    fmt.Println("  GOGC=100      (default: GC when heap doubles)")
    fmt.Println("  GOGC=200      (GC less frequently, use more memory)")
    fmt.Println("  GOGC=50       (GC more frequently, use less memory)")
    fmt.Println("  GOGC=off      (disable GC - not recommended)")
    fmt.Println("  GOMEMLIMIT=1GiB (set soft memory limit)")
    
    // runtime.SetFinalizer - cleanup เมื่อ object ถูก collect
    type Resource struct {
        name string
    }
    
    res := &Resource{name: "my-resource"}
    runtime.SetFinalizer(res, func(r *Resource) {
        fmt.Printf("Finalizer called for: %s\n", r.name)
    })
    
    // GC จะ call finalizer เมื่อ res ไม่มี references
    res = nil
    runtime.GC()
    time.Sleep(time.Millisecond) // ให้ finalizer รัน
}

func main() {
    gcInternalsDemo()
    gcTrigger()
    gcTuning()
}
```

---

## 4. Stack Management

```go
// stack_management.go - Go goroutine stack
package main

import (
    "fmt"
    "runtime"
)

/*
Go Stack Management:
- เริ่มต้น 2KB (Go 1.4+), เคย 8KB
- Grow ได้ถึง 1GB (default)
- ใช้ "contiguous stack" (copy-on-grow)

Stack Growth:
1. Function prologue ตรวจสอบ stack space
2. ถ้าไม่พอ: allocate stack ใหม่ที่ใหญ่กว่า 2x
3. Copy stack เดิมไปยังที่ใหม่
4. Update pointers

Stack Shrink:
- เกิดเมื่อ GC รัน
- ถ้า stack ใช้น้อยกว่า 1/4 ของขนาด -> shrink
*/

// stackGrowth ทดสอบ stack growth
func stackGrowth(depth int) int {
    if depth == 0 {
        // Report current stack info
        buf := make([]byte, 1024)
        n := runtime.Stack(buf, false)
        _ = buf[:n]
        return 0
    }
    
    // Allocate local vars เพื่อใช้ stack
    var arr [100]int
    for i := range arr {
        arr[i] = i
    }
    
    return arr[0] + stackGrowth(depth-1)
}

// goroutineStack แสดง stack ของ goroutines
func goroutineStack() {
    fmt.Println("=== Goroutine Stack Info ===\n")
    
    // Stack ของ goroutine ปัจจุบัน
    buf := make([]byte, 4096)
    n := runtime.Stack(buf, false)
    fmt.Printf("Current goroutine stack:\n%s\n", buf[:n])
    
    // Stack ทั้งหมด
    buf2 := make([]byte, 65536)
    n2 := runtime.Stack(buf2, true)
    fmt.Printf("Total goroutines stack size: %d bytes\n", n2)
    fmt.Printf("Num goroutines: %d\n", runtime.NumGoroutine())
}

// stackOverflow - แสดงว่า Go จัดการ stack overflow ยังไง
func stackDepthTest() {
    fmt.Println("\n=== Stack Depth Test ===")
    
    result := stackGrowth(100)
    fmt.Printf("Stack growth test (depth 100): %d\n", result)
    
    // Deep recursion test
    result2 := recursiveSum(10000)
    fmt.Printf("Recursive sum (10000): %d\n", result2)
}

func recursiveSum(n int) int {
    if n <= 0 {
        return 0
    }
    return n + recursiveSum(n-1)
}

// setMaxStack กำหนด max stack size
func stackLimitsDemo() {
    fmt.Println("\n=== Stack Limits ===")
    
    // Default max: 1GB
    // ปรับได้ด้วย runtime/debug.SetMaxStack
    
    // แสดง current stack stats
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    
    fmt.Printf("StackInuse:  %d bytes\n", ms.StackInuse)
    fmt.Printf("StackSys:    %d bytes\n", ms.StackSys)
    
    fmt.Println("\nStack tuning:")
    fmt.Println("  GOSTACK=8192   (initial stack size in bytes)")
    fmt.Println("  debug.SetMaxStack(N)  (max stack size)")
}

func main() {
    goroutineStack()
    stackDepthTest()
    stackLimitsDemo()
}
```

---

## 5. Runtime Hooks และ Monitoring

```go
// runtime_hooks.go - runtime monitoring
package main

import (
    "fmt"
    "os"
    "os/signal"
    "runtime"
    "runtime/pprof"
    "runtime/trace"
    "sync"
    "syscall"
    "time"
)

// RuntimeMonitor ตรวจสอบ runtime stats
type RuntimeMonitor struct {
    interval time.Duration
    stop     chan struct{}
    wg       sync.WaitGroup
}

// NewRuntimeMonitor สร้าง monitor
func NewRuntimeMonitor(interval time.Duration) *RuntimeMonitor {
    return &RuntimeMonitor{
        interval: interval,
        stop:     make(chan struct{}),
    }
}

// Start เริ่ม monitoring
func (m *RuntimeMonitor) Start() {
    m.wg.Add(1)
    go func() {
        defer m.wg.Done()
        ticker := time.NewTicker(m.interval)
        defer ticker.Stop()
        
        for {
            select {
            case <-ticker.C:
                m.report()
            case <-m.stop:
                return
            }
        }
    }()
}

// Stop หยุด monitoring
func (m *RuntimeMonitor) Stop() {
    close(m.stop)
    m.wg.Wait()
}

// report รายงาน stats
func (m *RuntimeMonitor) report() {
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    
    fmt.Printf("[%s] Goroutines=%d HeapAlloc=%dKB HeapSys=%dKB GC#=%d\n",
        time.Now().Format("15:04:05"),
        runtime.NumGoroutine(),
        ms.HeapAlloc/1024,
        ms.HeapSys/1024,
        ms.NumGC,
    )
}

// cpuProfile สร้าง CPU profile
func cpuProfile(duration time.Duration) {
    f, err := os.CreateTemp("", "cpu_profile_*.prof")
    if err != nil {
        fmt.Printf("Error creating profile: %v\n", err)
        return
    }
    defer f.Close()
    
    if err := pprof.StartCPUProfile(f); err != nil {
        fmt.Printf("Error starting CPU profile: %v\n", err)
        return
    }
    
    fmt.Printf("CPU profiling to: %s\n", f.Name())
    
    // รัน workload
    time.Sleep(duration)
    
    pprof.StopCPUProfile()
    fmt.Printf("CPU profile saved\n")
}

// heapProfile สร้าง heap profile
func heapProfile() {
    f, err := os.CreateTemp("", "heap_profile_*.prof")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer f.Close()
    
    runtime.GC()
    if err := pprof.WriteHeapProfile(f); err != nil {
        fmt.Printf("Error writing heap profile: %v\n", err)
        return
    }
    
    fmt.Printf("Heap profile saved to: %s\n", f.Name())
}

// executionTrace สร้าง execution trace
func executionTrace(duration time.Duration) {
    f, err := os.CreateTemp("", "trace_*.out")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer f.Close()
    
    if err := trace.Start(f); err != nil {
        fmt.Printf("Error starting trace: %v\n", err)
        return
    }
    
    fmt.Printf("Tracing to: %s\n", f.Name())
    time.Sleep(duration)
    
    trace.Stop()
    fmt.Printf("Trace saved\n")
    fmt.Printf("View with: go tool trace %s\n", f.Name())
}

// signalHandler จัดการ OS signals
func setupSignalHandler() {
    signals := make(chan os.Signal, 1)
    signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM, syscall.SIGUSR1)
    
    go func() {
        for sig := range signals {
            switch sig {
            case syscall.SIGUSR1:
                // Dump goroutine stacks
                buf := make([]byte, 1<<20)
                n := runtime.Stack(buf, true)
                fmt.Printf("=== Goroutine dump ===\n%s\n", buf[:n])
                
            case syscall.SIGINT, syscall.SIGTERM:
                fmt.Printf("Received %v, shutting down\n", sig)
                os.Exit(0)
            }
        }
    }()
}

func main() {
    fmt.Println("=== Runtime Monitoring Demo ===\n")
    
    // Start monitor
    monitor := NewRuntimeMonitor(500 * time.Millisecond)
    monitor.Start()
    
    // Setup signal handler
    setupSignalHandler()
    
    // Simulate workload
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Duration(id*100) * time.Millisecond)
        }(i)
    }
    
    wg.Wait()
    time.Sleep(200 * time.Millisecond)
    
    monitor.Stop()
    
    // Profiles
    fmt.Println("\n=== Profiling ===")
    heapProfile()
    
    fmt.Println("\nProfiling commands:")
    fmt.Println("  go tool pprof cpu.prof")
    fmt.Println("  go tool pprof heap.prof")
    fmt.Println("  go tool trace trace.out")
    fmt.Println("  go test -cpuprofile=cpu.prof -memprofile=mem.prof ./...")
}
```

---

## สรุป

บทนี้ครอบคลุม Go runtime internals:

1. **GMP Scheduler** - Goroutines, Machines, Processors และ work stealing
2. **Memory Allocator** - Size classes, mcache, mcentral, mheap
3. **Garbage Collector** - Tri-color mark-and-sweep concurrent GC
4. **Stack Management** - Contiguous stack, grow/shrink
5. **Runtime Monitoring** - pprof, traces, signals

### Key Takeaways

- GMP model ช่วยให้ Go scale goroutines ได้นับล้านตัว
- Memory allocator ใช้ size classes เพื่อลด fragmentation
- GC ทำงาน concurrent เพื่อลด pause time
- Stack เริ่มเล็ก (2KB) และขยายตัวอัตโนมัติ
- ใช้ pprof และ trace เพื่อ profile production code
