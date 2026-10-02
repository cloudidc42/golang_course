# Part 64: Metrics with Prometheus

## เป้าหมายการเรียนรู้
- Metric types (Counter, Gauge, Histogram, Summary)
- Prometheus client library
- Custom metrics
- Grafana dashboards
- Alert rules
- SLIs/SLOs/SLAs
- RED method
- ตัวอย่างโค้ด 20+ ตัวอย่าง

---

## 1. Metric Types

```
Counter   - เพิ่มขึ้นเรื่อยๆ (requests_total, errors_total)
Gauge     - ขึ้น/ลงได้ (active_connections, memory_usage)
Histogram - distribution ของ values (request_duration)
Summary   - quantiles ของ values (response_time_p99)
```

---

## 2. Basic Metrics Setup

```go
// metrics/metrics.go
package metrics

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// HTTP metrics
var (
	HTTPRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "app",
			Subsystem: "http",
			Name:      "requests_total",
			Help:      "Total number of HTTP requests",
		},
		[]string{"method", "path", "status_code"},
	)

	HTTPRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "app",
			Subsystem: "http",
			Name:      "request_duration_seconds",
			Help:      "HTTP request duration in seconds",
			Buckets:   []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
		},
		[]string{"method", "path"},
	)

	HTTPRequestSize = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "app",
			Subsystem: "http",
			Name:      "request_size_bytes",
			Help:      "HTTP request size in bytes",
			Buckets:   prometheus.ExponentialBuckets(100, 10, 8),
		},
		[]string{"method", "path"},
	)

	HTTPActiveRequests = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "app",
			Subsystem: "http",
			Name:      "active_requests",
			Help:      "Number of active HTTP requests",
		},
	)
)

// Database metrics
var (
	DBQueryDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "app",
			Subsystem: "db",
			Name:      "query_duration_seconds",
			Help:      "Database query duration",
			Buckets:   []float64{.001, .005, .01, .05, .1, .5, 1},
		},
		[]string{"operation", "table"},
	)

	DBConnectionsActive = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "app",
			Subsystem: "db",
			Name:      "connections_active",
			Help:      "Active database connections",
		},
	)

	DBErrors = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "app",
			Subsystem: "db",
			Name:      "errors_total",
			Help:      "Total database errors",
		},
		[]string{"operation", "error_type"},
	)
)

// Business metrics
var (
	OrdersCreated = promauto.NewCounter(
		prometheus.CounterOpts{
			Namespace: "app",
			Subsystem: "business",
			Name:      "orders_created_total",
			Help:      "Total orders created",
		},
	)

	OrderValue = promauto.NewHistogram(
		prometheus.HistogramOpts{
			Namespace: "app",
			Subsystem: "business",
			Name:      "order_value_baht",
			Help:      "Order value in Thai Baht",
			Buckets:   []float64{100, 500, 1000, 5000, 10000, 50000, 100000},
		},
	)

	ActiveUsers = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "app",
			Subsystem: "business",
			Name:      "active_users",
			Help:      "Currently active users",
		},
	)
)
```

---

## 3. HTTP Metrics Middleware

```go
// middleware/metrics.go
package middleware

import (
	"net/http"
	"strconv"
	"time"

	"github.com/example/app/metrics"
)

type responseWriter struct {
	http.ResponseWriter
	statusCode int
	bytes      int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.bytes += n
	return n, err
}

func MetricsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Track active requests
		metrics.HTTPActiveRequests.Inc()
		defer metrics.HTTPActiveRequests.Dec()

		start := time.Now()
		rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

		next.ServeHTTP(rw, r)

		duration := time.Since(start).Seconds()
		status := strconv.Itoa(rw.statusCode)

		// Record metrics
		metrics.HTTPRequestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
		metrics.HTTPRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
		metrics.HTTPRequestSize.WithLabelValues(r.Method, r.URL.Path).Observe(float64(r.ContentLength))
	})
}
```

---

## 4. Metrics Endpoint

```go
// server/metrics_server.go
package server

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func StartMetricsServer(addr string) error {
	mux := http.NewServeMux()
	mux.Handle("/metrics", promhttp.Handler())
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	})

	return http.ListenAndServe(addr, mux)
}
```

---

## 5. Custom Collectors

```go
// metrics/collector.go
package metrics

import (
	"runtime"

	"github.com/prometheus/client_golang/prometheus"
)

// GoRuntimeCollector collects Go runtime metrics
type GoRuntimeCollector struct {
	goroutines prometheus.Gauge
	memAlloc   prometheus.Gauge
	memSys     prometheus.Gauge
	gcRuns     prometheus.Counter
}

func NewGoRuntimeCollector() *GoRuntimeCollector {
	return &GoRuntimeCollector{
		goroutines: prometheus.NewGauge(prometheus.GaugeOpts{
			Name: "go_goroutines_current",
			Help: "Current number of goroutines",
		}),
		memAlloc: prometheus.NewGauge(prometheus.GaugeOpts{
			Name: "go_memory_alloc_bytes",
			Help: "Currently allocated heap memory",
		}),
		memSys: prometheus.NewGauge(prometheus.GaugeOpts{
			Name: "go_memory_sys_bytes",
			Help: "Total memory obtained from OS",
		}),
		gcRuns: prometheus.NewCounter(prometheus.CounterOpts{
			Name: "go_gc_runs_total",
			Help: "Total number of GC runs",
		}),
	}
}

func (c *GoRuntimeCollector) Describe(ch chan<- *prometheus.Desc) {
	c.goroutines.Describe(ch)
	c.memAlloc.Describe(ch)
	c.memSys.Describe(ch)
	c.gcRuns.Describe(ch)
}

func (c *GoRuntimeCollector) Collect(ch chan<- prometheus.Metric) {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)

	c.goroutines.Set(float64(runtime.NumGoroutine()))
	c.memAlloc.Set(float64(ms.Alloc))
	c.memSys.Set(float64(ms.Sys))
	c.gcRuns.Add(float64(ms.NumGC))

	c.goroutines.Collect(ch)
	c.memAlloc.Collect(ch)
	c.memSys.Collect(ch)
	c.gcRuns.Collect(ch)
}
```

---

## 6. SLI/SLO Implementation

```go
// slo/slo.go
package slo

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// SLI = Service Level Indicator (actual measurement)
// SLO = Service Level Objective (target)
// SLA = Service Level Agreement (contractual obligation)

// Availability SLI: successful requests / total requests
var (
	RequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "sli_requests_total",
			Help: "Total requests for SLI calculation",
		},
		[]string{"service", "success"},
	)

	// Latency SLI: requests under threshold / total requests
	LatencyBuckets = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "sli_request_duration_seconds",
			Help:    "Request duration for SLI calculation",
			Buckets: []float64{0.1, 0.25, 0.5, 1.0, 2.5},
		},
		[]string{"service"},
	)
)

// RecordRequest บันทึก request สำหรับ SLI calculation
func RecordRequest(service string, success bool, durationSecs float64) {
	successStr := "true"
	if !success {
		successStr = "false"
	}
	RequestsTotal.WithLabelValues(service, successStr).Inc()
	LatencyBuckets.WithLabelValues(service).Observe(durationSecs)
}

// SLO targets (expressed as Prometheus recording rules)
// - Availability: 99.9% of requests succeed
// - Latency P99: < 500ms for 99% of requests
// - Error rate: < 0.1% of requests
```

---

## 7. Prometheus Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: 'go-app'
    static_configs:
      - targets: ['app:9090']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

---

## 8. Alert Rules

```yaml
# alerts/app.yml
groups:
  - name: app_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(app_http_requests_total{status_code=~"5.."}[5m])
          /
          rate(app_http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(app_http_request_duration_seconds_bucket[5m])
          ) > 2.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency"
          description: "P99 latency is {{ $value }}s"

      # Service down
      - alert: ServiceDown
        expr: up{job="go-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"

      # High memory
      - alert: HighMemoryUsage
        expr: go_memory_alloc_bytes > 500e6
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage: {{ $value | humanize1024 }}B"
```

---

## 9. Grafana Dashboard JSON (ตัวอย่าง)

```json
{
  "dashboard": {
    "title": "Go Application Metrics",
    "panels": [
      {
        "title": "Request Rate (RED)",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(app_http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}} {{status_code}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(app_http_requests_total{status_code=~'5..'}[5m]) / rate(app_http_requests_total[5m]) * 100",
            "legendFormat": "Error %"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(app_http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P99"
          },
          {
            "expr": "histogram_quantile(0.95, rate(app_http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.50, rate(app_http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P50"
          }
        ]
      }
    ]
  }
}
```

---

## 10. RED Method

```
RED = Rate, Errors, Duration
ใช้สำหรับ services/APIs

Rate     = requests/second
Errors   = failed requests/second
Duration = distribution of request latency
```

```go
// metrics/red.go
package metrics

import (
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// REDMetrics implements the RED method
type REDMetrics struct {
	rate     *prometheus.CounterVec
	errors   *prometheus.CounterVec
	duration *prometheus.HistogramVec
}

func NewREDMetrics(service string) *REDMetrics {
	labels := []string{"method", "endpoint"}
	return &REDMetrics{
		rate: promauto.NewCounterVec(prometheus.CounterOpts{
			Name: service + "_requests_total",
			Help: "Rate: total requests",
		}, labels),
		errors: promauto.NewCounterVec(prometheus.CounterOpts{
			Name: service + "_errors_total",
			Help: "Errors: total failed requests",
		}, labels),
		duration: promauto.NewHistogramVec(prometheus.HistogramOpts{
			Name:    service + "_duration_seconds",
			Help:    "Duration: request latency distribution",
			Buckets: prometheus.DefBuckets,
		}, labels),
	}
}

func (m *REDMetrics) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rw := &responseWriter{ResponseWriter: w, statusCode: 200}

		next.ServeHTTP(rw, r)

		labels := prometheus.Labels{"method": r.Method, "endpoint": r.URL.Path}
		m.rate.With(labels).Inc()
		m.duration.With(labels).Observe(time.Since(start).Seconds())
		if rw.statusCode >= 500 {
			m.errors.With(labels).Inc()
		}
	})
}
```

---

## สรุป

| Metric Type | Use Case | Example |
|-------------|----------|---------|
| Counter | เพิ่มขึ้นเรื่อยๆ | requests_total |
| Gauge | ค่า snapshot | active_connections |
| Histogram | distribution | request_duration |
| Summary | quantiles | response_time_p99 |

**RED Method**: Rate + Errors + Duration = visibility ที่ดีสำหรับ services

---

**ต่อไป**: Part 65 - Kubernetes Deployment
