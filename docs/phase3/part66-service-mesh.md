# Part 66: Service Mesh

## เป้าหมายการเรียนรู้
- เข้าใจแนวคิด Service Mesh และประโยชน์ที่ได้รับ
- เรียนรู้ Istio และ Envoy Proxy
- จัดการ Traffic Management, mTLS, Circuit Breaker
- ใช้งาน Observability ด้วย Istio
- รู้จัก Linkerd เป็นทางเลือก

---

## 1. Service Mesh คืออะไร?

Service Mesh คือ infrastructure layer สำหรับจัดการการสื่อสารระหว่าง microservices โดยจะแทรก sidecar proxy ไว้ข้างๆ ทุก service เพื่อจัดการ:

- **Traffic Management**: load balancing, routing, retries
- **Security**: mTLS, authorization policies
- **Observability**: metrics, tracing, logging
- **Reliability**: circuit breakers, timeouts, fault injection

```
┌─────────────────────────────────────────┐
│              Service Mesh               │
│                                         │
│  ┌─────────┐      ┌─────────┐          │
│  │Service A│◄────►│Service B│          │
│  │  +Proxy │      │  +Proxy │          │
│  └─────────┘      └─────────┘          │
│       │                │               │
│       └────────────────┘               │
│            Control Plane               │
└─────────────────────────────────────────┘
```

### ปัญหาที่ Service Mesh แก้ไข

```go
// ปัญหาเดิม: ต้อง implement ทุกอย่างใน code
package main

import (
    "net/http"
    "time"
)

// แบบเก่า - ต้องใส่ logic ทุกอย่างใน application code
type ServiceClient struct {
    httpClient *http.Client
    retries    int
    timeout    time.Duration
}

func (c *ServiceClient) CallService(url string) (*http.Response, error) {
    // Retry logic
    // Circuit breaker logic  
    // Timeout logic
    // Metrics collection
    // Tracing
    // มีโค้ดซ้ำซ้อนในทุก service
    return c.httpClient.Get(url)
}

// แบบใหม่ด้วย Service Mesh - application code สะอาดขึ้น
func main() {
    // ไม่ต้องจัดการ retry, circuit breaker, metrics ใน code
    // Envoy proxy จัดการให้ทั้งหมด
    resp, err := http.Get("http://service-b/api/data")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
}
```

---

## 2. Istio Overview

Istio เป็น Service Mesh ที่ได้รับความนิยมสูงสุด ประกอบด้วย:

### Components หลัก

```yaml
# istiod - Control Plane
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istiocontrolplane
spec:
  profile: default
  components:
    pilot:
      enabled: true    # Service discovery, config distribution
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
    egressGateways:
    - name: istio-egressgateway
      enabled: true
```

### การติดตั้ง Istio

```bash
# ดาวน์โหลด istioctl
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/path/to/istio/bin"

# ติดตั้ง Istio ใน Kubernetes cluster
istioctl install --set profile=default -y

# Enable automatic sidecar injection
kubectl label namespace default istio-injection=enabled

# ตรวจสอบ installation
istioctl verify-install
kubectl get pods -n istio-system
```

### Go Application ใน Istio

```go
// main.go - Simple Go service ที่ทำงานใน Istio
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
)

type Response struct {
    Service   string    `json:"service"`
    Message   string    `json:"message"`
    Timestamp time.Time `json:"timestamp"`
    Pod       string    `json:"pod"`
}

func main() {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // Istio headers สำหรับ distributed tracing
        traceID := r.Header.Get("x-b3-traceid")
        spanID := r.Header.Get("x-b3-spanid")
        
        log.Printf("Request received - TraceID: %s, SpanID: %s", traceID, spanID)
        
        resp := Response{
            Service:   os.Getenv("SERVICE_NAME"),
            Message:   "Hello from Istio mesh",
            Timestamp: time.Now(),
            Pod:       os.Getenv("POD_NAME"),
        }
        
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(resp)
    })
    
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "OK")
    })
    
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    
    log.Printf("Starting server on port %s", port)
    log.Fatal(http.ListenAndServe(":"+port, mux))
}
```

```yaml
# kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  labels:
    app: my-service
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-service
      version: v1
  template:
    metadata:
      labels:
        app: my-service
        version: v1
      annotations:
        # Istio จะ inject sidecar อัตโนมัติ
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: my-service
        image: my-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "my-service"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

---

## 3. Envoy Proxy

Envoy เป็น high-performance proxy ที่ใช้เป็น sidecar ใน Istio

### Envoy Architecture

```
┌──────────────────────────────────────────────────┐
│                   Envoy Proxy                     │
│                                                   │
│  ┌─────────┐  ┌──────────┐  ┌─────────────────┐ │
│  │Listeners│→ │  Filters │→ │    Clusters     │ │
│  └─────────┘  └──────────┘  └─────────────────┘ │
│                                                   │
│  ┌──────────────────────────────────────────────┐│
│  │              xDS APIs                         ││
│  │  LDS  CDS  RDS  EDS  SDS  ADS               ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
```

### Envoy Configuration (ผ่าน Istio)

```yaml
# EnvoyFilter - customize Envoy configuration
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: custom-header-filter
  namespace: default
spec:
  workloadSelector:
    labels:
      app: my-service
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.LuaPerRoute"
          inline_code: |
            function envoy_on_request(request_handle)
              request_handle:headers():add("x-custom-header", "istio-mesh")
            end
```

### การ Propagate Tracing Headers ใน Go

```go
// tracing/propagation.go
package tracing

import "net/http"

// Headers ที่ต้อง propagate สำหรับ distributed tracing
var tracingHeaders = []string{
    "x-request-id",
    "x-b3-traceid",
    "x-b3-spanid",
    "x-b3-parentspanid",
    "x-b3-sampled",
    "x-b3-flags",
    "x-ot-span-context",
    "b3",
}

// PropagateHeaders คัดลอก tracing headers จาก incoming request ไปยัง outgoing request
func PropagateHeaders(incoming *http.Request, outgoing *http.Request) {
    for _, header := range tracingHeaders {
        val := incoming.Header.Get(header)
        if val != "" {
            outgoing.Header.Set(header, val)
        }
    }
}

// HTTPClient with header propagation
type TracingClient struct {
    client   *http.Client
    incoming *http.Request
}

func NewTracingClient(incoming *http.Request) *TracingClient {
    return &TracingClient{
        client:   &http.Client{},
        incoming: incoming,
    }
}

func (tc *TracingClient) Get(url string) (*http.Response, error) {
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    PropagateHeaders(tc.incoming, req)
    return tc.client.Do(req)
}
```

---

## 4. Traffic Management

### VirtualService - กำหนด Traffic Routing

```yaml
# traffic-management/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
  namespace: default
spec:
  hosts:
  - my-service
  http:
  # Route based on headers (A/B testing)
  - match:
    - headers:
        user-type:
          exact: premium
    route:
    - destination:
        host: my-service
        subset: v2
      weight: 100
  
  # Canary deployment - 10% ไปที่ v2
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 90
    - destination:
        host: my-service
        subset: v2
      weight: 10
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,retriable-4xx
    
    # Timeout
    timeout: 10s
    
    # Fault injection for testing
    fault:
      delay:
        percentage:
          value: 0.1  # 0.1% ของ requests จะถูก delay
        fixedDelay: 5s
```

```yaml
# DestinationRule - กำหนด subsets และ traffic policies
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service
  namespace: default
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # หรือ ROUND_ROBIN, RANDOM
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        http:
          http2MaxRequests: 500
```

### Gateway - Traffic Entry Point

```yaml
# ingress/gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: myapp-tls-secret
    hosts:
    - "myapp.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
  namespace: default
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - my-gateway
  http:
  - match:
    - uri:
        prefix: /api/v1
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: frontend-service
        port:
          number: 3000
```

### ServiceEntry - External Services

```yaml
# external/service-entry.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
  namespace: default
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
# อนุญาตให้ติดต่อ external service นี้ได้
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-api-vs
spec:
  hosts:
  - api.external.com
  http:
  - timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
    route:
    - destination:
        host: api.external.com
        port:
          number: 443
```

---

## 5. mTLS Between Services

### Mutual TLS คืออะไร?

mTLS (Mutual TLS) คือการที่ทั้ง client และ server ต่างยืนยันตัวตนซึ่งกันและกัน ใน Istio ทำได้อัตโนมัติ

```yaml
# security/peer-authentication.yaml - Enable mTLS ทั้ง namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # บังคับใช้ mTLS ทุก connection
```

```yaml
# DestinationRule สำหรับ mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: default-mtls
  namespace: default
spec:
  host: "*.default.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # ใช้ certificates ที่ Istio จัดการ
```

### Go Application ที่ใช้ mTLS แบบ Manual (ไม่ผ่าน Istio)

```go
// mtls/server.go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "log"
    "net/http"
    "os"
)

func createMTLSServer(certFile, keyFile, caCertFile string) (*http.Server, error) {
    // โหลด CA certificate
    caCert, err := os.ReadFile(caCertFile)
    if err != nil {
        return nil, fmt.Errorf("failed to read CA cert: %w", err)
    }
    
    caCertPool := x509.NewCertPool()
    if !caCertPool.AppendCertsFromPEM(caCert) {
        return nil, fmt.Errorf("failed to append CA cert")
    }
    
    // โหลด server certificate
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, fmt.Errorf("failed to load server cert: %w", err)
    }
    
    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientCAs:    caCertPool,
        ClientAuth:   tls.RequireAndVerifyClientCert, // บังคับ mTLS
        MinVersion:   tls.VersionTLS13,
    }
    
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // ดึงข้อมูล client certificate
        if len(r.TLS.PeerCertificates) > 0 {
            clientCert := r.TLS.PeerCertificates[0]
            log.Printf("Client: %s", clientCert.Subject.CommonName)
        }
        fmt.Fprint(w, "mTLS connection established!")
    })
    
    server := &http.Server{
        Addr:      ":8443",
        Handler:   mux,
        TLSConfig: tlsConfig,
    }
    
    return server, nil
}

// mtls/client.go
func createMTLSClient(certFile, keyFile, caCertFile string) (*http.Client, error) {
    // โหลด client certificate
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, fmt.Errorf("failed to load client cert: %w", err)
    }
    
    // โหลด CA certificate
    caCert, err := os.ReadFile(caCertFile)
    if err != nil {
        return nil, fmt.Errorf("failed to read CA cert: %w", err)
    }
    
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)
    
    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
    }
    
    transport := &http.Transport{TLSClientConfig: tlsConfig}
    client := &http.Client{Transport: transport}
    
    return client, nil
}
```

---

## 6. Circuit Breaker ใน Istio

### Outlier Detection

```yaml
# circuit-breaker/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: default
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      # จำนวน error ติดต่อกันก่อน eject
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 5
      # ตรวจสอบทุก 30 วินาที
      interval: 30s
      # เวลาที่ถูก eject ขั้นต่ำ
      baseEjectionTime: 30s
      # % สูงสุดของ hosts ที่ถูก eject ได้
      maxEjectionPercent: 50
      # eject เมื่อ error rate เกิน
      splitExternalLocalOriginErrors: true
    connectionPool:
      tcp:
        maxConnections: 10  # Circuit breaker connection limit
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
```

### Circuit Breaker ใน Go Code ร่วมกับ Istio

```go
// circuit_breaker.go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"
)

type State int

const (
    StateClosed   State = iota // ปกติ
    StateOpen                  // ปิดกั้น requests
    StateHalfOpen              // ทดสอบว่า service กลับมาหรือยัง
)

type CircuitBreaker struct {
    mu           sync.RWMutex
    state        State
    failures     int
    successes    int
    lastFailure  time.Time
    threshold    int
    timeout      time.Duration
    halfOpenMax  int
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:       StateClosed,
        threshold:   threshold,
        timeout:     timeout,
        halfOpenMax: 3,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    state := cb.state
    
    switch state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            log.Println("Circuit breaker: OPEN -> HALF_OPEN")
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    case StateHalfOpen:
        if cb.successes >= cb.halfOpenMax {
            cb.mu.Unlock()
            return errors.New("circuit breaker is half-open, max requests reached")
        }
    }
    cb.mu.Unlock()
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        
        if cb.state == StateHalfOpen || cb.failures >= cb.threshold {
            cb.state = StateOpen
            log.Printf("Circuit breaker: OPEN (failures: %d)", cb.failures)
        }
        return err
    }
    
    if cb.state == StateHalfOpen {
        cb.successes++
        if cb.successes >= cb.halfOpenMax {
            cb.state = StateClosed
            cb.failures = 0
            log.Println("Circuit breaker: HALF_OPEN -> CLOSED")
        }
    } else {
        cb.failures = 0
    }
    
    return nil
}

// ใช้งานกับ HTTP client
type ResilientClient struct {
    httpClient     *http.Client
    circuitBreaker *CircuitBreaker
}

func NewResilientClient() *ResilientClient {
    return &ResilientClient{
        httpClient:     &http.Client{Timeout: 5 * time.Second},
        circuitBreaker: NewCircuitBreaker(5, 30*time.Second),
    }
}

func (rc *ResilientClient) Get(ctx context.Context, url string) (*http.Response, error) {
    var resp *http.Response
    
    err := rc.circuitBreaker.Execute(func() error {
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return err
        }
        
        resp, err = rc.httpClient.Do(req)
        if err != nil {
            return err
        }
        
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        
        return nil
    })
    
    return resp, err
}
```

---

## 7. Observability กับ Istio

### Metrics ด้วย Prometheus

```yaml
# observability/telemetry.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default-metrics
  namespace: default
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: ALL_METRICS
      tagOverrides:
        response_code:
          value: "response.code"
        source_app:
          value: "source.labels['app']"
```

```yaml
# observability/prometheus-rule.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: istiod
  endpoints:
  - port: http-monitoring
    interval: 15s
```

### Distributed Tracing กับ Jaeger

```go
// tracing/jaeger.go
package tracing

import (
    "context"
    "fmt"
    "io"
    "net/http"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
    "go.opentelemetry.io/otel/trace"
)

func InitTracer(serviceName, jaegerEndpoint string) (func(), error) {
    exporter, err := jaeger.New(
        jaeger.WithCollectorEndpoint(
            jaeger.WithEndpoint(jaegerEndpoint),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create Jaeger exporter: %w", err)
    }
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
        )),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )
    
    otel.SetTracerProvider(tp)
    
    return func() {
        tp.Shutdown(context.Background())
    }, nil
}

// Middleware สำหรับ HTTP tracing
func TracingMiddleware(tracer trace.Tracer) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, span := tracer.Start(r.Context(), r.URL.Path)
            defer span.End()
            
            // Propagate Istio headers
            span.SetAttributes(
                semconv.HTTPMethodKey.String(r.Method),
                semconv.HTTPURLKey.String(r.URL.String()),
            )
            
            // Forward tracing context
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Downstream call with tracing
func CallDownstream(ctx context.Context, url string) ([]byte, error) {
    tracer := otel.Tracer("my-service")
    
    _, span := tracer.Start(ctx, "downstream-call")
    defer span.End()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    defer resp.Body.Close()
    
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    return body, nil
}
```

### Kiali Dashboard Config

```yaml
# observability/kiali.yaml
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  auth:
    strategy: anonymous
  deployment:
    accessible_namespaces:
    - '**'
  external_services:
    prometheus:
      url: "http://prometheus:9090"
    jaeger:
      in_cluster_url: "http://jaeger-query.tracing:16686"
    grafana:
      in_cluster_url: "http://grafana:3000"
```

---

## 8. Authorization Policy

```yaml
# security/authorization-policy.yaml
# อนุญาตเฉพาะ services ที่ระบุเข้าถึงได้
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  # อนุญาตให้ order-service เข้าถึง /process endpoint
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/payment/process"]
  # อนุญาต health check จากทุกที่
  - to:
    - operation:
        methods: ["GET"]
        paths: ["/health", "/metrics"]
```

```yaml
# Deny all traffic by default แล้วค่อย allow เฉพาะที่ต้องการ
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}  # ไม่มี rules = deny all
```

---

## 9. Linkerd เป็นทางเลือก

Linkerd เป็น lightweight service mesh ที่ใช้งานง่ายกว่า Istio

### การติดตั้ง Linkerd

```bash
# ติดตั้ง linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH="$PATH:$HOME/.linkerd2/bin"

# ตรวจสอบ cluster compatibility
linkerd check --pre

# ติดตั้ง Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# ตรวจสอบ installation
linkerd check

# Enable sidecar injection สำหรับ deployment
kubectl get deploy my-service -o yaml | linkerd inject - | kubectl apply -f -
```

### Linkerd Service Profile

```yaml
# linkerd/service-profile.yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: my-service.default.svc.cluster.local
  namespace: default
spec:
  routes:
  - name: GET /api/users
    condition:
      method: GET
      pathRegex: /api/users
    responseClasses:
    - condition:
        status:
          min: 500
      isFailure: true
    retryBudget:
      retryRatio: 0.2
      minRetriesPerSecond: 10
      ttl: 10s
  - name: POST /api/users
    condition:
      method: POST
      pathRegex: /api/users
    isRetryable: false
    timeout: 5s
```

### Go application กับ Linkerd

```go
// linkerd/server.go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/api/users", handleUsers)
    
    // Linkerd expects a /ready endpoint
    mux.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    
    // Linkerd expects a /live endpoint  
    mux.HandleFunc("/live", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    
    // Graceful shutdown (สำคัญสำหรับ service mesh)
    done := make(chan os.Signal, 1)
    signal.Notify(done, syscall.SIGINT, syscall.SIGTERM)
    
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    log.Println("Server started on :8080")
    <-done
    
    log.Println("Shutting down server...")
    server.Close()
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    users := []map[string]interface{}{
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

---

## 10. Traffic Mirroring (Shadowing)

```yaml
# traffic/mirror.yaml
# ส่ง copy ของ traffic ไปยัง shadow service โดยไม่กระทบ production
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
  namespace: default
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 100
    # Mirror 10% ของ traffic ไปยัง v2 สำหรับทดสอบ
    mirror:
      host: my-service
      subset: v2
    mirrorPercentage:
      value: 10.0
```

---

## Workshop: ออกแบบ Service Mesh สำหรับ E-commerce

```yaml
# workshop/ecommerce-mesh.yaml
---
# Frontend Gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ecommerce-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: ecommerce-tls
    hosts:
    - "shop.example.com"
---
# API Routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ecommerce-vs
spec:
  hosts:
  - "shop.example.com"
  gateways:
  - ecommerce-gateway
  http:
  - match:
    - uri:
        prefix: /api/products
    route:
    - destination:
        host: product-service
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 2s
    timeout: 10s
  - match:
    - uri:
        prefix: /api/orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
    retries:
      attempts: 2
      perTryTimeout: 3s
    timeout: 15s
  - match:
    - uri:
        prefix: /api/payment
    route:
    - destination:
        host: payment-service
        port:
          number: 8080
    retries:
      attempts: 0  # Payment ไม่ retry
    timeout: 30s
---
# Security: mTLS Strict
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: ecommerce-mtls
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT
---
# Circuit Breaker for Payment Service
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-cb
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 30
```

```go
// workshop/order-service/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

type Order struct {
    ID        string  `json:"id"`
    ProductID string  `json:"product_id"`
    Quantity  int     `json:"quantity"`
    Total     float64 `json:"total"`
    Status    string  `json:"status"`
}

type OrderService struct {
    productServiceURL string
    paymentServiceURL string
    httpClient        *http.Client
}

func NewOrderService(productURL, paymentURL string) *OrderService {
    return &OrderService{
        productServiceURL: productURL,
        paymentServiceURL: paymentURL,
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}

// สร้าง order พร้อม propagate tracing headers
func (s *OrderService) CreateOrder(ctx context.Context, incomingReq *http.Request, order *Order) error {
    // ตรวจสอบ product (propagate Istio headers)
    product, err := s.getProduct(ctx, incomingReq, order.ProductID)
    if err != nil {
        return fmt.Errorf("failed to get product: %w", err)
    }
    
    order.Total = float64(order.Quantity) * product["price"].(float64)
    
    // ประมวลผล payment (propagate Istio headers)
    if err := s.processPayment(ctx, incomingReq, order); err != nil {
        return fmt.Errorf("failed to process payment: %w", err)
    }
    
    order.Status = "completed"
    return nil
}

func (s *OrderService) getProduct(ctx context.Context, incomingReq *http.Request, productID string) (map[string]interface{}, error) {
    url := fmt.Sprintf("%s/products/%s", s.productServiceURL, productID)
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    // Propagate Istio tracing headers
    for _, h := range []string{"x-request-id", "x-b3-traceid", "x-b3-spanid", "x-b3-sampled"} {
        if val := incomingReq.Header.Get(h); val != "" {
            req.Header.Set(h, val)
        }
    }
    
    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var product map[string]interface{}
    body, _ := io.ReadAll(resp.Body)
    json.Unmarshal(body, &product)
    
    return product, nil
}

func (s *OrderService) processPayment(ctx context.Context, incomingReq *http.Request, order *Order) error {
    // Similar implementation...
    return nil
}

func main() {
    productURL := "http://product-service"
    paymentURL := "http://payment-service"
    
    svc := NewOrderService(productURL, paymentURL)
    
    http.HandleFunc("/orders", func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }
        
        var order Order
        if err := json.NewDecoder(r.Body).Decode(&order); err != nil {
            http.Error(w, "Invalid request body", http.StatusBadRequest)
            return
        }
        
        if err := svc.CreateOrder(r.Context(), r, &order); err != nil {
            log.Printf("Error creating order: %v", err)
            http.Error(w, "Failed to create order", http.StatusInternalServerError)
            return
        }
        
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(order)
    })
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## สรุป

Service Mesh ช่วยแก้ไขปัญหาที่พบบ่อยใน microservices architecture:

| Feature | Without Mesh | With Istio |
|---------|-------------|------------|
| mTLS | ต้องทำเอง | อัตโนมัติ |
| Circuit Breaker | ใส่ใน code | Config YAML |
| Load Balancing | DNS/k8s default | Intelligent LB |
| Observability | ต้องติดตั้งเอง | Built-in |
| Canary Deploy | ยาก | VirtualService |
| Retry/Timeout | ใส่ใน code | Config YAML |

### เมื่อไหร่ควรใช้ Service Mesh?
- มี microservices 10+ services
- ต้องการ security ระดับสูง (financial, healthcare)
- ต้องการ observability ครบถ้วน
- ทีมไม่ต้องการจัดการ cross-cutting concerns ใน code

### ข้อควรระวัง
- เพิ่ม latency เล็กน้อย (~1-5ms per hop)
- Complexity สูง
- Resource overhead (Envoy sidecar)
- Learning curve สูง

---

*จบ Part 66: Service Mesh*
