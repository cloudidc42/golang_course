# Part 85: Build Your Own HTTP Framework in Go

## เป้าหมายของบทเรียน
- สร้าง HTTP framework ตั้งแต่ต้น
- Router implementation
- Middleware system
- Context propagation
- Error handling
- Request validation
- Response formatting
- Testing utilities

---

## 1. Core Router Implementation

```go
// framework/router.go
package goweb

import (
    "fmt"
    "net/http"
    "strings"
)

// HandlerFunc type สำหรับ handler
type HandlerFunc func(*Context)

// MiddlewareFunc type สำหรับ middleware
type MiddlewareFunc func(HandlerFunc) HandlerFunc

// RouteParam parameters จาก URL path
type RouteParam struct {
    Key   string
    Value string
}

// treeNode node ใน prefix tree
type treeNode struct {
    path     string
    children map[string]*treeNode
    handler  HandlerFunc
    isParam  bool
    paramKey string
    isWild   bool
}

// newTreeNode สร้าง node ใหม่
func newTreeNode(path string) *treeNode {
    return &treeNode{
        path:     path,
        children: make(map[string]*treeNode),
    }
}

// Router HTTP router
type Router struct {
    trees       map[string]*treeNode // method -> root
    middlewares []MiddlewareFunc
    notFound    HandlerFunc
    methodNotAllowed HandlerFunc
}

// NewRouter สร้าง router ใหม่
func NewRouter() *Router {
    r := &Router{
        trees: make(map[string]*treeNode),
    }
    r.notFound = func(c *Context) {
        c.JSON(http.StatusNotFound, Map{"error": "Not Found"})
    }
    r.methodNotAllowed = func(c *Context) {
        c.JSON(http.StatusMethodNotAllowed, Map{"error": "Method Not Allowed"})
    }
    return r
}

// Handle ลงทะเบียน route handler
func (r *Router) Handle(method, path string, handlers ...HandlerFunc) {
    if len(handlers) == 0 {
        panic("at least one handler required")
    }
    
    method = strings.ToUpper(method)
    
    if _, ok := r.trees[method]; !ok {
        r.trees[method] = newTreeNode("/")
    }
    
    // Chain handlers
    finalHandler := handlers[len(handlers)-1]
    if len(handlers) > 1 {
        finalHandler = chainHandlers(handlers...)
    }
    
    r.insertRoute(r.trees[method], path, finalHandler)
}

// chainHandlers chain หลาย handlers
func chainHandlers(handlers ...HandlerFunc) HandlerFunc {
    return func(c *Context) {
        for _, h := range handlers {
            h(c)
            if c.isAborted() {
                return
            }
        }
    }
}

// insertRoute แทรก route เข้า tree
func (r *Router) insertRoute(root *treeNode, path string, handler HandlerFunc) {
    parts := splitPath(path)
    current := root
    
    for _, part := range parts {
        if strings.HasPrefix(part, ":") {
            // Parameter node
            if _, ok := current.children[":param"]; !ok {
                node := newTreeNode(part)
                node.isParam = true
                node.paramKey = part[1:]
                current.children[":param"] = node
            }
            current = current.children[":param"]
        } else if part == "*" {
            // Wildcard node
            if _, ok := current.children["*"]; !ok {
                node := newTreeNode("*")
                node.isWild = true
                current.children["*"] = node
            }
            current = current.children["*"]
        } else {
            if _, ok := current.children[part]; !ok {
                current.children[part] = newTreeNode(part)
            }
            current = current.children[part]
        }
    }
    
    current.handler = handler
}

// matchRoute จับคู่ route
func (r *Router) matchRoute(root *treeNode, path string) (HandlerFunc, []RouteParam) {
    parts := splitPath(path)
    current := root
    params := make([]RouteParam, 0)
    
    for _, part := range parts {
        if child, ok := current.children[part]; ok {
            current = child
        } else if child, ok := current.children[":param"]; ok {
            params = append(params, RouteParam{Key: child.paramKey, Value: part})
            current = child
        } else if child, ok := current.children["*"]; ok {
            params = append(params, RouteParam{Key: "wildcard", Value: path})
            current = child
            break
        } else {
            return nil, nil
        }
    }
    
    return current.handler, params
}

// splitPath แบ่ง path เป็น parts
func splitPath(path string) []string {
    parts := strings.Split(strings.Trim(path, "/"), "/")
    result := make([]string, 0, len(parts))
    for _, p := range parts {
        if p != "" {
            result = append(result, p)
        }
    }
    return result
}

// Use เพิ่ม global middleware
func (r *Router) Use(middleware ...MiddlewareFunc) {
    r.middlewares = append(r.middlewares, middleware...)
}

// GET ลงทะเบียน GET route
func (r *Router) GET(path string, handlers ...HandlerFunc) {
    r.Handle("GET", path, handlers...)
}

// POST ลงทะเบียน POST route
func (r *Router) POST(path string, handlers ...HandlerFunc) {
    r.Handle("POST", path, handlers...)
}

// PUT ลงทะเบียน PUT route
func (r *Router) PUT(path string, handlers ...HandlerFunc) {
    r.Handle("PUT", path, handlers...)
}

// DELETE ลงทะเบียน DELETE route
func (r *Router) DELETE(path string, handlers ...HandlerFunc) {
    r.Handle("DELETE", path, handlers...)
}

// ServeHTTP implements http.Handler
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := newContext(w, req)
    
    tree, ok := r.trees[req.Method]
    if !ok {
        r.methodNotAllowed(c)
        return
    }
    
    handler, params := r.matchRoute(tree, req.URL.Path)
    if handler == nil {
        r.notFound(c)
        return
    }
    
    // Set params
    for _, p := range params {
        c.Params[p.Key] = p.Value
    }
    
    // Apply middlewares
    final := handler
    for i := len(r.middlewares) - 1; i >= 0; i-- {
        final = r.middlewares[i](final)
    }
    
    final(c)
}
```

---

## 2. Context Implementation

```go
// framework/context.go
package goweb

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
)

// Map type alias
type Map map[string]interface{}

// Context แสดง request context
type Context struct {
    Request  *http.Request
    Response http.ResponseWriter
    Params   map[string]string
    keys     map[string]interface{}
    aborted  bool
    status   int
}

// newContext สร้าง context ใหม่
func newContext(w http.ResponseWriter, r *http.Request) *Context {
    return &Context{
        Request:  r,
        Response: w,
        Params:   make(map[string]string),
        keys:     make(map[string]interface{}),
        status:   http.StatusOK,
    }
}

// Set เก็บค่าใน context
func (c *Context) Set(key string, value interface{}) {
    c.keys[key] = value
}

// Get ดึงค่าจาก context
func (c *Context) Get(key string) (interface{}, bool) {
    v, ok := c.keys[key]
    return v, ok
}

// MustGet ดึงค่าและ panic ถ้าไม่มี
func (c *Context) MustGet(key string) interface{} {
    v, ok := c.Get(key)
    if !ok {
        panic(fmt.Sprintf("key '%s' not found in context", key))
    }
    return v
}

// Abort หยุดการประมวลผล
func (c *Context) Abort() {
    c.aborted = true
}

// AbortWithStatus หยุดและส่ง status code
func (c *Context) AbortWithStatus(code int) {
    c.status = code
    c.Response.WriteHeader(code)
    c.Abort()
}

// AbortWithJSON หยุดและส่ง JSON response
func (c *Context) AbortWithJSON(code int, obj interface{}) {
    c.JSON(code, obj)
    c.Abort()
}

// isAborted ตรวจสอบว่าถูก abort หรือไม่
func (c *Context) isAborted() bool {
    return c.aborted
}

// Param คืน URL parameter
func (c *Context) Param(key string) string {
    return c.Params[key]
}

// Query คืน query parameter
func (c *Context) Query(key string) string {
    return c.Request.URL.Query().Get(key)
}

// QueryDefault คืน query parameter หรือ default value
func (c *Context) QueryDefault(key, defaultValue string) string {
    if v := c.Query(key); v != "" {
        return v
    }
    return defaultValue
}

// QueryInt คืน query parameter เป็น int
func (c *Context) QueryInt(key string, defaultValue int) int {
    if v := c.Query(key); v != "" {
        if i, err := strconv.Atoi(v); err == nil {
            return i
        }
    }
    return defaultValue
}

// Header คืน request header
func (c *Context) Header(key string) string {
    return c.Request.Header.Get(key)
}

// SetHeader กำหนด response header
func (c *Context) SetHeader(key, value string) {
    c.Response.Header().Set(key, value)
}

// BindJSON parse JSON body
func (c *Context) BindJSON(obj interface{}) error {
    decoder := json.NewDecoder(c.Request.Body)
    return decoder.Decode(obj)
}

// JSON ส่ง JSON response
func (c *Context) JSON(code int, obj interface{}) {
    c.SetHeader("Content-Type", "application/json; charset=utf-8")
    c.Response.WriteHeader(code)
    
    if err := json.NewEncoder(c.Response).Encode(obj); err != nil {
        fmt.Printf("Error encoding JSON: %v\n", err)
    }
}

// String ส่ง string response
func (c *Context) String(code int, format string, values ...interface{}) {
    c.SetHeader("Content-Type", "text/plain; charset=utf-8")
    c.Response.WriteHeader(code)
    
    if len(values) > 0 {
        fmt.Fprintf(c.Response, format, values...)
    } else {
        fmt.Fprint(c.Response, format)
    }
}

// Status ส่ง status code เท่านั้น
func (c *Context) Status(code int) {
    c.Response.WriteHeader(code)
}

// Redirect redirect ไปยัง URL
func (c *Context) Redirect(code int, url string) {
    http.Redirect(c.Response, c.Request, url, code)
}

// ClientIP คืน client IP address
func (c *Context) ClientIP() string {
    // ตรวจสอบ X-Forwarded-For ก่อน
    if xff := c.Header("X-Forwarded-For"); xff != "" {
        parts := strings.Split(xff, ",")
        return strings.TrimSpace(parts[0])
    }
    
    // ตรวจสอบ X-Real-IP
    if xri := c.Header("X-Real-IP"); xri != "" {
        return xri
    }
    
    // ใช้ RemoteAddr
    addr := c.Request.RemoteAddr
    if i := strings.LastIndex(addr, ":"); i != -1 {
        return addr[:i]
    }
    return addr
}
```

---

## 3. Middleware System

```go
// framework/middleware.go
package goweb

import (
    "fmt"
    "log"
    "net/http"
    "runtime/debug"
    "time"
)

// Logger middleware สำหรับ logging
func Logger() MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            start := time.Now()
            path := c.Request.URL.Path
            method := c.Request.Method
            
            next(c)
            
            latency := time.Since(start)
            statusCode := c.status
            
            log.Printf("[%s] %s %s | %d | %v | %s",
                time.Now().Format("2006/01/02 - 15:04:05"),
                method,
                path,
                statusCode,
                latency,
                c.ClientIP(),
            )
        }
    }
}

// Recovery middleware สำหรับ panic recovery
func Recovery() MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            defer func() {
                if err := recover(); err != nil {
                    stack := debug.Stack()
                    log.Printf("[Recovery] panic: %v\n%s", err, stack)
                    c.JSON(http.StatusInternalServerError, Map{
                        "error": "Internal Server Error",
                    })
                }
            }()
            next(c)
        }
    }
}

// CORS middleware
func CORS(allowOrigins []string) MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            origin := c.Header("Origin")
            
            allowed := false
            for _, o := range allowOrigins {
                if o == "*" || o == origin {
                    allowed = true
                    break
                }
            }
            
            if allowed {
                if origin == "" {
                    origin = "*"
                }
                c.SetHeader("Access-Control-Allow-Origin", origin)
                c.SetHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS, PATCH")
                c.SetHeader("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
                c.SetHeader("Access-Control-Max-Age", "86400")
            }
            
            if c.Request.Method == "OPTIONS" {
                c.Status(http.StatusNoContent)
                c.Abort()
                return
            }
            
            next(c)
        }
    }
}

// RateLimit middleware
func RateLimit(maxRPS int) MiddlewareFunc {
    type client struct {
        count    int
        resetAt  time.Time
    }
    
    clients := make(map[string]*client)
    
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            ip := c.ClientIP()
            
            now := time.Now()
            cl, ok := clients[ip]
            
            if !ok || now.After(cl.resetAt) {
                clients[ip] = &client{
                    count:   1,
                    resetAt: now.Add(time.Second),
                }
            } else {
                cl.count++
                if cl.count > maxRPS {
                    c.SetHeader("Retry-After", "1")
                    c.AbortWithJSON(http.StatusTooManyRequests, Map{
                        "error": "Rate limit exceeded",
                    })
                    return
                }
            }
            
            next(c)
        }
    }
}

// RequestID middleware เพิ่ม unique request ID
func RequestID() MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            requestID := c.Header("X-Request-ID")
            if requestID == "" {
                requestID = fmt.Sprintf("%d", time.Now().UnixNano())
            }
            
            c.SetHeader("X-Request-ID", requestID)
            c.Set("request_id", requestID)
            
            next(c)
        }
    }
}

// BasicAuth middleware
func BasicAuth(username, password string) MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            u, p, ok := c.Request.BasicAuth()
            if !ok || u != username || p != password {
                c.SetHeader("WWW-Authenticate", `Basic realm="Restricted"`)
                c.AbortWithStatus(http.StatusUnauthorized)
                return
            }
            next(c)
        }
    }
}
```

---

## 4. Request Validation

```go
// framework/validation.go
package goweb

import (
    "fmt"
    "net/http"
    "reflect"
    "regexp"
    "strconv"
    "strings"
)

// ValidationError แสดง validation error
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// ValidationErrors collection ของ errors
type ValidationErrors []*ValidationError

func (e ValidationErrors) Error() string {
    msgs := make([]string, len(e))
    for i, err := range e {
        msgs[i] = err.Error()
    }
    return strings.Join(msgs, "; ")
}

// Validator validates struct fields ตาม tags
type Validator struct{}

// Validate validates ค่าตาม struct tags
func (v *Validator) Validate(obj interface{}) ValidationErrors {
    errors := make(ValidationErrors, 0)
    
    val := reflect.ValueOf(obj)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }
    
    typ := val.Type()
    
    for i := 0; i < val.NumField(); i++ {
        field := typ.Field(i)
        value := val.Field(i)
        
        validateTag := field.Tag.Get("validate")
        if validateTag == "" {
            continue
        }
        
        fieldName := field.Tag.Get("json")
        if fieldName == "" {
            fieldName = field.Name
        }
        
        rules := strings.Split(validateTag, ",")
        for _, rule := range rules {
            if err := v.validateField(fieldName, value, rule); err != nil {
                errors = append(errors, err)
            }
        }
    }
    
    if len(errors) > 0 {
        return errors
    }
    return nil
}

// validateField ตรวจสอบ field ตาม rule
func (v *Validator) validateField(fieldName string, value reflect.Value, rule string) *ValidationError {
    parts := strings.SplitN(rule, "=", 2)
    ruleName := parts[0]
    ruleValue := ""
    if len(parts) > 1 {
        ruleValue = parts[1]
    }
    
    switch ruleName {
    case "required":
        if value.IsZero() {
            return &ValidationError{Field: fieldName, Message: "is required"}
        }
        
    case "min":
        minVal, _ := strconv.ParseFloat(ruleValue, 64)
        switch value.Kind() {
        case reflect.String:
            if float64(len(value.String())) < minVal {
                return &ValidationError{Field: fieldName, 
                    Message: fmt.Sprintf("minimum length is %s", ruleValue)}
            }
        case reflect.Int, reflect.Int64:
            if float64(value.Int()) < minVal {
                return &ValidationError{Field: fieldName,
                    Message: fmt.Sprintf("minimum value is %s", ruleValue)}
            }
        }
        
    case "max":
        maxVal, _ := strconv.ParseFloat(ruleValue, 64)
        switch value.Kind() {
        case reflect.String:
            if float64(len(value.String())) > maxVal {
                return &ValidationError{Field: fieldName,
                    Message: fmt.Sprintf("maximum length is %s", ruleValue)}
            }
        case reflect.Int, reflect.Int64:
            if float64(value.Int()) > maxVal {
                return &ValidationError{Field: fieldName,
                    Message: fmt.Sprintf("maximum value is %s", ruleValue)}
            }
        }
        
    case "email":
        emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
        if !emailRegex.MatchString(value.String()) {
            return &ValidationError{Field: fieldName, Message: "must be a valid email"}
        }
        
    case "oneof":
        allowed := strings.Split(ruleValue, " ")
        str := value.String()
        found := false
        for _, a := range allowed {
            if a == str {
                found = true
                break
            }
        }
        if !found {
            return &ValidationError{Field: fieldName,
                Message: fmt.Sprintf("must be one of: %s", strings.Join(allowed, ", "))}
        }
    }
    
    return nil
}

// Bind and validate
func (c *Context) BindAndValidate(obj interface{}) ValidationErrors {
    if err := c.BindJSON(obj); err != nil {
        return ValidationErrors{
            {Field: "body", Message: "invalid JSON: " + err.Error()},
        }
    }
    
    v := &Validator{}
    return v.Validate(obj)
}

// ValidateJSON middleware สำหรับ validate request
func ValidateJSON(template interface{}) MiddlewareFunc {
    return func(next HandlerFunc) HandlerFunc {
        return func(c *Context) {
            if errs := c.BindAndValidate(template); errs != nil {
                c.AbortWithJSON(http.StatusBadRequest, Map{
                    "error":  "Validation failed",
                    "fields": errs,
                })
                return
            }
            c.Set("body", template)
            next(c)
        }
    }
}
```

---

## 5. Complete Framework Usage Example

```go
// main.go - ตัวอย่างการใช้งาน framework
package main

import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "strings"
)

// CreateUserRequest request สำหรับสร้าง user
type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=2,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Age      int    `json:"age" validate:"required,min=18,max=120"`
    Role     string `json:"role" validate:"required,oneof=admin user viewer"`
}

// User struct
type UserStruct struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
    Role  string `json:"role"`
}

var users = []UserStruct{
    {ID: 1, Name: "สมชาย", Email: "somchai@example.com", Age: 30, Role: "admin"},
    {ID: 2, Name: "สมหญิง", Email: "somying@example.com", Age: 25, Role: "user"},
}

func setupApp() *Router {
    r := NewRouter()
    
    // Global middlewares
    r.Use(Recovery())
    r.Use(Logger())
    r.Use(CORS([]string{"*"}))
    r.Use(RequestID())
    
    // Routes
    r.GET("/health", func(c *Context) {
        c.JSON(200, Map{"status": "ok"})
    })
    
    r.GET("/users", func(c *Context) {
        page := c.QueryInt("page", 1)
        limit := c.QueryInt("limit", 10)
        
        start := (page - 1) * limit
        end := start + limit
        
        if start >= len(users) {
            c.JSON(200, Map{"users": []UserStruct{}, "total": len(users)})
            return
        }
        if end > len(users) {
            end = len(users)
        }
        
        c.JSON(200, Map{
            "users": users[start:end],
            "total": len(users),
            "page":  page,
            "limit": limit,
        })
    })
    
    r.GET("/users/:id", func(c *Context) {
        id := c.Param("id")
        
        for _, u := range users {
            if fmt.Sprintf("%d", u.ID) == id {
                c.JSON(200, u)
                return
            }
        }
        
        c.JSON(404, Map{"error": "User not found"})
    })
    
    r.POST("/users", func(c *Context) {
        var req CreateUserRequest
        if errs := c.BindAndValidate(&req); errs != nil {
            c.AbortWithJSON(400, Map{
                "error":  "Validation failed",
                "fields": errs,
            })
            return
        }
        
        user := UserStruct{
            ID:    len(users) + 1,
            Name:  req.Name,
            Email: req.Email,
            Age:   req.Age,
            Role:  req.Role,
        }
        users = append(users, user)
        
        c.JSON(201, user)
    })
    
    r.DELETE("/users/:id", BasicAuth("admin", "secret"), func(c *Context) {
        id := c.Param("id")
        
        for i, u := range users {
            if fmt.Sprintf("%d", u.ID) == id {
                users = append(users[:i], users[i+1:]...)
                c.Status(204)
                return
            }
        }
        
        c.JSON(404, Map{"error": "User not found"})
    })
    
    return r
}

func main() {
    app := setupApp()
    
    fmt.Println("=== Custom HTTP Framework Demo ===\n")
    
    // Test cases
    tests := []struct {
        method string
        path   string
        body   string
    }{
        {"GET", "/health", ""},
        {"GET", "/users", ""},
        {"GET", "/users?page=1&limit=1", ""},
        {"GET", "/users/1", ""},
        {"GET", "/users/99", ""},
        {"POST", "/users", `{"name":"สมศักดิ์","email":"somsak@example.com","age":28,"role":"user"}`},
        {"POST", "/users", `{"name":"A","email":"not-email","age":15,"role":"invalid"}`},
    }
    
    for _, tc := range tests {
        var body *strings.Reader
        if tc.body != "" {
            body = strings.NewReader(tc.body)
        } else {
            body = strings.NewReader("")
        }
        
        req := httptest.NewRequest(tc.method, tc.path, body)
        if tc.body != "" {
            req.Header.Set("Content-Type", "application/json")
        }
        
        w := httptest.NewRecorder()
        app.ServeHTTP(w, req)
        
        fmt.Printf("%s %s -> %d\n", tc.method, tc.path, w.Code)
        if w.Body.Len() > 0 && w.Body.Len() < 200 {
            fmt.Printf("  Body: %s\n", strings.TrimSpace(w.Body.String()))
        }
    }
    
    fmt.Printf("\nServer would start on :8080\n")
    // http.ListenAndServe(":8080", app)
    _ = http.MethodGet
}
```

---

## สรุป

ในบทนี้เราได้สร้าง HTTP framework ตั้งแต่ต้น:

1. **Router** - Prefix tree routing พร้อม URL parameters และ wildcards
2. **Context** - Request/Response abstraction ที่ใช้งานง่าย
3. **Middleware** - Logger, Recovery, CORS, Rate Limiting, Auth
4. **Validation** - Struct validation ด้วย tags
5. **Integration** - การรวมทุกส่วนเข้าด้วยกัน

### Key Takeaways

- **Trie-based routing** ดีกว่า linear search สำหรับ routes จำนวนมาก
- **Middleware chain** ทำให้ cross-cutting concerns แยกจาก business logic
- **Context object** เป็น central point สำหรับ request/response data
- **Struct tags** ทำให้ validation เป็น declarative และง่ายต่อการอ่าน
