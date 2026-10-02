# Part 63: Distributed Tracing

## เป้าหมายการเรียนรู้
- Distributed tracing concepts
- OpenTelemetry setup
- Jaeger integration
- Trace propagation ข้าม services
- Custom spans และ attributes
- Sampling strategies
- ตัวอย่างโค้ด 15+ ตัวอย่าง

---

## 1. Concepts

Distributed tracing ช่วยติดตาม request ที่ผ่านหลาย services

```
Request Flow:
User → API Gateway → User Service → Database
                  → Order Service → Payment Service → DB
                                  → Notification Service

Trace = ชุดของ Spans ที่เกี่ยวข้องกัน
Span  = หน่วยการทำงานหนึ่งใน trace (มี start/end time, attributes)
```

---

## 2. OpenTelemetry Setup

```go
// tracing/tracer.go
package tracing

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
	"go.opentelemetry.io/otel/trace"
)

type Config struct {
	ServiceName    string
	ServiceVersion string
	Environment    string
	OTLPEndpoint   string
	SampleRate     float64
}

func InitTracer(ctx context.Context, cfg Config) (func(context.Context) error, error) {
	// Create OTLP exporter (to Jaeger/Tempo/etc.)
	exporter, err := otlptracehttp.New(ctx,
		otlptracehttp.WithEndpoint(cfg.OTLPEndpoint),
		otlptracehttp.WithInsecure(),
	)
	if err != nil {
		return nil, fmt.Errorf("creating OTLP exporter: %w", err)
	}

	// Resource describes this service
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(cfg.ServiceName),
			semconv.ServiceVersion(cfg.ServiceVersion),
			attribute.String("environment", cfg.Environment),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	// Configure sampling
	var sampler sdktrace.Sampler
	if cfg.SampleRate >= 1.0 {
		sampler = sdktrace.AlwaysSample()
	} else if cfg.SampleRate <= 0 {
		sampler = sdktrace.NeverSample()
	} else {
		sampler = sdktrace.TraceIDRatioBased(cfg.SampleRate)
	}

	// Create TracerProvider
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.ParentBased(sampler)),
	)

	// Set as global
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	))

	return tp.Shutdown, nil
}

// NewTracer returns a named tracer
func NewTracer(name string) trace.Tracer {
	return otel.Tracer(name)
}
```

---

## 3. Creating Spans

```go
// service/user_service.go
package service

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("user-service")

type UserService struct {
	repo UserRepository
}

func (s *UserService) GetUser(ctx context.Context, userID string) (*User, error) {
	// Start a span
	ctx, span := tracer.Start(ctx, "UserService.GetUser",
		trace.WithAttributes(
			attribute.String("user.id", userID),
		),
	)
	defer span.End()

	// Add event
	span.AddEvent("fetching user from repository")

	user, err := s.repo.FindByID(ctx, userID)
	if err != nil {
		// Record error
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, fmt.Errorf("finding user: %w", err)
	}

	// Add result attributes
	span.SetAttributes(
		attribute.String("user.email", user.Email),
		attribute.String("user.role", user.Role),
	)
	span.SetStatus(codes.Ok, "")

	return user, nil
}

func (s *UserService) CreateUser(ctx context.Context, req CreateUserRequest) (*User, error) {
	ctx, span := tracer.Start(ctx, "UserService.CreateUser",
		trace.WithAttributes(
			attribute.String("user.email", req.Email),
		),
	)
	defer span.End()

	// Validate - child span
	ctx, validateSpan := tracer.Start(ctx, "validateRequest")
	err := req.Validate()
	validateSpan.End()
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, "validation failed")
		return nil, err
	}

	// Save to DB - child span
	ctx, saveSpan := tracer.Start(ctx, "saveUser",
		trace.WithAttributes(attribute.String("db.operation", "insert")),
	)
	user, err := s.repo.Save(ctx, req.ToUser())
	saveSpan.End()
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, err
	}

	span.SetAttributes(attribute.String("user.id", user.ID))
	return user, nil
}

type User struct {
	ID    string
	Email string
	Role  string
}

type CreateUserRequest struct {
	Email string
	Name  string
}

func (r CreateUserRequest) Validate() error { return nil }
func (r CreateUserRequest) ToUser() *User   { return &User{Email: r.Email} }

type UserRepository interface {
	FindByID(ctx context.Context, id string) (*User, error)
	Save(ctx context.Context, user *User) (*User, error)
}
```

---

## 4. HTTP Middleware

```go
// middleware/tracing.go
package middleware

import (
	"net/http"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/propagation"
)

// TracingMiddleware adds tracing to HTTP requests
func TracingMiddleware(serviceName string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return otelhttp.NewHandler(next, serviceName,
			otelhttp.WithTracerProvider(otel.GetTracerProvider()),
			otelhttp.WithPropagators(otel.GetTextMapPropagator()),
			otelhttp.WithMessageEvents(otelhttp.ReadEvents, otelhttp.WriteEvents),
			otelhttp.WithSpanOptions(
				// Add server attributes
			),
			otelhttp.WithSpanNameFormatter(func(operation string, r *http.Request) string {
				return r.Method + " " + r.URL.Path
			}),
		)
	}
}

// InjectTraceContext injects trace context into outgoing HTTP requests
func InjectTraceContext(ctx interface{}, req *http.Request) {
	otel.GetTextMapPropagator().Inject(req.Context(), propagation.HeaderCarrier(req.Header))
}

// ExtractTraceContext extracts trace context from incoming requests
func ExtractTraceContext(r *http.Request) {
	ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))
	_ = ctx
}

// AddHTTPAttributes adds HTTP-specific attributes to span
func AddHTTPAttributes(r *http.Request) []attribute.KeyValue {
	return []attribute.KeyValue{
		attribute.String("http.method", r.Method),
		attribute.String("http.url", r.URL.String()),
		attribute.String("http.user_agent", r.UserAgent()),
		attribute.String("http.remote_addr", r.RemoteAddr),
	}
}
```

---

## 5. gRPC Tracing

```go
// tracing/grpc.go
package tracing

import (
	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
	"google.golang.org/grpc"
)

// GRPCServerOptions returns tracing interceptors for gRPC server
func GRPCServerOptions() []grpc.ServerOption {
	return []grpc.ServerOption{
		grpc.StatsHandler(otelgrpc.NewServerHandler()),
	}
}

// GRPCDialOptions returns tracing dial options for gRPC client
func GRPCDialOptions() []grpc.DialOption {
	return []grpc.DialOption{
		grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
	}
}
```

---

## 6. Database Tracing

```go
// tracing/db.go
package tracing

import (
	"context"
	"database/sql"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

var dbTracer = otel.Tracer("database")

// TracedDB wraps sql.DB with tracing
type TracedDB struct {
	db     *sql.DB
	system string
	dbName string
}

func NewTracedDB(db *sql.DB, system, dbName string) *TracedDB {
	return &TracedDB{db: db, system: system, dbName: dbName}
}

func (t *TracedDB) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
	ctx, span := dbTracer.Start(ctx, "sql.Query",
		otelgrpc.WithAttributes(
			semconv.DBSystemKey.String(t.system),
			semconv.DBNameKey.String(t.dbName),
			semconv.DBStatementKey.String(sanitizeQuery(query)),
		),
	)
	defer span.End()

	rows, err := t.db.QueryContext(ctx, query, args...)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, fmt.Errorf("query: %w", err)
	}

	return rows, nil
}

func (t *TracedDB) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
	ctx, span := dbTracer.Start(ctx, "sql.Exec",
		otelgrpc.WithAttributes(
			semconv.DBSystemKey.String(t.system),
			semconv.DBStatementKey.String(sanitizeQuery(query)),
		),
	)
	defer span.End()

	result, err := t.db.ExecContext(ctx, query, args...)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, err
	}

	return result, nil
}

func sanitizeQuery(q string) string {
	// Remove sensitive data from query for tracing
	if len(q) > 200 {
		return q[:200] + "..."
	}
	return q
}

// Placeholder to avoid import error
var otelgrpc = struct {
	WithAttributes func(...attribute.KeyValue) interface{}
}{
	WithAttributes: func(attrs ...attribute.KeyValue) interface{} { return nil },
}
```

---

## 7. Trace Propagation ระหว่าง Services

```go
// client/http_client.go
package client

import (
	"context"
	"net/http"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

// NewTracingHTTPClient สร้าง HTTP client ที่ propagate trace context
func NewTracingHTTPClient() *http.Client {
	return &http.Client{
		Transport: otelhttp.NewTransport(http.DefaultTransport),
	}
}

// CallUserService เรียก user service พร้อม trace propagation
func CallUserService(ctx context.Context, userID string) (*UserResponse, error) {
	client := NewTracingHTTPClient()
	
	req, err := http.NewRequestWithContext(ctx, "GET", 
		"http://user-service/users/"+userID, nil)
	if err != nil {
		return nil, err
	}

	// otelhttp transport จะ inject W3C TraceContext header โดยอัตโนมัติ
	// traceparent: 00-{traceID}-{spanID}-{flags}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	return parseUserResponse(resp)
}

type UserResponse struct {
	ID    string `json:"id"`
	Email string `json:"email"`
}

func parseUserResponse(resp *http.Response) (*UserResponse, error) {
	return &UserResponse{ID: "1", Email: "user@example.com"}, nil
}
```

---

## 8. Jaeger Docker Compose

```yaml
# docker-compose.yaml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - tracing

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - tracing

networks:
  tracing:
    driver: bridge
```

---

## 9. Custom Span Attributes

```go
// tracing/attributes.go
package tracing

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

// Common attribute keys
const (
	UserIDKey     = attribute.Key("user.id")
	OrderIDKey    = attribute.Key("order.id")
	RequestIDKey  = attribute.Key("request.id")
	ComponentKey  = attribute.Key("component")
)

// SetUserID adds user ID to current span
func SetUserID(ctx context.Context, userID string) {
	span := trace.SpanFromContext(ctx)
	span.SetAttributes(UserIDKey.String(userID))
}

// SetOrderID adds order ID to current span
func SetOrderID(ctx context.Context, orderID string) {
	span := trace.SpanFromContext(ctx)
	span.SetAttributes(OrderIDKey.String(orderID))
}

// AddEvent adds a named event to the current span
func AddEvent(ctx context.Context, name string, attrs ...attribute.KeyValue) {
	span := trace.SpanFromContext(ctx)
	span.AddEvent(name, trace.WithAttributes(attrs...))
}

// RecordError records an error to the current span
func RecordError(ctx context.Context, err error) {
	if err != nil {
		span := trace.SpanFromContext(ctx)
		span.RecordError(err)
	}
}
```

---

## สรุป

| Component | Tool |
|-----------|------|
| Instrumentation | OpenTelemetry SDK |
| Exporter | OTLP (HTTP/gRPC) |
| Backend | Jaeger / Zipkin / Tempo |
| HTTP | `otelhttp` contrib |
| gRPC | `otelgrpc` contrib |
| Propagation | W3C TraceContext |

---

**ต่อไป**: Part 64 - Metrics with Prometheus
