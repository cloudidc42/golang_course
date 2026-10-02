# Part 5: Loops and Iteration

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ใช้ `for` loop ในทุกรูปแบบ (basic, while-style, infinite)
- ใช้ `for range` กับ slices, maps, strings, channels
- ควบคุม loop ด้วย break, continue, goto
- ใช้ labeled loops สำหรับ nested loops
- เขียน nested loops ที่มีประสิทธิภาพ
- รู้จัก common loop patterns ที่ใช้บ่อย

---

## 5.1 for Loop พื้นฐาน (C-style)

Go มีเพียง `for` keyword เดียวสำหรับทำ loop ทุกรูปแบบ

```go
package main

import "fmt"

func main() {
    // for แบบ 3-component (init; condition; post)
    for i := 0; i < 5; i++ {
        fmt.Printf("i = %d\n", i)
    }

    // Counting down
    for i := 10; i >= 0; i-- {
        fmt.Printf("%d ", i)
    }
    fmt.Println()

    // Step ทีละ 2
    for i := 0; i <= 10; i += 2 {
        fmt.Printf("%d ", i)
    }
    fmt.Println()

    // loop หลาย variables
    for i, j := 0, 10; i < j; i, j = i+1, j-1 {
        fmt.Printf("i=%d j=%d\n", i, j)
    }

    // Sum 1 ถึง 100
    sum := 0
    for i := 1; i <= 100; i++ {
        sum += i
    }
    fmt.Printf("Sum 1-100 = %d\n", sum)
}
```

---

## 5.2 while-style for Loop

```go
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    // for เป็น while
    n := 1
    for n < 100 {
        n *= 2
    }
    fmt.Printf("n = %d\n", n)

    // นับจำนวนหลัก
    num := 12345
    digits := 0
    for num > 0 {
        digits++
        num /= 10
    }
    fmt.Printf("12345 มี %d หลัก\n", digits)

    // Collatz conjecture
    n2 := 27
    steps := 0
    fmt.Printf("Collatz(%d): ", n2)
    for n2 != 1 {
        fmt.Printf("%d ", n2)
        if n2%2 == 0 {
            n2 /= 2
        } else {
            n2 = 3*n2 + 1
        }
        steps++
    }
    fmt.Printf("1 (%d steps)\n", steps)

    // Simulate coin flip until heads
    attempts := 0
    for {
        attempts++
        if rand.Intn(2) == 0 {
            fmt.Printf("ออกหัวหลังโยน %d ครั้ง\n", attempts)
            break
        }
    }
}
```

---

## 5.3 Infinite Loop

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Infinite loop (ต้องมี break)
    count := 0
    for {
        count++
        if count >= 5 {
            break
        }
        fmt.Printf("รอบที่ %d\n", count)
    }
    fmt.Println("จบ loop")

    // Server-like loop (จำลอง)
    requests := []string{"GET /home", "POST /login", "GET /about", "STOP"}

    i := 0
    for {
        if i >= len(requests) {
            break
        }
        req := requests[i]
        i++

        if req == "STOP" {
            fmt.Println("Server หยุดทำงาน")
            break
        }
        fmt.Printf("Processing: %s\n", req)
        time.Sleep(1 * time.Millisecond) // จำลอง processing
    }
}
```

---

## 5.4 for range กับ Slice

```go
package main

import "fmt"

func main() {
    fruits := []string{"apple", "banana", "cherry", "durian", "elderberry"}

    // range ได้ทั้ง index และ value
    fmt.Println("=== range slice ===")
    for i, fruit := range fruits {
        fmt.Printf("[%d] %s\n", i, fruit)
    }

    // ไม่ต้องการ index
    fmt.Println("\n=== เฉพาะ value ===")
    for _, fruit := range fruits {
        fmt.Println(fruit)
    }

    // ไม่ต้องการ value
    fmt.Println("\n=== เฉพาะ index ===")
    for i := range fruits {
        fmt.Printf("index: %d\n", i)
    }

    // คำนวณรวม
    numbers := []int{10, 20, 30, 40, 50}
    sum := 0
    for _, n := range numbers {
        sum += n
    }
    fmt.Printf("\nSum: %d\n", sum)

    // หา max/min
    scores := []float64{85.5, 92.0, 78.3, 95.5, 88.0}
    max, min := scores[0], scores[0]
    for _, s := range scores[1:] {
        if s > max {
            max = s
        }
        if s < min {
            min = s
        }
    }
    fmt.Printf("Max: %.1f, Min: %.1f\n", max, min)

    // Filter
    var evens []int
    for _, n := range []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10} {
        if n%2 == 0 {
            evens = append(evens, n)
        }
    }
    fmt.Printf("เลขคู่: %v\n", evens)
}
```

---

## 5.5 for range กับ Map

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    // Map: ไม่ guarantee ลำดับ
    population := map[string]int{
        "กรุงเทพ":    10500000,
        "เชียงใหม่":  131111,
        "ภูเก็ต":     76032,
        "ขอนแก่น":   106591,
        "ชลบุรี":     61680,
    }

    fmt.Println("=== range map (ลำดับสุ่ม) ===")
    for city, pop := range population {
        fmt.Printf("%-15s: %,d คน\n", city, pop)
    }

    // เรียงตาม key
    fmt.Println("\n=== เรียงตาม key ===")
    cities := make([]string, 0, len(population))
    for city := range population {
        cities = append(cities, city)
    }
    sort.Strings(cities)
    for _, city := range cities {
        fmt.Printf("%-15s: %d คน\n", city, population[city])
    }

    // หาเมืองใหญ่สุด
    var biggestCity string
    var maxPop int
    for city, pop := range population {
        if pop > maxPop {
            maxPop = pop
            biggestCity = city
        }
    }
    fmt.Printf("\nเมืองที่ใหญ่สุด: %s (%d คน)\n", biggestCity, maxPop)

    // นับประเภท
    words := []string{"go", "python", "go", "java", "go", "python"}
    wordCount := make(map[string]int)
    for _, w := range words {
        wordCount[w]++
    }
    fmt.Println("\nนับคำ:")
    for word, count := range wordCount {
        fmt.Printf("  %s: %d\n", word, count)
    }
}
```

---

## 5.6 for range กับ String

```go
package main

import "fmt"

func main() {
    // range string ได้ rune (Unicode code point)
    text := "Hello, สวัสดี!"

    fmt.Println("=== range string (rune by rune) ===")
    for i, r := range text {
        if r > 127 { // Non-ASCII
            fmt.Printf("  [byte %d] U+%04X = %c\n", i, r, r)
        } else {
            fmt.Printf("  [byte %d] %c\n", i, r)
        }
    }

    // นับตัวอักษร vs bytes
    thai := "ยินดีต้อนรับ"
    byteCount := len(thai)
    runeCount := 0
    for range thai {
        runeCount++
    }
    fmt.Printf("\n%q\n", thai)
    fmt.Printf("bytes: %d, runes (ตัวอักษร): %d\n", byteCount, runeCount)

    // Reverse string
    str := "Hello"
    runes := []rune(str)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    fmt.Printf("\nReverse %q = %q\n", str, string(runes))

    // Count vowels
    vowels := "aeiouAEIOU"
    count := 0
    for _, r := range "Hello, World!" {
        for _, v := range vowels {
            if r == v {
                count++
                break
            }
        }
    }
    fmt.Printf("Vowels in 'Hello, World!': %d\n", count)
}
```

---

## 5.7 break และ continue

```go
package main

import "fmt"

func main() {
    // break: หยุด loop ทันที
    fmt.Println("=== break ===")
    for i := 0; i < 10; i++ {
        if i == 5 {
            fmt.Printf("หยุดที่ i=%d\n", i)
            break
        }
        fmt.Printf("%d ", i)
    }
    fmt.Println()

    // continue: ข้ามรอบนี้
    fmt.Println("\n=== continue ===")
    for i := 0; i < 10; i++ {
        if i%2 == 0 {
            continue // ข้ามเลขคู่
        }
        fmt.Printf("%d ", i)
    }
    fmt.Println()

    // break ใน switch ภายใน for
    for i := 0; i < 5; i++ {
        switch i {
        case 2:
            fmt.Printf("พบ 2 ใน switch, ออก switch\n")
            // break นี้ออกจาก switch ไม่ใช่ for!
            break
        default:
            fmt.Printf("i=%d\n", i)
        }
    }

    // หาตัวเลขแรกที่หารด้วย 7 และ 11 ลงตัว
    for i := 1; i <= 1000; i++ {
        if i%7 == 0 && i%11 == 0 {
            fmt.Printf("\nจำนวนแรกที่หารด้วย 7 และ 11 ลงตัว: %d\n", i)
            break
        }
    }

    // Skip invalid data
    data := []int{5, -3, 8, -1, 0, 12, -7, 3}
    sum := 0
    skipped := 0
    for _, v := range data {
        if v <= 0 {
            skipped++
            continue
        }
        sum += v
    }
    fmt.Printf("Sum (skip <=0): %d, skipped: %d\n", sum, skipped)
}
```

---

## 5.8 Labeled Loops

```go
package main

import "fmt"

func main() {
    // Label ใช้สำหรับ break/continue nested loops
    fmt.Println("=== Labeled break ===")

outer:
    for i := 0; i < 5; i++ {
        for j := 0; j < 5; j++ {
            if i+j == 6 {
                fmt.Printf("พบที่ i=%d, j=%d -> break outer\n", i, j)
                break outer
            }
            fmt.Printf("(%d,%d) ", i, j)
        }
        fmt.Println()
    }
    fmt.Println("ออกจาก loop แล้ว")

    fmt.Println("\n=== Labeled continue ===")
loop:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                fmt.Printf("  j=1 ที่ i=%d -> continue loop\n", i)
                continue loop // ข้ามไปรอบถัดไปของ outer loop
            }
            fmt.Printf("  i=%d, j=%d\n", i, j)
        }
    }

    // ตัวอย่างจริง: ค้นหาใน 2D array
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    target := 5
    found := false
    var foundRow, foundCol int

search:
    for i, row := range matrix {
        for j, val := range row {
            if val == target {
                found = true
                foundRow, foundCol = i, j
                break search
            }
        }
    }

    if found {
        fmt.Printf("\nพบ %d ที่ [%d][%d]\n", target, foundRow, foundCol)
    }
}
```

---

## 5.9 goto (ใช้น้อยมาก)

```go
package main

import "fmt"

func main() {
    // goto ใช้งานได้ใน Go แต่ไม่แนะนำ
    i := 0
loop:
    if i < 5 {
        fmt.Printf("i = %d\n", i)
        i++
        goto loop
    }
    fmt.Println("จบ")

    // goto ห้ามข้าม variable declaration
    // goto end    // ถ้าทำแบบนี้จะ error ถ้ามี var ระหว่างกลาง
    // x := 10
    // end:
    // fmt.Println(x)
}
```

---

## 5.10 Nested Loops

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // Multiplication table
    fmt.Println("=== ตารางสูตรคูณ ===")
    for i := 1; i <= 9; i++ {
        for j := 1; j <= 9; j++ {
            fmt.Printf("%3d", i*j)
        }
        fmt.Println()
    }

    // Triangle patterns
    n := 5

    fmt.Println("\n=== สามเหลี่ยมซ้าย ===")
    for i := 1; i <= n; i++ {
        fmt.Println(strings.Repeat("* ", i))
    }

    fmt.Println("=== สามเหลี่ยมกลาง ===")
    for i := 1; i <= n; i++ {
        fmt.Printf("%s%s\n",
            strings.Repeat(" ", n-i),
            strings.Repeat("* ", i))
    }

    fmt.Println("=== เพชร ===")
    for i := 1; i <= n; i++ {
        fmt.Printf("%s%s\n",
            strings.Repeat(" ", n-i),
            strings.Repeat("* ", i))
    }
    for i := n - 1; i >= 1; i-- {
        fmt.Printf("%s%s\n",
            strings.Repeat(" ", n-i),
            strings.Repeat("* ", i))
    }

    // Prime numbers (Sieve of Eratosthenes)
    limit := 50
    isPrime := make([]bool, limit+1)
    for i := 2; i <= limit; i++ {
        isPrime[i] = true
    }
    for i := 2; i*i <= limit; i++ {
        if isPrime[i] {
            for j := i * i; j <= limit; j += i {
                isPrime[j] = false
            }
        }
    }
    fmt.Printf("\nจำนวนเฉพาะถึง %d: ", limit)
    for i := 2; i <= limit; i++ {
        if isPrime[i] {
            fmt.Printf("%d ", i)
        }
    }
    fmt.Println()
}
```

---

## 5.11 Common Loop Patterns

### Pattern 1: Accumulate/Fold

```go
package main

import "fmt"

func sum(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func product(nums []int) int {
    result := 1
    for _, n := range nums {
        result *= n
    }
    return result
}

func max(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    m := nums[0]
    for _, n := range nums[1:] {
        if n > m {
            m = n
        }
    }
    return m
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6, 5, 3}
    fmt.Printf("nums: %v\n", nums)
    fmt.Printf("sum: %d\n", sum(nums))
    fmt.Printf("product: %d\n", product(nums))
    fmt.Printf("max: %d\n", max(nums))
}
```

### Pattern 2: Filter/Select

```go
package main

import "fmt"

func filter(nums []int, predicate func(int) bool) []int {
    var result []int
    for _, n := range nums {
        if predicate(n) {
            result = append(result, n)
        }
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    evens := filter(nums, func(n int) bool { return n%2 == 0 })
    odds := filter(nums, func(n int) bool { return n%2 != 0 })
    greaterThan5 := filter(nums, func(n int) bool { return n > 5 })

    fmt.Printf("ทั้งหมด: %v\n", nums)
    fmt.Printf("เลขคู่:  %v\n", evens)
    fmt.Printf("เลขคี่:  %v\n", odds)
    fmt.Printf("> 5:     %v\n", greaterThan5)
}
```

### Pattern 3: Map/Transform

```go
package main

import (
    "fmt"
    "strings"
)

func mapInts(nums []int, transform func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = transform(n)
    }
    return result
}

func mapStrings(strs []string, transform func(string) string) []string {
    result := make([]string, len(strs))
    for i, s := range strs {
        result[i] = transform(s)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    squared := mapInts(nums, func(n int) int { return n * n })
    doubled := mapInts(nums, func(n int) int { return n * 2 })

    fmt.Printf("original: %v\n", nums)
    fmt.Printf("squared:  %v\n", squared)
    fmt.Printf("doubled:  %v\n", doubled)

    words := []string{"hello", "world", "golang"}
    upper := mapStrings(words, strings.ToUpper)
    fmt.Printf("upper: %v\n", upper)
}
```

### Pattern 4: Search/Find

```go
package main

import "fmt"

func findFirst(nums []int, predicate func(int) bool) (int, bool) {
    for _, n := range nums {
        if predicate(n) {
            return n, true
        }
    }
    return 0, false
}

func contains(strs []string, target string) bool {
    for _, s := range strs {
        if s == target {
            return true
        }
    }
    return false
}

func indexOf(nums []int, target int) int {
    for i, n := range nums {
        if n == target {
            return i
        }
    }
    return -1
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}

    if first, ok := findFirst(nums, func(n int) bool { return n > 4 }); ok {
        fmt.Printf("หำค่าแรกที่ > 4: %d\n", first)
    }

    languages := []string{"Go", "Python", "Java", "C++"}
    fmt.Printf("มี Go: %v\n", contains(languages, "Go"))
    fmt.Printf("มี Rust: %v\n", contains(languages, "Rust"))

    idx := indexOf(nums, 9)
    fmt.Printf("index ของ 9: %d\n", idx)
}
```

### Pattern 5: Chunking/Batch

```go
package main

import "fmt"

func chunk(slice []int, size int) [][]int {
    var chunks [][]int
    for size < len(slice) {
        slice, chunks = slice[size:], append(chunks, slice[:size])
    }
    if len(slice) > 0 {
        chunks = append(chunks, slice)
    }
    return chunks
}

func main() {
    data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11}
    batches := chunk(data, 3)

    fmt.Printf("data: %v\n", data)
    fmt.Println("batches:")
    for i, batch := range batches {
        fmt.Printf("  batch %d: %v\n", i+1, batch)
    }
}
```

---

## 5.12 ตัวอย่างรวม: โปรแกรม FizzBuzz

```go
package main

import (
    "fmt"
    "strings"
)

func fizzBuzz(n int) string {
    var result strings.Builder
    if n%3 == 0 {
        result.WriteString("Fizz")
    }
    if n%5 == 0 {
        result.WriteString("Buzz")
    }
    if result.Len() == 0 {
        result.WriteString(fmt.Sprintf("%d", n))
    }
    return result.String()
}

func main() {
    for i := 1; i <= 30; i++ {
        fmt.Printf("%3d: %s\n", i, fizzBuzz(i))
    }
}
```

---

## 5.13 ตัวอย่าง: Matrix Operations

```go
package main

import "fmt"

func makeMatrix(rows, cols int) [][]int {
    m := make([][]int, rows)
    for i := range m {
        m[i] = make([]int, cols)
    }
    return m
}

func printMatrix(m [][]int) {
    for _, row := range m {
        for j, val := range row {
            if j > 0 {
                fmt.Print(" ")
            }
            fmt.Printf("%3d", val)
        }
        fmt.Println()
    }
}

func multiplyMatrix(a, b [][]int) [][]int {
    n := len(a)
    m := len(b[0])
    p := len(b)
    result := makeMatrix(n, m)
    for i := 0; i < n; i++ {
        for j := 0; j < m; j++ {
            for k := 0; k < p; k++ {
                result[i][j] += a[i][k] * b[k][j]
            }
        }
    }
    return result
}

func transposeMatrix(m [][]int) [][]int {
    rows := len(m)
    cols := len(m[0])
    t := makeMatrix(cols, rows)
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            t[j][i] = m[i][j]
        }
    }
    return t
}

func main() {
    // สร้าง identity matrix
    n := 4
    identity := makeMatrix(n, n)
    for i := range identity {
        identity[i][i] = 1
    }
    fmt.Println("Identity Matrix:")
    printMatrix(identity)

    // Matrix fill
    m := makeMatrix(3, 4)
    val := 1
    for i := range m {
        for j := range m[i] {
            m[i][j] = val
            val++
        }
    }
    fmt.Println("\nMatrix 3x4:")
    printMatrix(m)

    // Transpose
    fmt.Println("\nTranspose:")
    printMatrix(transposeMatrix(m))

    // Matrix multiply 2x3 * 3x2
    a := [][]int{{1, 2, 3}, {4, 5, 6}}
    b := [][]int{{7, 8}, {9, 10}, {11, 12}}

    fmt.Println("\na * b =")
    printMatrix(multiplyMatrix(a, b))
}
```

---

## Workshop: แบบฝึกหัด Part 5

### แบบฝึกหัดที่ 1: Fibonacci Sequence

เขียนโปรแกรมแสดง Fibonacci sequence 20 ตัวแรก:
0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...

### แบบฝึกหัดที่ 2: Number Guessing Stats

จำลองการเดาตัวเลข 1-100:
- ใช้ loop เดาตัวเลขจาก 50 ลงไป/ขึ้นไป (binary search style)
- นับจำนวนครั้งที่เดา
- แสดงประวัติการเดา

### แบบฝึกหัดที่ 3: Word Frequency

รับ slice ของ words แล้ว:
1. นับความถี่ของแต่ละคำ
2. หาคำที่ปรากฏบ่อยสุด
3. เรียงลำดับตามความถี่ (มากไปน้อย)

### แบบฝึกหัดที่ 4: Pascal's Triangle

แสดง Pascal's Triangle 8 แถว:
```
        1
       1 1
      1 2 1
     1 3 3 1
    1 4 6 4 1
```

### เฉลยแบบฝึกหัดที่ 1

```go
package main

import "fmt"

func main() {
    n := 20
    fibs := make([]int, n)
    fibs[0], fibs[1] = 0, 1
    for i := 2; i < n; i++ {
        fibs[i] = fibs[i-1] + fibs[i-2]
    }
    fmt.Printf("Fibonacci %d ตัวแรก:\n", n)
    for i, f := range fibs {
        fmt.Printf("F(%2d) = %d\n", i, f)
    }
}
```

### เฉลยแบบฝึกหัดที่ 4

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    rows := 8
    triangle := make([][]int, rows)
    for i := range triangle {
        triangle[i] = make([]int, i+1)
        triangle[i][0] = 1
        triangle[i][i] = 1
        for j := 1; j < i; j++ {
            triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j]
        }
    }

    for i, row := range triangle {
        // padding
        fmt.Print(strings.Repeat("  ", rows-i-1))
        for _, val := range row {
            fmt.Printf("%4d", val)
        }
        fmt.Println()
    }
}
```

---

## สรุป Part 5

| รูปแบบ | Syntax | การใช้งาน |
|--------|--------|----------|
| C-style | `for i := 0; i < n; i++` | นับจำนวนรอบที่รู้ล่วงหน้า |
| while-style | `for condition` | ไม่รู้จำนวนรอบล่วงหน้า |
| infinite | `for { ... break }` | server loop, event loop |
| range slice | `for i, v := range slice` | iterate elements |
| range map | `for k, v := range map` | iterate key-value |
| range string | `for i, r := range str` | iterate runes |
| labeled | `outer: for ... break outer` | nested loop control |

**Key Takeaways:**
- Go มีแค่ `for` keyword เดียว (ไม่มี while, do-while, foreach)
- `for range` ของ map ไม่ guarantee ลำดับ
- `break` และ `continue` ควบคุม loop ที่ใกล้ที่สุด
- Label ใช้สำหรับควบคุม outer loop จาก inner loop

## Resources

- [Go Spec - For statements](https://go.dev/ref/spec#For_statements)
- [Go by Example - For](https://gobyexample.com/for)
- [Go by Example - Range](https://gobyexample.com/range)
- [Effective Go - For](https://go.dev/doc/effective_go#for)
