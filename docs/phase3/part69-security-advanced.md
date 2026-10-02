# Part 69: Advanced Security

## เป้าหมายการเรียนรู้
- Zero Trust Security Model
- mTLS Implementation
- OAuth 2.0 Flows
- API Key Management
- Secrets Rotation
- HashiCorp Vault Integration
- Security Scanning ด้วย gosec
- Container Security

---

## 1. Zero Trust Security

Zero Trust คือแนวคิด "Never Trust, Always Verify" - ไม่เชื่อใคร แม้แต่ traffic ใน network เดียวกัน

```
Traditional Security:        Zero Trust:
┌─────────────────────┐     ┌─────────────────────┐
│  Trusted Zone       │     │  Verify Every Request│
│  ┌───┐  ┌───┐      │     │  ┌───┐  ┌───┐       │
│  │ A │→ │ B │      │     │  │ A │→ │ B │       │
│  └───┘  └───┘      │     │  └───┘  └───┘       │
│  (trusted inside)   │     │  (verify each call) │
└─────────────────────┘     └─────────────────────┘
```

### Zero Trust Implementation

```go
// zerotrust/middleware.go
package zerotrust

import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "strings"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

type ZeroTrustConfig struct {
    JWTSecret       []byte
    AllowedCerts    []string
    RequiredClaims  map[string]string
    RateLimit       int
}

// Middleware ที่รวมทุก zero trust checks
func ZeroTrustMiddleware(cfg ZeroTrustConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 1. Verify JWT
            claims, err := verifyJWT(r, cfg.JWTSecret)
            if err != nil {
                http.Error(w, "Unauthorized: "+err.Error(), http.StatusUnauthorized)
                return
            }
            
            // 2. Check required claims
            for key, required := range cfg.RequiredClaims {
                if val, ok := claims[key].(string); !ok || val != required {
                    http.Error(w, "Forbidden: insufficient claims", http.StatusForbidden)
                    return
                }
            }
            
            // 3. Verify client certificate (mTLS)
            if r.TLS != nil && len(r.TLS.PeerCertificates) > 0 {
                if err := verifyCert(r.TLS.PeerCertificates[0], cfg.AllowedCerts); err != nil {
                    http.Error(w, "Forbidden: invalid certificate", http.StatusForbidden)
                    return
                }
            }
            
            // 4. Add security context
            ctx := context.WithValue(r.Context(), "claims", claims)
            ctx = context.WithValue(ctx, "verified_at", time.Now())
            
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func verifyJWT(r *http.Request, secret []byte) (jwt.MapClaims, error) {
    authHeader := r.Header.Get("Authorization")
    if authHeader == "" {
        return nil, fmt.Errorf("missing authorization header")
    }
    
    parts := strings.SplitN(authHeader, " ", 2)
    if len(parts) != 2 || parts[0] != "Bearer" {
        return nil, fmt.Errorf("invalid authorization format")
    }
    
    token, err := jwt.Parse(parts[1], func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return secret, nil
    })
    
    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid claims")
    }
    
    return claims, nil
}

func verifyCert(cert *x509.Certificate, allowedCNs []string) error {
    for _, cn := range allowedCNs {
        if cert.Subject.CommonName == cn {
            return nil
        }
    }
    return fmt.Errorf("certificate CN %s not allowed", cert.Subject.CommonName)
}
```

---

## 2. mTLS Implementation

```go
// mtls/server.go
package mtls

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "net/http"
    "os"
)

type MTLSServer struct {
    certFile   string
    keyFile    string
    caCertFile string
    addr       string
}

func NewMTLSServer(addr, certFile, keyFile, caCertFile string) *MTLSServer {
    return &MTLSServer{
        addr:       addr,
        certFile:   certFile,
        keyFile:    keyFile,
        caCertFile: caCertFile,
    }
}

func (s *MTLSServer) ListenAndServeTLS(handler http.Handler) error {
    caCert, err := os.ReadFile(s.caCertFile)
    if err != nil {
        return fmt.Errorf("failed to read CA cert: %w", err)
    }
    
    caCertPool := x509.NewCertPool()
    if !caCertPool.AppendCertsFromPEM(caCert) {
        return fmt.Errorf("failed to parse CA cert")
    }
    
    cert, err := tls.LoadX509KeyPair(s.certFile, s.keyFile)
    if err != nil {
        return fmt.Errorf("failed to load server cert: %w", err)
    }
    
    tlsConfig := &tls.Config{
        ClientAuth:               tls.RequireAndVerifyClientCert,
        ClientCAs:                caCertPool,
        Certificates:             []tls.Certificate{cert},
        MinVersion:               tls.VersionTLS13,
        PreferServerCipherSuites: true,
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP256,
        },
    }
    
    server := &http.Server{
        Addr:      s.addr,
        Handler:   handler,
        TLSConfig: tlsConfig,
    }
    
    return server.ListenAndServeTLS(s.certFile, s.keyFile)
}
```

```go
// mtls/client.go
package mtls

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "net"
    "net/http"
    "os"
    "time"
)

type MTLSClient struct {
    httpClient *http.Client
}

func NewMTLSClient(certFile, keyFile, caCertFile string) (*MTLSClient, error) {
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, fmt.Errorf("failed to load client cert: %w", err)
    }
    
    caCert, err := os.ReadFile(caCertFile)
    if err != nil {
        return nil, fmt.Errorf("failed to read CA cert: %w", err)
    }
    
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)
    
    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
        MinVersion:   tls.VersionTLS13,
    }
    
    transport := &http.Transport{
        TLSClientConfig: tlsConfig,
        DialContext: (&net.Dialer{
            Timeout:   30 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        MaxIdleConns:          100,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   10 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
    }
    
    return &MTLSClient{
        httpClient: &http.Client{
            Transport: transport,
            Timeout:   30 * time.Second,
        },
    }, nil
}

func (c *MTLSClient) Get(url string) (*http.Response, error) {
    return c.httpClient.Get(url)
}
```

### Certificate Generation Script

```bash
#!/bin/bash
# scripts/gen-certs.sh - สร้าง certificates สำหรับ testing

# สร้าง CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
    -subj "/C=TH/O=MyApp/CN=MyApp CA"

# สร้าง Server Certificate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
    -subj "/C=TH/O=MyApp/CN=server"
openssl x509 -req -days 365 -in server.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out server.crt -extensions v3_req \
    -extfile <(echo "[v3_req]
subjectAltName=DNS:localhost,IP:127.0.0.1")

# สร้าง Client Certificate
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
    -subj "/C=TH/O=MyApp/CN=service-a"
openssl x509 -req -days 365 -in client.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out client.crt

echo "Certificates generated!"
```

---

## 3. API Key Management

```go
// apikey/manager.go
package apikey

import (
    "context"
    "crypto/rand"
    "crypto/sha256"
    "database/sql"
    "encoding/hex"
    "fmt"
    "time"
)

type APIKey struct {
    ID          string
    KeyHash     string   // เก็บ hash ไม่ใช่ key จริง
    UserID      string
    Name        string
    Scopes      []string
    RateLimit   int
    ExpiresAt   *time.Time
    LastUsedAt  *time.Time
    CreatedAt   time.Time
}

type APIKeyManager struct {
    db *sql.DB
}

// สร้าง API key ใหม่
func (m *APIKeyManager) Create(ctx context.Context, userID, name string, scopes []string, expiresIn *time.Duration) (string, *APIKey, error) {
    // Generate random key
    rawKey := make([]byte, 32)
    if _, err := rand.Read(rawKey); err != nil {
        return "", nil, fmt.Errorf("failed to generate key: %w", err)
    }
    
    // Format: prefix_base64
    keyString := "sk_" + hex.EncodeToString(rawKey)
    
    // Hash key สำหรับเก็บใน DB
    hash := sha256.Sum256([]byte(keyString))
    keyHash := hex.EncodeToString(hash[:])
    
    var expiresAt *time.Time
    if expiresIn != nil {
        t := time.Now().Add(*expiresIn)
        expiresAt = &t
    }
    
    key := &APIKey{
        ID:        generateID(),
        KeyHash:   keyHash,
        UserID:    userID,
        Name:      name,
        Scopes:    scopes,
        RateLimit: 1000, // default 1000 req/hour
        ExpiresAt: expiresAt,
        CreatedAt: time.Now(),
    }
    
    _, err := m.db.ExecContext(ctx, `
        INSERT INTO api_keys (id, key_hash, user_id, name, scopes, rate_limit, expires_at, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
    `, key.ID, key.KeyHash, key.UserID, key.Name, 
       pq.Array(key.Scopes), key.RateLimit, key.ExpiresAt, key.CreatedAt)
    
    if err != nil {
        return "", nil, fmt.Errorf("failed to save API key: %w", err)
    }
    
    // Return key string เพียงครั้งเดียว (ไม่เก็บใน DB)
    return keyString, key, nil
}

// Verify API key
func (m *APIKeyManager) Verify(ctx context.Context, keyString string) (*APIKey, error) {
    hash := sha256.Sum256([]byte(keyString))
    keyHash := hex.EncodeToString(hash[:])
    
    var key APIKey
    err := m.db.QueryRowContext(ctx, `
        SELECT id, key_hash, user_id, name, scopes, rate_limit, expires_at, last_used_at
        FROM api_keys
        WHERE key_hash = $1 AND revoked_at IS NULL
    `, keyHash).Scan(
        &key.ID, &key.KeyHash, &key.UserID, &key.Name,
        pq.Array(&key.Scopes), &key.RateLimit, &key.ExpiresAt, &key.LastUsedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("invalid API key")
    }
    if err != nil {
        return nil, err
    }
    
    // ตรวจสอบ expiration
    if key.ExpiresAt != nil && time.Now().After(*key.ExpiresAt) {
        return nil, fmt.Errorf("API key expired")
    }
    
    // Update last used
    go m.updateLastUsed(key.ID)
    
    return &key, nil
}

func (m *APIKeyManager) updateLastUsed(keyID string) {
    now := time.Now()
    m.db.Exec(
        "UPDATE api_keys SET last_used_at = $1 WHERE id = $2",
        now, keyID,
    )
}

// Revoke API key
func (m *APIKeyManager) Revoke(ctx context.Context, keyID, userID string) error {
    result, err := m.db.ExecContext(ctx, `
        UPDATE api_keys 
        SET revoked_at = $1 
        WHERE id = $2 AND user_id = $3
    `, time.Now(), keyID, userID)
    
    if err != nil {
        return err
    }
    
    rows, _ := result.RowsAffected()
    if rows == 0 {
        return fmt.Errorf("key not found or not owned by user")
    }
    
    return nil
}

// API Key Middleware
func APIKeyMiddleware(manager *APIKeyManager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // ดึง API key จาก header หรือ query
            keyString := r.Header.Get("X-API-Key")
            if keyString == "" {
                keyString = r.URL.Query().Get("api_key")
            }
            
            if keyString == "" {
                http.Error(w, "API key required", http.StatusUnauthorized)
                return
            }
            
            key, err := manager.Verify(r.Context(), keyString)
            if err != nil {
                http.Error(w, "Invalid API key: "+err.Error(), http.StatusUnauthorized)
                return
            }
            
            ctx := context.WithValue(r.Context(), "api_key", key)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

---

## 4. Secrets Rotation

```go
// secrets/rotation.go
package secrets

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"
)

type Secret struct {
    ID         string
    Key        string
    Value      string
    Version    int
    ExpiresAt  time.Time
    RotatedAt  *time.Time
}

type SecretManager struct {
    db       *sql.DB
    vault    VaultClient
}

// Rotate database credentials
func (sm *SecretManager) RotateDBCredentials(ctx context.Context, dbName string) error {
    log.Printf("Starting rotation for database: %s", dbName)
    
    // 1. Generate new password
    newPassword, err := generateSecurePassword(32)
    if err != nil {
        return fmt.Errorf("failed to generate password: %w", err)
    }
    
    // 2. Create new DB user with new password
    if err := sm.createNewDBUser(ctx, dbName, newPassword); err != nil {
        return fmt.Errorf("failed to create new user: %w", err)
    }
    
    // 3. Update secret in Vault
    if err := sm.vault.WriteSecret(ctx, "database/"+dbName, map[string]string{
        "password": newPassword,
        "version":  fmt.Sprintf("%d", time.Now().Unix()),
    }); err != nil {
        return fmt.Errorf("failed to update vault: %w", err)
    }
    
    // 4. Wait for applications to pick up new credentials
    time.Sleep(30 * time.Second)
    
    // 5. Revoke old credentials
    if err := sm.revokeOldDBUser(ctx, dbName); err != nil {
        log.Printf("Warning: failed to revoke old user: %v", err)
    }
    
    log.Printf("Rotation complete for database: %s", dbName)
    return nil
}

// Auto-rotation scheduler
type RotationScheduler struct {
    manager  *SecretManager
    interval time.Duration
}

func (s *RotationScheduler) Start(ctx context.Context) {
    ticker := time.NewTicker(s.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            secrets, err := s.manager.getExpiringSoon(ctx, 7*24*time.Hour)
            if err != nil {
                log.Printf("Failed to get expiring secrets: %v", err)
                continue
            }
            
            for _, secret := range secrets {
                go func(s Secret) {
                    if err := s.manager.rotateSecret(ctx, s.Key); err != nil {
                        log.Printf("Failed to rotate %s: %v", s.Key, err)
                    }
                }(secret)
            }
        }
    }
}
```

---

## 5. HashiCorp Vault Integration

```go
// vault/client.go
package vault

import (
    "context"
    "fmt"
    
    vault "github.com/hashicorp/vault/api"
)

type VaultClient struct {
    client *vault.Client
}

func NewVaultClient(address, token string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = address
    
    client, err := vault.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create vault client: %w", err)
    }
    
    client.SetToken(token)
    
    return &VaultClient{client: client}, nil
}

// อ่าน secret
func (v *VaultClient) ReadSecret(ctx context.Context, path string) (map[string]interface{}, error) {
    secret, err := v.client.KVv2("secret").Get(ctx, path)
    if err != nil {
        return nil, fmt.Errorf("failed to read secret at %s: %w", path, err)
    }
    
    if secret == nil || secret.Data == nil {
        return nil, fmt.Errorf("no data at path %s", path)
    }
    
    return secret.Data, nil
}

// เขียน secret
func (v *VaultClient) WriteSecret(ctx context.Context, path string, data map[string]interface{}) error {
    _, err := v.client.KVv2("secret").Put(ctx, path, data)
    if err != nil {
        return fmt.Errorf("failed to write secret at %s: %w", path, err)
    }
    return nil
}

// Dynamic credentials สำหรับ Database
func (v *VaultClient) GetDatabaseCreds(ctx context.Context, role string) (username, password string, err error) {
    secret, err := v.client.Logical().ReadWithContext(ctx, "database/creds/"+role)
    if err != nil {
        return "", "", fmt.Errorf("failed to get DB creds: %w", err)
    }
    
    username, ok := secret.Data["username"].(string)
    if !ok {
        return "", "", fmt.Errorf("invalid username in response")
    }
    
    password, ok = secret.Data["password"].(string)
    if !ok {
        return "", "", fmt.Errorf("invalid password in response")
    }
    
    return username, password, nil
}

// AppRole authentication (สำหรับ production)
func NewVaultClientAppRole(address, roleID, secretID string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = address
    
    client, err := vault.NewClient(config)
    if err != nil {
        return nil, err
    }
    
    // Authenticate with AppRole
    secret, err := client.Logical().Write("auth/approle/login", map[string]interface{}{
        "role_id":   roleID,
        "secret_id": secretID,
    })
    if err != nil {
        return nil, fmt.Errorf("AppRole auth failed: %w", err)
    }
    
    client.SetToken(secret.Auth.ClientToken)
    
    vc := &VaultClient{client: client}
    
    // Start token renewal goroutine
    go vc.renewToken(context.Background(), secret)
    
    return vc, nil
}

func (v *VaultClient) renewToken(ctx context.Context, secret *vault.Secret) {
    watcher, err := v.client.NewLifetimeWatcher(&vault.LifetimeWatcherInput{
        Secret: secret,
    })
    if err != nil {
        return
    }
    
    go watcher.Start()
    defer watcher.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case renewal := <-watcher.RenewCh():
            fmt.Printf("Token renewed, lease: %s\n", renewal.Secret.LeaseID)
        case err := <-watcher.DoneCh():
            if err != nil {
                fmt.Printf("Token renewal failed: %v\n", err)
            }
            return
        }
    }
}

// Vault-backed configuration
type VaultConfig struct {
    vault  *VaultClient
    cache  map[string]cachedValue
}

type cachedValue struct {
    value     string
    expiresAt time.Time
}

func (vc *VaultConfig) Get(ctx context.Context, key string) (string, error) {
    // Check cache
    if cached, ok := vc.cache[key]; ok && time.Now().Before(cached.expiresAt) {
        return cached.value, nil
    }
    
    // Read from Vault
    parts := strings.SplitN(key, "/", 2)
    if len(parts) != 2 {
        return "", fmt.Errorf("invalid key format")
    }
    
    data, err := vc.vault.ReadSecret(ctx, parts[0])
    if err != nil {
        return "", err
    }
    
    val, ok := data[parts[1]].(string)
    if !ok {
        return "", fmt.Errorf("key %s not found", key)
    }
    
    // Cache for 5 minutes
    vc.cache[key] = cachedValue{
        value:     val,
        expiresAt: time.Now().Add(5 * time.Minute),
    }
    
    return val, nil
}
```

---

## 6. Security Scanning ด้วย gosec

```go
// ตัวอย่าง code ที่ gosec จะ flag
package main

import (
    "crypto/md5"  // G501: MD5 is weak
    "crypto/sha1" // G505: SHA1 is weak
    "fmt"
    "math/rand"   // G404: weak random
    "net/http"
    "os/exec"    // G204: command injection risk
)

// ❌ Issues ที่ gosec จะตรวจจับ
func insecureExamples() {
    // G401: Use of weak hash MD5
    h := md5.New()
    h.Write([]byte("password"))
    
    // G501: import of crypto/sha1
    _ = sha1.New()
    
    // G404: Use of weak random
    n := rand.Intn(100)
    fmt.Println(n)
    
    // G204: Subprocess launched with variable
    userInput := "ls"
    exec.Command(userInput).Run()
    
    // G107: URL provided to HTTP request is formed from user input
    url := "http://user-controlled-url.com"
    http.Get(url)
}

// ✅ แก้ไข
import (
    "crypto/rand"
    "crypto/sha256"
    "math/big"
)

func secureExamples() {
    // ใช้ SHA-256 แทน MD5
    h := sha256.New()
    h.Write([]byte("password"))
    hash := h.Sum(nil)
    fmt.Printf("%x\n", hash)
    
    // ใช้ crypto/rand แทน math/rand
    max := big.NewInt(100)
    n, _ := rand.Int(rand.Reader, max)
    fmt.Println(n)
}
```

```bash
# การใช้งาน gosec
go install github.com/securego/gosec/v2/cmd/gosec@latest

# Scan ทั้ง project
gosec ./...

# Scan พร้อม output เป็น JSON
gosec -fmt json -out results.json ./...

# Exclude ปัญหาบางประเภท
gosec -exclude G204,G304 ./...

# กำหนด severity ขั้นต่ำ
gosec -severity high ./...
```

```yaml
# .gosec.yaml - configuration
rules:
  exclude:
    - G104  # Errors unhandled (ถ้า project ใหญ่มาก)
severity: medium
confidence: medium
nosec: false
```

---

## 7. Input Validation & Sanitization

```go
// security/validation.go
package security

import (
    "fmt"
    "net/url"
    "regexp"
    "strings"
    "unicode/utf8"
    
    "github.com/go-playground/validator/v10"
    "github.com/microcosm-cc/bluemonday"
)

var validate = validator.New()

type UserInput struct {
    Name     string `validate:"required,min=2,max=100,alphaunicode"`
    Email    string `validate:"required,email"`
    Phone    string `validate:"omitempty,e164"`
    Website  string `validate:"omitempty,url"`
    Age      int    `validate:"required,gte=0,lte=150"`
    Bio      string `validate:"omitempty,max=500"`
}

func ValidateUserInput(input UserInput) error {
    if err := validate.Struct(input); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    return nil
}

// HTML Sanitization
var policy = bluemonday.UGCPolicy() // ยอมรับ HTML ทั่วไป แต่ไม่มี script

func SanitizeHTML(input string) string {
    return policy.Sanitize(input)
}

// SQL Injection Prevention - ใช้ parameterized queries เสมอ
func safeQuery(db *sql.DB, userInput string) (*sql.Rows, error) {
    // ✅ ปลอดภัย - parameterized query
    return db.Query("SELECT * FROM users WHERE name = $1", userInput)
    
    // ❌ ไม่ปลอดภัย - string concatenation
    // return db.Query("SELECT * FROM users WHERE name = '" + userInput + "'")
}

// Path Traversal Prevention
func safeFilePath(baseDir, userPath string) (string, error) {
    // Clean path
    cleanPath := filepath.Clean(userPath)
    
    // ตรวจสอบว่าอยู่ใน base directory
    fullPath := filepath.Join(baseDir, cleanPath)
    
    rel, err := filepath.Rel(baseDir, fullPath)
    if err != nil || strings.HasPrefix(rel, "..") {
        return "", fmt.Errorf("path traversal detected")
    }
    
    return fullPath, nil
}

// SSRF Prevention
var (
    privateIPRanges = []*net.IPNet{
        mustParseCIDR("10.0.0.0/8"),
        mustParseCIDR("172.16.0.0/12"),
        mustParseCIDR("192.168.0.0/16"),
        mustParseCIDR("127.0.0.0/8"),
        mustParseCIDR("169.254.0.0/16"),
    }
)

func isPrivateIP(ip net.IP) bool {
    for _, rang := range privateIPRanges {
        if rang.Contains(ip) {
            return true
        }
    }
    return false
}

func safeHTTPRequest(rawURL string) (*http.Response, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return nil, fmt.Errorf("invalid URL")
    }
    
    // ตรวจสอบ scheme
    if u.Scheme != "https" {
        return nil, fmt.Errorf("only HTTPS allowed")
    }
    
    // Resolve hostname
    addrs, err := net.LookupHost(u.Hostname())
    if err != nil {
        return nil, fmt.Errorf("DNS lookup failed")
    }
    
    // ตรวจสอบว่าไม่ใช่ private IP (SSRF prevention)
    for _, addr := range addrs {
        ip := net.ParseIP(addr)
        if isPrivateIP(ip) {
            return nil, fmt.Errorf("access to private IP not allowed")
        }
    }
    
    return http.Get(rawURL)
}
```

---

## 8. Container Security

```dockerfile
# Dockerfile ที่ปลอดภัย
# Stage 1: Build
FROM golang:1.21-alpine AS builder

# ไม่ใช้ root ใน build stage
RUN adduser -D -g '' appuser

WORKDIR /build

# Copy และ download dependencies ก่อน (ประหยัด cache)
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build static binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s -extldflags '-static'" \
    -o /build/app ./cmd/server

# Stage 2: Minimal runtime image
FROM scratch

# Copy timezone data
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Copy CA certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy user info
COPY --from=builder /etc/passwd /etc/passwd

# Copy binary
COPY --from=builder /build/app /app

# ใช้ non-root user
USER appuser

# อย่า expose ข้อมูล sensitive ใน environment
ENV PORT=8080

EXPOSE 8080

ENTRYPOINT ["/app"]
```

```yaml
# kubernetes/security-context.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      # ไม่ให้ access service account token โดยไม่จำเป็น
      automountServiceAccountToken: false
      
      # Security context ระดับ pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 2000
      
      containers:
      - name: app
        image: secure-app:latest
        
        # Security context ระดับ container
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Resource limits (ป้องกัน resource exhaustion)
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        
        # Liveness probe ป้องกัน zombie processes
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        
        env:
        # ไม่ใส่ secrets ใน env โดยตรง
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        
        volumeMounts:
        # Mount writable directory แยก
        - name: tmp-dir
          mountPath: /tmp
      
      volumes:
      - name: tmp-dir
        emptyDir: {}
```

---

## Workshop: Secure API Gateway

```go
// workshop/secure_gateway.go
package main

import (
    "context"
    "log"
    "net/http"
    "net/http/httputil"
    "net/url"
    "time"
)

type SecureGateway struct {
    apiKeyManager *APIKeyManager
    rateLimiter   *RateLimiter
    vaultClient   *VaultClient
    proxy         *httputil.ReverseProxy
}

func NewSecureGateway(upstreamURL string) *SecureGateway {
    upstream, _ := url.Parse(upstreamURL)
    
    return &SecureGateway{
        apiKeyManager: NewAPIKeyManager(db),
        rateLimiter:   NewRateLimiter(redis),
        vaultClient:   NewVaultClient(vaultAddr, vaultToken),
        proxy:         httputil.NewSingleHostReverseProxy(upstream),
    }
}

func (g *SecureGateway) Handler() http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("/", g.proxyHandler)
    
    // Chain security middlewares
    handler := g.rateLimitMiddleware(
        g.authMiddleware(
            g.securityHeadersMiddleware(mux),
        ),
    )
    
    return handler
}

func (g *SecureGateway) authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        apiKey := r.Header.Get("X-API-Key")
        if apiKey == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        key, err := g.apiKeyManager.Verify(r.Context(), apiKey)
        if err != nil {
            http.Error(w, "Unauthorized: "+err.Error(), http.StatusUnauthorized)
            return
        }
        
        ctx := context.WithValue(r.Context(), "api_key", key)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func (g *SecureGateway) rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        apiKey := r.Context().Value("api_key").(*APIKey)
        
        allowed, remaining, resetAt := g.rateLimiter.Check(r.Context(), apiKey.ID, apiKey.RateLimit)
        
        w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", apiKey.RateLimit))
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", remaining))
        w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", resetAt.Unix()))
        
        if !allowed {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

func (g *SecureGateway) securityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Security headers
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // Remove sensitive headers from upstream
        r.Header.Del("X-Forwarded-For")
        r.Header.Del("X-Real-IP")
        
        next.ServeHTTP(w, r)
    })
}

func main() {
    gateway := NewSecureGateway("http://upstream:8080")
    
    server := &http.Server{
        Addr:         ":8443",
        Handler:      gateway.Handler(),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }
    
    log.Println("Secure gateway starting on :8443")
    log.Fatal(server.ListenAndServeTLS("server.crt", "server.key"))
}
```

---

## สรุป

| Security Layer | Tool/Technique | Purpose |
|----------------|----------------|---------|
| Transport | mTLS | Encrypt & verify connections |
| Authentication | JWT, API Keys | Who are you? |
| Authorization | RBAC, ABAC | What can you do? |
| Secrets | Vault | Store sensitive data |
| Input | Validation, Sanitization | Prevent injection |
| Code | gosec, semgrep | Find vulnerabilities |
| Container | Non-root, read-only FS | Reduce attack surface |
| Network | Zero Trust | Verify every request |

### Security Checklist
- [ ] TLS 1.3 ทุก connection
- [ ] Input validation ทุก endpoint
- [ ] Secrets ใน Vault ไม่ใช่ ENV vars
- [ ] Non-root container user
- [ ] Read-only filesystem
- [ ] Rate limiting
- [ ] Security headers
- [ ] CORS policy
- [ ] gosec scan ใน CI/CD
- [ ] Dependency vulnerability scan

---

*จบ Part 69: Advanced Security*
