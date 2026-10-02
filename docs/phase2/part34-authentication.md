# Part 34: Authentication ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ JWT basics
- สร้างและ validate JWT tokens
- ทำ Refresh tokens
- เขียน Auth Middleware
- ทำ Role-based Access Control (RBAC)
- Hash passwords ด้วย bcrypt
- เปรียบเทียบ Session vs JWT authentication

---

## 34.1 JWT Basics

JWT (JSON Web Token) มี 3 ส่วน:
1. **Header** - algorithm ที่ใช้
2. **Payload** - claims (ข้อมูล)
3. **Signature** - ลายเซ็นดิจิทัล

Format: `header.payload.signature` (base64url encoded)

### ติดตั้ง Dependencies

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto/bcrypt
```

---

## 34.2 Password Hashing ด้วย bcrypt

### Hash และ Verify Password (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "log"
    
    "golang.org/x/crypto/bcrypt"
)

func hashPassword(password string) (string, error) {
    // cost 12 ดี สำหรับ production
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return "", fmt.Errorf("hashing password: %w", err)
    }
    return string(bytes), nil
}

func checkPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

func main() {
    password := "MySecurePassword123!"
    
    // Hash password
    hash, err := hashPassword(password)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Original:", password)
    fmt.Println("Hashed:", hash)
    
    // Verify correct password
    if checkPassword(password, hash) {
        fmt.Println("Password verification: CORRECT")
    }
    
    // Verify wrong password
    if !checkPassword("wrongpassword", hash) {
        fmt.Println("Wrong password verification: REJECTED")
    }
    
    // Each hash is different (salt ต่างกัน)
    hash2, _ := hashPassword(password)
    fmt.Printf("Hash 1: %s\n", hash[:20]+"...")
    fmt.Printf("Hash 2: %s\n", hash2[:20]+"...")
    fmt.Printf("Hashes are different: %v\n", hash != hash2)
    // แต่ทั้งคู่ verify ได้
    fmt.Printf("Both verify: %v\n", checkPassword(password, hash) && checkPassword(password, hash2))
}
```

### bcrypt Cost Comparison (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "time"
    
    "golang.org/x/crypto/bcrypt"
)

func benchmarkBcryptCost(cost int, password string) time.Duration {
    start := time.Now()
    bcrypt.GenerateFromPassword([]byte(password), cost)
    return time.Since(start)
}

func main() {
    password := "TestPassword123"
    
    fmt.Println("bcrypt cost benchmarks:")
    for cost := 10; cost <= 14; cost++ {
        duration := benchmarkBcryptCost(cost, password)
        fmt.Printf("  Cost %d: %v\n", cost, duration)
    }
    
    // ใน production: ควรใช้ cost ที่ทำให้ hash ใช้เวลา ~100-300ms
    // ถ้าเร็วเกินไป จะ brute force ได้ง่าย
}
```

---

## 34.3 สร้าง JWT Tokens

### Create JWT Token (ตัวอย่างที่ 3)

```go
package main

import (
    "fmt"
    "log"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("my-super-secret-jwt-key-minimum-32-chars")

// Claims struct
type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateAccessToken(userID, email, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            Issuer:    "my-app",
            Subject:   userID,
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)), // short-lived
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ID:        fmt.Sprintf("%d", time.Now().UnixNano()), // unique token ID
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func main() {
    // สร้าง token
    token, err := generateAccessToken("user-123", "alice@example.com", "user")
    if err != nil {
        log.Fatal("Error creating token:", err)
    }
    
    fmt.Println("Generated Token:")
    fmt.Println(token)
    fmt.Printf("\nToken length: %d characters\n", len(token))
}
```

### Parse และ Validate JWT Token (ตัวอย่างที่ 4)

```go
package main

import (
    "errors"
    "fmt"
    "log"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("my-super-secret-jwt-key-minimum-32-chars")

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID, email, role string, duration time.Duration) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(duration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func parseToken(tokenString string) (*Claims, error) {
    claims := &Claims{}
    
    token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
        // ตรวจสอบ signing method
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, fmt.Errorf("token has expired")
        }
        if errors.Is(err, jwt.ErrTokenSignatureInvalid) {
            return nil, fmt.Errorf("invalid token signature")
        }
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    
    if !token.Valid {
        return nil, fmt.Errorf("token is not valid")
    }
    
    return claims, nil
}

func main() {
    // สร้าง token ที่ valid
    token, err := generateToken("user-123", "alice@example.com", "admin", time.Hour)
    if err != nil {
        log.Fatal(err)
    }
    
    // Parse token
    claims, err := parseToken(token)
    if err != nil {
        log.Fatal("Parse error:", err)
    }
    
    fmt.Println("=== Token Claims ===")
    fmt.Printf("User ID: %s\n", claims.UserID)
    fmt.Printf("Email: %s\n", claims.Email)
    fmt.Printf("Role: %s\n", claims.Role)
    fmt.Printf("Expires at: %v\n", claims.ExpiresAt.Time)
    
    // ทดสอบ invalid token
    _, err = parseToken("invalid.token.here")
    fmt.Println("\nInvalid token error:", err)
    
    // ทดสอบ expired token
    expiredToken, _ := generateToken("user-456", "bob@example.com", "user", -time.Minute)
    _, err = parseToken(expiredToken)
    fmt.Println("Expired token error:", err)
}
```

---

## 34.4 Refresh Tokens

### Access + Refresh Token System (ตัวอย่างที่ 5)

```go
package main

import (
    "fmt"
    "log"
    "sync"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

var (
    accessSecret  = []byte("access-token-secret-minimum-32-chars!!")
    refreshSecret = []byte("refresh-token-secret-minimum-32-chars!")
)

type AccessClaims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

type RefreshClaims struct {
    UserID    string `json:"user_id"`
    TokenFamily string `json:"family"` // สำหรับ token rotation
    jwt.RegisteredClaims
}

type TokenPair struct {
    AccessToken           string    `json:"access_token"`
    RefreshToken          string    `json:"refresh_token"`
    AccessTokenExpiresAt  time.Time `json:"access_expires_at"`
    RefreshTokenExpiresAt time.Time `json:"refresh_expires_at"`
}

// TokenStore เก็บ refresh tokens ที่ valid (ใน production ใช้ Redis)
type TokenStore struct {
    mu     sync.RWMutex
    tokens map[string]bool // tokenID -> isValid
}

func NewTokenStore() *TokenStore {
    return &TokenStore{tokens: make(map[string]bool)}
}

func (s *TokenStore) Add(tokenID string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.tokens[tokenID] = true
}

func (s *TokenStore) IsValid(tokenID string) bool {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.tokens[tokenID]
}

func (s *TokenStore) Revoke(tokenID string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.tokens, tokenID)
}

var tokenStore = NewTokenStore()

func generateTokenPair(userID, email, role string) (*TokenPair, error) {
    now := time.Now()
    
    // Access Token (15 minutes)
    accessExp := now.Add(15 * time.Minute)
    accessClaims := AccessClaims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(accessExp),
            IssuedAt:  jwt.NewNumericDate(now),
        },
    }
    accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims)
    accessTokenString, err := accessToken.SignedString(accessSecret)
    if err != nil {
        return nil, fmt.Errorf("creating access token: %w", err)
    }
    
    // Refresh Token (7 days)
    refreshExp := now.Add(7 * 24 * time.Hour)
    refreshTokenID := fmt.Sprintf("refresh-%s-%d", userID, now.UnixNano())
    refreshClaims := RefreshClaims{
        UserID:      userID,
        TokenFamily: "default",
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        refreshTokenID,
            ExpiresAt: jwt.NewNumericDate(refreshExp),
            IssuedAt:  jwt.NewNumericDate(now),
        },
    }
    refreshToken := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims)
    refreshTokenString, err := refreshToken.SignedString(refreshSecret)
    if err != nil {
        return nil, fmt.Errorf("creating refresh token: %w", err)
    }
    
    // Store refresh token ID
    tokenStore.Add(refreshTokenID)
    
    return &TokenPair{
        AccessToken:           accessTokenString,
        RefreshToken:          refreshTokenString,
        AccessTokenExpiresAt:  accessExp,
        RefreshTokenExpiresAt: refreshExp,
    }, nil
}

func refreshAccessToken(refreshTokenString string) (*TokenPair, error) {
    // Parse refresh token
    claims := &RefreshClaims{}
    _, err := jwt.ParseWithClaims(refreshTokenString, claims, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return refreshSecret, nil
    })
    if err != nil {
        return nil, fmt.Errorf("invalid refresh token: %w", err)
    }
    
    // ตรวจว่า token ถูก revoke ไหม
    if !tokenStore.IsValid(claims.ID) {
        return nil, fmt.Errorf("refresh token has been revoked")
    }
    
    // Revoke old refresh token (token rotation)
    tokenStore.Revoke(claims.ID)
    
    // ต้องดึง user info จาก DB ใน real app
    // Simulate: ดึง role จาก DB
    role := "user" // ปกติต้องดึงจาก DB
    
    // ออก token pair ใหม่
    return generateTokenPair(claims.UserID, "user@example.com", role)
}

func revokeAllUserTokens(userID string) {
    // ใน production: ลบ refresh tokens ทั้งหมดของ user นี้จาก store
    fmt.Printf("Revoked all tokens for user: %s\n", userID)
}

func main() {
    // Login
    tokens, err := generateTokenPair("user-123", "alice@example.com", "user")
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("=== Token Pair Generated ===")
    fmt.Printf("Access Token (first 50): %s...\n", tokens.AccessToken[:50])
    fmt.Printf("Access Expires: %v\n", tokens.AccessTokenExpiresAt)
    fmt.Printf("Refresh Expires: %v\n\n", tokens.RefreshTokenExpiresAt)
    
    // Refresh token
    newTokens, err := refreshAccessToken(tokens.RefreshToken)
    if err != nil {
        log.Fatal("Refresh error:", err)
    }
    
    fmt.Println("=== New Token Pair after Refresh ===")
    fmt.Printf("New Access Token (first 50): %s...\n", newTokens.AccessToken[:50])
    
    // ลอง refresh อีกครั้งด้วย token เดิม (ควร fail เพราะ rotation)
    _, err = refreshAccessToken(tokens.RefreshToken)
    if err != nil {
        fmt.Println("\nExpected error (token rotation):", err)
    }
    
    // Logout: revoke ทุก tokens
    revokeAllUserTokens("user-123")
}
```

---

## 34.5 Authentication Middleware สำหรับ Gin

### Complete Auth Middleware (ตัวอย่างที่ 6)

```go
package main

import (
    "net/http"
    "strings"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

var accessSecret = []byte("access-token-secret-minimum-32-chars!!")

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID, email, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(accessSecret)
}

func parseToken(tokenStr string) (*Claims, error) {
    claims := &Claims{}
    _, err := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
        return accessSecret, nil
    })
    return claims, err
}

// ContextKeys สำหรับเก็บ user info
type contextKey string
const (
    UserIDKey   contextKey = "user_id"
    UserRoleKey contextKey = "user_role"
    ClaimsKey   contextKey = "claims"
)

// AuthMiddleware ตรวจสอบ JWT token
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authorization header is required",
                "code":  "AUTH_REQUIRED",
            })
            return
        }
        
        if !strings.HasPrefix(authHeader, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid authorization format. Use: Bearer <token>",
                "code":  "INVALID_FORMAT",
            })
            return
        }
        
        tokenStr := authHeader[7:]
        
        claims, err := parseToken(tokenStr)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid or expired token",
                "code":  "INVALID_TOKEN",
            })
            return
        }
        
        // เก็บ user info ใน Gin context
        c.Set(string(UserIDKey), claims.UserID)
        c.Set(string(UserRoleKey), claims.Role)
        c.Set(string(ClaimsKey), claims)
        
        c.Next()
    }
}

// RequireRoles ตรวจสอบ role
func RequireRoles(roles ...string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole, _ := c.Get(string(UserRoleKey))
        
        for _, role := range roles {
            if role == userRole {
                c.Next()
                return
            }
        }
        
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "Insufficient permissions",
            "code":  "FORBIDDEN",
        })
    }
}

// OptionalAuth - ไม่บังคับ auth แต่ถ้ามีก็ parse
func OptionalAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader != "" && strings.HasPrefix(authHeader, "Bearer ") {
            tokenStr := authHeader[7:]
            if claims, err := parseToken(tokenStr); err == nil {
                c.Set(string(UserIDKey), claims.UserID)
                c.Set(string(UserRoleKey), claims.Role)
            }
        }
        c.Next()
    }
}

// GetCurrentUser helper
func GetCurrentUserID(c *gin.Context) string {
    userID, _ := c.Get(string(UserIDKey))
    if id, ok := userID.(string); ok {
        return id
    }
    return ""
}

func main() {
    r := gin.Default()
    
    // Login endpoint
    r.POST("/auth/login", func(c *gin.Context) {
        var req struct {
            Email    string `json:"email" binding:"required,email"`
            Password string `json:"password" binding:"required"`
        }
        
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        // Simulate authentication (ในจริงตรวจจาก DB)
        var role string
        if req.Email == "admin@example.com" {
            role = "admin"
        } else {
            role = "user"
        }
        
        token, err := generateToken("user-123", req.Email, role)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate token"})
            return
        }
        
        c.JSON(http.StatusOK, gin.H{
            "token":   token,
            "role":    role,
            "expires": time.Now().Add(time.Hour),
        })
    })
    
    // Public endpoint with optional auth
    r.GET("/api/posts", OptionalAuth(), func(c *gin.Context) {
        userID := GetCurrentUserID(c)
        if userID != "" {
            c.JSON(http.StatusOK, gin.H{
                "posts":   []string{"Post 1", "Post 2", "Your draft post"},
                "user_id": userID,
            })
        } else {
            c.JSON(http.StatusOK, gin.H{
                "posts": []string{"Post 1", "Post 2"},
            })
        }
    })
    
    // Protected endpoints
    protected := r.Group("/api")
    protected.Use(AuthMiddleware())
    {
        protected.GET("/profile", func(c *gin.Context) {
            userID := GetCurrentUserID(c)
            c.JSON(http.StatusOK, gin.H{
                "user_id": userID,
                "profile": "User Profile Data",
            })
        })
        
        // Admin only
        admin := protected.Group("/admin")
        admin.Use(RequireRoles("admin"))
        {
            admin.GET("/users", func(c *gin.Context) {
                c.JSON(http.StatusOK, gin.H{
                    "users": []string{"User 1", "User 2", "User 3"},
                })
            })
            
            admin.DELETE("/users/:id", func(c *gin.Context) {
                c.JSON(http.StatusOK, gin.H{
                    "message": "User deleted",
                    "deleted_by": GetCurrentUserID(c),
                })
            })
        }
        
        // Manager or Admin
        management := protected.Group("/management")
        management.Use(RequireRoles("admin", "manager"))
        {
            management.GET("/reports", func(c *gin.Context) {
                c.JSON(http.StatusOK, gin.H{"reports": "Management reports"})
            })
        }
    }
    
    r.Run(":8080")
}
```

---

## 34.6 Role-Based Access Control (RBAC)

### RBAC System (ตัวอย่างที่ 7)

```go
package main

import (
    "fmt"
    "net/http"
    
    "github.com/gin-gonic/gin"
)

// Permissions
type Permission string

const (
    PermReadUsers   Permission = "users:read"
    PermWriteUsers  Permission = "users:write"
    PermDeleteUsers Permission = "users:delete"
    PermReadPosts   Permission = "posts:read"
    PermWritePosts  Permission = "posts:write"
    PermDeletePosts Permission = "posts:delete"
    PermManageAll   Permission = "admin:all"
)

// Role definitions
type Role struct {
    Name        string
    Permissions []Permission
}

var roles = map[string]Role{
    "guest": {
        Name:        "guest",
        Permissions: []Permission{PermReadPosts},
    },
    "user": {
        Name: "user",
        Permissions: []Permission{
            PermReadPosts, PermWritePosts,
        },
    },
    "moderator": {
        Name: "moderator",
        Permissions: []Permission{
            PermReadPosts, PermWritePosts, PermDeletePosts,
            PermReadUsers,
        },
    },
    "admin": {
        Name: "admin",
        Permissions: []Permission{
            PermReadUsers, PermWriteUsers, PermDeleteUsers,
            PermReadPosts, PermWritePosts, PermDeletePosts,
            PermManageAll,
        },
    },
}

type RBAC struct {
    roles map[string]Role
}

func NewRBAC() *RBAC {
    return &RBAC{roles: roles}
}

func (r *RBAC) HasPermission(roleName string, permission Permission) bool {
    role, ok := r.roles[roleName]
    if !ok {
        return false
    }
    
    for _, p := range role.Permissions {
        if p == permission || p == PermManageAll {
            return true
        }
    }
    return false
}

func (r *RBAC) GetPermissions(roleName string) []Permission {
    if role, ok := r.roles[roleName]; ok {
        return role.Permissions
    }
    return nil
}

// RBAC Middleware
func RequirePermission(rbac *RBAC, permission Permission) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole, exists := c.Get("user_role")
        if !exists {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authentication required",
            })
            return
        }
        
        role, ok := userRole.(string)
        if !ok || !rbac.HasPermission(role, permission) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error":      "Insufficient permissions",
                "required":   string(permission),
                "your_role":  role,
            })
            return
        }
        
        c.Next()
    }
}

func main() {
    rbac := NewRBAC()
    
    // Test RBAC
    fmt.Println("=== RBAC Testing ===")
    testCases := []struct {
        role       string
        permission Permission
    }{
        {"user", PermReadPosts},
        {"user", PermDeleteUsers},
        {"admin", PermDeleteUsers},
        {"moderator", PermManageAll},
    }
    
    for _, tc := range testCases {
        result := rbac.HasPermission(tc.role, tc.permission)
        fmt.Printf("Role '%s' has permission '%s': %v\n", tc.role, tc.permission, result)
    }
    
    // Gin setup with RBAC
    r := gin.Default()
    
    // Simulate auth (set role in context)
    r.Use(func(c *gin.Context) {
        // ในจริงตรวจ JWT แล้ว set role
        role := c.GetHeader("X-User-Role")
        if role == "" {
            role = "guest"
        }
        c.Set("user_role", role)
        c.Next()
    })
    
    r.GET("/posts", RequirePermission(rbac, PermReadPosts), func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"posts": []string{"Post 1", "Post 2"}})
    })
    
    r.POST("/posts", RequirePermission(rbac, PermWritePosts), func(c *gin.Context) {
        c.JSON(http.StatusCreated, gin.H{"message": "Post created"})
    })
    
    r.DELETE("/users/:id", RequirePermission(rbac, PermDeleteUsers), func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "User deleted"})
    })
    
    r.Run(":8080")
}
```

---

## Workshop: Complete Auth API

```go
package main

import (
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "log"
    "net/http"
    "strings"
    "sync"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

// === Configuration ===
var (
    accessSecret  = []byte("access-secret-key-minimum-32-chars!!")
    refreshSecret = []byte("refresh-secret-key-minimum-32-chars!")
)

// === Models ===
type User struct {
    ID           string    `json:"id"`
    Name         string    `json:"name"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    Role         string    `json:"role"`
    Active       bool      `json:"active"`
    CreatedAt    time.Time `json:"created_at"`
    LastLoginAt  *time.Time `json:"last_login_at,omitempty"`
}

type RefreshToken struct {
    ID        string
    UserID    string
    ExpiresAt time.Time
}

// === In-memory stores (ใน production ใช้ DB) ===
type UserStore struct {
    mu    sync.RWMutex
    users map[string]*User // email -> user
    byID  map[string]*User // id -> user
}

func NewUserStore() *UserStore {
    s := &UserStore{
        users: make(map[string]*User),
        byID:  make(map[string]*User),
    }
    // Seed admin user
    password := "Admin@123456"
    hash, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    admin := &User{
        ID: "admin-001", Name: "Administrator",
        Email: "admin@example.com", PasswordHash: string(hash),
        Role: "admin", Active: true, CreatedAt: time.Now(),
    }
    s.users[admin.Email] = admin
    s.byID[admin.ID] = admin
    return s
}

func (s *UserStore) GetByEmail(email string) (*User, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    u, ok := s.users[strings.ToLower(email)]
    return u, ok
}

func (s *UserStore) GetByID(id string) (*User, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    u, ok := s.byID[id]
    return u, ok
}

func (s *UserStore) Create(user *User) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    email := strings.ToLower(user.Email)
    if _, exists := s.users[email]; exists {
        return fmt.Errorf("email already registered")
    }
    s.users[email] = user
    s.byID[user.ID] = user
    return nil
}

func (s *UserStore) UpdateLastLogin(id string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if u, ok := s.byID[id]; ok {
        now := time.Now()
        u.LastLoginAt = &now
    }
}

type RefreshTokenStore struct {
    mu     sync.RWMutex
    tokens map[string]*RefreshToken
}

func NewRefreshTokenStore() *RefreshTokenStore {
    return &RefreshTokenStore{tokens: make(map[string]*RefreshToken)}
}

func (s *RefreshTokenStore) Add(token *RefreshToken) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.tokens[token.ID] = token
}

func (s *RefreshTokenStore) Get(id string) (*RefreshToken, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    t, ok := s.tokens[id]
    return t, ok
}

func (s *RefreshTokenStore) Revoke(id string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.tokens, id)
}

func (s *RefreshTokenStore) RevokeAllForUser(userID string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for id, t := range s.tokens {
        if t.UserID == userID {
            delete(s.tokens, id)
        }
    }
}

// === JWT Claims ===
type AccessClaims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

type RefreshClaims struct {
    UserID string `json:"user_id"`
    jwt.RegisteredClaims
}

// === Auth Service ===
type AuthService struct {
    users         *UserStore
    refreshTokens *RefreshTokenStore
}

func NewAuthService() *AuthService {
    return &AuthService{
        users:         NewUserStore(),
        refreshTokens: NewRefreshTokenStore(),
    }
}

func generateID() string {
    b := make([]byte, 8)
    rand.Read(b)
    return hex.EncodeToString(b)
}

func (s *AuthService) Register(name, email, password string) (*User, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return nil, fmt.Errorf("hashing password: %w", err)
    }
    
    user := &User{
        ID:           "user-" + generateID(),
        Name:         name,
        Email:        strings.ToLower(email),
        PasswordHash: string(hash),
        Role:         "user",
        Active:       true,
        CreatedAt:    time.Now(),
    }
    
    if err := s.users.Create(user); err != nil {
        return nil, err
    }
    
    return user, nil
}

func (s *AuthService) Login(email, password string) (string, string, error) {
    user, ok := s.users.GetByEmail(strings.ToLower(email))
    if !ok {
        return "", "", fmt.Errorf("invalid credentials")
    }
    
    if !user.Active {
        return "", "", fmt.Errorf("account is inactive")
    }
    
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password)); err != nil {
        return "", "", fmt.Errorf("invalid credentials")
    }
    
    // สร้าง access token
    accessToken, err := s.createAccessToken(user)
    if err != nil {
        return "", "", err
    }
    
    // สร้าง refresh token
    refreshToken, err := s.createRefreshToken(user)
    if err != nil {
        return "", "", err
    }
    
    s.users.UpdateLastLogin(user.ID)
    
    return accessToken, refreshToken, nil
}

func (s *AuthService) createAccessToken(user *User) (string, error) {
    claims := AccessClaims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(accessSecret)
}

func (s *AuthService) createRefreshToken(user *User) (string, error) {
    tokenID := generateID()
    
    claims := RefreshClaims{
        UserID: user.ID,
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        tokenID,
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    tokenString, err := token.SignedString(refreshSecret)
    if err != nil {
        return "", err
    }
    
    s.refreshTokens.Add(&RefreshToken{
        ID:        tokenID,
        UserID:    user.ID,
        ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
    })
    
    return tokenString, nil
}

func (s *AuthService) RefreshToken(refreshTokenStr string) (string, string, error) {
    claims := &RefreshClaims{}
    _, err := jwt.ParseWithClaims(refreshTokenStr, claims, func(t *jwt.Token) (interface{}, error) {
        return refreshSecret, nil
    })
    if err != nil {
        return "", "", fmt.Errorf("invalid refresh token")
    }
    
    stored, ok := s.refreshTokens.Get(claims.ID)
    if !ok || stored.UserID != claims.UserID {
        return "", "", fmt.Errorf("refresh token revoked or invalid")
    }
    
    if time.Now().After(stored.ExpiresAt) {
        return "", "", fmt.Errorf("refresh token expired")
    }
    
    user, ok := s.users.GetByID(claims.UserID)
    if !ok {
        return "", "", fmt.Errorf("user not found")
    }
    
    // Revoke old token (rotation)
    s.refreshTokens.Revoke(claims.ID)
    
    // Issue new token pair
    accessToken, err := s.createAccessToken(user)
    if err != nil {
        return "", "", err
    }
    
    newRefreshToken, err := s.createRefreshToken(user)
    if err != nil {
        return "", "", err
    }
    
    return accessToken, newRefreshToken, nil
}

func (s *AuthService) Logout(userID, refreshTokenStr string) error {
    claims := &RefreshClaims{}
    jwt.ParseWithClaims(refreshTokenStr, claims, func(t *jwt.Token) (interface{}, error) {
        return refreshSecret, nil
    })
    if claims.ID != "" {
        s.refreshTokens.Revoke(claims.ID)
    }
    return nil
}

func (s *AuthService) ValidateAccessToken(tokenStr string) (*AccessClaims, error) {
    claims := &AccessClaims{}
    _, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return accessSecret, nil
    })
    if err != nil {
        return nil, err
    }
    return claims, nil
}

// === Handlers ===
type AuthHandler struct {
    service *AuthService
}

func NewAuthHandler(service *AuthService) *AuthHandler {
    return &AuthHandler{service: service}
}

type RegisterRequest struct {
    Name     string `json:"name" binding:"required,min=2"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

type LoginRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

type RefreshRequest struct {
    RefreshToken string `json:"refresh_token" binding:"required"`
}

func (h *AuthHandler) Register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    user, err := h.service.Register(req.Name, req.Email, req.Password)
    if err != nil {
        c.JSON(http.StatusConflict, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, gin.H{
        "message": "Registration successful",
        "user": gin.H{
            "id":    user.ID,
            "name":  user.Name,
            "email": user.Email,
            "role":  user.Role,
        },
    })
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    accessToken, refreshToken, err := h.service.Login(req.Email, req.Password)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "access_token":  accessToken,
        "refresh_token": refreshToken,
        "token_type":    "Bearer",
        "expires_in":    900, // 15 minutes
    })
}

func (h *AuthHandler) Refresh(c *gin.Context) {
    var req RefreshRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    accessToken, refreshToken, err := h.service.RefreshToken(req.RefreshToken)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "access_token":  accessToken,
        "refresh_token": refreshToken,
        "token_type":    "Bearer",
    })
}

func (h *AuthHandler) Logout(c *gin.Context) {
    userID, _ := c.Get("user_id")
    
    authHeader := c.GetHeader("Authorization")
    var refreshToken string
    if body := struct{ RefreshToken string `json:"refresh_token"` }{}; c.ShouldBindJSON(&body) == nil {
        refreshToken = body.RefreshToken
    }
    
    h.service.Logout(userID.(string), refreshToken)
    
    c.JSON(http.StatusOK, gin.H{"message": "Logged out successfully"})
}

func (h *AuthHandler) Me(c *gin.Context) {
    userID, _ := c.Get("user_id")
    user, ok := h.service.users.GetByID(userID.(string))
    if !ok {
        c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
        return
    }
    c.JSON(http.StatusOK, gin.H{
        "id":            user.ID,
        "name":          user.Name,
        "email":         user.Email,
        "role":          user.Role,
        "created_at":    user.CreatedAt,
        "last_login_at": user.LastLoginAt,
    })
}

// Auth middleware
func (h *AuthHandler) AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if !strings.HasPrefix(authHeader, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Authorization required"})
            return
        }
        
        claims, err := h.service.ValidateAccessToken(authHeader[7:])
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            return
        }
        
        c.Set("user_id", claims.UserID)
        c.Set("user_email", claims.Email)
        c.Set("user_role", claims.Role)
        c.Next()
    }
}

func main() {
    service := NewAuthService()
    handler := NewAuthHandler(service)
    
    r := gin.Default()
    
    // Auth routes
    auth := r.Group("/auth")
    {
        auth.POST("/register", handler.Register)
        auth.POST("/login", handler.Login)
        auth.POST("/refresh", handler.Refresh)
        auth.POST("/logout", handler.AuthRequired(), handler.Logout)
        auth.GET("/me", handler.AuthRequired(), handler.Me)
    }
    
    // Protected routes
    api := r.Group("/api")
    api.Use(handler.AuthRequired())
    {
        api.GET("/dashboard", func(c *gin.Context) {
            c.JSON(http.StatusOK, gin.H{
                "message": "Welcome to dashboard!",
                "user_id": c.GetString("user_id"),
                "role":    c.GetString("user_role"),
            })
        })
    }
    
    log.Println("Auth API running on :8080")
    log.Println("Try: POST /auth/register with {\"name\":\"Test\",\"email\":\"test@example.com\",\"password\":\"Test@123456\"}")
    log.Println("Try: POST /auth/login with {\"email\":\"admin@example.com\",\"password\":\"Admin@123456\"}")
    r.Run(":8080")
}
```

---

## สรุป Part 34

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| bcrypt | `GenerateFromPassword()`, `CompareHashAndPassword()` |
| JWT | Header.Payload.Signature, signed with secret |
| Access Token | Short-lived (15min), store in memory |
| Refresh Token | Long-lived (7d), store in DB/Redis |
| Token Rotation | Revoke old refresh token เมื่อ refresh |
| RBAC | Role + Permissions + Middleware |
| Auth Middleware | Parse JWT, set user in context |

### Security Best Practices
- ใช้ HTTPS เท่านั้น
- Access token: เก็บใน memory (JS), ไม่ใส่ใน localStorage
- Refresh token: HttpOnly Cookie
- ตรวจ expiry ทุก request
- Rate limit auth endpoints
- Log failed login attempts

### Resources
- [golang-jwt/jwt](https://github.com/golang-jwt/jwt)
- [golang.org/x/crypto/bcrypt](https://pkg.go.dev/golang.org/x/crypto/bcrypt)
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
