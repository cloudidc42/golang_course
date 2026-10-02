# Part 6: Functions

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ประกาศและใช้งาน functions ในรูปแบบต่างๆ
- ใช้ multiple return values ได้อย่างเหมาะสม
- ใช้ named return values
- สร้างและใช้ variadic functions
- เข้าใจและใช้ closures/anonymous functions
- ใช้ higher-order functions (functions as values)
- เขียน recursive functions ที่ถูกต้อง
- เข้าใจ init function และลำดับการทำงาน

---

## 6.1 Function Declaration พื้นฐาน

```go
package main

import "fmt"

// ฟังก์ชันที่ไม่มี parameter ไม่มี return value
func sayHello() {
    fmt.Println("สวัสดีครับ!")
}

// ฟังก์ชันที่มี parameter
func greet(name string) {
    fmt.Printf("สวัสดีครับ %s!\n", name)
}

// ฟังก์ชันที่มี parameter และ return value
func add(a, b int) int {
    return a + b
}

// parameters ที่มี type เดียวกัน เขียนย่อได้
func addXY(x, y int) int {
    return x + y
}

// ฟังก์ชันที่มี parameter หลาย type
func formatName(firstName, lastName string, age int) string {
    return fmt.Sprintf("%s %s (อายุ %d ปี)", firstName, lastName, age)
}

func main() {
    sayHello()
    greet("สมชาย")
    fmt.Printf("3 + 4 = %d\n", add(3, 4))
    fmt.Println(formatName("สมชาย", "จารึก", 28))
}
```

---

## 6.2 Multiple Return Values

ฟีเจอร์เด่นของ Go คือสามารถ return หลายค่าพร้อมกัน

```go
package main

import (
    "errors"
    "fmt"
    "math"
)

// Return 2 ค่า
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("ไม่สามารถหารด้วยศูนย์ได้")
    }
    return a / b, nil
}

// Return หลายค่า
func minMax(nums []int) (min, max int, err error) {
    if len(nums) == 0 {
        return 0, 0, errors.New("slice ว่างเปล่า")
    }
    min, max = nums[0], nums[0]
    for _, n := range nums[1:] {
        if n < min {
            min = n
        }
        if n > max {
            max = n
        }
    }
    return min, max, nil
}

// Swap 2 ค่า
func swap(a, b int) (int, int) {
    return b, a
}

// คืน statistics
func statistics(data []float64) (mean, stddev float64, count int) {
    count = len(data)
    if count == 0 {
        return
    }
    for _, v := range data {
        mean += v
    }
    mean /= float64(count)
    var sumSq float64
    for _, v := range data {
        d := v - mean
        sumSq += d * d
    }
    stddev = math.Sqrt(sumSq / float64(count))
    return
}

func main() {
    // ใช้ multiple return values
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("10 / 3 = %.4f\n", result)
    }

    _, err = divide(5, 0)
    if err != nil {
        fmt.Println("Error:", err)
    }

    // minMax
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}
    if min, max, err := minMax(nums); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Min=%d, Max=%d\n", min, max)
    }

    // Swap
    a, b := 10, 20
    a, b = swap(a, b)
    fmt.Printf("หลัง swap: a=%d, b=%d\n", a, b)

    // Statistics
    data := []float64{85.5, 92.0, 78.3, 95.5, 88.0}
    mean, std, n := statistics(data)
    fmt.Printf("mean=%.2f, stddev=%.2f, count=%d\n", mean, std, n)
}
```

---

## 6.3 Named Return Values

```go
package main

import "fmt"

// Named return values
func circleMetrics(radius float64) (area, perimeter float64) {
    const pi = 3.14159265358979
    area = pi * radius * radius
    perimeter = 2 * pi * radius
    return // naked return
}

// Named return ช่วยให้โค้ดอ่านง่ายขึ้น
func parseAddress(address string) (street, city, country string) {
    // จำลอง parsing
    street = "123 ถนนสุขุมวิท"
    city = "กรุงเทพ"
    country = "ไทย"
    return
}

// Named return กับ error handling
func safeDivide(a, b float64) (result float64, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()

    if b == 0 {
        err = fmt.Errorf("division by zero")
        return // result จะเป็น 0 (zero value)
    }
    result = a / b
    return
}

func main() {
    area, perimeter := circleMetrics(5.0)
    fmt.Printf("วงกลมรัศมี 5: area=%.4f, perimeter=%.4f\n", area, perimeter)

    street, city, country := parseAddress("...")
    fmt.Printf("ที่อยู่: %s, %s, %s\n", street, city, country)

    if r, err := safeDivide(10, 3); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("10 / 3 = %.4f\n", r)
    }
}
```

---

## 6.4 Variadic Functions

```go
package main

import "fmt"

// Variadic function: รับ arguments ได้ไม่จำกัด
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// Mix ระหว่าง regular และ variadic
func printWithPrefix(prefix string, values ...interface{}) {
    fmt.Printf("[%s] ", prefix)
    fmt.Println(values...)
}

// Pass slice ให้ variadic function ด้วย ...
func maxOf(first int, rest ...int) int {
    max := first
    for _, n := range rest {
        if n > max {
            max = n
        }
    }
    return max
}

// Variadic string concatenation
func concat(sep string, strs ...string) string {
    result := ""
    for i, s := range strs {
        if i > 0 {
            result += sep
        }
        result += s
    }
    return result
}

func main() {
    // เรียกด้วยจำนวนต่างกัน
    fmt.Println(sum())           // 0
    fmt.Println(sum(1))          // 1
    fmt.Println(sum(1, 2, 3))    // 6
    fmt.Println(sum(1, 2, 3, 4, 5)) // 15

    // Spread slice ด้วย ...
    nums := []int{10, 20, 30, 40, 50}
    fmt.Printf("sum of %v = %d\n", nums, sum(nums...))

    // Mixed variadic
    printWithPrefix("INFO", "สวัสดี", 42, true)
    printWithPrefix("ERROR", "เกิดข้อผิดพลาด", 404)

    fmt.Println(maxOf(5, 3, 8, 1, 9, 2)) // 9

    fmt.Println(concat(", ", "Go", "Python", "Java")) // Go, Python, Java
    fmt.Println(concat("-", "2024", "01", "15"))      // 2024-01-15
}
```

---

## 6.5 Anonymous Functions และ Closures

```go
package main

import "fmt"

func main() {
    // Anonymous function (ประกาศและเรียกทันที)
    result := func(a, b int) int {
        return a + b
    }(3, 4)
    fmt.Printf("3 + 4 = %d\n", result)

    // เก็บ function ในตัวแปร
    greet := func(name string) string {
        return fmt.Sprintf("สวัสดีครับ %s!", name)
    }
    fmt.Println(greet("สมชาย"))

    // Closure: function ที่ capture ตัวแปรจาก outer scope
    counter := func() func() int {
        count := 0 // state ที่ closure capture
        return func() int {
            count++
            return count
        }
    }()

    fmt.Println(counter()) // 1
    fmt.Println(counter()) // 2
    fmt.Println(counter()) // 3

    // Closure แยกกัน มี state แยกกัน
    c1 := makeCounter()
    c2 := makeCounter()
    fmt.Printf("c1: %d, %d\n", c1(), c1())
    fmt.Printf("c2: %d\n", c2()) // เริ่มต้นใหม่

    // Closure capture by reference
    x := 10
    add := func(n int) {
        x += n
    }
    add(5)
    add(3)
    fmt.Printf("x = %d\n", x) // 18

    // ระวัง: closure loop bug!
    funcs := make([]func(), 5)
    for i := 0; i < 5; i++ {
        i := i // shadow i (สร้าง copy ใหม่ในแต่ละรอบ)
        funcs[i] = func() {
            fmt.Printf("i = %d\n", i)
        }
    }
    for _, f := range funcs {
        f()
    }
}

func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
```

---

## 6.6 Higher-Order Functions

```go
package main

import (
    "fmt"
    "sort"
)

// Function type
type Predicate func(int) bool
type Transform func(int) int
type Reducer func(int, int) int

// Map: apply function ให้กับทุก element
func mapSlice(nums []int, f Transform) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = f(n)
    }
    return result
}

// Filter: เลือก elements ที่ผ่านเงื่อนไข
func filterSlice(nums []int, pred Predicate) []int {
    var result []int
    for _, n := range nums {
        if pred(n) {
            result = append(result, n)
        }
    }
    return result
}

// Reduce: รวม elements เป็นค่าเดียว
func reduce(nums []int, initial int, f Reducer) int {
    result := initial
    for _, n := range nums {
        result = f(result, n)
    }
    return result
}

// Function composition
func compose(f, g Transform) Transform {
    return func(x int) int {
        return f(g(x))
    }
}

// Memoize: cache ผลลัพธ์
func memoize(f func(int) int) func(int) int {
    cache := make(map[int]int)
    return func(n int) int {
        if v, ok := cache[n]; ok {
            return v
        }
        v := f(n)
        cache[n] = v
        return v
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Map
    squared := mapSlice(nums, func(n int) int { return n * n })
    doubled := mapSlice(nums, func(n int) int { return n * 2 })
    fmt.Printf("original: %v\n", nums)
    fmt.Printf("squared:  %v\n", squared)
    fmt.Printf("doubled:  %v\n", doubled)

    // Filter
    evens := filterSlice(nums, func(n int) bool { return n%2 == 0 })
    gt5 := filterSlice(nums, func(n int) bool { return n > 5 })
    fmt.Printf("evens:    %v\n", evens)
    fmt.Printf("> 5:      %v\n", gt5)

    // Reduce
    sum := reduce(nums, 0, func(acc, n int) int { return acc + n })
    product := reduce(nums, 1, func(acc, n int) int { return acc * n })
    fmt.Printf("sum:     %d\n", sum)
    fmt.Printf("product: %d\n", product)

    // Compose
    double := func(x int) int { return x * 2 }
    addOne := func(x int) int { return x + 1 }
    doubleAndAddOne := compose(addOne, double)
    fmt.Printf("doubleAndAddOne(5) = %d\n", doubleAndAddOne(5)) // (5*2)+1=11

    // Sort with function
    people := []struct{ Name string; Age int }{
        {"สมชาย", 28}, {"สมหญิง", 22}, {"สมศักดิ์", 35}, {"สมใจ", 19},
    }
    sort.Slice(people, func(i, j int) bool {
        return people[i].Age < people[j].Age
    })
    fmt.Println("\nเรียงตามอายุ:")
    for _, p := range people {
        fmt.Printf("  %s: %d\n", p.Name, p.Age)
    }

    // Memoize fibonacci
    var fib func(int) int
    fib = func(n int) int {
        if n <= 1 {
            return n
        }
        return fib(n-1) + fib(n-2)
    }
    memoFib := memoize(fib)
    for i := 0; i <= 10; i++ {
        fmt.Printf("fib(%d) = %d\n", i, memoFib(i))
    }
}
```

---

## 6.7 Function Types

```go
package main

import "fmt"

// Function type definition
type Handler func(string) string
type Middleware func(Handler) Handler

// ใช้ function type เป็น field ใน struct
type Pipeline struct {
    handlers []Handler
}

func (p *Pipeline) Use(h Handler) {
    p.handlers = append(p.handlers, h)
}

func (p *Pipeline) Execute(input string) string {
    result := input
    for _, h := range p.handlers {
        result = h(result)
    }
    return result
}

// Middleware pattern
func logging(next Handler) Handler {
    return func(s string) string {
        fmt.Printf("[LOG] Input: %q\n", s)
        result := next(s)
        fmt.Printf("[LOG] Output: %q\n", result)
        return result
    }
}

func main() {
    // Function ใน map
    operations := map[string]func(int, int) int{
        "+": func(a, b int) int { return a + b },
        "-": func(a, b int) int { return a - b },
        "*": func(a, b int) int { return a * b },
        "/": func(a, b int) int {
            if b == 0 { return 0 }
            return a / b
        },
    }

    for op, fn := range operations {
        fmt.Printf("10 %s 3 = %d\n", op, fn(10, 3))
    }

    // Pipeline
    pipeline := &Pipeline{}
    pipeline.Use(func(s string) string {
        return "[" + s + "]"
    })
    pipeline.Use(func(s string) string {
        return s + "!"
    })

    fmt.Println(pipeline.Execute("Hello")) // [Hello]!

    // Middleware
    var process Handler = func(s string) string {
        return "processed: " + s
    }
    withLogging := logging(process)
    withLogging("test input")
}
```

---

## 6.8 Recursion

```go
package main

import "fmt"

// Factorial
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

// Fibonacci (recursive)
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

// Power
func power(base, exp int) int {
    if exp == 0 {
        return 1
    }
    if exp%2 == 0 {
        half := power(base, exp/2)
        return half * half
    }
    return base * power(base, exp-1)
}

// Binary search (recursive)
func binarySearch(arr []int, target, low, high int) int {
    if low > high {
        return -1
    }
    mid := (low + high) / 2
    if arr[mid] == target {
        return mid
    } else if arr[mid] < target {
        return binarySearch(arr, target, mid+1, high)
    }
    return binarySearch(arr, target, low, mid-1)
}

// Flatten nested slice
func flatten(nested []interface{}) []int {
    var result []int
    for _, item := range nested {
        switch v := item.(type) {
        case int:
            result = append(result, v)
        case []interface{}:
            result = append(result, flatten(v)...)
        }
    }
    return result
}

// Tower of Hanoi
func hanoi(n int, from, to, via string) {
    if n == 1 {
        fmt.Printf("ย้ายแผ่น 1 จาก %s ไป %s\n", from, to)
        return
    }
    hanoi(n-1, from, via, to)
    fmt.Printf("ย้ายแผ่น %d จาก %s ไป %s\n", n, from, to)
    hanoi(n-1, via, to, from)
}

func main() {
    // Factorial
    for i := 0; i <= 10; i++ {
        fmt.Printf("%2d! = %d\n", i, factorial(i))
    }

    // Fibonacci
    fmt.Print("\nFibonacci: ")
    for i := 0; i < 10; i++ {
        fmt.Printf("%d ", fibonacci(i))
    }
    fmt.Println()

    // Power
    fmt.Printf("\n2^10 = %d\n", power(2, 10))
    fmt.Printf("3^5  = %d\n", power(3, 5))

    // Binary search
    arr := []int{1, 3, 5, 7, 9, 11, 13, 15, 17, 19}
    fmt.Printf("\nค้นหา 11 ใน %v = index %d\n", arr, binarySearch(arr, 11, 0, len(arr)-1))
    fmt.Printf("ค้นหา 6 ใน %v = index %d\n", arr, binarySearch(arr, 6, 0, len(arr)-1))

    // Hanoi
    fmt.Println("\nHanoi 3 แผ่น:")
    hanoi(3, "A", "C", "B")
}
```

---

## 6.9 init Function

```go
package main

import "fmt"

// init() ถูกเรียกอัตโนมัติ ก่อน main()
// แต่ละ package อาจมีหลาย init() ก็ได้
// init() ไม่รับ argument และไม่ return ค่า

var (
    config map[string]string
    db     []string
)

func init() {
    fmt.Println("[init 1] เริ่มต้น config")
    config = map[string]string{
        "host": "localhost",
        "port": "5432",
        "name": "mydb",
    }
}

func init() {
    fmt.Println("[init 2] เชื่อมต่อ database (จำลอง)")
    db = append(db, "connected to "+config["host"]+":"+config["port"]+"/"+config["name"])
}

func init() {
    fmt.Println("[init 3] โหลด initial data")
    db = append(db, "user1", "user2", "user3")
}

func main() {
    fmt.Println("[main] โปรแกรมเริ่มทำงาน")
    fmt.Printf("Config: %v\n", config)
    fmt.Printf("DB: %v\n", db)
}
```

**ผลลัพธ์:**
```
[init 1] เริ่มต้น config
[init 2] เชื่อมต่อ database (จำลอง)
[init 3] โหลด initial data
[main] โปรแกรมเริ่มทำงาน
Config: map[host:localhost name:mydb port:5432]
DB: [connected to localhost:5432/mydb user1 user2 user3]
```

---

## 6.10 Defer

```go
package main

import "fmt"

func cleanup() {
    fmt.Println("cleanup ถูกเรียก")
}

func withDefer() {
    defer cleanup() // จะถูกเรียกเมื่อ function นี้จบ
    fmt.Println("กำลังทำงาน...")
    fmt.Println("ทำงานเสร็จ")
}

// Defer stack: LIFO
func deferStack() {
    for i := 1; i <= 5; i++ {
        defer fmt.Printf("defer %d\n", i)
    }
    fmt.Println("function body")
}

// Defer กับ named return
func divide(a, b float64) (result float64, err error) {
    defer func() {
        if err != nil {
            fmt.Printf("divide(%v, %v) failed: %v\n", a, b, err)
        }
    }()

    if b == 0 {
        err = fmt.Errorf("division by zero")
        return
    }
    result = a / b
    return
}

// Resource cleanup pattern
func processFile(filename string) error {
    fmt.Printf("เปิดไฟล์ %s\n", filename)
    // เปิดไฟล์จริงๆ: f, err := os.Open(filename)
    defer fmt.Printf("ปิดไฟล์ %s\n", filename) // จะถูกเรียกเสมอ

    fmt.Println("อ่านข้อมูล...")
    // ทำงานกับไฟล์...
    return nil
}

func main() {
    withDefer()
    fmt.Println()

    deferStack()
    fmt.Println()

    divide(10, 3)
    divide(10, 0)
    fmt.Println()

    processFile("data.txt")
}
```

---

## 6.11 ตัวอย่างรวม: Functional Pipeline

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

// Pipeline operations
type StringOp func(string) string

func pipeline(input string, ops ...StringOp) string {
    result := input
    for _, op := range ops {
        result = op(result)
    }
    return result
}

// Operations
func trim(s string) string         { return strings.TrimSpace(s) }
func lower(s string) string        { return strings.ToLower(s) }
func upper(s string) string        { return strings.ToUpper(s) }
func removeSpaces(s string) string { return strings.ReplaceAll(s, " ", "") }

func titleCase(s string) string {
    words := strings.Fields(s)
    for i, w := range words {
        if len(w) > 0 {
            words[i] = strings.ToUpper(w[:1]) + strings.ToLower(w[1:])
        }
    }
    return strings.Join(words, " ")
}

func removeNonAlpha(s string) string {
    var result strings.Builder
    for _, r := range s {
        if unicode.IsLetter(r) || unicode.IsSpace(r) {
            result.WriteRune(r)
        }
    }
    return result.String()
}

func repeat(n int) StringOp {
    return func(s string) string {
        return strings.Repeat(s, n)
    }
}

func addPrefix(prefix string) StringOp {
    return func(s string) string {
        return prefix + s
    }
}

func addSuffix(suffix string) StringOp {
    return func(s string) string {
        return s + suffix
    }
}

func main() {
    inputs := []string{
        "  HELLO WORLD  ",
        "  สวัสดีครับ  ",
        "Hello, 123 World!",
    }

    for _, input := range inputs {
        result := pipeline(input,
            trim,
            removeNonAlpha,
            trim,
            titleCase,
        )
        fmt.Printf("%-25q -> %q\n", input, result)
    }

    // Dynamic pipeline
    fmt.Println()
    ops := []StringOp{trim, lower, removeSpaces}
    for _, input := range inputs {
        fmt.Printf("%-25q -> %q\n", input, pipeline(input, ops...))
    }

    // Compose operations
    fmt.Println()
    slug := pipeline("  Hello World Go!!  ",
        trim,
        lower,
        func(s string) string {
            var b strings.Builder
            prevSpace := false
            for _, r := range s {
                if unicode.IsLetter(r) || unicode.IsDigit(r) {
                    if prevSpace && b.Len() > 0 {
                        b.WriteRune('-')
                    }
                    b.WriteRune(r)
                    prevSpace = false
                } else if unicode.IsSpace(r) {
                    prevSpace = true
                }
            }
            return b.String()
        },
    )
    fmt.Printf("Slug: %q\n", slug)
}
```

---

## 6.12 ตัวอย่าง: Event System ด้วย Functions

```go
package main

import "fmt"

type EventType string

const (
    EventCreated EventType = "created"
    EventUpdated EventType = "updated"
    EventDeleted EventType = "deleted"
)

type Event struct {
    Type    EventType
    Payload interface{}
}

type Handler func(Event)

type EventBus struct {
    handlers map[EventType][]Handler
}

func NewEventBus() *EventBus {
    return &EventBus{
        handlers: make(map[EventType][]Handler),
    }
}

func (eb *EventBus) Subscribe(eventType EventType, handler Handler) {
    eb.handlers[eventType] = append(eb.handlers[eventType], handler)
}

func (eb *EventBus) Publish(event Event) {
    if handlers, ok := eb.handlers[event.Type]; ok {
        for _, h := range handlers {
            h(event)
        }
    }
}

func main() {
    bus := NewEventBus()

    // Subscribe handlers
    bus.Subscribe(EventCreated, func(e Event) {
        fmt.Printf("[Logger] สร้างข้อมูลใหม่: %v\n", e.Payload)
    })

    bus.Subscribe(EventCreated, func(e Event) {
        fmt.Printf("[Email] ส่งอีเมลแจ้งเตือน: %v\n", e.Payload)
    })

    bus.Subscribe(EventUpdated, func(e Event) {
        fmt.Printf("[Logger] อัปเดตข้อมูล: %v\n", e.Payload)
    })

    bus.Subscribe(EventDeleted, func(e Event) {
        fmt.Printf("[Logger] ลบข้อมูล: %v\n", e.Payload)
        fmt.Printf("[Audit] บันทึกการลบ: %v\n", e.Payload)
    })

    // Publish events
    bus.Publish(Event{EventCreated, map[string]string{"name": "สมชาย", "email": "somchai@example.com"}})
    bus.Publish(Event{EventUpdated, map[string]string{"id": "1", "field": "email"}})
    bus.Publish(Event{EventDeleted, map[string]string{"id": "1"}})
}
```

---

## Workshop: แบบฝึกหัด Part 6

### แบบฝึกหัดที่ 1: Calculator Library

สร้าง package ที่มี functions:
- `Add(a, b float64) float64`
- `Subtract(a, b float64) float64`
- `Multiply(a, b float64) float64`
- `Divide(a, b float64) (float64, error)`
- `Power(base, exp float64) float64`
- `Sqrt(n float64) (float64, error)`
- `Mean(nums ...float64) float64`

### แบบฝึกหัดที่ 2: Closure Counter

สร้าง `makeCounter(start, step int)` ที่:
- เริ่มนับจาก start
- เพิ่มทีละ step
- มี method: Next(), Reset(), Value()

### แบบฝึกหัดที่ 3: Retry Function

เขียน `retry(attempts int, fn func() error) error`:
- ลองทำ fn ซ้ำไม่เกิน attempts ครั้ง
- ถ้าสำเร็จ return nil
- ถ้าล้มเหลวทุกครั้ง return error ครั้งสุดท้าย

### แบบฝึกหัดที่ 4: Recursive Tree

สร้าง binary tree และเขียน:
- Insert(value int)
- InOrder() []int (left, root, right)
- Height() int

### เฉลยแบบฝึกหัดที่ 3

```go
package main

import (
    "fmt"
    "math/rand"
)

func retry(attempts int, fn func() error) error {
    var lastErr error
    for i := 0; i < attempts; i++ {
        lastErr = fn()
        if lastErr == nil {
            return nil
        }
        fmt.Printf("  attempt %d failed: %v\n", i+1, lastErr)
    }
    return fmt.Errorf("ล้มเหลวหลังจาก %d attempts: %w", attempts, lastErr)
}

func unreliableOperation() error {
    if rand.Float64() < 0.7 { // 70% fail
        return fmt.Errorf("connection timeout")
    }
    return nil
}

func main() {
    rand.Seed(42)
    err := retry(5, unreliableOperation)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Println("สำเร็จ!")
    }
}
```

---

## สรุป Part 6

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Declaration | `func name(params) returns { body }` |
| Multiple returns | `return a, b, err` |
| Named returns | `func f() (x int, err error)` |
| Variadic | `func f(nums ...int)`, ส่ง slice ด้วย `slice...` |
| Anonymous | `func(x int) int { return x*2 }` |
| Closure | Function ที่ capture ตัวแปรจาก outer scope |
| Higher-order | รับ function เป็น parameter หรือ return function |
| Recursion | Base case + recursive case |
| init | ทำงานก่อน main, LIFO order |
| defer | ทำงานเมื่อ function จบ, LIFO order |

**Key Takeaways:**
- Multiple return values เป็น idiomatic Go สำหรับ error handling
- Closures ต้อง capture by reference ระวัง loop bug
- `defer` ใช้สำหรับ resource cleanup เสมอ
- `init()` ใช้สำหรับ package initialization

## Resources

- [Go Spec - Function declarations](https://go.dev/ref/spec#Function_declarations)
- [Go by Example - Functions](https://gobyexample.com/functions)
- [Go by Example - Closures](https://gobyexample.com/closures)
- [Go by Example - Variadic Functions](https://gobyexample.com/variadic-functions)
- [Effective Go - Functions](https://go.dev/doc/effective_go#functions)
