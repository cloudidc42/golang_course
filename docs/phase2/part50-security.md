# Part 50: Security ใน Go Applications

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ป้องกัน OWASP Top 10 vulnerabilities
- ป้องกัน SQL Injection
- ป้องกัน XSS
- ป้องกัน CSRF
- Validate inputs
- ตั้งค่า Secure Headers
- ใช้ TLS/HTTPS
- จัดการ Secrets อย่างปลอดภัย
- ทำ Security Testing

---

## 1. OWASP Top 10 Overview

```go
// ตัวอย่าง 1: OWASP Top 10 ที่พบบ่อยใน Go

// A01: Broken Access Control
// A02: Cryptographic Failures
// A03: Injection (SQL, Command, LDAP)
// A04: Insecure Design
// A05: Security Misconfiguration
// A06: Vulnerable and Outdated Components
// A07: Authentication and Identification Failures
// A08: Software and Data Integrity Failures
// A09: Security Logging and Monitoring Failures
// A10: Server-Side Request Forgery (SSRF)

package security_examples

import (
	"fmt"
	"log/slog"
	"net/http"
	"time"
)

// Security audit logger
type SecurityLogger struct {
	logger *slog.Logger
}

func NewSecurityLogger() *SecurityLogger {
	return &SecurityLogger{
		logger: slog.Default(),
	}
}

func (l *SecurityLogger) LogFailedAuth(r *http.Request, userID string) {
	l.logger.Warn("authentication failed",
		"user_id", userID,
		"ip", r.RemoteAddr,
		"path", r.URL.Path,
		"timestamp", time.Now().UTC(),
	)
}

func (l *SecurityLogger) LogAccessDenied(r *http.Request, userID, resource string) {
	l.logger.Warn("access denied",
		"user_id", userID,
		"resource", resource,
		"ip", r.RemoteAddr,
	)
}

func (l *SecurityLogger) LogSuspiciousActivity(r *http.Request, reason string) {
	l.logger.Error("suspicious activity",
		"reason", reason,
		"ip", r.RemoteAddr,
		"path", r.URL.Path,
		"user_agent", r.Header.Get("User-Agent"),
	)
}

func main() {
	fmt.Println("Security examples - see each section below")
}
```

---

## 2. SQL Injection Prevention

```go
// ตัวอย่าง 2: SQL Injection prevention

package security_examples

import (
	"database/sql"
	"fmt"
	"net/http"
)

// VULNERABLE: SQL Injection
func getUserVulnerable(db *sql.DB, username string) (string, error) {
	// NEVER do this!
	query := fmt.Sprintf("SELECT email FROM users WHERE username = '%s'", username)
	// Input: ' OR '1'='1'--
	// Query becomes: SELECT email FROM users WHERE username = '' OR '1'='1'--'
	
	var email string
	err := db.QueryRow(query).Scan(&email)
	return email, err
}

// SAFE: Parameterized queries
func getUserSafe(db *sql.DB, username string) (string, error) {
	query := "SELECT email FROM users WHERE username = $1"
	// Parameters are escaped by the driver
	
	var email string
	err := db.QueryRow(query, username).Scan(&email)
	return email, err
}

// SAFE: Multiple parameters
func getUserByFilter(db *sql.DB, minAge, maxAge int, role string) ([]*User, error) {
	query := `
		SELECT id, name, email 
		FROM users 
		WHERE age BETWEEN $1 AND $2 AND role = $3
		ORDER BY name`
	
	rows, err := db.Query(query, minAge, maxAge, role)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	
	var users []*User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
			return nil, err
		}
		users = append(users, &u)
	}
	return users, rows.Err()
}

// ตัวอย่าง 3: Dynamic ORDER BY (cannot use params for column names)
func getUsersSorted(db *sql.DB, sortField, sortDir string) ([]*User, error) {
	// Whitelist allowed columns
	allowedColumns := map[string]bool{
		"name":       true,
		"email":      true,
		"created_at": true,
	}
	
	if !allowedColumns[sortField] {
		return nil, fmt.Errorf("invalid sort field: %s", sortField)
	}
	
	// Whitelist direction
	if sortDir != "ASC" && sortDir != "DESC" {
		sortDir = "ASC"
	}
	
	// Safe to use in query now
	query := fmt.Sprintf("SELECT id, name, email FROM users ORDER BY %s %s",
		sortField, sortDir)
	
	rows, err := db.Query(query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	
	var users []*User
	for rows.Next() {
		var u User
		rows.Scan(&u.ID, &u.Name, &u.Email)
		users = append(users, &u)
	}
	return users, nil
}

type User struct {
	ID    int
	Name  string
	Email string
}
```

---

## 3. XSS Prevention

```go
// ตัวอย่าง 4: XSS prevention

package security_examples

import (
	"html/template"
	"net/http"
)

// VULNERABLE: Using fmt.Fprintf directly
func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	// XSS: <script>alert('xss')</script>
	fmt.Fprintf(w, "<h1>Hello, %s!</h1>", name) // UNSAFE!
}

// SAFE: Using html/template (auto-escapes)
var safeTemplate = template.Must(template.New("greeting").Parse(
	`<!DOCTYPE html>
<html>
<head><title>Greeting</title></head>
<body>
    <h1>Hello, {{.Name}}!</h1>
    <p>Your message: {{.Message}}</p>
</body>
</html>`))

type GreetingData struct {
	Name    string
	Message string
}

func safeHandler(w http.ResponseWriter, r *http.Request) {
	data := GreetingData{
		Name:    r.URL.Query().Get("name"),
		Message: r.URL.Query().Get("message"),
	}
	
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	safeTemplate.Execute(w, data) // Auto-escapes HTML
}

// ตัวอย่าง 5: Manual HTML escaping
func manualEscape() {
	input := `<script>alert('xss')</script>`
	escaped := template.HTMLEscapeString(input)
	fmt.Printf("Escaped: %s\n", escaped)
	// Output: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
}

// Content Security Policy
func addCSPHeader(w http.ResponseWriter) {
	w.Header().Set("Content-Security-Policy",
		"default-src 'self'; "+
		"script-src 'self' 'nonce-abc123'; "+
		"style-src 'self'; "+
		"img-src 'self' data:; "+
		"connect-src 'self'; "+
		"frame-ancestors 'none'")
}
```

---

## 4. CSRF Protection

```go
// ตัวอย่าง 6: CSRF token implementation

package security_examples

import (
	"crypto/rand"
	"encoding/base64"
	"net/http"
	"sync"
)

type CSRFProtection struct {
	tokens map[string]struct{}
	mu     sync.RWMutex
}

func NewCSRFProtection() *CSRFProtection {
	return &CSRFProtection{
		tokens: make(map[string]struct{}),
	}
}

func (c *CSRFProtection) GenerateToken() (string, error) {
	b := make([]byte, 32)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	
	token := base64.URLEncoding.EncodeToString(b)
	
	c.mu.Lock()
	c.tokens[token] = struct{}{}
	c.mu.Unlock()
	
	return token, nil
}

func (c *CSRFProtection) ValidateToken(token string) bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	_, valid := c.tokens[token]
	if valid {
		delete(c.tokens, token) // One-time use
	}
	return valid
}

// Middleware
func (c *CSRFProtection) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodPost ||
			r.Method == http.MethodPut ||
			r.Method == http.MethodDelete ||
			r.Method == http.MethodPatch {
			
			token := r.Header.Get("X-CSRF-Token")
			if token == "" {
				token = r.FormValue("csrf_token")
			}
			
			if !c.ValidateToken(token) {
				http.Error(w, "CSRF token invalid", http.StatusForbidden)
				return
			}
		}
		
		// Add CSRF token to response for next request
		token, _ := c.GenerateToken()
		w.Header().Set("X-CSRF-Token", token)
		
		next.ServeHTTP(w, r)
	})
}
```

---

## 5. Input Validation

```go
// ตัวอย่าง 7: Input validation

package security_examples

import (
	"errors"
	"fmt"
	"net/mail"
	"regexp"
	"strings"
	"unicode"
)

var (
	usernameRegex = regexp.MustCompile(`^[a-zA-Z0-9_-]{3,50}$`)
	phoneRegex    = regexp.MustCompile(`^\+?[1-9]\d{1,14}$`)
)

type ValidationError struct {
	Field   string
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

type ValidationErrors []*ValidationError

func (e ValidationErrors) Error() string {
	msgs := make([]string, len(e))
	for i, err := range e {
		msgs[i] = err.Error()
	}
	return strings.Join(msgs, "; ")
}

type RegisterRequest struct {
	Username string
	Email    string
	Password string
	Phone    string
	Age      int
}

func ValidateRegisterRequest(req *RegisterRequest) ValidationErrors {
	var errs ValidationErrors
	
	// Username validation
	if req.Username == "" {
		errs = append(errs, &ValidationError{"username", "required"})
	} else if !usernameRegex.MatchString(req.Username) {
		errs = append(errs, &ValidationError{"username",
			"must be 3-50 chars, alphanumeric and - _"})
	}
	
	// Email validation
	if req.Email == "" {
		errs = append(errs, &ValidationError{"email", "required"})
	} else if _, err := mail.ParseAddress(req.Email); err != nil {
		errs = append(errs, &ValidationError{"email", "invalid format"})
	}
	
	// Password validation
	if err := validatePassword(req.Password); err != nil {
		errs = append(errs, &ValidationError{"password", err.Error()})
	}
	
	// Phone validation (optional)
	if req.Phone != "" && !phoneRegex.MatchString(req.Phone) {
		errs = append(errs, &ValidationError{"phone", "invalid format"})
	}
	
	// Age validation
	if req.Age < 13 || req.Age > 150 {
		errs = append(errs, &ValidationError{"age", "must be between 13 and 150"})
	}
	
	return errs
}

func validatePassword(password string) error {
	if len(password) < 8 {
		return errors.New("must be at least 8 characters")
	}
	if len(password) > 100 {
		return errors.New("must be at most 100 characters")
	}
	
	var hasUpper, hasLower, hasDigit, hasSpecial bool
	for _, c := range password {
		switch {
		case unicode.IsUpper(c):
			hasUpper = true
		case unicode.IsLower(c):
			hasLower = true
		case unicode.IsDigit(c):
			hasDigit = true
		case unicode.IsPunct(c) || unicode.IsSymbol(c):
			hasSpecial = true
		}
	}
	
	if !hasUpper || !hasLower || !hasDigit || !hasSpecial {
		return errors.New("must contain uppercase, lowercase, digit, and special character")
	}
	
	return nil
}

// ตัวอย่าง 8: Path traversal prevention

func safePath(baseDir, userInput string) (string, error) {
	// Clean the path
	cleaned := filepath.Clean(filepath.Join(baseDir, userInput))
	
	// Ensure it's within baseDir
	if !strings.HasPrefix(cleaned, baseDir) {
		return "", fmt.Errorf("path traversal detected")
	}
	
	return cleaned, nil
}
```

---

## 6. Secure Headers

```go
// ตัวอย่าง 9: Security headers middleware

package security_examples

import (
	"net/http"
)

func SecureHeadersMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		
		// Prevent MIME sniffing
		h.Set("X-Content-Type-Options", "nosniff")
		
		// Prevent clickjacking
		h.Set("X-Frame-Options", "DENY")
		
		// Enable XSS protection (legacy browsers)
		h.Set("X-XSS-Protection", "1; mode=block")
		
		// Force HTTPS
		h.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload")
		
		// Content Security Policy
		h.Set("Content-Security-Policy",
			"default-src 'self'; "+
			"script-src 'self'; "+
			"style-src 'self' 'unsafe-inline'; "+
			"img-src 'self' data: https:; "+
			"frame-ancestors 'none'; "+
			"base-uri 'self'; "+
			"form-action 'self'")
		
		// Referrer policy
		h.Set("Referrer-Policy", "strict-origin-when-cross-origin")
		
		// Permissions policy
		h.Set("Permissions-Policy",
			"camera=(), microphone=(), geolocation=(self), payment=()")
		
		// Remove server info
		h.Del("Server")
		h.Del("X-Powered-By")
		
		next.ServeHTTP(w, r)
	})
}

// Rate limit middleware
type RateLimiter struct {
	visitors map[string]*rate.Limiter
	mu       sync.RWMutex
}

func NewRateLimiter(rps float64, burst int) *RateLimiter {
	return &RateLimiter{
		visitors: make(map[string]*rate.Limiter),
	}
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	
	limiter, ok := rl.visitors[ip]
	if !ok {
		limiter = rate.NewLimiter(rate.Limit(10), 20)
		rl.visitors[ip] = limiter
	}
	return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ip := r.RemoteAddr
		limiter := rl.getLimiter(ip)
		
		if !limiter.Allow() {
			http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
			return
		}
		
		next.ServeHTTP(w, r)
	})
}
```

---

## 7. TLS/HTTPS

```go
// ตัวอย่าง 10: Secure TLS configuration

package security_examples

import (
	"crypto/tls"
	"net/http"
	"time"
)

func createTLSServer(addr, certFile, keyFile string) *http.Server {
	tlsConfig := &tls.Config{
		// Minimum TLS version
		MinVersion: tls.VersionTLS12,
		
		// Preferred cipher suites
		CipherSuites: []uint16{
			tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
			tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
			tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
			tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
			tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
			tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
		},
		
		// Prefer server's cipher suites
		PreferServerCipherSuites: true,
		
		// Curve preferences
		CurvePreferences: []tls.CurveID{
			tls.X25519,
			tls.CurveP256,
		},
	}
	
	server := &http.Server{
		Addr:      addr,
		TLSConfig: tlsConfig,
		
		// Timeouts to prevent Slowloris attack
		ReadTimeout:       5 * time.Second,
		WriteTimeout:      10 * time.Second,
		IdleTimeout:       120 * time.Second,
		ReadHeaderTimeout: 5 * time.Second,
		
		MaxHeaderBytes: 1 << 20, // 1MB
	}
	
	return server
}

// Redirect HTTP to HTTPS
func redirectHTTPS(w http.ResponseWriter, r *http.Request) {
	target := "https://" + r.Host + r.URL.Path
	if len(r.URL.RawQuery) > 0 {
		target += "?" + r.URL.RawQuery
	}
	http.Redirect(w, r, target, http.StatusMovedPermanently)
}

func runSecureServer(handler http.Handler, certFile, keyFile string) error {
	// HTTP server (redirect to HTTPS)
	httpServer := &http.Server{
		Addr:    ":80",
		Handler: http.HandlerFunc(redirectHTTPS),
	}
	go httpServer.ListenAndServe()
	
	// HTTPS server
	httpsServer := createTLSServer(":443", certFile, keyFile)
	httpsServer.Handler = handler
	
	return httpsServer.ListenAndServeTLS(certFile, keyFile)
}
```

---

## 8. Secret Management

```go
// ตัวอย่าง 11: Secure secret handling

package security_examples

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"io"
	"os"
)

// Never hardcode secrets!
// BAD:
// const apiKey = "sk-1234567890abcdef"

// GOOD: Load from environment
func loadSecrets() (string, error) {
	apiKey := os.Getenv("API_KEY")
	if apiKey == "" {
		return "", fmt.Errorf("API_KEY environment variable not set")
	}
	return apiKey, nil
}

// AES-GCM encryption for storing sensitive data
type Encryptor struct {
	gcm cipher.AEAD
}

func NewEncryptor(key []byte) (*Encryptor, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	
	return &Encryptor{gcm: gcm}, nil
}

func (e *Encryptor) Encrypt(plaintext []byte) (string, error) {
	nonce := make([]byte, e.gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return "", err
	}
	
	ciphertext := e.gcm.Seal(nonce, nonce, plaintext, nil)
	return base64.URLEncoding.EncodeToString(ciphertext), nil
}

func (e *Encryptor) Decrypt(encodedCiphertext string) ([]byte, error) {
	ciphertext, err := base64.URLEncoding.DecodeString(encodedCiphertext)
	if err != nil {
		return nil, err
	}
	
	nonceSize := e.gcm.NonceSize()
	if len(ciphertext) < nonceSize {
		return nil, fmt.Errorf("ciphertext too short")
	}
	
	nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
	return e.gcm.Open(nil, nonce, ciphertext, nil)
}

// ตัวอย่าง 12: Password hashing with bcrypt

func hashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", err
	}
	return string(hash), nil
}

func verifyPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}

func passwordDemo() {
	password := "SecureP@ssw0rd!"
	
	hash, err := hashPassword(password)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	fmt.Printf("Hash: %s\n", hash[:30]+"...")
	
	// Verify
	if verifyPassword(hash, password) {
		fmt.Println("Password verified!")
	}
	
	if !verifyPassword(hash, "wrongpassword") {
		fmt.Println("Wrong password rejected!")
	}
}
```

---

## 9. Command Injection Prevention

```go
// ตัวอย่าง 13: Prevent command injection

package security_examples

import (
	"fmt"
	"os/exec"
)

// VULNERABLE: Command injection
func vulnerableExec(filename string) (string, error) {
	// Input: "file.txt; rm -rf /"
	cmd := exec.Command("sh", "-c", "cat "+filename) // UNSAFE!
	output, err := cmd.Output()
	return string(output), err
}

// SAFE: Use args instead of shell string
func safeExec(filename string) (string, error) {
	// Each argument is separate - no shell interpretation
	cmd := exec.Command("cat", filename) // SAFE
	output, err := cmd.Output()
	return string(output), err
}

// SAFE: Validate before using
func safeExecWithValidation(filename string) (string, error) {
	// Whitelist valid characters in filename
	for _, c := range filename {
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			c == '.' || c == '-' || c == '_') {
			return "", fmt.Errorf("invalid filename")
		}
	}
	
	cmd := exec.Command("cat", filename)
	output, err := cmd.Output()
	return string(output), err
}

// SSRF Prevention
func safeHTTPRequest(userURL string) (*http.Response, error) {
	// Parse URL
	u, err := url.Parse(userURL)
	if err != nil {
		return nil, fmt.Errorf("invalid URL: %w", err)
	}
	
	// Whitelist allowed schemes
	if u.Scheme != "http" && u.Scheme != "https" {
		return nil, fmt.Errorf("only http/https allowed")
	}
	
	// Block private IP ranges
	host := u.Hostname()
	ips, err := net.LookupHost(host)
	if err != nil {
		return nil, fmt.Errorf("DNS lookup failed: %w", err)
	}
	
	for _, ipStr := range ips {
		ip := net.ParseIP(ipStr)
		if ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast() {
			return nil, fmt.Errorf("requests to private IPs not allowed")
		}
	}
	
	client := &http.Client{Timeout: 10 * time.Second}
	return client.Get(userURL)
}
```

---

## 10. Dependency Security

```bash
# ตัวอย่าง 14: Dependency security scanning

# Check for known vulnerabilities
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# Output example:
# Vulnerability #1: GO-2024-2491
# ...
# Call stacks in your code:
#   github.com/myapp/api.handleLogin calls crypto/tls.Conn.Handshake
#     affecting golang.org/x/net@v0.17.0

# Nancy - check go.sum against OSS Index
go install github.com/sonatypecommunity/nancy@latest
go list -json -m all | nancy sleuth

# Snyk
snyk test

# Update dependencies
go get -u ./...
go mod tidy
```

---

## 11. Security Testing

```go
// ตัวอย่าง 15: Security-focused tests

package security_test

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestSQLInjectionPrevention(t *testing.T) {
	injections := []string{
		"' OR '1'='1",
		"'; DROP TABLE users; --",
		"1 UNION SELECT * FROM users",
		`" OR 1=1--`,
		"' AND 1=0 UNION SELECT username,password FROM users--",
	}
	
	for _, injection := range injections {
		t.Run("injection: "+injection[:min(len(injection), 20)], func(t *testing.T) {
			// These should not cause errors or return unexpected data
			// In a real test, you'd verify against actual DB behavior
			
			// Verify the query would be safe (parameterized)
			// This is a structural test - verify parameterized queries are used
			query, args := buildUserQuery(injection)
			
			// Should use placeholders, not raw interpolation
			if strings.Contains(query, injection) {
				t.Error("SQL injection possible: user input in query")
			}
			if len(args) == 0 {
				t.Error("No parameters used - potential SQL injection")
			}
		})
	}
}

func buildUserQuery(username string) (string, []interface{}) {
	return "SELECT * FROM users WHERE username = $1", []interface{}{username}
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func TestXSSPrevention(t *testing.T) {
	xssPayloads := []string{
		"<script>alert('xss')</script>",
		"<img src=x onerror=alert('xss')>",
		"javascript:alert('xss')",
		`"><script>alert('xss')</script>`,
		`'><script>alert('xss')</script>`,
	}
	
	for _, payload := range xssPayloads {
		t.Run("xss payload", func(t *testing.T) {
			req := httptest.NewRequest("GET", "/search?q="+payload, nil)
			w := httptest.NewRecorder()
			
			searchHandler(w, req)
			
			body := w.Body.String()
			
			// Body should not contain raw script tags
			if strings.Contains(body, "<script>") {
				t.Errorf("XSS vulnerability: raw script in response")
			}
		})
	}
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query().Get("q")
	// Safe template rendering
	tmpl := template.Must(template.New("search").Parse(
		`<p>Results for: {{.}}</p>`))
	tmpl.Execute(w, q)
}

func TestSecureHeaders(t *testing.T) {
	handler := SecureHeadersMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))
	
	req := httptest.NewRequest("GET", "/", nil)
	w := httptest.NewRecorder()
	handler.ServeHTTP(w, req)
	
	resp := w.Result()
	
	headers := map[string]string{
		"X-Content-Type-Options":    "nosniff",
		"X-Frame-Options":           "DENY",
		"X-XSS-Protection":          "1; mode=block",
		"Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
	}
	
	for header, expected := range headers {
		t.Run(header, func(t *testing.T) {
			actual := resp.Header.Get(header)
			if actual != expected {
				t.Errorf("Header %s: expected %q, got %q", header, expected, actual)
			}
		})
	}
}
```

---

## 12. JWT Security

```go
// ตัวอย่าง 16: Secure JWT implementation

package security_examples

import (
	"errors"
	"time"
	
	"github.com/golang-jwt/jwt/v5"
)

type JWTConfig struct {
	Secret     []byte
	Expiry     time.Duration
	Issuer     string
}

type CustomClaims struct {
	UserID int    `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

func createSecureJWT(config JWTConfig, userID int, role string) (string, error) {
	now := time.Now()
	
	claims := CustomClaims{
		UserID: userID,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			Issuer:    config.Issuer,
			Subject:   fmt.Sprintf("%d", userID),
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(config.Expiry)),
			NotBefore: jwt.NewNumericDate(now),
			// Add unique ID to enable revocation
			ID: generateJTI(),
		},
	}
	
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(config.Secret)
}

func generateJTI() string {
	b := make([]byte, 16)
	rand.Read(b)
	return base64.URLEncoding.EncodeToString(b)
}

func validateSecureJWT(config JWTConfig, tokenStr string) (*CustomClaims, error) {
	token, err := jwt.ParseWithClaims(tokenStr, &CustomClaims{},
		func(token *jwt.Token) (interface{}, error) {
			// Verify algorithm
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v",
					token.Header["alg"])
			}
			return config.Secret, nil
		},
		jwt.WithIssuer(config.Issuer),
		jwt.WithExpirationRequired(),
	)
	
	if err != nil {
		if errors.Is(err, jwt.ErrTokenExpired) {
			return nil, fmt.Errorf("token expired")
		}
		return nil, fmt.Errorf("invalid token: %w", err)
	}
	
	claims, ok := token.Claims.(*CustomClaims)
	if !ok {
		return nil, fmt.Errorf("invalid claims")
	}
	
	return claims, nil
}
```

---

## สรุป

ใน Part 50 เราได้เรียนรู้:

1. **OWASP Top 10**: vulnerabilities ที่พบบ่อย
2. **SQL Injection**: ป้องกันด้วย parameterized queries
3. **XSS**: ป้องกันด้วย html/template
4. **CSRF**: token-based protection
5. **Input Validation**: whitelist, regex, struct validation
6. **Secure Headers**: CSP, HSTS, X-Frame-Options
7. **TLS/HTTPS**: secure TLS configuration
8. **Secret Management**: environment variables, bcrypt, AES encryption
9. **Command Injection**: separate args, validate input
10. **Dependency Security**: govulncheck, nancy, snyk
11. **Security Testing**: SQL injection, XSS, header tests
12. **JWT Security**: proper signing, validation, expiry

---

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [Go Security Guidelines](https://github.com/OWASP/Go-SCP)
- [securego/gosec](https://github.com/securego/gosec)
- [Let's Encrypt - Free TLS](https://letsencrypt.org/)
- [golang-jwt](https://github.com/golang-jwt/jwt)
