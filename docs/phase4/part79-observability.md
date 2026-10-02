# Part 79: Observability in Go

## เป้าหมายของบทเรียน
- เข้าใจ Three Pillars of Observability: Logs, Metrics, Traces
- ทำ OpenTelemetry deep dive
- ใช้ Correlation IDs
- Distributed context propagation
- สร้าง SLI/SLO dashboards
- ออกแบบ alerting strategies
- จัดการ incident response

---

## 1. The Three Pillars: Logs, Metrics, Traces

### Structured Logging

```go
// logging/structured.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "os"
    "runtime"
    "time"
)

// LogLevel ระดับของ log
type LogLevel int

const (
    DEBUG LogLevel = iota
    INFO
    WARN
    ERROR
    FATAL
)

func (l LogLevel) String() string {
    switch l {
    case DEBUG:
        return "DEBUG"
    case INFO:
        return "INFO"
    case WARN:
        return "WARN"
    case ERROR:
        return "ERROR"
    case FATAL:
        return "FATAL"
    default:
        return "UNKNOWN"
    }
}

// LogEntry แสดง log entry
type LogEntry struct {
    Timestamp   string                 `json:"timestamp"`
    Level       string                 `json:"level"`
    Message     string                 `json:"message"`
    Service     string                 `json:"service"`
    TraceID     string                 `json:"trace_id,omitempty"`
    SpanID      string                 `json:"span_id,omitempty"`
    RequestID   string                 `json:"request_id,omitempty"`
    UserID      string                 `json:"user_id,omitempty"`
    Fields      map[string]interface{} `json:"fields,omitempty"`
    Caller      string                 `json:"caller,omitempty"`
    ErrorStack  string                 `json:"error_stack,omitempty"`
}

// Logger structured logger
type Logger struct {
    output  io.Writer
    service string
    level   LogLevel
    fields  map[string]interface{}
}

// NewLogger สร้าง logger ใหม่
func NewLogger(service string, level LogLevel) *Logger {
    return &Logger{
        output:  os.Stdout,
        service: service,
        level:   level,
        fields:  make(map[string]interface{}),
    }
}

// WithField เพิ่ม field
func (l *Logger) WithField(key string, value interface{}) *Logger {
    newLogger := &Logger{
        output:  l.output,
        service: l.service,
        level:   l.level,
        fields:  make(map[string]interface{}),
    }
    for k, v := range l.fields {
        newLogger.fields[k] = v
    }
    newLogger.fields[key] = value
    return newLogger
}

// WithContext เพิ่ม context values
func (l *Logger) WithContext(ctx context.Context) *Logger {
    newLogger := l.WithField("dummy", nil)
    delete(newLogger.fields, "dummy")
    
    if traceID := ctx.Value("trace_id"); traceID != nil {
        newLogger.fields["trace_id"] = traceID
    }
    if requestID := ctx.Value("request_id"); requestID != nil {
        newLogger.fields["request_id"] = requestID
    }
    return newLogger
}

// log เขียน log entry
func (l *Logger) log(ctx context.Context, level LogLevel, msg string, extraFields ...map[string]interface{}) {
    if level < l.level {
        return
    }
    
    _, file, line, _ := runtime.Caller(2)
    
    entry := LogEntry{
        Timestamp: time.Now().UTC().Format(time.RFC3339Nano),
        Level:     level.String(),
        Message:   msg,
        Service:   l.service,
        Caller:    fmt.Sprintf("%s:%d", file, line),
        Fields:    make(map[string]interface{}),
    }
    
    // Copy fields
    for k, v := range l.fields {
        entry.Fields[k] = v
    }
    
    // Extra fields
    for _, ef := range extraFields {
        for k, v := range ef {
            entry.Fields[k] = v
        }
    }
    
    // Context values
    if ctx != nil {
        if traceID := ctx.Value("trace_id"); traceID != nil {
            entry.TraceID = fmt.Sprintf("%v", traceID)
        }
        if spanID := ctx.Value("span_id"); spanID != nil {
            entry.SpanID = fmt.Sprintf("%v", spanID)
        }
        if requestID := ctx.Value("request_id"); requestID != nil {
            entry.RequestID = fmt.Sprintf("%v", requestID)
        }
    }
    
    data, _ := json.Marshal(entry)
    fmt.Fprintln(l.output, string(data))
    
    if level == FATAL {
        os.Exit(1)
    }
}

func (l *Logger) Debug(ctx context.Context, msg string, fields ...map[string]interface{}) {
    l.log(ctx, DEBUG, msg, fields...)
}

func (l *Logger) Info(ctx context.Context, msg string, fields ...map[string]interface{}) {
    l.log(ctx, INFO, msg, fields...)
}

func (l *Logger) Warn(ctx context.Context, msg string, fields ...map[string]interface{}) {
    l.log(ctx, WARN, msg, fields...)
}

func (l *Logger) Error(ctx context.Context, msg string, fields ...map[string]interface{}) {
    l.log(ctx, ERROR, msg, fields...)
}

func main() {
    logger := NewLogger("payment-service", INFO)
    
    ctx := context.WithValue(context.Background(), "trace_id", "abc123def456")
    ctx = context.WithValue(ctx, "request_id", "req-789xyz")
    
    logger.Info(ctx, "Processing payment", map[string]interface{}{
        "amount":   100.00,
        "currency": "THB",
        "user_id":  "user-123",
    })
    
    logger.WithField("component", "database").Info(ctx, "Executing query", map[string]interface{}{
        "query":    "SELECT * FROM payments WHERE id = $1",
        "duration": "15ms",
    })
    
    logger.Error(ctx, "Payment failed", map[string]interface{}{
        "error":      "insufficient funds",
        "amount":     500.00,
        "balance":    200.00,
    })
}
```

---

## 2. Metrics with Prometheus

```go
// metrics/prometheus.go
package main

import (
    "fmt"
    "math/rand"
    "net/http"
    "sync"
    "time"
)

// MetricType ประเภทของ metric
type MetricType string

const (
    Counter   MetricType = "counter"
    Gauge     MetricType = "gauge"
    Histogram MetricType = "histogram"
    Summary   MetricType = "summary"
)

// CounterMetric นับจำนวน events
type CounterMetric struct {
    mu     sync.Mutex
    name   string
    help   string
    labels map[string]string
    value  float64
}

// Inc เพิ่มค่า counter
func (c *CounterMetric) Inc(labels ...map[string]string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

// Add เพิ่มค่าตามจำนวน
func (c *CounterMetric) Add(v float64, labels ...map[string]string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value += v
}

// Value คืนค่าปัจจุบัน
func (c *CounterMetric) Value() float64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

// GaugeMetric วัดค่าปัจจุบัน
type GaugeMetric struct {
    mu    sync.Mutex
    name  string
    help  string
    value float64
}

func (g *GaugeMetric) Set(v float64) {
    g.mu.Lock()
    defer g.mu.Unlock()
    g.value = v
}

func (g *GaugeMetric) Inc() {
    g.mu.Lock()
    defer g.mu.Unlock()
    g.value++
}

func (g *GaugeMetric) Dec() {
    g.mu.Lock()
    defer g.mu.Unlock()
    g.value--
}

func (g *GaugeMetric) Value() float64 {
    g.mu.Lock()
    defer g.mu.Unlock()
    return g.value
}

// HistogramMetric วัด distribution ของค่า
type HistogramMetric struct {
    mu      sync.Mutex
    name    string
    help    string
    buckets []float64
    counts  []int64
    sum     float64
    count   int64
}

// NewHistogram สร้าง histogram ใหม่
func NewHistogram(name, help string, buckets []float64) *HistogramMetric {
    return &HistogramMetric{
        name:    name,
        help:    help,
        buckets: buckets,
        counts:  make([]int64, len(buckets)+1),
    }
}

// Observe เพิ่มค่า observation
func (h *HistogramMetric) Observe(v float64) {
    h.mu.Lock()
    defer h.mu.Unlock()
    
    h.sum += v
    h.count++
    
    for i, b := range h.buckets {
        if v <= b {
            h.counts[i]++
        }
    }
    h.counts[len(h.buckets)]++ // +Inf bucket
}

// Percentile คำนวณ percentile
func (h *HistogramMetric) Percentile(p float64) float64 {
    h.mu.Lock()
    defer h.mu.Unlock()
    
    if h.count == 0 {
        return 0
    }
    
    target := int64(float64(h.count) * p / 100.0)
    
    var cumulative int64
    for i, b := range h.buckets {
        cumulative += h.counts[i]
        if cumulative >= target {
            return b
        }
    }
    
    return h.buckets[len(h.buckets)-1]
}

// MetricsRegistry เก็บ metrics ทั้งหมด
type MetricsRegistry struct {
    mu       sync.RWMutex
    counters  map[string]*CounterMetric
    gauges    map[string]*GaugeMetric
    histograms map[string]*HistogramMetric
}

// NewMetricsRegistry สร้าง registry ใหม่
func NewMetricsRegistry() *MetricsRegistry {
    return &MetricsRegistry{
        counters:   make(map[string]*CounterMetric),
        gauges:     make(map[string]*GaugeMetric),
        histograms: make(map[string]*HistogramMetric),
    }
}

// NewCounter สร้าง counter metric
func (r *MetricsRegistry) NewCounter(name, help string) *CounterMetric {
    r.mu.Lock()
    defer r.mu.Unlock()
    c := &CounterMetric{name: name, help: help}
    r.counters[name] = c
    return c
}

// NewGauge สร้าง gauge metric
func (r *MetricsRegistry) NewGauge(name, help string) *GaugeMetric {
    r.mu.Lock()
    defer r.mu.Unlock()
    g := &GaugeMetric{name: name, help: help}
    r.gauges[name] = g
    return g
}

// NewHistogram สร้าง histogram metric
func (r *MetricsRegistry) NewHistogram(name, help string, buckets []float64) *HistogramMetric {
    r.mu.Lock()
    defer r.mu.Unlock()
    h := NewHistogram(name, help, buckets)
    r.histograms[name] = h
    return h
}

// Expose แสดง metrics ใน Prometheus format
func (r *MetricsRegistry) Expose() string {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    result := ""
    
    for name, c := range r.counters {
        result += fmt.Sprintf("# HELP %s %s\n", name, c.help)
        result += fmt.Sprintf("# TYPE %s counter\n", name)
        result += fmt.Sprintf("%s %.2f\n\n", name, c.Value())
    }
    
    for name, g := range r.gauges {
        result += fmt.Sprintf("# HELP %s %s\n", name, g.help)
        result += fmt.Sprintf("# TYPE %s gauge\n", name)
        result += fmt.Sprintf("%s %.2f\n\n", name, g.Value())
    }
    
    for name, h := range r.histograms {
        h.mu.Lock()
        result += fmt.Sprintf("# HELP %s %s\n", name, h.help)
        result += fmt.Sprintf("# TYPE %s histogram\n", name)
        for i, b := range h.buckets {
            result += fmt.Sprintf("%s_bucket{le=\"%.3f\"} %d\n", name, b, h.counts[i])
        }
        result += fmt.Sprintf("%s_bucket{le=\"+Inf\"} %d\n", name, h.count)
        result += fmt.Sprintf("%s_sum %.4f\n", name, h.sum)
        result += fmt.Sprintf("%s_count %d\n\n", name, h.count)
        h.mu.Unlock()
    }
    
    return result
}

// SLI/SLO Metrics
type SLIMetrics struct {
    RequestTotal   *CounterMetric
    RequestSuccess *CounterMetric
    RequestLatency *HistogramMetric
    ActiveUsers    *GaugeMetric
}

// NewSLIMetrics สร้าง SLI metrics
func NewSLIMetrics(registry *MetricsRegistry) *SLIMetrics {
    return &SLIMetrics{
        RequestTotal: registry.NewCounter(
            "http_requests_total",
            "Total number of HTTP requests",
        ),
        RequestSuccess: registry.NewCounter(
            "http_requests_success_total",
            "Total number of successful HTTP requests",
        ),
        RequestLatency: registry.NewHistogram(
            "http_request_duration_seconds",
            "HTTP request latency in seconds",
            []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
        ),
        ActiveUsers: registry.NewGauge(
            "active_users",
            "Number of currently active users",
        ),
    }
}

// TrackRequest บันทึก request
func (s *SLIMetrics) TrackRequest(success bool, duration time.Duration) {
    s.RequestTotal.Inc()
    if success {
        s.RequestSuccess.Inc()
    }
    s.RequestLatency.Observe(duration.Seconds())
}

// SuccessRate คืนอัตราสำเร็จ
func (s *SLIMetrics) SuccessRate() float64 {
    total := s.RequestTotal.Value()
    if total == 0 {
        return 1.0
    }
    return s.RequestSuccess.Value() / total
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    registry := NewMetricsRegistry()
    sli := NewSLIMetrics(registry)
    
    // จำลอง HTTP requests
    fmt.Println("Simulating HTTP requests...")
    
    for i := 0; i < 1000; i++ {
        latency := time.Duration(rand.Intn(500)) * time.Millisecond
        success := rand.Float64() < 0.98 // 98% success rate
        
        sli.TrackRequest(success, latency)
        sli.ActiveUsers.Set(float64(rand.Intn(100) + 50))
    }
    
    fmt.Printf("Success Rate: %.2f%%\n", sli.SuccessRate()*100)
    fmt.Printf("P50 Latency: %.3fs\n", sli.RequestLatency.Percentile(50))
    fmt.Printf("P95 Latency: %.3fs\n", sli.RequestLatency.Percentile(95))
    fmt.Printf("P99 Latency: %.3fs\n", sli.RequestLatency.Percentile(99))
    
    // Setup metrics endpoint
    http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, registry.Expose())
    })
    
    fmt.Println("\nMetrics available at http://localhost:9090/metrics")
    fmt.Println("(Not starting server in demo mode)")
}
```

---

## 3. OpenTelemetry Deep Dive

```go
// otel/tracing.go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

// TraceID unique identifier สำหรับ trace
type TraceID string

// SpanID unique identifier สำหรับ span
type SpanID string

// SpanKind ประเภทของ span
type SpanKind int

const (
    SpanKindInternal SpanKind = iota
    SpanKindServer
    SpanKindClient
    SpanKindProducer
    SpanKindConsumer
)

// SpanStatus สถานะของ span
type SpanStatus int

const (
    SpanStatusUnset SpanStatus = iota
    SpanStatusOK
    SpanStatusError
)

// Attribute key-value attribute
type Attribute struct {
    Key   string
    Value interface{}
}

// SpanContext context ของ span
type SpanContext struct {
    TraceID TraceID
    SpanID  SpanID
}

// Span แสดง distributed trace span
type Span struct {
    ctx        context.Context
    name       string
    traceID    TraceID
    spanID     SpanID
    parentID   SpanID
    kind       SpanKind
    status     SpanStatus
    startTime  time.Time
    endTime    time.Time
    attributes []Attribute
    events     []SpanEvent
    tracer     *Tracer
}

// SpanEvent event ใน span
type SpanEvent struct {
    Name       string
    Timestamp  time.Time
    Attributes []Attribute
}

// SetAttribute เพิ่ม attribute
func (s *Span) SetAttribute(key string, value interface{}) {
    s.attributes = append(s.attributes, Attribute{Key: key, Value: value})
}

// AddEvent เพิ่ม event
func (s *Span) AddEvent(name string, attrs ...Attribute) {
    s.events = append(s.events, SpanEvent{
        Name:       name,
        Timestamp:  time.Now(),
        Attributes: attrs,
    })
}

// SetStatus กำหนด status
func (s *Span) SetStatus(status SpanStatus, description string) {
    s.status = status
    if description != "" {
        s.SetAttribute("status.description", description)
    }
}

// End จบ span
func (s *Span) End() {
    s.endTime = time.Now()
    if s.tracer != nil {
        s.tracer.export(s)
    }
}

// Context คืน context ที่มี span
func (s *Span) Context() context.Context {
    return context.WithValue(s.ctx, "span", s)
}

// Tracer สำหรับสร้าง spans
type Tracer struct {
    serviceName string
    exporter    SpanExporter
}

// SpanExporter interface สำหรับ export spans
type SpanExporter interface {
    Export(span *Span)
}

// ConsoleExporter export spans ไปยัง console
type ConsoleExporter struct{}

func (e *ConsoleExporter) Export(span *Span) {
    duration := span.endTime.Sub(span.startTime)
    
    status := "OK"
    if span.status == SpanStatusError {
        status = "ERROR"
    }
    
    fmt.Printf("[Trace] %s | span=%s | trace=%s | duration=%v | status=%s\n",
        span.name,
        string(span.spanID)[:8],
        string(span.traceID)[:8],
        duration.Round(time.Millisecond),
        status,
    )
    
    for _, attr := range span.attributes {
        fmt.Printf("  attr: %s=%v\n", attr.Key, attr.Value)
    }
    
    for _, event := range span.events {
        fmt.Printf("  event: %s @ %v\n", event.Name, event.Timestamp.Format("15:04:05.000"))
    }
}

// NewTracer สร้าง tracer ใหม่
func NewTracer(serviceName string, exporter SpanExporter) *Tracer {
    return &Tracer{
        serviceName: serviceName,
        exporter:    exporter,
    }
}

// Start เริ่ม span ใหม่
func (t *Tracer) Start(ctx context.Context, name string) (context.Context, *Span) {
    // หา parent span จาก context
    var parentID SpanID
    var traceID TraceID
    
    if parentSpan, ok := ctx.Value("span").(*Span); ok && parentSpan != nil {
        parentID = parentSpan.spanID
        traceID = parentSpan.traceID
    } else {
        traceID = generateTraceID()
    }
    
    span := &Span{
        ctx:       ctx,
        name:      name,
        traceID:   traceID,
        spanID:    generateSpanID(),
        parentID:  parentID,
        kind:      SpanKindInternal,
        status:    SpanStatusUnset,
        startTime: time.Now(),
        tracer:    t,
    }
    
    newCtx := context.WithValue(ctx, "span", span)
    span.ctx = newCtx
    
    return newCtx, span
}

func (t *Tracer) export(span *Span) {
    if t.exporter != nil {
        t.exporter.Export(span)
    }
}

func generateTraceID() TraceID {
    return TraceID(fmt.Sprintf("%016x%016x", rand.Int63(), rand.Int63()))
}

func generateSpanID() SpanID {
    return SpanID(fmt.Sprintf("%016x", rand.Int63()))
}

// Middleware สำหรับ HTTP tracing
func TracingMiddleware(tracer *Tracer, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx, span := tracer.Start(r.Context(), fmt.Sprintf("HTTP %s %s", r.Method, r.URL.Path))
        defer span.End()
        
        span.SetAttribute("http.method", r.Method)
        span.SetAttribute("http.url", r.URL.String())
        span.SetAttribute("http.user_agent", r.UserAgent())
        
        // สร้าง response writer ที่จับ status code
        wrapped := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(wrapped, r.WithContext(ctx))
        
        span.SetAttribute("http.status_code", wrapped.status)
        if wrapped.status >= 500 {
            span.SetStatus(SpanStatusError, fmt.Sprintf("HTTP %d", wrapped.status))
        } else {
            span.SetStatus(SpanStatusOK, "")
        }
    })
}

type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    exporter := &ConsoleExporter{}
    tracer := NewTracer("order-service", exporter)
    
    ctx := context.Background()
    
    // จำลอง request processing พร้อม nested spans
    fmt.Println("=== OpenTelemetry Tracing Demo ===\n")
    
    ctx, rootSpan := tracer.Start(ctx, "ProcessOrder")
    rootSpan.SetAttribute("order.id", "ord-123")
    rootSpan.SetAttribute("user.id", "user-456")
    
    // Child span: Validate order
    ctx2, validateSpan := tracer.Start(ctx, "ValidateOrder")
    validateSpan.SetAttribute("items.count", 3)
    time.Sleep(10 * time.Millisecond)
    validateSpan.AddEvent("validation.complete")
    validateSpan.End()
    _ = ctx2
    
    // Child span: Check inventory
    _, inventorySpan := tracer.Start(ctx, "CheckInventory")
    inventorySpan.SetAttribute("warehouse", "BKK-01")
    time.Sleep(15 * time.Millisecond)
    inventorySpan.AddEvent("inventory.confirmed", Attribute{"available", true})
    inventorySpan.End()
    
    // Child span: Process payment
    _, paymentSpan := tracer.Start(ctx, "ProcessPayment")
    paymentSpan.SetAttribute("amount", 500.00)
    paymentSpan.SetAttribute("currency", "THB")
    paymentSpan.SetAttribute("method", "credit_card")
    time.Sleep(50 * time.Millisecond)
    paymentSpan.AddEvent("payment.authorized")
    paymentSpan.End()
    
    // Child span: Send confirmation
    _, notifySpan := tracer.Start(ctx, "SendConfirmation")
    notifySpan.SetAttribute("email", "user@example.com")
    time.Sleep(5 * time.Millisecond)
    notifySpan.End()
    
    rootSpan.SetStatus(SpanStatusOK, "")
    rootSpan.End()
}
```

---

## 4. Correlation IDs & Context Propagation

```go
// correlation/propagation.go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
    "math/rand"
)

// ContextKey ประเภท key สำหรับ context
type ContextKey string

const (
    CorrelationIDKey ContextKey = "correlation_id"
    TraceIDKey       ContextKey = "trace_id"
    SpanIDKey        ContextKey = "span_id"
    UserIDKey        ContextKey = "user_id"
    SessionIDKey     ContextKey = "session_id"
)

// RequestContext เก็บ context ของ request
type RequestContext struct {
    CorrelationID string
    TraceID       string
    SpanID        string
    UserID        string
    SessionID     string
    StartTime     time.Time
}

// NewRequestContext สร้าง context ใหม่
func NewRequestContext() *RequestContext {
    return &RequestContext{
        CorrelationID: generateID(),
        TraceID:       generateID(),
        SpanID:        generateShortID(),
        StartTime:     time.Now(),
    }
}

func generateID() string {
    return fmt.Sprintf("%08x-%04x-%04x", rand.Int31(), rand.Int31n(0xffff), rand.Int31n(0xffff))
}

func generateShortID() string {
    return fmt.Sprintf("%08x", rand.Int31())
}

// ToContext บรรจุลงใน context
func (rc *RequestContext) ToContext(ctx context.Context) context.Context {
    ctx = context.WithValue(ctx, CorrelationIDKey, rc.CorrelationID)
    ctx = context.WithValue(ctx, TraceIDKey, rc.TraceID)
    ctx = context.WithValue(ctx, SpanIDKey, rc.SpanID)
    if rc.UserID != "" {
        ctx = context.WithValue(ctx, UserIDKey, rc.UserID)
    }
    return ctx
}

// FromContext ดึงข้อมูลจาก context
func FromContext(ctx context.Context) *RequestContext {
    rc := &RequestContext{}
    
    if v := ctx.Value(CorrelationIDKey); v != nil {
        rc.CorrelationID = v.(string)
    }
    if v := ctx.Value(TraceIDKey); v != nil {
        rc.TraceID = v.(string)
    }
    if v := ctx.Value(SpanIDKey); v != nil {
        rc.SpanID = v.(string)
    }
    if v := ctx.Value(UserIDKey); v != nil {
        rc.UserID = v.(string)
    }
    
    return rc
}

// HTTP Header names สำหรับ propagation
const (
    HeaderCorrelationID = "X-Correlation-ID"
    HeaderTraceID       = "X-Trace-ID"
    HeaderSpanID        = "X-Span-ID"
    HeaderUserID        = "X-User-ID"
)

// InjectHeaders ฉีด context เข้า HTTP headers
func InjectHeaders(ctx context.Context, headers http.Header) {
    rc := FromContext(ctx)
    
    if rc.CorrelationID != "" {
        headers.Set(HeaderCorrelationID, rc.CorrelationID)
    }
    if rc.TraceID != "" {
        headers.Set(HeaderTraceID, rc.TraceID)
    }
    if rc.SpanID != "" {
        headers.Set(HeaderSpanID, rc.SpanID)
    }
    if rc.UserID != "" {
        headers.Set(HeaderUserID, rc.UserID)
    }
}

// ExtractHeaders ดึง context จาก HTTP headers
func ExtractHeaders(headers http.Header) *RequestContext {
    rc := &RequestContext{
        CorrelationID: headers.Get(HeaderCorrelationID),
        TraceID:       headers.Get(HeaderTraceID),
        SpanID:        headers.Get(HeaderSpanID),
        UserID:        headers.Get(HeaderUserID),
        StartTime:     time.Now(),
    }
    
    if rc.CorrelationID == "" {
        rc.CorrelationID = generateID()
    }
    if rc.TraceID == "" {
        rc.TraceID = generateID()
    }
    if rc.SpanID == "" {
        rc.SpanID = generateShortID()
    }
    
    return rc
}

// CorrelationMiddleware middleware สำหรับ HTTP
func CorrelationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        rc := ExtractHeaders(r.Header)
        ctx := rc.ToContext(r.Context())
        
        // เพิ่ม correlation ID ใน response headers
        w.Header().Set(HeaderCorrelationID, rc.CorrelationID)
        w.Header().Set(HeaderTraceID, rc.TraceID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// ServiceClient จำลอง service-to-service client
type ServiceClient struct {
    name     string
    baseURL  string
    client   *http.Client
}

// NewServiceClient สร้าง client ใหม่
func NewServiceClient(name, baseURL string) *ServiceClient {
    return &ServiceClient{
        name:    name,
        baseURL: baseURL,
        client:  &http.Client{Timeout: 10 * time.Second},
    }
}

// Get ส่ง GET request พร้อม context propagation
func (sc *ServiceClient) Get(ctx context.Context, path string) error {
    req, err := http.NewRequestWithContext(ctx, "GET", sc.baseURL+path, nil)
    if err != nil {
        return err
    }
    
    // Propagate context
    InjectHeaders(ctx, req.Header)
    
    rc := FromContext(ctx)
    fmt.Printf("[%s] Calling %s%s (correlation_id: %s)\n",
        sc.name, sc.baseURL, path, rc.CorrelationID[:8])
    
    // ในการทำงานจริง: return sc.client.Do(req)
    return nil
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    fmt.Println("=== Correlation ID & Context Propagation ===\n")
    
    // จำลอง incoming request
    incomingHeaders := make(http.Header)
    // บางครั้ง client ส่ง correlation ID มาเอง
    // incomingHeaders.Set(HeaderCorrelationID, "external-req-123")
    
    rc := ExtractHeaders(incomingHeaders)
    ctx := rc.ToContext(context.Background())
    
    fmt.Printf("Request context:\n")
    fmt.Printf("  Correlation ID: %s\n", rc.CorrelationID)
    fmt.Printf("  Trace ID: %s\n", rc.TraceID)
    fmt.Printf("  Span ID: %s\n", rc.SpanID)
    
    // จำลอง service calls
    orderService := NewServiceClient("gateway", "http://order-service")
    inventoryService := NewServiceClient("gateway", "http://inventory-service")
    paymentService := NewServiceClient("gateway", "http://payment-service")
    
    orderService.Get(ctx, "/api/orders")
    inventoryService.Get(ctx, "/api/inventory/check")
    paymentService.Get(ctx, "/api/payments/process")
    
    // แสดง headers ที่จะส่งไป downstream services
    fmt.Println("\nHeaders propagated to downstream services:")
    outHeaders := make(http.Header)
    InjectHeaders(ctx, outHeaders)
    for k, v := range outHeaders {
        fmt.Printf("  %s: %s\n", k, v[0])
    }
}
```

---

## 5. SLI/SLO Dashboard & Alerting

```go
// slo/manager.go
package main

import (
    "fmt"
    "math"
    "time"
)

// SLI Service Level Indicator
type SLI struct {
    Name        string
    Description string
    Query       string // PromQL query
    GoodEvents  func() float64
    TotalEvents func() float64
}

// Calculate คำนวณ SLI value
func (s *SLI) Calculate() float64 {
    good := s.GoodEvents()
    total := s.TotalEvents()
    if total == 0 {
        return 1.0
    }
    return good / total
}

// SLO Service Level Objective
type SLO struct {
    Name        string
    SLI         *SLI
    Target      float64       // เช่น 0.999 = 99.9%
    Window      time.Duration // ระยะเวลา
    ErrorBudget float64       // คำนวณอัตโนมัติ
}

// NewSLO สร้าง SLO ใหม่
func NewSLO(name string, sli *SLI, target float64, window time.Duration) *SLO {
    return &SLO{
        Name:        name,
        SLI:         sli,
        Target:      target,
        Window:      window,
        ErrorBudget: 1 - target,
    }
}

// ErrorBudgetRemaining คำนวณ error budget ที่เหลือ
func (s *SLO) ErrorBudgetRemaining() float64 {
    currentSLI := s.SLI.Calculate()
    consumedBudget := s.Target - currentSLI
    if consumedBudget < 0 {
        return s.ErrorBudget // 100% remaining
    }
    remaining := s.ErrorBudget - consumedBudget
    if remaining < 0 {
        return 0
    }
    return remaining
}

// ErrorBudgetRemainingPercent คืนเปอร์เซ็นต์ error budget ที่เหลือ
func (s *SLO) ErrorBudgetRemainingPercent() float64 {
    return (s.ErrorBudgetRemaining() / s.ErrorBudget) * 100
}

// IsMet ตรวจสอบว่า SLO ถูก meet หรือไม่
func (s *SLO) IsMet() bool {
    return s.SLI.Calculate() >= s.Target
}

// BurnRate คำนวณ error budget burn rate
func (s *SLO) BurnRate() float64 {
    currentSLI := s.SLI.Calculate()
    if s.ErrorBudget == 0 {
        return math.Inf(1)
    }
    errorRate := 1 - currentSLI
    return errorRate / s.ErrorBudget
}

// AlertPolicy กำหนด alerting policy
type AlertPolicy struct {
    Name        string
    SLO         *SLO
    BurnRateThreshold float64
    ShortWindow time.Duration
    LongWindow  time.Duration
}

// ShouldAlert ตรวจสอบว่าควร alert หรือไม่
func (ap *AlertPolicy) ShouldAlert() (bool, string) {
    burnRate := ap.SLO.BurnRate()
    
    if burnRate > ap.BurnRateThreshold {
        return true, fmt.Sprintf(
            "Error budget burning too fast: burn_rate=%.2f (threshold=%.2f)",
            burnRate, ap.BurnRateThreshold,
        )
    }
    
    if !ap.SLO.IsMet() {
        return true, fmt.Sprintf(
            "SLO violated: current=%.4f%%, target=%.4f%%",
            ap.SLO.SLI.Calculate()*100,
            ap.SLO.Target*100,
        )
    }
    
    remaining := ap.SLO.ErrorBudgetRemainingPercent()
    if remaining < 10 {
        return true, fmt.Sprintf(
            "Error budget nearly exhausted: %.1f%% remaining",
            remaining,
        )
    }
    
    return false, ""
}

// SLODashboard แสดง SLO dashboard
type SLODashboard struct {
    slos    []*SLO
    alerts  []*AlertPolicy
}

// NewSLODashboard สร้าง dashboard ใหม่
func NewSLODashboard() *SLODashboard {
    return &SLODashboard{
        slos:   make([]*SLO, 0),
        alerts: make([]*AlertPolicy, 0),
    }
}

// AddSLO เพิ่ม SLO
func (d *SLODashboard) AddSLO(slo *SLO) {
    d.slos = append(d.slos, slo)
}

// AddAlert เพิ่ม alert policy
func (d *SLODashboard) AddAlert(alert *AlertPolicy) {
    d.alerts = append(d.alerts, alert)
}

// Print แสดง dashboard
func (d *SLODashboard) Print() {
    fmt.Println("╔══════════════════════════════════════════════════════════╗")
    fmt.Println("║                    SLO Dashboard                        ║")
    fmt.Println("╠══════════════════════════════════════════════════════════╣")
    
    for _, slo := range d.slos {
        sliValue := slo.SLI.Calculate() * 100
        isMet := slo.IsMet()
        remaining := slo.ErrorBudgetRemainingPercent()
        burnRate := slo.BurnRate()
        
        status := "✓ MET"
        if !isMet {
            status = "✗ VIOLATED"
        }
        
        fmt.Printf("║ %-20s                                        ║\n", slo.Name)
        fmt.Printf("║   SLI:           %.4f%%                              ║\n", sliValue)
        fmt.Printf("║   Target:        %.4f%%                              ║\n", slo.Target*100)
        fmt.Printf("║   Status:        %-10s                              ║\n", status)
        fmt.Printf("║   Error Budget:  %.1f%% remaining                     ║\n", remaining)
        fmt.Printf("║   Burn Rate:     %.2fx                                ║\n", burnRate)
        fmt.Println("║──────────────────────────────────────────────────────║")
    }
    
    // Alerts
    fmt.Println("║                    Active Alerts                        ║")
    fmt.Println("║──────────────────────────────────────────────────────────║")
    
    hasAlerts := false
    for _, alert := range d.alerts {
        if firing, reason := alert.ShouldAlert(); firing {
            hasAlerts = true
            fmt.Printf("║ ⚠ %-54s ║\n", alert.Name)
            fmt.Printf("║   %-54s ║\n", reason)
        }
    }
    
    if !hasAlerts {
        fmt.Println("║  No active alerts                                       ║")
    }
    
    fmt.Println("╚══════════════════════════════════════════════════════════╝")
}

func main() {
    // จำลอง metrics
    var successCount, totalCount float64
    successCount = 9985
    totalCount = 10000
    
    // สร้าง SLIs
    availabilitySLI := &SLI{
        Name:        "availability",
        Description: "เปอร์เซ็นต์ requests ที่สำเร็จ",
        GoodEvents:  func() float64 { return successCount },
        TotalEvents: func() float64 { return totalCount },
    }
    
    // สร้าง SLOs
    availabilitySLO := NewSLO(
        "Payment Service Availability",
        availabilitySLI,
        0.999, // 99.9%
        30*24*time.Hour,
    )
    
    // สร้าง Dashboard
    dashboard := NewSLODashboard()
    dashboard.AddSLO(availabilitySLO)
    
    // สร้าง Alert policies
    dashboard.AddAlert(&AlertPolicy{
        Name:              "High Burn Rate Alert",
        SLO:               availabilitySLO,
        BurnRateThreshold: 2.0,
        ShortWindow:       1 * time.Hour,
        LongWindow:        6 * time.Hour,
    })
    
    // แสดง dashboard
    dashboard.Print()
    
    // จำลอง SLO violation
    fmt.Println("\n--- Simulating SLO violation ---")
    successCount = 9950
    dashboard.Print()
}
```

---

## 6. Incident Response

```go
// incident/response.go
package main

import (
    "fmt"
    "strings"
    "time"
)

// IncidentSeverity ระดับความรุนแรง
type IncidentSeverity int

const (
    SEV1 IncidentSeverity = 1 // Critical: ระบบล่ม
    SEV2 IncidentSeverity = 2 // High: ฟีเจอร์หลักใช้ไม่ได้
    SEV3 IncidentSeverity = 3 // Medium: ฟีเจอร์รองใช้ไม่ได้
    SEV4 IncidentSeverity = 4 // Low: ปัญหาเล็กน้อย
)

// IncidentStatus สถานะของ incident
type IncidentStatus string

const (
    StatusOpen        IncidentStatus = "OPEN"
    StatusInvestigating IncidentStatus = "INVESTIGATING"
    StatusIdentified  IncidentStatus = "IDENTIFIED"
    StatusMonitoring  IncidentStatus = "MONITORING"
    StatusResolved    IncidentStatus = "RESOLVED"
)

// Incident แสดง incident
type Incident struct {
    ID          string
    Title       string
    Description string
    Severity    IncidentSeverity
    Status      IncidentStatus
    StartTime   time.Time
    EndTime     *time.Time
    Commander   string
    Teams       []string
    Timeline    []TimelineEntry
    Impact      string
    RootCause   string
    Resolution  string
}

// TimelineEntry entry ใน incident timeline
type TimelineEntry struct {
    Timestamp  time.Time
    Author     string
    Message    string
    Type       TimelineType
}

// TimelineType ประเภทของ timeline entry
type TimelineType string

const (
    TypeUpdate   TimelineType = "UPDATE"
    TypeAction   TimelineType = "ACTION"
    TypeMilestone TimelineType = "MILESTONE"
)

// NewIncident สร้าง incident ใหม่
func NewIncident(title string, severity IncidentSeverity) *Incident {
    return &Incident{
        ID:        fmt.Sprintf("INC-%d", time.Now().Unix()),
        Title:     title,
        Severity:  severity,
        Status:    StatusOpen,
        StartTime: time.Now(),
        Timeline:  make([]TimelineEntry, 0),
        Teams:     make([]string, 0),
    }
}

// AddUpdate เพิ่ม update
func (i *Incident) AddUpdate(author, message string) {
    i.Timeline = append(i.Timeline, TimelineEntry{
        Timestamp: time.Now(),
        Author:    author,
        Message:   message,
        Type:      TypeUpdate,
    })
}

// SetStatus อัปเดต status
func (i *Incident) SetStatus(status IncidentStatus, author, note string) {
    i.Status = status
    message := fmt.Sprintf("Status changed to %s. %s", status, note)
    i.Timeline = append(i.Timeline, TimelineEntry{
        Timestamp: time.Now(),
        Author:    author,
        Message:   message,
        Type:      TypeMilestone,
    })
}

// Resolve resolve incident
func (i *Incident) Resolve(author, resolution string) {
    now := time.Now()
    i.EndTime = &now
    i.Resolution = resolution
    i.SetStatus(StatusResolved, author, resolution)
}

// Duration คืนระยะเวลาของ incident
func (i *Incident) Duration() time.Duration {
    if i.EndTime != nil {
        return i.EndTime.Sub(i.StartTime)
    }
    return time.Since(i.StartTime)
}

// Print แสดง incident details
func (i *Incident) Print() {
    severityStr := map[IncidentSeverity]string{
        SEV1: "SEV1 🔴",
        SEV2: "SEV2 🟠",
        SEV3: "SEV3 🟡",
        SEV4: "SEV4 🟢",
    }[i.Severity]
    
    fmt.Println(strings.Repeat("=", 60))
    fmt.Printf("Incident: %s\n", i.ID)
    fmt.Printf("Title: %s\n", i.Title)
    fmt.Printf("Severity: %s\n", severityStr)
    fmt.Printf("Status: %s\n", i.Status)
    fmt.Printf("Duration: %v\n", i.Duration().Round(time.Second))
    
    if i.Commander != "" {
        fmt.Printf("Commander: %s\n", i.Commander)
    }
    
    if i.Impact != "" {
        fmt.Printf("Impact: %s\n", i.Impact)
    }
    
    if i.RootCause != "" {
        fmt.Printf("Root Cause: %s\n", i.RootCause)
    }
    
    if i.Resolution != "" {
        fmt.Printf("Resolution: %s\n", i.Resolution)
    }
    
    if len(i.Timeline) > 0 {
        fmt.Println("\nTimeline:")
        for _, entry := range i.Timeline {
            fmt.Printf("  [%s] %s (%s): %s\n",
                entry.Timestamp.Format("15:04:05"),
                entry.Type,
                entry.Author,
                entry.Message,
            )
        }
    }
    
    fmt.Println(strings.Repeat("=", 60))
}

// PostMortemTemplate แม่แบบสำหรับ post-mortem
type PostMortemTemplate struct {
    Incident      *Incident
    Date          time.Time
    Authors       []string
    Summary       string
    Impact        string
    RootCause     string
    Timeline      []TimelineEntry
    ActionItems   []ActionItem
    LessonsLearned []string
}

// ActionItem งานที่ต้องทำหลัง incident
type ActionItem struct {
    Description string
    Owner       string
    DueDate     time.Time
    Priority    string
    Status      string
}

// GeneratePostMortem สร้าง post-mortem document
func GeneratePostMortem(incident *Incident) *PostMortemTemplate {
    return &PostMortemTemplate{
        Incident: incident,
        Date:     time.Now(),
        Timeline: incident.Timeline,
    }
}

// Print แสดง post-mortem
func (pm *PostMortemTemplate) Print() {
    fmt.Println("# Post-Mortem Report")
    fmt.Printf("**Date:** %s\n", pm.Date.Format("2006-01-02"))
    fmt.Printf("**Incident:** %s - %s\n\n", pm.Incident.ID, pm.Incident.Title)
    
    fmt.Println("## Summary")
    fmt.Println(pm.Summary)
    
    fmt.Println("\n## Impact")
    fmt.Println(pm.Incident.Impact)
    
    fmt.Println("\n## Root Cause")
    fmt.Println(pm.Incident.RootCause)
    
    fmt.Println("\n## Timeline")
    for _, entry := range pm.Timeline {
        fmt.Printf("- **%s** [%s]: %s\n",
            entry.Timestamp.Format("15:04:05"),
            entry.Author,
            entry.Message,
        )
    }
    
    if len(pm.ActionItems) > 0 {
        fmt.Println("\n## Action Items")
        for i, item := range pm.ActionItems {
            fmt.Printf("%d. **%s** - Owner: %s, Due: %s, Priority: %s\n",
                i+1,
                item.Description,
                item.Owner,
                item.DueDate.Format("2006-01-02"),
                item.Priority,
            )
        }
    }
    
    if len(pm.LessonsLearned) > 0 {
        fmt.Println("\n## Lessons Learned")
        for _, lesson := range pm.LessonsLearned {
            fmt.Printf("- %s\n", lesson)
        }
    }
}

func main() {
    // จำลอง incident lifecycle
    incident := NewIncident("Payment Service Unavailable", SEV1)
    incident.Commander = "On-Call Engineer"
    incident.Teams = []string{"Platform", "Backend", "Database"}
    incident.Impact = "100% ของ payment transactions ล้มเหลว ผู้ใช้ ~5,000 คนได้รับผลกระทบ"
    
    // Timeline
    incident.AddUpdate("monitoring-bot", "Alert fired: payment-service error rate > 50%")
    incident.SetStatus(StatusInvestigating, "on-call-eng", "กำลังตรวจสอบ")
    
    time.Sleep(100 * time.Millisecond)
    incident.AddUpdate("db-team", "พบ connection pool exhausted ใน database")
    incident.SetStatus(StatusIdentified, "db-team", "Root cause: database connection leak")
    incident.RootCause = "Database connection leak ใน payment service v2.3.1 ที่ deploy เมื่อ 30 นาทีที่แล้ว"
    
    time.Sleep(100 * time.Millisecond)
    incident.AddUpdate("backend-team", "Rollback payment service to v2.3.0")
    incident.SetStatus(StatusMonitoring, "backend-team", "Rolled back, monitoring recovery")
    
    time.Sleep(100 * time.Millisecond)
    incident.Resolve("on-call-eng", "Payment service rolled back to v2.3.0, all transactions processing normally")
    
    incident.Print()
    
    // สร้าง post-mortem
    pm := GeneratePostMortem(incident)
    pm.Summary = "Payment service กลายเป็น unavailable หลังจาก deploy v2.3.1 ที่มี connection leak bug"
    pm.LessonsLearned = []string{
        "ต้องมี canary deployment ก่อน full rollout",
        "ต้องมี connection pool monitoring alert",
        "ต้องทำ load test ที่ simulate production load",
    }
    pm.ActionItems = []ActionItem{
        {
            Description: "เพิ่ม connection pool metrics ใน dashboard",
            Owner:       "Platform Team",
            DueDate:     time.Now().Add(7 * 24 * time.Hour),
            Priority:    "HIGH",
            Status:      "TODO",
        },
        {
            Description: "สร้าง canary deployment pipeline",
            Owner:       "DevOps Team",
            DueDate:     time.Now().Add(14 * 24 * time.Hour),
            Priority:    "HIGH",
            Status:      "TODO",
        },
    }
    
    fmt.Println("\n")
    pm.Print()
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Structured Logging** - การเขียน logs แบบ structured พร้อม context
2. **Metrics** - Counter, Gauge, Histogram และ SLI metrics
3. **OpenTelemetry** - Distributed tracing ด้วย spans และ context propagation
4. **Correlation IDs** - การ trace requests ข้าม services
5. **SLI/SLO** - การวัดและจัดการ service levels
6. **Incident Response** - การจัดการ incidents และ post-mortems

### Key Takeaways

- **Logs + Metrics + Traces** ต้องทำงานร่วมกันเพื่อ observability ที่สมบูรณ์
- **Correlation IDs** เชื่อม logs จาก services ต่างๆ เข้าด้วยกัน
- **SLO-based alerting** ดีกว่า threshold-based alerting
- **Post-mortems** ต้อง blameless เพื่อเรียนรู้จากความผิดพลาด
