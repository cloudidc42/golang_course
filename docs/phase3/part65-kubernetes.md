# Part 65: Kubernetes Deployment

## เป้าหมายการเรียนรู้
- Kubernetes overview
- Deploy Go app บน Kubernetes
- Service และ Ingress
- ConfigMaps และ Secrets
- Health Probes
- HPA (Horizontal Pod Autoscaler)
- Rolling Updates
- Helm charts
- ตัวอย่างโค้ด 20+ ตัวอย่าง

---

## 1. Kubernetes Overview

```
Kubernetes Components:
- Pod         = กลุ่ม containers (unit ที่เล็กที่สุด)
- Deployment  = จัดการ Pods (replicas, rolling update)
- Service     = load balancing ระหว่าง Pods
- Ingress     = HTTP routing จากภายนอก
- ConfigMap   = configuration data
- Secret      = sensitive data
- HPA         = auto-scaling based on metrics
- Namespace   = logical isolation
```

---

## 2. Dockerfile สำหรับ Production

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Download dependencies first (cache layer)
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s -X main.Version=$(git describe --tags --always)" \
    -o /bin/app ./cmd/server

# Final stage - minimal image
FROM gcr.io/distroless/static:nonroot

COPY --from=builder /bin/app /app

USER nonroot:nonroot
EXPOSE 8080 9090

ENTRYPOINT ["/app"]
```

---

## 3. Deployment YAML

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-app
  namespace: production
  labels:
    app: go-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: go-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # Zero-downtime deployment
  template:
    metadata:
      labels:
        app: go-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: go-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        fsGroup: 65532
      containers:
        - name: go-app
          image: registry.example.com/go-app:1.0.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          env:
            - name: PORT
              value: "8080"
            - name: METRICS_PORT
              value: "9090"
            - name: ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: go-app-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: go-app-secrets
                  key: db-password
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
                  name: go-app-config
                  key: app-name
          envFrom:
            - configMapRef:
                name: go-app-config
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /healthz/startup
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: go-app
```

---

## 4. Service YAML

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: go-app
  namespace: production
  labels:
    app: go-app
spec:
  selector:
    app: go-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: 9090
  type: ClusterIP
```

---

## 5. Ingress YAML

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-app
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: go-app
                port:
                  name: http
```

---

## 6. ConfigMap และ Secret

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: go-app-config
  namespace: production
data:
  app-name: "go-app"
  log-level: "info"
  max-connections: "100"
  feature-flags: |
    NEW_CHECKOUT=true
    EXPERIMENTAL_SEARCH=false
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: go-app-secrets
  namespace: production
type: Opaque
stringData:          # stringData = base64 อัตโนมัติ
  db-host: "postgres.database.svc.cluster.local"
  db-password: "super-secret-password"
  jwt-secret: "my-jwt-signing-key"
  redis-url: "redis://redis.cache.svc.cluster.local:6379"
```

---

## 7. Health Probes ใน Go

```go
// health/probes.go
package health

import (
	"context"
	"encoding/json"
	"net/http"
	"sync/atomic"
	"time"
)

type Checker interface {
	Name() string
	Check(ctx context.Context) error
}

type Server struct {
	checks  []Checker
	ready   atomic.Bool
	started atomic.Bool
}

func NewServer(checks ...Checker) *Server {
	s := &Server{checks: checks}
	return s
}

func (s *Server) SetReady(ready bool)   { s.ready.Store(ready) }
func (s *Server) SetStarted(ok bool)    { s.started.Store(ok) }

// /healthz/live - liveness: is the process running?
func (s *Server) LiveHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "alive"})
}

// /healthz/ready - readiness: can it serve traffic?
func (s *Server) ReadyHandler(w http.ResponseWriter, r *http.Request) {
	if !s.ready.Load() {
		http.Error(w, `{"status":"not ready"}`, http.StatusServiceUnavailable)
		return
	}

	ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer cancel()

	results := make(map[string]string)
	allOk := true

	for _, check := range s.checks {
		if err := check.Check(ctx); err != nil {
			results[check.Name()] = "unhealthy: " + err.Error()
			allOk = false
		} else {
			results[check.Name()] = "healthy"
		}
	}

	status := http.StatusOK
	if !allOk {
		status = http.StatusServiceUnavailable
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status": map[bool]string{true: "ready", false: "not ready"}[allOk],
		"checks": results,
	})
}

// /healthz/startup - startup: has the app finished initializing?
func (s *Server) StartupHandler(w http.ResponseWriter, r *http.Request) {
	if !s.started.Load() {
		http.Error(w, `{"status":"starting"}`, http.StatusServiceUnavailable)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "started"})
}

// Database health check
type DBChecker struct {
	db interface{ PingContext(context.Context) error }
}

func (c *DBChecker) Name() string { return "database" }
func (c *DBChecker) Check(ctx context.Context) error {
	return c.db.PingContext(ctx)
}
```

---

## 8. HPA (Horizontal Pod Autoscaler)

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: go-app
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: go-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

---

## 9. Graceful Shutdown

```go
// cmd/server/main.go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// Build app
	app := buildApp()

	srv := &http.Server{
		Addr:         ":" + getEnv("PORT", "8080"),
		Handler:      app.Handler(),
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start server
	go func() {
		log.Printf("Starting server on %s", srv.Addr)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// Signal handling for graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Println("Shutting down gracefully...")

	// Mark as not ready (Kubernetes will stop routing traffic)
	app.Health.SetReady(false)
	
	// Give time for Kubernetes to detect not-ready
	time.Sleep(5 * time.Second)

	// Shutdown with timeout
	ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("Forced shutdown: %v", err)
	}

	// Close other resources
	app.DB.Close()
	app.Cache.Close()

	log.Println("Server stopped")
}

func getEnv(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

type App struct {
	Health  *HealthServer
	DB      interface{ Close() error }
	Cache   interface{ Close() error }
}

type HealthServer struct{}

func (h *HealthServer) SetReady(r bool) {}

func (a *App) Handler() http.Handler {
	return http.DefaultServeMux
}

func buildApp() *App {
	return &App{
		Health: &HealthServer{},
	}
}
```

---

## 10. Helm Chart

```yaml
# helm/go-app/Chart.yaml
apiVersion: v2
name: go-app
description: Go Application Helm Chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# helm/go-app/values.yaml
replicaCount: 3

image:
  repository: registry.example.com/go-app
  pullPolicy: Always
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.example.com
  tls:
    enabled: true
    secretName: api-tls

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 70

config:
  logLevel: info
  maxConnections: "100"

secrets:
  dbHost: ""        # set via --set or external secrets
  dbPassword: ""
  jwtSecret: ""

probes:
  liveness:
    path: /healthz/live
    initialDelaySeconds: 10
    periodSeconds: 10
  readiness:
    path: /healthz/ready
    initialDelaySeconds: 5
    periodSeconds: 5
```

```yaml
# helm/go-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "go-app.fullname" . }}
  labels:
    {{- include "go-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "go-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "go-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
```

---

## 11. Namespace และ RBAC

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
---
# k8s/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: go-app
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: go-app
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: go-app
  namespace: production
subjects:
  - kind: ServiceAccount
    name: go-app
    namespace: production
roleRef:
  kind: Role
  name: go-app
  apiGroup: rbac.authorization.k8s.io
```

---

## 12. PodDisruptionBudget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: go-app
  namespace: production
spec:
  minAvailable: 2   # หรือ maxUnavailable: 1
  selector:
    matchLabels:
      app: go-app
```

---

## 13. Deploy Commands

```bash
# Build and push image
docker build -t registry.example.com/go-app:1.0.0 .
docker push registry.example.com/go-app:1.0.0

# Apply Kubernetes manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# Helm deploy
helm upgrade --install go-app ./helm/go-app \
  --namespace production \
  --set image.tag=1.0.0 \
  --set secrets.dbPassword=secret \
  --values ./helm/go-app/values-prod.yaml

# Check rollout
kubectl rollout status deployment/go-app -n production

# Rollback
kubectl rollout undo deployment/go-app -n production

# Scale manually
kubectl scale deployment/go-app --replicas=5 -n production

# View logs
kubectl logs -l app=go-app -n production --tail=100 -f

# Port forward for debugging
kubectl port-forward svc/go-app 8080:80 -n production
```

---

## สรุป

| Component | Purpose |
|-----------|---------|
| Deployment | จัดการ Pod lifecycle |
| Service | Load balancing ภายใน cluster |
| Ingress | HTTP routing จากภายนอก |
| ConfigMap | Non-sensitive config |
| Secret | Sensitive data |
| HPA | Auto-scaling |
| PDB | Availability during disruptions |
| Helm | Package manager สำหรับ Kubernetes |

---

**จบ Phase 3** - Microservices & Cloud Native Development

**ต่อไป Phase 4**: Performance Optimization, Security, Testing Advanced
