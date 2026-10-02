# Part 24: Context ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ `context.Context` interface
- ใช้ `context.Background()` และ `context.TODO()`
- ใช้ `context.WithCancel` สำหรับ cancellation
- ใช้ `context.WithTimeout` และ `context.WithDeadline`
- ใช้ `context.WithValue` ส่งข้อมูลผ่าน call chain
- ส่ง context ผ่าน function chain
- ใช้ context ใน HTTP handlers
- รู้จัก best practices

---

## 24.1 Context Interface

### 24.1.1 context.Context

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// context.Context interface:
// type Context interface {
//     Deadline() (deadline time.Time, ok bool)
//     Done() <-chan struct{}
//     Err() error
//     Value(key any) any
// }

func main() {
    // Background - root context (ไม่มี deadline, cancel, หรือ value)
    bg := context.Background()
    fmt.Printf("Background: %v\n", bg)
    
    deadline, ok := bg.Deadline()
    fmt.Printf("Has deadline: %v, deadline: %v\n", ok, deadline)
    fmt.Printf("Done channel: %v\n", bg.Done())
    fmt.Printf("Error: %v\n", bg.Err())
    
    // TODO - เหมือน Background แต่บอกว่ายังไม่รู้จะใช้ context อะไร
    todo := context.TODO()
    fmt.Printf("\nTODO: %v\n", todo)
    
    // ตรวจสอบ context
    ctx, cancel := context.WithTimeout(bg, 5*time.Second)
    defer cancel()
    
    deadline2, ok2 := ctx.Deadline()
    fmt.Printf("\nWithTimeout:\n")
    fmt.Printf("  Has deadline: %v\n", ok2)
    fmt.Printf("  Deadline: %v\n", deadline2.Format("15:04:05.000"))
    fmt.Printf("  Time remaining: %v\n", time.Until(deadline2).Round(time.Millisecond))
}
```

---

## 24.2 context.WithCancel

### 24.2.1 Cancellation

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func doWork(ctx context.Context, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d cancelled: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d doing work...\n", id)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    var wg sync.WaitGroup
    
    // เริ่ม workers
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go doWork(ctx, i, &wg)
    }
    
    // ทำงาน 2 วินาทีแล้ว cancel
    time.Sleep(2 * time.Second)
    fmt.Println("\n=== Cancelling! ===")
    cancel() // ส่งสัญญาณ cancel ไปยังทุก goroutine
    
    wg.Wait()
    fmt.Println("All workers stopped")
    
    // ตรวจสอบ context หลัง cancel
    fmt.Printf("Context error: %v\n", ctx.Err())
}
```

### 24.2.2 Cancel Propagation

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func level3(ctx context.Context) {
    ticker := time.NewTicker(200 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Level 3 cancelled:", ctx.Err())
            return
        case t := <-ticker.C:
            fmt.Printf("Level 3 working at %s\n", t.Format("15:04:05.000"))
        }
    }
}

func level2(ctx context.Context) {
    fmt.Println("Level 2 started")
    defer fmt.Println("Level 2 done")
    
    // สร้าง child context
    childCtx, cancel := context.WithCancel(ctx)
    defer cancel() // cancel child เมื่อ level2 return
    
    go level3(childCtx)
    
    select {
    case <-ctx.Done():
        fmt.Println("Level 2 parent cancelled:", ctx.Err())
        return
    case <-time.After(1 * time.Second):
        fmt.Println("Level 2 timed out on its own")
        return
    }
}

func level1(ctx context.Context) {
    fmt.Println("Level 1 started")
    defer fmt.Println("Level 1 done")
    
    go level2(ctx)
    
    select {
    case <-ctx.Done():
        fmt.Println("Level 1 parent cancelled:", ctx.Err())
        return
    case <-time.After(2 * time.Second):
        fmt.Println("Level 1 timed out on its own")
        return
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go level1(ctx)
    
    time.Sleep(500 * time.Millisecond)
    fmt.Println("\n=== Cancelling from root ===")
    cancel()
    
    time.Sleep(500 * time.Millisecond)
    fmt.Println("Done")
}
```

---

## 24.3 context.WithTimeout

### 24.3.1 Timeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func fetchData(ctx context.Context, url string) (string, error) {
    // Simulate HTTP request
    fmt.Printf("Fetching %s...\n", url)
    
    select {
    case <-ctx.Done():
        return "", fmt.Errorf("fetch cancelled: %w", ctx.Err())
    case <-time.After(2 * time.Second): // simulate 2s request
        return fmt.Sprintf("data from %s", url), nil
    }
}

func processWithTimeout(url string, timeout time.Duration) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel() // ดี practice: cancel เสมอแม้จะ timeout ก่อน
    
    result, err := fetchData(ctx, url)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Result: %s\n", result)
}

func main() {
    fmt.Println("=== Successful request (5s timeout) ===")
    processWithTimeout("https://api.example.com/data", 5*time.Second)
    
    fmt.Println("\n=== Failed request (1s timeout) ===")
    processWithTimeout("https://api.example.com/slow", 1*time.Second)
    
    // ตัวอย่างที่ดีกว่า: ส่ง context ตลอด call chain
    fmt.Println("\n=== Proper context chain ===")
    
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    data, err := fetchData(ctx, "https://api.example.com/users")
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            fmt.Println("Request timed out!")
        } else if ctx.Err() == context.Canceled {
            fmt.Println("Request was cancelled!")
        } else {
            fmt.Printf("Error: %v\n", err)
        }
        return
    }
    fmt.Println("Got:", data)
}
```

### 24.3.2 Timeout ใน Pipeline

```go
package main

import (
    "context"
    "fmt"
    "time"
)

type Result struct {
    Data  string
    Error error
}

func step1(ctx context.Context, input string) (string, error) {
    select {
    case <-ctx.Done():
        return "", fmt.Errorf("step1 cancelled: %w", ctx.Err())
    case <-time.After(300 * time.Millisecond):
        return "processed_" + input, nil
    }
}

func step2(ctx context.Context, input string) (string, error) {
    select {
    case <-ctx.Done():
        return "", fmt.Errorf("step2 cancelled: %w", ctx.Err())
    case <-time.After(500 * time.Millisecond):
        return "validated_" + input, nil
    }
}

func step3(ctx context.Context, input string) (string, error) {
    select {
    case <-ctx.Done():
        return "", fmt.Errorf("step3 cancelled: %w", ctx.Err())
    case <-time.After(200 * time.Millisecond):
        return "saved_" + input, nil
    }
}

func pipeline(ctx context.Context, input string) (string, error) {
    // แต่ละ step ได้รับ deadline เดียวกัน
    result1, err := step1(ctx, input)
    if err != nil {
        return "", err
    }
    
    result2, err := step2(ctx, result1)
    if err != nil {
        return "", err
    }
    
    result3, err := step3(ctx, result2)
    if err != nil {
        return "", err
    }
    
    return result3, nil
}

func main() {
    // สำเร็จ (timeout มากกว่า 1s)
    fmt.Println("=== With 2s timeout ===")
    ctx1, cancel1 := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel1()
    
    if result, err := pipeline(ctx1, "data"); err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Result: %s\n", result)
    }
    
    // ล้มเหลว (timeout น้อยกว่า total time)
    fmt.Println("\n=== With 700ms timeout ===")
    ctx2, cancel2 := context.WithTimeout(context.Background(), 700*time.Millisecond)
    defer cancel2()
    
    if result, err := pipeline(ctx2, "data"); err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Result: %s\n", result)
    }
}
```

---

## 24.4 context.WithDeadline

### 24.4.1 Deadline

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func doExpensiveWork(ctx context.Context) error {
    fmt.Println("Starting expensive work...")
    
    for i := 0; i < 10; i++ {
        // ตรวจสอบ context ทุกๆ iteration
        select {
        case <-ctx.Done():
            return fmt.Errorf("work cancelled at step %d: %w", i, ctx.Err())
        default:
        }
        
        fmt.Printf("Step %d...\n", i+1)
        time.Sleep(200 * time.Millisecond)
    }
    
    return nil
}

func main() {
    // WithDeadline - กำหนดเวลาสิ้นสุดแบบ absolute
    deadline := time.Now().Add(1 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    
    fmt.Printf("Deadline: %s\n", deadline.Format("15:04:05.000"))
    fmt.Printf("Starting at: %s\n", time.Now().Format("15:04:05.000"))
    
    err := doExpensiveWork(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Println("Work completed!")
    }
    
    // ตรวจสอบ deadline
    if d, ok := ctx.Deadline(); ok {
        fmt.Printf("Was deadline: %v\n", d.Format("15:04:05.000"))
        fmt.Printf("Exceeded by: %v\n", time.Since(d))
    }
    
    // WithTimeout vs WithDeadline
    // WithTimeout(ctx, 5*time.Second)
    // ≡ WithDeadline(ctx, time.Now().Add(5*time.Second))
    
    // ใช้ WithDeadline เมื่อต้องการ absolute time
    // ใช้ WithTimeout เมื่อต้องการ relative time
    
    // ตัวอย่าง: process ทั้งหมดต้องเสร็จภายใน EOD
    eod := time.Now().Add(8 * time.Hour)
    ctx2, cancel2 := context.WithDeadline(context.Background(), eod)
    defer cancel2()
    
    dl, _ := ctx2.Deadline()
    fmt.Printf("\nEOD deadline: %s (in %v)\n",
        dl.Format("15:04:05"),
        time.Until(dl).Round(time.Hour))
}
```

---

## 24.5 context.WithValue

### 24.5.1 ส่งค่าผ่าน Context

```go
package main

import (
    "context"
    "fmt"
)

// Best practice: ใช้ unexported key type เพื่อป้องกัน collision
type contextKey int

const (
    requestIDKey contextKey = iota
    userIDKey
    traceIDKey
)

// หรือใช้ struct key
type contextKeyString struct{ name string }

var (
    RequestIDKey = contextKeyString{"request-id"}
    UserIDKey    = contextKeyString{"user-id"}
    TraceIDKey   = contextKeyString{"trace-id"}
)

// Helper functions
func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, RequestIDKey, id)
}

func GetRequestID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(RequestIDKey).(string)
    return id, ok
}

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, UserIDKey, id)
}

func GetUserID(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(UserIDKey).(int)
    return id, ok
}

// Middleware-style usage
func handler(ctx context.Context) {
    reqID, _ := GetRequestID(ctx)
    userID, _ := GetUserID(ctx)
    
    fmt.Printf("Handler: requestID=%s, userID=%d\n", reqID, userID)
    
    // ส่งต่อไปยัง service
    service(ctx)
}

func service(ctx context.Context) {
    reqID, _ := GetRequestID(ctx)
    userID, _ := GetUserID(ctx)
    
    fmt.Printf("Service: requestID=%s, userID=%d\n", reqID, userID)
    
    // ส่งต่อไปยัง repository
    repository(ctx)
}

func repository(ctx context.Context) {
    reqID, _ := GetRequestID(ctx)
    
    fmt.Printf("Repository: requestID=%s\n", reqID)
    fmt.Println("Executing query with request tracing...")
}

func main() {
    // สร้าง request context
    ctx := context.Background()
    ctx = WithRequestID(ctx, "req-abc123")
    ctx = WithUserID(ctx, 42)
    
    // เรียก handler
    handler(ctx)
    
    // ตัวอย่าง: nested context values
    fmt.Println("\n=== Nested Values ===")
    
    parent := context.WithValue(context.Background(), "key1", "value1")
    child := context.WithValue(parent, "key2", "value2")
    grandchild := context.WithValue(child, "key1", "override") // override parent
    
    fmt.Println("grandchild key1:", grandchild.Value("key1")) // override
    fmt.Println("grandchild key2:", grandchild.Value("key2")) // from child
    fmt.Println("child key1:", child.Value("key1"))           // from parent
    fmt.Println("parent key2:", parent.Value("key2"))          // nil (not found)
}
```

### 24.5.2 Context Values Best Practices

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Pattern: Context Values สำหรับ Request Metadata

type RequestMetadata struct {
    RequestID  string
    UserID     int
    ClientIP   string
    UserAgent  string
    StartTime  time.Time
    TraceID    string
}

type metadataKey struct{}

func WithMetadata(ctx context.Context, meta RequestMetadata) context.Context {
    return context.WithValue(ctx, metadataKey{}, meta)
}

func GetMetadata(ctx context.Context) (RequestMetadata, bool) {
    meta, ok := ctx.Value(metadataKey{}).(RequestMetadata)
    return meta, ok
}

func logRequest(ctx context.Context, message string) {
    if meta, ok := GetMetadata(ctx); ok {
        elapsed := time.Since(meta.StartTime)
        fmt.Printf("[%s] [trace:%s] [user:%d] [%.2fms] %s\n",
            meta.RequestID, meta.TraceID, meta.UserID,
            elapsed.Seconds()*1000, message)
    } else {
        fmt.Println(message)
    }
}

func processOrder(ctx context.Context, orderID int) error {
    logRequest(ctx, fmt.Sprintf("Processing order %d", orderID))
    
    // Simulate work
    time.Sleep(10 * time.Millisecond)
    
    logRequest(ctx, fmt.Sprintf("Order %d processed successfully", orderID))
    return nil
}

func main() {
    meta := RequestMetadata{
        RequestID: "req-001",
        UserID:    123,
        ClientIP:  "192.168.1.100",
        UserAgent: "Mozilla/5.0",
        StartTime: time.Now(),
        TraceID:   "trace-abc",
    }
    
    ctx := WithMetadata(context.Background(), meta)
    
    // ทุก function ในสาย ใช้ context เดียวกัน
    logRequest(ctx, "Request received")
    processOrder(ctx, 456)
    logRequest(ctx, "Request completed")
    
    // สิ่งที่ไม่ควรใส่ใน context:
    // - Optional parameters (ใส่ใน function signature แทน)
    // - Mutable state (ใช้ struct แทน)
    // - Database connections (ส่งผ่าน dependency injection)
    
    fmt.Println("\n=== What NOT to put in context ===")
    fmt.Println("// Don't do this:")
    fmt.Println("// ctx = context.WithValue(ctx, \"db\", database)  // use DI instead")
    fmt.Println("// ctx = context.WithValue(ctx, \"timeout\", 30)   // use WithTimeout instead")
}
```

---

## 24.6 Context ใน HTTP Handlers

### 24.6.1 HTTP Context

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "net/http/httptest"
    "time"
)

type contextKeyType int

const (
    requestIDCtxKey contextKeyType = iota
    authUserCtxKey
)

type AuthUser struct {
    ID    int
    Name  string
    Role  string
    Token string
}

// Middleware: เพิ่ม request ID
func requestIDMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        requestID := fmt.Sprintf("req-%d", time.Now().UnixNano())
        ctx := context.WithValue(r.Context(), requestIDCtxKey, requestID)
        next(w, r.WithContext(ctx))
    }
}

// Middleware: Authentication
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Simulate token validation
        user := &AuthUser{
            ID:    1,
            Name:  "Alice",
            Role:  "admin",
            Token: token,
        }
        
        ctx := context.WithValue(r.Context(), authUserCtxKey, user)
        next(w, r.WithContext(ctx))
    }
}

// Middleware: Timeout
func timeoutMiddleware(timeout time.Duration) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            
            next(w, r.WithContext(ctx))
        }
    }
}

// Handler ที่ใช้ context
func usersHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // ดึง request ID
    reqID, _ := ctx.Value(requestIDCtxKey).(string)
    
    // ดึง authenticated user
    user, ok := ctx.Value(authUserCtxKey).(*AuthUser)
    if !ok {
        http.Error(w, "No auth user in context", http.StatusInternalServerError)
        return
    }
    
    // Simulate database query with context
    users, err := queryUsers(ctx)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            http.Error(w, "Query timed out", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    fmt.Fprintf(w, "RequestID: %s\nUser: %s (%s)\nUsers: %v",
        reqID, user.Name, user.Role, users)
}

func queryUsers(ctx context.Context) ([]string, error) {
    // ตรวจสอบ context ก่อน query
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Simulate query time
    time.Sleep(50 * time.Millisecond)
    
    // ตรวจสอบอีกครั้งหลัง query
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    return []string{"Alice", "Bob", "Charlie"}, nil
}

func main() {
    // สร้าง handler chain
    handler := requestIDMiddleware(
        authMiddleware(
            timeoutMiddleware(5 * time.Second)(usersHandler),
        ),
    )
    
    // Test server
    server := httptest.NewServer(handler)
    defer server.Close()
    
    // ทดสอบ request สำเร็จ
    req, _ := http.NewRequest("GET", server.URL+"/users", nil)
    req.Header.Set("Authorization", "Bearer abc123")
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        fmt.Printf("Request error: %v\n", err)
        return
    }
    defer resp.Body.Close()
    
    body := make([]byte, 1024)
    n, _ := resp.Body.Read(body)
    fmt.Printf("Response (%d):\n%s\n", resp.StatusCode, body[:n])
    
    // ทดสอบ request ไม่มี auth
    req2, _ := http.NewRequest("GET", server.URL+"/users", nil)
    resp2, _ := http.DefaultClient.Do(req2)
    fmt.Printf("\nNo auth response: %d\n", resp2.StatusCode)
    resp2.Body.Close()
}
```

---

## 24.7 Advanced Context Patterns

### 24.7.1 Context Cancellation Fan-out

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type DataResult struct {
    Source string
    Data   string
    Error  error
}

func fetchFromSource(ctx context.Context, source string, delay time.Duration) DataResult {
    select {
    case <-ctx.Done():
        return DataResult{Source: source, Error: ctx.Err()}
    case <-time.After(delay):
        return DataResult{Source: source, Data: fmt.Sprintf("data from %s", source)}
    }
}

// แรกที่สำเร็จ ยกเลิกที่เหลือ
func raceRequest(ctx context.Context, sources []string) (DataResult, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    resultCh := make(chan DataResult, len(sources))
    
    for i, source := range sources {
        source := source
        delay := time.Duration(i+1) * 300 * time.Millisecond
        
        go func() {
            result := fetchFromSource(ctx, source, delay)
            resultCh <- result
        }()
    }
    
    for i := 0; i < len(sources); i++ {
        result := <-resultCh
        if result.Error == nil {
            cancel() // ยกเลิกที่เหลือ
            return result, nil
        }
    }
    
    return DataResult{}, fmt.Errorf("all sources failed")
}

// รอทุกแหล่งข้อมูล
func gatherAll(ctx context.Context, sources []string) ([]DataResult, error) {
    var wg sync.WaitGroup
    results := make([]DataResult, len(sources))
    
    for i, source := range sources {
        wg.Add(1)
        i, source := i, source
        
        go func() {
            defer wg.Done()
            delay := time.Duration(i+1) * 200 * time.Millisecond
            results[i] = fetchFromSource(ctx, source, delay)
        }()
    }
    
    wg.Wait()
    
    // ตรวจสอบ errors
    var errors []string
    for _, r := range results {
        if r.Error != nil {
            errors = append(errors, fmt.Sprintf("%s: %v", r.Source, r.Error))
        }
    }
    
    if len(errors) > 0 {
        return results, fmt.Errorf("some sources failed: %v", errors)
    }
    
    return results, nil
}

func main() {
    sources := []string{"server1", "server2", "server3"}
    
    // Race - แรกที่สำเร็จ
    fmt.Println("=== Race ===")
    ctx1, cancel1 := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel1()
    
    result, err := raceRequest(ctx1, sources)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Winner: %s - %s\n", result.Source, result.Data)
    }
    
    // Gather all
    fmt.Println("\n=== Gather All ===")
    ctx2, cancel2 := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel2()
    
    allResults, err := gatherAll(ctx2, sources)
    if err != nil {
        fmt.Printf("Warning: %v\n", err)
    }
    
    for _, r := range allResults {
        if r.Error != nil {
            fmt.Printf("  %s: ERROR - %v\n", r.Source, r.Error)
        } else {
            fmt.Printf("  %s: %s\n", r.Source, r.Data)
        }
    }
    
    // Timeout too short
    fmt.Println("\n=== Timeout too short ===")
    ctx3, cancel3 := context.WithTimeout(context.Background(), 200*time.Millisecond)
    defer cancel3()
    
    _, err = raceRequest(ctx3, sources)
    if err != nil {
        fmt.Printf("Expected error: %v\n", err)
    }
}
```

---

## 24.8 Workshop: Context-aware Service

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "time"
)

// Errors
var (
    ErrNotFound   = errors.New("not found")
    ErrTimeout    = errors.New("operation timed out")
    ErrCancelled  = errors.New("operation cancelled")
    ErrUnauthorized = errors.New("unauthorized")
)

// Models
type User struct {
    ID    int
    Name  string
    Email string
    Role  string
}

type Order struct {
    ID     int
    UserID int
    Total  float64
    Status string
    Items  []string
}

// Context keys
type ctxKey int

const (
    ctxKeyRequestID ctxKey = iota
    ctxKeyUserID
    ctxKeyTenantID
    ctxKeyLogger
)

type Logger struct {
    prefix string
}

func (l *Logger) Log(format string, args ...interface{}) {
    fmt.Printf("[%s] %s\n", l.prefix, fmt.Sprintf(format, args...))
}

// Context helpers
func WithRequestContext(ctx context.Context, requestID string, userID int) context.Context {
    ctx = context.WithValue(ctx, ctxKeyRequestID, requestID)
    ctx = context.WithValue(ctx, ctxKeyUserID, userID)
    return ctx
}

func contextLogger(ctx context.Context) *Logger {
    if l, ok := ctx.Value(ctxKeyLogger).(*Logger); ok {
        return l
    }
    return &Logger{prefix: "default"}
}

// Database layer
type DB struct {
    mu    sync.RWMutex
    users  map[int]*User
    orders map[int]*Order
}

func NewDB() *DB {
    return &DB{
        users: map[int]*User{
            1: {ID: 1, Name: "Alice", Email: "alice@example.com", Role: "admin"},
            2: {ID: 2, Name: "Bob", Email: "bob@example.com", Role: "user"},
            3: {ID: 3, Name: "Charlie", Email: "charlie@example.com", Role: "user"},
        },
        orders: map[int]*Order{
            101: {ID: 101, UserID: 1, Total: 350.0, Status: "completed", Items: []string{"iPhone", "Case"}},
            102: {ID: 102, UserID: 2, Total: 150.0, Status: "pending", Items: []string{"Book"}},
        },
    }
}

func (db *DB) GetUser(ctx context.Context, id int) (*User, error) {
    // ตรวจ context
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("GetUser: %w", ctx.Err())
    default:
    }
    
    // Simulate DB latency
    time.Sleep(20 * time.Millisecond)
    
    // ตรวจ context หลัง query
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("GetUser: %w", ctx.Err())
    default:
    }
    
    db.mu.RLock()
    defer db.mu.RUnlock()
    
    user, ok := db.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

func (db *DB) GetUserOrders(ctx context.Context, userID int) ([]*Order, error) {
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("GetUserOrders: %w", ctx.Err())
    default:
    }
    
    time.Sleep(30 * time.Millisecond)
    
    db.mu.RLock()
    defer db.mu.RUnlock()
    
    var orders []*Order
    for _, o := range db.orders {
        if o.UserID == userID {
            orders = append(orders, o)
        }
    }
    return orders, nil
}

func (db *DB) CreateOrder(ctx context.Context, order Order) (*Order, error) {
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("CreateOrder: %w", ctx.Err())
    default:
    }
    
    time.Sleep(50 * time.Millisecond)
    
    db.mu.Lock()
    defer db.mu.Unlock()
    
    order.ID = len(db.orders) + 1
    db.orders[order.ID] = &order
    return &order, nil
}

// Service layer
type UserService struct {
    db *DB
}

func NewUserService(db *DB) *UserService {
    return &UserService{db: db}
}

func (s *UserService) GetProfile(ctx context.Context, userID int) (*User, error) {
    logger := contextLogger(ctx)
    
    reqID, _ := ctx.Value(ctxKeyRequestID).(string)
    logger.Log("[%s] Getting profile for user %d", reqID, userID)
    
    // Authorization check
    callerID, _ := ctx.Value(ctxKeyUserID).(int)
    if callerID != userID {
        // Check if caller is admin
        caller, err := s.db.GetUser(ctx, callerID)
        if err != nil {
            return nil, fmt.Errorf("auth check failed: %w", err)
        }
        if caller.Role != "admin" {
            return nil, ErrUnauthorized
        }
    }
    
    user, err := s.db.GetUser(ctx, userID)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, fmt.Errorf("user %d: %w", userID, ErrNotFound)
        }
        return nil, err
    }
    
    logger.Log("[%s] Profile retrieved for %s", reqID, user.Name)
    return user, nil
}

func (s *UserService) GetUserWithOrders(ctx context.Context, userID int) (*User, []*Order, error) {
    // รัน user query และ orders query พร้อมกัน
    var (
        user   *User
        orders []*Order
        wg     sync.WaitGroup
        userErr, ordersErr error
    )
    
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        user, userErr = s.db.GetUser(ctx, userID)
    }()
    
    go func() {
        defer wg.Done()
        orders, ordersErr = s.db.GetUserOrders(ctx, userID)
    }()
    
    wg.Wait()
    
    if userErr != nil {
        return nil, nil, userErr
    }
    if ordersErr != nil {
        return nil, nil, ordersErr
    }
    
    return user, orders, nil
}

func main() {
    db := NewDB()
    userSvc := NewUserService(db)
    
    logger := &Logger{prefix: "app"}
    
    // Test 1: Get own profile
    fmt.Println("=== Test 1: Get own profile ===")
    ctx1 := context.WithValue(
        WithRequestContext(context.Background(), "req-001", 1),
        ctxKeyLogger, logger,
    )
    
    ctx1, cancel1 := context.WithTimeout(ctx1, 5*time.Second)
    defer cancel1()
    
    user, err := userSvc.GetProfile(ctx1, 1)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Got user: %s (%s)\n", user.Name, user.Role)
    }
    
    // Test 2: Admin gets other user's profile
    fmt.Println("\n=== Test 2: Admin access ===")
    ctx2 := WithRequestContext(context.Background(), "req-002", 1) // caller is user 1 (admin)
    ctx2 = context.WithValue(ctx2, ctxKeyLogger, logger)
    ctx2, cancel2 := context.WithTimeout(ctx2, 5*time.Second)
    defer cancel2()
    
    user2, err := userSvc.GetProfile(ctx2, 2) // getting user 2's profile
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Got user: %s (%s)\n", user2.Name, user2.Role)
    }
    
    // Test 3: Unauthorized access
    fmt.Println("\n=== Test 3: Unauthorized ===")
    ctx3 := WithRequestContext(context.Background(), "req-003", 2) // caller is user 2 (user)
    ctx3, cancel3 := context.WithTimeout(ctx3, 5*time.Second)
    defer cancel3()
    
    _, err = userSvc.GetProfile(ctx3, 1) // trying to get user 1's profile
    if errors.Is(err, ErrUnauthorized) {
        fmt.Println("Expected: unauthorized!")
    } else {
        fmt.Printf("Unexpected result: %v\n", err)
    }
    
    // Test 4: User with orders (concurrent)
    fmt.Println("\n=== Test 4: User with orders ===")
    ctx4 := WithRequestContext(context.Background(), "req-004", 1)
    ctx4, cancel4 := context.WithTimeout(ctx4, 5*time.Second)
    defer cancel4()
    
    start := time.Now()
    u, orders, err := userSvc.GetUserWithOrders(ctx4, 1)
    elapsed := time.Since(start)
    
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("User: %s\n", u.Name)
        fmt.Printf("Orders: %d\n", len(orders))
        fmt.Printf("Time (parallel): %v\n", elapsed)
    }
    
    // Test 5: Timeout
    fmt.Println("\n=== Test 5: Timeout ===")
    ctx5 := WithRequestContext(context.Background(), "req-005", 1)
    ctx5, cancel5 := context.WithTimeout(ctx5, 10*time.Millisecond) // very short!
    defer cancel5()
    
    _, err = userSvc.GetProfile(ctx5, 1)
    if err != nil {
        fmt.Printf("Got expected error: %v\n", err)
    }
    
    fmt.Println("\nAll tests completed!")
}
```

---

## สรุป

| Function | การใช้งาน |
|---------|----------|
| `context.Background()` | Root context, ใช้ใน main/init |
| `context.TODO()` | Placeholder, ยังไม่รู้จะใช้อะไร |
| `WithCancel` | Manual cancellation |
| `WithTimeout` | Relative timeout |
| `WithDeadline` | Absolute deadline |
| `WithValue` | Metadata ใน call chain |

### Context Rules

1. ส่ง context เป็น parameter แรกของ function เสมอ
2. ไม่เก็บ context ใน struct (ยกเว้น request-scoped)
3. ใช้ `context.Background()` เป็น root
4. cancel function ต้องถูกเรียกเสมอ (ใช้ defer)
5. ไม่ใช้ context.WithValue สำหรับ optional parameters

## Resources

- [context package](https://pkg.go.dev/context)
- [Go blog: Contexts and structs](https://go.dev/blog/context-and-structs)
- [Go blog: pipelines and cancellation](https://go.dev/blog/pipelines)
- [Using Context in Go HTTP requests](https://www.sohamkamani.com/golang/context/)
