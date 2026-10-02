# Part 45: Docker สำหรับ Go Applications

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เขียน Dockerfile สำหรับ Go application
- ใช้ Multi-stage builds เพื่อลดขนาด image
- เลือกระหว่าง Alpine vs scratch base image
- ใช้ Docker Compose
- จัดการ environment variables อย่างปลอดภัย
- ตั้งค่า health checks
- Optimize Docker images

---

## 1. Dockerfile พื้นฐานสำหรับ Go

```dockerfile
# ตัวอย่าง 1: Basic Dockerfile

FROM golang:1.22

WORKDIR /app

# Copy go.mod and go.sum first for caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build
RUN go build -o server ./cmd/server

# Run
CMD ["./server"]
```

**ปัญหา:** image ขนาดใหญ่ ~1GB เพราะมี Go compiler

---

## 2. Multi-stage Build

```dockerfile
# ตัวอย่าง 2: Multi-stage Dockerfile

# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Install build tools
RUN apk add --no-cache gcc musl-dev

# Copy go.mod and go.sum
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-w -s -X main.version=${VERSION}" \
    -o server \
    ./cmd/server

# Stage 2: Run
FROM alpine:3.19

# Add ca-certificates for HTTPS
RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/server .

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser
USER appuser

EXPOSE 8080

CMD ["./server"]
```

ลดขนาดจาก ~1GB เหลือ ~15MB

---

## 3. Scratch Image (ขนาดเล็กที่สุด)

```dockerfile
# ตัวอย่าง 3: Using scratch base image

# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Must disable CGO for scratch
RUN CGO_ENABLED=0 GOOS=linux \
    go build -ldflags="-w -s" -o server ./cmd/server

# Stage 2: Certificates (for HTTPS)
FROM alpine:3.19 AS certs
RUN apk --no-cache add ca-certificates

# Stage 3: Scratch
FROM scratch

# Copy certificates
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy timezone data
COPY --from=builder /usr/local/go/lib/time/zoneinfo.zip /

# Copy binary
COPY --from=builder /app/server /server

# Set timezone env
ENV ZONEINFO=/zoneinfo.zip

EXPOSE 8080

ENTRYPOINT ["/server"]
```

ขนาด image ~5-10MB

---

## 4. เปรียบเทียบ Base Images

| Base Image | ขนาด | Shell | Package Manager | Use Case |
|------------|------|-------|----------------|----------|
| `golang:1.22` | ~800MB | bash | apt | Development |
| `golang:1.22-alpine` | ~300MB | sh | apk | Build stage |
| `alpine:3.19` | ~7MB | sh | apk | Small production |
| `distroless` | ~3MB | none | none | Security-focused |
| `scratch` | 0 | none | none | Minimal |

```dockerfile
# ตัวอย่าง 4: Distroless image (Google)

FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

# Distroless - no shell, minimal attack surface
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

## 5. Build Arguments และ Labels

```dockerfile
# ตัวอย่าง 5: Build args and labels

FROM golang:1.22-alpine AS builder

# Build arguments
ARG VERSION=dev
ARG COMMIT=unknown
ARG BUILD_DATE=unknown

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build \
    -ldflags="-w -s \
    -X main.version=${VERSION} \
    -X main.commit=${COMMIT} \
    -X main.buildDate=${BUILD_DATE}" \
    -o server ./cmd/server

FROM alpine:3.19

RUN apk --no-cache add ca-certificates

# OCI labels (standard)
LABEL org.opencontainers.image.title="My Go App" \
      org.opencontainers.image.description="Go REST API" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.revision="${COMMIT}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/org/repo"

WORKDIR /app
COPY --from=builder /app/server .

RUN addgroup -g 1001 app && adduser -u 1001 -G app -D app
USER app

EXPOSE 8080
CMD ["./server"]
```

```bash
# Build with args
docker build \
  --build-arg VERSION=1.2.3 \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t myapp:1.2.3 .
```

---

## 6. Health Checks

```dockerfile
# ตัวอย่าง 6: Health check in Dockerfile

FROM alpine:3.19

RUN apk --no-cache add ca-certificates wget

WORKDIR /app
COPY --from=builder /app/server .

# Health check using wget
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=5s \
            --retries=3 \
            CMD wget -qO- http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["./server"]
```

```go
// ตัวอย่าง 7: Health check endpoint in Go

package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"runtime"
	"time"
)

type HealthStatus struct {
	Status    string            `json:"status"`
	Timestamp time.Time         `json:"timestamp"`
	Version   string            `json:"version"`
	Checks    map[string]string `json:"checks"`
	Uptime    string            `json:"uptime"`
}

var startTime = time.Now()

func healthHandler(w http.ResponseWriter, r *http.Request) {
	checks := map[string]string{
		"memory": "ok",
		"goroutines": fmt.Sprintf("%d", runtime.NumGoroutine()),
	}
	
	// Check database connection
	// if err := db.PingContext(r.Context()); err != nil {
	//     checks["database"] = "failed"
	// } else {
	//     checks["database"] = "ok"
	// }
	
	status := "healthy"
	httpStatus := http.StatusOK
	
	// Mark unhealthy if any check fails
	for _, v := range checks {
		if v == "failed" {
			status = "unhealthy"
			httpStatus = http.StatusServiceUnavailable
			break
		}
	}
	
	health := HealthStatus{
		Status:    status,
		Timestamp: time.Now(),
		Version:   "1.0.0",
		Checks:    checks,
		Uptime:    time.Since(startTime).String(),
	}
	
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(httpStatus)
	json.NewEncoder(w).Encode(health)
}

func main() {
	http.HandleFunc("/health", healthHandler)
	http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
		// Readiness check - is the app ready to receive traffic?
		w.WriteHeader(http.StatusOK)
		fmt.Fprint(w, "ready")
	})
	
	fmt.Println("Server running on :8080")
	// http.ListenAndServe(":8080", nil)
}
```

---

## 7. Docker Compose

```yaml
# ตัวอย่าง 8: docker-compose.yml for Go + PostgreSQL + Redis

version: '3.8'

services:
  # Go Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        VERSION: ${VERSION:-dev}
    image: myapp:${VERSION:-dev}
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=${DB_NAME:-myapp}
      - DB_USER=${DB_USER:-postgres}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    ports:
      - "5432:5432"  # Remove in production

  # Redis
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-secret}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-secret}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    ports:
      - "6379:6379"  # Remove in production

  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    container_name: myapp-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
```

---

## 8. Environment Variables การจัดการ

```go
// ตัวอย่าง 9: Config from environment variables

package config

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

type Config struct {
	App      AppConfig
	Database DatabaseConfig
	Redis    RedisConfig
	JWT      JWTConfig
}

type AppConfig struct {
	Env     string
	Port    int
	Debug   bool
	Version string
}

type DatabaseConfig struct {
	Host     string
	Port     int
	Name     string
	User     string
	Password string
	SSLMode  string
	MaxConns int
	MaxIdle  int
}

type RedisConfig struct {
	URL      string
	Password string
	DB       int
}

type JWTConfig struct {
	Secret     string
	Expiration time.Duration
}

func Load() (*Config, error) {
	cfg := &Config{}
	
	// App config
	cfg.App.Env = getEnv("APP_ENV", "development")
	cfg.App.Version = getEnv("APP_VERSION", "dev")
	
	port, err := getEnvInt("APP_PORT", 8080)
	if err != nil {
		return nil, fmt.Errorf("invalid APP_PORT: %w", err)
	}
	cfg.App.Port = port
	cfg.App.Debug = getEnvBool("APP_DEBUG", false)
	
	// Database config
	cfg.Database.Host = getEnvRequired("DB_HOST")
	cfg.Database.Name = getEnvRequired("DB_NAME")
	cfg.Database.User = getEnvRequired("DB_USER")
	cfg.Database.Password = getEnvRequired("DB_PASSWORD")
	
	dbPort, _ := getEnvInt("DB_PORT", 5432)
	cfg.Database.Port = dbPort
	cfg.Database.SSLMode = getEnv("DB_SSL_MODE", "disable")
	cfg.Database.MaxConns, _ = getEnvInt("DB_MAX_CONNS", 25)
	cfg.Database.MaxIdle, _ = getEnvInt("DB_MAX_IDLE", 5)
	
	// Redis
	cfg.Redis.URL = getEnv("REDIS_URL", "redis://localhost:6379")
	cfg.Redis.Password = getEnv("REDIS_PASSWORD", "")
	cfg.Redis.DB, _ = getEnvInt("REDIS_DB", 0)
	
	// JWT
	cfg.JWT.Secret = getEnvRequired("JWT_SECRET")
	
	expSecs, _ := getEnvInt("JWT_EXPIRATION_SECS", 86400)
	cfg.JWT.Expiration = time.Duration(expSecs) * time.Second
	
	return cfg, nil
}

func (c *DatabaseConfig) DSN() string {
	return fmt.Sprintf(
		"host=%s port=%d dbname=%s user=%s password=%s sslmode=%s",
		c.Host, c.Port, c.Name, c.User, c.Password, c.SSLMode,
	)
}

func getEnv(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

func getEnvRequired(key string) string {
	val := os.Getenv(key)
	if val == "" {
		panic(fmt.Sprintf("required environment variable %s is not set", key))
	}
	return val
}

func getEnvInt(key string, defaultVal int) (int, error) {
	val := os.Getenv(key)
	if val == "" {
		return defaultVal, nil
	}
	return strconv.Atoi(val)
}

func getEnvBool(key string, defaultVal bool) bool {
	val := os.Getenv(key)
	if val == "" {
		return defaultVal
	}
	b, err := strconv.ParseBool(val)
	if err != nil {
		return defaultVal
	}
	return b
}
```

---

## 9. .env File Management

```bash
# ตัวอย่าง 10: .env file structure

# .env (DO NOT COMMIT TO GIT)
APP_ENV=development
APP_PORT=8080
APP_DEBUG=true

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_dev
DB_USER=postgres
DB_PASSWORD=dev_password_123

# Redis
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=dev-secret-change-in-production
JWT_EXPIRATION_SECS=86400
```

```bash
# .env.example (COMMIT TO GIT - template without secrets)
APP_ENV=development
APP_PORT=8080
APP_DEBUG=false

DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=postgres
DB_PASSWORD=

REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

JWT_SECRET=
JWT_EXPIRATION_SECS=86400
```

```gitignore
# .gitignore
.env
.env.local
.env.*.local
*.env
```

---

## 10. Image Optimization

```dockerfile
# ตัวอย่าง 11: Optimized Dockerfile with caching

FROM golang:1.22-alpine AS builder

WORKDIR /app

# Layer 1: System deps (rarely changes)
RUN apk add --no-cache git

# Layer 2: Go modules (changes when go.mod changes)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Layer 3: Source code (changes often)
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -trimpath \
    -o server \
    ./cmd/server

# Final stage
FROM alpine:3.19 AS final

# Install runtime deps
RUN apk --no-cache add ca-certificates tzdata && \
    # Create user
    addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup && \
    # Create app directory
    mkdir -p /app && \
    chown appuser:appgroup /app

WORKDIR /app

# Copy binary
COPY --from=builder --chown=appuser:appgroup /app/server .

# Copy static files if needed
# COPY --from=builder /app/static ./static

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["./server"]
```

---

## 11. Docker Compose Override สำหรับ Development

```yaml
# ตัวอย่าง 12: docker-compose.override.yml (development)

version: '3.8'

services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - go-cache:/go/pkg/mod
    environment:
      - APP_ENV=development
      - APP_DEBUG=true
    command: ["air", "-c", ".air.toml"]

volumes:
  go-cache:
```

```dockerfile
# ตัวอย่าง 13: Dockerfile.dev with hot reload

FROM golang:1.22-alpine

WORKDIR /app

# Install air for hot reload
RUN go install github.com/air-verse/air@latest

# Install tools
RUN apk add --no-cache git make

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["air"]
```

```toml
# ตัวอย่าง 14: .air.toml configuration

root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  args_bin = []
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ./cmd/server"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  kill_delay = "0s"
  log = "build-errors.log"
  send_interrupt = false
  stop_on_error = true

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  time = false

[misc]
  clean_on_exit = false

[screen]
  clear_on_rebuild = false
```

---

## 12. Makefile สำหรับ Docker

```makefile
# ตัวอย่าง 15: Makefile with Docker targets

APP_NAME    := myapp
VERSION     ?= $(shell git describe --tags --always --dirty)
COMMIT      := $(shell git rev-parse --short HEAD)
BUILD_DATE  := $(shell date -u +"%Y-%m-%dT%H:%M:%SZ")
REGISTRY    := ghcr.io/myorg
IMAGE       := $(REGISTRY)/$(APP_NAME)

.PHONY: build push run stop logs clean

## Docker targets
docker-build:
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg COMMIT=$(COMMIT) \
		--build-arg BUILD_DATE=$(BUILD_DATE) \
		-t $(IMAGE):$(VERSION) \
		-t $(IMAGE):latest \
		.

docker-push:
	docker push $(IMAGE):$(VERSION)
	docker push $(IMAGE):latest

docker-run:
	docker run --rm -it \
		--name $(APP_NAME) \
		-p 8080:8080 \
		--env-file .env \
		$(IMAGE):$(VERSION)

docker-compose-up:
	docker compose up -d

docker-compose-down:
	docker compose down

docker-compose-logs:
	docker compose logs -f app

docker-compose-ps:
	docker compose ps

docker-clean:
	docker compose down -v
	docker rmi $(IMAGE):$(VERSION) || true

## Size analysis
docker-size:
	docker images $(IMAGE) --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

docker-layers:
	docker history $(IMAGE):$(VERSION)

## Dive analysis (requires dive installed)
docker-dive:
	dive $(IMAGE):$(VERSION)
```

---

## 13. Multi-platform Build

```bash
# ตัวอย่าง 16: Build for multiple platforms

# Setup buildx
docker buildx create --name mybuilder --use
docker buildx inspect --bootstrap

# Build for multiple platforms
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --build-arg VERSION=1.0.0 \
  -t myapp:1.0.0 \
  --push \
  .
```

```dockerfile
# ตัวอย่าง 17: Multi-platform Dockerfile

FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Cross-compile for target platform
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-w -s" -o server ./cmd/server

FROM --platform=$TARGETPLATFORM alpine:3.19

RUN apk --no-cache add ca-certificates

WORKDIR /app
COPY --from=builder /app/server .

RUN addgroup -g 1001 app && adduser -u 1001 -G app -D app
USER app

EXPOSE 8080
CMD ["./server"]
```

---

## 14. Production Nginx Config

```nginx
# ตัวอย่าง 18: nginx.conf for Go app

worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream go_app {
        server app:8080;
        keepalive 32;
    }

    server {
        listen 80;
        server_name example.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000";

        location / {
            proxy_pass http://go_app;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            proxy_read_timeout 90;
            proxy_connect_timeout 90;
        }

        location /health {
            proxy_pass http://go_app;
            access_log off;
        }
    }
}
```

---

## 15. Container Security Best Practices

```dockerfile
# ตัวอย่าง 19: Security-hardened Dockerfile

FROM golang:1.22-alpine AS builder

# Vulnerability scanning with trivy (in CI/CD)
# RUN trivy fs --exit-code 1 --no-progress /app

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download && go mod verify  # Verify checksums

COPY . .

# govulncheck for known vulnerabilities
RUN go install golang.org/x/vuln/cmd/govulncheck@latest && \
    govulncheck ./...

RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-w -s" \
    -o server ./cmd/server

FROM scratch

# Read-only filesystem
# No shell, no package manager, no OS utilities

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server

# Run as non-root (numeric UID/GID)
USER 65534:65534

EXPOSE 8080

ENTRYPOINT ["/server"]
```

---

## 16. Docker Compose สำหรับ Testing

```yaml
# ตัวอย่าง 20: docker-compose.test.yml

version: '3.8'

services:
  # Test database
  test-postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    tmpfs:
      - /var/lib/postgresql/data  # In-memory for speed
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Test redis
  test-redis:
    image: redis:7-alpine
    tmpfs:
      - /data

  # Test runner
  test:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - DB_HOST=test-postgres
      - DB_PORT=5432
      - DB_NAME=testdb
      - DB_USER=testuser
      - DB_PASSWORD=testpass
      - REDIS_URL=redis://test-redis:6379
    depends_on:
      test-postgres:
        condition: service_healthy
      test-redis:
        condition: service_started
    command: ["go", "test", "-v", "-race", "./..."]
```

```dockerfile
# ตัวอย่าง 21: Dockerfile.test

FROM golang:1.22-alpine

WORKDIR /app

RUN apk add --no-cache gcc musl-dev

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["go", "test", "-v", "-race", "-count=1", "./..."]
```

---

## สรุป

ใน Part 45 เราได้เรียนรู้:

1. **Basic Dockerfile**: พื้นฐาน Dockerfile สำหรับ Go
2. **Multi-stage Build**: ลดขนาด image จาก 1GB เหลือ ~15MB
3. **Base Images**: golang, alpine, distroless, scratch
4. **Build Args & Labels**: versioning และ metadata
5. **Health Checks**: ทั้ง Dockerfile HEALTHCHECK และ endpoint
6. **Docker Compose**: Full stack (App + PostgreSQL + Redis + Nginx)
7. **Environment Variables**: Config management ที่ปลอดภัย
8. **Dev Workflow**: Hot reload ด้วย Air
9. **Multi-platform Build**: buildx สำหรับ arm64, amd64
10. **Security**: Non-root user, scratch image, vulnerability scanning

---

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Go Official Docker Images](https://hub.docker.com/_/golang)
- [Docker Best Practices for Go](https://docs.docker.com/language/golang/)
- [Trivy - Container Security Scanner](https://github.com/aquasecurity/trivy)
- [Air - Hot Reload](https://github.com/air-verse/air)
- [Dive - Image Layer Explorer](https://github.com/wagoodman/dive)
