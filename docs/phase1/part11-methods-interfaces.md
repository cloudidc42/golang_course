# Part 11: Methods and Interfaces ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ประกาศและใช้งาน methods บน types ต่างๆ ได้
- เข้าใจความแตกต่างระหว่าง value receiver และ pointer receiver
- ประกาศและ implement interfaces
- ใช้ interface composition
- ใช้ empty interface และ type assertions
- เข้าใจ Stringer interface และ common interfaces
- ออกแบบ code ด้วย interface-based programming

---

## 11.1 Methods คืออะไร?

**Method** คือ function ที่ผูกกับ type หนึ่งๆ (receiver) ทำให้ type นั้นมี behavior

```go
package main

import "fmt"

type Rectangle struct {
    Width, Height float64
}

// Method บน Rectangle
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    
    fmt.Printf("พื้นที่: %.2f\n", rect.Area())
    fmt.Printf("เส้นรอบรูป: %.2f\n", rect.Perimeter())
}
```

---

## 11.2 Value Receiver vs Pointer Receiver

### Value Receiver

```go
package main

import "fmt"

type Temperature struct {
    Celsius float64
}

// Value receiver - อ่านค่าได้ ไม่แก้ไข original
func (t Temperature) ToFahrenheit() float64 {
    return t.Celsius*9/5 + 32
}

func (t Temperature) ToKelvin() float64 {
    return t.Celsius + 273.15
}

func (t Temperature) String() string {
    return fmt.Sprintf("%.2f°C", t.Celsius)
}

func main() {
    temp := Temperature{Celsius: 100}
    fmt.Printf("Celsius: %s\n", temp)
    fmt.Printf("Fahrenheit: %.2f°F\n", temp.ToFahrenheit())
    fmt.Printf("Kelvin: %.2f K\n", temp.ToKelvin())
}
```

### Pointer Receiver

```go
package main

import (
    "fmt"
    "strings"
)

type StringBuilder struct {
    data []string
}

// Pointer receiver - แก้ไข struct ได้
func (sb *StringBuilder) Write(s string) *StringBuilder {
    sb.data = append(sb.data, s)
    return sb // ส่งคืน pointer สำหรับ method chaining
}

func (sb *StringBuilder) WriteLn(s string) *StringBuilder {
    return sb.Write(s + "\n")
}

func (sb *StringBuilder) WriteF(format string, args ...interface{}) *StringBuilder {
    return sb.Write(fmt.Sprintf(format, args...))
}

func (sb *StringBuilder) Reset() *StringBuilder {
    sb.data = sb.data[:0]
    return sb
}

// Value receiver - แค่อ่าน
func (sb StringBuilder) String() string {
    return strings.Join(sb.data, "")
}

func (sb StringBuilder) Length() int {
    total := 0
    for _, s := range sb.data {
        total += len(s)
    }
    return total
}

func main() {
    var sb StringBuilder
    
    // Method chaining
    sb.WriteLn("สวัสดี Go!").
        WriteF("วันนี้วันที่ %s\n", "2024-01-15").
        WriteLn("-------------------").
        Write("จบแล้ว")
    
    fmt.Println(sb.String())
    fmt.Printf("ความยาวรวม: %d ตัวอักษร\n", sb.Length())
}
```

### กฎสำหรับการเลือก Receiver

```go
package main

import "fmt"

type Counter struct {
    n int
}

// ใช้ pointer receiver เมื่อ:
// 1. ต้องการแก้ไข state
func (c *Counter) Increment() {
    c.n++
}

func (c *Counter) Reset() {
    c.n = 0
}

// 2. Struct ใหญ่ (หลีกเลี่ยง copy)
// 3. Consistency - ถ้ามี method ที่ใช้ pointer receiver ควรใช้ทั้งหมด

// ใช้ value receiver เมื่อ:
// 1. ไม่แก้ไข state
func (c Counter) Value() int {
    return c.n
}

func (c Counter) IsZero() bool {
    return c.n == 0
}

func main() {
    c := Counter{}
    fmt.Printf("Zero: %v\n", c.IsZero())
    
    c.Increment()
    c.Increment()
    c.Increment()
    
    fmt.Printf("Value: %d\n", c.Value())
    
    c.Reset()
    fmt.Printf("After reset: %d\n", c.Value())
}
```

---

## 11.3 Methods บน Types ต่างๆ

### Methods บน Slice Type

```go
package main

import (
    "fmt"
    "sort"
)

type IntSlice []int

func (s IntSlice) Sum() int {
    total := 0
    for _, v := range s {
        total += v
    }
    return total
}

func (s IntSlice) Average() float64 {
    if len(s) == 0 {
        return 0
    }
    return float64(s.Sum()) / float64(len(s))
}

func (s IntSlice) Max() int {
    if len(s) == 0 {
        return 0
    }
    max := s[0]
    for _, v := range s[1:] {
        if v > max {
            max = v
        }
    }
    return max
}

func (s IntSlice) Min() int {
    if len(s) == 0 {
        return 0
    }
    min := s[0]
    for _, v := range s[1:] {
        if v < min {
            min = v
        }
    }
    return min
}

func (s IntSlice) Sorted() IntSlice {
    result := make(IntSlice, len(s))
    copy(result, s)
    sort.Ints(result)
    return result
}

func main() {
    nums := IntSlice{5, 2, 8, 1, 9, 3, 7, 4, 6}
    
    fmt.Printf("ข้อมูล: %v\n", nums)
    fmt.Printf("ผลรวม: %d\n", nums.Sum())
    fmt.Printf("เฉลี่ย: %.2f\n", nums.Average())
    fmt.Printf("มากสุด: %d\n", nums.Max())
    fmt.Printf("น้อยสุด: %d\n", nums.Min())
    fmt.Printf("เรียงแล้ว: %v\n", nums.Sorted())
}
```

### Methods บน Map Type

```go
package main

import (
    "fmt"
    "sort"
)

type WordCount map[string]int

func (wc WordCount) Add(word string) {
    wc[word]++
}

func (wc WordCount) Total() int {
    total := 0
    for _, count := range wc {
        total += count
    }
    return total
}

func (wc WordCount) TopN(n int) []string {
    type pair struct {
        word  string
        count int
    }
    
    pairs := make([]pair, 0, len(wc))
    for word, count := range wc {
        pairs = append(pairs, pair{word, count})
    }
    
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].count > pairs[j].count
    })
    
    result := make([]string, 0, n)
    for i, p := range pairs {
        if i >= n {
            break
        }
        result = append(result, fmt.Sprintf("%s(%d)", p.word, p.count))
    }
    return result
}

func main() {
    words := WordCount{}
    
    text := []string{"go", "is", "fast", "go", "is", "great", "go", "golang", "fast"}
    for _, w := range text {
        words.Add(w)
    }
    
    fmt.Printf("จำนวนคำทั้งหมด: %d\n", words.Total())
    fmt.Printf("Top 3 คำ: %v\n", words.TopN(3))
}
```

---

## 11.4 Interface Declaration

**Interface** คือ contract ที่กำหนด behavior ของ type ว่าต้องมี methods อะไรบ้าง

```go
package main

import (
    "fmt"
    "math"
)

// Interface declaration
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Circle implements Shape
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

// Rectangle implements Shape
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Triangle implements Shape
type Triangle struct {
    A, B, C float64 // sides
}

func (t Triangle) Area() float64 {
    s := (t.A + t.B + t.C) / 2
    return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
}

func (t Triangle) Perimeter() float64 {
    return t.A + t.B + t.C
}

// ฟังก์ชันที่รับ interface
func printShapeInfo(s Shape) {
    fmt.Printf("พื้นที่: %.2f, เส้นรอบรูป: %.2f\n", s.Area(), s.Perimeter())
}

func totalArea(shapes []Shape) float64 {
    total := 0.0
    for _, s := range shapes {
        total += s.Area()
    }
    return total
}

func main() {
    shapes := []Shape{
        Circle{Radius: 5},
        Rectangle{Width: 4, Height: 6},
        Triangle{A: 3, B: 4, C: 5},
    }
    
    for _, shape := range shapes {
        fmt.Printf("%T: ", shape)
        printShapeInfo(shape)
    }
    
    fmt.Printf("\nพื้นที่รวม: %.2f\n", totalArea(shapes))
}
```

---

## 11.5 Implementing Interfaces

ใน Go การ implement interface ทำโดย **implicit** - ไม่ต้องประกาศชัดเจน

```go
package main

import "fmt"

type Animal interface {
    Name() string
    Sound() string
    Legs() int
}

type Dog struct {
    name string
}

func (d Dog) Name() string  { return d.name }
func (d Dog) Sound() string { return "โห่ง" }
func (d Dog) Legs() int     { return 4 }

type Bird struct {
    name string
}

func (b Bird) Name() string  { return b.name }
func (b Bird) Sound() string { return "จิ๊บ" }
func (b Bird) Legs() int     { return 2 }

type Snake struct {
    name string
}

func (s Snake) Name() string  { return s.name }
func (s Snake) Sound() string { return "ฟ่อ" }
func (s Snake) Legs() int     { return 0 }

func describe(a Animal) {
    fmt.Printf("%s: ร้อง '%s', มี %d ขา\n",
        a.Name(), a.Sound(), a.Legs())
}

func main() {
    animals := []Animal{
        Dog{name: "บักหมา"},
        Bird{name: "นกแก้ว"},
        Snake{name: "งูเขียว"},
    }
    
    for _, a := range animals {
        describe(a)
    }
}
```

---

## 11.6 Interface Composition

Interface สามารถประกอบจาก interface อื่นๆ ได้

```go
package main

import "fmt"

// Basic interfaces
type Reader interface {
    Read() string
}

type Writer interface {
    Write(data string)
}

type Closer interface {
    Close()
}

// Composed interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// ตัวอย่าง: File เป็น ReadWriteCloser
type File struct {
    name    string
    content string
    closed  bool
}

func (f *File) Read() string {
    if f.closed {
        return ""
    }
    return f.content
}

func (f *File) Write(data string) {
    if f.closed {
        return
    }
    f.content += data
}

func (f *File) Close() {
    f.closed = true
    fmt.Printf("File %s closed\n", f.name)
}

func readAndPrint(r Reader) {
    fmt.Println("Content:", r.Read())
}

func copyData(r Reader, w Writer) {
    data := r.Read()
    w.Write(data)
}

func main() {
    file := &File{name: "test.txt"}
    
    // ทำงานกับ interface ReadWriter
    var rw ReadWriter = file
    rw.Write("Hello, ")
    rw.Write("World!")
    
    // ทำงานกับ interface Reader
    readAndPrint(rw)
    
    // Close file
    var rwc ReadWriteCloser = file
    rwc.Close()
    
    fmt.Println("After close:", rwc.Read()) // empty
}
```

---

## 11.7 Empty Interface (interface{} / any)

Empty interface ไม่มี methods จึง implement ได้โดยทุก type

```go
package main

import "fmt"

// any เป็น alias ของ interface{} (Go 1.18+)
func printAnything(v any) {
    fmt.Printf("Type: %T, Value: %v\n", v, v)
}

func main() {
    printAnything(42)
    printAnything("สวัสดี")
    printAnything(3.14)
    printAnything(true)
    printAnything([]int{1, 2, 3})
    printAnything(map[string]int{"a": 1})
    printAnything(nil)
}
```

### ใช้กับ collections

```go
package main

import "fmt"

type Registry struct {
    items map[string]any
}

func NewRegistry() *Registry {
    return &Registry{items: make(map[string]any)}
}

func (r *Registry) Set(key string, value any) {
    r.items[key] = value
}

func (r *Registry) Get(key string) (any, bool) {
    v, ok := r.items[key]
    return v, ok
}

func main() {
    reg := NewRegistry()
    reg.Set("name", "สมชาย")
    reg.Set("age", 25)
    reg.Set("active", true)
    reg.Set("scores", []int{90, 85, 92})
    
    keys := []string{"name", "age", "active", "scores", "missing"}
    for _, k := range keys {
        if v, ok := reg.Get(k); ok {
            fmt.Printf("%s: %v (%T)\n", k, v, v)
        } else {
            fmt.Printf("%s: not found\n", k)
        }
    }
}
```

---

## 11.8 Type Assertions

Type assertion ใช้ดึง concrete type ออกจาก interface

```go
package main

import "fmt"

func processValue(v any) {
    // รูปแบบที่ 1: assert โดยตรง (อาจ panic ถ้าผิด type)
    // n := v.(int) // อันตราย!
    
    // รูปแบบที่ 2: safe assertion ด้วย comma-ok pattern
    if n, ok := v.(int); ok {
        fmt.Printf("int: %d\n", n)
        return
    }
    if s, ok := v.(string); ok {
        fmt.Printf("string: %q\n", s)
        return
    }
    if f, ok := v.(float64); ok {
        fmt.Printf("float64: %f\n", f)
        return
    }
    fmt.Printf("unknown type: %T\n", v)
}

func main() {
    values := []any{42, "hello", 3.14, true, []int{1, 2}}
    
    for _, v := range values {
        processValue(v)
    }
}
```

---

## 11.9 Type Switches

Type switch ดีกว่าการใช้ type assertions หลายๆ ครั้ง

```go
package main

import (
    "fmt"
    "strings"
)

func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case int64:
        return fmt.Sprintf("int64: %d", v)
    case float64:
        return fmt.Sprintf("float64: %.2f", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case bool:
        if v {
            return "bool: true"
        }
        return "bool: false"
    case []int:
        return fmt.Sprintf("[]int: %v (len=%d)", v, len(v))
    case []string:
        return fmt.Sprintf("[]string: [%s]", strings.Join(v, ", "))
    case map[string]int:
        return fmt.Sprintf("map[string]int with %d entries", len(v))
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

func main() {
    values := []any{
        42,
        int64(100),
        3.14,
        "สวัสดี",
        true,
        false,
        []int{1, 2, 3},
        []string{"a", "b", "c"},
        map[string]int{"x": 1, "y": 2},
        nil,
    }
    
    for _, v := range values {
        fmt.Println(describe(v))
    }
}
```

### JSON Unmarshaling ด้วย Type Switch

```go
package main

import (
    "encoding/json"
    "fmt"
)

func parseJSON(data string) {
    var result interface{}
    err := json.Unmarshal([]byte(data), &result)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    printValue(result, 0)
}

func printValue(v interface{}, indent int) {
    prefix := strings.Repeat("  ", indent)
    
    switch val := v.(type) {
    case map[string]interface{}:
        fmt.Println(prefix + "{")
        for k, v := range val {
            fmt.Printf("%s  %q: ", prefix, k)
            printValue(v, indent+1)
        }
        fmt.Println(prefix + "}")
    case []interface{}:
        fmt.Println(prefix + "[")
        for _, item := range val {
            printValue(item, indent+1)
        }
        fmt.Println(prefix + "]")
    case string:
        fmt.Printf("%q\n", val)
    case float64:
        fmt.Printf("%.2f\n", val)
    case bool:
        fmt.Printf("%v\n", val)
    case nil:
        fmt.Println("null")
    }
}

import "strings"

func main() {
    jsonData := `{
        "name": "สมชาย",
        "age": 25,
        "active": true,
        "scores": [90, 85, 92],
        "address": {
            "city": "กรุงเทพฯ"
        }
    }`
    
    parseJSON(jsonData)
}
```

---

## 11.10 Stringer Interface

`fmt.Stringer` เป็น interface ที่กำหนด `String() string` method

```go
package main

import "fmt"

type Direction int

const (
    North Direction = iota
    South
    East
    West
)

// Implement Stringer interface
func (d Direction) String() string {
    switch d {
    case North:
        return "เหนือ"
    case South:
        return "ใต้"
    case East:
        return "ตะวันออก"
    case West:
        return "ตะวันตก"
    default:
        return "ไม่ทราบทิศ"
    }
}

type Card struct {
    Suit  string
    Value int
}

func (c Card) String() string {
    values := map[int]string{
        1: "A", 11: "J", 12: "Q", 13: "K",
    }
    val, ok := values[c.Value]
    if !ok {
        val = fmt.Sprintf("%d", c.Value)
    }
    return fmt.Sprintf("%s%s", val, c.Suit)
}

type Hand []Card

func (h Hand) String() string {
    cards := make([]string, len(h))
    for i, card := range h {
        cards[i] = card.String()
    }
    return "[" + strings.Join(cards, " ") + "]"
}

import "strings"

func main() {
    directions := []Direction{North, South, East, West}
    for _, d := range directions {
        fmt.Printf("ทิศ: %s\n", d) // เรียก String() อัตโนมัติ
    }
    
    hand := Hand{
        {Suit: "♠", Value: 1},
        {Suit: "♥", Value: 13},
        {Suit: "♦", Value: 7},
        {Suit: "♣", Value: 10},
    }
    
    fmt.Printf("\nไพ่: %s\n", hand)
}
```

---

## 11.11 Common Interfaces

### error Interface

```go
package main

import (
    "errors"
    "fmt"
)

// error interface มีแค่ Error() string
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s - %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return ValidationError{Field: "age", Message: "ต้องไม่ติดลบ"}
    }
    if age > 150 {
        return ValidationError{Field: "age", Message: "ค่าสูงเกินไป"}
    }
    return nil
}

func main() {
    ages := []int{25, -1, 200, 18}
    
    for _, age := range ages {
        err := validateAge(age)
        if err != nil {
            fmt.Printf("อายุ %d: %v\n", age, err)
            
            var ve ValidationError
            if errors.As(err, &ve) {
                fmt.Printf("  Field: %s\n", ve.Field)
            }
        } else {
            fmt.Printf("อายุ %d: OK\n", age)
        }
    }
}
```

### io.Reader Interface

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

// ฟังก์ชันที่รับ io.Reader - ทำงานได้กับ string, file, network, etc.
func readAll(r io.Reader) (string, error) {
    buf := make([]byte, 1024)
    var result strings.Builder
    
    for {
        n, err := r.Read(buf)
        if n > 0 {
            result.Write(buf[:n])
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            return "", err
        }
    }
    
    return result.String(), nil
}

// Custom Reader
type RepeatReader struct {
    data    string
    repeats int
    pos     int
}

func (r *RepeatReader) Read(p []byte) (n int, err error) {
    if r.pos >= len(r.data)*r.repeats {
        return 0, io.EOF
    }
    
    remaining := len(r.data)*r.repeats - r.pos
    toCopy := len(p)
    if toCopy > remaining {
        toCopy = remaining
    }
    
    for i := 0; i < toCopy; i++ {
        p[i] = r.data[(r.pos+i)%len(r.data)]
    }
    
    r.pos += toCopy
    return toCopy, nil
}

func main() {
    // strings.NewReader implements io.Reader
    r1 := strings.NewReader("Hello, World!")
    content, _ := readAll(r1)
    fmt.Printf("From string: %q\n", content)
    
    // Custom reader
    r2 := &RepeatReader{data: "Go!", repeats: 3}
    content2, _ := readAll(r2)
    fmt.Printf("From RepeatReader: %q\n", content2)
}
```

### io.Writer Interface

```go
package main

import (
    "fmt"
    "io"
    "strings"
    "unicode"
)

// UppercaseWriter transforms to uppercase
type UppercaseWriter struct {
    w io.Writer
}

func (uw UppercaseWriter) Write(p []byte) (n int, err error) {
    upper := make([]byte, len(p))
    for i, b := range p {
        upper[i] = byte(unicode.ToUpper(rune(b)))
    }
    return uw.w.Write(upper)
}

// PrefixWriter adds prefix to each line
type PrefixWriter struct {
    w      io.Writer
    prefix string
    newline bool
}

func NewPrefixWriter(w io.Writer, prefix string) *PrefixWriter {
    return &PrefixWriter{w: w, prefix: prefix, newline: true}
}

func (pw *PrefixWriter) Write(p []byte) (n int, err error) {
    for _, b := range p {
        if pw.newline {
            fmt.Fprint(pw.w, pw.prefix)
            pw.newline = false
        }
        if b == '\n' {
            pw.newline = true
        }
        pw.w.Write([]byte{b})
    }
    return len(p), nil
}

func main() {
    // UppercaseWriter
    var buf strings.Builder
    uw := UppercaseWriter{w: &buf}
    fmt.Fprintln(uw, "hello, world!")
    fmt.Printf("Uppercase: %s", buf.String())
    
    // PrefixWriter
    var buf2 strings.Builder
    pw := NewPrefixWriter(&buf2, "[INFO] ")
    fmt.Fprintln(pw, "เริ่มต้นระบบ")
    fmt.Fprintln(pw, "โหลดข้อมูล...")
    fmt.Fprintln(pw, "พร้อมทำงาน")
    fmt.Print(buf2.String())
}
```

---

## 11.12 Interface Best Practices

### Accept Interfaces, Return Concrete Types

```go
package main

import "fmt"

// Interface ที่เล็กและเฉพาะเจาะจง
type Saver interface {
    Save(data string) error
}

type Loader interface {
    Load(key string) (string, error)
}

// รับ interface
func processData(s Saver, data string) error {
    return s.Save(data)
}

// Concrete type ที่ implement interface
type MemoryStore struct {
    data map[string]string
}

func NewMemoryStore() *MemoryStore {
    return &MemoryStore{data: make(map[string]string)}
}

func (m *MemoryStore) Save(data string) error {
    m.data["latest"] = data
    return nil
}

func (m *MemoryStore) Load(key string) (string, error) {
    v, ok := m.data[key]
    if !ok {
        return "", fmt.Errorf("key not found: %s", key)
    }
    return v, nil
}

func main() {
    store := NewMemoryStore()
    
    err := processData(store, "ข้อมูลสำคัญ")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    data, err := store.Load("latest")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Println("Loaded:", data)
}
```

### Interface Segregation

```go
package main

import "fmt"

// ไม่ดี - interface ใหญ่เกินไป
type HugeInterface interface {
    Read() string
    Write(s string)
    Delete(id int)
    List() []string
    Count() int
    Search(query string) []string
    Export() []byte
    Import(data []byte) error
}

// ดีกว่า - แยก interface ย่อย
type Reader interface {
    Read() string
}

type Writer interface {
    Write(s string)
}

type Lister interface {
    List() []string
}

// รวมเฉพาะที่ต้องการ
type ReadWriter interface {
    Reader
    Writer
}

func main() {
    fmt.Println("Interface Segregation Principle")
}
```

---

## Workshop: Shape Calculator

สร้างโปรแกรมคำนวณรูปทรงที่ใช้ interface อย่างสมบูรณ์

```go
package main

import (
    "fmt"
    "math"
    "sort"
    "strings"
)

// ====== Interfaces ======

type Shape interface {
    Area() float64
    Perimeter() float64
    Name() string
    String() string
}

type Resizable interface {
    Scale(factor float64)
}

type Drawable interface {
    Draw() string
}

// ====== Shapes ======

type Circle struct {
    radius float64
    color  string
}

func NewCircle(radius float64, color string) *Circle {
    return &Circle{radius: radius, color: color}
}

func (c *Circle) Area() float64 {
    return math.Pi * c.radius * c.radius
}

func (c *Circle) Perimeter() float64 {
    return 2 * math.Pi * c.radius
}

func (c *Circle) Name() string { return "Circle" }

func (c *Circle) Scale(factor float64) {
    c.radius *= factor
}

func (c *Circle) Draw() string {
    size := int(c.radius)
    if size > 5 {
        size = 5
    }
    if size < 1 {
        size = 1
    }
    var sb strings.Builder
    for i := 0; i < size*2+1; i++ {
        for j := 0; j < size*2+1; j++ {
            dx := float64(i - size)
            dy := float64(j - size)
            dist := math.Sqrt(dx*dx + dy*dy)
            if math.Abs(dist-float64(size)) < 0.8 {
                sb.WriteString("*")
            } else {
                sb.WriteString(" ")
            }
        }
        sb.WriteString("\n")
    }
    return sb.String()
}

func (c Circle) String() string {
    return fmt.Sprintf("Circle(r=%.2f, color=%s)", c.radius, c.color)
}

type Rectangle struct {
    width, height float64
    color         string
}

func NewRectangle(width, height float64, color string) *Rectangle {
    return &Rectangle{width: width, height: height, color: color}
}

func (r *Rectangle) Area() float64 { return r.width * r.height }

func (r *Rectangle) Perimeter() float64 { return 2 * (r.width + r.height) }

func (r *Rectangle) Name() string { return "Rectangle" }

func (r *Rectangle) Scale(factor float64) {
    r.width *= factor
    r.height *= factor
}

func (r *Rectangle) Draw() string {
    w := int(r.width)
    h := int(r.height)
    if w > 20 {
        w = 20
    }
    if h > 10 {
        h = 10
    }
    if w < 2 {
        w = 2
    }
    if h < 2 {
        h = 2
    }
    
    var sb strings.Builder
    for i := 0; i < h; i++ {
        for j := 0; j < w; j++ {
            if i == 0 || i == h-1 || j == 0 || j == w-1 {
                sb.WriteString("*")
            } else {
                sb.WriteString(" ")
            }
        }
        sb.WriteString("\n")
    }
    return sb.String()
}

func (r Rectangle) String() string {
    return fmt.Sprintf("Rectangle(%.2fx%.2f, color=%s)", r.width, r.height, r.color)
}

type Triangle struct {
    base, height float64
    color        string
}

func NewTriangle(base, height float64, color string) *Triangle {
    return &Triangle{base: base, height: height, color: color}
}

func (t *Triangle) Area() float64 { return 0.5 * t.base * t.height }

func (t *Triangle) Perimeter() float64 {
    hyp := math.Sqrt(t.base*t.base/4 + t.height*t.height)
    return t.base + 2*hyp
}

func (t *Triangle) Name() string { return "Triangle" }

func (t *Triangle) Scale(factor float64) {
    t.base *= factor
    t.height *= factor
}

func (t *Triangle) Draw() string {
    h := int(t.height)
    if h > 8 {
        h = 8
    }
    if h < 2 {
        h = 2
    }
    var sb strings.Builder
    for i := 1; i <= h; i++ {
        width := i * 2 - 1
        spaces := h - i
        sb.WriteString(strings.Repeat(" ", spaces))
        sb.WriteString(strings.Repeat("*", width))
        sb.WriteString("\n")
    }
    return sb.String()
}

func (t Triangle) String() string {
    return fmt.Sprintf("Triangle(base=%.2f, height=%.2f, color=%s)", t.base, t.height, t.color)
}

// ====== Calculator ======

type ShapeCalculator struct {
    shapes []Shape
}

func NewShapeCalculator() *ShapeCalculator {
    return &ShapeCalculator{}
}

func (sc *ShapeCalculator) Add(s Shape) {
    sc.shapes = append(sc.shapes, s)
}

func (sc *ShapeCalculator) TotalArea() float64 {
    total := 0.0
    for _, s := range sc.shapes {
        total += s.Area()
    }
    return total
}

func (sc *ShapeCalculator) TotalPerimeter() float64 {
    total := 0.0
    for _, s := range sc.shapes {
        total += s.Perimeter()
    }
    return total
}

func (sc *ShapeCalculator) LargestByArea() Shape {
    if len(sc.shapes) == 0 {
        return nil
    }
    largest := sc.shapes[0]
    for _, s := range sc.shapes[1:] {
        if s.Area() > largest.Area() {
            largest = s
        }
    }
    return largest
}

func (sc *ShapeCalculator) SortByArea() []Shape {
    sorted := make([]Shape, len(sc.shapes))
    copy(sorted, sc.shapes)
    sort.Slice(sorted, func(i, j int) bool {
        return sorted[i].Area() < sorted[j].Area()
    })
    return sorted
}

func (sc *ShapeCalculator) GroupByType() map[string][]Shape {
    groups := make(map[string][]Shape)
    for _, s := range sc.shapes {
        groups[s.Name()] = append(groups[s.Name()], s)
    }
    return groups
}

func (sc *ShapeCalculator) ScaleAll(factor float64) {
    for _, s := range sc.shapes {
        if r, ok := s.(Resizable); ok {
            r.Scale(factor)
        }
    }
}

func (sc *ShapeCalculator) PrintReport() {
    fmt.Println("=== รายงานรูปทรง ===")
    fmt.Printf("จำนวนรูปทรง: %d\n\n", len(sc.shapes))
    
    for i, s := range sc.shapes {
        fmt.Printf("%d. %s\n", i+1, s)
        fmt.Printf("   พื้นที่: %.4f\n", s.Area())
        fmt.Printf("   เส้นรอบรูป: %.4f\n", s.Perimeter())
        
        // ลอง draw ถ้า implement Drawable
        if d, ok := s.(Drawable); ok {
            fmt.Println("   รูปร่าง:")
            for _, line := range strings.Split(d.Draw(), "\n") {
                if line != "" {
                    fmt.Printf("   %s\n", line)
                }
            }
        }
        fmt.Println()
    }
    
    fmt.Printf("พื้นที่รวม: %.4f\n", sc.TotalArea())
    fmt.Printf("เส้นรอบรูปรวม: %.4f\n", sc.TotalPerimeter())
    
    if largest := sc.LargestByArea(); largest != nil {
        fmt.Printf("รูปที่ใหญ่สุด: %s (พื้นที่ %.4f)\n",
            largest.Name(), largest.Area())
    }
}

func main() {
    calc := NewShapeCalculator()
    
    calc.Add(NewCircle(5, "red"))
    calc.Add(NewRectangle(8, 4, "blue"))
    calc.Add(NewTriangle(6, 5, "green"))
    calc.Add(NewCircle(3, "yellow"))
    calc.Add(NewRectangle(10, 3, "purple"))
    
    calc.PrintReport()
    
    fmt.Println("\n=== เรียงตามพื้นที่ (น้อย -> มาก) ===")
    for _, s := range calc.SortByArea() {
        fmt.Printf("  %s: %.4f\n", s, s.Area())
    }
    
    fmt.Println("\n=== จัดกลุ่มตามประเภท ===")
    for typeName, shapes := range calc.GroupByType() {
        fmt.Printf("  %s (%d รูป):\n", typeName, len(shapes))
        for _, s := range shapes {
            fmt.Printf("    - %s (พื้นที่ %.2f)\n", s, s.Area())
        }
    }
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| Value Receiver | Method ที่รับ copy ของ struct |
| Pointer Receiver | Method ที่รับ pointer แก้ไข struct ได้ |
| Interface | Contract กำหนด methods ที่ type ต้องมี |
| Implicit Implementation | ไม่ต้องประกาศ implements ชัดเจน |
| Interface Composition | สร้าง interface ใหม่จาก interface ย่อย |
| Empty Interface | `any`/`interface{}` - รับทุก type |
| Type Assertion | ดึง concrete type จาก interface |
| Type Switch | เลือก action ตาม type |
| Stringer | Interface สำหรับ custom string representation |

## Resources

- [Go Tour: Methods](https://go.dev/tour/methods/1)
- [Go Tour: Interfaces](https://go.dev/tour/methods/9)
- [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces)
- [Go by Example: Interfaces](https://gobyexample.com/interfaces)

---

*Part 11 จบแล้ว! ต่อไป Part 12: Error Handling*
