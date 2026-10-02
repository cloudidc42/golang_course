# Part 19: Testing ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เขียน unit tests ด้วย `testing` package
- ใช้ table-driven tests pattern
- เขียน subtests ด้วย `t.Run`
- เขียน benchmarks
- เขียน example tests สำหรับ documentation
- สร้าง test helpers
- Mock dependencies ด้วย interfaces
- วัด test coverage
- ใช้ `testify` library

---

## 19.1 testing Package พื้นฐาน

### 19.1.1 โครงสร้าง Test File

```go
// calculator.go
package calculator

import "errors"

var ErrDivisionByZero = errors.New("division by zero")

func Add(a, b int) int {
    return a + b
}

func Subtract(a, b int) int {
    return a - b
}

func Multiply(a, b int) int {
    return a * b
}

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, ErrDivisionByZero
    }
    return a / b, nil
}

func Fibonacci(n int) int {
    if n <= 0 {
        return 0
    }
    if n == 1 {
        return 1
    }
    return Fibonacci(n-1) + Fibonacci(n-2)
}

func IsPrime(n int) bool {
    if n < 2 {
        return false
    }
    for i := 2; i*i <= n; i++ {
        if n%i == 0 {
            return false
        }
    }
    return true
}
```

```go
// calculator_test.go
package calculator

import (
    "testing"
)

// Test function ต้องขึ้นต้นด้วย Test และรับ *testing.T
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        // t.Errorf - รายงาน error แต่ยังทำ test ต่อ
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

func TestSubtract(t *testing.T) {
    result := Subtract(10, 4)
    expected := 6
    
    if result != expected {
        t.Errorf("Subtract(10, 4) = %d; want %d", result, expected)
    }
}

func TestMultiply(t *testing.T) {
    result := Multiply(3, 4)
    expected := 12
    
    if result != expected {
        t.Errorf("Multiply(3, 4) = %d; want %d", result, expected)
    }
}

func TestDivide(t *testing.T) {
    // ทดสอบกรณีปกติ
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("Divide(10, 2) unexpected error: %v", err)
    }
    if result != 5.0 {
        t.Errorf("Divide(10, 2) = %f; want 5.0", result)
    }
    
    // ทดสอบ division by zero
    _, err = Divide(10, 0)
    if err == nil {
        // t.Fatal - หยุด test ทันที
        t.Fatal("Divide(10, 0) should return error")
    }
    if err != ErrDivisionByZero {
        t.Errorf("Divide(10, 0) error = %v; want %v", err, ErrDivisionByZero)
    }
}

func TestIsPrime(t *testing.T) {
    if !IsPrime(7) {
        t.Error("7 should be prime")
    }
    if IsPrime(4) {
        t.Error("4 should not be prime")
    }
    if IsPrime(1) {
        t.Error("1 should not be prime")
    }
    if !IsPrime(2) {
        t.Error("2 should be prime")
    }
}
```

### 19.1.2 รัน Tests

```bash
# รัน tests ทั้งหมดใน package
go test ./...

# รันพร้อม verbose output
go test -v ./...

# รัน test เฉพาะ
go test -run TestAdd

# รัน tests ที่ match pattern
go test -run TestD

# รัน tests หลายครั้ง
go test -count=3 ./...

# กำหนด timeout
go test -timeout 30s ./...
```

---

## 19.2 Table-Driven Tests

### 19.2.1 Basic Table-Driven Test

```go
package calculator

import "testing"

func TestAddTableDriven(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed", -2, 3, 1},
        {"zeros", 0, 0, 0},
        {"large numbers", 1000000, 2000000, 3000000},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}

func TestDivideTableDriven(t *testing.T) {
    tests := []struct {
        name        string
        a, b        float64
        expected    float64
        expectError bool
        err         error
    }{
        {"normal division", 10, 2, 5.0, false, nil},
        {"divide by zero", 10, 0, 0, true, ErrDivisionByZero},
        {"negative", -10, 2, -5.0, false, nil},
        {"decimal", 7, 2, 3.5, false, nil},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Divide(tt.a, tt.b)
            
            if tt.expectError {
                if err == nil {
                    t.Fatalf("expected error %v, got nil", tt.err)
                }
                if err != tt.err {
                    t.Errorf("expected error %v, got %v", tt.err, err)
                }
            } else {
                if err != nil {
                    t.Fatalf("unexpected error: %v", err)
                }
                if result != tt.expected {
                    t.Errorf("Divide(%v, %v) = %v; want %v",
                        tt.a, tt.b, result, tt.expected)
                }
            }
        })
    }
}

func TestIsPrimeTableDriven(t *testing.T) {
    tests := []struct {
        input    int
        expected bool
    }{
        {-1, false},
        {0, false},
        {1, false},
        {2, true},
        {3, true},
        {4, false},
        {5, true},
        {6, false},
        {7, true},
        {11, true},
        {13, true},
        {17, true},
        {18, false},
        {97, true},
        {100, false},
    }
    
    for _, tt := range tests {
        t.Run(fmt.Sprintf("IsPrime(%d)", tt.input), func(t *testing.T) {
            result := IsPrime(tt.input)
            if result != tt.expected {
                t.Errorf("IsPrime(%d) = %v; want %v",
                    tt.input, result, tt.expected)
            }
        })
    }
}
```

---

## 19.3 Subtests

### 19.3.1 t.Run

```go
package main

import (
    "fmt"
    "strings"
    "testing"
)

type StringProcessor struct{}

func (sp StringProcessor) Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func (sp StringProcessor) CountVowels(s string) int {
    count := 0
    for _, r := range strings.ToLower(s) {
        switch r {
        case 'a', 'e', 'i', 'o', 'u':
            count++
        }
    }
    return count
}

func (sp StringProcessor) Palindrome(s string) bool {
    s = strings.ToLower(s)
    rev := sp.Reverse(s)
    return s == rev
}

func TestStringProcessor(t *testing.T) {
    sp := StringProcessor{}
    
    // Subtest group 1: Reverse
    t.Run("Reverse", func(t *testing.T) {
        t.Run("simple", func(t *testing.T) {
            if got := sp.Reverse("hello"); got != "olleh" {
                t.Errorf("got %q; want %q", got, "olleh")
            }
        })
        
        t.Run("unicode", func(t *testing.T) {
            if got := sp.Reverse("สวัสดี"); got == "สวัสดี" {
                // palindrome shouldn't stay same
                // Just check it's not empty
            }
        })
        
        t.Run("empty", func(t *testing.T) {
            if got := sp.Reverse(""); got != "" {
                t.Errorf("got %q; want empty string", got)
            }
        })
        
        t.Run("single char", func(t *testing.T) {
            if got := sp.Reverse("a"); got != "a" {
                t.Errorf("got %q; want %q", got, "a")
            }
        })
    })
    
    // Subtest group 2: CountVowels
    t.Run("CountVowels", func(t *testing.T) {
        tests := []struct{ input string; expected int }{
            {"hello", 2},
            {"HELLO", 2},
            {"rhythm", 0},
            {"aeiou", 5},
            {"", 0},
        }
        
        for _, tt := range tests {
            tt := tt // capture
            t.Run(fmt.Sprintf("input=%q", tt.input), func(t *testing.T) {
                if got := sp.CountVowels(tt.input); got != tt.expected {
                    t.Errorf("CountVowels(%q) = %d; want %d",
                        tt.input, got, tt.expected)
                }
            })
        }
    })
    
    // Subtest group 3: Palindrome
    t.Run("Palindrome", func(t *testing.T) {
        t.Run("true cases", func(t *testing.T) {
            palindromes := []string{"racecar", "level", "madam", "a", ""}
            for _, p := range palindromes {
                if !sp.Palindrome(p) {
                    t.Errorf("%q should be palindrome", p)
                }
            }
        })
        
        t.Run("false cases", func(t *testing.T) {
            nonPalindromes := []string{"hello", "world", "golang"}
            for _, p := range nonPalindromes {
                if sp.Palindrome(p) {
                    t.Errorf("%q should not be palindrome", p)
                }
            }
        })
    })
}
```

---

## 19.4 Benchmarks

### 19.4.1 เขียน Benchmark

```go
package calculator

import (
    "testing"
)

// Benchmark functions ต้องขึ้นต้นด้วย Benchmark
func BenchmarkFibonacci(b *testing.B) {
    // b.N - จำนวนครั้งที่รัน (Go จะปรับอัตโนมัติ)
    for i := 0; i < b.N; i++ {
        Fibonacci(20)
    }
}

func BenchmarkIsPrime(b *testing.B) {
    for i := 0; i < b.N; i++ {
        IsPrime(97)
    }
}

// เปรียบเทียบหลาย implementations
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(100, 200)
    }
}

// Benchmark กับ input ต่างๆ
func BenchmarkFibonacciSizes(b *testing.B) {
    sizes := []int{5, 10, 15, 20, 25}
    
    for _, size := range sizes {
        size := size
        b.Run(fmt.Sprintf("n=%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                Fibonacci(size)
            }
        })
    }
}

// String concatenation benchmark
func BenchmarkStringConcat(b *testing.B) {
    b.Run("plus operator", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := ""
            for j := 0; j < 100; j++ {
                s += "x"
            }
            _ = s
        }
    })
    
    b.Run("strings.Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            for j := 0; j < 100; j++ {
                sb.WriteByte('x')
            }
            _ = sb.String()
        }
    })
    
    b.Run("byte slice", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            buf := make([]byte, 0, 100)
            for j := 0; j < 100; j++ {
                buf = append(buf, 'x')
            }
            _ = string(buf)
        }
    })
}

// ResetTimer - ไม่นับเวลา setup
func BenchmarkWithSetup(b *testing.B) {
    // Setup
    data := make([]int, 1000)
    for i := range data {
        data[i] = i
    }
    
    b.ResetTimer() // เริ่มนับเวลาใหม่หลัง setup
    
    for i := 0; i < b.N; i++ {
        sum := 0
        for _, v := range data {
            sum += v
        }
        _ = sum
    }
}

// ReportAllocs - รายงาน memory allocations
func BenchmarkMemory(b *testing.B) {
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        s := fmt.Sprintf("hello %d world %d", i, i*2)
        _ = s
    }
}
```

```bash
# รัน benchmarks
go test -bench=. ./...

# รัน benchmark เฉพาะ
go test -bench=BenchmarkFibonacci

# รันพร้อม memory stats
go test -bench=. -benchmem

# รัน N ครั้ง
go test -bench=. -benchtime=5s

# Compare benchmarks ด้วย benchstat
go test -bench=. -count=10 > old.txt
# แก้ code
go test -bench=. -count=10 > new.txt
# benchstat old.txt new.txt
```

---

## 19.5 Example Tests

### 19.5.1 Example Functions

```go
package calculator

import "fmt"

// Example functions ต้องขึ้นต้นด้วย Example
// Comment บรรทัดสุดท้าย // Output: จะถูกตรวจสอบ
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}

func ExampleSubtract() {
    result := Subtract(10, 4)
    fmt.Println(result)
    // Output: 6
}

func ExampleDivide() {
    result, err := Divide(10, 2)
    if err != nil {
        fmt.Println("error:", err)
        return
    }
    fmt.Printf("%.1f\n", result)
    // Output: 5.0
}

func ExampleDivide_byZero() {
    _, err := Divide(10, 0)
    fmt.Println(err)
    // Output: division by zero
}

func ExampleIsPrime() {
    fmt.Println(IsPrime(7))
    fmt.Println(IsPrime(4))
    fmt.Println(IsPrime(1))
    // Output:
    // true
    // false
    // false
}

func ExampleFibonacci() {
    for i := 0; i <= 8; i++ {
        fmt.Printf("Fibonacci(%d) = %d\n", i, Fibonacci(i))
    }
    // Output:
    // Fibonacci(0) = 0
    // Fibonacci(1) = 1
    // Fibonacci(2) = 1
    // Fibonacci(3) = 2
    // Fibonacci(4) = 3
    // Fibonacci(5) = 5
    // Fibonacci(6) = 8
    // Fibonacci(7) = 13
    // Fibonacci(8) = 21
}
```

---

## 19.6 Test Helpers

### 19.6.1 Helper Functions

```go
package testutil

import (
    "encoding/json"
    "os"
    "testing"
)

// assertEqual - ใช้ t.Helper() เพื่อรายงาน error ที่ caller
func assertEqual(t *testing.T, expected, actual interface{}) {
    t.Helper() // บอกว่านี่คือ helper function
    if expected != actual {
        t.Errorf("\nexpected: %v\n  actual: %v", expected, actual)
    }
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func assertError(t *testing.T, err error) {
    t.Helper()
    if err == nil {
        t.Fatal("expected error, got nil")
    }
}

func assertErrorIs(t *testing.T, err, target error) {
    t.Helper()
    if !errors.Is(err, target) {
        t.Errorf("expected error %v, got %v", target, err)
    }
}

// LoadFixture - โหลด test fixture จาก file
func LoadFixture(t *testing.T, filename string) []byte {
    t.Helper()
    data, err := os.ReadFile("testdata/" + filename)
    if err != nil {
        t.Fatalf("failed to load fixture %s: %v", filename, err)
    }
    return data
}

// LoadJSONFixture - โหลดและ parse JSON fixture
func LoadJSONFixture(t *testing.T, filename string, v interface{}) {
    t.Helper()
    data := LoadFixture(t, filename)
    if err := json.Unmarshal(data, v); err != nil {
        t.Fatalf("failed to parse JSON fixture %s: %v", filename, err)
    }
}

// TempDir - สร้าง temp directory และ cleanup อัตโนมัติ
func TempDir(t *testing.T) string {
    t.Helper()
    dir, err := os.MkdirTemp("", "test-*")
    if err != nil {
        t.Fatalf("failed to create temp dir: %v", err)
    }
    t.Cleanup(func() {
        os.RemoveAll(dir)
    })
    return dir
}

// TempFile - สร้าง temp file และ cleanup อัตโนมัติ
func TempFile(t *testing.T, content string) string {
    t.Helper()
    f, err := os.CreateTemp("", "test-*.txt")
    if err != nil {
        t.Fatalf("failed to create temp file: %v", err)
    }
    
    if content != "" {
        f.WriteString(content)
    }
    f.Close()
    
    t.Cleanup(func() {
        os.Remove(f.Name())
    })
    return f.Name()
}

// ตัวอย่างการใช้ helper
func TestWithHelpers(t *testing.T) {
    // assertEqual
    assertEqual(t, 5, Add(2, 3))
    assertEqual(t, "hello", "hello")
    
    // assertNoError
    _, err := Divide(10, 2)
    assertNoError(t, err)
    
    // assertError
    _, err2 := Divide(10, 0)
    assertError(t, err2)
    
    // TempDir
    tmpDir := TempDir(t) // cleanup อัตโนมัติเมื่อ test จบ
    _ = tmpDir
    
    // TempFile
    tmpFile := TempFile(t, "hello world")
    _ = tmpFile
}
```

---

## 19.7 Mocking กับ Interfaces

### 19.7.1 Interface-based Mocking

```go
package main

import (
    "errors"
    "fmt"
    "testing"
)

// Interface definitions
type UserRepository interface {
    GetByID(id int) (*User, error)
    Save(user *User) error
    Delete(id int) error
}

type EmailService interface {
    Send(to, subject, body string) error
}

type User struct {
    ID    int
    Name  string
    Email string
    Age   int
}

// Business logic (service layer)
type UserService struct {
    repo  UserRepository
    email EmailService
}

func NewUserService(repo UserRepository, email EmailService) *UserService {
    return &UserService{repo: repo, email: email}
}

func (s *UserService) RegisterUser(name, email string, age int) (*User, error) {
    if age < 18 {
        return nil, errors.New("must be 18 or older")
    }
    
    user := &User{Name: name, Email: email, Age: age}
    
    if err := s.repo.Save(user); err != nil {
        return nil, fmt.Errorf("save failed: %w", err)
    }
    
    if err := s.email.Send(email, "Welcome!", "Welcome to our service!"); err != nil {
        // Log แต่ไม่ return error (email ไม่ critical)
        fmt.Printf("warning: failed to send welcome email: %v\n", err)
    }
    
    return user, nil
}

func (s *UserService) GetUser(id int) (*User, error) {
    user, err := s.repo.GetByID(id)
    if err != nil {
        return nil, fmt.Errorf("get user failed: %w", err)
    }
    return user, nil
}

// Mock implementations
type MockUserRepository struct {
    users      map[int]*User
    nextID     int
    SaveErr    error
    GetErr     error
    DeleteErr  error
    SaveCalls  int
    GetCalls   int
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users:  make(map[int]*User),
        nextID: 1,
    }
}

func (m *MockUserRepository) GetByID(id int) (*User, error) {
    m.GetCalls++
    if m.GetErr != nil {
        return nil, m.GetErr
    }
    user, ok := m.users[id]
    if !ok {
        return nil, fmt.Errorf("user %d not found", id)
    }
    return user, nil
}

func (m *MockUserRepository) Save(user *User) error {
    m.SaveCalls++
    if m.SaveErr != nil {
        return m.SaveErr
    }
    user.ID = m.nextID
    m.users[m.nextID] = user
    m.nextID++
    return nil
}

func (m *MockUserRepository) Delete(id int) error {
    if m.DeleteErr != nil {
        return m.DeleteErr
    }
    delete(m.users, id)
    return nil
}

type MockEmailService struct {
    SendErr   error
    SentMails []struct{ To, Subject, Body string }
}

func (m *MockEmailService) Send(to, subject, body string) error {
    if m.SendErr != nil {
        return m.SendErr
    }
    m.SentMails = append(m.SentMails, struct{ To, Subject, Body string }{to, subject, body})
    return nil
}

// Tests
func TestUserService_RegisterUser(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        repo := NewMockUserRepository()
        emailSvc := &MockEmailService{}
        svc := NewUserService(repo, emailSvc)
        
        user, err := svc.RegisterUser("Alice", "alice@example.com", 25)
        
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
        if user == nil {
            t.Fatal("expected user, got nil")
        }
        if user.Name != "Alice" {
            t.Errorf("expected name Alice, got %s", user.Name)
        }
        if user.ID == 0 {
            t.Error("expected non-zero ID")
        }
        if repo.SaveCalls != 1 {
            t.Errorf("expected 1 save call, got %d", repo.SaveCalls)
        }
        if len(emailSvc.SentMails) != 1 {
            t.Errorf("expected 1 email sent, got %d", len(emailSvc.SentMails))
        }
    })
    
    t.Run("underage", func(t *testing.T) {
        repo := NewMockUserRepository()
        emailSvc := &MockEmailService{}
        svc := NewUserService(repo, emailSvc)
        
        _, err := svc.RegisterUser("Bob", "bob@example.com", 16)
        
        if err == nil {
            t.Fatal("expected error for underage user")
        }
        if repo.SaveCalls != 0 {
            t.Error("should not save underage user")
        }
    })
    
    t.Run("repo error", func(t *testing.T) {
        repo := NewMockUserRepository()
        repo.SaveErr = errors.New("database down")
        emailSvc := &MockEmailService{}
        svc := NewUserService(repo, emailSvc)
        
        _, err := svc.RegisterUser("Charlie", "charlie@example.com", 30)
        
        if err == nil {
            t.Fatal("expected error when repo fails")
        }
        if len(emailSvc.SentMails) != 0 {
            t.Error("should not send email when save fails")
        }
    })
    
    t.Run("email error is non-fatal", func(t *testing.T) {
        repo := NewMockUserRepository()
        emailSvc := &MockEmailService{SendErr: errors.New("smtp down")}
        svc := NewUserService(repo, emailSvc)
        
        // ควร register สำเร็จแม้ email จะ fail
        user, err := svc.RegisterUser("Dave", "dave@example.com", 22)
        
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
        if user == nil {
            t.Fatal("expected user, got nil")
        }
    })
}
```

---

## 19.8 Test Coverage

### 19.8.1 วัด Coverage

```bash
# รันพร้อมวัด coverage
go test -cover ./...

# สร้าง coverage report
go test -coverprofile=coverage.out ./...

# ดู coverage ใน HTML
go tool cover -html=coverage.out

# ดู coverage ต่อ function
go tool cover -func=coverage.out

# coverage minimum threshold
go test -cover -covermode=atomic ./...
```

### 19.8.2 Coverage ใน Code

```go
package math

// sqrt.go
import "math"

func Sqrt(x float64) float64 {
    if x < 0 {
        panic("negative input")
    }
    return math.Sqrt(x)
}

func Abs(x float64) float64 {
    if x < 0 {
        return -x
    }
    return x
}

func Max(a, b float64) float64 {
    if a > b {
        return a
    }
    return b
}

func Min(a, b float64) float64 {
    if a < b {
        return a
    }
    return b
}

func Clamp(value, min, max float64) float64 {
    if value < min {
        return min
    }
    if value > max {
        return max
    }
    return value
}
```

```go
// sqrt_test.go
package math

import (
    "testing"
)

func TestSqrt(t *testing.T) {
    tests := []struct {
        input    float64
        expected float64
    }{
        {4, 2},
        {9, 3},
        {0, 0},
        {2, 1.4142135623730951},
    }
    
    for _, tt := range tests {
        got := Sqrt(tt.input)
        if got != tt.expected {
            t.Errorf("Sqrt(%v) = %v; want %v", tt.input, got, tt.expected)
        }
    }
}

func TestSqrtPanic(t *testing.T) {
    defer func() {
        if r := recover(); r == nil {
            t.Error("expected panic for negative input")
        }
    }()
    Sqrt(-1)
}

// ครอบคลุม Abs, Max, Min, Clamp
func TestMathFunctions(t *testing.T) {
    // Abs
    if Abs(-5) != 5 {
        t.Error("Abs(-5) should be 5")
    }
    if Abs(5) != 5 {
        t.Error("Abs(5) should be 5")
    }
    
    // Max
    if Max(3, 5) != 5 {
        t.Error("Max(3, 5) should be 5")
    }
    if Max(5, 3) != 5 {
        t.Error("Max(5, 3) should be 5")
    }
    
    // Min
    if Min(3, 5) != 3 {
        t.Error("Min(3, 5) should be 3")
    }
    
    // Clamp
    if Clamp(10, 0, 5) != 5 {
        t.Error("Clamp(10, 0, 5) should be 5")
    }
    if Clamp(-1, 0, 5) != 0 {
        t.Error("Clamp(-1, 0, 5) should be 0")
    }
    if Clamp(3, 0, 5) != 3 {
        t.Error("Clamp(3, 0, 5) should be 3")
    }
}
```

---

## 19.9 testify Library

### 19.9.1 testify/assert

```go
package main

import (
    "errors"
    "testing"
    
    // go get github.com/stretchr/testify
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

type Calculator struct{}

func (c Calculator) Add(a, b int) int { return a + b }
func (c Calculator) Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func TestCalculatorWithTestify(t *testing.T) {
    calc := Calculator{}
    
    // assert - ทำ test ต่อแม้จะ fail
    assert.Equal(t, 5, calc.Add(2, 3), "2+3 should equal 5")
    assert.NotEqual(t, 6, calc.Add(2, 3))
    
    // require - หยุดทันทีถ้า fail
    result, err := calc.Divide(10, 2)
    require.NoError(t, err, "division should not error")
    require.Equal(t, 5.0, result)
    
    // assert กับ types
    assert.IsType(t, 0, calc.Add(1, 2))
    
    // assert error
    _, err2 := calc.Divide(10, 0)
    assert.Error(t, err2)
    assert.EqualError(t, err2, "division by zero")
    
    // assert nil/not nil
    assert.Nil(t, err)     // err จาก Divide(10,2) ควรเป็น nil
    assert.NotNil(t, err2)
    
    // Slice assertions
    got := []int{1, 2, 3}
    assert.Equal(t, []int{1, 2, 3}, got)
    assert.Contains(t, got, 2)
    assert.Len(t, got, 3)
    
    // String assertions
    str := "Hello, World!"
    assert.Contains(t, str, "World")
    assert.HasPrefix(t, str, "Hello")  // ใน testify v2
    
    // Bool assertions
    assert.True(t, 1 == 1)
    assert.False(t, 1 == 2)
    
    // Greater/Less
    assert.Greater(t, 5, 3)
    assert.Less(t, 3, 5)
    assert.GreaterOrEqual(t, 5, 5)
    assert.LessOrEqual(t, 3, 3)
}

func TestWithSuite(t *testing.T) {
    // Table driven + testify
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"basic", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 5, 5},
    }
    
    calc := Calculator{}
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            assert.Equal(t, tt.expected, calc.Add(tt.a, tt.b))
        })
    }
}
```

### 19.9.2 testify/mock

```go
package main

import (
    "testing"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// Interface
type DataStore interface {
    Get(key string) (string, error)
    Set(key, value string) error
    Delete(key string) error
}

// Mock
type MockDataStore struct {
    mock.Mock
}

func (m *MockDataStore) Get(key string) (string, error) {
    args := m.Called(key)
    return args.String(0), args.Error(1)
}

func (m *MockDataStore) Set(key, value string) error {
    args := m.Called(key, value)
    return args.Error(0)
}

func (m *MockDataStore) Delete(key string) error {
    args := m.Called(key)
    return args.Error(0)
}

// Service
type CacheService struct {
    store DataStore
}

func (s *CacheService) GetOrCreate(key string, creator func() string) (string, error) {
    if val, err := s.store.Get(key); err == nil {
        return val, nil
    }
    
    newVal := creator()
    if err := s.store.Set(key, newVal); err != nil {
        return "", err
    }
    return newVal, nil
}

func TestCacheService(t *testing.T) {
    t.Run("returns cached value", func(t *testing.T) {
        mockStore := new(MockDataStore)
        svc := &CacheService{store: mockStore}
        
        // กำหนดว่า Get จะ return อะไร
        mockStore.On("Get", "mykey").Return("cached_value", nil)
        
        result, err := svc.GetOrCreate("mykey", func() string {
            return "new_value"
        })
        
        assert.NoError(t, err)
        assert.Equal(t, "cached_value", result)
        
        // ตรวจสอบว่า Set ไม่ถูกเรียก (ใช้ cached value)
        mockStore.AssertNotCalled(t, "Set")
        mockStore.AssertExpectations(t)
    })
    
    t.Run("creates when not cached", func(t *testing.T) {
        mockStore := new(MockDataStore)
        svc := &CacheService{store: mockStore}
        
        // Get return error (cache miss)
        mockStore.On("Get", "newkey").Return("", errors.New("not found"))
        mockStore.On("Set", "newkey", "created_value").Return(nil)
        
        result, err := svc.GetOrCreate("newkey", func() string {
            return "created_value"
        })
        
        assert.NoError(t, err)
        assert.Equal(t, "created_value", result)
        mockStore.AssertExpectations(t)
    })
}
```

---

## 19.10 Setup/Teardown

### 19.10.1 TestMain

```go
package mypackage

import (
    "fmt"
    "os"
    "testing"
)

var testDB *MockDB

type MockDB struct {
    data map[string]string
}

func (db *MockDB) Close() {
    fmt.Println("Closing mock DB")
}

// TestMain - รันก่อน/หลัง tests ทั้งหมดใน package
func TestMain(m *testing.M) {
    fmt.Println("=== Test Setup ===")
    
    // Setup
    testDB = &MockDB{data: make(map[string]string)}
    testDB.data["user:1"] = `{"id":1,"name":"Alice"}`
    
    // รัน tests
    code := m.Run()
    
    // Teardown
    fmt.Println("=== Test Teardown ===")
    testDB.Close()
    
    os.Exit(code)
}

func TestUseDB(t *testing.T) {
    if testDB == nil {
        t.Fatal("testDB not initialized")
    }
    
    val, ok := testDB.data["user:1"]
    if !ok {
        t.Fatal("user:1 not found")
    }
    
    if val == "" {
        t.Error("expected non-empty value")
    }
    
    fmt.Println("Got:", val)
}
```

### 19.10.2 t.Cleanup

```go
package main

import (
    "os"
    "testing"
)

func setupTempFile(t *testing.T, content string) string {
    t.Helper()
    
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatalf("failed to create temp file: %v", err)
    }
    
    if content != "" {
        f.WriteString(content)
    }
    f.Close()
    
    // t.Cleanup - รันเมื่อ test/subtest จบ
    t.Cleanup(func() {
        os.Remove(f.Name())
        t.Logf("cleaned up: %s", f.Name())
    })
    
    return f.Name()
}

func TestWithCleanup(t *testing.T) {
    filename := setupTempFile(t, "test content")
    
    // อ่านไฟล์
    data, err := os.ReadFile(filename)
    if err != nil {
        t.Fatalf("read failed: %v", err)
    }
    
    if string(data) != "test content" {
        t.Errorf("content mismatch: %q", data)
    }
    
    t.Run("subtest", func(t *testing.T) {
        // Cleanup ของ parent test จะรันหลังจาก subtest ทั้งหมดจบ
        f2 := setupTempFile(t, "subtest content")
        _ = f2
    })
}
```

---

## สรุป

| เครื่องมือ | การใช้งาน |
|-----------|----------|
| `testing.T` | Unit tests, error reporting |
| `testing.B` | Benchmarks |
| `testing.M` | Package-level setup/teardown |
| `t.Run` | Subtests |
| `t.Helper` | Test helpers |
| `t.Cleanup` | Cleanup functions |
| Table-driven | Test multiple cases |
| Interfaces + Mocks | Dependency isolation |
| `testify` | Assertion helpers |

### คำสั่ง go test

```bash
go test ./...                    # รัน tests ทั้งหมด
go test -v ./...                 # verbose
go test -run TestName            # รัน test เฉพาะ
go test -bench=.                 # รัน benchmarks
go test -benchmem                # รัน benchmarks + memory
go test -cover                   # วัด coverage
go test -coverprofile=c.out      # สร้าง coverage report
go tool cover -html=c.out        # ดู coverage ใน HTML
go test -race                    # race condition detection
go test -timeout 30s             # กำหนด timeout
```

## Resources

- [testing package documentation](https://pkg.go.dev/testing)
- [testify library](https://github.com/stretchr/testify)
- [Go testing tutorial](https://go.dev/doc/tutorial/add-a-test)
- [Table Driven Tests](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests)
