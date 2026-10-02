# Part 2: Variables, Constants, and Data Types

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ประกาศและใช้งาน variables ด้วยรูปแบบต่างๆ ใน Go
- เข้าใจ zero values และความสำคัญของมัน
- รู้จัก data types พื้นฐานทั้งหมดใน Go
- ทำ type conversion ได้อย่างถูกต้องและปลอดภัย
- ใช้ constants และ iota สำหรับ enumeration
- ใช้ multiple assignments และ blank identifier ได้

---

## 2.1 Variable Declaration (การประกาศตัวแปร)

ใน Go การประกาศตัวแปรมีหลายรูปแบบ แต่ทุกรูปแบบมีความชัดเจนและ explicit เสมอ Go เป็นภาษาที่ **type-safe** อย่างเข้มงวด หมายความว่าเมื่อกำหนด type ให้ตัวแปรแล้ว จะเปลี่ยน type ไม่ได้

### รูปแบบที่ 1: var keyword (แบบ explicit)

```go
package main

import "fmt"

func main() {
    // รูปแบบพื้นฐาน: var ชื่อตัวแปร type
    var name string
    var age int
    var price float64
    var isActive bool

    // ตอนนี้ตัวแปรมีค่าเป็น zero value ของแต่ละ type
    fmt.Println(name)     // "" (empty string)
    fmt.Println(age)      // 0
    fmt.Println(price)    // 0
    fmt.Println(isActive) // false

    // กำหนดค่าให้ตัวแปร
    name = "Somchai"
    age = 25
    price = 199.99
    isActive = true

    fmt.Println(name)     // Somchai
    fmt.Println(age)      // 25
    fmt.Println(price)    // 199.99
    fmt.Println(isActive) // true
}
```

### รูปแบบที่ 2: var พร้อมกำหนดค่าเริ่มต้น

```go
package main

import "fmt"

func main() {
    // ประกาศพร้อมกำหนดค่า - Go จะ infer type อัตโนมัติ
    var name string = "Malee"
    var age int = 30
    var height float64 = 165.5

    // หรือให้ Go infer type เอง
    var city = "Bangkok"      // เป็น string
    var population = 10000000 // เป็น int
    var temp = 35.5           // เป็น float64

    fmt.Printf("Name: %s, Age: %d, Height: %.1f\n", name, age, height)
    fmt.Printf("City: %s, Population: %d, Temp: %.1f°C\n", city, population, temp)
}
```

### รูปแบบที่ 3: var block declaration

```go
package main

import "fmt"

func main() {
    // ประกาศหลายตัวแปรพร้อมกันในบล็อก
    var (
        firstName string = "Somchai"
        lastName  string = "Jaruek"
        age       int    = 28
        salary    float64 = 45000.00
        isManager bool   = false
    )

    fmt.Printf("Employee: %s %s\n", firstName, lastName)
    fmt.Printf("Age: %d, Salary: %.2f\n", age, salary)
    fmt.Printf("Manager: %v\n", isManager)
}
```

---

## 2.2 Short Variable Declaration `:=`

รูปแบบนี้เป็นที่นิยมมากใน Go เพราะสั้นกระชับ ใช้ได้เฉพาะภายใน function เท่านั้น (ไม่สามารถใช้ที่ package level ได้)

```go
package main

import "fmt"

func main() {
    // Short declaration - Go infer type จากค่าที่กำหนด
    name := "Somying"
    age := 22
    gpa := 3.75
    passed := true

    fmt.Printf("%s อายุ %d ปี GPA: %.2f ผ่าน: %v\n", name, age, gpa, passed)

    // ประกาศหลายตัวแปรพร้อมกัน
    x, y := 10, 20
    fmt.Printf("x = %d, y = %d\n", x, y)

    // Swap values
    x, y = y, x
    fmt.Printf("หลัง swap: x = %d, y = %d\n", x, y)

    // ใช้กับ function ที่ return หลายค่า
    quotient, remainder := divmod(17, 5)
    fmt.Printf("17 / 5 = %d เศษ %d\n", quotient, remainder)
}

func divmod(a, b int) (int, int) {
    return a / b, a % b
}
```

### ข้อควรระวัง: `:=` ต้องมีตัวแปรใหม่อย่างน้อย 1 ตัว

```go
package main

import "fmt"

func main() {
    x := 10
    fmt.Println(x)

    // ถูกต้อง: y เป็นตัวแปรใหม่
    x, y := 20, 30
    fmt.Println(x, y)

    // ผิด! ทั้ง x และ y มีอยู่แล้ว
    // x, y := 40, 50  // compile error: no new variables on left side of :=

    // ถูกต้อง: ใช้ = แทน
    x, y = 40, 50
    fmt.Println(x, y)
}
```

---

## 2.3 Zero Values

ใน Go ทุก type มี zero value เป็นค่าเริ่มต้นเมื่อประกาศแต่ไม่กำหนดค่า นี่คือ feature สำคัญที่ช่วยป้องกัน uninitialized variable bugs

```go
package main

import "fmt"

func main() {
    // Zero values ของแต่ละ type
    var b bool      // false
    var i int       // 0
    var i8 int8     // 0
    var i16 int16   // 0
    var i32 int32   // 0
    var i64 int64   // 0
    var u uint      // 0
    var f32 float32 // 0
    var f64 float64 // 0
    var s string    // "" (empty string)
    var p *int      // nil (pointer)

    fmt.Printf("bool:    %v\n", b)
    fmt.Printf("int:     %d\n", i)
    fmt.Printf("int8:    %d\n", i8)
    fmt.Printf("int16:   %d\n", i16)
    fmt.Printf("int32:   %d\n", i32)
    fmt.Printf("int64:   %d\n", i64)
    fmt.Printf("uint:    %d\n", u)
    fmt.Printf("float32: %f\n", f32)
    fmt.Printf("float64: %f\n", f64)
    fmt.Printf("string:  %q\n", s)
    fmt.Printf("*int:    %v\n", p)
}
```

**ผลลัพธ์:**
```
bool:    false
int:     0
int8:    0
int16:   0
int32:   0
int64:   0
uint:    0
float32: 0.000000
float64: 0.000000
string:  ""
*int:    <nil>
```

---

## 2.4 Basic Types: Boolean

```go
package main

import "fmt"

func main() {
    var isOpen bool = true
    var hasError bool = false

    // Logical operations
    fmt.Println(isOpen && hasError)  // false (AND)
    fmt.Println(isOpen || hasError)  // true  (OR)
    fmt.Println(!isOpen)             // false (NOT)
    fmt.Println(!hasError)           // true

    // Boolean ในเงื่อนไข
    age := 20
    isAdult := age >= 18
    fmt.Printf("อายุ %d ปี เป็นผู้ใหญ่: %v\n", age, isAdult)

    // การเปรียบเทียบ
    x, y := 5, 10
    fmt.Printf("%d == %d: %v\n", x, y, x == y)
    fmt.Printf("%d != %d: %v\n", x, y, x != y)
    fmt.Printf("%d < %d: %v\n", x, y, x < y)
    fmt.Printf("%d > %d: %v\n", x, y, x > y)
}
```

---

## 2.5 Integer Types

Go มี integer types หลายขนาด แต่ละขนาดมีช่วงค่าที่กำหนดชัดเจน

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    // Signed integers
    var i8 int8 = 127      // -128 ถึง 127
    var i16 int16 = 32767  // -32768 ถึง 32767
    var i32 int32 = 2147483647
    var i64 int64 = 9223372036854775807

    // Unsigned integers
    var u8 uint8 = 255     // 0 ถึง 255
    var u16 uint16 = 65535
    var u32 uint32 = 4294967295
    var u64 uint64 = 18446744073709551615

    // Platform-dependent
    var i int = 42   // 32 หรือ 64 bit ขึ้นอยู่กับ platform
    var u uint = 42

    fmt.Printf("int8:   %d (max: %d)\n", i8, math.MaxInt8)
    fmt.Printf("int16:  %d (max: %d)\n", i16, math.MaxInt16)
    fmt.Printf("int32:  %d (max: %d)\n", i32, math.MaxInt32)
    fmt.Printf("int64:  %d (max: %d)\n", i64, math.MaxInt64)
    fmt.Printf("uint8:  %d (max: %d)\n", u8, math.MaxUint8)
    fmt.Printf("uint16: %d (max: %d)\n", u16, math.MaxUint16)
    fmt.Printf("uint32: %d (max: %d)\n", u32, math.MaxUint32)
    fmt.Printf("uint64: %d (max: %d)\n", u64, uint64(math.MaxUint64))
    fmt.Printf("int:    %d\n", i)
    fmt.Printf("uint:   %d\n", u)
}
```

### Integer Overflow

```go
package main

import "fmt"

func main() {
    // ระวัง integer overflow!
    var x int8 = 127
    fmt.Println(x)  // 127

    x++ // overflow! ไม่มี panic แต่ wrap around
    fmt.Println(x)  // -128

    // ตัวอย่างการใช้งานจริง
    var count int = 0
    for i := 0; i < 10; i++ {
        count++
    }
    fmt.Printf("นับ: %d\n", count)

    // Hexadecimal, Octal, Binary literals
    decimal := 42
    octal := 0o52   // Go 1.13+
    hex := 0x2A
    binary := 0b101010

    fmt.Printf("Decimal: %d\n", decimal)
    fmt.Printf("Octal:   %d (0o%o)\n", octal, octal)
    fmt.Printf("Hex:     %d (0x%X)\n", hex, hex)
    fmt.Printf("Binary:  %d (0b%b)\n", binary, binary)
}
```

---

## 2.6 Floating Point Types

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    // float32: ~7 decimal digits precision
    var f32 float32 = 3.14159265358979323846
    // float64: ~15-17 decimal digits precision
    var f64 float64 = 3.14159265358979323846

    fmt.Printf("float32: %.20f\n", f32)
    fmt.Printf("float64: %.20f\n", f64)

    // Special values
    posInf := math.Inf(1)
    negInf := math.Inf(-1)
    nan := math.NaN()

    fmt.Printf("+Inf: %f\n", posInf)
    fmt.Printf("-Inf: %f\n", negInf)
    fmt.Printf("NaN:  %f\n", nan)

    // ระวัง floating point comparison!
    a := 0.1 + 0.2
    b := 0.3
    fmt.Printf("0.1 + 0.2 = %.17f\n", a)
    fmt.Printf("0.3       = %.17f\n", b)
    fmt.Printf("0.1 + 0.2 == 0.3: %v\n", a == b) // false!

    // วิธีที่ถูกต้อง: ใช้ epsilon comparison
    epsilon := 1e-9
    fmt.Printf("ต่างกัน < %.0e: %v\n", epsilon, math.Abs(a-b) < epsilon)

    // การคำนวณ
    radius := 5.0
    area := math.Pi * radius * radius
    fmt.Printf("วงกลมรัศมี %.1f มีพื้นที่ %.4f\n", radius, area)
}
```

---

## 2.7 Complex Numbers

```go
package main

import (
    "fmt"
    "math/cmplx"
)

func main() {
    // complex64 ใช้ float32 สำหรับ real และ imaginary parts
    var c64 complex64 = 3 + 4i

    // complex128 ใช้ float64
    var c128 complex128 = 3 + 4i

    fmt.Printf("complex64:  %v\n", c64)
    fmt.Printf("complex128: %v\n", c128)

    // ดึง real และ imaginary parts
    fmt.Printf("Real: %f, Imag: %f\n", real(c128), imag(c128))

    // สร้าง complex ด้วย complex()
    c := complex(3.0, 4.0)
    fmt.Printf("Magnitude: %f\n", cmplx.Abs(c)) // 5.0

    // การคำนวณ
    c1 := 1 + 2i
    c2 := 3 + 4i
    fmt.Printf("(%v) + (%v) = %v\n", c1, c2, c1+c2)
    fmt.Printf("(%v) * (%v) = %v\n", c1, c2, c1*c2)
    fmt.Printf("sqrt(-1) = %v\n", cmplx.Sqrt(-1))
}
```

---

## 2.8 String Type

String ใน Go เป็น immutable sequence of bytes (UTF-8 encoded)

```go
package main

import (
    "fmt"
    "strings"
    "unicode/utf8"
)

func main() {
    // String literal
    greeting := "สวัสดีครับ"
    name := "Golang"

    // String concatenation
    message := greeting + " " + name + "!"
    fmt.Println(message)

    // String length
    s := "Hello, World!"
    fmt.Printf("len(\"%s\") = %d bytes\n", s, len(s))

    // ภาษาไทย - length ไม่ใช่จำนวนตัวอักษร!
    thai := "สวัสดี"
    fmt.Printf("len(\"%s\") = %d bytes (ไม่ใช่ %d ตัวอักษร)\n",
        thai, len(thai), utf8.RuneCountInString(thai))

    // วนซ้ำ string ด้วย range (ได้ rune ไม่ใช่ byte)
    fmt.Println("\nตัวอักษรใน 'Hello':")
    for i, ch := range "Hello" {
        fmt.Printf("  index %d: %c (U+%04X)\n", i, ch, ch)
    }

    // String methods
    s2 := "  Hello, World!  "
    fmt.Printf("Trim:       %q\n", strings.TrimSpace(s2))
    fmt.Printf("Upper:      %s\n", strings.ToUpper(s))
    fmt.Printf("Lower:      %s\n", strings.ToLower(s))
    fmt.Printf("Contains:   %v\n", strings.Contains(s, "World"))
    fmt.Printf("Replace:    %s\n", strings.Replace(s, "World", "Go", 1))
    fmt.Printf("Split:      %v\n", strings.Split(s, ", "))
    fmt.Printf("HasPrefix:  %v\n", strings.HasPrefix(s, "Hello"))
    fmt.Printf("HasSuffix:  %v\n", strings.HasSuffix(s, "!"))

    // Raw string literal (ใช้ backtick)
    raw := `สวัสดี
นี่คือ raw string
ไม่ต้อง escape "\n" หรือ "\t"
`
    fmt.Println(raw)
}
```

---

## 2.9 Byte และ Rune

```go
package main

import "fmt"

func main() {
    // byte เป็น alias ของ uint8
    var b byte = 'A'
    fmt.Printf("byte: %d = %c\n", b, b)

    // rune เป็น alias ของ int32 (Unicode code point)
    var r rune = 'ก'
    fmt.Printf("rune: %d = %c (U+%04X)\n", r, r, r)

    // string to bytes
    s := "Hello"
    bytes := []byte(s)
    fmt.Printf("string to bytes: %v\n", bytes)
    fmt.Printf("bytes to string: %s\n", string(bytes))

    // string to runes
    thai := "สวัสดี"
    runes := []rune(thai)
    fmt.Printf("จำนวน bytes: %d\n", len(thai))
    fmt.Printf("จำนวน runes: %d\n", len(runes))
    fmt.Printf("runes: %v\n", runes)

    // แก้ไข string ต้องแปลงเป็น []byte ก่อน
    bs := []byte("Hello, World!")
    bs[0] = 'h' // แปลงเป็น lowercase
    fmt.Println(string(bs))

    // Byte operations
    var x byte = 65  // 'A'
    var y byte = 32  // space
    fmt.Printf("%c + space = %c\n", x, x+y) // 'a' (lowercase)
}
```

---

## 2.10 Type Conversion

Go ไม่มี implicit type conversion ทุกครั้งต้องทำ explicit conversion

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Numeric conversions
    var i int = 42
    var f float64 = float64(i)
    var u uint = uint(f)

    fmt.Printf("int: %d, float64: %f, uint: %d\n", i, f, u)

    // ระวัง: อาจสูญเสีย precision
    var big float64 = 3.99
    var small int = int(big) // truncate ไม่ใช่ round
    fmt.Printf("float64 %.2f -> int %d (ตัดทิ้ง ไม่ใช่ปัดเศษ)\n", big, small)

    // int32 (rune) และ string
    var r rune = 'A'
    s := string(r)
    fmt.Printf("rune %d = string %q\n", r, s)

    // String <-> int conversion ต้องใช้ strconv
    numStr := "42"
    num, err := strconv.Atoi(numStr)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("string %q -> int %d\n", numStr, num)
    }

    // int -> string
    n := 123
    str := strconv.Itoa(n)
    fmt.Printf("int %d -> string %q\n", n, str)

    // ParseFloat
    floatStr := "3.14"
    fVal, err := strconv.ParseFloat(floatStr, 64)
    if err == nil {
        fmt.Printf("string %q -> float64 %f\n", floatStr, fVal)
    }

    // FormatFloat
    fStr := strconv.FormatFloat(3.14159, 'f', 2, 64)
    fmt.Printf("float64 3.14159 -> string %q\n", fStr)

    // ParseBool
    boolStr := "true"
    bVal, _ := strconv.ParseBool(boolStr)
    fmt.Printf("string %q -> bool %v\n", boolStr, bVal)

    // ตัวอย่างการแปลง byte/string
    bytes := []byte{72, 101, 108, 108, 111}
    fmt.Printf("bytes %v -> string %q\n", bytes, string(bytes))

    str2 := "Hello"
    fmt.Printf("string %q -> bytes %v\n", str2, []byte(str2))
}
```

---

## 2.11 Constants

Constants ใน Go เป็นค่าที่ไม่เปลี่ยนแปลงตลอด runtime และต้องกำหนดค่าตอน compile time

```go
package main

import (
    "fmt"
    "math"
)

// Package-level constants
const Pi = 3.14159265358979323846
const AppName = "MyGoApp"
const Version = "1.0.0"
const MaxRetries = 3

// Typed constant
const MaxAge int = 150

// Block constant declaration
const (
    StatusOK        = 200
    StatusNotFound  = 404
    StatusError     = 500
    StatusCreated   = 201
)

func main() {
    fmt.Printf("Pi = %v\n", Pi)
    fmt.Printf("App: %s v%s\n", AppName, Version)
    fmt.Printf("Max Retries: %d\n", MaxRetries)
    fmt.Printf("HTTP Status: OK=%d, NotFound=%d, Error=%d\n",
        StatusOK, StatusNotFound, StatusError)

    // Untyped constants มีความยืดหยุ่นสูง
    const bigNum = 1 << 62
    fmt.Printf("bigNum = %d\n", bigNum)

    // ใช้กับ math
    const radius = 5
    area := math.Pi * radius * radius
    fmt.Printf("Circle area = %.4f\n", area)

    // String constants
    const greeting = "สวัสดีครับ"
    const prefix = "คุณ"
    // ไม่สามารถ reassign ได้
    // greeting = "hello"  // compile error!
    fmt.Println(prefix + "สมชาย: " + greeting)
}
```

---

## 2.12 iota - Enumeration

`iota` เป็น special constant ที่ใช้สำหรับสร้าง enumeration values

```go
package main

import "fmt"

// Weekday enumeration
type Weekday int

const (
    Sunday Weekday = iota // 0
    Monday                // 1
    Tuesday               // 2
    Wednesday             // 3
    Thursday              // 4
    Friday                // 5
    Saturday              // 6
)

func (d Weekday) String() string {
    names := [...]string{
        "Sunday", "Monday", "Tuesday", "Wednesday",
        "Thursday", "Friday", "Saturday",
    }
    if d < Sunday || d > Saturday {
        return fmt.Sprintf("Unknown(%d)", d)
    }
    return names[d]
}

// File permissions (bitwise iota)
type Permission uint

const (
    Read Permission = 1 << iota // 1 (001)
    Write                       // 2 (010)
    Execute                     // 4 (100)
)

// Byte size constants
const (
    _           = iota // ข้ามค่าแรก (0)
    KB = 1 << (10 * iota) // 1 << 10 = 1024
    MB                    // 1 << 20
    GB                    // 1 << 30
    TB                    // 1 << 40
)

// Status codes
type Status int

const (
    StatusPending Status = iota + 1 // เริ่มจาก 1
    StatusActive
    StatusInactive
    StatusDeleted
)

func main() {
    // Weekday
    day := Wednesday
    fmt.Printf("Day: %s (%d)\n", day, day)

    for d := Sunday; d <= Saturday; d++ {
        fmt.Printf("  %d = %s\n", d, d)
    }

    // Permissions
    perm := Read | Write
    fmt.Printf("\nPermission: %d (binary: %03b)\n", perm, perm)
    fmt.Printf("Has Read:    %v\n", perm&Read != 0)
    fmt.Printf("Has Write:   %v\n", perm&Write != 0)
    fmt.Printf("Has Execute: %v\n", perm&Execute != 0)

    // Byte sizes
    fmt.Printf("\n1 KB = %d bytes\n", KB)
    fmt.Printf("1 MB = %d bytes\n", MB)
    fmt.Printf("1 GB = %d bytes\n", GB)
    fmt.Printf("1 TB = %d bytes\n", TB)

    // Status
    fmt.Printf("\nStatus values: Pending=%d, Active=%d, Inactive=%d, Deleted=%d\n",
        StatusPending, StatusActive, StatusInactive, StatusDeleted)
}
```

---

## 2.13 Multiple Assignments

```go
package main

import "fmt"

func getCoordinates() (float64, float64) {
    return 13.7563, 100.5018 // Bangkok coordinates
}

func getNameAge() (string, int) {
    return "Somchai", 25
}

func main() {
    // Multiple assignment พร้อมกัน
    x, y := 10, 20
    fmt.Printf("x=%d, y=%d\n", x, y)

    // Swap without temp variable
    x, y = y, x
    fmt.Printf("After swap: x=%d, y=%d\n", x, y)

    // รับค่าจาก function หลายค่า
    lat, lng := getCoordinates()
    fmt.Printf("Bangkok: %.4f, %.4f\n", lat, lng)

    name, age := getNameAge()
    fmt.Printf("%s อายุ %d ปี\n", name, age)

    // กำหนดค่าหลายตัวแปรพร้อมกัน
    a, b, c := 1, 2, 3
    fmt.Printf("a=%d, b=%d, c=%d\n", a, b, c)

    // Mixed: บางตัวใหม่ บางตัวมีอยู่แล้ว
    d, e := 4, 5
    d, f := 40, 60 // d มีอยู่แล้ว แต่ f ใหม่
    fmt.Printf("d=%d, e=%d, f=%d\n", d, e, f)
}
```

---

## 2.14 Blank Identifier `_`

`_` ใช้สำหรับทิ้งค่าที่ไม่ต้องการใช้

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func getStats() (min, max, avg float64) {
    return 10.5, 99.9, 55.2
}

func main() {
    // ทิ้งค่าที่ไม่ต้องการจาก multiple return
    _, max, _ := getStats()
    fmt.Printf("Max: %.1f\n", max)

    // ทิ้ง error (ไม่แนะนำในโค้ด production!)
    num, _ := strconv.Atoi("42")
    fmt.Printf("num: %d\n", num)

    // ใช้ใน for range เมื่อไม่ต้องการ index
    fruits := []string{"apple", "banana", "cherry"}
    for _, fruit := range fruits {
        fmt.Println(fruit)
    }

    // ใช้ใน for range เมื่อไม่ต้องการ value
    for i := range fruits {
        fmt.Printf("index: %d\n", i)
    }

    // Import package เพื่อ side effect เท่านั้น
    // import _ "image/png" // register PNG decoder

    // Suppress unused import (จริงๆ ควรลบออก แต่บางครั้งต้องการ side effects)
    _ = os.Stdout // บอก Go ว่า os ถูกใช้งาน

    // ใช้ตรวจสอบว่า interface implemented
    // var _ MyInterface = (*MyStruct)(nil)
}
```

---

## 2.15 ตัวอย่างรวม: โปรแกรมคำนวณค่าสถิติ

```go
package main

import (
    "fmt"
    "math"
    "strconv"
    "strings"
)

func main() {
    // ข้อมูลนักเรียน
    const className = "โปรแกรมมิ่ง 101"
    const totalStudents = 5

    // คะแนนนักเรียน
    scores := [totalStudents]float64{85.5, 92.0, 78.3, 95.5, 88.0}
    names := [totalStudents]string{"สมชาย", "สมหญิง", "สมศักดิ์", "สมใจ", "สมพร"}

    fmt.Printf("=== ผลการเรียน %s ===\n\n", className)

    // คำนวณสถิติ
    var sum float64
    min := scores[0]
    max := scores[0]

    for i, score := range scores {
        sum += score
        if score < min {
            min = score
        }
        if score > max {
            max = score
        }
        fmt.Printf("%-10s: %.1f\n", names[i], score)
    }

    avg := sum / float64(totalStudents)

    // Standard deviation
    var sumSqDiff float64
    for _, score := range scores {
        diff := score - avg
        sumSqDiff += diff * diff
    }
    stdDev := math.Sqrt(sumSqDiff / float64(totalStudents))

    fmt.Println(strings.Repeat("-", 25))
    fmt.Printf("%-10s: %.2f\n", "เฉลี่ย", avg)
    fmt.Printf("%-10s: %.1f\n", "ต่ำสุด", min)
    fmt.Printf("%-10s: %.1f\n", "สูงสุด", max)
    fmt.Printf("%-10s: %.2f\n", "Std Dev", stdDev)

    // แปลงคะแนนเฉลี่ยเป็น string
    avgStr := strconv.FormatFloat(avg, 'f', 2, 64)
    fmt.Printf("\nคะแนนเฉลี่ยเป็น string: %q\n", avgStr)

    // Grade calculation
    fmt.Println("\nเกรด:")
    for i, score := range scores {
        var grade string
        switch {
        case score >= 90:
            grade = "A"
        case score >= 80:
            grade = "B"
        case score >= 70:
            grade = "C"
        case score >= 60:
            grade = "D"
        default:
            grade = "F"
        }
        fmt.Printf("  %-10s: %.1f -> %s\n", names[i], score, grade)
    }
}
```

---

## 2.16 ตัวอย่าง: ระบบสมาชิก (Member System)

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

type MemberLevel int

const (
    Bronze MemberLevel = iota + 1
    Silver
    Gold
    Platinum
)

func (m MemberLevel) String() string {
    switch m {
    case Bronze:
        return "Bronze"
    case Silver:
        return "Silver"
    case Gold:
        return "Gold"
    case Platinum:
        return "Platinum"
    default:
        return "Unknown"
    }
}

func (m MemberLevel) Discount() float64 {
    switch m {
    case Bronze:
        return 0.05
    case Silver:
        return 0.10
    case Gold:
        return 0.15
    case Platinum:
        return 0.20
    default:
        return 0
    }
}

func main() {
    // ข้อมูลสมาชิก
    memberName := "สมชาย จารึก"
    memberID := "MBR-2024-001"
    level := Gold
    joinYear := 2020
    currentYear := time.Now().Year()
    yearsOfMembership := currentYear - joinYear

    // ราคาสินค้า
    originalPrice := 1500.00

    // คำนวณส่วนลด
    discount := level.Discount()
    discountAmount := originalPrice * discount
    finalPrice := originalPrice - discountAmount

    fmt.Println(strings.Repeat("=", 40))
    fmt.Println("         ข้อมูลสมาชิก")
    fmt.Println(strings.Repeat("=", 40))
    fmt.Printf("ชื่อ:          %s\n", memberName)
    fmt.Printf("รหัสสมาชิก:    %s\n", memberID)
    fmt.Printf("ระดับ:         %s\n", level)
    fmt.Printf("สมาชิกมา:      %d ปี\n", yearsOfMembership)
    fmt.Println(strings.Repeat("-", 40))
    fmt.Printf("ราคาเต็ม:      ฿%,.2f\n", originalPrice)
    fmt.Printf("ส่วนลด %d%%:   -฿%.2f\n", int(discount*100), discountAmount)
    fmt.Printf("ราคาสุทธิ:     ฿%.2f\n", finalPrice)
    fmt.Println(strings.Repeat("=", 40))
}
```

---

## Workshop: แบบฝึกหัด Part 2

### แบบฝึกหัดที่ 1: ตัวแปรและ Types

สร้างโปรแกรมเก็บข้อมูลนักเรียน:
- ชื่อ, นามสกุล (string)
- อายุ (int)
- GPA (float64)
- ผ่าน/ไม่ผ่าน (bool)
- รหัสนักเรียน (uint32)

แสดงผลข้อมูลทั้งหมดและบอก type ของแต่ละตัวแปรด้วย `%T`

### แบบฝึกหัดที่ 2: Type Conversion

เขียนโปรแกรมที่รับค่า temperature เป็น Celsius แล้วแปลงเป็น Fahrenheit และ Kelvin:
- Fahrenheit = (Celsius × 9/5) + 32
- Kelvin = Celsius + 273.15
- แสดงผลทั้งสามค่า โดยใช้ type ที่เหมาะสม

### แบบฝึกหัดที่ 3: Constants และ iota

สร้าง enumeration สำหรับ:
1. วันในสัปดาห์ (พร้อม method String())
2. ขนาดเสื้อผ้า: XS, S, M, L, XL, XXL (พร้อม method ToCM() ที่ return ขนาดหน้าอก)
3. Directions: North, South, East, West

### แบบฝึกหัดที่ 4: String Manipulation

รับ full name ในรูปแบบ "ชื่อ นามสกุล" แล้ว:
1. แยกชื่อและนามสกุล
2. นับจำนวนตัวอักษร
3. แปลงเป็น UPPER CASE
4. สร้าง username จาก first char ของชื่อ + นามสกุล lowercase

### เฉลยแบบฝึกหัดที่ 2

```go
package main

import "fmt"

func main() {
    // Input
    celsius := 100.0 // น้ำเดือด

    // Conversion
    fahrenheit := (celsius * 9.0 / 5.0) + 32.0
    kelvin := celsius + 273.15

    fmt.Printf("อุณหภูมิ:\n")
    fmt.Printf("  Celsius:    %.2f°C\n", celsius)
    fmt.Printf("  Fahrenheit: %.2f°F\n", fahrenheit)
    fmt.Printf("  Kelvin:     %.2f K\n", kelvin)
}
```

---

## สรุป Part 2

ใน Part นี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Variable Declaration | `var`, `:=`, var block |
| Zero Values | ทุก type มีค่าเริ่มต้น ไม่มี undefined |
| Integer Types | int8/16/32/64, uint, ระวัง overflow |
| Float Types | float32/64, ระวัง precision |
| String | UTF-8, immutable, `len()` คืน bytes |
| Byte/Rune | byte=uint8, rune=int32 (Unicode) |
| Type Conversion | Explicit เสมอ ใช้ strconv สำหรับ string |
| Constants | `const`, ไม่เปลี่ยนแปลง, compile-time |
| iota | Enumeration, bitwise flags |
| Blank Identifier | `_` ทิ้งค่าที่ไม่ใช้ |

**Key Takeaways:**
- Go เป็น statically typed language - type ถูกกำหนดตอน compile
- ไม่มี implicit conversion ต้องทำ explicit เสมอ
- Zero values ช่วยป้องกัน uninitialized bugs
- ใช้ `:=` สำหรับ local variables, `var` สำหรับ package-level หรือต้องการ explicit type

## Resources

- [Go Specification - Types](https://go.dev/ref/spec#Types)
- [Go Tour - Basics](https://go.dev/tour/basics)
- [Effective Go - Variables](https://go.dev/doc/effective_go#variables)
- [Go by Example - Variables](https://gobyexample.com/variables)
- [Go by Example - Constants](https://gobyexample.com/constants)
