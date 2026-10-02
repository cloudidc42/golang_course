# Part 53: API Gateway

## เป้าหมายการเรียนรู้
- เข้าใจ API Gateway pattern
- สร้าง Custom API Gateway ด้วย Go
- ทำ Request routing
- Authentication ที่ gateway
- Rate limiting
- Request/Response transformation
- Response aggregation

---

## 1. API Gateway Pattern

API Gateway ทำหน้าที่เป็น "ประตูหน้าบ้าน" ของ microservices ระบบ - รับ request จาก client แล้วส่งต่อไปยัง services ที่เกี่ยวข้อง

```
Client --> API Gateway --> User Service
                      --> Order Service
                      --> Product Service
```

### ประโยชน์ของ API Gateway
1. **Single entry point** - client ไม่ต้องรู้ที่อยู่ของ services ต่างๆ
2. **Cross-cutting concerns** - auth, logging, rate limiting ทำที่เดียว
3. **Protocol translation** - รับ REST แล้วส่งต่อเป็น gRPC ได้
4. **Response aggregation** - รวม response จากหลาย services

```go
// gateway/gateway.go
package gateway

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"strings"
	"time"
)

// Route defines a routing rule
type Route struct {
	Path        string
	ServiceName string
	ServiceURL  string
	Methods     []string
	Middlewares []Middleware
	StripPrefix bool
}

type Middleware func(http.Handler) http.Handler

// Gateway is the main API gateway
type Gateway struct {
	routes     []Route
	middleware []Middleware
	discovery  ServiceDiscoverer
}

type ServiceDiscoverer interface {
	Discover(ctx context.Context, name string) (string, error)
}

func New(discovery ServiceDiscoverer) *Gateway {
	return &Gateway{
		discovery: discovery,
	}
}

func (g *Gateway) Use(m Middleware) {
	g.middleware = append(g.middleware, m)
}

func (g *Gateway) AddRoute(r Route) {
	g.routes = append(g.routes, r)
}

func (g *Gateway) Handler() http.Handler {
	mux := http.NewServeMux()

	for _, route := range g.routes {
		r := route // capture for closure
		handler := g.buildHandler(r)

		// Apply route-level middleware
		for i := len(r.Middlewares) - 1; i >= 0; i-- {
			handler = r.Middlewares[i](handler)
		}

		mux.HandleFunc(r.Path, func(w http.ResponseWriter, req *http.Request) {
			// Check method
			if len(r.Methods) > 0 {
				allowed := false
				for _, m := range r.Methods {
					if m == req.Method {
						allowed = true
						break
					}
				}
				if !allowed {
					http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
					return
				}
			}
			handler.ServeHTTP(w, req)
		})
	}

	// Apply global middleware
	var h http.Handler = mux
	for i := len(g.middleware) - 1; i >= 0; i-- {
		h = g.middleware[i](h)
	}
	return h
}

func (g *Gateway) buildHandler(route Route) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Discover service URL
		serviceURL := route.ServiceURL
		if serviceURL == "" && g.discovery != nil {
			var err error
			serviceURL, err = g.discovery.Discover(r.Context(), route.ServiceName)
			if err != nil {
				log.Printf("Service discovery failed for %s: %v", route.ServiceName, err)
				http.Error(w, "Service Unavailable", http.StatusServiceUnavailable)
				return
			}
		}

		target, err := url.Parse(serviceURL)
		if err != nil {
			http.Error(w, "Invalid service URL", http.StatusInternalServerError)
			return
		}

		proxy := httputil.NewSingleHostReverseProxy(target)
		proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
			log.Printf("Proxy error: %v", err)
			http.Error(w, "Bad Gateway", http.StatusBadGateway)
		}

		// Strip prefix if needed
		if route.StripPrefix {
			r.URL.Path = strings.TrimPrefix(r.URL.Path, route.Path)
			if r.URL.Path == "" {
				r.URL.Path = "/"
			}
		}

		proxy.ServeHTTP(w, r)
	})
}
```

---

## 2. Authentication Middleware

```go
// gateway/auth/jwt.go
package auth

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

type Claims struct {
	UserID string   `json:"user_id"`
	Email  string   `json:"email"`
	Roles  []string `json:"roles"`
	jwt.RegisteredClaims
}

type JWTConfig struct {
	SecretKey     []byte
	TokenExpiry   time.Duration
	RefreshExpiry time.Duration
}

type JWTMiddleware struct {
	config JWTConfig
}

func NewJWTMiddleware(config JWTConfig) *JWTMiddleware {
	return &JWTMiddleware{config: config}
}

func (m *JWTMiddleware) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token, err := m.extractToken(r)
		if err != nil {
			http.Error(w, "Unauthorized: "+err.Error(), http.StatusUnauthorized)
			return
		}

		claims, err := m.validateToken(token)
		if err != nil {
			http.Error(w, "Unauthorized: invalid token", http.StatusUnauthorized)
			return
		}

		// Add claims to context
		ctx := context.WithValue(r.Context(), claimsKey{}, claims)
		
		// Add user info to headers for downstream services
		r.Header.Set("X-User-ID", claims.UserID)
		r.Header.Set("X-User-Email", claims.Email)
		r.Header.Set("X-User-Roles", strings.Join(claims.Roles, ","))

		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

type claimsKey struct{}

func GetClaims(ctx context.Context) (*Claims, bool) {
	claims, ok := ctx.Value(claimsKey{}).(*Claims)
	return claims, ok
}

func (m *JWTMiddleware) extractToken(r *http.Request) (string, error) {
	// Check Authorization header
	authHeader := r.Header.Get("Authorization")
	if authHeader != "" {
		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) == 2 && parts[0] == "Bearer" {
			return parts[1], nil
		}
	}

	// Check cookie
	cookie, err := r.Cookie("access_token")
	if err == nil {
		return cookie.Value, nil
	}

	// Check query parameter (not recommended for production)
	if token := r.URL.Query().Get("access_token"); token != "" {
		return token, nil
	}

	return "", errors.New("no token provided")
}

func (m *JWTMiddleware) validateToken(tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return m.config.SecretKey, nil
	})

	if err != nil {
		return nil, err
	}

	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid {
		return nil, errors.New("invalid token claims")
	}

	return claims, nil
}

func (m *JWTMiddleware) GenerateToken(userID, email string, roles []string) (string, error) {
	claims := &Claims{
		UserID: userID,
		Email:  email,
		Roles:  roles,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(m.config.TokenExpiry)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			ID:        fmt.Sprintf("%d", time.Now().UnixNano()),
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(m.config.SecretKey)
}

// Role-based authorization middleware
type RBACMiddleware struct {
	requiredRoles []string
}

func RequireRoles(roles ...string) func(http.Handler) http.Handler {
	m := &RBACMiddleware{requiredRoles: roles}
	return m.Middleware
}

func (m *RBACMiddleware) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims, ok := GetClaims(r.Context())
		if !ok {
			http.Error(w, "Forbidden: no auth context", http.StatusForbidden)
			return
		}

		roleMap := make(map[string]bool)
		for _, role := range claims.Roles {
			roleMap[role] = true
		}

		for _, required := range m.requiredRoles {
			if !roleMap[required] {
				http.Error(w, "Forbidden: insufficient permissions", http.StatusForbidden)
				return
			}
		}

		next.ServeHTTP(w, r)
	})
}
```

---

## 3. Rate Limiting

```go
// gateway/ratelimit/limiter.go
package ratelimit

import (
	"fmt"
	"net/http"
	"sync"
	"time"
)

// TokenBucket rate limiter
type TokenBucket struct {
	capacity   int
	tokens     float64
	refillRate float64 // tokens per second
	lastRefill time.Time
	mu         sync.Mutex
}

func NewTokenBucket(capacity int, refillRate float64) *TokenBucket {
	return &TokenBucket{
		capacity:   capacity,
		tokens:     float64(capacity),
		refillRate: refillRate,
		lastRefill: time.Now(),
	}
}

func (tb *TokenBucket) Allow() bool {
	tb.mu.Lock()
	defer tb.mu.Unlock()

	now := time.Now()
	elapsed := now.Sub(tb.lastRefill).Seconds()
	tb.tokens += elapsed * tb.refillRate
	if tb.tokens > float64(tb.capacity) {
		tb.tokens = float64(tb.capacity)
	}
	tb.lastRefill = now

	if tb.tokens >= 1 {
		tb.tokens--
		return true
	}
	return false
}

// Per-IP rate limiter
type IPRateLimiter struct {
	mu       sync.RWMutex
	limiters map[string]*TokenBucket
	capacity int
	rate     float64
	cleanup  *time.Ticker
}

func NewIPRateLimiter(capacity int, ratePerSecond float64) *IPRateLimiter {
	limiter := &IPRateLimiter{
		limiters: make(map[string]*TokenBucket),
		capacity: capacity,
		rate:     ratePerSecond,
		cleanup:  time.NewTicker(5 * time.Minute),
	}

	go limiter.cleanupLoop()
	return limiter
}

func (l *IPRateLimiter) Allow(ip string) bool {
	l.mu.RLock()
	bucket, ok := l.limiters[ip]
	l.mu.RUnlock()

	if !ok {
		l.mu.Lock()
		// Double-check after acquiring write lock
		if bucket, ok = l.limiters[ip]; !ok {
			bucket = NewTokenBucket(l.capacity, l.rate)
			l.limiters[ip] = bucket
		}
		l.mu.Unlock()
	}

	return bucket.Allow()
}

func (l *IPRateLimiter) cleanupLoop() {
	for range l.cleanup.C {
		l.mu.Lock()
		for ip, bucket := range l.limiters {
			bucket.mu.Lock()
			if bucket.tokens >= float64(bucket.capacity) {
				delete(l.limiters, ip)
			}
			bucket.mu.Unlock()
		}
		l.mu.Unlock()
	}
}

// Middleware
func (l *IPRateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ip := getClientIP(r)

		if !l.Allow(ip) {
			w.Header().Set("Retry-After", "1")
			w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", l.capacity))
			http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
			return
		}

		next.ServeHTTP(w, r)
	})
}

func getClientIP(r *http.Request) string {
	// Check X-Forwarded-For header
	if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
		parts := splitAndTrim(xff)
		if len(parts) > 0 {
			return parts[0]
		}
	}
	// Check X-Real-IP header
	if xrip := r.Header.Get("X-Real-IP"); xrip != "" {
		return xrip
	}
	// Fall back to RemoteAddr
	return r.RemoteAddr
}

func splitAndTrim(s string) []string {
	var result []string
	for _, p := range splitComma(s) {
		trimmed := trimSpace(p)
		if trimmed != "" {
			result = append(result, trimmed)
		}
	}
	return result
}

func splitComma(s string) []string {
	var result []string
	start := 0
	for i := 0; i < len(s); i++ {
		if s[i] == ',' {
			result = append(result, s[start:i])
			start = i + 1
		}
	}
	result = append(result, s[start:])
	return result
}

func trimSpace(s string) string {
	for len(s) > 0 && (s[0] == ' ' || s[0] == '\t') {
		s = s[1:]
	}
	for len(s) > 0 && (s[len(s)-1] == ' ' || s[len(s)-1] == '\t') {
		s = s[:len(s)-1]
	}
	return s
}
```

---

## 4. Request Transformation

```go
// gateway/transform/request.go
package transform

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// RequestTransformer แปลง request ก่อนส่งต่อ
type RequestTransformer struct {
	transformers []RequestTransformFunc
}

type RequestTransformFunc func(r *http.Request) error

func NewRequestTransformer(fns ...RequestTransformFunc) *RequestTransformer {
	return &RequestTransformer{transformers: fns}
}

func (t *RequestTransformer) Apply(r *http.Request) error {
	for _, fn := range t.transformers {
		if err := fn(r); err != nil {
			return err
		}
	}
	return nil
}

// AddHeader เพิ่ม header
func AddHeader(key, value string) RequestTransformFunc {
	return func(r *http.Request) error {
		r.Header.Set(key, value)
		return nil
	}
}

// RemoveHeader ลบ header
func RemoveHeader(key string) RequestTransformFunc {
	return func(r *http.Request) error {
		r.Header.Del(key)
		return nil
	}
}

// AddQueryParam เพิ่ม query parameter
func AddQueryParam(key, value string) RequestTransformFunc {
	return func(r *http.Request) error {
		q := r.URL.Query()
		q.Add(key, value)
		r.URL.RawQuery = q.Encode()
		return nil
	}
}

// RewriteBodyField แก้ไข field ใน JSON body
func RewriteBodyField(oldKey, newKey string) RequestTransformFunc {
	return func(r *http.Request) error {
		if r.Body == nil {
			return nil
		}

		body, err := io.ReadAll(r.Body)
		if err != nil {
			return fmt.Errorf("reading body: %w", err)
		}
		r.Body.Close()

		var data map[string]interface{}
		if err := json.Unmarshal(body, &data); err != nil {
			// Not JSON, restore original
			r.Body = io.NopCloser(bytes.NewBuffer(body))
			return nil
		}

		if val, ok := data[oldKey]; ok {
			data[newKey] = val
			delete(data, oldKey)
		}

		newBody, err := json.Marshal(data)
		if err != nil {
			return fmt.Errorf("marshaling body: %w", err)
		}

		r.Body = io.NopCloser(bytes.NewBuffer(newBody))
		r.ContentLength = int64(len(newBody))
		return nil
	}
}

// ResponseTransformer แปลง response
type ResponseTransformer struct {
	transformers []ResponseTransformFunc
}

type ResponseTransformFunc func(w http.ResponseWriter, r *http.Request, body []byte) ([]byte, error)

func NewResponseTransformer(fns ...ResponseTransformFunc) *ResponseTransformer {
	return &ResponseTransformer{transformers: fns}
}

// RecordingResponseWriter บันทึก response สำหรับ transformation
type RecordingResponseWriter struct {
	http.ResponseWriter
	StatusCode int
	Body       bytes.Buffer
}

func NewRecordingResponseWriter(w http.ResponseWriter) *RecordingResponseWriter {
	return &RecordingResponseWriter{ResponseWriter: w}
}

func (r *RecordingResponseWriter) WriteHeader(statusCode int) {
	r.StatusCode = statusCode
	r.ResponseWriter.WriteHeader(statusCode)
}

func (r *RecordingResponseWriter) Write(b []byte) (int, error) {
	r.Body.Write(b)
	return r.ResponseWriter.Write(b)
}

// AddFieldToResponse เพิ่ม field ใน JSON response
func AddFieldToResponse(key string, value interface{}) ResponseTransformFunc {
	return func(w http.ResponseWriter, r *http.Request, body []byte) ([]byte, error) {
		var data map[string]interface{}
		if err := json.Unmarshal(body, &data); err != nil {
			return body, nil // ไม่ใช่ JSON
		}
		data[key] = value
		return json.Marshal(data)
	}
}
```

---

## 5. Response Aggregation

```go
// gateway/aggregation/aggregator.go
package aggregation

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
	"time"
)

// ServiceCall แทน call ไปยัง downstream service
type ServiceCall struct {
	Name    string
	URL     string
	Method  string
	Headers map[string]string
}

// AggregationResult ผลลัพธ์จาก service call
type AggregationResult struct {
	Name    string
	Data    interface{}
	Error   error
	Latency time.Duration
}

// Aggregator รวม responses จากหลาย services
type Aggregator struct {
	client  *http.Client
	timeout time.Duration
}

func NewAggregator(timeout time.Duration) *Aggregator {
	return &Aggregator{
		client:  &http.Client{Timeout: timeout},
		timeout: timeout,
	}
}

func (a *Aggregator) Aggregate(ctx context.Context, calls []ServiceCall) map[string]AggregationResult {
	ctx, cancel := context.WithTimeout(ctx, a.timeout)
	defer cancel()

	results := make(map[string]AggregationResult)
	var (
		wg sync.WaitGroup
		mu sync.Mutex
	)

	for _, call := range calls {
		wg.Add(1)
		go func(sc ServiceCall) {
			defer wg.Done()

			start := time.Now()
			result := AggregationResult{Name: sc.Name}

			data, err := a.callService(ctx, sc)
			result.Data = data
			result.Error = err
			result.Latency = time.Since(start)

			mu.Lock()
			results[sc.Name] = result
			mu.Unlock()
		}(call)
	}

	wg.Wait()
	return results
}

func (a *Aggregator) callService(ctx context.Context, call ServiceCall) (interface{}, error) {
	method := call.Method
	if method == "" {
		method = "GET"
	}

	req, err := http.NewRequestWithContext(ctx, method, call.URL, nil)
	if err != nil {
		return nil, fmt.Errorf("creating request: %w", err)
	}

	for k, v := range call.Headers {
		req.Header.Set(k, v)
	}

	resp, err := a.client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("executing request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 400 {
		return nil, fmt.Errorf("service returned %d", resp.StatusCode)
	}

	var data interface{}
	if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
		return nil, fmt.Errorf("decoding response: %w", err)
	}
	return data, nil
}

// BFF (Backend for Frontend) handler
type BFFHandler struct {
	aggregator *Aggregator
	userURL    string
	orderURL   string
	productURL string
}

func NewBFFHandler(aggregator *Aggregator, userURL, orderURL, productURL string) *BFFHandler {
	return &BFFHandler{
		aggregator: aggregator,
		userURL:    userURL,
		orderURL:   orderURL,
		productURL: productURL,
	}
}

// GetDashboard รวบรวมข้อมูลจากหลาย services สำหรับหน้า dashboard
func (h *BFFHandler) GetDashboard(w http.ResponseWriter, r *http.Request) {
	userID := r.URL.Query().Get("user_id")
	if userID == "" {
		http.Error(w, "user_id required", http.StatusBadRequest)
		return
	}

	calls := []ServiceCall{
		{
			Name:   "user",
			URL:    fmt.Sprintf("%s/users/%s", h.userURL, userID),
			Method: "GET",
		},
		{
			Name:   "orders",
			URL:    fmt.Sprintf("%s/orders?user_id=%s&limit=5", h.orderURL, userID),
			Method: "GET",
		},
		{
			Name:   "recommendations",
			URL:    fmt.Sprintf("%s/recommendations?user_id=%s", h.productURL, userID),
			Method: "GET",
		},
	}

	results := h.aggregator.Aggregate(r.Context(), calls)

	// สร้าง unified response
	dashboard := map[string]interface{}{
		"user_id": userID,
	}

	for name, result := range results {
		if result.Error != nil {
			dashboard[name] = map[string]interface{}{
				"error":   result.Error.Error(),
				"latency": result.Latency.Milliseconds(),
			}
		} else {
			dashboard[name] = map[string]interface{}{
				"data":    result.Data,
				"latency": result.Latency.Milliseconds(),
			}
		}
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(dashboard)
}
```

---

## 6. Gateway Logging และ Monitoring

```go
// gateway/middleware/logging.go
package middleware

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"
)

type ResponseRecorder struct {
	http.ResponseWriter
	StatusCode int
	Body       bytes.Buffer
	Written    int64
}

func NewResponseRecorder(w http.ResponseWriter) *ResponseRecorder {
	return &ResponseRecorder{
		ResponseWriter: w,
		StatusCode:     http.StatusOK,
	}
}

func (r *ResponseRecorder) WriteHeader(code int) {
	r.StatusCode = code
	r.ResponseWriter.WriteHeader(code)
}

func (r *ResponseRecorder) Write(b []byte) (int, error) {
	n, err := r.ResponseWriter.Write(b)
	r.Written += int64(n)
	return n, err
}

// LoggingMiddleware logs request/response details
func LoggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// Generate request ID
		reqID := generateRequestID()
		r.Header.Set("X-Request-ID", reqID)
		w.Header().Set("X-Request-ID", reqID)

		recorder := NewResponseRecorder(w)

		defer func() {
			duration := time.Since(start)
			log.Printf(
				"[%s] %s %s %s %d %d %v %s",
				reqID,
				r.RemoteAddr,
				r.Method,
				r.URL.Path,
				recorder.StatusCode,
				recorder.Written,
				duration,
				r.UserAgent(),
			)
		}()

		next.ServeHTTP(recorder, r)
	})
}

func generateRequestID() string {
	return fmt.Sprintf("%d", time.Now().UnixNano())
}

// TracingMiddleware adds trace context
func TracingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		traceID := r.Header.Get("X-Trace-ID")
		if traceID == "" {
			traceID = generateRequestID()
		}
		spanID := generateRequestID()

		r.Header.Set("X-Trace-ID", traceID)
		r.Header.Set("X-Span-ID", spanID)
		w.Header().Set("X-Trace-ID", traceID)

		next.ServeHTTP(w, r)
	})
}

// CORSMiddleware handles CORS
func CORSMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
	originMap := make(map[string]bool)
	for _, o := range allowedOrigins {
		originMap[o] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")
			if originMap[origin] || originMap["*"] {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH, OPTIONS")
				w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
				w.Header().Set("Access-Control-Max-Age", "86400")
			}

			if r.Method == "OPTIONS" {
				w.WriteHeader(http.StatusNoContent)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// RequestSizeLimit จำกัดขนาด request body
func RequestSizeLimitMiddleware(maxBytes int64) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
			if _, err := io.ReadAll(r.Body); err != nil {
				http.Error(w, "Request Too Large", http.StatusRequestEntityTooLarge)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

---

## 7. Complete API Gateway

```go
// gateway/main.go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gorilla/mux"
)

type Config struct {
	Port           string
	JWTSecret      string
	RateLimit      int
	UserServiceURL string
	OrderServiceURL string
	ProductServiceURL string
}

func loadConfig() Config {
	getEnv := func(key, def string) string {
		if v := os.Getenv(key); v != "" {
			return v
		}
		return def
	}
	return Config{
		Port:              getEnv("PORT", "8080"),
		JWTSecret:         getEnv("JWT_SECRET", "super-secret-key"),
		RateLimit:         100,
		UserServiceURL:    getEnv("USER_SERVICE_URL", "http://localhost:8081"),
		OrderServiceURL:   getEnv("ORDER_SERVICE_URL", "http://localhost:8082"),
		ProductServiceURL: getEnv("PRODUCT_SERVICE_URL", "http://localhost:8083"),
	}
}

func main() {
	cfg := loadConfig()

	r := mux.NewRouter()

	// Global middleware
	r.Use(loggingMiddleware)
	r.Use(corsMiddleware)
	r.Use(rateLimitMiddleware(cfg.RateLimit))

	// Public routes
	r.HandleFunc("/health", healthHandler).Methods("GET")
	r.HandleFunc("/auth/login", loginHandler(cfg.JWTSecret)).Methods("POST")

	// Protected routes
	api := r.PathPrefix("/api/v1").Subrouter()
	api.Use(jwtMiddleware(cfg.JWTSecret))

	// User routes
	api.PathPrefix("/users").HandlerFunc(proxyHandler(cfg.UserServiceURL, "/api/v1"))
	
	// Order routes
	api.PathPrefix("/orders").HandlerFunc(proxyHandler(cfg.OrderServiceURL, "/api/v1"))
	
	// Product routes (public read, protected write)
	r.PathPrefix("/api/v1/products").Methods("GET").HandlerFunc(proxyHandler(cfg.ProductServiceURL, "/api/v1"))
	api.PathPrefix("/products").Methods("POST", "PUT", "DELETE").HandlerFunc(
		requireRole("admin", proxyHandler(cfg.ProductServiceURL, "/api/v1")),
	)

	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      r,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	go func() {
		log.Printf("API Gateway starting on port %s", cfg.Port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("ListenAndServe error: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("Server shutdown error: %v", err)
	}
	log.Println("API Gateway stopped")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status":  "healthy",
		"service": "api-gateway",
		"time":    time.Now().Format(time.RFC3339),
	})
}

func loginHandler(jwtSecret string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req struct {
			Email    string `json:"email"`
			Password string `json:"password"`
		}
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "Invalid request", http.StatusBadRequest)
			return
		}

		// Mock validation - ใน production ต้อง call user service
		if req.Email == "admin@example.com" && req.Password == "password" {
			// Generate token
			token := "mock.jwt.token"
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(map[string]string{
				"access_token": token,
				"token_type":   "Bearer",
			})
			return
		}
		http.Error(w, "Invalid credentials", http.StatusUnauthorized)
	}
}

func proxyHandler(targetURL, stripPrefix string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "Proxy not implemented in this demo", http.StatusNotImplemented)
	}
}

func jwtMiddleware(secret string) mux.MiddlewareFunc {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			token := r.Header.Get("Authorization")
			if token == "" {
				http.Error(w, "Unauthorized", http.StatusUnauthorized)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

func requireRole(role string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Check role from context (set by JWT middleware)
		next(w, r)
	}
}

func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
	})
}

func corsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
		if r.Method == "OPTIONS" {
			w.WriteHeader(http.StatusNoContent)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func rateLimitMiddleware(rps int) mux.MiddlewareFunc {
	limiter := NewTokenBucket(rps, float64(rps))
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if !limiter.Allow() {
				http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

// Simple token bucket for demo
type TokenBucket struct {
	capacity   int
	tokens     float64
	refillRate float64
	lastRefill time.Time
	mu         chan struct{}
}

func NewTokenBucket(capacity int, rate float64) *TokenBucket {
	mu := make(chan struct{}, 1)
	mu <- struct{}{}
	return &TokenBucket{
		capacity:   capacity,
		tokens:     float64(capacity),
		refillRate: rate,
		lastRefill: time.Now(),
		mu:         mu,
	}
}

func (tb *TokenBucket) Allow() bool {
	<-tb.mu
	defer func() { tb.mu <- struct{}{} }()

	now := time.Now()
	elapsed := now.Sub(tb.lastRefill).Seconds()
	tb.tokens += elapsed * tb.refillRate
	if tb.tokens > float64(tb.capacity) {
		tb.tokens = float64(tb.capacity)
	}
	tb.lastRefill = now

	if tb.tokens >= 1 {
		tb.tokens--
		return true
	}
	return false
}
```

---

## สรุป

| Component | หน้าที่ |
|-----------|--------|
| Route Handler | map URL paths ไปยัง backend services |
| Auth Middleware | ตรวจสอบ JWT token |
| Rate Limiter | ป้องกัน abuse |
| Request Transform | แปลง request ก่อนส่งต่อ |
| Response Transform | แปลง response ก่อนส่งกลับ |
| Aggregator | รวม responses จากหลาย services |
| Logging/Tracing | บันทึกและ trace requests |

---

**ต่อไป**: Part 54 - Load Balancing
