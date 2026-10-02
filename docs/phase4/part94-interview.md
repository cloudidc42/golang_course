# Part 94: Go Interview Preparation

## เป้าหมายของบทเรียน
- คำถาม Go interview ที่พบบ่อย 50+ ข้อ
- Goroutines, channels, GC questions
- Data structures และ algorithms
- System design questions
- Coding challenges

---

## Section 1: Go Language Fundamentals (Q1-15)

**Q1: Goroutine คืออะไร และต่างจาก thread อย่างไร?**

```go
// A: Goroutine เป็น lightweight concurrent execution unit
// ต่างจาก OS thread:
// - Stack เริ่มต้น 2-8KB (thread: 1-8MB)
// - Managed by Go runtime (GMP scheduler)
// - เปลี่ยน context โดยไม่ต้อง syscall
// - สร้างได้นับล้านตัว

package main

import (
    "fmt"
    "runtime"
    "sync"
)

func goroutineVsThread() {
    fmt.Printf("Goroutines demo:\n")
    
    var wg sync.WaitGroup
    count := 0
    var mu sync.Mutex
    
    // สร้าง 10,000 goroutines
    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            count++
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Count: %d, Goroutines used: %d\n", count, runtime.NumGoroutine())
}
```

---

**Q2: Channel คืออะไร? Buffered vs Unbuffered?**

```go
// A: Channel เป็น typed conduit สำหรับ goroutine communication
// Unbuffered: ส่งและรับต้อง sync กัน
// Buffered: ส่งได้โดยไม่ต้องรอถ้า buffer ยังว่าง

func channelDemo() {
    // Unbuffered
    ch := make(chan int)
    go func() { ch <- 42 }()
    v := <-ch
    fmt.Printf("Unbuffered: %d\n", v)
    
    // Buffered
    bch := make(chan int, 3)
    bch <- 1
    bch <- 2
    bch <- 3
    // bch <- 4  // would block!
    fmt.Printf("Buffered len=%d cap=%d\n", len(bch), cap(bch))
    
    // Directional channels
    send := make(chan<- int, 1)  // send-only
    recv := make(<-chan int, 1)  // receive-only
    _ = send
    _ = recv
}
```

---

**Q3: defer ทำงานอย่างไร?**

```go
// A: defer เลื่อนการ execute function จนกว่า surrounding function จะ return
// LIFO order, arguments evaluated immediately

func deferDemo() {
    // LIFO order
    defer fmt.Println("third")
    defer fmt.Println("second")
    defer fmt.Println("first")
    
    // Arguments evaluated at defer time
    x := 10
    defer fmt.Printf("x was: %d\n", x) // prints 10
    x = 20
    fmt.Printf("x is: %d\n", x) // prints 20
    
    // With panic/recover
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Recovered: %v\n", r)
        }
    }()
    // panic("oops") // would be recovered
}
```

---

**Q4: panic และ recover ทำงานอย่างไร?**

```go
func panicRecoverDemo() {
    safeDiv := func(a, b int) (result int, err error) {
        defer func() {
            if r := recover(); r != nil {
                err = fmt.Errorf("recovered: %v", r)
            }
        }()
        
        return a / b, nil // panics if b==0
    }
    
    v, err := safeDiv(10, 2)
    fmt.Printf("10/2 = %d, err=%v\n", v, err)
    
    v, err = safeDiv(10, 0)
    fmt.Printf("10/0 = %d, err=%v\n", v, err)
}
```

---

**Q5: interface ใน Go คืออะไร?**

```go
// A: Interface เป็น set of method signatures
// Go ใช้ implicit implementation (structural typing)

type Animal interface {
    Sound() string
    Name() string
}

type Dog struct{ name string }
func (d Dog) Sound() string { return "Woof" }
func (d Dog) Name() string  { return d.name }

type Cat struct{ name string }
func (c Cat) Sound() string { return "Meow" }
func (c Cat) Name() string  { return c.name }

func interfaceDemo() {
    animals := []Animal{
        Dog{name: "Rex"},
        Cat{name: "Whiskers"},
    }
    
    for _, a := range animals {
        fmt.Printf("%s says %s\n", a.Name(), a.Sound())
    }
    
    // Empty interface
    var anything interface{} = "hello"
    anything = 42
    anything = []int{1, 2, 3}
    fmt.Printf("anything: %v\n", anything)
    
    // Type assertion
    if s, ok := anything.([]int); ok {
        fmt.Printf("It's a slice: %v\n", s)
    }
}
```

---

**Q6: make vs new?**

```go
func makeVsNew() {
    // new: allocates memory, returns pointer to zero value
    p := new(int)           // *int pointing to 0
    s := new([]string)      // *[]string pointing to nil
    fmt.Printf("new int: %v\n", *p)
    fmt.Printf("new slice: %v\n", *s)
    
    // make: creates slices, maps, channels (initialized)
    sl := make([]int, 3, 5) // len=3, cap=5
    m := make(map[string]int)
    ch := make(chan int, 10)
    
    sl[0] = 1
    m["key"] = 42
    ch <- 99
    
    fmt.Printf("make slice: %v (len=%d cap=%d)\n", sl, len(sl), cap(sl))
    fmt.Printf("make map: %v\n", m)
    fmt.Printf("make chan: %v\n", <-ch)
}
```

---

**Q7: Closures ใน Go คืออะไร?**

```go
func closureDemo() {
    // Closure captures variables from outer scope
    counter := func() func() int {
        count := 0
        return func() int {
            count++
            return count
        }
    }
    
    c1 := counter()
    c2 := counter()
    
    fmt.Println(c1(), c1(), c1()) // 1 2 3
    fmt.Println(c2(), c2())       // 1 2 (independent)
    
    // Common gotcha: loop variable capture
    funcs := make([]func(), 5)
    for i := 0; i < 5; i++ {
        i := i // shadow to capture correctly
        funcs[i] = func() { fmt.Printf("%d ", i) }
    }
    for _, f := range funcs {
        f()
    }
    fmt.Println()
}
```

---

**Q8: Slice internals?**

```go
func sliceInternals() {
    // Slice = pointer + length + capacity
    original := []int{1, 2, 3, 4, 5}
    
    // Slice of slice shares underlying array!
    sub := original[1:3]
    fmt.Printf("sub: %v (len=%d cap=%d)\n", sub, len(sub), cap(sub))
    
    // Modifying sub modifies original
    sub[0] = 99
    fmt.Printf("original after modify: %v\n", original)
    
    // Append may allocate new array
    sub = append(sub, 100)
    fmt.Printf("original after append: %v\n", original)
    
    // copy to avoid sharing
    safe := make([]int, len(original[1:3]))
    copy(safe, original[1:3])
    safe[0] = 42
    fmt.Printf("original unchanged: %v\n", original)
}
```

---

## Section 2: Concurrency (Q9-20)

**Q9: WaitGroup ใช้ยังไง?**

```go
func waitGroupDemo() {
    var wg sync.WaitGroup
    results := make([]int, 5)
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            results[id] = id * id
        }(i)
    }
    
    wg.Wait()
    fmt.Printf("Results: %v\n", results)
}
```

---

**Q10: Mutex vs RWMutex?**

```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func (m *SafeMap) Set(key string, val int) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = val
}

func (m *SafeMap) Get(key string) (int, bool) {
    m.mu.RLock()  // Multiple readers can hold simultaneously
    defer m.mu.RUnlock()
    v, ok := m.data[key]
    return v, ok
}

// Use RWMutex when reads >> writes
// Use Mutex when writes are frequent
```

---

**Q11: Select statement?**

```go
func selectDemo() {
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)
    
    ch1 <- "one"
    ch2 <- "two"
    
    // select picks whichever is ready
    select {
    case msg := <-ch1:
        fmt.Println("ch1:", msg)
    case msg := <-ch2:
        fmt.Println("ch2:", msg)
    default:
        fmt.Println("no message ready")
    }
    
    // Timeout pattern
    timeout := make(chan bool, 1)
    go func() {
        // time.Sleep(2 * time.Second)
        timeout <- true
    }()
    
    select {
    case result := <-ch2:
        fmt.Println("got:", result)
    case <-timeout:
        fmt.Println("timed out")
    }
}
```

---

**Q12: Context ใช้ยังไง?**

```go
import "context"

func contextDemo() {
    // Cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go func(ctx context.Context) {
        select {
        case <-ctx.Done():
            fmt.Println("goroutine cancelled:", ctx.Err())
        }
    }(ctx)
    
    cancel() // Signal cancellation
    
    // Timeout
    ctx2, cancel2 := context.WithTimeout(context.Background(), 
        100*time.Millisecond)
    defer cancel2()
    
    select {
    case <-ctx2.Done():
        fmt.Println("timed out:", ctx2.Err())
    }
    
    // Values
    ctx3 := context.WithValue(context.Background(), "userID", 42)
    userID := ctx3.Value("userID").(int)
    fmt.Printf("userID: %d\n", userID)
}
```

---

**Q13: Race condition detection?**

```go
// Run with: go run -race main.go
// Or: go test -race ./...

var counter int  // Shared without protection

func raceDemo() {
    var wg sync.WaitGroup
    
    // BAD: race condition
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE!
        }()
    }
    wg.Wait()
    
    // GOOD: use atomic or mutex
    var atomicCounter int64
    var mu sync.Mutex
    
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            atomicCounter++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Printf("Safe counter: %d\n", atomicCounter)
}
```

---

## Section 3: Error Handling (Q14-20)

**Q14: Error handling patterns?**

```go
import "errors"

// Custom error type
type NotFoundError struct {
    ID   int
    Type string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with ID %d not found", e.Type, e.ID)
}

func findUser(id int) error {
    if id <= 0 {
        return fmt.Errorf("invalid id: %d: %w", id, ErrInvalidInput)
    }
    if id > 100 {
        return &NotFoundError{ID: id, Type: "user"}
    }
    return nil
}

var ErrInvalidInput = errors.New("invalid input")

func errorHandlingDemo() {
    err := findUser(-1)
    
    // errors.Is checks chain
    if errors.Is(err, ErrInvalidInput) {
        fmt.Println("invalid input error")
    }
    
    err2 := findUser(999)
    
    // errors.As extracts concrete type
    var notFound *NotFoundError
    if errors.As(err2, &notFound) {
        fmt.Printf("not found: %s %d\n", notFound.Type, notFound.ID)
    }
}
```

---

## Section 4: Algorithms (Q21-35)

**Q21: Binary Search**

```go
func binarySearch(arr []int, target int) int {
    left, right := 0, len(arr)-1
    
    for left <= right {
        mid := left + (right-left)/2 // avoid overflow
        
        if arr[mid] == target {
            return mid
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    
    return -1
}
```

---

**Q22: Two Sum**

```go
func twoSum(nums []int, target int) (int, int) {
    seen := make(map[int]int)
    
    for i, n := range nums {
        complement := target - n
        if j, ok := seen[complement]; ok {
            return j, i
        }
        seen[n] = i
    }
    
    return -1, -1
}
```

---

**Q23: Reverse a String**

```go
func reverseString(s string) string {
    runes := []rune(s)  // Handle Unicode
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

---

**Q24: Fibonacci (multiple approaches)**

```go
// Recursive O(2^n) - slow
func fibRecursive(n int) int {
    if n <= 1 { return n }
    return fibRecursive(n-1) + fibRecursive(n-2)
}

// Iterative O(n) - fast
func fibIterative(n int) int {
    if n <= 1 { return n }
    a, b := 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}

// Memoized O(n) with cache
func fibMemo(n int, memo map[int]int) int {
    if n <= 1 { return n }
    if v, ok := memo[n]; ok { return v }
    memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo)
    return memo[n]
}
```

---

**Q25: Linked List operations**

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

// Reverse linked list
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    
    return prev
}

// Detect cycle (Floyd's algorithm)
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    
    return false
}
```

---

## Section 5: System Design (Q36-50)

**Q36: Design a Rate Limiter**

```go
// Token Bucket Rate Limiter
type RateLimiter struct {
    tokens   float64
    maxTokens float64
    refillRate float64 // tokens per second
    lastRefill time.Time
    mu        sync.Mutex
}

func NewRateLimiter(maxTokens, refillRate float64) *RateLimiter {
    return &RateLimiter{
        tokens:    maxTokens,
        maxTokens: maxTokens,
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

func (r *RateLimiter) Allow() bool {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(r.lastRefill).Seconds()
    r.tokens = min64(r.maxTokens, r.tokens+elapsed*r.refillRate)
    r.lastRefill = now
    
    if r.tokens >= 1 {
        r.tokens--
        return true
    }
    return false
}

func min64(a, b float64) float64 {
    if a < b { return a }
    return b
}
```

---

**Q37: Design an LRU Cache**

```go
type LRUNode struct {
    key, val    int
    prev, next  *LRUNode
}

type LRUCache struct {
    cap  int
    data map[int]*LRUNode
    head, tail *LRUNode
}

func NewLRUCache(cap int) *LRUCache {
    head := &LRUNode{}
    tail := &LRUNode{}
    head.next = tail
    tail.prev = head
    
    return &LRUCache{
        cap:  cap,
        data: make(map[int]*LRUNode),
        head: head,
        tail: tail,
    }
}

func (c *LRUCache) Get(key int) int {
    if node, ok := c.data[key]; ok {
        c.remove(node)
        c.addFront(node)
        return node.val
    }
    return -1
}

func (c *LRUCache) Put(key, val int) {
    if node, ok := c.data[key]; ok {
        node.val = val
        c.remove(node)
        c.addFront(node)
        return
    }
    
    node := &LRUNode{key: key, val: val}
    c.data[key] = node
    c.addFront(node)
    
    if len(c.data) > c.cap {
        lru := c.tail.prev
        c.remove(lru)
        delete(c.data, lru.key)
    }
}

func (c *LRUCache) remove(node *LRUNode) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) addFront(node *LRUNode) {
    node.next = c.head.next
    node.prev = c.head
    c.head.next.prev = node
    c.head.next = node
}
```

---

## Main Function

```go
import (
    "fmt"
    "sync"
    "time"
)

func main() {
    fmt.Println("=== Go Interview Preparation ===\n")
    
    fmt.Println("1. Goroutine vs Thread:")
    goroutineVsThread()
    
    fmt.Println("\n2. Channel Demo:")
    channelDemo()
    
    fmt.Println("\n3. Defer:")
    deferDemo()
    
    fmt.Println("\n4. Panic/Recover:")
    panicRecoverDemo()
    
    fmt.Println("\n5. Make vs New:")
    makeVsNew()
    
    fmt.Println("\n6. Closures:")
    closureDemo()
    
    fmt.Println("\n7. Slice Internals:")
    sliceInternals()
    
    fmt.Println("\n8. WaitGroup:")
    waitGroupDemo()
    
    fmt.Println("\n9. Error Handling:")
    errorHandlingDemo()
    
    // Algorithms
    fmt.Println("\n10. Binary Search:")
    arr := []int{1, 3, 5, 7, 9, 11, 13, 15}
    fmt.Printf("Search 7: index=%d\n", binarySearch(arr, 7))
    fmt.Printf("Search 4: index=%d\n", binarySearch(arr, 4))
    
    fmt.Println("\n11. Two Sum:")
    i, j := twoSum([]int{2, 7, 11, 15}, 9)
    fmt.Printf("Indices: %d, %d\n", i, j)
    
    fmt.Println("\n12. LRU Cache:")
    cache := NewLRUCache(3)
    cache.Put(1, 1)
    cache.Put(2, 2)
    cache.Put(3, 3)
    fmt.Printf("Get(1)=%d\n", cache.Get(1))
    cache.Put(4, 4) // evicts key 2
    fmt.Printf("Get(2)=%d (evicted)\n", cache.Get(2))
    fmt.Printf("Get(3)=%d\n", cache.Get(3))
    
    fmt.Println("\n13. Rate Limiter:")
    rl := NewRateLimiter(3, 1) // 3 tokens, refill 1/sec
    for i := 0; i < 5; i++ {
        allowed := rl.Allow()
        fmt.Printf("Request %d: allowed=%v\n", i+1, allowed)
    }
    
    _ = time.Second
    _ = sync.Mutex{}
}
```

---

## สรุป

บทนี้ครอบคลุม Go interview preparation:

1. **Language Fundamentals** - goroutines, channels, defer, interface
2. **Concurrency** - WaitGroup, Mutex, Select, Context
3. **Error Handling** - custom errors, errors.Is/As
4. **Algorithms** - binary search, two sum, linked list, fibonacci
5. **System Design** - rate limiter, LRU cache

### Interview Tips

- อธิบาย trade-offs ของแต่ละ approach
- พูดถึง edge cases เสมอ
- เขียน tests ถ้ามีเวลา
- ถามคำถามเพื่อทำความเข้าใจ requirements
- ใช้ Go idioms: error handling, interfaces, channels
