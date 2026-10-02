# Part 3: Operators and Expressions

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ใช้ arithmetic operators ทุกตัวได้อย่างถูกต้อง
- เปรียบเทียบค่าด้วย comparison operators
- ใช้ logical operators สร้าง complex conditions
- ทำ bitwise operations สำหรับการจัดการ bits
- ใช้ assignment operators แบบย่อ
- เข้าใจ operator precedence และใช้ parentheses อย่างเหมาะสม

---

## 3.1 Arithmetic Operators (ตัวดำเนินการคณิตศาสตร์)

```go
package main

import "fmt"

func main() {
    a, b := 15, 4

    fmt.Println("=== Arithmetic Operators ===")
    fmt.Printf("%d + %d = %d\n", a, b, a+b)   // 19 (บวก)
    fmt.Printf("%d - %d = %d\n", a, b, a-b)   // 11 (ลบ)
    fmt.Printf("%d * %d = %d\n", a, b, a*b)   // 60 (คูณ)
    fmt.Printf("%d / %d = %d\n", a, b, a/b)   // 3  (หาร - integer division)
    fmt.Printf("%d %% %d = %d\n", a, b, a%b)  // 3  (modulo/เศษ)

    // Integer division truncates (ตัดทิ้ง ไม่ปัดเศษ)
    fmt.Printf("\n%d / %d = %d (ไม่ใช่ %.2f)\n", 7, 2, 7/2, float64(7)/float64(2))

    // Floating point division
    x, y := 15.0, 4.0
    fmt.Printf("\n%.1f / %.1f = %.4f\n", x, y, x/y) // 3.75

    // Negative modulo
    fmt.Printf("\n%d %% %d = %d\n", -7, 3, -7%3)  // -1
    fmt.Printf("%d %% %d = %d\n", 7, -3, 7%-3)    // 1

    // Overflow example
    var n int8 = 100
    fmt.Printf("\nint8 %d * 2 = %d (overflow!)\n", n, n*2)
}
```

### ตัวอย่างการคำนวณจริง: แบ่งของ

```go
package main

import "fmt"

func main() {
    // แบ่งของให้คนเท่าๆ กัน
    items := 100
    people := 7

    perPerson := items / people
    remainder := items % people

    fmt.Printf("ของ %d ชิ้น แบ่งให้ %d คน\n", items, people)
    fmt.Printf("แต่ละคนได้ %d ชิ้น เหลือ %d ชิ้น\n", perPerson, remainder)

    // ตรวจสอบว่าหารลงตัวหรือไม่
    if items%people == 0 {
        fmt.Println("แบ่งได้พอดี!")
    } else {
        fmt.Printf("เหลือ %d ชิ้น\n", items%people)
    }

    // คำนวณราคา
    price := 299.0
    quantity := 3
    discount := 0.1 // 10%

    subtotal := price * float64(quantity)
    discountAmt := subtotal * discount
    total := subtotal - discountAmt

    fmt.Printf("\nราคา: %.2f x %d = %.2f\n", price, quantity, subtotal)
    fmt.Printf("ส่วนลด 10%%: -%.2f\n", discountAmt)
    fmt.Printf("รวม: %.2f\n", total)
}
```

---

## 3.2 Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

```go
package main

import "fmt"

func main() {
    a, b := 10, 20

    fmt.Println("=== Comparison Operators ===")
    fmt.Printf("%d == %d : %v\n", a, b, a == b)  // false
    fmt.Printf("%d != %d : %v\n", a, b, a != b)  // true
    fmt.Printf("%d < %d  : %v\n", a, b, a < b)   // true
    fmt.Printf("%d > %d  : %v\n", a, b, a > b)   // false
    fmt.Printf("%d <= %d : %v\n", a, b, a <= b)  // true
    fmt.Printf("%d >= %d : %v\n", a, b, a >= b)  // false

    // String comparison (เปรียบเทียบ lexicographically)
    s1, s2 := "apple", "banana"
    fmt.Printf("\n%q == %q : %v\n", s1, s2, s1 == s2)
    fmt.Printf("%q < %q  : %v\n", s1, s2, s1 < s2)  // a < b

    // Bool comparison
    fmt.Printf("\ntrue == false : %v\n", true == false)
    fmt.Printf("true != false : %v\n", true != false)

    // Comparison ใน conditions
    score := 85
    var grade string
    if score >= 90 {
        grade = "A"
    } else if score >= 80 {
        grade = "B"
    } else if score >= 70 {
        grade = "C"
    } else {
        grade = "F"
    }
    fmt.Printf("\nคะแนน %d = เกรด %s\n", score, grade)

    // ระวัง: == กับ float ไม่แม่นยำ
    f1 := 0.1 + 0.2
    f2 := 0.3
    fmt.Printf("\n0.1 + 0.2 == 0.3: %v (ไม่ควรใช้ == กับ float)\n", f1 == f2)
}
```

---

## 3.3 Logical Operators (ตัวดำเนินการทางตรรกะ)

```go
package main

import "fmt"

func main() {
    t, f := true, false

    fmt.Println("=== Logical Operators ===")
    fmt.Printf("true  && true  = %v\n", t && t)   // true
    fmt.Printf("true  && false = %v\n", t && f)   // false
    fmt.Printf("false && true  = %v\n", f && t)   // false
    fmt.Printf("false && false = %v\n", f && f)   // false

    fmt.Printf("\ntrue  || true  = %v\n", t || t)  // true
    fmt.Printf("true  || false = %v\n", t || f)  // true
    fmt.Printf("false || true  = %v\n", f || t)  // true
    fmt.Printf("false || false = %v\n", f || f)  // false

    fmt.Printf("\n!true  = %v\n", !t)   // false
    fmt.Printf("!false = %v\n", !f)    // true

    // Short-circuit evaluation
    // && : ถ้าด้านซ้ายเป็น false จะไม่ evaluate ด้านขวา
    // || : ถ้าด้านซ้ายเป็น true จะไม่ evaluate ด้านขวา

    x := 0
    // ถ้า x == 0 จะไม่ evaluate 10/x
    if x != 0 && 10/x > 1 {
        fmt.Println("ผ่าน")
    } else {
        fmt.Println("x เป็น 0 ป้องกัน division by zero")
    }

    // ตัวอย่างจริง: ตรวจสอบ user login
    isLoggedIn := true
    isAdmin := false
    hasPermission := true

    canEdit := isLoggedIn && (isAdmin || hasPermission)
    canDelete := isLoggedIn && isAdmin
    canView := isLoggedIn || hasPermission

    fmt.Printf("\nสามารถแก้ไข: %v\n", canEdit)
    fmt.Printf("สามารถลบ:   %v\n", canDelete)
    fmt.Printf("สามารถดู:   %v\n", canView)

    // De Morgan's Law
    a, b2 := true, false
    fmt.Printf("\n!(a && b) == (!a || !b): %v\n", !(a && b2) == (!a || !b2))
    fmt.Printf("!(a || b) == (!a && !b): %v\n", !(a || b2) == (!a && !b2))
}
```

### ตัวอย่าง Short-circuit Evaluation

```go
package main

import "fmt"

var callCount int

func expensiveCheck() bool {
    callCount++
    fmt.Printf("  [expensiveCheck ถูกเรียก %d ครั้ง]\n", callCount)
    return true
}

func cheapCheck() bool {
    return false
}

func main() {
    fmt.Println("=== Short-circuit Evaluation ===")

    // && : cheapCheck() เป็น false -> expensiveCheck() จะไม่ถูกเรียก
    callCount = 0
    result := cheapCheck() && expensiveCheck()
    fmt.Printf("false && expensive = %v (expensive ไม่ถูกเรียก)\n", result)

    // || : cheapCheck() เป็น false -> expensiveCheck() จะถูกเรียก
    callCount = 0
    result = cheapCheck() || expensiveCheck()
    fmt.Printf("false || expensive = %v (expensive ถูกเรียก)\n", result)

    // ประโยชน์: ป้องกัน nil pointer dereference
    var slice []int
    if slice != nil && len(slice) > 0 {
        fmt.Println(slice[0])
    } else {
        fmt.Println("slice ว่างหรือ nil")
    }
}
```

---

## 3.4 Bitwise Operators (ตัวดำเนินการ Bitwise)

```go
package main

import "fmt"

func main() {
    a, b := uint8(0b10110100), uint8(0b11001010) // 180, 202

    fmt.Printf("a  = %08b (%d)\n", a, a)
    fmt.Printf("b  = %08b (%d)\n", b, b)
    fmt.Printf("\n")

    fmt.Printf("a & b  = %08b (%d)  AND\n", a&b, a&b)
    fmt.Printf("a | b  = %08b (%d)  OR\n", a|b, a|b)
    fmt.Printf("a ^ b  = %08b (%d)  XOR\n", a^b, a^b)
    fmt.Printf("^a     = %08b (%d)  NOT\n", ^a, ^a)
    fmt.Printf("a &^ b = %08b (%d)  AND NOT\n", a&^b, a&^b)

    fmt.Println("\n=== Shift Operators ===")
    n := uint8(1)
    fmt.Printf("1 << 0 = %d\n", n<<0)
    fmt.Printf("1 << 1 = %d\n", n<<1)
    fmt.Printf("1 << 2 = %d\n", n<<2)
    fmt.Printf("1 << 3 = %d\n", n<<3)
    fmt.Printf("1 << 7 = %d\n", n<<7)

    x := uint8(128)
    fmt.Printf("\n128 >> 0 = %d\n", x>>0)
    fmt.Printf("128 >> 1 = %d\n", x>>1)
    fmt.Printf("128 >> 2 = %d\n", x>>2)
    fmt.Printf("128 >> 7 = %d\n", x>>7)
}
```

### ตัวอย่าง Bitwise: File Permissions

```go
package main

import "fmt"

// Unix-style permissions
const (
    OwnerRead    = 1 << 8 // 0400
    OwnerWrite   = 1 << 7 // 0200
    OwnerExec    = 1 << 6 // 0100
    GroupRead    = 1 << 5 // 040
    GroupWrite   = 1 << 4 // 020
    GroupExec    = 1 << 3 // 010
    OthersRead   = 1 << 2 // 04
    OthersWrite  = 1 << 1 // 02
    OthersExec   = 1 << 0 // 01
)

func permString(perm int) string {
    result := ""
    checks := []struct {
        bit  int
        char string
    }{
        {OwnerRead, "r"}, {OwnerWrite, "w"}, {OwnerExec, "x"},
        {GroupRead, "r"}, {GroupWrite, "w"}, {GroupExec, "x"},
        {OthersRead, "r"}, {OthersWrite, "w"}, {OthersExec, "x"},
    }
    for _, c := range checks {
        if perm&c.bit != 0 {
            result += c.char
        } else {
            result += "-"
        }
    }
    return result
}

func main() {
    // rwxr-xr-x (755)
    perm := OwnerRead | OwnerWrite | OwnerExec |
            GroupRead | GroupExec |
            OthersRead | OthersExec

    fmt.Printf("Octal: %04o\n", perm)
    fmt.Printf("Perms: %s\n", permString(perm))

    // เพิ่ม group write
    perm |= GroupWrite
    fmt.Printf("\nหลังเพิ่ม group write:\n")
    fmt.Printf("Octal: %04o\n", perm)
    fmt.Printf("Perms: %s\n", permString(perm))

    // ลบ others permissions
    perm &^= (OthersRead | OthersWrite | OthersExec)
    fmt.Printf("\nหลังลบ others permissions:\n")
    fmt.Printf("Octal: %04o\n", perm)
    fmt.Printf("Perms: %s\n", permString(perm))

    // ตรวจสอบ bit
    fmt.Printf("\nOwner สามารถอ่าน: %v\n", perm&OwnerRead != 0)
    fmt.Printf("Others สามารถอ่าน: %v\n", perm&OthersRead != 0)
}
```

### Bitwise Tricks

```go
package main

import "fmt"

func main() {
    // เป็นเลขคู่หรือคี่
    for i := 0; i <= 10; i++ {
        if i&1 == 0 {
            fmt.Printf("%d เป็นเลขคู่\n", i)
        }
    }

    // คูณ/หารด้วย 2 ด้วย shift
    n := 16
    fmt.Printf("\n%d * 2 = %d (left shift 1)\n", n, n<<1)
    fmt.Printf("%d / 2 = %d (right shift 1)\n", n, n>>1)
    fmt.Printf("%d * 4 = %d (left shift 2)\n", n, n<<2)
    fmt.Printf("%d / 4 = %d (right shift 2)\n", n, n>>2)

    // Swap without temp (XOR swap)
    a, b := 42, 73
    fmt.Printf("\nก่อน swap: a=%d, b=%d\n", a, b)
    a ^= b
    b ^= a
    a ^= b
    fmt.Printf("หลัง swap:  a=%d, b=%d\n", a, b)

    // Toggle bit
    flags := 0b00000000
    mask := 0b00000100 // bit 2
    fmt.Printf("\nflags: %08b\n", flags)
    flags ^= mask // toggle bit 2
    fmt.Printf("หลัง toggle bit 2: %08b\n", flags)
    flags ^= mask // toggle again
    fmt.Printf("toggle อีกครั้ง: %08b\n", flags)

    // Check if power of 2
    for i := 1; i <= 16; i++ {
        if i&(i-1) == 0 {
            fmt.Printf("%d เป็น power of 2\n", i)
        }
    }
}
```

---

## 3.5 Assignment Operators (ตัวดำเนินการกำหนดค่า)

```go
package main

import "fmt"

func main() {
    x := 10
    fmt.Printf("เริ่มต้น: x = %d\n", x)

    x += 5
    fmt.Printf("x += 5 : x = %d\n", x) // 15

    x -= 3
    fmt.Printf("x -= 3 : x = %d\n", x) // 12

    x *= 2
    fmt.Printf("x *= 2 : x = %d\n", x) // 24

    x /= 4
    fmt.Printf("x /= 4 : x = %d\n", x) // 6

    x %= 4
    fmt.Printf("x %%= 4 : x = %d\n", x) // 2

    // Bitwise assignment
    y := 0b11110000
    y &= 0b10101010
    fmt.Printf("\ny &= 0b10101010 : %08b\n", y)

    y |= 0b00001111
    fmt.Printf("y |= 0b00001111  : %08b\n", y)

    y ^= 0b11111111
    fmt.Printf("y ^= 0b11111111  : %08b\n", y)

    y <<= 2
    fmt.Printf("y <<= 2          : %08b\n", y)

    y >>= 1
    fmt.Printf("y >>= 1          : %08b\n", y)

    // ตัวอย่างการใช้งาน: สะสมคะแนน
    score := 0
    score += 100  // ตอบถูก
    score += 50   // bonus
    score -= 20   // penalty
    score *= 2    // double score event
    fmt.Printf("\nคะแนนสุดท้าย: %d\n", score)
}
```

---

## 3.6 Increment และ Decrement

```go
package main

import "fmt"

func main() {
    // Go มีแค่ ++ และ -- เป็น statement (ไม่ใช่ expression)
    i := 5
    i++
    fmt.Printf("i++ : i = %d\n", i) // 6

    i--
    fmt.Printf("i-- : i = %d\n", i) // 5

    // ไม่สามารถใช้เป็น expression ได้
    // x := i++  // compile error!
    // if i++ > 5 {} // compile error!

    // ไม่มี prefix ++ หรือ --
    // ++i  // compile error!
    // --i  // compile error!

    // การใช้งานใน loop
    count := 0
    for count < 5 {
        fmt.Printf("count = %d\n", count)
        count++
    }

    // Decrement
    n := 3
    for n > 0 {
        fmt.Printf("นับถอยหลัง: %d\n", n)
        n--
    }
    fmt.Println("ออกสตาร์ท!")
}
```

---

## 3.7 Operator Precedence (ลำดับการดำเนินการ)

```go
package main

import "fmt"

func main() {
    // ลำดับ operator precedence (สูง -> ต่ำ):
    // 5: * / % << >> & &^
    // 4: + - | ^
    // 3: == != < <= > >=
    // 2: &&
    // 1: ||

    fmt.Println("=== Operator Precedence ===")

    // คณิตศาสตร์ก่อน
    result := 2 + 3*4
    fmt.Printf("2 + 3 * 4 = %d (ไม่ใช่ %d)\n", result, (2+3)*4)

    // ใช้ () เพื่อความชัดเจน
    result2 := (2 + 3) * 4
    fmt.Printf("(2 + 3) * 4 = %d\n", result2)

    // Comparison ก่อน Logical
    a, b, c := 5, 10, 15
    check := a < b && b < c
    fmt.Printf("%d < %d && %d < %d = %v\n", a, b, b, c, check)

    // Bitwise มีลำดับสูงกว่า +
    n := 2 + 3<<1  // 2 + (3 << 1) = 2 + 6 = 8
    fmt.Printf("2 + 3 << 1 = %d (= 2 + 6)\n", n)

    // ตัวอย่างที่ซับซ้อน
    x := 5
    y := 3
    z := 2
    expr := x*y + z*y - x/z
    fmt.Printf("\n%d*%d + %d*%d - %d/%d = %d\n", x, y, z, y, x, z, expr)

    // Logical precedence
    p, q, r := true, false, true
    result3 := p || q && r  // p || (q && r) = true || false = true
    result4 := (p || q) && r // true && true = true
    fmt.Printf("\np || q && r     = %v\n", result3)
    fmt.Printf("(p || q) && r   = %v\n", result4)

    // คำแนะนำ: ใช้ () เสมอเพื่อความชัดเจน
    clear := (a < b) && (b < c) && (c > 0)
    fmt.Printf("\n(a<b) && (b<c) && (c>0) = %v\n", clear)
}
```

---

## 3.8 ตัวอย่างรวม: Calculator

```go
package main

import (
    "fmt"
    "math"
)

func add(a, b float64) float64      { return a + b }
func subtract(a, b float64) float64 { return a - b }
func multiply(a, b float64) float64 { return a * b }
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("ไม่สามารถหารด้วยศูนย์ได้")
    }
    return a / b, nil
}
func power(base, exp float64) float64 { return math.Pow(base, exp) }
func sqrt(n float64) (float64, error) {
    if n < 0 {
        return 0, fmt.Errorf("ไม่สามารถหาค่ารากที่สองของจำนวนลบได้")
    }
    return math.Sqrt(n), nil
}

func main() {
    a, b := 15.0, 4.0

    fmt.Printf("=== เครื่องคิดเลข (a=%.0f, b=%.0f) ===\n", a, b)
    fmt.Printf("a + b  = %.2f\n", add(a, b))
    fmt.Printf("a - b  = %.2f\n", subtract(a, b))
    fmt.Printf("a * b  = %.2f\n", multiply(a, b))

    if result, err := divide(a, b); err != nil {
        fmt.Printf("a / b  = Error: %v\n", err)
    } else {
        fmt.Printf("a / b  = %.4f\n", result)
    }

    fmt.Printf("a ^ b  = %.2f\n", power(a, b))

    if result, err := sqrt(a); err != nil {
        fmt.Printf("√a     = Error: %v\n", err)
    } else {
        fmt.Printf("√a     = %.4f\n", result)
    }

    // Test division by zero
    _, err := divide(a, 0)
    fmt.Printf("a / 0  = Error: %v\n", err)

    // Test negative sqrt
    _, err = sqrt(-4)
    fmt.Printf("√(-4)  = Error: %v\n", err)

    // Compound calculations
    fmt.Println("\n=== การคำนวณซับซ้อน ===")
    // สูตร quadratic: (-b ± √(b²-4ac)) / 2a
    qa, qb, qc := 1.0, -5.0, 6.0
    discriminant := qb*qb - 4*qa*qc
    if discriminant >= 0 {
        x1 := (-qb + math.Sqrt(discriminant)) / (2 * qa)
        x2 := (-qb - math.Sqrt(discriminant)) / (2 * qa)
        fmt.Printf("%.0fx² + %.0fx + %.0f = 0\n", qa, qb, qc)
        fmt.Printf("x1 = %.2f, x2 = %.2f\n", x1, x2)
    }
}
```

---

## 3.9 ตัวอย่าง: Bitwise Color Manipulation

```go
package main

import "fmt"

type Color uint32

func RGB(r, g, b uint8) Color {
    return Color(uint32(r)<<16 | uint32(g)<<8 | uint32(b))
}

func (c Color) R() uint8 { return uint8(c >> 16) }
func (c Color) G() uint8 { return uint8(c >> 8) }
func (c Color) B() uint8 { return uint8(c) }
func (c Color) Hex() string {
    return fmt.Sprintf("#%06X", uint32(c))
}

func Blend(c1, c2 Color, ratio float64) Color {
    r := uint8(float64(c1.R())*(1-ratio) + float64(c2.R())*ratio)
    g := uint8(float64(c1.G())*(1-ratio) + float64(c2.G())*ratio)
    b := uint8(float64(c1.B())*(1-ratio) + float64(c2.B())*ratio)
    return RGB(r, g, b)
}

func Grayscale(c Color) Color {
    // Luminance formula
    gray := uint8(0.299*float64(c.R()) + 0.587*float64(c.G()) + 0.114*float64(c.B()))
    return RGB(gray, gray, gray)
}

func main() {
    red := RGB(255, 0, 0)
    blue := RGB(0, 0, 255)
    green := RGB(0, 255, 0)
    white := RGB(255, 255, 255)

    fmt.Printf("Red:   %s (R=%d, G=%d, B=%d)\n", red.Hex(), red.R(), red.G(), red.B())
    fmt.Printf("Blue:  %s (R=%d, G=%d, B=%d)\n", blue.Hex(), blue.R(), blue.G(), blue.B())
    fmt.Printf("Green: %s (R=%d, G=%d, B=%d)\n", green.Hex(), green.R(), green.G(), green.B())
    fmt.Printf("White: %s (R=%d, G=%d, B=%d)\n", white.Hex(), white.R(), white.G(), white.B())

    // Blend red and blue -> purple
    purple := Blend(red, blue, 0.5)
    fmt.Printf("\nRed + Blue (50%%): %s\n", purple.Hex())

    // Grayscale
    orange := RGB(255, 165, 0)
    grayOrange := Grayscale(orange)
    fmt.Printf("Orange %s -> Grayscale %s\n", orange.Hex(), grayOrange.Hex())
}
```

---

## 3.10 ตัวอย่าง: Logical Operations ในระบบจริง

```go
package main

import (
    "fmt"
    "time"
)

type User struct {
    Name      string
    Age       int
    IsVerified bool
    IsBanned  bool
    Credits   float64
}

type Product struct {
    Name      string
    Price     float64
    InStock   bool
    AgeLimit  int
}

func canPurchase(user User, product Product) (bool, string) {
    if user.IsBanned {
        return false, "บัญชีถูกระงับ"
    }
    if !user.IsVerified {
        return false, "กรุณายืนยันตัวตนก่อน"
    }
    if !product.InStock {
        return false, "สินค้าหมด"
    }
    if product.AgeLimit > 0 && user.Age < product.AgeLimit {
        return false, fmt.Sprintf("ต้องอายุ %d ปีขึ้นไป", product.AgeLimit)
    }
    if user.Credits < product.Price {
        return false, fmt.Sprintf("เครดิตไม่พอ (มี %.2f ต้องการ %.2f)", user.Credits, product.Price)
    }
    return true, "ซื้อได้"
}

func isBusinessHour() bool {
    now := time.Now()
    hour := now.Hour()
    weekday := now.Weekday()
    isWeekday := weekday >= time.Monday && weekday <= time.Friday
    isWorkingHour := hour >= 9 && hour < 18
    return isWeekday && isWorkingHour
}

func main() {
    users := []User{
        {"สมชาย", 25, true, false, 1000.0},
        {"สมหญิง", 17, true, false, 500.0},
        {"สมศักดิ์", 30, false, false, 2000.0},
        {"สมใจ", 22, true, true, 800.0},
        {"สมพร", 28, true, false, 50.0},
    }

    product := Product{"สินค้าพรีเมียม", 299.0, true, 18}

    fmt.Printf("สินค้า: %s ราคา %.2f บาท\n\n", product.Name, product.Price)
    for _, user := range users {
        ok, msg := canPurchase(user, product)
        status := "❌"
        if ok {
            status = "✓"
        }
        fmt.Printf("%s %-10s: %s (%s)\n", status, user.Name, msg, func() string {
            if ok { return "ผ่าน" }
            return "ไม่ผ่าน"
        }())
    }

    fmt.Printf("\nเวลาทำการ: %v\n", isBusinessHour())
}
```

---

## Workshop: แบบฝึกหัด Part 3

### แบบฝึกหัดที่ 1: เครื่องคิดเลข BMI

เขียนโปรแกรมคำนวณ BMI:
- BMI = weight(kg) / height(m)²
- เกณฑ์: <18.5 (ผอม), 18.5-24.9 (ปกติ), 25-29.9 (น้ำหนักเกิน), ≥30 (อ้วน)
- แสดงผลพร้อม category

### แบบฝึกหัดที่ 2: Bit Flags

สร้างระบบ permission ด้วย bit flags:
- CREATE = 1, READ = 2, UPDATE = 4, DELETE = 8
- สร้าง function เพื่อ set, unset, check permission
- แสดง permission ที่มีทั้งหมด

### แบบฝึกหัดที่ 3: Temperature Range Checker

เขียนโปรแกรมตรวจสอบอุณหภูมิ:
- ถ้าต่ำกว่า 0°C = "หนาวมาก" และ เตือนว่า "ระวังน้ำแข็ง"
- ถ้า 0-15°C = "หนาว"
- ถ้า 15-28°C = "สบาย"
- ถ้า 28-35°C = "ร้อน"
- ถ้าสูงกว่า 35°C = "ร้อนมาก" และ เตือน "ดื่มน้ำมากๆ"

### เฉลยแบบฝึกหัดที่ 1

```go
package main

import (
    "fmt"
    "math"
)

func calculateBMI(weight, height float64) float64 {
    return weight / math.Pow(height, 2)
}

func bmiCategory(bmi float64) string {
    switch {
    case bmi < 18.5:
        return "น้ำหนักน้อย (ผอม)"
    case bmi < 25.0:
        return "น้ำหนักปกติ"
    case bmi < 30.0:
        return "น้ำหนักเกิน"
    default:
        return "อ้วน"
    }
}

func main() {
    testCases := []struct {
        name   string
        weight float64
        height float64
    }{
        {"สมชาย", 70.0, 1.75},
        {"สมหญิง", 45.0, 1.60},
        {"สมศักดิ์", 90.0, 1.70},
    }

    fmt.Println("=== BMI Calculator ===")
    for _, tc := range testCases {
        bmi := calculateBMI(tc.weight, tc.height)
        fmt.Printf("%-10s: น้ำหนัก=%.1fkg, ส่วนสูง=%.2fm, BMI=%.2f (%s)\n",
            tc.name, tc.weight, tc.height, bmi, bmiCategory(bmi))
    }
}
```

---

## สรุป Part 3

| Operator | ชนิด | ตัวอย่าง |
|----------|------|---------|
| `+` `-` `*` `/` `%` | Arithmetic | `10 % 3 == 1` |
| `==` `!=` `<` `>` `<=` `>=` | Comparison | `5 >= 3 == true` |
| `&&` `\|\|` `!` | Logical | `true && false == false` |
| `&` `\|` `^` `<<` `>>` `&^` | Bitwise | `0b1100 & 0b1010 == 0b1000` |
| `=` `+=` `-=` `*=` `/=` `%=` | Assignment | `x += 5` |
| `++` `--` | Increment/Decrement | `i++` (statement only) |

**Key Takeaways:**
- Integer division truncates (ไม่ปัดเศษ)
- Short-circuit evaluation ช่วยป้องกัน runtime errors
- `++` และ `--` เป็น statement ใน Go (ไม่ใช่ expression)
- ใช้ `()` เพื่อความชัดเจนในนิพจน์ซับซ้อน

## Resources

- [Go Spec - Operators](https://go.dev/ref/spec#Operators)
- [Go by Example - Variables](https://gobyexample.com/variables)
- [Go Tour - Basic types](https://go.dev/tour/basics/11)
