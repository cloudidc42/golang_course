# Part 21: Advanced Types ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้ Named Types
- เข้าใจความแตกต่างระหว่าง Type Aliases และ Named Types
- ใช้ Function Types
- เข้าใจ Interface Types อย่างลึกซึ้ง
- ใช้ Reflection ด้วย `reflect` package
- ใช้ Type Assertions อย่างถูกต้อง
- เข้าใจ `any` type (Go 1.18+)
- รู้จัก Comparable types

---

## 21.1 Named Types

### 21.1.1 Named Types พื้นฐาน

```go
package main

import "fmt"

// Named types - สร้าง type ใหม่จาก underlying type
type Celsius float64
type Fahrenheit float64
type Kelvin float64

// Methods บน named type
func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (c Celsius) ToKelvin() Kelvin {
    return Kelvin(c + 273.15)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

// Named type สำหรับ ID
type UserID int64
type ProductID int64
type OrderID int64

// ป้องกันการส่งผิด type
func getUserByID(id UserID) string {
    return fmt.Sprintf("User #%d", id)
}

func getProductByID(id ProductID) string {
    return fmt.Sprintf("Product #%d", id)
}

// Named type สำหรับ string
type Email string
type URL string
type PhoneNumber string

func (e Email) IsValid() bool {
    return strings.Contains(string(e), "@")
}

func (u URL) IsHTTPS() bool {
    return strings.HasPrefix(string(u), "https://")
}

func main() {
    // Temperature conversions
    temp := Celsius(100)
    fmt.Printf("%.1f°C = %.1f°F = %.2fK\n",
        temp, temp.ToFahrenheit(), temp.ToKelvin())
    
    body := Fahrenheit(98.6)
    fmt.Printf("%.1f°F = %.1f°C\n", body, body.ToCelsius())
    
    // ID types ป้องกัน confusion
    uid := UserID(123)
    pid := ProductID(456)
    
    fmt.Println(getUserByID(uid))
    fmt.Println(getProductByID(pid))
    
    // ต่อไปนี้จะ compile error:
    // getUserByID(pid)  // cannot use pid (type ProductID) as type UserID
    
    // Email
    email := Email("user@example.com")
    fmt.Printf("Email %q valid: %v\n", email, email.IsValid())
    
    // Conversion
    var c Celsius = 25
    var f float64 = float64(c) // ต้องแปลงอย่างชัดเจน
    fmt.Printf("Celsius: %v, float64: %v\n", c, f)
}
```

### 21.1.2 Named Types สำหรับ Slice/Map/Channel

```go
package main

import (
    "fmt"
    "sort"
)

// Named slice types
type IntSlice []int
type StringSlice []string

func (s IntSlice) Sum() int {
    total := 0
    for _, v := range s {
        total += v
    }
    return total
}

func (s IntSlice) Max() int {
    if len(s) == 0 {
        panic("empty slice")
    }
    max := s[0]
    for _, v := range s[1:] {
        if v > max {
            max = v
        }
    }
    return max
}

func (s IntSlice) Filter(pred func(int) bool) IntSlice {
    var result IntSlice
    for _, v := range s {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func (s IntSlice) Map(f func(int) int) IntSlice {
    result := make(IntSlice, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

func (s StringSlice) Contains(target string) bool {
    for _, v := range s {
        if v == target {
            return true
        }
    }
    return false
}

func (s StringSlice) Sorted() StringSlice {
    result := make(StringSlice, len(s))
    copy(result, s)
    sort.Strings(result)
    return result
}

// Named map type
type Registry map[string]interface{}

func (r Registry) Register(key string, value interface{}) {
    r[key] = value
}

func (r Registry) Get(key string) (interface{}, bool) {
    v, ok := r[key]
    return v, ok
}

func (r Registry) Keys() []string {
    keys := make([]string, 0, len(r))
    for k := range r {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    return keys
}

func main() {
    // IntSlice
    nums := IntSlice{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    fmt.Println("Numbers:", nums)
    fmt.Println("Sum:", nums.Sum())
    fmt.Println("Max:", nums.Max())
    
    evens := nums.Filter(func(n int) bool { return n%2 == 0 })
    fmt.Println("Evens:", evens)
    
    doubled := nums.Map(func(n int) int { return n * 2 })
    fmt.Println("Doubled:", doubled)
    
    // StringSlice
    fruits := StringSlice{"banana", "apple", "cherry", "date"}
    
    fmt.Println("\nFruits:", fruits)
    fmt.Println("Contains apple:", fruits.Contains("apple"))
    fmt.Println("Sorted:", fruits.Sorted())
    
    // Registry
    reg := make(Registry)
    reg.Register("server", "localhost:8080")
    reg.Register("debug", true)
    reg.Register("maxRetries", 3)
    
    fmt.Println("\nRegistry keys:", reg.Keys())
    if v, ok := reg.Get("server"); ok {
        fmt.Printf("server: %v\n", v)
    }
}
```

---

## 21.2 Type Aliases

### 21.2.1 Type Alias vs Named Type

```go
package main

import "fmt"

// Named type - type ใหม่จริงๆ
type Celsius float64

// Type alias - แค่ชื่อใหม่สำหรับ type เดิม
type MyFloat64 = float64 // alias

// ตัวอย่าง: byte = uint8 และ rune = int32 ใน stdlib

func main() {
    // Named type
    var c Celsius = 25.0
    var f float64 = 25.0
    
    // ต้อง explicit conversion
    c2 := Celsius(f)
    f2 := float64(c)
    fmt.Println(c2, f2)
    
    // Type alias
    var mf MyFloat64 = 3.14
    var ff float64 = mf // ไม่ต้อง conversion
    var mf2 MyFloat64 = ff // ไม่ต้อง conversion
    fmt.Println(mf, ff, mf2)
    
    // Alias ใช้งานได้เหมือน original type
    fmt.Printf("type of c: %T\n", c)    // main.Celsius
    fmt.Printf("type of mf: %T\n", mf)  // float64 (ไม่ใช่ main.MyFloat64)
    
    // การใช้งาน alias ที่พบบ่อย
    type Handler = func(string) error  // alias สำหรับ function type
    
    var h Handler = func(s string) error {
        fmt.Println("handling:", s)
        return nil
    }
    h("request")
    
    // Alias สำหรับ refactoring
    // package old → new โดยไม่ break code
    // type OldType = NewPackage.NewType
}
```

---

## 21.3 Function Types

### 21.3.1 Function Types พื้นฐาน

```go
package main

import (
    "fmt"
    "sort"
)

// Function type
type Predicate func(int) bool
type Transformer func(int) int
type Comparator func(a, b int) bool
type Handler func(string) (string, error)
type Middleware func(Handler) Handler

// Named function type สามารถมี methods ได้
type PredicateFunc func(int) bool

func (p PredicateFunc) And(other PredicateFunc) PredicateFunc {
    return func(n int) bool {
        return p(n) && other(n)
    }
}

func (p PredicateFunc) Or(other PredicateFunc) PredicateFunc {
    return func(n int) bool {
        return p(n) || other(n)
    }
}

func (p PredicateFunc) Not() PredicateFunc {
    return func(n int) bool {
        return !p(n)
    }
}

// Higher-order functions
func filter(nums []int, pred Predicate) []int {
    var result []int
    for _, n := range nums {
        if pred(n) {
            result = append(result, n)
        }
    }
    return result
}

func transform(nums []int, fn Transformer) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = fn(n)
    }
    return result
}

func sortWith(nums []int, less Comparator) []int {
    result := make([]int, len(nums))
    copy(result, nums)
    sort.Slice(result, func(i, j int) bool {
        return less(result[i], result[j])
    })
    return result
}

// Middleware pattern
func chain(h Handler, middlewares ...Middleware) Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func loggingMiddleware(next Handler) Handler {
    return func(req string) (string, error) {
        fmt.Printf("[LOG] request: %s\n", req)
        resp, err := next(req)
        fmt.Printf("[LOG] response: %s, err: %v\n", resp, err)
        return resp, err
    }
}

func authMiddleware(next Handler) Handler {
    return func(req string) (string, error) {
        if req == "unauthorized" {
            return "", fmt.Errorf("unauthorized")
        }
        return next(req)
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    // Filter
    isEven := Predicate(func(n int) bool { return n%2 == 0 })
    evens := filter(nums, isEven)
    fmt.Println("Evens:", evens)
    
    // Transform
    doubled := transform(nums, Transformer(func(n int) int { return n * 2 }))
    fmt.Println("Doubled:", doubled)
    
    // Sort
    reversed := sortWith(nums, func(a, b int) bool { return a > b })
    fmt.Println("Reversed:", reversed)
    
    // PredicateFunc chaining
    isPositive := PredicateFunc(func(n int) bool { return n > 0 })
    isSmall := PredicateFunc(func(n int) bool { return n < 5 })
    
    combined := isPositive.And(isSmall)
    fmt.Println("\nPositive AND Small:")
    for _, n := range []int{-1, 0, 1, 3, 5, 10} {
        fmt.Printf("  %d: %v\n", n, combined(n))
    }
    
    notEven := isEven.Not
    evens2 := filter(nums, Predicate(PredicateFunc(func(n int) bool { return n%2 == 0 }).Not()))
    fmt.Println("\nOdds:", evens2)
    _ = notEven
    
    // Middleware
    fmt.Println("\n=== Middleware ===")
    baseHandler := Handler(func(req string) (string, error) {
        return "response for: " + req, nil
    })
    
    handler := chain(baseHandler, loggingMiddleware, authMiddleware)
    
    handler("hello")
    fmt.Println()
    handler("unauthorized")
}
```

---

## 21.4 Interface Types Deep Dive

### 21.4.1 Interface ภายใน

```go
package main

import (
    "fmt"
    "math"
)

// Interface เป็น pair (type, value)
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Nil interface
func demonstrateNilInterface() {
    var s Shape // nil interface (type=nil, value=nil)
    fmt.Printf("nil interface: %v, isNil: %v\n", s, s == nil)
    
    // ระวัง! nil pointer ใน interface ไม่เท่ากับ nil interface!
    var c *Circle = nil
    s = c // interface มี type (*Circle) แต่ value เป็น nil
    
    fmt.Printf("nil pointer in interface: %v, isNil: %v\n", s, s == nil)
    // s != nil แม้ว่า c จะเป็น nil!
}

// Interface embedding
type Stringer interface {
    String() string
}

type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader // embed Reader
    Writer // embed Writer
}

type ReadWriteStringer interface {
    ReadWriter
    Stringer
}

// Implicit interface satisfaction
type Point struct {
    X, Y float64
}

func (p Point) String() string {
    return fmt.Sprintf("(%.1f, %.1f)", p.X, p.Y)
}

func (p Point) Distance() float64 {
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func main() {
    demonstrateNilInterface()
    
    shapes := []Shape{
        Circle{Radius: 5},
        Rectangle{Width: 4, Height: 6},
        Circle{Radius: 3},
    }
    
    for _, s := range shapes {
        fmt.Printf("%T: area=%.2f, perimeter=%.2f\n",
            s, s.Area(), s.Perimeter())
    }
    
    // Interface comparison
    var s1 Shape = Circle{Radius: 5}
    var s2 Shape = Circle{Radius: 5}
    fmt.Printf("\ns1 == s2: %v\n", s1 == s2) // true (same type and value)
    
    p := Point{3, 4}
    fmt.Printf("\nPoint: %s, Distance: %.2f\n", p, p.Distance())
    
    // Point implements Stringer implicitly
    var stringer Stringer = p
    fmt.Println("As Stringer:", stringer.String())
}
```

### 21.4.2 Interface Composition

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

// ออกแบบ interface ขนาดเล็กๆ แล้วนำมา compose
type Opener interface {
    Open() error
}

type Closer interface {
    Close() error
}

type Pinger interface {
    Ping() error
}

type Connection interface {
    Opener
    Closer
    Pinger
    io.ReadWriter
}

// Database connection ที่ implement Connection
type DBConnection struct {
    host   string
    port   int
    open   bool
    buffer strings.Builder
}

func (d *DBConnection) Open() error {
    d.open = true
    fmt.Printf("Connected to %s:%d\n", d.host, d.port)
    return nil
}

func (d *DBConnection) Close() error {
    d.open = false
    fmt.Printf("Disconnected from %s:%d\n", d.host, d.port)
    return nil
}

func (d *DBConnection) Ping() error {
    if !d.open {
        return fmt.Errorf("connection is closed")
    }
    fmt.Println("PING OK")
    return nil
}

func (d *DBConnection) Read(p []byte) (n int, err error) {
    // simulate read
    data := "database response"
    n = copy(p, data)
    return n, io.EOF
}

func (d *DBConnection) Write(p []byte) (n int, err error) {
    d.buffer.Write(p)
    return len(p), nil
}

func useConnection(conn Connection) {
    conn.Open()
    defer conn.Close()
    
    conn.Ping()
    
    conn.Write([]byte("SELECT * FROM users"))
    
    buf := make([]byte, 100)
    n, _ := conn.Read(buf)
    fmt.Printf("Response: %s\n", buf[:n])
}

func main() {
    conn := &DBConnection{host: "localhost", port: 5432}
    useConnection(conn)
}
```

---

## 21.5 Reflection (reflect Package)

### 21.5.1 reflect พื้นฐาน

```go
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    Name    string `json:"name" validate:"required"`
    Age     int    `json:"age" validate:"min=0,max=150"`
    Email   string `json:"email" validate:"email"`
    private string // ไม่ exported
}

func (p Person) Greet() string {
    return fmt.Sprintf("Hello, I'm %s", p.Name)
}

func inspectValue(v interface{}) {
    t := reflect.TypeOf(v)
    val := reflect.ValueOf(v)
    
    fmt.Printf("Type: %v\n", t)
    fmt.Printf("Kind: %v\n", t.Kind())
    fmt.Printf("Value: %v\n", val)
    
    if t.Kind() == reflect.Struct {
        fmt.Printf("Num fields: %d\n", t.NumField())
        
        for i := 0; i < t.NumField(); i++ {
            field := t.Field(i)
            value := val.Field(i)
            
            fmt.Printf("\nField: %s\n", field.Name)
            fmt.Printf("  Type: %v\n", field.Type)
            fmt.Printf("  Kind: %v\n", field.Type.Kind())
            
            // แสดงค่าถ้า exported
            if field.IsExported() {
                fmt.Printf("  Value: %v\n", value)
            } else {
                fmt.Printf("  Value: (unexported)\n")
            }
            
            // Tags
            if jsonTag := field.Tag.Get("json"); jsonTag != "" {
                fmt.Printf("  JSON tag: %s\n", jsonTag)
            }
            if validateTag := field.Tag.Get("validate"); validateTag != "" {
                fmt.Printf("  Validate tag: %s\n", validateTag)
            }
        }
        
        // Methods
        fmt.Printf("\nMethods: %d\n", t.NumMethod())
        for i := 0; i < t.NumMethod(); i++ {
            method := t.Method(i)
            fmt.Printf("  %s\n", method.Name)
        }
    }
}

func main() {
    p := Person{
        Name:  "Alice",
        Age:   30,
        Email: "alice@example.com",
    }
    
    inspectValue(p)
}
```

### 21.5.2 reflect.Value - อ่านและแก้ไขข้อมูล

```go
package main

import (
    "fmt"
    "reflect"
)

type Config struct {
    Host     string
    Port     int
    Debug    bool
    Tags     []string
    Settings map[string]string
}

func setField(obj interface{}, fieldName string, value interface{}) error {
    // ต้องส่ง pointer เพื่อแก้ไขได้
    val := reflect.ValueOf(obj)
    if val.Kind() != reflect.Ptr || val.IsNil() {
        return fmt.Errorf("ต้องส่ง pointer ที่ไม่ nil")
    }
    
    // Dereference pointer
    val = val.Elem()
    if val.Kind() != reflect.Struct {
        return fmt.Errorf("ต้องเป็น struct")
    }
    
    // หา field
    field := val.FieldByName(fieldName)
    if !field.IsValid() {
        return fmt.Errorf("ไม่พบ field %q", fieldName)
    }
    if !field.CanSet() {
        return fmt.Errorf("ไม่สามารถแก้ไข field %q (unexported)", fieldName)
    }
    
    // ตรวจสอบ type compatibility
    rv := reflect.ValueOf(value)
    if field.Type() != rv.Type() {
        // ลอง convert
        if rv.Type().ConvertibleTo(field.Type()) {
            rv = rv.Convert(field.Type())
        } else {
            return fmt.Errorf("type mismatch: %v ≠ %v", rv.Type(), field.Type())
        }
    }
    
    field.Set(rv)
    return nil
}

func getField(obj interface{}, fieldName string) (interface{}, error) {
    val := reflect.ValueOf(obj)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }
    
    field := val.FieldByName(fieldName)
    if !field.IsValid() {
        return nil, fmt.Errorf("ไม่พบ field %q", fieldName)
    }
    if !field.CanInterface() {
        return nil, fmt.Errorf("ไม่สามารถอ่าน field %q", fieldName)
    }
    
    return field.Interface(), nil
}

func cloneStruct(src interface{}) interface{} {
    srcVal := reflect.ValueOf(src)
    if srcVal.Kind() == reflect.Ptr {
        srcVal = srcVal.Elem()
    }
    
    // สร้าง instance ใหม่
    dstVal := reflect.New(srcVal.Type()).Elem()
    dstVal.Set(srcVal) // copy ทุก field
    
    return dstVal.Interface()
}

func main() {
    cfg := &Config{
        Host:  "localhost",
        Port:  8080,
        Debug: false,
        Tags:  []string{"web", "api"},
        Settings: map[string]string{"timeout": "30s"},
    }
    
    fmt.Printf("Before: %+v\n", *cfg)
    
    // แก้ไข fields
    setField(cfg, "Host", "production.example.com")
    setField(cfg, "Port", 443)
    setField(cfg, "Debug", true)
    
    fmt.Printf("After: %+v\n", *cfg)
    
    // อ่าน fields
    host, _ := getField(cfg, "Host")
    port, _ := getField(cfg, "Port")
    fmt.Printf("\nHost: %v, Port: %v\n", host, port)
    
    // Clone
    original := Config{Host: "original", Port: 9090}
    cloned := cloneStruct(original).(Config)
    cloned.Host = "cloned"
    
    fmt.Printf("\nOriginal: %v\n", original.Host)
    fmt.Printf("Cloned: %v\n", cloned.Host)
}
```

### 21.5.3 reflect สำหรับ dynamic method calls

```go
package main

import (
    "fmt"
    "reflect"
)

type Calculator struct{}

func (c Calculator) Add(a, b int) int      { return a + b }
func (c Calculator) Multiply(a, b int) int { return a * b }
func (c Calculator) Power(base, exp int) int {
    result := 1
    for i := 0; i < exp; i++ {
        result *= base
    }
    return result
}

func callMethod(obj interface{}, methodName string, args ...interface{}) ([]interface{}, error) {
    val := reflect.ValueOf(obj)
    method := val.MethodByName(methodName)
    
    if !method.IsValid() {
        return nil, fmt.Errorf("method %q not found", methodName)
    }
    
    // สร้าง arguments
    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
    }
    
    // เรียก method
    out := method.Call(in)
    
    // แปลง results
    results := make([]interface{}, len(out))
    for i, v := range out {
        results[i] = v.Interface()
    }
    
    return results, nil
}

func main() {
    calc := Calculator{}
    
    methods := []struct {
        name string
        args []interface{}
    }{
        {"Add", []interface{}{3, 4}},
        {"Multiply", []interface{}{5, 6}},
        {"Power", []interface{}{2, 10}},
    }
    
    for _, m := range methods {
        results, err := callMethod(calc, m.name, m.args...)
        if err != nil {
            fmt.Printf("error: %v\n", err)
            continue
        }
        fmt.Printf("%s(%v) = %v\n", m.name, m.args, results)
    }
    
    // reflect.DeepEqual
    fmt.Println("\n=== DeepEqual ===")
    a := []int{1, 2, 3}
    b := []int{1, 2, 3}
    c := []int{1, 2, 4}
    
    fmt.Printf("a == b: %v\n", reflect.DeepEqual(a, b)) // true
    fmt.Printf("a == c: %v\n", reflect.DeepEqual(a, c)) // false
    
    m1 := map[string]int{"a": 1, "b": 2}
    m2 := map[string]int{"b": 2, "a": 1}
    fmt.Printf("m1 == m2: %v\n", reflect.DeepEqual(m1, m2)) // true
}
```

---

## 21.6 Type Assertions

### 21.6.1 Type Assertion พื้นฐาน

```go
package main

import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
}

type Circle struct{ Radius float64 }
type Rectangle struct{ Width, Height float64 }
type Triangle struct{ Base, Height float64 }

func (c Circle) Area() float64    { return math.Pi * c.Radius * c.Radius }
func (r Rectangle) Area() float64 { return r.Width * r.Height }
func (t Triangle) Area() float64  { return 0.5 * t.Base * t.Height }

func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

type Perimeter interface {
    Perimeter() float64
}

func describe(s Shape) {
    fmt.Printf("Type: %T, Area: %.2f\n", s, s.Area())
    
    // Type assertion (panic ถ้าผิด type)
    if c, ok := s.(Circle); ok {
        fmt.Printf("  Circle radius: %.2f\n", c.Radius)
        fmt.Printf("  Circle perimeter: %.2f\n", c.Perimeter())
    }
    
    // Check if implements Perimeter
    if p, ok := s.(Perimeter); ok {
        fmt.Printf("  Perimeter: %.2f\n", p.Perimeter())
    }
}

// Type switch
func processShape(s Shape) string {
    switch v := s.(type) {
    case Circle:
        return fmt.Sprintf("Circle with radius %.2f", v.Radius)
    case Rectangle:
        return fmt.Sprintf("Rectangle %.2fx%.2f", v.Width, v.Height)
    case Triangle:
        return fmt.Sprintf("Triangle base=%.2f, height=%.2f", v.Base, v.Height)
    default:
        return fmt.Sprintf("Unknown shape: %T", v)
    }
}

func main() {
    shapes := []Shape{
        Circle{Radius: 5},
        Rectangle{Width: 4, Height: 6},
        Triangle{Base: 3, Height: 8},
    }
    
    for _, s := range shapes {
        describe(s)
        fmt.Println(" →", processShape(s))
        fmt.Println()
    }
    
    // Unsafe assertion (panics if wrong)
    var s Shape = Circle{Radius: 3}
    c := s.(Circle)
    fmt.Printf("Direct assertion: Circle radius = %.2f\n", c.Radius)
    
    // Safe assertion
    if r, ok := s.(Rectangle); ok {
        fmt.Println("Is rectangle:", r)
    } else {
        fmt.Println("Not a rectangle (as expected)")
    }
}
```

---

## 21.7 any Type (Go 1.18+)

### 21.7.1 any = interface{}

```go
package main

import "fmt"

// any คือ alias สำหรับ interface{}
// type any = interface{}

func printAnything(v any) {
    fmt.Printf("Value: %v, Type: %T\n", v, v)
}

func toStringSlice(items []any) []string {
    result := make([]string, len(items))
    for i, v := range items {
        result[i] = fmt.Sprintf("%v", v)
    }
    return result
}

// Generic container ด้วย any (Go 1.18+ ใช้ generics แทน)
type Container struct {
    items []any
}

func (c *Container) Add(item any) {
    c.items = append(c.items, item)
}

func (c *Container) Get(index int) any {
    return c.items[index]
}

func (c *Container) Len() int {
    return len(c.items)
}

func (c *Container) Filter(pred func(any) bool) *Container {
    result := &Container{}
    for _, item := range c.items {
        if pred(item) {
            result.Add(item)
        }
    }
    return result
}

func main() {
    // any type
    values := []any{
        42,
        "hello",
        3.14,
        true,
        []int{1, 2, 3},
        map[string]int{"a": 1},
    }
    
    for _, v := range values {
        printAnything(v)
    }
    
    // Container
    fmt.Println("\n=== Container ===")
    c := &Container{}
    c.Add(1)
    c.Add("hello")
    c.Add(3.14)
    c.Add(true)
    
    fmt.Printf("Len: %d\n", c.Len())
    
    // Type switch กับ any
    for i := 0; i < c.Len(); i++ {
        v := c.Get(i)
        switch val := v.(type) {
        case int:
            fmt.Printf("int: %d\n", val)
        case string:
            fmt.Printf("string: %q\n", val)
        case float64:
            fmt.Printf("float64: %.2f\n", val)
        case bool:
            fmt.Printf("bool: %v\n", val)
        default:
            fmt.Printf("other: %T = %v\n", val, val)
        }
    }
    
    // Filter
    nums := &Container{}
    for _, n := range []any{1, 2, 3, 4, 5, 6, 7, 8, 9, 10} {
        nums.Add(n)
    }
    
    evens := nums.Filter(func(v any) bool {
        if n, ok := v.(int); ok {
            return n%2 == 0
        }
        return false
    })
    
    fmt.Printf("\nEvens: ")
    for i := 0; i < evens.Len(); i++ {
        fmt.Printf("%v ", evens.Get(i))
    }
    fmt.Println()
}
```

---

## 21.8 Comparable Types

### 21.8.1 Comparable vs Non-Comparable

```go
package main

import "fmt"

func main() {
    // Comparable types: สามารถใช้ == และ != ได้
    // - bool, int, float, complex, string, pointer, channel
    // - struct ที่ทุก field เป็น comparable
    // - array ของ comparable types
    
    // Non-comparable: slice, map, function
    
    // Comparable types
    a, b := 42, 42
    fmt.Println("int:", a == b) // true
    
    s1, s2 := "hello", "hello"
    fmt.Println("string:", s1 == s2) // true
    
    type Point struct{ X, Y int }
    p1, p2 := Point{1, 2}, Point{1, 2}
    fmt.Println("struct:", p1 == p2) // true
    
    arr1, arr2 := [3]int{1, 2, 3}, [3]int{1, 2, 3}
    fmt.Println("array:", arr1 == arr2) // true
    
    // Non-comparable
    sl1 := []int{1, 2, 3}
    _ = sl1
    // sl1 == sl2 → compile error: invalid operation
    
    // ใช้ reflect.DeepEqual สำหรับ non-comparable
    
    // Map keys ต้องเป็น comparable
    type ComparableKey struct {
        ID   int
        Name string
    }
    
    m := map[ComparableKey]string{
        {1, "Alice"}: "admin",
        {2, "Bob"}:   "user",
    }
    
    key := ComparableKey{1, "Alice"}
    fmt.Println("\nMap value:", m[key])
    
    // Interface comparison
    var i1, i2 interface{} = 42, 42
    fmt.Println("\nInterface equal:", i1 == i2) // true
    
    var i3, i4 interface{} = []int{1}, []int{1}
    // i3 == i4 → runtime panic: comparing incomparable types
    // ต้องระวังเรื่องนี้!
    
    safeCompare := func(a, b interface{}) (equal bool, ok bool) {
        defer func() {
            if r := recover(); r != nil {
                ok = false
            }
        }()
        return a == b, true
    }
    
    eq, ok := safeCompare(42, 42)
    fmt.Printf("safe compare ints: equal=%v, ok=%v\n", eq, ok)
    
    eq2, ok2 := safeCompare(i3, i4)
    fmt.Printf("safe compare slices: equal=%v, ok=%v\n", eq2, ok2)
}
```

---

## 21.9 Workshop: Type System

```go
package main

import (
    "fmt"
    "reflect"
    "strconv"
    "strings"
)

// Schema validation ด้วย reflection

type FieldValidator struct {
    Required bool
    MinLen   int
    MaxLen   int
    Min      float64
    Max      float64
    Pattern  string
}

type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

type Validator struct {
    rules map[string]FieldValidator
}

func NewValidator() *Validator {
    return &Validator{rules: make(map[string]FieldValidator)}
}

func (v *Validator) AddRule(field string, rule FieldValidator) {
    v.rules[field] = rule
}

func (v *Validator) Validate(obj interface{}) []ValidationError {
    var errors []ValidationError
    
    val := reflect.ValueOf(obj)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }
    
    if val.Kind() != reflect.Struct {
        return []ValidationError{{Field: "_", Message: "ต้องเป็น struct"}}
    }
    
    t := val.Type()
    
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldVal := val.Field(i)
        
        if !field.IsExported() {
            continue
        }
        
        // ดึง validation tag
        validateTag := field.Tag.Get("validate")
        if validateTag == "" {
            continue
        }
        
        // parse validation rules จาก tag
        rule := parseValidateTag(validateTag)
        fieldName := field.Tag.Get("json")
        if fieldName == "" {
            fieldName = strings.ToLower(field.Name)
        }
        // ตัด ,omitempty ถ้ามี
        if idx := strings.Index(fieldName, ","); idx != -1 {
            fieldName = fieldName[:idx]
        }
        
        errs := validateField(fieldName, fieldVal, rule)
        errors = append(errors, errs...)
    }
    
    return errors
}

func parseValidateTag(tag string) FieldValidator {
    rule := FieldValidator{}
    parts := strings.Split(tag, ",")
    
    for _, part := range parts {
        kv := strings.SplitN(part, "=", 2)
        key := strings.TrimSpace(kv[0])
        
        switch key {
        case "required":
            rule.Required = true
        case "min":
            if len(kv) > 1 {
                if n, err := strconv.ParseFloat(kv[1], 64); err == nil {
                    rule.Min = n
                }
            }
        case "max":
            if len(kv) > 1 {
                if n, err := strconv.ParseFloat(kv[1], 64); err == nil {
                    rule.Max = n
                }
            }
        case "minlen":
            if len(kv) > 1 {
                if n, err := strconv.Atoi(kv[1]); err == nil {
                    rule.MinLen = n
                }
            }
        case "maxlen":
            if len(kv) > 1 {
                if n, err := strconv.Atoi(kv[1]); err == nil {
                    rule.MaxLen = n
                }
            }
        }
    }
    
    return rule
}

func validateField(name string, val reflect.Value, rule FieldValidator) []ValidationError {
    var errors []ValidationError
    
    switch val.Kind() {
    case reflect.String:
        s := val.String()
        if rule.Required && s == "" {
            errors = append(errors, ValidationError{name, "ต้องไม่ว่างเปล่า"})
        }
        if rule.MinLen > 0 && len(s) < rule.MinLen {
            errors = append(errors, ValidationError{
                name, fmt.Sprintf("ต้องมีอย่างน้อย %d ตัวอักษร", rule.MinLen),
            })
        }
        if rule.MaxLen > 0 && len(s) > rule.MaxLen {
            errors = append(errors, ValidationError{
                name, fmt.Sprintf("ต้องมีไม่เกิน %d ตัวอักษร", rule.MaxLen),
            })
        }
        
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        n := float64(val.Int())
        if rule.Required && n == 0 {
            errors = append(errors, ValidationError{name, "ต้องไม่เป็น 0"})
        }
        if rule.Min != 0 && n < rule.Min {
            errors = append(errors, ValidationError{
                name, fmt.Sprintf("ต้องมากกว่าหรือเท่ากับ %.0f", rule.Min),
            })
        }
        if rule.Max != 0 && n > rule.Max {
            errors = append(errors, ValidationError{
                name, fmt.Sprintf("ต้องน้อยกว่าหรือเท่ากับ %.0f", rule.Max),
            })
        }
    }
    
    return errors
}

// Test structs
type UserRegistration struct {
    Username string `json:"username" validate:"required,minlen=3,maxlen=20"`
    Email    string `json:"email" validate:"required"`
    Password string `json:"password" validate:"required,minlen=8"`
    Age      int    `json:"age" validate:"required,min=18,max=120"`
    Bio      string `json:"bio" validate:"maxlen=500"`
}

func main() {
    v := NewValidator()
    
    // Test valid user
    validUser := UserRegistration{
        Username: "alice123",
        Email:    "alice@example.com",
        Password: "SecurePass123!",
        Age:      25,
        Bio:      "I love Go!",
    }
    
    fmt.Println("=== Valid User ===")
    if errors := v.Validate(validUser); len(errors) > 0 {
        for _, e := range errors {
            fmt.Printf("  Error: %v\n", e)
        }
    } else {
        fmt.Println("  Validation passed!")
    }
    
    // Test invalid user
    invalidUser := UserRegistration{
        Username: "ab",        // too short
        Email:    "",          // required
        Password: "weak",     // too short
        Age:      15,          // too young
    }
    
    fmt.Println("\n=== Invalid User ===")
    if errors := v.Validate(invalidUser); len(errors) > 0 {
        for _, e := range errors {
            fmt.Printf("  Error: %v\n", e)
        }
    } else {
        fmt.Println("  Validation passed!")
    }
    
    // Inspect struct
    fmt.Println("\n=== Struct Inspection ===")
    t := reflect.TypeOf(UserRegistration{})
    for i := 0; i < t.NumField(); i++ {
        f := t.Field(i)
        fmt.Printf("  %s: json=%q, validate=%q\n",
            f.Name, f.Tag.Get("json"), f.Tag.Get("validate"))
    }
}
```

---

## สรุป

| Concept | การใช้งาน |
|---------|----------|
| Named Types | Type safety, domain modeling |
| Type Alias | Renaming, compatibility |
| Function Types | Callbacks, middlewares, higher-order |
| Interface | Polymorphism, abstraction |
| Reflection | Dynamic inspection, serialization |
| Type Assertion | Type switching, type checking |
| `any` | Generic containers (ก่อน generics) |
| Comparable | Map keys, == operator |

## Resources

- [Go spec: Types](https://go.dev/ref/spec#Types)
- [reflect package](https://pkg.go.dev/reflect)
- [Laws of reflection](https://go.dev/blog/laws-of-reflection)
- [Interface internals](https://research.swtch.com/interfaces)
