# Part 10: Pointers ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- อธิบายว่า pointer คืออะไรและทำงานอย่างไร
- ประกาศและใช้งาน pointer ได้อย่างถูกต้อง
- เข้าใจความแตกต่างระหว่าง pass by value และ pass by pointer
- ใช้ pointer กับ structs ได้
- จัดการกับ nil pointers อย่างปลอดภัย
- ตัดสินใจได้ว่าควรใช้ pointer หรือ value

---

## 10.1 Pointer คืออะไร?

**Pointer** คือตัวแปรที่เก็บ **memory address** ของตัวแปรอื่น แทนที่จะเก็บค่าโดยตรง

จินตนาการว่า:
- ตัวแปรปกติ = กล่องที่เก็บของ
- Pointer = กระดาษที่เขียนที่อยู่ของกล่องนั้น

```
Memory:
Address  | Value
---------|-------
0x1000   | 42      <- ตัวแปร x
0x1004   | 0x1000  <- pointer p ชี้ไปที่ x
```

```go
package main

import "fmt"

func main() {
    x := 42
    p := &x  // p เก็บ address ของ x
    
    fmt.Printf("ค่า x: %d\n", x)
    fmt.Printf("address ของ x: %p\n", &x)
    fmt.Printf("ค่าใน p (address): %p\n", p)
    fmt.Printf("ค่าที่ p ชี้ไป: %d\n", *p)  // dereference
}
```

---

## 10.2 ประกาศและใช้งาน Pointers

### Operator สำคัญ

| Operator | ความหมาย |
|----------|---------|
| `&` | Address-of operator - ดึง address ของตัวแปร |
| `*` | Dereference operator - ดึงค่าที่ pointer ชี้ไป |
| `*T` | Pointer type - pointer ที่ชี้ไปที่ type T |

```go
package main

import "fmt"

func main() {
    // ประกาศตัวแปร
    num := 100
    
    // สร้าง pointer ด้วย &
    var ptr *int = &num
    
    fmt.Printf("num = %d\n", num)
    fmt.Printf("&num = %p\n", &num)   // address ของ num
    fmt.Printf("ptr = %p\n", ptr)    // ค่าใน ptr (address)
    fmt.Printf("*ptr = %d\n", *ptr)  // ค่าที่ ptr ชี้ไป
    
    // แก้ไขค่าผ่าน pointer
    *ptr = 200
    fmt.Printf("\nหลังแก้ไขผ่าน pointer:\n")
    fmt.Printf("num = %d\n", num)    // 200 - ค่าเปลี่ยน!
    fmt.Printf("*ptr = %d\n", *ptr)  // 200
}
```

### Short declaration

```go
package main

import "fmt"

func main() {
    // ประกาศแบบสั้น
    name := "สมชาย"
    pName := &name
    
    fmt.Printf("name: %s\n", name)
    fmt.Printf("*pName: %s\n", *pName)
    
    // แก้ไขผ่าน pointer
    *pName = "สมหญิง"
    fmt.Printf("\nหลังแก้ไข:\n")
    fmt.Printf("name: %s\n", name)     // สมหญิง
    fmt.Printf("*pName: %s\n", *pName) // สมหญิง
    
    // Pointer types ต่างๆ
    i := 42
    f := 3.14
    b := true
    s := "hello"
    
    pi := &i
    pf := &f
    pb := &b
    ps := &s
    
    fmt.Printf("\n%T %T %T %T\n", pi, pf, pb, ps)
    // *int *float64 *bool *string
}
```

---

## 10.3 Dereferencing Pointers

```go
package main

import "fmt"

func main() {
    x := 10
    y := 20
    
    px := &x
    py := &y
    
    // อ่านค่า (read dereference)
    fmt.Printf("x=%d, y=%d\n", *px, *py)
    
    // เขียนค่า (write dereference)
    *px = 100
    *py = *px + *py  // 100 + 20 = 120
    
    fmt.Printf("x=%d, y=%d\n", x, y)
    
    // Swap สองค่าด้วย pointer
    *px, *py = *py, *px
    fmt.Printf("หลัง swap: x=%d, y=%d\n", x, y)
}
```

### Swap function ด้วย pointer

```go
package main

import "fmt"

// ต้องใช้ pointer จึงจะ swap ได้จริง
func swap(a, b *int) {
    *a, *b = *b, *a
}

// แบบ value - ไม่ได้ swap จริง
func swapWrong(a, b int) {
    a, b = b, a
    // แก้แค่ local copy
}

func main() {
    x, y := 10, 20
    fmt.Printf("ก่อน swap: x=%d, y=%d\n", x, y)
    
    swapWrong(x, y)
    fmt.Printf("หลัง swapWrong: x=%d, y=%d\n", x, y) // ไม่เปลี่ยน
    
    swap(&x, &y)
    fmt.Printf("หลัง swap: x=%d, y=%d\n", x, y) // เปลี่ยนแล้ว!
}
```

---

## 10.4 Pointers vs Values

ความแตกต่างหลักคือการ **ส่งค่าเข้า function**:

### Pass by Value (copy)

```go
package main

import "fmt"

type Rectangle struct {
    Width, Height float64
}

// รับ value - ได้รับ copy
func doubleByValue(r Rectangle) {
    r.Width *= 2
    r.Height *= 2
    fmt.Printf("Inside doubleByValue: %+v\n", r)
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Printf("ก่อน: %+v\n", rect)
    
    doubleByValue(rect)
    
    fmt.Printf("หลัง: %+v\n", rect) // ไม่เปลี่ยน!
}
```

### Pass by Pointer

```go
package main

import "fmt"

type Rectangle struct {
    Width, Height float64
}

// รับ pointer - แก้ไขได้
func doubleByPointer(r *Rectangle) {
    r.Width *= 2
    r.Height *= 2
}

func area(r *Rectangle) float64 {
    return r.Width * r.Height
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Printf("ก่อน: %+v\n", rect)
    fmt.Printf("พื้นที่: %.2f\n", area(&rect))
    
    doubleByPointer(&rect)
    
    fmt.Printf("หลัง: %+v\n", rect)  // เปลี่ยนแล้ว!
    fmt.Printf("พื้นที่ใหม่: %.2f\n", area(&rect))
}
```

### เปรียบเทียบ Performance

```go
package main

import "fmt"

// ใหญ่มาก - ควรใช้ pointer
type BigStruct struct {
    Data [1000]int
    Name string
    Info string
}

// แบบ value - copy ข้อมูล 1000 int ทุกครั้งที่เรียก
func processByValue(s BigStruct) string {
    return s.Name
}

// แบบ pointer - ส่งแค่ address (8 bytes)
func processByPointer(s *BigStruct) string {
    return s.Name
}

func main() {
    big := BigStruct{Name: "ข้อมูลใหญ่", Info: "..."}
    big.Data[0] = 1
    
    fmt.Println(processByValue(big))
    fmt.Println(processByPointer(&big))
}
```

---

## 10.5 Pointer to Struct

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func (p *Person) Birthday() {
    p.Age++
    fmt.Printf("Happy Birthday %s! อายุ %d ปีแล้ว\n", p.Name, p.Age)
}

func (p *Person) ChangeName(newName string) {
    p.Name = newName
}

func main() {
    // สร้าง pointer to struct
    p1 := &Person{Name: "สมชาย", Age: 24}
    
    fmt.Printf("p1: %+v\n", *p1)
    fmt.Printf("p1.Name: %s\n", p1.Name) // auto-dereference
    
    p1.Birthday() // auto-dereference: (*p1).Birthday()
    p1.Birthday()
    
    p1.ChangeName("สมศักดิ์")
    fmt.Printf("ชื่อใหม่: %s\n", p1.Name)
    
    // ตรวจสอบว่า pointer ชี้ไปที่ตัวแปรเดียวกัน
    p2 := p1  // p2 ชี้ไปที่ Person เดิม
    p2.Age = 100
    fmt.Printf("\nหลังแก้ผ่าน p2:\n")
    fmt.Printf("p1.Age: %d\n", p1.Age) // 100!
    fmt.Printf("p2.Age: %d\n", p2.Age) // 100
    
    // เพื่อ copy ต้องทำแบบนี้
    p3 := *p1 // copy ค่า
    p3.Age = 1
    fmt.Printf("\nหลัง copy p3:\n")
    fmt.Printf("p1.Age: %d\n", p1.Age) // ยังเป็น 100
    fmt.Printf("p3.Age: %d\n", p3.Age) // 1
}
```

---

## 10.6 Nil Pointers

Pointer ที่ไม่ได้ initialized จะมีค่าเป็น `nil`

```go
package main

import "fmt"

type Node struct {
    Value int
    Next  *Node // pointer ที่อาจเป็น nil
}

func main() {
    var p *int  // nil pointer
    fmt.Printf("p = %v\n", p) // <nil>
    fmt.Printf("p == nil: %v\n", p == nil) // true
    
    // อย่า dereference nil pointer! จะ panic
    // fmt.Println(*p) // panic: runtime error!
    
    // ตรวจสอบก่อนเสมอ
    if p != nil {
        fmt.Println("ค่า:", *p)
    } else {
        fmt.Println("pointer เป็น nil")
    }
    
    // Linked list ด้วย nil pointer
    n3 := &Node{Value: 30, Next: nil}
    n2 := &Node{Value: 20, Next: n3}
    n1 := &Node{Value: 10, Next: n2}
    
    // traverse
    fmt.Println("\nLinked List:")
    current := n1
    for current != nil {
        fmt.Printf("%d", current.Value)
        if current.Next != nil {
            fmt.Print(" -> ")
        }
        current = current.Next
    }
    fmt.Println()
}
```

### Safe Nil Handling

```go
package main

import "fmt"

type Config struct {
    Debug   bool
    Verbose bool
    Port    int
}

func getPort(cfg *Config) int {
    if cfg == nil {
        return 8080 // default
    }
    if cfg.Port == 0 {
        return 8080 // default
    }
    return cfg.Port
}

func printConfig(cfg *Config) {
    if cfg == nil {
        fmt.Println("Config: ใช้ default settings")
        return
    }
    fmt.Printf("Debug: %v, Verbose: %v, Port: %d\n",
        cfg.Debug, cfg.Verbose, cfg.Port)
}

func main() {
    // nil config
    printConfig(nil)
    fmt.Printf("Port: %d\n\n", getPort(nil))
    
    // actual config
    cfg := &Config{Debug: true, Port: 9090}
    printConfig(cfg)
    fmt.Printf("Port: %d\n", getPort(cfg))
}
```

---

## 10.7 new() Function

`new(T)` สร้าง pointer ไปที่ zero value ของ type T

```go
package main

import "fmt"

type Point struct {
    X, Y float64
}

func main() {
    // new() สร้าง pointer ไปที่ zero value
    p := new(int)
    fmt.Printf("p = %v, *p = %d\n", p, *p) // address, 0
    
    *p = 42
    fmt.Printf("*p = %d\n", *p)
    
    // new() กับ struct
    pt := new(Point)
    fmt.Printf("pt = %+v\n", *pt) // {X:0 Y:0}
    
    pt.X = 3.14
    pt.Y = 2.71
    fmt.Printf("pt = %+v\n", *pt)
    
    // เปรียบเทียบ new() กับ & literal
    p1 := new(Point)           // pointer ไปที่ zero value
    p2 := &Point{}             // เหมือนกัน
    p3 := &Point{X: 1, Y: 2}  // pointer ไปที่ initialized value
    
    fmt.Printf("p1 = %+v\n", *p1)
    fmt.Printf("p2 = %+v\n", *p2)
    fmt.Printf("p3 = %+v\n", *p3)
}
```

---

## 10.8 Pointer Receivers in Methods

เราใช้ pointer receiver เมื่อต้องการแก้ไข struct หรือ struct ใหญ่มาก

```go
package main

import (
    "fmt"
    "strings"
)

type Stack struct {
    items []int
    size  int
}

// Pointer receiver - แก้ไข struct ได้
func (s *Stack) Push(item int) {
    s.items = append(s.items, item)
    s.size++
}

func (s *Stack) Pop() (int, bool) {
    if s.size == 0 {
        return 0, false
    }
    item := s.items[s.size-1]
    s.items = s.items[:s.size-1]
    s.size--
    return item, true
}

func (s *Stack) Peek() (int, bool) {
    if s.size == 0 {
        return 0, false
    }
    return s.items[s.size-1], true
}

// Value receiver - แค่อ่านข้อมูล
func (s Stack) IsEmpty() bool {
    return s.size == 0
}

func (s Stack) Size() int {
    return s.size
}

func (s Stack) String() string {
    if s.IsEmpty() {
        return "Stack: []"
    }
    strs := make([]string, len(s.items))
    for i, v := range s.items {
        strs[i] = fmt.Sprintf("%d", v)
    }
    return "Stack: [" + strings.Join(strs, ", ") + "]"
}

func main() {
    s := Stack{}
    
    fmt.Printf("Empty: %v\n", s.IsEmpty())
    
    s.Push(10)
    s.Push(20)
    s.Push(30)
    s.Push(40)
    
    fmt.Println(s)
    fmt.Printf("Size: %d\n", s.Size())
    
    if top, ok := s.Peek(); ok {
        fmt.Printf("Top: %d\n", top)
    }
    
    fmt.Println("\nPopping:")
    for !s.IsEmpty() {
        if val, ok := s.Pop(); ok {
            fmt.Printf("  popped: %d\n", val)
        }
    }
    
    fmt.Printf("Empty: %v\n", s.IsEmpty())
}
```

---

## 10.9 Pointers กับ Slices และ Maps

Slices และ Maps มีลักษณะคล้าย pointer อยู่แล้ว

```go
package main

import "fmt"

// Slice เป็น reference type อยู่แล้ว
func modifySlice(s []int) {
    for i := range s {
        s[i] *= 2
    }
}

// Map เป็น reference type อยู่แล้ว
func modifyMap(m map[string]int) {
    m["new"] = 999
}

// แต่การ reassign ต้องใช้ pointer
func appendToSlice(s *[]int, val int) {
    *s = append(*s, val)
}

func main() {
    // Slice - modify ได้โดยไม่ต้องใช้ pointer
    nums := []int{1, 2, 3, 4, 5}
    fmt.Printf("ก่อน: %v\n", nums)
    modifySlice(nums)
    fmt.Printf("หลัง: %v\n", nums)  // เปลี่ยนแล้ว
    
    // Map - modify ได้โดยไม่ต้องใช้ pointer
    scores := map[string]int{"a": 90, "b": 85}
    fmt.Printf("\nก่อน: %v\n", scores)
    modifyMap(scores)
    fmt.Printf("หลัง: %v\n", scores)  // เปลี่ยนแล้ว
    
    // แต่ถ้าต้องการ reassign slice ต้องใช้ pointer
    list := []int{1, 2, 3}
    fmt.Printf("\nก่อน append: %v (len=%d)\n", list, len(list))
    appendToSlice(&list, 4)
    appendToSlice(&list, 5)
    fmt.Printf("หลัง append: %v (len=%d)\n", list, len(list))
}
```

---

## 10.10 เมื่อไหร่ควรใช้ Pointer

### ใช้ Pointer เมื่อ:

```go
package main

import "fmt"

// 1. ต้องการแก้ไขค่า
type Counter struct {
    count int
}

func (c *Counter) Increment() { // pointer receiver
    c.count++
}

// 2. Struct ใหญ่มาก (หลีกเลี่ยง copy overhead)
type LargeData struct {
    buffer [1024]byte
    data   [512]int
}

func processLarge(d *LargeData) { // ส่ง pointer แทน copy
    d.buffer[0] = 1
}

// 3. Optional value (nil หมายถึงไม่มีค่า)
type User struct {
    Name  string
    Email *string // optional - nil ถ้าไม่มี email
}

func main() {
    // Counter
    c := &Counter{}
    c.Increment()
    c.Increment()
    c.Increment()
    fmt.Printf("Count: %d\n", c.count)
    
    // Optional email
    u1 := User{Name: "สมชาย"}
    email := "somchai@example.com"
    u2 := User{Name: "สมหญิง", Email: &email}
    
    fmt.Printf("\nu1 email: %v\n", u1.Email)  // <nil>
    if u2.Email != nil {
        fmt.Printf("u2 email: %s\n", *u2.Email)
    }
}
```

### ไม่ควรใช้ Pointer เมื่อ:

```go
package main

import "fmt"

// Simple types - ไม่จำเป็นต้องใช้ pointer
func addWrong(a, b *int) *int {
    result := *a + *b
    return &result  // ไม่ดี - ซับซ้อนเกินไป
}

func addBetter(a, b int) int {
    return a + b  // ดีกว่า - simple และชัดเจน
}

// Small structs ที่ไม่ต้องแก้ไข
type Point struct {
    X, Y int
}

func distanceFromOriginWrong(p *Point) float64 {
    // ไม่จำเป็นต้องใช้ pointer ถ้าไม่แก้ไข
    return float64(p.X*p.X + p.Y*p.Y)
}

func distanceFromOriginBetter(p Point) float64 {
    return float64(p.X*p.X + p.Y*p.Y)  // ดีกว่า
}

func main() {
    a, b := 10, 20
    fmt.Println(addBetter(a, b))
    
    p := Point{3, 4}
    fmt.Printf("Distance^2: %.0f\n", distanceFromOriginBetter(p))
}
```

---

## 10.11 Double Pointers

```go
package main

import "fmt"

func changePointer(pp **int, newVal int) {
    newInt := newVal
    *pp = &newInt
}

func main() {
    x := 10
    p := &x
    pp := &p  // pointer to pointer
    
    fmt.Printf("x = %d\n", x)
    fmt.Printf("*p = %d\n", *p)
    fmt.Printf("**pp = %d\n", **pp)
    
    // แก้ไขค่าผ่าน double pointer
    **pp = 100
    fmt.Printf("\nหลังแก้ **pp = 100:\n")
    fmt.Printf("x = %d\n", x)    // 100
    fmt.Printf("*p = %d\n", *p)  // 100
    
    // เปลี่ยน pointer ผ่าน double pointer
    y := 999
    *pp = &y
    fmt.Printf("\nหลังเปลี่ยน pointer:\n")
    fmt.Printf("*p = %d\n", *p)   // 999 (p ชี้ไปที่ y แล้ว)
    fmt.Printf("x = %d\n", x)     // ยังเป็น 100
}
```

---

## 10.12 Pointers ใน Concurrent Programming

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Final count: %d\n", counter.Value())
}
```

---

## 10.13 Pointer Patterns ทั่วไป

### Functional Options Pattern

```go
package main

import "fmt"

type Server struct {
    host    string
    port    int
    timeout int
    maxConn int
}

type ServerOption func(*Server)

func WithHost(host string) ServerOption {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) ServerOption {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout int) ServerOption {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30,
        maxConn: 100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

func (s *Server) Start() {
    fmt.Printf("Server starting at %s:%d\n", s.host, s.port)
    fmt.Printf("Timeout: %ds, MaxConn: %d\n", s.timeout, s.maxConn)
}

func main() {
    // Default server
    s1 := NewServer()
    s1.Start()
    
    fmt.Println()
    
    // Custom server
    s2 := NewServer(
        WithHost("0.0.0.0"),
        WithPort(9000),
        WithTimeout(60),
    )
    s2.Start()
}
```

### Builder Pattern ด้วย Pointer

```go
package main

import "fmt"

type QueryBuilder struct {
    table  string
    where  []string
    fields []string
    limit  int
    offset int
}

func NewQuery(table string) *QueryBuilder {
    return &QueryBuilder{
        table:  table,
        fields: []string{"*"},
    }
}

func (q *QueryBuilder) Select(fields ...string) *QueryBuilder {
    q.fields = fields
    return q
}

func (q *QueryBuilder) Where(condition string) *QueryBuilder {
    q.where = append(q.where, condition)
    return q
}

func (q *QueryBuilder) Limit(n int) *QueryBuilder {
    q.limit = n
    return q
}

func (q *QueryBuilder) Offset(n int) *QueryBuilder {
    q.offset = n
    return q
}

func (q *QueryBuilder) Build() string {
    query := fmt.Sprintf("SELECT %v FROM %s", q.fields, q.table)
    
    if len(q.where) > 0 {
        query += " WHERE "
        for i, w := range q.where {
            if i > 0 {
                query += " AND "
            }
            query += w
        }
    }
    
    if q.limit > 0 {
        query += fmt.Sprintf(" LIMIT %d", q.limit)
    }
    if q.offset > 0 {
        query += fmt.Sprintf(" OFFSET %d", q.offset)
    }
    
    return query
}

func main() {
    // Method chaining ด้วย pointer receiver
    query := NewQuery("users").
        Select("id", "name", "email").
        Where("age > 18").
        Where("active = true").
        Limit(10).
        Offset(20).
        Build()
    
    fmt.Println(query)
}
```

---

## 10.14 Memory Layout

```go
package main

import (
    "fmt"
    "unsafe"
)

type Example struct {
    A int8    // 1 byte
    B int64   // 8 bytes
    C int8    // 1 byte
    D int32   // 4 bytes
}

func main() {
    var e Example
    
    fmt.Printf("Size of Example: %d bytes\n", unsafe.Sizeof(e))
    fmt.Printf("Address of A: %p\n", &e.A)
    fmt.Printf("Address of B: %p\n", &e.B)
    fmt.Printf("Address of C: %p\n", &e.C)
    fmt.Printf("Address of D: %p\n", &e.D)
    
    // Pointer arithmetic (ระวัง!)
    x := [3]int{1, 2, 3}
    p := &x[0]
    fmt.Printf("\nArray addresses:\n")
    fmt.Printf("x[0]: %p = %d\n", &x[0], x[0])
    fmt.Printf("x[1]: %p = %d\n", &x[1], x[1])
    fmt.Printf("x[2]: %p = %d\n", &x[2], x[2])
    fmt.Printf("p points to: %d\n", *p)
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| `&` | ดึง address ของตัวแปร |
| `*` | Dereference - ดึงค่าที่ pointer ชี้ไป |
| `*T` | Pointer type ที่ชี้ไปที่ type T |
| nil pointer | Pointer ที่ไม่ชี้ไปที่ไหน - ต้องระวัง dereference |
| `new(T)` | สร้าง pointer ไปที่ zero value ของ T |
| Pass by pointer | ส่ง address เพื่อให้ function แก้ไขค่าได้ |
| Pointer receiver | Method ที่แก้ไข struct ได้ |
| Auto-dereference | Go อนุญาตให้ใช้ `.` กับ pointer โดยตรง |

### กฎง่ายๆ ในการตัดสินใจ:

1. **ใช้ Pointer เมื่อ**: ต้องแก้ไขค่า, struct ใหญ่, optional value
2. **ใช้ Value เมื่อ**: ข้อมูลเล็ก, ไม่ต้องแก้ไข, ต้องการ immutability

## Resources

- [Go Spec: Pointer types](https://go.dev/ref/spec#Pointer_types)
- [Go Tour: Pointers](https://go.dev/tour/moretypes/1)
- [Go by Example: Pointers](https://gobyexample.com/pointers)
- [Dave Cheney: Pointers in Go](https://dave.cheney.net/2017/04/26/understand-go-pointers-in-less-than-800-words-or-your-money-back)

---

*Part 10 จบแล้ว! ต่อไป Part 11: Methods and Interfaces*
