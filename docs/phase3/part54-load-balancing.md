# Part 54: Load Balancing

## เป้าหมายการเรียนรู้
- เข้าใจ Load Balancing algorithms ต่างๆ
- ใช้ Go reverse proxy
- ทำ health checks
- Layer 4 vs Layer 7 load balancing
- Sticky sessions

---

## 1. Load Balancing Algorithms

```go
// loadbalancer/algorithms.go
package loadbalancer

import (
	"context"
	"crypto/md5"
	"fmt"
	"net/http"
	"sync"
	"sync/atomic"
	"time"
)

type Backend struct {
	URL         string
	Weight      int
	Healthy     bool
	ActiveConns int64
	mu          sync.RWMutex
	fails       int
	lastCheck   time.Time
}

func (b *Backend) IsHealthy() bool {
	b.mu.RLock()
	defer b.mu.RUnlock()
	return b.Healthy
}

func (b *Backend) SetHealthy(healthy bool) {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.Healthy = healthy
	if !healthy {
		b.fails++
	} else {
		b.fails = 0
	}
	b.lastCheck = time.Now()
}

func (b *Backend) IncrementConns() {
	atomic.AddInt64(&b.ActiveConns, 1)
}

func (b *Backend) DecrementConns() {
	atomic.AddInt64(&b.ActiveConns, -1)
}

// LoadBalancer interface
type LoadBalancer interface {
	Next(ctx context.Context) (*Backend, error)
	Add(backend *Backend)
	Remove(url string)
	Backends() []*Backend
}

// HealthyBackends ดึง backends ที่ healthy เท่านั้น
func HealthyBackends(backends []*Backend) []*Backend {
	var healthy []*Backend
	for _, b := range backends {
		if b.IsHealthy() {
			healthy = append(healthy, b)
		}
	}
	return healthy
}

// Round Robin Algorithm
type RoundRobinLB struct {
	mu       sync.RWMutex
	backends []*Backend
	current  uint64
}

func NewRoundRobin() *RoundRobinLB {
	return &RoundRobinLB{}
}

func (lb *RoundRobinLB) Add(b *Backend) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	lb.backends = append(lb.backends, b)
}

func (lb *RoundRobinLB) Remove(url string) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	for i, b := range lb.backends {
		if b.URL == url {
			lb.backends = append(lb.backends[:i], lb.backends[i+1:]...)
			return
		}
	}
}

func (lb *RoundRobinLB) Backends() []*Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()
	return lb.backends
}

func (lb *RoundRobinLB) Next(ctx context.Context) (*Backend, error) {
	lb.mu.RLock()
	defer lb.mu.RUnlock()

	healthy := HealthyBackends(lb.backends)
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends available")
	}

	idx := atomic.AddUint64(&lb.current, 1) % uint64(len(healthy))
	return healthy[idx], nil
}

// Least Connections Algorithm
type LeastConnLB struct {
	mu       sync.RWMutex
	backends []*Backend
}

func NewLeastConn() *LeastConnLB {
	return &LeastConnLB{}
}

func (lb *LeastConnLB) Add(b *Backend) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	lb.backends = append(lb.backends, b)
}

func (lb *LeastConnLB) Remove(url string) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	for i, b := range lb.backends {
		if b.URL == url {
			lb.backends = append(lb.backends[:i], lb.backends[i+1:]...)
			return
		}
	}
}

func (lb *LeastConnLB) Backends() []*Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()
	return lb.backends
}

func (lb *LeastConnLB) Next(ctx context.Context) (*Backend, error) {
	lb.mu.RLock()
	defer lb.mu.RUnlock()

	healthy := HealthyBackends(lb.backends)
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends available")
	}

	var best *Backend
	minConns := int64(-1)

	for _, b := range healthy {
		conns := atomic.LoadInt64(&b.ActiveConns)
		if minConns == -1 || conns < minConns {
			minConns = conns
			best = b
		}
	}
	return best, nil
}

// IP Hash Algorithm - same client always goes to same backend
type IPHashLB struct {
	mu       sync.RWMutex
	backends []*Backend
}

func NewIPHash() *IPHashLB {
	return &IPHashLB{}
}

func (lb *IPHashLB) Add(b *Backend) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	lb.backends = append(lb.backends, b)
}

func (lb *IPHashLB) Remove(url string) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	for i, b := range lb.backends {
		if b.URL == url {
			lb.backends = append(lb.backends[:i], lb.backends[i+1:]...)
			return
		}
	}
}

func (lb *IPHashLB) Backends() []*Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()
	return lb.backends
}

func (lb *IPHashLB) Next(ctx context.Context) (*Backend, error) {
	// ต้องได้รับ client IP จาก context
	ip, ok := ctx.Value(clientIPKey{}).(string)
	if !ok || ip == "" {
		ip = "0.0.0.0"
	}
	return lb.nextForIP(ip)
}

func (lb *IPHashLB) nextForIP(ip string) (*Backend, error) {
	lb.mu.RLock()
	defer lb.mu.RUnlock()

	healthy := HealthyBackends(lb.backends)
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends available")
	}

	hash := md5.Sum([]byte(ip))
	idx := int(hash[0]) % len(healthy)
	return healthy[idx], nil
}

type clientIPKey struct{}

func WithClientIP(ctx context.Context, ip string) context.Context {
	return context.WithValue(ctx, clientIPKey{}, ip)
}

// Weighted Round Robin
type WeightedRRLB struct {
	mu       sync.RWMutex
	backends []*Backend
	current  int
	currentW int
	maxW     int
	gcdW     int
}

func NewWeightedRR() *WeightedRRLB {
	return &WeightedRRLB{current: -1}
}

func (lb *WeightedRRLB) Add(b *Backend) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	if b.Weight <= 0 {
		b.Weight = 1
	}
	lb.backends = append(lb.backends, b)
	lb.updateGCD()
}

func (lb *WeightedRRLB) Remove(url string) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	for i, b := range lb.backends {
		if b.URL == url {
			lb.backends = append(lb.backends[:i], lb.backends[i+1:]...)
			lb.updateGCD()
			return
		}
	}
}

func (lb *WeightedRRLB) Backends() []*Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()
	return lb.backends
}

func (lb *WeightedRRLB) updateGCD() {
	if len(lb.backends) == 0 {
		lb.maxW = 0
		lb.gcdW = 0
		return
	}
	lb.maxW = lb.backends[0].Weight
	lb.gcdW = lb.backends[0].Weight
	for _, b := range lb.backends[1:] {
		if b.Weight > lb.maxW {
			lb.maxW = b.Weight
		}
		lb.gcdW = gcd(lb.gcdW, b.Weight)
	}
}

func (lb *WeightedRRLB) Next(ctx context.Context) (*Backend, error) {
	lb.mu.Lock()
	defer lb.mu.Unlock()

	healthy := HealthyBackends(lb.backends)
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends available")
	}

	// Nginx Smooth Weighted Round Robin algorithm
	for {
		lb.current = (lb.current + 1) % len(healthy)
		if lb.current == 0 {
			lb.currentW -= lb.gcdW
			if lb.currentW <= 0 {
				lb.currentW = lb.maxW
			}
		}
		if healthy[lb.current].Weight >= lb.currentW {
			return healthy[lb.current], nil
		}
	}
}

func gcd(a, b int) int {
	for b != 0 {
		a, b = b, a%b
	}
	return a
}
```

---

## 2. Go Reverse Proxy

```go
// proxy/reverse_proxy.go
package proxy

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"time"
)

type LoadBalancerProxy struct {
	lb      LoadBalancer
	timeout time.Duration
}

type LoadBalancer interface {
	Next(ctx context.Context) (*Backend, error)
}

type Backend struct {
	URL     string
	Healthy bool
}

func NewLoadBalancerProxy(lb LoadBalancer, timeout time.Duration) *LoadBalancerProxy {
	return &LoadBalancerProxy{
		lb:      lb,
		timeout: timeout,
	}
}

func (p *LoadBalancerProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	if p.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, p.timeout)
		defer cancel()
	}

	backend, err := p.lb.Next(ctx)
	if err != nil {
		log.Printf("Load balancer error: %v", err)
		http.Error(w, "Service Unavailable", http.StatusServiceUnavailable)
		return
	}

	target, err := url.Parse(backend.URL)
	if err != nil {
		log.Printf("Invalid backend URL %s: %v", backend.URL, err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	proxy := &httputil.ReverseProxy{
		Director: func(req *http.Request) {
			req.URL.Scheme = target.Scheme
			req.URL.Host = target.Host
			req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path)
			req.Host = target.Host
			req.Header.Set("X-Real-IP", r.RemoteAddr)
			req.Header.Set("X-Forwarded-Host", r.Host)
			req.Header.Set("X-Backend", backend.URL)
			if target.RawQuery == "" || req.URL.RawQuery == "" {
				req.URL.RawQuery = target.RawQuery + req.URL.RawQuery
			} else {
				req.URL.RawQuery = target.RawQuery + "&" + req.URL.RawQuery
			}
		},
		ErrorHandler: func(w http.ResponseWriter, r *http.Request, err error) {
			log.Printf("Proxy error to %s: %v", backend.URL, err)
			http.Error(w, "Bad Gateway", http.StatusBadGateway)
		},
		ModifyResponse: func(resp *http.Response) error {
			resp.Header.Set("X-Served-By", backend.URL)
			return nil
		},
	}

	proxy.ServeHTTP(w, r.WithContext(ctx))
}

func singleJoiningSlash(a, b string) string {
	aslash := len(a) > 0 && a[len(a)-1] == '/'
	bslash := len(b) > 0 && b[0] == '/'
	switch {
	case aslash && bslash:
		return a + b[1:]
	case !aslash && !bslash:
		return a + "/" + b
	}
	return a + b
}
```

---

## 3. Health Checker

```go
// healthcheck/checker.go
package healthcheck

import (
	"context"
	"log"
	"net/http"
	"sync"
	"time"
)

type Backend struct {
	URL     string
	Healthy bool
	mu      sync.RWMutex
}

func (b *Backend) SetHealthy(h bool) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.Healthy != h {
		if h {
			log.Printf("Backend %s is UP", b.URL)
		} else {
			log.Printf("Backend %s is DOWN", b.URL)
		}
		b.Healthy = h
	}
}

type HealthChecker struct {
	backends []*Backend
	interval time.Duration
	timeout  time.Duration
	client   *http.Client
}

func NewHealthChecker(backends []*Backend, interval, timeout time.Duration) *HealthChecker {
	return &HealthChecker{
		backends: backends,
		interval: interval,
		timeout:  timeout,
		client: &http.Client{
			Timeout: timeout,
		},
	}
}

func (hc *HealthChecker) Start(ctx context.Context) {
	ticker := time.NewTicker(hc.interval)
	defer ticker.Stop()

	// Initial check
	hc.checkAll(ctx)

	for {
		select {
		case <-ctx.Done():
			log.Println("Health checker stopped")
			return
		case <-ticker.C:
			hc.checkAll(ctx)
		}
	}
}

func (hc *HealthChecker) checkAll(ctx context.Context) {
	var wg sync.WaitGroup
	for _, backend := range hc.backends {
		wg.Add(1)
		go func(b *Backend) {
			defer wg.Done()
			healthy := hc.check(ctx, b.URL)
			b.SetHealthy(healthy)
		}(backend)
	}
	wg.Wait()
}

func (hc *HealthChecker) check(ctx context.Context, url string) bool {
	checkCtx, cancel := context.WithTimeout(ctx, hc.timeout)
	defer cancel()

	req, err := http.NewRequestWithContext(checkCtx, "GET", url+"/health", nil)
	if err != nil {
		return false
	}

	resp, err := hc.client.Do(req)
	if err != nil {
		return false
	}
	defer resp.Body.Close()

	return resp.StatusCode == http.StatusOK
}
```

---

## 4. Sticky Sessions

```go
// stickysession/session.go
package stickysession

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"net/http"
	"sync"
)

const sessionCookieName = "lb_session"

type StickySessionLB struct {
	mu       sync.RWMutex
	backends []*Backend
	sessions map[string]string // sessionID -> backendURL
	secret   []byte
}

type Backend struct {
	URL     string
	Healthy bool
}

func NewStickySessionLB(secret []byte) *StickySessionLB {
	return &StickySessionLB{
		sessions: make(map[string]string),
		secret:   secret,
	}
}

func (lb *StickySessionLB) Add(b *Backend) {
	lb.mu.Lock()
	defer lb.mu.Unlock()
	lb.backends = append(lb.backends, b)
}

func (lb *StickySessionLB) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	backend := lb.getBackend(w, r)
	if backend == nil {
		http.Error(w, "Service Unavailable", http.StatusServiceUnavailable)
		return
	}
	// Proxy to backend (simplified)
	fmt.Fprintf(w, "Proxying to: %s\n", backend.URL)
}

func (lb *StickySessionLB) getBackend(w http.ResponseWriter, r *http.Request) *Backend {
	// Check existing session cookie
	cookie, err := r.Cookie(sessionCookieName)
	if err == nil && lb.verifySession(cookie.Value) {
		sessionID := lb.extractSessionID(cookie.Value)
		lb.mu.RLock()
		backendURL, ok := lb.sessions[sessionID]
		lb.mu.RUnlock()

		if ok {
			// Find the backend
			lb.mu.RLock()
			defer lb.mu.RUnlock()
			for _, b := range lb.backends {
				if b.URL == backendURL && b.Healthy {
					return b
				}
			}
		}
	}

	// No valid session, pick a backend
	backend := lb.pickBackend()
	if backend == nil {
		return nil
	}

	// Create session
	sessionID := generateSessionID(r)
	cookieValue := lb.createSignedSession(sessionID)

	lb.mu.Lock()
	lb.sessions[sessionID] = backend.URL
	lb.mu.Unlock()

	http.SetCookie(w, &http.Cookie{
		Name:     sessionCookieName,
		Value:    cookieValue,
		Path:     "/",
		HttpOnly: true,
		Secure:   r.TLS != nil,
		SameSite: http.SameSiteStrictMode,
	})

	return backend
}

func (lb *StickySessionLB) pickBackend() *Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()
	for _, b := range lb.backends {
		if b.Healthy {
			return b
		}
	}
	return nil
}

func (lb *StickySessionLB) createSignedSession(sessionID string) string {
	mac := hmac.New(sha256.New, lb.secret)
	mac.Write([]byte(sessionID))
	sig := hex.EncodeToString(mac.Sum(nil))
	return sessionID + "." + sig
}

func (lb *StickySessionLB) verifySession(cookieValue string) bool {
	for i := len(cookieValue) - 1; i >= 0; i-- {
		if cookieValue[i] == '.' {
			sessionID := cookieValue[:i]
			expected := lb.createSignedSession(sessionID)
			return hmac.Equal([]byte(cookieValue), []byte(expected))
		}
	}
	return false
}

func (lb *StickySessionLB) extractSessionID(cookieValue string) string {
	for i := len(cookieValue) - 1; i >= 0; i-- {
		if cookieValue[i] == '.' {
			return cookieValue[:i]
		}
	}
	return cookieValue
}

func generateSessionID(r *http.Request) string {
	mac := hmac.New(sha256.New, []byte("random-salt"))
	mac.Write([]byte(r.RemoteAddr))
	mac.Write([]byte(fmt.Sprint(r.Header)))
	return hex.EncodeToString(mac.Sum(nil))[:16]
}
```

---

## 5. Layer 4 vs Layer 7 Load Balancing

```go
// layer4/tcp_lb.go - Layer 4 (TCP/UDP)
package layer4

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"sync/atomic"
)

// Layer 4 LB ทำงานที่ transport layer
// ไม่อ่าน HTTP headers เพียงแค่ proxy TCP connections

type Backend struct {
	Address  string
	Healthy  bool
	Conns    int64
	mu       sync.RWMutex
}

func (b *Backend) IsHealthy() bool {
	b.mu.RLock()
	defer b.mu.RUnlock()
	return b.Healthy
}

type TCPLoadBalancer struct {
	listener net.Listener
	backends []*Backend
	counter  uint64
	mu       sync.RWMutex
}

func New(listenAddr string, backends []string) (*TCPLoadBalancer, error) {
	ln, err := net.Listen("tcp", listenAddr)
	if err != nil {
		return nil, fmt.Errorf("listening on %s: %w", listenAddr, err)
	}

	lb := &TCPLoadBalancer{listener: ln}
	for _, addr := range backends {
		lb.backends = append(lb.backends, &Backend{
			Address: addr,
			Healthy: true,
		})
	}
	return lb, nil
}

func (lb *TCPLoadBalancer) Start(ctx context.Context) error {
	log.Printf("Layer 4 LB listening on %s", lb.listener.Addr())

	go func() {
		<-ctx.Done()
		lb.listener.Close()
	}()

	for {
		conn, err := lb.listener.Accept()
		if err != nil {
			select {
			case <-ctx.Done():
				return nil
			default:
				log.Printf("Accept error: %v", err)
				continue
			}
		}
		go lb.handleConnection(ctx, conn)
	}
}

func (lb *TCPLoadBalancer) handleConnection(ctx context.Context, clientConn net.Conn) {
	defer clientConn.Close()

	backend := lb.nextBackend()
	if backend == nil {
		log.Println("No healthy backends available")
		return
	}

	// Connect to backend
	backendConn, err := net.Dial("tcp", backend.Address)
	if err != nil {
		log.Printf("Error connecting to backend %s: %v", backend.Address, err)
		backend.mu.Lock()
		backend.Healthy = false
		backend.mu.Unlock()
		return
	}
	defer backendConn.Close()

	atomic.AddInt64(&backend.Conns, 1)
	defer atomic.AddInt64(&backend.Conns, -1)

	// Bidirectional proxy
	done := make(chan struct{}, 2)

	go func() {
		io.Copy(backendConn, clientConn)
		done <- struct{}{}
	}()

	go func() {
		io.Copy(clientConn, backendConn)
		done <- struct{}{}
	}()

	select {
	case <-ctx.Done():
	case <-done:
	}
}

func (lb *TCPLoadBalancer) nextBackend() *Backend {
	lb.mu.RLock()
	defer lb.mu.RUnlock()

	var healthy []*Backend
	for _, b := range lb.backends {
		if b.IsHealthy() {
			healthy = append(healthy, b)
		}
	}

	if len(healthy) == 0 {
		return nil
	}

	idx := atomic.AddUint64(&lb.counter, 1) % uint64(len(healthy))
	return healthy[idx]
}
```

---

## 6. Complete Load Balancer with All Features

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"flag"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"
)

func main() {
	var (
		addr     = flag.String("addr", ":8080", "Listen address")
		backends = flag.String("backends", "http://localhost:8081,http://localhost:8082,http://localhost:8083", "Backend URLs")
		algo     = flag.String("algo", "roundrobin", "Algorithm: roundrobin, leastconn, iphash")
		interval = flag.Duration("health-interval", 10*time.Second, "Health check interval")
	)
	flag.Parse()

	backendURLs := strings.Split(*backends, ",")

	// Create backends
	var bkds []*BackendServer
	for _, url := range backendURLs {
		bkds = append(bkds, &BackendServer{
			URL:     strings.TrimSpace(url),
			Healthy: true,
		})
	}

	// Create load balancer based on algorithm
	var lb BalancingStrategy
	switch *algo {
	case "leastconn":
		lb = NewLeastConnStrategy(bkds)
	case "iphash":
		lb = NewIPHashStrategy(bkds)
	default:
		lb = NewRoundRobinStrategy(bkds)
	}

	log.Printf("Using %s load balancing algorithm", *algo)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Start health checker
	checker := NewSimpleHealthChecker(bkds, *interval)
	go checker.Start(ctx)

	// Stats handler
	mux := http.NewServeMux()
	mux.HandleFunc("/lb/stats", statsHandler(bkds))
	mux.HandleFunc("/", proxyHandler(lb))

	srv := &http.Server{
		Addr:    *addr,
		Handler: mux,
	}

	go func() {
		log.Printf("Load Balancer listening on %s", *addr)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	shutCtx, shutCancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer shutCancel()
	srv.Shutdown(shutCtx)
	cancel()
	log.Println("Load Balancer stopped")
}

type BackendServer struct {
	URL        string
	Healthy    bool
	ActiveReqs int64
}

type BalancingStrategy interface {
	Next() (*BackendServer, error)
}

type RoundRobinStrategy struct {
	backends []*BackendServer
	counter  uint64
}

func NewRoundRobinStrategy(backends []*BackendServer) *RoundRobinStrategy {
	return &RoundRobinStrategy{backends: backends}
}

func (s *RoundRobinStrategy) Next() (*BackendServer, error) {
	var healthy []*BackendServer
	for _, b := range s.backends {
		if b.Healthy {
			healthy = append(healthy, b)
		}
	}
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends")
	}
	idx := atomic.AddUint64(&s.counter, 1) % uint64(len(healthy))
	return healthy[idx], nil
}

type LeastConnStrategy struct {
	backends []*BackendServer
}

func NewLeastConnStrategy(backends []*BackendServer) *LeastConnStrategy {
	return &LeastConnStrategy{backends: backends}
}

func (s *LeastConnStrategy) Next() (*BackendServer, error) {
	var best *BackendServer
	minReqs := int64(-1)
	for _, b := range s.backends {
		if !b.Healthy {
			continue
		}
		if minReqs == -1 || b.ActiveReqs < minReqs {
			minReqs = b.ActiveReqs
			best = b
		}
	}
	if best == nil {
		return nil, fmt.Errorf("no healthy backends")
	}
	return best, nil
}

type IPHashStrategy struct {
	backends []*BackendServer
}

func NewIPHashStrategy(backends []*BackendServer) *IPHashStrategy {
	return &IPHashStrategy{backends: backends}
}

func (s *IPHashStrategy) Next() (*BackendServer, error) {
	var healthy []*BackendServer
	for _, b := range s.backends {
		if b.Healthy {
			healthy = append(healthy, b)
		}
	}
	if len(healthy) == 0 {
		return nil, fmt.Errorf("no healthy backends")
	}
	return healthy[0], nil
}

type SimpleHealthChecker struct {
	backends []*BackendServer
	interval time.Duration
	client   *http.Client
}

func NewSimpleHealthChecker(backends []*BackendServer, interval time.Duration) *SimpleHealthChecker {
	return &SimpleHealthChecker{
		backends: backends,
		interval: interval,
		client:   &http.Client{Timeout: 3 * time.Second},
	}
}

func (c *SimpleHealthChecker) Start(ctx context.Context) {
	ticker := time.NewTicker(c.interval)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			for _, b := range c.backends {
				go c.checkBackend(ctx, b)
			}
		}
	}
}

func (c *SimpleHealthChecker) checkBackend(ctx context.Context, b *BackendServer) {
	req, err := http.NewRequestWithContext(ctx, "GET", b.URL+"/health", nil)
	if err != nil {
		b.Healthy = false
		return
	}
	resp, err := c.client.Do(req)
	if err != nil || resp.StatusCode >= 500 {
		if b.Healthy {
			log.Printf("Backend %s became unhealthy", b.URL)
		}
		b.Healthy = false
		return
	}
	resp.Body.Close()
	if !b.Healthy {
		log.Printf("Backend %s became healthy", b.URL)
	}
	b.Healthy = true
}

func statsHandler(backends []*BackendServer) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var stats []map[string]interface{}
		for _, b := range backends {
			stats = append(stats, map[string]interface{}{
				"url":         b.URL,
				"healthy":     b.Healthy,
				"active_reqs": b.ActiveReqs,
			})
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"backends": stats,
			"time":     time.Now().Format(time.RFC3339),
		})
	}
}

func proxyHandler(lb BalancingStrategy) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		backend, err := lb.Next()
		if err != nil {
			http.Error(w, "Service Unavailable", http.StatusServiceUnavailable)
			return
		}
		// In real implementation, proxy the request
		w.Header().Set("X-Backend", backend.URL)
		fmt.Fprintf(w, "Routed to: %s\n", backend.URL)
	}
}

// suppress unused
import (
	"fmt"
	"sync/atomic"
)
```

---

## สรุป

| Algorithm | เหมาะกับ | ข้อดี | ข้อเสีย |
|-----------|---------|-------|--------|
| Round Robin | Stateless services | ง่าย, กระจาย traffic สม่ำเสมอ | ไม่คำนึง server load |
| Least Connections | Long-lived connections | กระจาย load ตาม connections | Overhead ในการ track |
| IP Hash | Stateful sessions | Consistent routing | ไม่กระจาย traffic สม่ำเสมอ |
| Weighted RR | Heterogeneous servers | รองรับ server ที่มี capacity ต่างกัน | ต้อง configure weights |
| Sticky Session | Session-based apps | Session consistency | Single point of failure |

---

**ต่อไป**: Part 55 - Circuit Breaker
