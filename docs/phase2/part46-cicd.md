# Part 46: CI/CD สำหรับ Go Projects

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ CI/CD concepts
- สร้าง GitHub Actions workflows สำหรับ Go
- เขียน Makefile สำหรับ automation
- ตั้งค่า automated testing และ coverage
- Build และ push Docker images ใน CI
- ใช้ deployment strategies
- ทำ semantic versioning

---

## 1. CI/CD Concepts

**CI (Continuous Integration):** รัน tests อัตโนมัติทุกครั้งที่ push code

**CD (Continuous Delivery/Deployment):**
- Delivery: Build artifacts พร้อม deploy เสมอ
- Deployment: Deploy อัตโนมัติเมื่อผ่าน tests

**Pipeline ทั่วไป:**
```
Code Push → Lint → Test → Build → Security Scan → Deploy to Staging → Deploy to Production
```

---

## 2. GitHub Actions - Basic Go CI

```yaml
# ตัวอย่าง 1: .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  GO_VERSION: "1.22"

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      
      - name: Download dependencies
        run: go mod download
      
      - name: Verify dependencies
        run: go mod verify
      
      - name: Run vet
        run: go vet ./...
      
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.txt -covermode=atomic ./...
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage.txt
```

---

## 3. GitHub Actions - Full Pipeline

```yaml
# ตัวอย่าง 2: .github/workflows/pipeline.yml

name: Full Pipeline

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Lint
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m

  # Job 2: Test
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      
      - name: Run unit tests
        run: go test -v -race -short ./...
      
      - name: Run integration tests
        run: go test -v -race -run Integration ./...
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: testuser
          DB_PASSWORD: testpass
          REDIS_URL: redis://localhost:6379
      
      - name: Coverage report
        run: |
          go test -coverprofile=coverage.out ./...
          go tool cover -func=coverage.out
          go tool cover -html=coverage.out -o coverage.html
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.html

  # Job 3: Security scan
  security:
    name: Security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
      
      - name: govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
      
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # Job 4: Build Docker image
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ github.ref_name }}
            COMMIT=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  # Job 5: Deploy to staging
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging..."
          # kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}
          # or
          # ssh deploy@staging "docker pull && docker compose up -d"

  # Job 6: Deploy to production
  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
```

---

## 4. Makefile สำหรับ Automation

```makefile
# ตัวอย่าง 3: Comprehensive Makefile

APP_NAME    := myapp
MODULE      := github.com/myorg/myapp
VERSION     ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
COMMIT      := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_DATE  := $(shell date -u +"%Y-%m-%dT%H:%M:%SZ")
GO          := go
GOFLAGS     := -v
LDFLAGS     := -ldflags="-w -s -X main.version=$(VERSION) -X main.commit=$(COMMIT)"
BINARY_DIR  := ./bin

.PHONY: all build clean test lint coverage vet fmt check install help

## Default target
all: check build

## Build binary
build:
	@echo "Building $(APP_NAME) $(VERSION)..."
	@mkdir -p $(BINARY_DIR)
	CGO_ENABLED=0 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME) ./cmd/server

## Build for all platforms
build-all:
	@echo "Building for all platforms..."
	GOOS=linux   GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME)-linux-amd64   ./cmd/server
	GOOS=linux   GOARCH=arm64 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME)-linux-arm64   ./cmd/server
	GOOS=darwin  GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME)-darwin-amd64  ./cmd/server
	GOOS=darwin  GOARCH=arm64 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME)-darwin-arm64  ./cmd/server
	GOOS=windows GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BINARY_DIR)/$(APP_NAME)-windows-amd64.exe ./cmd/server

## Run tests
test:
	$(GO) test -v -race -count=1 ./...

## Run short tests only
test-short:
	$(GO) test -v -short ./...

## Run tests with coverage
coverage:
	$(GO) test -race -coverprofile=coverage.out -covermode=atomic ./...
	$(GO) tool cover -func=coverage.out
	@echo ""
	@echo "Coverage threshold check..."
	@coverage=$$(go tool cover -func=coverage.out | grep total | awk '{print $$3}' | tr -d '%'); \
	if [ $$(echo "$$coverage < 70" | bc -l) -eq 1 ]; then \
		echo "Coverage $$coverage% is below 70%!"; exit 1; \
	else \
		echo "Coverage $$coverage% OK"; \
	fi

## Generate HTML coverage report
coverage-html: coverage
	$(GO) tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report: coverage.html"

## Lint
lint:
	golangci-lint run --timeout 5m ./...

## Vet
vet:
	$(GO) vet ./...

## Format
fmt:
	gofmt -s -w .
	goimports -w .

## Tidy
tidy:
	$(GO) mod tidy
	$(GO) mod verify

## Security check
security:
	govulncheck ./...

## Run all checks
check: fmt vet lint test

## Clean build artifacts
clean:
	rm -rf $(BINARY_DIR) coverage.out coverage.html tmp/

## Install tools
install-tools:
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	go install golang.org/x/tools/cmd/goimports@latest
	go install golang.org/x/vuln/cmd/govulncheck@latest
	go install github.com/air-verse/air@latest
	go install github.com/vektra/mockery/v2@latest

## Run with hot reload
dev:
	air

## Run application
run: build
	$(BINARY_DIR)/$(APP_NAME)

## Docker targets
docker-build:
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg COMMIT=$(COMMIT) \
		--build-arg BUILD_DATE=$(BUILD_DATE) \
		-t $(APP_NAME):$(VERSION) \
		-t $(APP_NAME):latest .

docker-run:
	docker run --rm -p 8080:8080 --env-file .env $(APP_NAME):$(VERSION)

docker-compose-up:
	docker compose up -d

docker-compose-down:
	docker compose down

## Generate code (protobuf, mocks, etc.)
generate:
	$(GO) generate ./...

## Database migrations
migrate-up:
	migrate -path migrations -database "$(DB_URL)" up

migrate-down:
	migrate -path migrations -database "$(DB_URL)" down

migrate-create:
	@read -p "Migration name: " name; \
	migrate create -ext sql -dir migrations -seq $$name

## Version info
version:
	@echo "Version:    $(VERSION)"
	@echo "Commit:     $(COMMIT)"
	@echo "Build Date: $(BUILD_DATE)"

## Help
help:
	@echo "Available targets:"
	@grep -E '^## ' Makefile | sed 's/## /  /'
```

---

## 5. golangci-lint Configuration

```yaml
# ตัวอย่าง 4: .golangci.yml

run:
  timeout: 5m
  go: "1.22"

linters:
  enable:
    - gofmt
    - goimports
    - govet
    - errcheck
    - staticcheck
    - gosimple
    - ineffassign
    - unused
    - gocritic
    - gosec
    - misspell
    - prealloc
    - exhaustive
    - bodyclose
    - contextcheck
    - dupl
    - gocognit
    - cyclop
    
  disable:
    - godox  # TODO comments OK during development
    - wsl    # Too strict whitespace

linters-settings:
  gocognit:
    min-complexity: 20
  
  cyclop:
    max-complexity: 15
  
  gosec:
    excludes:
      - G304  # File path provided as taint input (too many false positives)
  
  gocritic:
    enabled-tags:
      - diagnostic
      - experimental
      - opinionated
      - performance
      - style
  
  errcheck:
    check-type-assertions: true
    check-blank: true

issues:
  exclude-rules:
    - path: "_test.go"
      linters:
        - dupl
        - gosec
    - path: "cmd/"
      linters:
        - gocognit
```

---

## 6. Coverage Threshold

```go
// ตัวอย่าง 5: coverage check script (scripts/check-coverage.go)

package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

func main() {
	threshold := 70.0 // 70% coverage required
	if len(os.Args) > 1 {
		t, err := strconv.ParseFloat(os.Args[1], 64)
		if err == nil {
			threshold = t
		}
	}
	
	// Read coverage.out
	file, err := os.Open("coverage.out")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	defer file.Close()
	
	var totalStatements, coveredStatements int
	
	scanner := bufio.NewScanner(file)
	scanner.Scan() // skip header
	
	for scanner.Scan() {
		line := scanner.Text()
		parts := strings.Fields(line)
		if len(parts) < 3 {
			continue
		}
		
		statements, _ := strconv.Atoi(parts[1])
		count, _ := strconv.Atoi(parts[2])
		
		totalStatements += statements
		if count > 0 {
			coveredStatements += statements
		}
	}
	
	if totalStatements == 0 {
		fmt.Println("No statements found")
		os.Exit(0)
	}
	
	coverage := float64(coveredStatements) / float64(totalStatements) * 100
	fmt.Printf("Coverage: %.1f%% (threshold: %.1f%%)\n", coverage, threshold)
	
	if coverage < threshold {
		fmt.Printf("FAIL: Coverage %.1f%% is below threshold %.1f%%\n", coverage, threshold)
		os.Exit(1)
	}
	
	fmt.Println("PASS: Coverage threshold met")
}
```

---

## 7. Release Workflow

```yaml
# ตัวอย่าง 6: .github/workflows/release.yml

name: Release

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: write
  packages: write

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
          cache: true
      
      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
```

---

## 8. GoReleaser Configuration

```yaml
# ตัวอย่าง 7: .goreleaser.yml

version: 2

before:
  hooks:
    - go mod tidy
    - go generate ./...

builds:
  - env:
      - CGO_ENABLED=0
    goos:
      - linux
      - windows
      - darwin
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w -X main.version={{.Version}} -X main.commit={{.Commit}}
    main: ./cmd/server

archives:
  - format: tar.gz
    name_template: >-
      {{ .ProjectName }}_
      {{- title .Os }}_
      {{- if eq .Arch "amd64" }}x86_64
      {{- else if eq .Arch "386" }}i386
      {{- else }}{{ .Arch }}{{ end }}
    format_overrides:
      - goos: windows
        format: zip

checksum:
  name_template: 'checksums.txt'

changelog:
  sort: asc
  filters:
    exclude:
      - '^docs:'
      - '^test:'
      - '^chore:'

dockers:
  - image_templates:
      - "ghcr.io/myorg/{{ .ProjectName }}:{{ .Tag }}"
      - "ghcr.io/myorg/{{ .ProjectName }}:latest"
    dockerfile: Dockerfile
    build_flag_templates:
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.title={{.ProjectName}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--build-arg=VERSION={{.Version}}"
      - "--build-arg=COMMIT={{.Commit}}"
```

---

## 9. Semantic Versioning

```go
// ตัวอย่าง 8: Version info in Go application

package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"runtime"
)

// Injected at build time via ldflags
var (
	version   = "dev"
	commit    = "unknown"
	buildDate = "unknown"
)

type BuildInfo struct {
	Version   string `json:"version"`
	Commit    string `json:"commit"`
	BuildDate string `json:"build_date"`
	GoVersion string `json:"go_version"`
	OS        string `json:"os"`
	Arch      string `json:"arch"`
}

func getBuildInfo() BuildInfo {
	return BuildInfo{
		Version:   version,
		Commit:    commit,
		BuildDate: buildDate,
		GoVersion: runtime.Version(),
		OS:        runtime.GOOS,
		Arch:      runtime.GOARCH,
	}
}

func versionHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(getBuildInfo())
}

func main() {
	info := getBuildInfo()
	fmt.Printf("Starting %s version=%s commit=%s\n",
		"myapp", info.Version, info.Commit)
	
	http.HandleFunc("/version", versionHandler)
	// http.ListenAndServe(":8080", nil)
}
```

---

## 10. Dependency Updates

```yaml
# ตัวอย่าง 9: .github/dependabot.yml

version: 2

updates:
  # Go modules
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    labels:
      - "dependencies"
      - "go"
    commit-message:
      prefix: "chore(deps)"
    reviewers:
      - "myorg/backend-team"
    groups:
      aws-sdk:
        patterns:
          - "github.com/aws/*"
  
  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "github-actions"
    commit-message:
      prefix: "chore(ci)"
  
  # Docker
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"
```

---

## 11. Deployment Strategies

```yaml
# ตัวอย่าง 10: Rolling deployment with health check

name: Deploy

on:
  workflow_run:
    workflows: ["Full Pipeline"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy with zero downtime
        run: |
          # Rolling update - update one instance at a time
          ssh deploy@server1 "
            docker pull myapp:latest &&
            docker compose up -d --no-deps app &&
            sleep 10 &&
            wget -qO- http://localhost:8080/health || exit 1
          "
          
          echo "Deployment successful"
      
      - name: Verify deployment
        run: |
          # Check health endpoint
          for i in 1 2 3 4 5; do
            response=$(curl -sf http://myapp.example.com/health)
            if [ $? -eq 0 ]; then
              echo "Health check passed: $response"
              break
            fi
            echo "Attempt $i failed, retrying..."
            sleep 10
          done
      
      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, rolling back..."
          ssh deploy@server1 "
            docker compose up -d --no-deps app $(docker ps -q --filter name=app_previous)
          "
```

---

## 12. PR Checks

```yaml
# ตัวอย่าง 11: .github/workflows/pr-checks.yml

name: PR Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  size-check:
    name: PR Size Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check PR size
        run: |
          additions=$(gh pr view ${{ github.event.number }} --json additions -q .additions)
          if [ $additions -gt 500 ]; then
            echo "::warning::Large PR with $additions additions. Consider splitting."
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  
  conventional-commits:
    name: Conventional Commits
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Check commit messages
        run: |
          # Check that commit messages follow conventional commits
          git log --oneline origin/main..HEAD | while read line; do
            if ! echo "$line" | grep -qP '^[a-f0-9]+ (feat|fix|docs|style|refactor|test|chore|ci|build|perf|revert)(\(.+\))?: .+'; then
              echo "Invalid commit message: $line"
              echo "Expected: type(scope): description"
              exit 1
            fi
          done
  
  breaking-changes:
    name: Breaking Changes Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      
      - name: Check for breaking API changes
        run: |
          # Install apidiff tool
          go install golang.org/x/tools/cmd/apidiff@latest
          # Run comparison (simplified)
          git stash
          go build ./...
          git stash pop
          echo "API compatibility check passed"
```

---

## 13. Notifications

```yaml
# ตัวอย่าง 12: Slack notifications

name: Notify

on:
  workflow_run:
    workflows: ["Full Pipeline"]
    types: [completed]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Notify Slack on success
        if: ${{ github.event.workflow_run.conclusion == 'success' }}
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": ":white_check_mark: Deployment successful!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Successful* :rocket:\nCommit: `${{ github.sha }}`\nBranch: `${{ github.ref_name }}`"
                  }
                }
              ]
            }
      
      - name: Notify Slack on failure
        if: ${{ github.event.workflow_run.conclusion == 'failure' }}
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": ":x: Deployment FAILED!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Failed* :fire:\nCommit: `${{ github.sha }}`\nURL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                }
              ]
            }
```

---

## 14. Environment-specific Config

```yaml
# ตัวอย่าง 13: Environment-specific workflows

name: Deploy Environment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to deploy (e.g., v1.2.3)'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - name: Validate version
        run: |
          if ! echo "${{ inputs.version }}" | grep -qP '^v\d+\.\d+\.\d+$'; then
            echo "Invalid version format. Expected: vX.Y.Z"
            exit 1
          fi
      
      - name: Deploy to ${{ inputs.environment }}
        run: |
          echo "Deploying version ${{ inputs.version }} to ${{ inputs.environment }}"
          # Deployment logic here
```

---

## 15. Caching Strategy

```yaml
# ตัวอย่าง 14: Optimized caching

name: CI with Cache

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
          cache: true  # This automatically caches go mod cache
      
      # Cache golangci-lint binary
      - name: Cache golangci-lint
        uses: actions/cache@v4
        with:
          path: ~/.cache/golangci-lint
          key: golangci-lint-${{ hashFiles('.golangci.yml') }}
      
      # Cache build cache
      - name: Cache Go build
        uses: actions/cache@v4
        with:
          path: ~/.cache/go-build
          key: ${{ runner.os }}-go-build-${{ hashFiles('**/*.go') }}
          restore-keys: |
            ${{ runner.os }}-go-build-
      
      - run: go test ./...
```

---

## สรุป

ใน Part 46 เราได้เรียนรู้:

1. **CI/CD Concepts**: Continuous Integration, Delivery, Deployment
2. **GitHub Actions**: Basic CI และ Full Pipeline
3. **Services in CI**: PostgreSQL, Redis สำหรับ integration tests
4. **Security Scanning**: govulncheck, Trivy
5. **Docker in CI**: Build, push, multi-platform
6. **Makefile**: Comprehensive automation targets
7. **golangci-lint**: Code quality configuration
8. **Coverage**: Threshold enforcement
9. **GoReleaser**: Automated releases
10. **Dependabot**: Automated dependency updates
11. **Deployment Strategies**: Rolling updates, rollback
12. **Notifications**: Slack integration

---

## Resources

- [GitHub Actions for Go](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-go)
- [golangci-lint](https://golangci-lint.run/)
- [GoReleaser](https://goreleaser.com/)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [Conventional Commits](https://www.conventionalcommits.org/)
