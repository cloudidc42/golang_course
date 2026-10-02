# Part 20: Defer, Panic, และ Recover ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ `defer` statement และลำดับการทำงาน
- ใช้ `defer` สำหรับ cleanup (ปิดไฟล์, unlock mutex)
- เข้าใจ deferred function evaluation
- รู้จัก `panic` และเมื่อไหรควรใช้
- ใช้ `recover` เพื่อดักจับ panic
- เข้าใจความแตกต่างระหว่าง panic และ error
- รู้จัก best practices ในการใช้งาน

---

## 20.1 defer Statement

### 20.1.1 พื้นฐาน defer

```go
package main

import "fmt"

func greet(name string) {
    defer fmt.Println("Goodbye,", name) // รันเมื่อ function return
    
    fmt.Println("Hello,", name)
    fmt.Println("How are you?")
}

func main() {
    greet("Alice")
    
    fmt.Println("\n--- Multiple defers ---")
    // defer ทำงานแบบ LIFO (Last In, First Out) = สุดท้ายสุด รันก่อน
    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")
    defer fmt.Println("defer 3")
    
    fmt.Println("Main body")
}
// Output:
// Hello, Alice
// How are you?
// Goodbye, Alice
//
// --- Multiple defers ---
// Main body
// defer 3
// defer 2
// defer 1
```

### 20.1.2 defer กับ Return Value

```go
package main

import "fmt"

// defer รันหลัง return แต่ก่อนที่ caller จะรับค่า
func countdown() {
    for i := 5; i >= 0; i-- {
        defer fmt.Println(i)
    }
}

// Named return values + defer
func readFileContent(filename string) (content string, err error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    
    // defer สามารถเปลี่ยน named return ได้
    defer func() {
        f.Close()
        if err != nil {
            content = "" // clean up content on error
        }
    }()
    
    // อ่านเนื้อหา...
    return "file content", nil
}

func doubleReturn() (result int) {
    defer func() {
        result *= 2 // แก้ไข named return ใน defer
    }()
    
    return 5 // จะถูก defer แก้เป็น 10
}

func main() {
    countdown()
    
    fmt.Println("\ndoubleReturn:", doubleReturn()) // 10
}
```

### 20.1.3 defer Argument Evaluation

```go
package main

import "fmt"

func main() {
    // Arguments ของ defer ถูก evaluate ทันทีที่เรียก defer
    // แต่ function body รันทีหลัง
    
    x := 10
    
    // Argument x=10 ถูก capture ณ ตอนที่ defer ถูกเรียก
    defer fmt.Println("defer x =", x) // จะ print 10
    
    x = 20 // เปลี่ยน x หลัง defer
    fmt.Println("x =", x)
    
    // Closure capture ตัวแปร by reference
    y := 100
    defer func() {
        fmt.Println("defer y =", y) // จะ print 200 (current value)
    }()
    y = 200
    fmt.Println("y =", y)
    
    fmt.Println("\n--- Loop demo ---")
    for i := 0; i < 3; i++ {
        i := i // สร้าง copy สำหรับ closure
        defer fmt.Printf("loop defer i=%d\n", i)
    }
}
// Output:
// x = 20
// y = 200
// loop defer i=2
// loop defer i=1
// loop defer i=0
// defer y = 200
// defer x = 10
```

---

## 20.2 defer สำหรับ Cleanup

### 20.2.1 ปิดไฟล์

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("เปิดไฟล์ไม่ได้: %w", err)
    }
    defer f.Close() // ปิดไฟล์เมื่อ function return ไม่ว่ากรณีใด
    
    scanner := bufio.NewScanner(f)
    lineNum := 0
    for scanner.Scan() {
        lineNum++
        fmt.Printf("%d: %s\n", lineNum, scanner.Text())
    }
    
    return scanner.Err()
}

func copyFile(src, dst string) error {
    source, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("เปิดไฟล์ต้นทาง: %w", err)
    }
    defer source.Close()
    
    destination, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("สร้างไฟล์ปลายทาง: %w", err)
    }
    defer destination.Close()
    
    _, err = io.Copy(destination, source)
    return err
}

func main() {
    os.WriteFile("test.txt", []byte("line1\nline2\nline3\n"), 0644)
    defer os.Remove("test.txt")
    
    if err := processFile("test.txt"); err != nil {
        fmt.Printf("error: %v\n", err)
    }
}
```

### 20.2.2 Unlock Mutex

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    count map[string]int
}

func NewSafeCounter() *SafeCounter {
    return &SafeCounter{count: make(map[string]int)}
}

func (c *SafeCounter) Increment(key string) {
    c.mu.Lock()
    defer c.mu.Unlock() // unlock เมื่อ function return
    
    c.count[key]++
}

func (c *SafeCounter) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    return c.count[key]
}

func (c *SafeCounter) IncrementIfLess(key string, limit int) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if c.count[key] >= limit {
        return false // defer จะ unlock ที่นี่ด้วย
    }
    
    c.count[key]++
    return true
}

func main() {
    counter := NewSafeCounter()
    
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment("hits")
        }()
    }
    
    wg.Wait()
    fmt.Println("Hits:", counter.Get("hits")) // 100
    
    // Test IncrementIfLess
    counter2 := NewSafeCounter()
    for i := 0; i < 5; i++ {
        ok := counter2.IncrementIfLess("requests", 3)
        fmt.Printf("Increment %d: %v (count=%d)\n", i+1, ok, counter2.Get("requests"))
    }
}
```

### 20.2.3 Database Transaction

```go
package main

import (
    "database/sql"
    "fmt"
)

// ตัวอย่าง (ไม่ต้องรัน - ต้องการ database จริง)
func transferMoney(db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    
    // defer rollback - ถ้า commit สำเร็จ rollback จะ return error ที่เราไม่สน
    defer tx.Rollback()
    
    // ดึงเงินจาก from account
    var balance float64
    err = tx.QueryRow("SELECT balance FROM accounts WHERE id = $1", fromID).Scan(&balance)
    if err != nil {
        return fmt.Errorf("get balance: %w", err)
    }
    
    if balance < amount {
        return fmt.Errorf("insufficient funds: %.2f < %.2f", balance, amount)
    }
    
    // หักเงิน
    _, err = tx.Exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
    if err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    
    // เพิ่มเงิน
    _, err = tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)
    if err != nil {
        return fmt.Errorf("credit: %w", err)
    }
    
    // commit - ถ้าสำเร็จ defer rollback จะ no-op
    return tx.Commit()
}

func main() {
    fmt.Println("Database transaction example (requires database)")
    fmt.Println("Pattern: defer tx.Rollback() → tx.Commit()")
}
```

### 20.2.4 HTTP Response Body

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/http/httptest"
)

func fetchData(url string) (string, error) {
    resp, err := http.Get(url)
    if err != nil {
        return "", fmt.Errorf("http get: %w", err)
    }
    defer resp.Body.Close() // ต้องปิด Body เสมอ!
    
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", fmt.Errorf("read body: %w", err)
    }
    
    return string(body), nil
}

func main() {
    // สร้าง test server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello from server!")
    }))
    defer server.Close()
    
    data, err := fetchData(server.URL)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("Response: %q\n", data)
}
```

---

## 20.3 defer กับ Loops

### 20.3.1 ปัญหาของ defer ใน Loop

```go
package main

import (
    "fmt"
    "os"
)

// ผิด - defer ไม่ถูก execute จนกว่า function จะ return
func badLoop() {
    files := []string{"a.txt", "b.txt", "c.txt"}
    
    for _, filename := range files {
        os.WriteFile(filename, []byte("content"), 0644)
        f, err := os.Open(filename)
        if err != nil {
            continue
        }
        defer f.Close() // ทุกไฟล์จะถูกปิดพร้อมกันตอน function return เท่านั้น!
        // ... ประมวลผล f ...
        _ = f
    }
}

// ถูก - ใช้ function แยก
func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // ปิดเมื่อ function นี้ return
    
    // อ่านและประมวลผล
    fmt.Printf("Processing: %s\n", filename)
    return nil
}

func goodLoop() {
    files := []string{"a.txt", "b.txt", "c.txt"}
    
    for _, filename := range files {
        if err := processFile(filename); err != nil {
            fmt.Printf("error processing %s: %v\n", filename, err)
        }
    }
}

// หรือใช้ closure
func goodLoopWithClosure() {
    files := []string{"a.txt", "b.txt", "c.txt"}
    
    for _, filename := range files {
        func() {
            f, err := os.Open(filename)
            if err != nil {
                return
            }
            defer f.Close()
            
            fmt.Printf("Processing: %s\n", filename)
        }()
    }
}

func main() {
    // สร้างไฟล์
    for _, name := range []string{"a.txt", "b.txt", "c.txt"} {
        os.WriteFile(name, []byte("hello"), 0644)
        defer os.Remove(name)
    }
    
    goodLoop()
    goodLoopWithClosure()
}
```

---

## 20.4 panic

### 20.4.1 panic พื้นฐาน

```go
package main

import "fmt"

func divide(a, b int) int {
    if b == 0 {
        panic("division by zero") // panic หยุด execution ทันที
    }
    return a / b
}

func mustPositive(n int) int {
    if n <= 0 {
        panic(fmt.Sprintf("expected positive number, got %d", n))
    }
    return n
}

func main() {
    // การเรียก panic จะหยุดโปรแกรมและ print stack trace
    // (ถ้าไม่มี recover)
    
    fmt.Println(divide(10, 2))   // 5
    
    // ตัวอย่าง Go runtime panic
    // var s []int
    // fmt.Println(s[0]) // runtime panic: index out of range
    
    // Panic กับ non-string value
    // panic(42)          // panic กับ int
    // panic(error)       // panic กับ error
    // panic(struct{}{})  // panic กับ struct
    
    fmt.Println("Program running normally")
    // ถ้า uncomment divide(10, 0) โปรแกรมจะหยุดทันที
}
```

### 20.4.2 เมื่อไหรควรใช้ panic

```go
package main

import (
    "fmt"
    "regexp"
)

// ควรใช้ panic เมื่อ:

// 1. Initialization ที่ต้องสำเร็จ (ใช้ Must pattern)
func MustCompile(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(fmt.Sprintf("invalid regexp pattern %q: %v", pattern, err))
    }
    return re
}

// 2. Programmer errors (ไม่ใช่ runtime errors)
func getElementAt(slice []int, index int) int {
    if index < 0 || index >= len(slice) {
        // นี่คือ programmer error - ควร panic ไม่ใช่ return error
        panic(fmt.Sprintf("index %d out of range [0, %d)", index, len(slice)))
    }
    return slice[index]
}

// 3. Impossible states (logic errors)
func dayOfWeek(n int) string {
    switch n {
    case 0:
        return "Sunday"
    case 1:
        return "Monday"
    case 2:
        return "Tuesday"
    case 3:
        return "Wednesday"
    case 4:
        return "Thursday"
    case 5:
        return "Friday"
    case 6:
        return "Saturday"
    default:
        panic(fmt.Sprintf("invalid day: %d", n))
    }
}

// ไม่ควรใช้ panic เมื่อ:
// - Input validation (ใช้ error แทน)
// - Network errors
// - File I/O errors
// - User input errors

// ตัวอย่างที่ดี: ใช้ error
func parseAge(s string) (int, error) {
    var age int
    _, err := fmt.Sscanf(s, "%d", &age)
    if err != nil {
        return 0, fmt.Errorf("invalid age %q: %w", s, err)
    }
    if age < 0 || age > 150 {
        return 0, fmt.Errorf("age %d out of range", age)
    }
    return age, nil
}

func main() {
    // Must pattern
    re := MustCompile(`\d+`)
    fmt.Println(re.MatchString("123")) // true
    
    // getElementAt
    data := []int{10, 20, 30}
    fmt.Println(getElementAt(data, 1)) // 20
    
    // dayOfWeek
    fmt.Println(dayOfWeek(1)) // Monday
    
    // parseAge
    age, err := parseAge("25")
    if err != nil {
        fmt.Printf("error: %v\n", err)
    } else {
        fmt.Printf("Age: %d\n", age)
    }
    
    _, err = parseAge("invalid")
    fmt.Printf("Parse error: %v\n", err)
}
```

---

## 20.5 recover

### 20.5.1 recover พื้นฐาน

```go
package main

import "fmt"

func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            // r คือค่าที่ส่งไปกับ panic
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    
    return a / b, nil
}

func main() {
    // กรณีปกติ
    result, err := safeDiv(10, 2)
    fmt.Printf("10/2 = %d, err = %v\n", result, err)
    
    // กรณี panic (division by zero)
    result, err = safeDiv(10, 0)
    fmt.Printf("10/0 = %d, err = %v\n", result, err)
    
    // recover ต้องอยู่ใน deferred function เท่านั้น
    fmt.Println("Program continues after recovered panic")
}
```

### 20.5.2 recover pattern ที่ใช้บ่อย

```go
package main

import (
    "fmt"
    "runtime/debug"
)

// SafeRun - รัน function โดย catch panic ทั้งหมด
func SafeRun(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch v := r.(type) {
            case error:
                err = v
            case string:
                err = fmt.Errorf("panic: %s", v)
            default:
                err = fmt.Errorf("panic: %v", r)
            }
        }
    }()
    
    f()
    return nil
}

// SafeRunWithStack - เพิ่ม stack trace
func SafeRunWithStack(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            stack := debug.Stack()
            err = fmt.Errorf("panic: %v\n\nStack trace:\n%s", r, stack)
        }
    }()
    
    f()
    return nil
}

// SafeGo - รัน goroutine โดย catch panic
func SafeGo(f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Printf("goroutine panic recovered: %v\n", r)
            }
        }()
        f()
    }()
}

// Middleware pattern สำหรับ HTTP handler
type HandlerFunc func(w ResponseWriter, r *Request)

type ResponseWriter interface {
    WriteHeader(status int)
    Write(body string)
}

type Request struct {
    Path   string
    Method string
}

type MockResponseWriter struct {
    status int
    body   string
}

func (m *MockResponseWriter) WriteHeader(status int) { m.status = status }
func (m *MockResponseWriter) Write(body string)      { m.body = body }

func RecoveryMiddleware(handler HandlerFunc) HandlerFunc {
    return func(w ResponseWriter, r *Request) {
        defer func() {
            if rec := recover(); rec != nil {
                fmt.Printf("Handler panic: %v\n", rec)
                w.WriteHeader(500)
                w.Write("Internal Server Error")
            }
        }()
        
        handler(w, r)
    }
}

func main() {
    // SafeRun
    err := SafeRun(func() {
        fmt.Println("Normal execution")
    })
    fmt.Printf("SafeRun normal: err = %v\n", err)
    
    err = SafeRun(func() {
        panic("something went wrong!")
    })
    fmt.Printf("SafeRun panic: err = %v\n", err)
    
    err = SafeRun(func() {
        var s []int
        _ = s[0] // runtime panic
    })
    fmt.Printf("SafeRun runtime panic: err = %v\n", err)
    
    // SafeGo
    fmt.Println("\n--- SafeGo ---")
    ch := make(chan struct{})
    SafeGo(func() {
        defer func() { close(ch) }()
        panic("goroutine panic!")
    })
    <-ch
    
    // HTTP Recovery Middleware
    fmt.Println("\n--- Recovery Middleware ---")
    
    panicHandler := func(w ResponseWriter, r *Request) {
        panic(fmt.Sprintf("handler panic for %s", r.Path))
    }
    
    safeHandler := RecoveryMiddleware(panicHandler)
    
    w := &MockResponseWriter{}
    r := &Request{Path: "/api/users", Method: "GET"}
    safeHandler(w, r)
    fmt.Printf("Response status: %d\n", w.status)
    fmt.Printf("Response body: %s\n", w.body)
}
```

### 20.5.3 re-panic pattern

```go
package main

import (
    "fmt"
)

// บางครั้งต้อง re-panic หลัง log/cleanup
func processWithLogging(f func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("LOG: panic occurred: %v\n", r)
            panic(r) // re-panic! ส่งต่อไปให้ caller จัดการ
        }
    }()
    
    f()
}

// recover เฉพาะ panic ที่คาดไว้
type AppError struct {
    Code    int
    Message string
}

func (e AppError) Error() string {
    return fmt.Sprintf("AppError[%d]: %s", e.Code, e.Message)
}

func safeExecute(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            // ดักเฉพาะ AppError
            if appErr, ok := r.(AppError); ok {
                err = appErr
                return
            }
            // re-panic สำหรับ panic อื่นๆ
            panic(r)
        }
    }()
    
    f()
    return nil
}

func main() {
    // re-panic example
    fmt.Println("=== re-panic ===")
    
    func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Printf("outer recover: %v\n", r)
            }
        }()
        
        processWithLogging(func() {
            panic("critical error!")
        })
    }()
    
    // selective recover
    fmt.Println("\n=== selective recover ===")
    
    // AppError → return as error
    err := safeExecute(func() {
        panic(AppError{Code: 404, Message: "not found"})
    })
    fmt.Printf("AppError result: %v\n", err)
    
    // Other panic → re-panic
    func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Printf("caught re-panicked: %v\n", r)
            }
        }()
        
        err2 := safeExecute(func() {
            panic("unexpected panic!")
        })
        _ = err2
    }()
    
    fmt.Println("Done!")
}
```

---

## 20.6 Panic vs Error

### 20.6.1 เมื่อไหรใช้อะไร

```go
package main

import (
    "errors"
    "fmt"
    "strconv"
)

// ==== ใช้ Error ====

// อ่านไฟล์ - fail เพราะ runtime condition
func readConfig(path string) (map[string]string, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config %q: %w", path, err)
    }
    // parse...
    return map[string]string{}, nil
}

// Network request - fail เพราะ external condition
func fetchUser(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("invalid user id: %d", id)
    }
    // HTTP request...
    return &User{ID: id}, nil
}

// User input validation
func parseAndValidate(input string) (int, error) {
    n, err := strconv.Atoi(input)
    if err != nil {
        return 0, fmt.Errorf("invalid number %q: %w", input, err)
    }
    if n < 0 {
        return 0, fmt.Errorf("number must be positive, got %d", n)
    }
    return n, nil
}

// ==== ใช้ Panic ====

// Static initialization - ต้องสำเร็จเสมอ
var (
    emailRegexp = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
    phoneRegexp = regexp.MustCompile(`^0[6-9]\d{8}$`)
)

// เข้าถึง required environment variable
func mustGetEnv(key string) string {
    val := os.Getenv(key)
    if val == "" {
        panic(fmt.Sprintf("required environment variable %q is not set", key))
    }
    return val
}

// Programmer contract violation
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Pop() T {
    if len(s.items) == 0 {
        panic("pop from empty stack") // programming error
    }
    last := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return last
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

type User struct {
    ID   int
    Name string
}

func main() {
    // Error handling
    n, err := parseAndValidate("42")
    if err != nil {
        fmt.Printf("error: %v\n", err)
    } else {
        fmt.Printf("parsed: %d\n", n)
    }
    
    _, err = parseAndValidate("abc")
    fmt.Printf("parse error: %v\n", err)
    
    // Panic (Static init)
    fmt.Printf("email valid: %v\n", emailRegexp.MatchString("test@example.com"))
    
    // Stack with panic
    s := &Stack[int]{}
    s.Push(1)
    s.Push(2)
    s.Push(3)
    
    for len(s.items) > 0 {
        fmt.Printf("pop: %d\n", s.Pop())
    }
    
    // ดักจับ panic จาก Stack
    err = SafeRun(func() {
        s.Pop() // empty stack → panic
    })
    fmt.Printf("empty pop error: %v\n", err)
}

func SafeRun(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    f()
    return nil
}
```

---

## 20.7 Best Practices

### 20.7.1 Pattern Collection

```go
package main

import (
    "fmt"
    "os"
    "sync"
    "time"
)

// Pattern 1: Resource Cleanup
type Resource struct {
    name   string
    closed bool
}

func (r *Resource) Close() {
    r.closed = true
    fmt.Printf("Resource %q closed\n", r.name)
}

func useResource() error {
    r := &Resource{name: "db-connection"}
    defer r.Close()
    
    // ใช้งาน resource...
    fmt.Printf("Using resource %q\n", r.name)
    return nil
}

// Pattern 2: Timing
func timeit(name string) func() {
    start := time.Now()
    return func() {
        elapsed := time.Since(start)
        fmt.Printf("%s took %v\n", name, elapsed)
    }
}

func expensiveOperation() {
    defer timeit("expensiveOperation")()
    
    // simulate work
    time.Sleep(10 * time.Millisecond)
    fmt.Println("Work done")
}

// Pattern 3: Panic-safe goroutine pool
type WorkerPool struct {
    workers int
    jobs    chan func()
    wg      sync.WaitGroup
}

func NewWorkerPool(workers int) *WorkerPool {
    p := &WorkerPool{
        workers: workers,
        jobs:    make(chan func(), 100),
    }
    p.start()
    return p
}

func (p *WorkerPool) start() {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go func(workerID int) {
            defer p.wg.Done()
            defer func() {
                if r := recover(); r != nil {
                    fmt.Printf("Worker %d recovered: %v\n", workerID, r)
                }
            }()
            
            for job := range p.jobs {
                func() {
                    defer func() {
                        if r := recover(); r != nil {
                            fmt.Printf("Job panic: %v\n", r)
                        }
                    }()
                    job()
                }()
            }
        }(i + 1)
    }
}

func (p *WorkerPool) Submit(job func()) {
    p.jobs <- job
}

func (p *WorkerPool) Close() {
    close(p.jobs)
    p.wg.Wait()
}

// Pattern 4: Defer สำหรับ log
func withLogging(name string, f func() error) error {
    fmt.Printf("[START] %s\n", name)
    
    var err error
    defer func() {
        if err != nil {
            fmt.Printf("[FAIL] %s: %v\n", name, err)
        } else {
            fmt.Printf("[SUCCESS] %s\n", name)
        }
    }()
    
    err = f()
    return err
}

func main() {
    fmt.Println("=== Resource Cleanup ===")
    useResource()
    
    fmt.Println("\n=== Timing ===")
    expensiveOperation()
    
    fmt.Println("\n=== Worker Pool ===")
    pool := NewWorkerPool(3)
    
    for i := 1; i <= 5; i++ {
        i := i
        pool.Submit(func() {
            if i == 3 {
                panic(fmt.Sprintf("job %d failed!", i))
            }
            fmt.Printf("Job %d completed\n", i)
        })
    }
    
    pool.Close()
    
    fmt.Println("\n=== Logging Pattern ===")
    withLogging("database query", func() error {
        fmt.Println("  executing query...")
        return nil
    })
    
    withLogging("risky operation", func() error {
        return fmt.Errorf("connection refused")
    })
    
    // ตัวอย่าง defer order
    fmt.Println("\n=== Defer Order Demo ===")
    func() {
        defer fmt.Println("1: first defer")
        defer fmt.Println("2: second defer")
        defer fmt.Println("3: third defer")
        fmt.Println("function body")
    }()
}
```

---

## 20.8 Workshop: Safe Function Executor

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "runtime/debug"
    "sync"
    "time"
)

// PanicError wraps a panic value as an error
type PanicError struct {
    Value interface{}
    Stack []byte
}

func (e *PanicError) Error() string {
    return fmt.Sprintf("panic: %v\n\nstack:\n%s", e.Value, e.Stack)
}

// Executor - รัน functions อย่างปลอดภัย
type Executor struct {
    mu       sync.Mutex
    running  map[string]bool
    timeout  time.Duration
    maxRetry int
}

func NewExecutor(timeout time.Duration, maxRetry int) *Executor {
    return &Executor{
        running:  make(map[string]bool),
        timeout:  timeout,
        maxRetry: maxRetry,
    }
}

// Run - รัน function พร้อม panic recovery
func (e *Executor) Run(name string, f func() error) error {
    return e.runWithContext(context.Background(), name, f)
}

// RunWithContext - รัน function พร้อม context
func (e *Executor) RunWithContext(ctx context.Context, name string, f func() error) error {
    return e.runWithContext(ctx, name, f)
}

func (e *Executor) runWithContext(ctx context.Context, name string, f func() error) error {
    // ตรวจสอบว่า name ซ้ำกันหรือไม่
    e.mu.Lock()
    if e.running[name] {
        e.mu.Unlock()
        return fmt.Errorf("task %q is already running", name)
    }
    e.running[name] = true
    e.mu.Unlock()
    
    defer func() {
        e.mu.Lock()
        delete(e.running, name)
        e.mu.Unlock()
    }()
    
    // รันพร้อม timeout
    type result struct {
        err error
    }
    
    resultCh := make(chan result, 1)
    
    go func() {
        resultCh <- result{e.safeRun(f)}
    }()
    
    var timeout <-chan time.Time
    if e.timeout > 0 {
        timeout = time.After(e.timeout)
    }
    
    select {
    case <-ctx.Done():
        return fmt.Errorf("task %q: %w", name, ctx.Err())
    case <-timeout:
        return fmt.Errorf("task %q: timeout after %v", name, e.timeout)
    case r := <-resultCh:
        return r.err
    }
}

func (e *Executor) safeRun(f func() error) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = &PanicError{
                Value: r,
                Stack: debug.Stack(),
            }
        }
    }()
    
    return f()
}

// RunWithRetry - รันพร้อม retry
func (e *Executor) RunWithRetry(name string, f func() error) error {
    var lastErr error
    
    for attempt := 1; attempt <= e.maxRetry; attempt++ {
        err := e.safeRun(f)
        if err == nil {
            if attempt > 1 {
                fmt.Printf("Task %q succeeded on attempt %d\n", name, attempt)
            }
            return nil
        }
        
        lastErr = err
        
        // ไม่ retry ถ้าเป็น panic
        var panicErr *PanicError
        if errors.As(err, &panicErr) {
            return fmt.Errorf("task %q panicked (no retry): %w", name, err)
        }
        
        if attempt < e.maxRetry {
            fmt.Printf("Task %q attempt %d failed: %v, retrying...\n",
                name, attempt, err)
            time.Sleep(time.Duration(attempt) * 100 * time.Millisecond)
        }
    }
    
    return fmt.Errorf("task %q failed after %d attempts: %w",
        name, e.maxRetry, lastErr)
}

func main() {
    exec := NewExecutor(5*time.Second, 3)
    
    // รัน function ปกติ
    fmt.Println("=== Normal execution ===")
    err := exec.Run("normal", func() error {
        fmt.Println("Normal task executing...")
        return nil
    })
    fmt.Printf("Result: err=%v\n", err)
    
    // รัน function ที่ return error
    fmt.Println("\n=== Error execution ===")
    err = exec.Run("erroring", func() error {
        return fmt.Errorf("something failed")
    })
    fmt.Printf("Result: err=%v\n", err)
    
    // รัน function ที่ panic
    fmt.Println("\n=== Panic execution ===")
    err = exec.Run("panicking", func() error {
        panic("unexpected condition!")
    })
    var panicErr *PanicError
    if errors.As(err, &panicErr) {
        fmt.Printf("Caught panic: %v\n", panicErr.Value)
    } else {
        fmt.Printf("Result: err=%v\n", err)
    }
    
    // รันพร้อม retry
    fmt.Println("\n=== Retry execution ===")
    attempt := 0
    err = exec.RunWithRetry("flaky", func() error {
        attempt++
        if attempt < 3 {
            return fmt.Errorf("temporary error (attempt %d)", attempt)
        }
        return nil
    })
    fmt.Printf("Result after %d attempts: err=%v\n", attempt, err)
    
    // รันพร้อม context timeout
    fmt.Println("\n=== Context timeout ===")
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()
    
    err = exec.RunWithContext(ctx, "slow", func() error {
        time.Sleep(100 * time.Millisecond)
        return nil
    })
    fmt.Printf("Result: err=%v\n", err)
    
    // ป้องกัน concurrent execution ของ task เดียวกัน
    fmt.Println("\n=== Concurrent prevention ===")
    
    startCh := make(chan struct{})
    doneCh := make(chan struct{}, 2)
    
    for i := 0; i < 2; i++ {
        go func(id int) {
            <-startCh
            err := exec.Run("unique-task", func() error {
                time.Sleep(50 * time.Millisecond)
                return nil
            })
            fmt.Printf("Goroutine %d: err=%v\n", id, err)
            doneCh <- struct{}{}
        }(i + 1)
    }
    
    close(startCh)
    <-doneCh
    <-doneCh
    
    fmt.Println("\nAll tasks completed!")
}
```

---

## สรุป

| Feature | การใช้งาน |
|---------|----------|
| `defer` | Cleanup (close, unlock, rollback) |
| `defer` + named return | เปลี่ยน return value |
| `panic` | Programmer errors, impossible states, Must functions |
| `recover` | ดักจับ panic เพื่อ graceful handling |
| defer LIFO order | สุดท้ายที่ defer ทำงานก่อน |

### Do's and Don'ts

| Do | Don't |
|----|-------|
| ใช้ panic สำหรับ programming errors | ใช้ panic แทน error returns |
| ใช้ recover ใน server/middleware | ใช้ recover ใน library code |
| defer สำหรับ cleanup | defer ใน tight loop |
| Must pattern สำหรับ static init | ignore recovered values |

## Resources

- [defer, panic, and recover](https://go.dev/blog/defer-panic-and-recover)
- [Go blog: Defer](https://go.dev/doc/effective_go#defer)
- [Error handling in Go](https://go.dev/blog/error-handling-and-go)
