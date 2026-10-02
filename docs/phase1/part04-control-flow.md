# Part 4: Control Flow - if/else and switch

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ใช้ if/else statements ในรูปแบบต่างๆ ได้อย่างคล่องแคล่ว
- ใช้ if with initialization statement
- สร้าง switch statements ที่มีประสิทธิภาพ
- ใช้ fallthrough ใน switch ได้ถูกต้อง
- ใช้ type switch สำหรับ runtime type checking
- เลือกใช้ if หรือ switch ได้เหมาะสมกับสถานการณ์

---

## 4.1 if Statement พื้นฐาน

```go
package main

import "fmt"

func main() {
    temperature := 35.0

    // if แบบง่าย
    if temperature > 30 {
        fmt.Println("ร้อนมาก ควรดื่มน้ำเยอะๆ")
    }

    // if-else
    age := 17
    if age >= 18 {
        fmt.Println("ผู้ใหญ่ สามารถซื้อเครื่องดื่มแอลกอฮอล์ได้")
    } else {
        fmt.Println("ผู้เยาว์ ไม่สามารถซื้อเครื่องดื่มแอลกอฮอล์ได้")
    }

    // if-else if-else chain
    score := 75
    var grade string
    if score >= 90 {
        grade = "A"
    } else if score >= 80 {
        grade = "B"
    } else if score >= 70 {
        grade = "C"
    } else if score >= 60 {
        grade = "D"
    } else {
        grade = "F"
    }
    fmt.Printf("คะแนน %d = เกรด %s\n", score, grade)

    // if ที่ไม่มี else (early return pattern)
    name := ""
    if name == "" {
        fmt.Println("กรุณากรอกชื่อ")
        return
    }
    fmt.Printf("สวัสดี %s\n", name)
}
```

---

## 4.2 if with Initialization Statement

รูปแบบนี้เป็นเอกลักษณ์ของ Go ช่วยให้ scoping ของ variable ชัดเจน

```go
package main

import (
    "fmt"
    "strconv"
)

func getUser(id int) (string, bool) {
    users := map[int]string{
        1: "สมชาย",
        2: "สมหญิง",
        3: "สมศักดิ์",
    }
    name, ok := users[id]
    return name, ok
}

func parseInt(s string) (int, error) {
    return strconv.Atoi(s)
}

func main() {
    // if with initialization
    if user, ok := getUser(2); ok {
        fmt.Printf("พบผู้ใช้: %s\n", user)
    } else {
        fmt.Println("ไม่พบผู้ใช้")
    }

    // user variable จะ scope อยู่แค่ใน if block
    // ไม่สามารถใช้ user ข้างนอกได้

    // ตัวอย่าง: parse และตรวจสอบ
    inputs := []string{"42", "abc", "100", "-5", ""}
    for _, input := range inputs {
        if n, err := parseInt(input); err != nil {
            fmt.Printf("%-5q -> Error: %v\n", input, err)
        } else {
            fmt.Printf("%-5q -> %d\n", input, n)
        }
    }

    // ตัวอย่าง: file-like operation
    if result, err := processData("valid data"); err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("ผลลัพธ์: %s\n", result)
    }
}

func processData(data string) (string, error) {
    if data == "" {
        return "", fmt.Errorf("data ว่างเปล่า")
    }
    return "processed: " + data, nil
}
```

---

## 4.3 Nested if และ Complex Conditions

```go
package main

import "fmt"

type Product struct {
    Name      string
    Price     float64
    Stock     int
    Category  string
    IsActive  bool
}

func canOrder(product Product, quantity int, userBalance float64) (bool, string) {
    if !product.IsActive {
        return false, "สินค้าไม่เปิดขาย"
    }

    if product.Stock <= 0 {
        return false, "สินค้าหมด"
    }

    if quantity <= 0 {
        return false, "จำนวนต้องมากกว่า 0"
    }

    if quantity > product.Stock {
        return false, fmt.Sprintf("สต็อกมีแค่ %d ชิ้น", product.Stock)
    }

    totalPrice := product.Price * float64(quantity)
    if userBalance < totalPrice {
        return false, fmt.Sprintf("เงินไม่พอ (มี %.2f ต้องการ %.2f)", userBalance, totalPrice)
    }

    return true, fmt.Sprintf("สั่งซื้อได้ ราคารวม %.2f", totalPrice)
}

func main() {
    product := Product{
        Name:     "Go Programming Book",
        Price:    350.0,
        Stock:    10,
        Category: "books",
        IsActive: true,
    }

    tests := []struct {
        qty     int
        balance float64
    }{
        {2, 1000.0},
        {0, 1000.0},
        {15, 1000.0},
        {3, 100.0},
        {1, 350.0},
    }

    fmt.Printf("สินค้า: %s ราคา %.2f บาท (สต็อก: %d)\n\n",
        product.Name, product.Price, product.Stock)

    for _, t := range tests {
        ok, msg := canOrder(product, t.qty, t.balance)
        status := "✓"
        if !ok {
            status = "✗"
        }
        fmt.Printf("%s จำนวน=%d, เงิน=%.2f -> %s\n", status, t.qty, t.balance, msg)
    }
}
```

---

## 4.4 switch Statement พื้นฐาน

```go
package main

import "fmt"

func main() {
    day := "Wednesday"

    // Switch พื้นฐาน
    switch day {
    case "Monday":
        fmt.Println("วันจันทร์ เริ่มต้นสัปดาห์")
    case "Tuesday":
        fmt.Println("วันอังคาร")
    case "Wednesday":
        fmt.Println("วันพุธ กลางสัปดาห์แล้ว!")
    case "Thursday":
        fmt.Println("วันพฤหัสบดี")
    case "Friday":
        fmt.Println("วันศุกร์ ใกล้หยุดแล้ว!")
    case "Saturday", "Sunday": // หลาย case ในบรรทัดเดียว
        fmt.Println("วันหยุด!")
    default:
        fmt.Printf("ไม่รู้จักวัน: %s\n", day)
    }

    // Switch กับ int
    statusCode := 404
    switch statusCode {
    case 200:
        fmt.Println("OK")
    case 201:
        fmt.Println("Created")
    case 400:
        fmt.Println("Bad Request")
    case 401:
        fmt.Println("Unauthorized")
    case 403:
        fmt.Println("Forbidden")
    case 404:
        fmt.Println("Not Found")
    case 500:
        fmt.Println("Internal Server Error")
    default:
        fmt.Printf("Unknown status: %d\n", statusCode)
    }
}
```

---

## 4.5 switch with Initialization

```go
package main

import (
    "fmt"
    "time"
)

func getTimeOfDay() string {
    hour := time.Now().Hour()
    switch {
    case hour < 6:
        return "ดึกดื่น"
    case hour < 12:
        return "เช้า"
    case hour < 18:
        return "บ่าย"
    default:
        return "เย็น/ค่ำ"
    }
}

func main() {
    // Switch with initialization
    switch lang := "Go"; lang {
    case "Go":
        fmt.Println("Gopher!")
    case "Python":
        fmt.Println("Pythonista!")
    case "JavaScript":
        fmt.Println("JavaScripter!")
    default:
        fmt.Printf("นักพัฒนา %s\n", lang)
    }

    // Switch กับ time
    switch day := time.Now().Weekday(); day {
    case time.Saturday, time.Sunday:
        fmt.Println("วันหยุด!")
    default:
        fmt.Printf("วันทำงาน: %s\n", day)
    }

    fmt.Printf("ตอนนี้เป็น%s\n", getTimeOfDay())
}
```

---

## 4.6 switch ไม่มี condition (แทน if-else chain)

```go
package main

import "fmt"

func classifyBMI(bmi float64) string {
    switch {
    case bmi < 18.5:
        return "น้ำหนักน้อยกว่าเกณฑ์"
    case bmi < 25.0:
        return "น้ำหนักปกติ"
    case bmi < 30.0:
        return "น้ำหนักเกิน"
    case bmi < 35.0:
        return "อ้วนระดับ 1"
    case bmi < 40.0:
        return "อ้วนระดับ 2"
    default:
        return "อ้วนระดับ 3 (ต้องพบแพทย์)"
    }
}

func getDiscount(purchaseAmount float64, isMember bool, isNewUser bool) float64 {
    switch {
    case isNewUser && purchaseAmount >= 500:
        return 0.20 // 20% สำหรับ new user ซื้อ >= 500
    case isNewUser:
        return 0.10 // 10% สำหรับ new user ทุกยอด
    case isMember && purchaseAmount >= 1000:
        return 0.15 // 15% สำหรับสมาชิก ซื้อ >= 1000
    case isMember:
        return 0.05 // 5% สำหรับสมาชิก
    case purchaseAmount >= 2000:
        return 0.10 // 10% สำหรับซื้อมาก
    default:
        return 0
    }
}

func main() {
    bmis := []float64{16.0, 22.5, 27.3, 32.1, 37.5, 42.0}
    for _, bmi := range bmis {
        fmt.Printf("BMI %.1f: %s\n", bmi, classifyBMI(bmi))
    }

    fmt.Println()
    type ShopCase struct {
        amount   float64
        member   bool
        newUser  bool
    }
    cases := []ShopCase{
        {600, false, true},
        {300, false, true},
        {1500, true, false},
        {500, true, false},
        {2500, false, false},
        {100, false, false},
    }
    for _, c := range cases {
        disc := getDiscount(c.amount, c.member, c.newUser)
        fmt.Printf("ซื้อ %.0f, สมาชิก=%v, ใหม่=%v -> ส่วนลด %.0f%%\n",
            c.amount, c.member, c.newUser, disc*100)
    }
}
```

---

## 4.7 fallthrough

```go
package main

import "fmt"

func main() {
    // Go switch ไม่ fallthrough โดย default (ต่างจาก C/Java)
    n := 2
    switch n {
    case 1:
        fmt.Println("หนึ่ง")
    case 2:
        fmt.Println("สอง")
        // จะไม่ไปทำ case 3 โดยอัตโนมัติ
    case 3:
        fmt.Println("สาม")
    }

    fmt.Println("---")

    // ใช้ fallthrough เพื่อบังคับ
    m := 2
    switch m {
    case 1:
        fmt.Println("หนึ่ง")
        fallthrough
    case 2:
        fmt.Println("สอง")
        fallthrough // จะทำ case ถัดไปโดยไม่ตรวจ condition
    case 3:
        fmt.Println("สาม")
        fallthrough
    case 4:
        fmt.Println("สี่")
    case 5:
        fmt.Println("ห้า")
    }

    fmt.Println("---")

    // ตัวอย่างการใช้งานจริง: semantic version range
    version := 3
    fmt.Printf("Version %d features:\n", version)
    switch version {
    case 5:
        fmt.Println("  - Feature E (v5)")
        fallthrough
    case 4:
        fmt.Println("  - Feature D (v4)")
        fallthrough
    case 3:
        fmt.Println("  - Feature C (v3)")
        fallthrough
    case 2:
        fmt.Println("  - Feature B (v2)")
        fallthrough
    case 1:
        fmt.Println("  - Feature A (v1) - base")
    default:
        fmt.Println("  Unknown version")
    }
}
```

---

## 4.8 Type Switch

Type switch ใช้สำหรับตรวจสอบ type ของ interface value ณ runtime

```go
package main

import "fmt"

func describe(i interface{}) string {
    switch v := i.(type) {
    case nil:
        return "nil"
    case bool:
        return fmt.Sprintf("bool: %v", v)
    case int:
        return fmt.Sprintf("int: %d", v)
    case int64:
        return fmt.Sprintf("int64: %d", v)
    case float64:
        return fmt.Sprintf("float64: %f", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case []int:
        return fmt.Sprintf("[]int: %v (len=%d)", v, len(v))
    case map[string]int:
        return fmt.Sprintf("map[string]int: %v", v)
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

func main() {
    values := []interface{}{
        nil,
        true,
        42,
        int64(100),
        3.14,
        "Hello, Go!",
        []int{1, 2, 3},
        map[string]int{"a": 1},
        struct{ Name string }{"Go"},
    }

    for _, v := range values {
        fmt.Printf("%-40s\n", describe(v))
    }
}
```

### Type Switch กับ Interface

```go
package main

import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
    Perimeter() float64
}

type Circle struct {
    Radius float64
}

type Rectangle struct {
    Width, Height float64
}

type Triangle struct {
    A, B, C float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}
func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}
func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Heron's formula
func (t Triangle) Area() float64 {
    s := (t.A + t.B + t.C) / 2
    return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
}
func (t Triangle) Perimeter() float64 {
    return t.A + t.B + t.C
}

func describeShape(s Shape) {
    switch v := s.(type) {
    case Circle:
        fmt.Printf("วงกลม (r=%.2f): พื้นที่=%.4f, เส้นรอบวง=%.4f\n",
            v.Radius, v.Area(), v.Perimeter())
    case Rectangle:
        fmt.Printf("สี่เหลี่ยม (%,.2fx%.2f): พื้นที่=%.4f, เส้นรอบรูป=%.4f\n",
            v.Width, v.Height, v.Area(), v.Perimeter())
    case Triangle:
        fmt.Printf("สามเหลี่ยม (a=%.2f,b=%.2f,c=%.2f): พื้นที่=%.4f, เส้นรอบรูป=%.4f\n",
            v.A, v.B, v.C, v.Area(), v.Perimeter())
    default:
        fmt.Printf("รูปทรงที่ไม่รู้จัก: %T\n", v)
    }
}

func main() {
    shapes := []Shape{
        Circle{Radius: 5},
        Rectangle{Width: 4, Height: 6},
        Triangle{A: 3, B: 4, C: 5},
    }

    for _, shape := range shapes {
        describeShape(shape)
    }

    // หาพื้นที่รวม
    var totalArea float64
    for _, s := range shapes {
        totalArea += s.Area()
    }
    fmt.Printf("\nพื้นที่รวมทั้งหมด: %.4f\n", totalArea)
}
```

---

## 4.9 ตัวอย่างรวม: State Machine

```go
package main

import "fmt"

type OrderStatus int

const (
    OrderPending OrderStatus = iota
    OrderConfirmed
    OrderPreparing
    OrderShipping
    OrderDelivered
    OrderCancelled
    OrderRefunded
)

func (s OrderStatus) String() string {
    switch s {
    case OrderPending:
        return "รอดำเนินการ"
    case OrderConfirmed:
        return "ยืนยันแล้ว"
    case OrderPreparing:
        return "กำลังเตรียมสินค้า"
    case OrderShipping:
        return "กำลังจัดส่ง"
    case OrderDelivered:
        return "ส่งมอบแล้ว"
    case OrderCancelled:
        return "ยกเลิก"
    case OrderRefunded:
        return "คืนเงินแล้ว"
    default:
        return fmt.Sprintf("Unknown(%d)", s)
    }
}

func (s OrderStatus) NextStates() []OrderStatus {
    switch s {
    case OrderPending:
        return []OrderStatus{OrderConfirmed, OrderCancelled}
    case OrderConfirmed:
        return []OrderStatus{OrderPreparing, OrderCancelled}
    case OrderPreparing:
        return []OrderStatus{OrderShipping}
    case OrderShipping:
        return []OrderStatus{OrderDelivered}
    case OrderDelivered:
        return []OrderStatus{OrderRefunded}
    default:
        return nil
    }
}

func canTransition(from, to OrderStatus) bool {
    for _, next := range from.NextStates() {
        if next == to {
            return true
        }
    }
    return false
}

type Order struct {
    ID     string
    Status OrderStatus
}

func (o *Order) Transition(newStatus OrderStatus) error {
    if !canTransition(o.Status, newStatus) {
        return fmt.Errorf("ไม่สามารถเปลี่ยนจาก '%s' เป็น '%s' ได้",
            o.Status, newStatus)
    }
    o.Status = newStatus
    return nil
}

func main() {
    order := &Order{ID: "ORD-001", Status: OrderPending}
    fmt.Printf("สร้าง Order %s: %s\n", order.ID, order.Status)

    transitions := []OrderStatus{
        OrderConfirmed,
        OrderPreparing,
        OrderShipping,
        OrderDelivered,
    }

    for _, newStatus := range transitions {
        if err := order.Transition(newStatus); err != nil {
            fmt.Printf("Error: %v\n", err)
        } else {
            fmt.Printf("เปลี่ยนสถานะเป็น: %s\n", order.Status)
        }
    }

    // ลองเปลี่ยนสถานะที่ไม่ถูกต้อง
    fmt.Println("\nลองเปลี่ยนสถานะไม่ถูกต้อง:")
    if err := order.Transition(OrderCancelled); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

---

## 4.10 ตัวอย่าง: Command Dispatcher

```go
package main

import (
    "fmt"
    "strings"
)

type Command struct {
    Name string
    Args []string
}

func parseCommand(input string) Command {
    parts := strings.Fields(input)
    if len(parts) == 0 {
        return Command{}
    }
    return Command{
        Name: strings.ToLower(parts[0]),
        Args: parts[1:],
    }
}

func executeCommand(cmd Command) {
    switch cmd.Name {
    case "help":
        fmt.Println("Commands: help, greet, add, version, quit")

    case "greet":
        if len(cmd.Args) > 0 {
            fmt.Printf("สวัสดีครับ %s!\n", strings.Join(cmd.Args, " "))
        } else {
            fmt.Println("สวัสดีครับ!")
        }

    case "add":
        if len(cmd.Args) < 2 {
            fmt.Println("ใช้: add <num1> <num2>")
            return
        }
        var a, b int
        fmt.Sscanf(cmd.Args[0], "%d", &a)
        fmt.Sscanf(cmd.Args[1], "%d", &b)
        fmt.Printf("%d + %d = %d\n", a, b, a+b)

    case "version":
        fmt.Println("Go Course CLI v1.0.0")

    case "quit", "exit", "q":
        fmt.Println("ลาก่อน!")

    case "":
        // ไม่ทำอะไร

    default:
        fmt.Printf("ไม่รู้จัก command: %s (พิมพ์ 'help' สำหรับคำสั่งทั้งหมด)\n", cmd.Name)
    }
}

func main() {
    inputs := []string{
        "help",
        "greet สมชาย",
        "add 15 27",
        "version",
        "unknown",
        "quit",
    }

    for _, input := range inputs {
        fmt.Printf("> %s\n", input)
        cmd := parseCommand(input)
        executeCommand(cmd)
        fmt.Println()
    }
}
```

---

## 4.11 เปรียบเทียบ if vs switch

```go
package main

import "fmt"

func getGradeIf(score int) string {
    if score >= 90 {
        return "A"
    } else if score >= 80 {
        return "B"
    } else if score >= 70 {
        return "C"
    } else if score >= 60 {
        return "D"
    } else {
        return "F"
    }
}

func getGradeSwitch(score int) string {
    switch {
    case score >= 90:
        return "A"
    case score >= 80:
        return "B"
    case score >= 70:
        return "C"
    case score >= 60:
        return "D"
    default:
        return "F"
    }
}

// Switch อ่านง่ายกว่าเมื่อตรวจสอบ discrete values
func getDayTypeIf(day string) string {
    if day == "Saturday" || day == "Sunday" {
        return "วันหยุด"
    }
    return "วันทำงาน"
}

func getDayTypeSwitch(day string) string {
    switch day {
    case "Saturday", "Sunday":
        return "วันหยุด"
    default:
        return "วันทำงาน"
    }
}

func main() {
    scores := []int{95, 85, 75, 65, 55}
    for _, s := range scores {
        fmt.Printf("คะแนน %d: if=%s, switch=%s\n",
            s, getGradeIf(s), getGradeSwitch(s))
    }
}
```

---

## Workshop: แบบฝึกหัด Part 4

### แบบฝึกหัดที่ 1: เครื่องแปลงป้ายทะเบียนรถ

เขียน function ที่รับจังหวัดเป็น string และ return ภาค:
- กรุงเทพ = "กลาง"
- เชียงใหม่, เชียงราย, ลำพูน, ลำปาง = "เหนือ"
- นครราชสีมา, ขอนแก่น, อุดรธานี = "อีสาน"
- ภูเก็ต, กระบี่, สุราษฎร์ธานี = "ใต้"

### แบบฝึกหัดที่ 2: ระบบ Login

เขียนโปรแกรมจำลองระบบ login:
- ถ้า username = "" -> "กรุณากรอก username"
- ถ้า password = "" -> "กรุณากรอก password"
- ถ้า username = "admin" และ password = "1234" -> "เข้าสู่ระบบสำเร็จ"
- อื่นๆ -> "username หรือ password ไม่ถูกต้อง"
- พยายาม login ผิด 3 ครั้ง -> "บัญชีถูกล็อค"

### แบบฝึกหัดที่ 3: Type Checker

เขียน function `typeInfo(v interface{}) string` ที่:
- ตรวจสอบ type
- ถ้าเป็น int -> บอก positive/negative/zero
- ถ้าเป็น string -> บอก length
- ถ้าเป็น bool -> แปลงเป็น "ใช่"/"ไม่ใช่"
- ถ้าเป็น float64 -> round เป็น 2 ทศนิยม
- อื่นๆ -> บอก type name

### เฉลยแบบฝึกหัดที่ 1

```go
package main

import "fmt"

func getRegion(province string) string {
    switch province {
    case "กรุงเทพมหานคร", "นนทบุรี", "ปทุมธานี", "สมุทรปราการ":
        return "กลาง"
    case "เชียงใหม่", "เชียงราย", "ลำพูน", "ลำปาง", "แม่ฮ่องสอน":
        return "เหนือ"
    case "นครราชสีมา", "ขอนแก่น", "อุดรธานี", "อุบลราชธานี", "สกลนคร":
        return "อีสาน"
    case "ภูเก็ต", "กระบี่", "สุราษฎร์ธานี", "นครศรีธรรมราช", "สงขลา":
        return "ใต้"
    case "ชลบุรี", "ระยอง", "จันทบุรี", "ตราด":
        return "ตะวันออก"
    default:
        return "ไม่ทราบภาค"
    }
}

func main() {
    provinces := []string{
        "กรุงเทพมหานคร", "เชียงใหม่",
        "ขอนแก่น", "ภูเก็ต", "ชลบุรี",
    }
    for _, p := range provinces {
        fmt.Printf("%-20s -> ภาค%s\n", p, getRegion(p))
    }
}
```

---

## สรุป Part 4

| Statement | การใช้งาน |
|-----------|----------|
| `if` | ตรวจสอบ condition เดียว |
| `if-else` | แยก 2 เส้นทาง |
| `if init; condition` | ประกาศตัวแปร + ตรวจสอบ |
| `switch value` | เปรียบเทียบค่า |
| `switch {}` | แทน if-else chain |
| `switch type` | Type assertion |
| `fallthrough` | ข้ามไป case ถัดไป |

**Key Takeaways:**
- Go switch ไม่ fallthrough โดย default (ต่างจาก C/Java)
- ใช้ if initialization statement เพื่อ scope ตัวแปรให้แคบที่สุด
- Type switch ใช้ `switch v := i.(type)`
- switch โดยไม่มี condition ทำงานเหมือน if-else chain

## Resources

- [Go Spec - If statements](https://go.dev/ref/spec#If_statements)
- [Go Spec - Switch statements](https://go.dev/ref/spec#Switch_statements)
- [Go by Example - If/Else](https://gobyexample.com/if-else)
- [Go by Example - Switch](https://gobyexample.com/switch)
- [Effective Go - Switch](https://go.dev/doc/effective_go#switch)
