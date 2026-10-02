# Part 15: Packages and Modules ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ package system ของ Go
- สร้างและใช้ packages ได้
- เข้าใจ exported vs unexported names
- ใช้ Go modules (go.mod, go.sum)
- ใช้ semantic versioning
- ใช้ go get, go mod tidy
- เข้าใจ init() function
- สร้าง internal packages
- หลีกเลี่ยง circular imports
- รู้จัก popular packages ecosystem

---

## 15.1 Package System Overview

ใน Go ทุก file ต้องอยู่ใน package หนึ่ง Package คือ unit ของ code organization

```
myproject/
├── main.go          (package main)
├── go.mod
├── go.sum
├── user/
│   ├── user.go      (package user)
│   └── user_test.go
├── auth/
│   ├── auth.go      (package auth)
│   └── token.go     (package auth)
└── utils/
    └── string.go    (package utils)
```

```go
// main.go
package main

import (
    "fmt"
    "myproject/user"
    "myproject/auth"
)

func main() {
    u := user.New("สมชาย", "somchai@example.com")
    token, err := auth.GenerateToken(u.ID)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("User: %s, Token: %s\n", u.Name, token)
}
```

---

## 15.2 สร้าง Package

### Package Declaration

```go
// ไฟล์ทุกไฟล์ในโฟลเดอร์เดียวกันต้องใช้ package ชื่อเดียวกัน

// math/calculator.go
package math

// math/geometry.go
package math  // ชื่อเดียวกัน

// main.go
package main  // package หลัก
```

### ตัวอย่าง: Package mathutil

```go
// mathutil/mathutil.go
package mathutil

import "math"

// Exported functions (ขึ้นต้นด้วยตัวใหญ่)

// Abs returns absolute value
func Abs(x float64) float64 {
    if x < 0 {
        return -x
    }
    return x
}

// Max returns maximum of two values
func Max(a, b float64) float64 {
    if a > b {
        return a
    }
    return b
}

// Min returns minimum of two values
func Min(a, b float64) float64 {
    if a < b {
        return a
    }
    return b
}

// Clamp constrains value between min and max
func Clamp(val, min, max float64) float64 {
    if val < min {
        return min
    }
    if val > max {
        return max
    }
    return val
}

// Sqrt returns square root, or 0 for negative numbers
func Sqrt(x float64) float64 {
    if x < 0 {
        return 0
    }
    return math.Sqrt(x)
}

// unexported helper function (ขึ้นต้นด้วยตัวเล็ก)
func roundToDecimal(val float64, places int) float64 {
    pow := math.Pow(10, float64(places))
    return math.Round(val*pow) / pow
}

// Round rounds to specified decimal places
func Round(val float64, places int) float64 {
    return roundToDecimal(val, places)
}
```

### ตัวอย่าง: Package stringutil

```go
// stringutil/stringutil.go
package stringutil

import (
    "strings"
    "unicode"
)

// Reverse reverses a string
func Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// Capitalize capitalizes first letter of each word
func Capitalize(s string) string {
    return strings.Title(strings.ToLower(s))
}

// IsPalindrome checks if string is palindrome
func IsPalindrome(s string) bool {
    s = strings.ToLower(s)
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        if runes[i] != runes[j] {
            return false
        }
    }
    return true
}

// WordCount counts words in string
func WordCount(s string) map[string]int {
    counts := make(map[string]int)
    words := strings.Fields(s)
    for _, w := range words {
        word := strings.ToLower(strings.TrimFunc(w, func(r rune) bool {
            return !unicode.IsLetter(r) && !unicode.IsNumber(r)
        }))
        if word != "" {
            counts[word]++
        }
    }
    return counts
}

// Truncate truncates string to maxLen with suffix
func Truncate(s string, maxLen int, suffix string) string {
    if len([]rune(s)) <= maxLen {
        return s
    }
    runes := []rune(s)
    return string(runes[:maxLen-len([]rune(suffix))]) + suffix
}

// IsEmpty returns true if string is empty or whitespace only
func IsEmpty(s string) bool {
    return strings.TrimSpace(s) == ""
}
```

---

## 15.3 Exported vs Unexported Names

ใน Go การ export ทำด้วยการขึ้นต้นด้วย **ตัวอักษรใหญ่**

```go
package user

import (
    "fmt"
    "time"
)

// ====== Exported ======

// User is an exported type
type User struct {
    ID        int       // exported field
    Name      string    // exported field
    Email     string    // exported field
    CreatedAt time.Time // exported field
    active    bool      // unexported field - เฉพาะ package user เท่านั้น
    password  string    // unexported field
}

// New is an exported constructor
func New(name, email string) *User {
    return &User{
        ID:        generateID(),
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
        active:    true,
    }
}

// IsActive is an exported method
func (u *User) IsActive() bool {
    return u.active
}

// SetPassword is an exported method
func (u *User) SetPassword(password string) {
    u.password = hashPassword(password)
}

// Activate is an exported method
func (u *User) Activate() {
    u.active = true
}

// Deactivate is an exported method  
func (u *User) Deactivate() {
    u.active = false
}

func (u User) String() string {
    return fmt.Sprintf("User{ID:%d, Name:%s, Email:%s}", u.ID, u.Name, u.Email)
}

// ====== Unexported ======

var nextID = 1

func generateID() int {
    id := nextID
    nextID++
    return id
}

func hashPassword(password string) string {
    // simplified hash
    return fmt.Sprintf("hashed:%s", password)
}
```

---

## 15.4 init() Function

`init()` รันโดยอัตโนมัติก่อน `main()` ใช้สำหรับ setup ต่างๆ

```go
// config/config.go
package config

import (
    "fmt"
    "os"
)

var (
    AppName    string
    AppVersion string
    Debug      bool
    DBUrl      string
)

// init() รันอัตโนมัติเมื่อ package ถูก import
func init() {
    AppName = getEnvOrDefault("APP_NAME", "MyApp")
    AppVersion = getEnvOrDefault("APP_VERSION", "1.0.0")
    Debug = os.Getenv("DEBUG") == "true"
    DBUrl = getEnvOrDefault("DB_URL", "postgres://localhost:5432/mydb")
    
    fmt.Printf("Config initialized: %s v%s (debug=%v)\n", AppName, AppVersion, Debug)
}

func getEnvOrDefault(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}
```

### Multiple init() functions

```go
// database/db.go
package database

import "fmt"

var connection string

func init() {
    fmt.Println("database: init 1 - checking environment")
    connection = "localhost:5432"
}

func init() {
    // Go อนุญาต init() หลายตัวในไฟล์เดียวกัน
    fmt.Println("database: init 2 - registering drivers")
}

// ลำดับ init():
// 1. imported packages (recursive)
// 2. package-level variables
// 3. init() functions (ตามลำดับในไฟล์)
```

### init() Order

```go
package main

import "fmt"

// Global variables initialized first
var x = initX()

func initX() int {
    fmt.Println("initializing x")
    return 10
}

func init() {
    fmt.Println("init() 1")
}

func init() {
    fmt.Println("init() 2")
}

func main() {
    fmt.Println("main()")
    fmt.Println("x =", x)
}

// Output:
// initializing x
// init() 1
// init() 2
// main()
// x = 10
```

---

## 15.5 Go Modules

Go modules คือระบบจัดการ dependencies ของ Go

### สร้าง Module ใหม่

```bash
# สร้าง directory และ initialize module
mkdir myapp
cd myapp
go mod init github.com/username/myapp
```

### go.mod

```
module github.com/username/myapp

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-redis/redis/v9 v9.3.0
    gorm.io/gorm v1.25.5
    gorm.io/driver/postgres v1.5.4
)

require (
    // indirect dependencies (ถูก import โดย dependencies อื่น)
    github.com/bytedance/sonic v1.10.0 // indirect
    github.com/chenzhuoyu/base64x v0.0.0-20230717121745-296ad89f973d // indirect
    // ... more indirect deps
)
```

### go.sum

```
# go.sum เก็บ checksums เพื่อ verify integrity
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPys6S2REjD9s3kqITozp1ycEPIv8EsTMTOebfiAiEw=
```

---

## 15.6 Semantic Versioning

Go modules ใช้ semantic versioning: `MAJOR.MINOR.PATCH`

```
v1.2.3
 │ │ └── PATCH: bug fixes (backward compatible)
 │ └──── MINOR: new features (backward compatible)
 └────── MAJOR: breaking changes
```

### ตัวอย่าง Versioning

```bash
# ติดตั้ง latest
go get github.com/some/package

# ติดตั้ง specific version
go get github.com/some/package@v1.2.3

# ติดตั้ง latest minor/patch
go get github.com/some/package@v1

# ติดตั้ง version range
go get github.com/some/package@latest

# Major version v2+ ต้องมี /v2 ใน import path
import "github.com/some/package/v2"
```

### Version ใน go.mod

```go
// go.mod
module myapp

go 1.21

require (
    github.com/pkg/errors v0.9.1        // stable release
    golang.org/x/text v0.14.0           // stable
    github.com/some/beta v1.0.0-beta.1  // pre-release
)
```

---

## 15.7 go get Commands

```bash
# เพิ่ม dependency ใหม่
go get github.com/gorilla/mux

# อัปเดตเป็น latest
go get -u github.com/gorilla/mux

# อัปเดต patch versions เท่านั้น
go get -u=patch github.com/gorilla/mux

# ลบ dependency ที่ไม่ใช้แล้ว
go mod tidy

# ดู dependencies ทั้งหมด
go list -m all

# ดู versions ที่ available
go list -m -versions github.com/gorilla/mux

# Download dependencies โดยไม่ install
go mod download

# Vendor mode (copy deps เข้า vendor folder)
go mod vendor
go build -mod=vendor
```

---

## 15.8 go mod Commands

```bash
# Initialize module
go mod init module-name

# Tidy - ลบ unused, เพิ่ม missing
go mod tidy

# Verify checksums
go mod verify

# Download all dependencies
go mod download

# Create vendor directory
go mod vendor

# Show dependency graph
go mod graph

# Edit go.mod
go mod edit -require github.com/pkg/errors@v0.9.1
go mod edit -droprequire github.com/old/package

# Why is a package needed?
go mod why github.com/some/package
```

---

## 15.9 Internal Packages

`internal` package สามารถ import ได้เฉพาะ packages ที่อยู่ใน parent directory เดียวกัน

```
myapp/
├── main.go
├── go.mod
├── api/
│   ├── handler.go      (สามารถ import myapp/internal/...)
│   └── middleware.go
├── internal/
│   ├── database/
│   │   └── db.go       (เข้าถึงได้เฉพาะ myapp/...)
│   ├── auth/
│   │   └── jwt.go      (เข้าถึงได้เฉพาะ myapp/...)
│   └── config/
│       └── config.go
└── pkg/
    └── utils/
        └── helpers.go  (เข้าถึงได้จากภายนอก)
```

```go
// internal/database/db.go
package database

import "fmt"

type DB struct {
    url string
}

func New(url string) *DB {
    return &DB{url: url}
}

func (db *DB) Connect() error {
    fmt.Printf("Connecting to %s\n", db.url)
    return nil
}

// internal/auth/jwt.go
package auth

import (
    "fmt"
    "time"
)

type Claims struct {
    UserID    int
    ExpiresAt time.Time
}

func GenerateToken(userID int) (string, error) {
    claims := Claims{
        UserID:    userID,
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }
    return fmt.Sprintf("token.%d.%d", claims.UserID, claims.ExpiresAt.Unix()), nil
}

func ValidateToken(token string) (*Claims, error) {
    if token == "" {
        return nil, fmt.Errorf("empty token")
    }
    return &Claims{UserID: 1}, nil
}
```

---

## 15.10 Circular Imports

Go ไม่อนุญาต circular imports - ต้องออกแบบ structure ให้ดี

```
# ไม่ดี - circular import!
package a imports package b
package b imports package a  <- ERROR!

# วิธีแก้:
# 1. แยก shared code เป็น package ใหม่
# 2. ใช้ interface แทน concrete type
# 3. ย้าย code ไปอยู่ใน package เดียวกัน
```

```go
// วิธีหลีกเลี่ยง circular imports ด้วย interface

// types/types.go - shared types
package types

type UserID int

type User struct {
    ID   UserID
    Name string
    Email string
}

// order/order.go - ใช้ types จาก types package
package order

import "myapp/types"

type Order struct {
    ID       int
    UserID   types.UserID  // ใช้ shared type
    Products []string
    Total    float64
}

// notification/notification.go - ใช้ interface แทน concrete type
package notification

// ประกาศ interface ที่ต้องการ แทนการ import user package
type UserGetter interface {
    GetUser(id int) (string, error)
}

type Service struct {
    users UserGetter
}

func NewService(users UserGetter) *Service {
    return &Service{users: users}
}

func (s *Service) NotifyUser(userID int, message string) error {
    name, err := s.users.GetUser(userID)
    if err != nil {
        return err
    }
    // ส่ง notification ให้ name
    _ = name
    return nil
}
```

---

## 15.11 Private Modules

```bash
# ตั้งค่า GONOSUMCHECK สำหรับ private modules
export GONOSUMCHECK=github.com/company/*
export GOFLAGS=-mod=mod

# ตั้งค่า GONOSUMDB
export GONOSUMDB=github.com/company/*

# ตั้งค่า GOPRIVATE
export GOPRIVATE=github.com/company/*

# ใน go.mod
module github.com/company/private-app

require (
    github.com/company/private-lib v1.0.0
)

# replace directive สำหรับ local development
replace github.com/company/private-lib => ../private-lib
```

---

## 15.12 Popular Packages Ecosystem

### Standard Library

```go
package main

import (
    // Core
    "fmt"          // formatting
    "os"           // OS operations
    "io"           // I/O primitives
    "bufio"        // buffered I/O
    
    // Data processing
    "encoding/json"  // JSON
    "encoding/xml"   // XML
    "encoding/csv"   // CSV (via bufio)
    
    // Networking
    "net/http"       // HTTP client/server
    "net"            // TCP/UDP
    
    // Concurrency
    "sync"           // Mutex, WaitGroup
    "sync/atomic"    // Atomic operations
    
    // Strings
    "strings"        // string manipulation
    "strconv"        // string conversion
    "regexp"         // regular expressions
    
    // Math
    "math"           // math functions
    "math/rand"      // random numbers
    "math/big"       // big numbers
    
    // Time
    "time"           // time and duration
    
    // Errors
    "errors"         // error handling
    
    // Testing
    "testing"        // test framework
    "testing/quick"  // property-based testing
    
    // Crypto
    "crypto/sha256"  // SHA-256
    "crypto/rand"    // secure random
    
    // Compression
    "compress/gzip"  // gzip
    
    // Context
    "context"        // context for cancellation
    
    // Sort
    "sort"           // sorting
)

func main() {
    // ตัวอย่างใช้ standard library
    _ = fmt.Sprintf
    _ = os.ReadFile
}
```

### Popular Third-Party Packages

```go
// Web Frameworks
import (
    "github.com/gin-gonic/gin"          // Gin - fast HTTP framework
    "github.com/labstack/echo/v4"        // Echo - minimalist web framework
    "github.com/gorilla/mux"             // Gorilla Mux - router
    "github.com/go-chi/chi/v5"           // Chi - lightweight router
)

// Database
import (
    "gorm.io/gorm"                       // GORM - ORM
    "gorm.io/driver/postgres"
    "github.com/jmoiron/sqlx"            // sqlx - extensions for database/sql
    "github.com/go-redis/redis/v9"       // Redis client
    "go.mongodb.org/mongo-driver/mongo"  // MongoDB
)

// Config & Environment
import (
    "github.com/spf13/viper"             // Config management
    "github.com/joho/godotenv"           // .env file loader
    "github.com/spf13/cobra"             // CLI framework
)

// Validation
import (
    "github.com/go-playground/validator/v10" // Struct validation
)

// Logging
import (
    "go.uber.org/zap"                    // Zap - structured logging
    "github.com/sirupsen/logrus"         // Logrus - structured logging
    "github.com/rs/zerolog"              // Zerolog - zero allocation logging
)

// Testing
import (
    "github.com/stretchr/testify"        // Testify - assertions
    "github.com/golang/mock"             // Mock generation
)

// Utilities
import (
    "github.com/google/uuid"             // UUID generation
    "github.com/pkg/errors"              // Better error handling
    "golang.org/x/sync/errgroup"         // Error groups for goroutines
)
```

---

## 15.13 HTTP Server Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
)

// Domain types
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// Repository
type UserRepository struct {
    users  map[int]User
    nextID int
}

func NewUserRepository() *UserRepository {
    return &UserRepository{
        users:  make(map[int]User),
        nextID: 1,
    }
}

func (r *UserRepository) Create(name, email string) User {
    user := User{
        ID:    r.nextID,
        Name:  name,
        Email: email,
    }
    r.users[r.nextID] = user
    r.nextID++
    return user
}

func (r *UserRepository) FindAll() []User {
    users := make([]User, 0, len(r.users))
    for _, u := range r.users {
        users = append(users, u)
    }
    return users
}

func (r *UserRepository) FindByID(id int) (User, bool) {
    u, ok := r.users[id]
    return u, ok
}

// Handler
type UserHandler struct {
    repo *UserRepository
}

func (h *UserHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    // Simple routing
    path := strings.TrimPrefix(r.URL.Path, "/users")
    
    switch {
    case path == "" || path == "/":
        switch r.Method {
        case http.MethodGet:
            h.list(w, r)
        case http.MethodPost:
            h.create(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    default:
        // /users/{id}
        idStr := strings.TrimPrefix(path, "/")
        id, err := strconv.Atoi(idStr)
        if err != nil {
            http.Error(w, "Invalid ID", http.StatusBadRequest)
            return
        }
        h.getByID(w, r, id)
    }
}

func (h *UserHandler) list(w http.ResponseWriter, r *http.Request) {
    users := h.repo.FindAll()
    json.NewEncoder(w).Encode(users)
}

func (h *UserHandler) create(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    user := h.repo.Create(req.Name, req.Email)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) getByID(w http.ResponseWriter, r *http.Request, id int) {
    user, ok := h.repo.FindByID(id)
    if !ok {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    json.NewEncoder(w).Encode(user)
}

func main() {
    repo := NewUserRepository()
    
    // Seed data
    repo.Create("สมชาย ใจดี", "somchai@example.com")
    repo.Create("สมหญิง รักเรียน", "somying@example.com")
    
    handler := &UserHandler{repo: repo}
    
    mux := http.NewServeMux()
    mux.Handle("/users", handler)
    mux.Handle("/users/", handler)
    
    fmt.Println("Server starting at :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## Workshop: Create Reusable Package

สร้าง package `validator` ที่ reusable และ well-tested

### โครงสร้าง

```
validator/
├── go.mod
├── validator.go
├── rules.go
├── errors.go
├── validator_test.go
└── README.md
```

```go
// validator/go.mod
module github.com/username/validator

go 1.21
```

```go
// validator/errors.go
package validator

import (
    "fmt"
    "strings"
)

// ValidationError represents a single field validation error
type ValidationError struct {
    Field   string
    Rule    string
    Message string
    Value   interface{}
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("field '%s' failed '%s': %s", e.Field, e.Rule, e.Message)
}

// ValidationErrors is a collection of validation errors
type ValidationErrors []ValidationError

func (ve ValidationErrors) Error() string {
    msgs := make([]string, len(ve))
    for i, e := range ve {
        msgs[i] = e.Error()
    }
    return strings.Join(msgs, "; ")
}

func (ve ValidationErrors) HasErrors() bool {
    return len(ve) > 0
}

func (ve ValidationErrors) ForField(field string) []ValidationError {
    var errors []ValidationError
    for _, e := range ve {
        if e.Field == field {
            errors = append(errors, e)
        }
    }
    return errors
}

func (ve ValidationErrors) FirstMessage(field string) string {
    for _, e := range ve {
        if e.Field == field {
            return e.Message
        }
    }
    return ""
}
```

```go
// validator/rules.go
package validator

import (
    "fmt"
    "regexp"
    "strings"
    "unicode"
)

// RuleFunc is a validation function
type RuleFunc func(field string, value interface{}) *ValidationError

// Built-in rules

// Required validates field is not empty
func Required(field string, value interface{}) *ValidationError {
    var isEmpty bool
    switch v := value.(type) {
    case string:
        isEmpty = strings.TrimSpace(v) == ""
    case nil:
        isEmpty = true
    default:
        isEmpty = false
    }
    
    if isEmpty {
        return &ValidationError{
            Field:   field,
            Rule:    "required",
            Message: "ต้องไม่ว่าง",
            Value:   value,
        }
    }
    return nil
}

// MinLength validates minimum string length
func MinLength(min int) RuleFunc {
    return func(field string, value interface{}) *ValidationError {
        s, ok := value.(string)
        if !ok {
            return nil
        }
        if len([]rune(s)) < min {
            return &ValidationError{
                Field:   field,
                Rule:    "min_length",
                Message: fmt.Sprintf("ต้องมีอย่างน้อย %d ตัวอักษร", min),
                Value:   value,
            }
        }
        return nil
    }
}

// MaxLength validates maximum string length
func MaxLength(max int) RuleFunc {
    return func(field string, value interface{}) *ValidationError {
        s, ok := value.(string)
        if !ok {
            return nil
        }
        if len([]rune(s)) > max {
            return &ValidationError{
                Field:   field,
                Rule:    "max_length",
                Message: fmt.Sprintf("ต้องไม่เกิน %d ตัวอักษร", max),
                Value:   value,
            }
        }
        return nil
    }
}

// Email validates email format
func Email(field string, value interface{}) *ValidationError {
    s, ok := value.(string)
    if !ok {
        return nil
    }
    
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(s) {
        return &ValidationError{
            Field:   field,
            Rule:    "email",
            Message: "รูปแบบ email ไม่ถูกต้อง",
            Value:   value,
        }
    }
    return nil
}

// MinValue validates minimum numeric value
func MinValue(min int) RuleFunc {
    return func(field string, value interface{}) *ValidationError {
        var n int
        switch v := value.(type) {
        case int:
            n = v
        case int64:
            n = int(v)
        default:
            return nil
        }
        
        if n < min {
            return &ValidationError{
                Field:   field,
                Rule:    "min_value",
                Message: fmt.Sprintf("ต้องมีค่าอย่างน้อย %d", min),
                Value:   value,
            }
        }
        return nil
    }
}

// MaxValue validates maximum numeric value
func MaxValue(max int) RuleFunc {
    return func(field string, value interface{}) *ValidationError {
        var n int
        switch v := value.(type) {
        case int:
            n = v
        case int64:
            n = int(v)
        default:
            return nil
        }
        
        if n > max {
            return &ValidationError{
                Field:   field,
                Rule:    "max_value",
                Message: fmt.Sprintf("ต้องมีค่าไม่เกิน %d", max),
                Value:   value,
            }
        }
        return nil
    }
}

// Pattern validates against regex pattern
func Pattern(pattern, message string) RuleFunc {
    re := regexp.MustCompile(pattern)
    return func(field string, value interface{}) *ValidationError {
        s, ok := value.(string)
        if !ok {
            return nil
        }
        if !re.MatchString(s) {
            return &ValidationError{
                Field:   field,
                Rule:    "pattern",
                Message: message,
                Value:   value,
            }
        }
        return nil
    }
}

// AlphaNumeric validates alphanumeric string
func AlphaNumeric(field string, value interface{}) *ValidationError {
    s, ok := value.(string)
    if !ok {
        return nil
    }
    for _, r := range s {
        if !unicode.IsLetter(r) && !unicode.IsNumber(r) {
            return &ValidationError{
                Field:   field,
                Rule:    "alphanumeric",
                Message: "ต้องประกอบด้วยตัวอักษรและตัวเลขเท่านั้น",
                Value:   value,
            }
        }
    }
    return nil
}

// OneOf validates value is one of allowed values
func OneOf(allowed ...string) RuleFunc {
    allowedMap := make(map[string]bool)
    for _, a := range allowed {
        allowedMap[a] = true
    }
    
    return func(field string, value interface{}) *ValidationError {
        s, ok := value.(string)
        if !ok {
            return nil
        }
        if !allowedMap[s] {
            return &ValidationError{
                Field:   field,
                Rule:    "one_of",
                Message: fmt.Sprintf("ต้องเป็นหนึ่งใน: %s", strings.Join(allowed, ", ")),
                Value:   value,
            }
        }
        return nil
    }
}
```

```go
// validator/validator.go
package validator

// FieldRule defines validation rules for a single field
type FieldRule struct {
    Field string
    Value interface{}
    Rules []RuleFunc
}

// Builder builds validation rules
type Builder struct {
    fields []FieldRule
}

// New creates a new validator builder
func New() *Builder {
    return &Builder{}
}

// Field adds a field to validate
func (b *Builder) Field(name string, value interface{}, rules ...RuleFunc) *Builder {
    b.fields = append(b.fields, FieldRule{
        Field: name,
        Value: value,
        Rules: rules,
    })
    return b
}

// Validate runs all validations and returns errors
func (b *Builder) Validate() ValidationErrors {
    var errors ValidationErrors
    
    for _, f := range b.fields {
        for _, rule := range f.Rules {
            if err := rule(f.Field, f.Value); err != nil {
                errors = append(errors, *err)
            }
        }
    }
    
    return errors
}

// ValidateStruct validates using a simple map of field->value->rules
func ValidateStruct(rules map[string]struct {
    Value interface{}
    Rules []RuleFunc
}) ValidationErrors {
    b := New()
    for field, config := range rules {
        b.Field(field, config.Value, config.Rules...)
    }
    return b.Validate()
}
```

```go
// main.go - using the validator package
package main

import (
    "fmt"
    "github.com/username/validator"
)

type RegisterForm struct {
    Username string
    Email    string
    Password string
    Age      int
    Role     string
}

func validateRegister(form RegisterForm) error {
    errs := validator.New().
        Field("username", form.Username,
            validator.Required,
            validator.MinLength(3),
            validator.MaxLength(20),
            validator.AlphaNumeric,
        ).
        Field("email", form.Email,
            validator.Required,
            validator.Email,
        ).
        Field("password", form.Password,
            validator.Required,
            validator.MinLength(8),
            validator.MaxLength(100),
        ).
        Field("age", form.Age,
            validator.MinValue(13),
            validator.MaxValue(120),
        ).
        Field("role", form.Role,
            validator.OneOf("user", "admin", "moderator"),
        ).
        Validate()
    
    if errs.HasErrors() {
        return errs
    }
    return nil
}

func main() {
    // Test valid form
    validForm := RegisterForm{
        Username: "somchai123",
        Email:    "somchai@example.com",
        Password: "mypassword123",
        Age:      25,
        Role:     "user",
    }
    
    if err := validateRegister(validForm); err != nil {
        fmt.Println("Valid form errors:", err)
    } else {
        fmt.Println("Valid form: OK")
    }
    
    // Test invalid form
    invalidForm := RegisterForm{
        Username: "ab",
        Email:    "invalid-email",
        Password: "short",
        Age:      10,
        Role:     "superadmin",
    }
    
    if err := validateRegister(invalidForm); err != nil {
        fmt.Println("\nInvalid form errors:")
        if ve, ok := err.(validator.ValidationErrors); ok {
            for _, e := range ve {
                fmt.Printf("  - %s\n", e)
            }
        }
        
        // Get specific field error
        if ve, ok := err.(validator.ValidationErrors); ok {
            msg := ve.FirstMessage("username")
            fmt.Printf("\nUsername error: %s\n", msg)
        }
    }
}
```

---

## 15.14 Workspace Mode (go.work)

Go workspace mode ใช้สำหรับ develop หลาย modules พร้อมกัน

```bash
# สร้าง workspace
mkdir workspace && cd workspace
mkdir myapp mylib

# Initialize modules
cd myapp && go mod init github.com/user/myapp && cd ..
cd mylib && go mod init github.com/user/mylib && cd ..

# สร้าง workspace
go work init myapp mylib
```

```
# go.work
go 1.21

use (
    ./myapp
    ./mylib
)
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| Package | Unit of code organization |
| `package main` | Entry point ของโปรแกรม |
| Exported | ขึ้นต้นด้วยตัวใหญ่ - เข้าถึงได้จากภายนอก |
| Unexported | ขึ้นต้นด้วยตัวเล็ก - เฉพาะภายใน package |
| `init()` | รันอัตโนมัติก่อน main() |
| go.mod | กำหนด module path และ dependencies |
| go.sum | เก็บ checksums ของ dependencies |
| `go get` | เพิ่ม/อัปเดต dependency |
| `go mod tidy` | ทำความสะอาด dependencies |
| internal package | เข้าถึงได้เฉพาะภายใน subtree |
| Circular import | ไม่อนุญาต - ต้องออกแบบ structure ให้ดี |
| Semantic versioning | `MAJOR.MINOR.PATCH` |

## Resources

- [Go Blog: Using Go Modules](https://go.dev/blog/using-go-modules)
- [Go Modules Reference](https://go.dev/ref/mod)
- [Organizing Go Code](https://go.dev/doc/code)
- [Package naming](https://go.dev/blog/package-names)
- [Standard library](https://pkg.go.dev/std)
- [pkg.go.dev](https://pkg.go.dev) - Go package directory

---

*Part 15 จบแล้ว! ยินดีด้วย - คุณเรียน Go พื้นฐานครบแล้ว!*

## What's Next?

หลังจาก Part 15 คุณพร้อมเรียน:
- **Phase 2**: Advanced Go - Generics, Testing, Benchmarking, Build Tags
- **Phase 3**: Standard Library Deep Dive - HTTP, Database, File I/O
- **Phase 4**: Real-world Applications - REST API, CLI tools, Microservices
