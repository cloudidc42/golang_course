# Part 27: HTTP Server ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง HTTP Server ด้วย `net/http`
- ใช้ `http.HandleFunc` และ `http.Handler` interface
- เข้าใจ ServeMux ทั้งแบบ default และ custom
- อ่านข้อมูลจาก Request (URL, Method, Headers, Body)
- ส่ง Response (StatusCode, Headers, Body)
- ส่ง JSON responses
- Parse path และ query parameters
- Serve static files

---

## 27.1 HTTP Server พื้นฐาน

### Hello World Server (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // ลงทะเบียน handler สำหรับ path "/"
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })
    
    // เริ่ม server บน port 8080
    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Handler Function แยกไฟล์ (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// สร้าง handler functions แยกกัน
func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to Home Page!\nTime: %s", time.Now().Format(time.RFC1123))
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "About Us Page")
}

func contactHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Contact Us: contact@example.com")
}

func main() {
    // ลงทะเบียน handlers
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)
    http.HandleFunc("/contact", contactHandler)
    
    fmt.Println("Server running on http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 27.2 http.Handler Interface

### สร้าง Custom Handler Type (ตัวอย่างที่ 3)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

// ต้อง implement ServeHTTP method
type HelloHandler struct {
    Name string
}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!\n", h.Name)
    fmt.Fprintf(w, "Method: %s\n", r.Method)
    fmt.Fprintf(w, "Path: %s\n", r.URL.Path)
}

// Handler ที่เก็บ configuration
type AppHandler struct {
    Version string
    Env     string
}

func (a *AppHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "App v%s running in %s environment\n", a.Version, a.Env)
}

func main() {
    // ลงทะเบียน handler objects
    http.Handle("/hello", &HelloHandler{Name: "World"})
    http.Handle("/app", &AppHandler{Version: "1.0.0", Env: "development"})
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 27.3 Custom ServeMux

### ทำไมต้องใช้ Custom ServeMux (ตัวอย่างที่ 4)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // Default ServeMux (global, อันตราย ถ้า import package ที่ register handlers)
    // http.HandleFunc("/", handler)
    
    // Custom ServeMux (แนะนำ!)
    mux := http.NewServeMux()
    
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // DefaultServeMux จะ match "/" กับทุก path ที่ไม่ match อื่น
        if r.URL.Path != "/" {
            http.NotFound(w, r)
            return
        }
        fmt.Fprintf(w, "Home page")
    })
    
    mux.HandleFunc("/api/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "API endpoint: %s", r.URL.Path)
    })
    
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, `{"status":"ok"}`)
    })
    
    // ส่ง custom mux ให้ server
    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }
    
    fmt.Println("Server running on :8080")
    log.Fatal(server.ListenAndServe())
}
```

---

## 27.4 อ่านข้อมูลจาก Request

### อ่าน Request Information ทั้งหมด (ตัวอย่างที่ 5)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func inspectRequest(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "=== Request Details ===\n\n")
    
    // Method
    fmt.Fprintf(w, "Method: %s\n", r.Method)
    
    // URL
    fmt.Fprintf(w, "URL: %s\n", r.URL.String())
    fmt.Fprintf(w, "Path: %s\n", r.URL.Path)
    fmt.Fprintf(w, "RawQuery: %s\n", r.URL.RawQuery)
    
    // Protocol
    fmt.Fprintf(w, "Proto: %s\n", r.Proto)
    
    // Host
    fmt.Fprintf(w, "Host: %s\n", r.Host)
    
    // Remote Address
    fmt.Fprintf(w, "RemoteAddr: %s\n", r.RemoteAddr)
    
    // Headers
    fmt.Fprintf(w, "\n=== Headers ===\n")
    for name, values := range r.Header {
        for _, value := range values {
            fmt.Fprintf(w, "%s: %s\n", name, value)
        }
    }
    
    // Body
    if r.Body != nil {
        body, _ := io.ReadAll(r.Body)
        if len(body) > 0 {
            fmt.Fprintf(w, "\n=== Body ===\n%s\n", string(body))
        }
    }
}

func main() {
    http.HandleFunc("/inspect", inspectRequest)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### อ่าน Query Parameters (ตัวอย่างที่ 6)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strconv"
)

func queryHandler(w http.ResponseWriter, r *http.Request) {
    // อ่าน query params
    query := r.URL.Query()
    
    // อ่านแบบ Get (ได้ค่าแรก)
    name := query.Get("name")
    city := query.Get("city")
    
    // อ่านแบบ array (ถ้ามีหลายค่า)
    hobbies := query["hobby"]
    
    // อ่านแล้ว parse เป็น int
    ageStr := query.Get("age")
    age, err := strconv.Atoi(ageStr)
    if err != nil {
        age = 0
    }
    
    fmt.Fprintf(w, "Name: %s\n", name)
    fmt.Fprintf(w, "City: %s\n", city)
    fmt.Fprintf(w, "Age: %d\n", age)
    fmt.Fprintf(w, "Hobbies: %v\n", hobbies)
    
    // ตรวจสอบว่ามี param ไหม
    if _, ok := query["debug"]; ok {
        fmt.Fprintf(w, "Debug mode enabled!\n")
    }
}

func main() {
    http.HandleFunc("/query", queryHandler)
    // ทดสอบ: curl "http://localhost:8080/query?name=สมชาย&city=กรุงเทพ&age=25&hobby=coding&hobby=reading"
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### อ่าน Request Headers (ตัวอย่างที่ 7)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func headersHandler(w http.ResponseWriter, r *http.Request) {
    // อ่าน header ที่สนใจ
    authHeader := r.Header.Get("Authorization")
    contentType := r.Header.Get("Content-Type")
    userAgent := r.Header.Get("User-Agent")
    acceptLang := r.Header.Get("Accept-Language")
    
    // Header ที่ Go parse ให้แล้ว
    fmt.Fprintf(w, "Authorization: %s\n", authHeader)
    fmt.Fprintf(w, "Content-Type: %s\n", contentType)
    fmt.Fprintf(w, "User-Agent: %s\n", userAgent)
    fmt.Fprintf(w, "Accept-Language: %s\n", acceptLang)
    
    // ตรวจสอบ bearer token
    if len(authHeader) > 7 && authHeader[:7] == "Bearer " {
        token := authHeader[7:]
        fmt.Fprintf(w, "Bearer Token: %s\n", token)
    }
}

func main() {
    http.HandleFunc("/headers", headersHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### อ่าน JSON Body (ตัวอย่างที่ 8)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    // ตรวจสอบ method
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // ตรวจสอบ Content-Type
    contentType := r.Header.Get("Content-Type")
    if contentType != "application/json" {
        http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
        return
    }
    
    // Decode JSON body
    var req CreateUserRequest
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields() // ไม่ยอมรับ field ที่ไม่รู้จัก
    
    if err := decoder.Decode(&req); err != nil {
        http.Error(w, "Invalid JSON: "+err.Error(), http.StatusBadRequest)
        return
    }
    
    // Validate
    if req.Name == "" {
        http.Error(w, "Name is required", http.StatusBadRequest)
        return
    }
    
    // ทำงานกับข้อมูล
    fmt.Fprintf(w, "Created user: %s (Email: %s, Age: %d)\n", req.Name, req.Email, req.Age)
}

func main() {
    http.HandleFunc("/users", createUserHandler)
    // ทดสอบ: curl -X POST -H "Content-Type: application/json" -d '{"name":"John","email":"john@example.com","age":30}' http://localhost:8080/users
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 27.5 ส่ง Response

### Response Status Codes (ตัวอย่างที่ 9)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func statusCodesHandler(w http.ResponseWriter, r *http.Request) {
    code := r.URL.Query().Get("code")
    
    switch code {
    case "200":
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "OK")
    case "201":
        w.WriteHeader(http.StatusCreated)
        fmt.Fprintf(w, "Created")
    case "204":
        w.WriteHeader(http.StatusNoContent)
        // No body for 204
    case "400":
        http.Error(w, "Bad Request", http.StatusBadRequest)
    case "401":
        w.Header().Set("WWW-Authenticate", "Bearer")
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
    case "403":
        http.Error(w, "Forbidden", http.StatusForbidden)
    case "404":
        http.NotFound(w, r)
    case "500":
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
    default:
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "Status: %s\n", code)
    }
}

func main() {
    http.HandleFunc("/status", statusCodesHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### ส่ง Response Headers (ตัวอย่างที่ 10)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

func headersResponseHandler(w http.ResponseWriter, r *http.Request) {
    // ต้องตั้งค่า Headers ก่อน WriteHeader หรือ Write
    w.Header().Set("Content-Type", "text/plain; charset=utf-8")
    w.Header().Set("X-Custom-Header", "my-value")
    w.Header().Set("Cache-Control", "no-cache, no-store, must-revalidate")
    w.Header().Set("X-Request-ID", "req-12345")
    
    // เพิ่ม header (ไม่ replace)
    w.Header().Add("X-Tags", "go")
    w.Header().Add("X-Tags", "http")
    w.Header().Add("X-Tags", "server")
    
    // CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    
    // เขียน status code (ต้องอยู่หลัง Header แต่ก่อน Body)
    w.WriteHeader(http.StatusOK)
    
    fmt.Fprintf(w, "Response with custom headers\n")
    fmt.Fprintf(w, "Time: %s\n", time.Now().Format(time.RFC3339))
}

func main() {
    http.HandleFunc("/headers-response", headersResponseHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### ส่ง JSON Response (ตัวอย่างที่ 11)

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

type APIResponse struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Message string      `json:"message,omitempty"`
}

// Helper function สำหรับส่ง JSON response
func respondJSON(w http.ResponseWriter, statusCode int, payload interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(payload)
}

func respondError(w http.ResponseWriter, statusCode int, message string) {
    respondJSON(w, statusCode, APIResponse{
        Success: false,
        Error:   message,
    })
}

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    user := User{
        ID:        1,
        Name:      "สมชาย รักเรียน",
        Email:     "somchai@example.com",
        CreatedAt: time.Now(),
    }
    
    respondJSON(w, http.StatusOK, APIResponse{
        Success: true,
        Data:    user,
    })
}

func getUsersHandler(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "สมชาย", Email: "somchai@example.com", CreatedAt: time.Now()},
        {ID: 2, Name: "สมหญิง", Email: "somying@example.com", CreatedAt: time.Now()},
    }
    
    respondJSON(w, http.StatusOK, APIResponse{
        Success: true,
        Data:    users,
        Message: "Found 2 users",
    })
}

func main() {
    http.HandleFunc("/user", getUserHandler)
    http.HandleFunc("/users", getUsersHandler)
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 27.6 Path Parameters (Manual Parsing)

### Parse Path Parameters (ตัวอย่างที่ 12)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
)

func userHandler(w http.ResponseWriter, r *http.Request) {
    // URL pattern: /users/{id}
    // URL pattern: /users/{id}/posts
    
    // ตัด prefix "/users/" ออก
    path := strings.TrimPrefix(r.URL.Path, "/users/")
    parts := strings.Split(path, "/")
    
    if len(parts) < 1 || parts[0] == "" {
        http.Error(w, "Missing user ID", http.StatusBadRequest)
        return
    }
    
    // Parse user ID
    userID, err := strconv.Atoi(parts[0])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    // ตรวจสอบว่ามี sub-path ไหม
    if len(parts) >= 2 {
        switch parts[1] {
        case "posts":
            fmt.Fprintf(w, "Posts of user %d", userID)
        case "profile":
            fmt.Fprintf(w, "Profile of user %d", userID)
        default:
            http.NotFound(w, r)
        }
        return
    }
    
    // User detail
    fmt.Fprintf(w, "User ID: %d\n", userID)
    fmt.Fprintf(w, "Name: User_%d\n", userID)
}

func main() {
    http.HandleFunc("/users/", userHandler)
    // ทดสอบ: 
    //   curl http://localhost:8080/users/1
    //   curl http://localhost:8080/users/1/posts
    //   curl http://localhost:8080/users/1/profile
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Router Helper Function (ตัวอย่างที่ 13)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "regexp"
    "strings"
)

// Route เก็บข้อมูล pattern และ handler
type Route struct {
    Pattern *regexp.Regexp
    Methods []string
    Handler http.HandlerFunc
    Params  []string // ชื่อของ captured groups
}

// SimpleRouter เป็น router แบบง่าย
type SimpleRouter struct {
    routes []Route
}

func (router *SimpleRouter) Handle(method, pattern string, handler http.HandlerFunc) {
    // แปลง /users/{id}/posts เป็น regex
    params := []string{}
    regexPattern := regexp.MustCompile(`\{([^}]+)\}`).ReplaceAllStringFunc(pattern, func(match string) string {
        param := match[1 : len(match)-1]
        params = append(params, param)
        return `([^/]+)`
    })
    
    route := Route{
        Pattern: regexp.MustCompile("^" + regexPattern + "$"),
        Methods: []string{method},
        Handler: handler,
        Params:  params,
    }
    router.routes = append(router.routes, route)
}

func (router *SimpleRouter) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    path := r.URL.Path
    
    for _, route := range router.routes {
        matches := route.Pattern.FindStringSubmatch(path)
        if matches == nil {
            continue
        }
        
        // ตรวจ method
        allowed := false
        for _, m := range route.Methods {
            if m == r.Method {
                allowed = true
                break
            }
        }
        
        if !allowed {
            http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
            return
        }
        
        // เพิ่ม params เข้า context (simplified version)
        // ใน production ควรใช้ context.WithValue
        for i, param := range route.Params {
            r.Header.Set("X-Param-"+strings.Title(param), matches[i+1])
        }
        
        route.Handler(w, r)
        return
    }
    
    http.NotFound(w, r)
}

func main() {
    router := &SimpleRouter{}
    
    router.Handle("GET", "/users", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "List all users")
    })
    
    router.Handle("GET", "/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Param-Id")
        fmt.Fprintf(w, "Get user: %s", id)
    })
    
    router.Handle("GET", "/users/{id}/posts", func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Param-Id")
        fmt.Fprintf(w, "Posts of user: %s", id)
    })
    
    log.Fatal(http.ListenAndServe(":8080", router))
}
```

---

## 27.7 Static File Serving

### Serve Static Files (ตัวอย่างที่ 14)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()
    
    // Serve static files จาก directory "./static"
    fs := http.FileServer(http.Dir("./static"))
    
    // Mount ที่ /static/
    mux.Handle("/static/", http.StripPrefix("/static/", fs))
    
    // Handler สำหรับ HTML page
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/" {
            http.NotFound(w, r)
            return
        }
        
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprintf(w, `<!DOCTYPE html>
<html>
<head><title>Go Server</title></head>
<body>
    <h1>Hello from Go Server!</h1>
    <p>Static files at <a href="/static/">/static/</a></p>
</body>
</html>`)
    })
    
    fmt.Println("Server running on :8080")
    fmt.Println("Static files at ./static/")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 27.8 Middleware Pattern

### Basic Middleware (ตัวอย่างที่ 15)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// Middleware type
type Middleware func(http.HandlerFunc) http.HandlerFunc

// Logging middleware
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // ก่อน handler
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        
        // เรียก next handler
        next(w, r)
        
        // หลัง handler
        log.Printf("Completed %s %s in %v", r.Method, r.URL.Path, time.Since(start))
    }
}

// Auth middleware
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // ตรวจ token (simplified)
        if token != "Bearer valid-token" {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        next(w, r)
    }
}

// Chain middlewares
func chain(handler http.HandlerFunc, middlewares ...Middleware) http.HandlerFunc {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func protectedHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Protected resource accessed!")
}

func publicHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Public resource")
}

func main() {
    mux := http.NewServeMux()
    
    // Public route พร้อม logging
    mux.HandleFunc("/public", chain(publicHandler, loggingMiddleware))
    
    // Protected route พร้อม logging + auth
    mux.HandleFunc("/protected", chain(protectedHandler, loggingMiddleware, authMiddleware))
    
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 27.9 Graceful Shutdown

### Server พร้อม Graceful Shutdown (ตัวอย่างที่ 16)

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // Simulate slow handler
        time.Sleep(2 * time.Second)
        fmt.Fprintf(w, "Hello World!")
    })
    
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // เริ่ม server ใน goroutine
    go func() {
        fmt.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal("Server error:", err)
        }
    }()
    
    // รอ signal สำหรับ shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    
    <-quit // Block จนกว่าจะได้ signal
    fmt.Println("\nShutting down server...")
    
    // สร้าง context พร้อม timeout สำหรับ shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Graceful shutdown
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    
    fmt.Println("Server stopped gracefully")
}
```

---

## 27.10 Complete HTTP Server Example

### Complete REST-like Server (ตัวอย่างที่ 17)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
    "sync"
    "time"
)

// Model
type Product struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Price     float64   `json:"price"`
    Stock     int       `json:"stock"`
    CreatedAt time.Time `json:"created_at"`
}

// In-memory storage
type ProductStore struct {
    mu       sync.RWMutex
    products map[int]*Product
    nextID   int
}

func NewProductStore() *ProductStore {
    store := &ProductStore{
        products: make(map[int]*Product),
        nextID:   1,
    }
    // ข้อมูลตัวอย่าง
    store.Create(&Product{Name: "Go Programming Book", Price: 499.00, Stock: 100})
    store.Create(&Product{Name: "Laptop Stand", Price: 1200.00, Stock: 50})
    return store
}

func (s *ProductStore) Create(p *Product) *Product {
    s.mu.Lock()
    defer s.mu.Unlock()
    p.ID = s.nextID
    p.CreatedAt = time.Now()
    s.products[p.ID] = p
    s.nextID++
    return p
}

func (s *ProductStore) GetAll() []*Product {
    s.mu.RLock()
    defer s.mu.RUnlock()
    products := make([]*Product, 0, len(s.products))
    for _, p := range s.products {
        products = append(products, p)
    }
    return products
}

func (s *ProductStore) GetByID(id int) (*Product, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    p, ok := s.products[id]
    return p, ok
}

func (s *ProductStore) Update(id int, p *Product) (*Product, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.products[id]; !ok {
        return nil, false
    }
    p.ID = id
    s.products[id] = p
    return p, true
}

func (s *ProductStore) Delete(id int) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.products[id]; !ok {
        return false
    }
    delete(s.products, id)
    return true
}

// Handler
type ProductHandler struct {
    store *ProductStore
}

func NewProductHandler(store *ProductStore) *ProductHandler {
    return &ProductHandler{store: store}
}

func (h *ProductHandler) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *ProductHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // route: /products atau /products/{id}
    path := strings.TrimPrefix(r.URL.Path, "/products")
    path = strings.TrimPrefix(path, "/")
    
    if path == "" {
        // /products
        switch r.Method {
        case http.MethodGet:
            h.listProducts(w, r)
        case http.MethodPost:
            h.createProduct(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
        return
    }
    
    // /products/{id}
    id, err := strconv.Atoi(path)
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    
    switch r.Method {
    case http.MethodGet:
        h.getProduct(w, r, id)
    case http.MethodPut:
        h.updateProduct(w, r, id)
    case http.MethodDelete:
        h.deleteProduct(w, r, id)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (h *ProductHandler) listProducts(w http.ResponseWriter, r *http.Request) {
    products := h.store.GetAll()
    h.respondJSON(w, http.StatusOK, map[string]interface{}{
        "data":  products,
        "total": len(products),
    })
}

func (h *ProductHandler) createProduct(w http.ResponseWriter, r *http.Request) {
    var product Product
    if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    created := h.store.Create(&product)
    h.respondJSON(w, http.StatusCreated, created)
}

func (h *ProductHandler) getProduct(w http.ResponseWriter, r *http.Request, id int) {
    product, ok := h.store.GetByID(id)
    if !ok {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    h.respondJSON(w, http.StatusOK, product)
}

func (h *ProductHandler) updateProduct(w http.ResponseWriter, r *http.Request, id int) {
    var product Product
    if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    updated, ok := h.store.Update(id, &product)
    if !ok {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    h.respondJSON(w, http.StatusOK, updated)
}

func (h *ProductHandler) deleteProduct(w http.ResponseWriter, r *http.Request, id int) {
    if !h.store.Delete(id) {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    w.WriteHeader(http.StatusNoContent)
}

func main() {
    store := NewProductStore()
    handler := NewProductHandler(store)
    
    mux := http.NewServeMux()
    mux.Handle("/products", handler)
    mux.Handle("/products/", handler)
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })
    
    fmt.Println("Product Server running on :8080")
    fmt.Println("Endpoints:")
    fmt.Println("  GET    /products")
    fmt.Println("  POST   /products")
    fmt.Println("  GET    /products/{id}")
    fmt.Println("  PUT    /products/{id}")
    fmt.Println("  DELETE /products/{id}")
    
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## สรุป Part 27

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Server Setup | `http.ListenAndServe(addr, handler)` |
| HandleFunc | `http.HandleFunc(pattern, func(w, r))` |
| Handler Interface | `ServeHTTP(w http.ResponseWriter, r *http.Request)` |
| ServeMux | `http.NewServeMux()` - แนะนำใช้แทน default |
| Request | `r.Method`, `r.URL`, `r.Header`, `r.Body` |
| Response | `w.Header().Set()`, `w.WriteHeader()`, `w.Write()` |
| JSON | `json.NewDecoder(r.Body).Decode()`, `json.NewEncoder(w).Encode()` |
| Path Params | Parse manually จาก `r.URL.Path` |
| Static Files | `http.FileServer(http.Dir("path"))` |
| Graceful Shutdown | `server.Shutdown(ctx)` + signal handling |

### Resources
- [net/http package](https://pkg.go.dev/net/http)
- [Effective Go - Web servers](https://go.dev/doc/effective_go#web_server)
- [Go blog: HTTP handlers](https://go.dev/blog/error-handling-and-go)
