# Part 12: Error Handling ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ error interface และวิธีใช้งาน
- สร้าง errors ด้วย `errors.New` และ `fmt.Errorf`
- ใช้ error wrapping ด้วย `%w`
- ใช้ `errors.Is` และ `errors.As` ได้อย่างถูกต้อง
- สร้าง custom error types
- จัดการ multiple errors
- เข้าใจความแตกต่างระหว่าง panic กับ error
- เขียน code ที่จัดการ error ได้ดี

---

## 12.1 Error Interface

ใน Go **error** คือ interface ที่เรียบง่าย:

```go
type error interface {
    Error() string
}
```

ทุก type ที่มี method `Error() string` คือ error

```go
package main

import (
    "errors"
    "fmt"
)

func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("ไม่สามารถหารด้วยศูนย์ได้")
    }
    return a / b, nil
}

func main() {
    // การใช้งาน error pattern พื้นฐาน
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("10 / 2 = %.2f\n", result)
    }
    
    result, err = divide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("ผลลัพธ์: %.2f\n", result)
    }
}
```

---

## 12.2 สร้าง Errors

### errors.New

```go
package main

import (
    "errors"
    "fmt"
)

var (
    ErrNotFound    = errors.New("ไม่พบข้อมูล")
    ErrPermission  = errors.New("ไม่มีสิทธิ์เข้าถึง")
    ErrInvalidInput = errors.New("ข้อมูลไม่ถูกต้อง")
)

func getUser(id int) (string, error) {
    users := map[int]string{
        1: "สมชาย",
        2: "สมหญิง",
        3: "สมศักดิ์",
    }
    
    if id <= 0 {
        return "", ErrInvalidInput
    }
    
    user, ok := users[id]
    if !ok {
        return "", ErrNotFound
    }
    
    return user, nil
}

func main() {
    ids := []int{1, 5, -1, 2}
    
    for _, id := range ids {
        user, err := getUser(id)
        if err != nil {
            fmt.Printf("ID %d: Error - %v\n", id, err)
        } else {
            fmt.Printf("ID %d: %s\n", id, user)
        }
    }
}
```

### fmt.Errorf

```go
package main

import (
    "fmt"
)

func validateAge(age int) error {
    if age < 0 {
        return fmt.Errorf("อายุ %d ไม่ถูกต้อง: ต้องไม่ติดลบ", age)
    }
    if age > 150 {
        return fmt.Errorf("อายุ %d ไม่ถูกต้อง: ค่าเกิน 150 ปี", age)
    }
    return nil
}

func validateEmail(email string) error {
    if len(email) == 0 {
        return fmt.Errorf("email ไม่สามารถว่างได้")
    }
    for _, c := range email {
        if c == '@' {
            return nil
        }
    }
    return fmt.Errorf("email '%s' ไม่มีเครื่องหมาย @", email)
}

func main() {
    ages := []int{25, -5, 200, 0}
    for _, age := range ages {
        if err := validateAge(age); err != nil {
            fmt.Printf("อายุ %d: %v\n", age, err)
        } else {
            fmt.Printf("อายุ %d: OK\n", age)
        }
    }
    
    emails := []string{"test@example.com", "invalid", ""}
    for _, email := range emails {
        if err := validateEmail(email); err != nil {
            fmt.Printf("Email '%s': %v\n", email, err)
        } else {
            fmt.Printf("Email '%s': OK\n", email)
        }
    }
}
```

---

## 12.3 Error Wrapping

Error wrapping ช่วยเพิ่มบริบทให้กับ error โดยยังเก็บ original error ไว้

```go
package main

import (
    "errors"
    "fmt"
)

var ErrDatabase = errors.New("database error")
var ErrNotFound = errors.New("not found")

func queryDB(id int) (string, error) {
    if id == 999 {
        return "", fmt.Errorf("queryDB failed: %w", ErrDatabase)
    }
    if id > 100 {
        return "", fmt.Errorf("user %d: %w", id, ErrNotFound)
    }
    return fmt.Sprintf("user-%d", id), nil
}

func getProfile(userID int) (string, error) {
    data, err := queryDB(userID)
    if err != nil {
        // Wrap error เพิ่ม context
        return "", fmt.Errorf("getProfile(id=%d): %w", userID, err)
    }
    return data, nil
}

func handleRequest(userID int) error {
    profile, err := getProfile(userID)
    if err != nil {
        return fmt.Errorf("handleRequest: %w", err)
    }
    fmt.Printf("Profile: %s\n", profile)
    return nil
}

func main() {
    ids := []int{1, 150, 999}
    
    for _, id := range ids {
        err := handleRequest(id)
        if err != nil {
            fmt.Printf("\nError for id=%d:\n", id)
            fmt.Printf("  Full: %v\n", err)
            
            // Unwrap error chain
            fmt.Println("  Chain:")
            e := err
            for e != nil {
                fmt.Printf("    -> %v\n", e)
                e = errors.Unwrap(e)
            }
        }
    }
}
```

---

## 12.4 errors.Is และ errors.As

### errors.Is - ตรวจสอบว่า error ตรงกับ sentinel error

```go
package main

import (
    "errors"
    "fmt"
)

var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrBadRequest    = errors.New("bad request")
)

func fetchData(key string) (string, error) {
    data := map[string]string{
        "public": "ข้อมูลสาธารณะ",
        "secret": "ข้อมูลลับ",
    }
    
    if key == "" {
        return "", fmt.Errorf("key ว่าง: %w", ErrBadRequest)
    }
    
    if key == "secret" {
        return "", fmt.Errorf("key '%s' ต้องมีสิทธิ์: %w", key, ErrUnauthorized)
    }
    
    val, ok := data[key]
    if !ok {
        return "", fmt.Errorf("key '%s': %w", key, ErrNotFound)
    }
    
    return val, nil
}

func main() {
    keys := []string{"public", "secret", "missing", ""}
    
    for _, key := range keys {
        data, err := fetchData(key)
        if err != nil {
            fmt.Printf("key=%q error: %v\n", key, err)
            
            // ตรวจสอบประเภท error ด้วย errors.Is
            switch {
            case errors.Is(err, ErrNotFound):
                fmt.Println("  -> แนะนำ: กรุณาตรวจสอบ key ที่ใช้")
            case errors.Is(err, ErrUnauthorized):
                fmt.Println("  -> แนะนำ: กรุณา login ก่อน")
            case errors.Is(err, ErrBadRequest):
                fmt.Println("  -> แนะนำ: ตรวจสอบ input ที่ส่งมา")
            }
        } else {
            fmt.Printf("key=%q data: %s\n", key, data)
        }
    }
}
```

### errors.As - ดึง concrete error type

```go
package main

import (
    "errors"
    "fmt"
    "net/http"
)

// Custom error types
type HTTPError struct {
    Code    int
    Message string
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP %d: %s", e.Code, e.Message)
}

type ValidationError struct {
    Field   string
    Value   interface{}
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s (value: %v)",
        e.Field, e.Message, e.Value)
}

func callAPI(endpoint string) error {
    if endpoint == "/admin" {
        return fmt.Errorf("API call failed: %w", &HTTPError{
            Code:    http.StatusForbidden,
            Message: "ไม่มีสิทธิ์เข้าถึง",
        })
    }
    if endpoint == "" {
        return fmt.Errorf("invalid request: %w", &ValidationError{
            Field:   "endpoint",
            Value:   endpoint,
            Message: "endpoint ต้องไม่ว่าง",
        })
    }
    return nil
}

func main() {
    endpoints := []string{"/api/users", "/admin", ""}
    
    for _, ep := range endpoints {
        err := callAPI(ep)
        if err != nil {
            fmt.Printf("Endpoint %q: %v\n", ep, err)
            
            // ใช้ errors.As เพื่อดึง concrete type
            var httpErr *HTTPError
            var valErr *ValidationError
            
            switch {
            case errors.As(err, &httpErr):
                fmt.Printf("  HTTP Error Code: %d\n", httpErr.Code)
                if httpErr.Code == http.StatusForbidden {
                    fmt.Println("  -> Redirect to login page")
                }
            case errors.As(err, &valErr):
                fmt.Printf("  Validation Field: %s\n", valErr.Field)
                fmt.Printf("  Validation Message: %s\n", valErr.Message)
            }
        } else {
            fmt.Printf("Endpoint %q: OK\n", ep)
        }
    }
}
```

---

## 12.5 Custom Error Types

```go
package main

import (
    "errors"
    "fmt"
    "strings"
    "time"
)

// Simple custom error
type AppError struct {
    Code    string
    Message string
    Err     error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// Multiple validation errors
type MultiError struct {
    Errors []error
}

func (me *MultiError) Error() string {
    msgs := make([]string, len(me.Errors))
    for i, err := range me.Errors {
        msgs[i] = err.Error()
    }
    return strings.Join(msgs, "; ")
}

func (me *MultiError) Add(err error) {
    if err != nil {
        me.Errors = append(me.Errors, err)
    }
}

func (me *MultiError) HasErrors() bool {
    return len(me.Errors) > 0
}

// Retryable error
type RetryableError struct {
    Cause     error
    RetryAt   time.Time
    MaxRetries int
    Attempt   int
}

func (e *RetryableError) Error() string {
    return fmt.Sprintf("retryable error (attempt %d/%d): %v, retry after %s",
        e.Attempt, e.MaxRetries, e.Cause, e.RetryAt.Format("15:04:05"))
}

func (e *RetryableError) Unwrap() error { return e.Cause }

func (e *RetryableError) ShouldRetry() bool {
    return e.Attempt < e.MaxRetries && time.Now().Before(e.RetryAt)
}

// ตัวอย่างการใช้
func validateRegistration(username, email, password string) error {
    me := &MultiError{}
    
    if len(username) < 3 {
        me.Add(fmt.Errorf("username ต้องมีอย่างน้อย 3 ตัวอักษร"))
    }
    if len(username) > 20 {
        me.Add(fmt.Errorf("username ต้องไม่เกิน 20 ตัวอักษร"))
    }
    if !strings.Contains(email, "@") {
        me.Add(fmt.Errorf("email ไม่ถูกต้อง"))
    }
    if len(password) < 8 {
        me.Add(fmt.Errorf("password ต้องมีอย่างน้อย 8 ตัวอักษร"))
    }
    
    if me.HasErrors() {
        return me
    }
    return nil
}

func main() {
    // AppError
    err := &AppError{
        Code:    "USER_404",
        Message: "ไม่พบผู้ใช้งาน",
        Err:     errors.New("database query returned 0 rows"),
    }
    fmt.Println("AppError:", err)
    
    // Unwrap
    var appErr *AppError
    if errors.As(err, &appErr) {
        fmt.Printf("Code: %s\n", appErr.Code)
    }
    
    fmt.Println()
    
    // MultiError
    testCases := []struct {
        username, email, password string
    }{
        {"ab", "invalid-email", "short"},
        {"validuser", "user@example.com", "password123"},
        {"a", "", "pass"},
    }
    
    for _, tc := range testCases {
        err := validateRegistration(tc.username, tc.email, tc.password)
        if err != nil {
            fmt.Printf("Validation failed for '%s':\n", tc.username)
            if me, ok := err.(*MultiError); ok {
                for _, e := range me.Errors {
                    fmt.Printf("  - %v\n", e)
                }
            }
        } else {
            fmt.Printf("'%s': validation passed\n", tc.username)
        }
        fmt.Println()
    }
    
    // RetryableError
    retryErr := &RetryableError{
        Cause:      errors.New("connection refused"),
        RetryAt:    time.Now().Add(5 * time.Second),
        MaxRetries: 3,
        Attempt:    1,
    }
    fmt.Println("RetryableError:", retryErr)
    fmt.Printf("Should retry: %v\n", retryErr.ShouldRetry())
}
```

---

## 12.6 Error Sentinel Values

```go
package main

import (
    "errors"
    "fmt"
    "io"
)

// Sentinel errors - ประกาศไว้ที่ package level
var (
    ErrDivisionByZero = errors.New("division by zero")
    ErrNegativeNumber = errors.New("negative number not allowed")
    ErrOverflow       = errors.New("arithmetic overflow")
    ErrInvalidOp      = errors.New("invalid operation")
)

type Calculator struct {
    history []string
}

func (c *Calculator) Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, ErrDivisionByZero
    }
    result := a / b
    c.history = append(c.history, fmt.Sprintf("%.2f / %.2f = %.2f", a, b, result))
    return result, nil
}

func (c *Calculator) Sqrt(n float64) (float64, error) {
    if n < 0 {
        return 0, ErrNegativeNumber
    }
    // simplified sqrt
    result := n / 2
    c.history = append(c.history, fmt.Sprintf("sqrt(%.2f) = %.2f", n, result))
    return result, nil
}

func main() {
    calc := &Calculator{}
    
    ops := []struct {
        name string
        fn   func() (float64, error)
    }{
        {"10/2", func() (float64, error) { return calc.Divide(10, 2) }},
        {"10/0", func() (float64, error) { return calc.Divide(10, 0) }},
        {"sqrt(9)", func() (float64, error) { return calc.Sqrt(9) }},
        {"sqrt(-1)", func() (float64, error) { return calc.Sqrt(-1) }},
    }
    
    for _, op := range ops {
        result, err := op.fn()
        if err != nil {
            fmt.Printf("%s: ERROR - %v\n", op.name, err)
            
            // ตรวจสอบ sentinel error
            switch {
            case errors.Is(err, ErrDivisionByZero):
                fmt.Println("  -> ตรวจสอบ divisor ก่อน")
            case errors.Is(err, ErrNegativeNumber):
                fmt.Println("  -> ใช้ค่าบวกเท่านั้น")
            }
        } else {
            fmt.Printf("%s = %.4f\n", op.name, result)
        }
    }
    
    // io.EOF เป็น sentinel error ที่ใช้บ่อย
    reader := strings.NewReader("data")
    buf := make([]byte, 10)
    n, err := reader.Read(buf)
    fmt.Printf("\nRead %d bytes: %s\n", n, buf[:n])
    
    n, err = reader.Read(buf)
    if errors.Is(err, io.EOF) {
        fmt.Println("ถึง end of file แล้ว")
    }
    _ = err
}

import "strings"
```

---

## 12.7 Multiple Errors Handling

```go
package main

import (
    "errors"
    "fmt"
    "strings"
)

// errors.Join (Go 1.20+) - รวม errors หลายตัว
func validateForm(data map[string]string) error {
    var errs []error
    
    if name := data["name"]; name == "" {
        errs = append(errs, errors.New("name ต้องไม่ว่าง"))
    } else if len(name) < 2 {
        errs = append(errs, fmt.Errorf("name '%s' สั้นเกินไป", name))
    }
    
    if email := data["email"]; email == "" {
        errs = append(errs, errors.New("email ต้องไม่ว่าง"))
    } else if !strings.Contains(email, "@") {
        errs = append(errs, fmt.Errorf("email '%s' ไม่ถูกต้อง", email))
    }
    
    if phone := data["phone"]; phone != "" && len(phone) != 10 {
        errs = append(errs, fmt.Errorf("phone '%s' ต้องมี 10 หลัก", phone))
    }
    
    return errors.Join(errs...)
}

// Manual multiple error handling
type FieldError struct {
    Field   string
    Message string
}

func (fe FieldError) Error() string {
    return fmt.Sprintf("%s: %s", fe.Field, fe.Message)
}

type FormErrors []FieldError

func (fe FormErrors) Error() string {
    msgs := make([]string, len(fe))
    for i, e := range fe {
        msgs[i] = e.Error()
    }
    return "form validation failed: " + strings.Join(msgs, ", ")
}

func (fe FormErrors) HasField(field string) bool {
    for _, e := range fe {
        if e.Field == field {
            return true
        }
    }
    return false
}

func validateUser(name, email string, age int) error {
    var errs FormErrors
    
    if name == "" {
        errs = append(errs, FieldError{"name", "ต้องไม่ว่าง"})
    }
    if !strings.Contains(email, "@") {
        errs = append(errs, FieldError{"email", "รูปแบบไม่ถูกต้อง"})
    }
    if age < 0 || age > 150 {
        errs = append(errs, FieldError{"age", "ต้องอยู่ระหว่าง 0-150"})
    }
    
    if len(errs) > 0 {
        return errs
    }
    return nil
}

func main() {
    // errors.Join
    fmt.Println("=== errors.Join ===")
    
    forms := []map[string]string{
        {"name": "สมชาย", "email": "somchai@example.com"},
        {"name": "a", "email": "invalid", "phone": "123"},
        {"name": "", "email": ""},
    }
    
    for _, form := range forms {
        err := validateForm(form)
        if err != nil {
            fmt.Printf("Form error: %v\n\n", err)
        } else {
            fmt.Printf("Form OK: %v\n\n", form)
        }
    }
    
    // Custom FormErrors
    fmt.Println("=== FormErrors ===")
    
    users := []struct{ name, email string; age int }{
        {"สมชาย", "somchai@example.com", 25},
        {"", "invalid-email", 200},
    }
    
    for _, u := range users {
        err := validateUser(u.name, u.email, u.age)
        if err != nil {
            fmt.Printf("User validation failed:\n")
            if fe, ok := err.(FormErrors); ok {
                for _, e := range fe {
                    fmt.Printf("  - %s\n", e)
                }
                fmt.Printf("  Has email error: %v\n", fe.HasField("email"))
            }
        } else {
            fmt.Printf("User '%s' OK\n", u.name)
        }
        fmt.Println()
    }
}
```

---

## 12.8 Panic vs Error

### ความแตกต่าง

| | error | panic |
|--|-------|-------|
| ใช้เมื่อ | ความผิดพลาดที่คาดได้ | ความผิดพลาดที่ไม่ควรเกิด |
| recoverable | ใช่ | ใช่ (ด้วย recover) |
| ประสิทธิภาพ | ดีกว่า | แย่กว่า |
| ตัวอย่าง | file not found | index out of bounds |

```go
package main

import "fmt"

// ใช้ error สำหรับ expected errors
func parseAge(s string) (int, error) {
    if s == "" {
        return 0, fmt.Errorf("age string ว่างเปล่า")
    }
    // ตัวอย่างง่ายๆ
    if s == "invalid" {
        return 0, fmt.Errorf("รูปแบบอายุไม่ถูกต้อง: %q", s)
    }
    return 25, nil // simplified
}

// ใช้ panic สำหรับ programmer errors (bugs)
func mustParseAge(s string) int {
    age, err := parseAge(s)
    if err != nil {
        panic(fmt.Sprintf("mustParseAge: %v", err))
    }
    return age
}

// Recover from panic
func safeOperation(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    f()
    return nil
}

func main() {
    // Normal error handling
    age, err := parseAge("25")
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Age:", age)
    }
    
    // Recover from panic
    err = safeOperation(func() {
        panic("something went wrong!")
    })
    fmt.Println("Recovered:", err)
    
    err = safeOperation(func() {
        // Safe operation
        fmt.Println("Safe operation completed")
    })
    fmt.Println("Normal err:", err)
    
    // ระวัง! mustParse จะ panic ถ้า input ผิด
    // ใช้เฉพาะที่แน่ใจว่า input ถูกต้องเสมอ
    age = mustParseAge("30")
    fmt.Println("Must age:", age)
}
```

### defer, panic, recover pattern

```go
package main

import (
    "fmt"
    "runtime/debug"
)

func riskyOperation(n int) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Recovered in riskyOperation(%d): %v\n", n, r)
            // อาจ log stack trace
            debug.PrintStack()
        }
    }()
    
    if n == 0 {
        panic("n cannot be zero")
    }
    
    result := 100 / n
    fmt.Printf("100 / %d = %d\n", n, result)
}

type SafeDB struct {
    connected bool
}

func (db *SafeDB) Query(sql string) (result string, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("query panic: %v", r)
        }
    }()
    
    if !db.connected {
        panic("database not connected")
    }
    
    return "mock result", nil
}

func main() {
    // Recover from panic
    for _, n := range []int{5, 0, 3} {
        riskyOperation(n)
    }
    
    // Panic in method
    db := &SafeDB{connected: false}
    result, err := db.Query("SELECT * FROM users")
    if err != nil {
        fmt.Println("\nDB Error:", err)
    } else {
        fmt.Println("Result:", result)
    }
    
    // Connected
    db.connected = true
    result, err = db.Query("SELECT * FROM users")
    if err != nil {
        fmt.Println("DB Error:", err)
    } else {
        fmt.Println("Result:", result)
    }
}
```

---

## 12.9 Error Handling Best Practices

### 1. ตรวจสอบ error ทุกครั้ง

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // ไม่ดี - ignore error
    // f, _ := os.Open("file.txt")
    
    // ดี - ตรวจสอบ error
    f, err := os.Open("file.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "ไม่สามารถเปิดไฟล์: %v\n", err)
        os.Exit(1)
    }
    defer f.Close()
    
    fmt.Printf("เปิดไฟล์: %s\n", f.Name())
}
```

### 2. Handle errors อย่างเหมาะสม

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

type UserService struct {
    users map[int]string
}

func (s *UserService) GetUser(id int) (string, error) {
    user, ok := s.users[id]
    if !ok {
        return "", fmt.Errorf("GetUser(id=%d): %w", id, ErrNotFound)
    }
    return user, nil
}

func (s *UserService) DeleteUser(id int) error {
    _, err := s.GetUser(id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            // ถ้าไม่พบก็ไม่เป็นไร - idempotent
            return nil
        }
        return fmt.Errorf("DeleteUser: %w", err)
    }
    delete(s.users, id)
    return nil
}

func main() {
    svc := &UserService{
        users: map[int]string{
            1: "สมชาย",
            2: "สมหญิง",
        },
    }
    
    // Get existing user
    user, err := svc.GetUser(1)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("User:", user)
    }
    
    // Get non-existing user
    _, err = svc.GetUser(99)
    if err != nil {
        fmt.Println("Error:", err)
        if errors.Is(err, ErrNotFound) {
            fmt.Println("-> User ไม่มีในระบบ")
        }
    }
    
    // Delete - idempotent
    fmt.Println("\nDelete user 1:", svc.DeleteUser(1))
    fmt.Println("Delete user 1 again:", svc.DeleteUser(1)) // no error
    fmt.Println("Delete user 99:", svc.DeleteUser(99))     // no error
}
```

### 3. Error wrapping chain

```go
package main

import (
    "errors"
    "fmt"
)

// Layer structure
type Repository struct{}
type Service struct{ repo *Repository }
type Handler struct{ svc *Service }

var ErrRecordNotFound = errors.New("record not found")

func (r *Repository) Find(id int) (string, error) {
    if id != 1 {
        return "", fmt.Errorf("repository.Find(id=%d): %w", id, ErrRecordNotFound)
    }
    return "record-1", nil
}

func (s *Service) GetRecord(id int) (string, error) {
    record, err := s.repo.Find(id)
    if err != nil {
        return "", fmt.Errorf("service.GetRecord: %w", err)
    }
    return record, nil
}

func (h *Handler) HandleGet(id int) {
    record, err := h.svc.GetRecord(id)
    if err != nil {
        // Log full error chain
        fmt.Printf("Handler error: %v\n", err)
        
        // Check specific error for user response
        if errors.Is(err, ErrRecordNotFound) {
            fmt.Println("Response: 404 Not Found")
        } else {
            fmt.Println("Response: 500 Internal Server Error")
        }
        return
    }
    fmt.Printf("Response: 200 OK - %s\n", record)
}

func main() {
    repo := &Repository{}
    svc := &Service{repo: repo}
    handler := &Handler{svc: svc}
    
    fmt.Println("GET /records/1:")
    handler.HandleGet(1)
    
    fmt.Println("\nGET /records/99:")
    handler.HandleGet(99)
}
```

---

## 12.10 Error Pattern ใน Real Applications

### Result Type Pattern

```go
package main

import "fmt"

type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: value}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOk() bool {
    return r.err == nil
}

func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic(fmt.Sprintf("called Unwrap on error Result: %v", r.err))
    }
    return r.value
}

func (r Result[T]) UnwrapOr(defaultVal T) T {
    if r.err != nil {
        return defaultVal
    }
    return r.value
}

func (r Result[T]) Error() error {
    return r.err
}

func parseNumber(s string) Result[int] {
    if s == "" {
        return Err[int](fmt.Errorf("empty string"))
    }
    if s == "42" {
        return Ok(42)
    }
    return Err[int](fmt.Errorf("cannot parse %q as int", s))
}

func main() {
    tests := []string{"42", "", "abc", "100"}
    
    for _, t := range tests {
        result := parseNumber(t)
        if result.IsOk() {
            fmt.Printf("parseNumber(%q) = %d\n", t, result.Unwrap())
        } else {
            fmt.Printf("parseNumber(%q) error: %v\n", t, result.Error())
            fmt.Printf("  default: %d\n", result.UnwrapOr(0))
        }
    }
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| `error` interface | Interface ที่มีแค่ `Error() string` |
| `errors.New` | สร้าง simple error |
| `fmt.Errorf` | สร้าง formatted error |
| `%w` | Wrap error เพื่อเก็บ chain |
| `errors.Is` | ตรวจสอบ error ตรงกับ sentinel |
| `errors.As` | ดึง concrete error type |
| `errors.Unwrap` | Unwrap error ชั้นเดียว |
| `errors.Join` | รวม errors หลายตัว (Go 1.20+) |
| Custom error type | Struct ที่ implement error interface |
| Panic | สำหรับ programmer errors ที่ไม่ควรเกิด |
| Recover | จัดการ panic ใน defer function |

## Resources

- [Go Blog: Error handling and Go](https://go.dev/blog/error-handling-and-go)
- [Go Blog: Errors are values](https://go.dev/blog/errors-are-values)
- [Go Blog: Working with Errors](https://go.dev/blog/go1.13-errors)
- [Go by Example: Errors](https://gobyexample.com/errors)
- [Effective Go: Errors](https://go.dev/doc/effective_go#errors)

---

*Part 12 จบแล้ว! ต่อไป Part 13: Goroutines and Concurrency*
