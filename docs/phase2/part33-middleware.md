# Part 33: Middleware ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Middleware concept
- เขียน Middleware สำหรับ Gin framework
- สร้าง Custom Middleware
- ทำ CORS middleware
- ทำ Rate Limiting middleware
- ทำ Authentication middleware
- ทำ Request Logging middleware
- ทำ Recovery middleware
- ทำ Compression middleware

---

## 33.1 Middleware Concept

Middleware คือ function ที่ทำงานระหว่าง request และ response สามารถ:
- แก้ไข request ก่อนส่งต่อ handler
- แก้ไข response หลัง handler
- หยุด request chain
- เพิ่ม logic ที่ใช้ร่วมกันหลาย handlers

```
Request → [Middleware 1] → [Middleware 2] → [Handler] → [Middleware 2] → [Middleware 1] → Response
```

### Middleware Pattern ใน Go (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// Middleware type สำหรับ net/http
type Middleware func(http.Handler) http.Handler

// Logging middleware
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        fmt.Printf("[%s] %s %s started\n", start.Format(time.RFC3339), r.Method, r.URL.Path)
        
        next.ServeHTTP(w, r)
        
        fmt.Printf("[%s] %s %s completed in %v\n",
            time.Now().Format(time.RFC3339),
            r.Method, r.URL.Path,
            time.Since(start))
    })
}

// Chain middlewares
func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello World!")
    })
    
    handler := Chain(mux, LoggingMiddleware)
    http.ListenAndServe(":8080", handler)
}
```

---

## 33.2 Gin Middleware

### Basic Gin Middleware (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
)

// Middleware function signature: func(*gin.Context)
func TimingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        // ก่อน handler
        c.Next() // เรียก next middleware หรือ handler
        
        // หลัง handler
        latency := time.Since(start)
        fmt.Printf("Path: %s, Latency: %v\n", c.Request.URL.Path, latency)
        
        // เพิ่ม header ใน response
        c.Header("X-Response-Time", latency.String())
    }
}

func VersionMiddleware(version string) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-API-Version", version)
        c.Next()
    }
}

func main() {
    r := gin.New() // gin.New() ไม่มี default middleware
    
    // Global middleware
    r.Use(gin.Logger())
    r.Use(gin.Recovery())
    r.Use(TimingMiddleware())
    r.Use(VersionMiddleware("1.0.0"))
    
    r.GET("/", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "Hello!"})
    })
    
    r.Run(":8080")
}
```

### c.Abort vs c.Next (ตัวอย่างที่ 3)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        
        if token == "" {
            // หยุด chain ทันที (ไม่เรียก next handlers)
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authorization required",
            })
            return
        }
        
        if token != "Bearer valid-token" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid token",
            })
            return
        }
        
        // เก็บข้อมูลใน context สำหรับ handlers ถัดไป
        c.Set("user_id", "user-123")
        c.Set("user_role", "admin")
        
        c.Next() // ผ่านไป next handler
    }
}

func main() {
    r := gin.Default()
    
    public := r.Group("/public")
    {
        public.GET("/info", func(c *gin.Context) {
            c.JSON(http.StatusOK, gin.H{"message": "Public endpoint"})
        })
    }
    
    private := r.Group("/private")
    private.Use(AuthRequired()) // Apply middleware เฉพาะ group นี้
    {
        private.GET("/dashboard", func(c *gin.Context) {
            userID, _ := c.Get("user_id")
            role, _ := c.Get("user_role")
            c.JSON(http.StatusOK, gin.H{
                "user_id": userID,
                "role":    role,
                "message": "Private dashboard",
            })
        })
    }
    
    r.Run(":8080")
}
```

---

## 33.3 CORS Middleware

### Custom CORS Middleware (ตัวอย่างที่ 4)

```go
package main

import (
    "net/http"
    "strings"
    
    "github.com/gin-gonic/gin"
)

type CORSConfig struct {
    AllowedOrigins   []string
    AllowedMethods   []string
    AllowedHeaders   []string
    ExposedHeaders   []string
    AllowCredentials bool
    MaxAge           int
}

func DefaultCORSConfig() CORSConfig {
    return CORSConfig{
        AllowedOrigins:   []string{"*"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
        ExposedHeaders:   []string{"X-Request-ID", "X-Response-Time"},
        AllowCredentials: false,
        MaxAge:           86400,
    }
}

func CORS(config CORSConfig) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.GetHeader("Origin")
        
        // ตรวจสอบว่า origin อนุญาตไหม
        allowOrigin := ""
        for _, allowed := range config.AllowedOrigins {
            if allowed == "*" || allowed == origin {
                allowOrigin = origin
                if allowed == "*" {
                    allowOrigin = "*"
                }
                break
            }
        }
        
        if allowOrigin == "" {
            c.Next()
            return
        }
        
        // ตั้งค่า CORS headers
        c.Header("Access-Control-Allow-Origin", allowOrigin)
        c.Header("Access-Control-Allow-Methods", strings.Join(config.AllowedMethods, ", "))
        c.Header("Access-Control-Allow-Headers", strings.Join(config.AllowedHeaders, ", "))
        
        if len(config.ExposedHeaders) > 0 {
            c.Header("Access-Control-Expose-Headers", strings.Join(config.ExposedHeaders, ", "))
        }
        
        if config.AllowCredentials {
            c.Header("Access-Control-Allow-Credentials", "true")
        }
        
        if config.MaxAge > 0 {
            c.Header("Access-Control-Max-Age", fmt.Sprintf("%d", config.MaxAge))
        }
        
        // Handle preflight
        if c.Request.Method == http.MethodOptions {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        
        c.Next()
    }
}

import "fmt"

func main() {
    r := gin.Default()
    
    // Apply CORS with custom config
    r.Use(CORS(CORSConfig{
        AllowedOrigins:   []string{"http://localhost:3000", "https://myapp.com"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE"},
        AllowedHeaders:   []string{"Authorization", "Content-Type"},
        AllowCredentials: true,
    }))
    
    r.GET("/api/users", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"users": []string{"Alice", "Bob"}})
    })
    
    r.Run(":8080")
}
```

### ใช้ gin-contrib/cors (ตัวอย่างที่ 5)

```bash
go get github.com/gin-contrib/cors
```

```go
package main

import (
    "time"
    
    "github.com/gin-contrib/cors"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // ง่ายสุด - allow all
    // r.Use(cors.Default())
    
    // Custom config
    r.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"http://localhost:3000", "https://myapp.com"},
        AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
        ExposeHeaders:    []string{"Content-Length", "X-Request-ID"},
        AllowCredentials: true,
        MaxAge:           12 * time.Hour,
    }))
    
    r.GET("/api/data", func(c *gin.Context) {
        c.JSON(200, gin.H{"data": "hello"})
    })
    
    r.Run(":8080")
}
```

---

## 33.4 Rate Limiting Middleware

### Token Bucket Rate Limiter (ตัวอย่างที่ 6)

```go
package main

import (
    "net/http"
    "sync"
    "time"
    
    "github.com/gin-gonic/gin"
)

// TokenBucket implements token bucket rate limiting
type TokenBucket struct {
    rate     float64   // tokens per second
    capacity float64   // max tokens
    tokens   float64
    lastTime time.Time
    mu       sync.Mutex
}

func NewTokenBucket(rate, capacity float64) *TokenBucket {
    return &TokenBucket{
        rate:     rate,
        capacity: capacity,
        tokens:   capacity,
        lastTime: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()
    tb.tokens += elapsed * tb.rate
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastTime = now
    
    if tb.tokens < 1 {
        return false
    }
    tb.tokens--
    return true
}

// IP-based Rate Limiter
type IPRateLimiter struct {
    limiters map[string]*TokenBucket
    mu       sync.RWMutex
    rate     float64
    capacity float64
}

func NewIPRateLimiter(rate, capacity float64) *IPRateLimiter {
    return &IPRateLimiter{
        limiters: make(map[string]*TokenBucket),
        rate:     rate,
        capacity: capacity,
    }
}

func (l *IPRateLimiter) GetLimiter(ip string) *TokenBucket {
    l.mu.RLock()
    limiter, exists := l.limiters[ip]
    l.mu.RUnlock()
    
    if !exists {
        l.mu.Lock()
        limiter = NewTokenBucket(l.rate, l.capacity)
        l.limiters[ip] = limiter
        l.mu.Unlock()
    }
    
    return limiter
}

func RateLimitMiddleware(limiter *IPRateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        // ดึง client IP
        ip := c.ClientIP()
        
        if !limiter.GetLimiter(ip).Allow() {
            c.Header("Retry-After", "1")
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error":   "Too Many Requests",
                "message": "Rate limit exceeded. Please wait before retrying.",
            })
            return
        }
        
        c.Next()
    }
}

func main() {
    r := gin.Default()
    
    // 10 requests per second, burst of 20
    limiter := NewIPRateLimiter(10, 20)
    r.Use(RateLimitMiddleware(limiter))
    
    r.GET("/api/data", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"data": "success"})
    })
    
    r.Run(":8080")
}
```

### Rate Limiting ด้วย golang.org/x/time/rate (ตัวอย่างที่ 7)

```bash
go get golang.org/x/time/rate
```

```go
package main

import (
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

type Client struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

type RateLimiterStore struct {
    clients map[string]*Client
    mu      sync.Mutex
    r       rate.Limit // requests per second
    b       int        // burst
}

func NewRateLimiterStore(r rate.Limit, b int) *RateLimiterStore {
    store := &RateLimiterStore{
        clients: make(map[string]*Client),
        r:       r,
        b:       b,
    }
    
    // Cleanup goroutine
    go store.cleanup()
    return store
}

func (s *RateLimiterStore) GetLimiter(ip string) *rate.Limiter {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    client, ok := s.clients[ip]
    if !ok {
        client = &Client{
            limiter: rate.NewLimiter(s.r, s.b),
        }
        s.clients[ip] = client
    }
    
    client.lastSeen = time.Now()
    return client.limiter
}

func (s *RateLimiterStore) cleanup() {
    for {
        time.Sleep(time.Minute)
        s.mu.Lock()
        for ip, client := range s.clients {
            if time.Since(client.lastSeen) > 3*time.Minute {
                delete(s.clients, ip)
            }
        }
        s.mu.Unlock()
    }
}

func RateLimit(store *RateLimiterStore) gin.HandlerFunc {
    return func(c *gin.Context) {
        limiter := store.GetLimiter(c.ClientIP())
        
        if !limiter.Allow() {
            retryAfter := fmt.Sprintf("%.0f", 1/float64(store.r))
            c.Header("Retry-After", retryAfter)
            c.Header("X-RateLimit-Limit", fmt.Sprintf("%d", store.b))
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
            })
            return
        }
        
        c.Next()
    }
}

func main() {
    store := NewRateLimiterStore(rate.Limit(5), 10) // 5 req/s, burst 10
    
    r := gin.Default()
    r.Use(RateLimit(store))
    
    r.GET("/api/endpoint", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    
    r.Run(":8080")
}
```

---

## 33.5 Authentication Middleware

### JWT Auth Middleware (ตัวอย่างที่ 8)

```go
package main

import (
    "net/http"
    "strings"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("my-secret-key")

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func JWTAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authorization header required",
            })
            return
        }
        
        // Format: "Bearer <token>"
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid authorization format",
            })
            return
        }
        
        tokenString := parts[1]
        
        // Parse และ validate token
        claims := &Claims{}
        token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, jwt.ErrSignatureInvalid
            }
            return jwtSecret, nil
        })
        
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid or expired token",
            })
            return
        }
        
        // เก็บ claims ใน context
        c.Set("user_id", claims.UserID)
        c.Set("user_role", claims.Role)
        c.Set("claims", claims)
        
        c.Next()
    }
}

// Role-based middleware
func RequireRole(roles ...string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole, exists := c.Get("user_role")
        if !exists {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "No role found"})
            return
        }
        
        for _, role := range roles {
            if role == userRole {
                c.Next()
                return
            }
        }
        
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "Insufficient permissions",
        })
    }
}

func main() {
    r := gin.Default()
    
    // Auth endpoint (no middleware)
    r.POST("/auth/login", func(c *gin.Context) {
        // Simplified authentication
        token, _ := generateToken("user-123", "admin")
        c.JSON(http.StatusOK, gin.H{"token": token})
    })
    
    // Protected routes
    protected := r.Group("/api")
    protected.Use(JWTAuthMiddleware())
    {
        protected.GET("/profile", func(c *gin.Context) {
            userID, _ := c.Get("user_id")
            c.JSON(http.StatusOK, gin.H{"user_id": userID})
        })
        
        // Admin only
        admin := protected.Group("/admin")
        admin.Use(RequireRole("admin"))
        {
            admin.GET("/users", func(c *gin.Context) {
                c.JSON(http.StatusOK, gin.H{"message": "Admin users list"})
            })
        }
    }
    
    r.Run(":8080")
}
```

---

## 33.6 Request Logging Middleware

### Structured Request Logging (ตัวอย่างที่ 9)

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "log/slog"
    "net/http"
    "os"
    "time"
    
    "github.com/gin-gonic/gin"
)

// ResponseWriter wrapper เพื่อดักจับ status code
type responseWriter struct {
    gin.ResponseWriter
    statusCode int
    bodySize   int
}

func newResponseWriter(w gin.ResponseWriter) *responseWriter {
    return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bodySize += n
    return n, err
}

func RequestLogger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        // Wrap response writer
        rw := newResponseWriter(c.Writer)
        c.Writer = rw
        
        // Request ID
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = fmt.Sprintf("req-%d", time.Now().UnixNano())
        }
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)
        
        // Read body for logging (be careful with large bodies)
        var bodyBytes []byte
        if c.Request.Body != nil && c.Request.ContentLength > 0 && c.Request.ContentLength < 1024 {
            bodyBytes, _ = io.ReadAll(c.Request.Body)
            c.Request.Body = io.NopCloser(bytes.NewBuffer(bodyBytes))
        }
        
        c.Next()
        
        latency := time.Since(start)
        
        // Log fields
        attrs := []any{
            slog.String("request_id", requestID),
            slog.String("method", c.Request.Method),
            slog.String("path", c.Request.URL.Path),
            slog.String("query", c.Request.URL.RawQuery),
            slog.String("ip", c.ClientIP()),
            slog.String("user_agent", c.Request.UserAgent()),
            slog.Int("status", rw.statusCode),
            slog.Int("response_size", rw.bodySize),
            slog.Duration("latency", latency),
        }
        
        if len(bodyBytes) > 0 {
            attrs = append(attrs, slog.String("request_body", string(bodyBytes)))
        }
        
        // เลือก log level ตาม status code
        level := slog.LevelInfo
        if rw.statusCode >= 500 {
            level = slog.LevelError
        } else if rw.statusCode >= 400 {
            level = slog.LevelWarn
        }
        
        logger.Log(c.Request.Context(), level, "HTTP Request", attrs...)
    }
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))
    
    r := gin.New()
    r.Use(gin.Recovery())
    r.Use(RequestLogger(logger))
    
    r.GET("/api/users", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"users": []string{"Alice", "Bob"}})
    })
    
    r.POST("/api/users", func(c *gin.Context) {
        c.JSON(http.StatusCreated, gin.H{"message": "Created"})
    })
    
    r.GET("/api/error", func(c *gin.Context) {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Something went wrong"})
    })
    
    r.Run(":8080")
}
```

---

## 33.7 Recovery Middleware

### Custom Recovery Middleware (ตัวอย่างที่ 10)

```go
package main

import (
    "fmt"
    "net/http"
    "runtime/debug"
    
    "github.com/gin-gonic/gin"
)

func RecoveryMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                // Log stack trace
                stack := debug.Stack()
                fmt.Printf("PANIC recovered: %v\n%s\n", err, string(stack))
                
                // ส่ง error response
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error":   "Internal Server Error",
                    "message": "An unexpected error occurred",
                    "request_id": c.GetString("request_id"),
                })
            }
        }()
        
        c.Next()
    }
}

func main() {
    r := gin.New()
    
    // ใช้ custom recovery แทน default
    r.Use(RecoveryMiddleware())
    
    r.GET("/safe", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "This is safe"})
    })
    
    r.GET("/panic", func(c *gin.Context) {
        // Simulate panic
        panic("something went terribly wrong!")
    })
    
    r.GET("/nil-panic", func(c *gin.Context) {
        var s *string
        _ = *s // nil pointer dereference
    })
    
    r.Run(":8080")
}
```

---

## 33.8 Compression Middleware

### GZIP Compression (ตัวอย่างที่ 11)

```bash
go get github.com/gin-contrib/gzip
```

```go
package main

import (
    "net/http"
    "strings"
    
    "github.com/gin-contrib/gzip"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // Gzip compression middleware
    r.Use(gzip.Gzip(gzip.DefaultCompression))
    
    // หรือ custom compression
    r.Use(gzip.Gzip(gzip.BestCompression, gzip.WithExcludedPaths([]string{"/health"})))
    
    r.GET("/large-data", func(c *gin.Context) {
        // Simulate large response
        data := strings.Repeat("Hello World! ", 1000)
        c.JSON(http.StatusOK, gin.H{
            "data":   data,
            "length": len(data),
        })
    })
    
    r.GET("/health", func(c *gin.Context) {
        // ไม่ compress (excluded)
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    
    r.Run(":8080")
}
```

---

## 33.9 Middleware สำหรับ Performance

### Request Timeout Middleware (ตัวอย่างที่ 12)

```go
package main

import (
    "context"
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
)

func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        // สร้าง context พร้อม timeout
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()
        
        // แทนที่ request context
        c.Request = c.Request.WithContext(ctx)
        
        // Channel สำหรับ done signal
        done := make(chan struct{})
        
        go func() {
            c.Next()
            close(done)
        }()
        
        select {
        case <-done:
            // Handler completed
        case <-ctx.Done():
            // Timeout!
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "error":   "Request Timeout",
                "message": "The request took too long to process",
            })
        }
    }
}

func main() {
    r := gin.Default()
    
    // 5 second timeout for all routes
    r.Use(TimeoutMiddleware(5 * time.Second))
    
    r.GET("/fast", func(c *gin.Context) {
        time.Sleep(100 * time.Millisecond)
        c.JSON(http.StatusOK, gin.H{"message": "Fast response"})
    })
    
    r.GET("/slow", func(c *gin.Context) {
        // Check if context is done
        select {
        case <-c.Request.Context().Done():
            return // Context cancelled
        case <-time.After(10 * time.Second): // This will timeout
        }
        c.JSON(http.StatusOK, gin.H{"message": "Slow response"})
    })
    
    r.Run(":8080")
}
```

---

## Workshop: Complete Middleware Stack

```go
package main

import (
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "strings"
    "time"
    
    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

// === Simple Rate Limiter ===
type SimpleLimiter struct {
    limiter *rate.Limiter
}

func NewSimpleLimiter(r rate.Limit, b int) *SimpleLimiter {
    return &SimpleLimiter{limiter: rate.NewLimiter(r, b)}
}

func (l *SimpleLimiter) Allow() bool {
    return l.limiter.Allow()
}

// === Middlewares ===

// 1. Request ID
func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = fmt.Sprintf("req-%d", time.Now().UnixNano())
        }
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}

// 2. Logging
func LoggingMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        
        logger.Info("http_request",
            slog.String("method", c.Request.Method),
            slog.String("path", c.FullPath()),
            slog.Int("status", c.Writer.Status()),
            slog.Duration("latency", time.Since(start)),
            slog.String("ip", c.ClientIP()),
            slog.String("request_id", c.GetString("request_id")),
        )
    }
}

// 3. Security Headers
func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Next()
    }
}

// 4. CORS
func CORSMiddleware(allowedOrigins []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.GetHeader("Origin")
        
        for _, allowed := range allowedOrigins {
            if allowed == "*" || allowed == origin {
                if allowed == "*" {
                    c.Header("Access-Control-Allow-Origin", "*")
                } else {
                    c.Header("Access-Control-Allow-Origin", origin)
                }
                break
            }
        }
        
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Authorization, Content-Type, X-Request-ID")
        
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        c.Next()
    }
}

// 5. Rate Limit
func RateLimitMiddleware(limiter *SimpleLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "Too Many Requests",
            })
            return
        }
        c.Next()
    }
}

// 6. Auth
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        auth := c.GetHeader("Authorization")
        if !strings.HasPrefix(auth, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Unauthorized",
            })
            return
        }
        token := auth[7:]
        // Simplified validation
        if token != "valid-token" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid token",
            })
            return
        }
        c.Set("user_id", "user-123")
        c.Next()
    }
}

// 7. Recovery
func RecoveryMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                logger.Error("panic_recovered",
                    slog.Any("error", err),
                    slog.String("request_id", c.GetString("request_id")),
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "Internal Server Error",
                })
            }
        }()
        c.Next()
    }
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    
    limiter := NewSimpleLimiter(rate.Limit(100), 200) // 100 req/s, burst 200
    
    r := gin.New()
    
    // Global middleware stack (order matters!)
    r.Use(RecoveryMiddleware(logger))       // 1. Recovery first (catch panics)
    r.Use(RequestIDMiddleware())            // 2. Request ID
    r.Use(LoggingMiddleware(logger))        // 3. Logging
    r.Use(SecurityHeaders())               // 4. Security headers
    r.Use(CORSMiddleware([]string{"*"}))   // 5. CORS
    r.Use(RateLimitMiddleware(limiter))    // 6. Rate limiting
    
    // Public routes
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":     "healthy",
            "request_id": c.GetString("request_id"),
        })
    })
    
    r.POST("/auth/login", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "token":      "valid-token",
            "expires_in": 86400,
        })
    })
    
    // Protected routes
    api := r.Group("/api/v1")
    api.Use(AuthMiddleware())
    {
        api.GET("/users", func(c *gin.Context) {
            c.JSON(http.StatusOK, gin.H{
                "users":      []string{"Alice", "Bob"},
                "request_id": c.GetString("request_id"),
            })
        })
        
        api.GET("/profile", func(c *gin.Context) {
            userID, _ := c.Get("user_id")
            c.JSON(http.StatusOK, gin.H{
                "user_id": userID,
                "name":    "Test User",
            })
        })
        
        api.GET("/panic", func(c *gin.Context) {
            panic("test panic - recovered by middleware!")
        })
    }
    
    fmt.Println("Server with middleware stack running on :8080")
    r.Run(":8080")
}
```

---

## สรุป Part 33

| Middleware | ใช้สำหรับ | สำคัญ |
|-----------|---------|------|
| Logger | บันทึก request/response | ตั้ง request ID ก่อน |
| Recovery | จัดการ panics | ต้องอยู่ต้น chain |
| CORS | Cross-origin requests | ต้องจัดการ OPTIONS |
| Auth | ตรวจ JWT/sessions | ใช้ c.AbortWith... เมื่อ fail |
| Rate Limit | จำกัด request rate | ใช้ per-IP limiter |
| Security Headers | HTTP security | ควรใส่ทุก app |
| Timeout | จำกัดเวลา request | ใช้ context.WithTimeout |
| Compression | ลด response size | ตรวจ Accept-Encoding |

### Resources
- [Gin Middleware](https://gin-gonic.com/docs/examples/using-middleware/)
- [gin-contrib](https://github.com/gin-contrib)
- [golang.org/x/time/rate](https://pkg.go.dev/golang.org/x/time/rate)
