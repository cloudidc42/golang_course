# Part 7: Arrays and Slices

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ประกาศและใช้งาน arrays รวมถึง multi-dimensional arrays
- เข้าใจความแตกต่างระหว่าง array และ slice
- สร้างและจัดการ slices ด้วย make, append, copy
- เข้าใจ slice header (pointer, len, cap) และ backing array
- ใช้ slice tricks สำหรับ delete, insert, filter
- จัดการ 2D slices
- รู้จัก common patterns ที่ใช้บ่อย

---

## 7.1 Arrays: Declaration และ Initialization

Array ใน Go มีขนาดคงที่ กำหนดตอน compile time

```go
package main

import "fmt"

func main() {
    // ประกาศ array
    var arr1 [5]int
    fmt.Printf("zero array: %v\n", arr1)

    // ประกาศพร้อมกำหนดค่า
    arr2 := [5]int{10, 20, 30, 40, 50}
    fmt.Printf("initialized: %v\n", arr2)

    // ให้ Go นับขนาดให้
    arr3 := [...]int{1, 2, 3, 4, 5, 6}
    fmt.Printf("auto-size: %v (len=%d)\n", arr3, len(arr3))

    // กำหนดค่า specific index
    arr4 := [5]int{0: 100, 2: 300, 4: 500}
    fmt.Printf("sparse: %v\n", arr4)

    // String array
    names := [3]string{"Go", "Python", "Java"}
    fmt.Printf("names: %v\n", names)

    // Bool array
    flags := [4]bool{true, false, true, true}
    fmt.Printf("flags: %v\n", flags)

    // Access elements
    fmt.Printf("arr2[0]=%d, arr2[4]=%d\n", arr2[0], arr2[4])

    // แก้ไขค่า
    arr2[2] = 999
    fmt.Printf("modified: %v\n", arr2)

    // Length
    fmt.Printf("len(arr2) = %d\n", len(arr2))

    // Array เป็น value type (copy เมื่อส่งให้ function หรือ assign)
    original := [3]int{1, 2, 3}
    copied := original
    copied[0] = 100
    fmt.Printf("original: %v, copied: %v\n", original, copied) // ไม่กระทบกัน
}
```

---

## 7.2 Array Iteration

```go
package main

import "fmt"

func main() {
    scores := [5]int{85, 92, 78, 95, 88}

    // for loop
    fmt.Println("=== for loop ===")
    for i := 0; i < len(scores); i++ {
        fmt.Printf("  scores[%d] = %d\n", i, scores[i])
    }

    // for range
    fmt.Println("=== for range ===")
    for i, s := range scores {
        fmt.Printf("  [%d] = %d\n", i, s)
    }

    // คำนวณค่าเฉลี่ย
    sum := 0
    for _, s := range scores {
        sum += s
    }
    avg := float64(sum) / float64(len(scores))
    fmt.Printf("Average: %.2f\n", avg)

    // หา max/min
    min, max := scores[0], scores[0]
    for _, s := range scores[1:] {
        if s < min {
            min = s
        }
        if s > max {
            max = s
        }
    }
    fmt.Printf("Min: %d, Max: %d\n", min, max)
}
```

---

## 7.3 Multi-dimensional Arrays

```go
package main

import "fmt"

func main() {
    // 2D array
    var matrix [3][4]int
    fmt.Printf("zero 2D array: %v\n", matrix)

    // กำหนดค่า
    grid := [3][3]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }

    fmt.Println("3x3 grid:")
    for i, row := range grid {
        for j, val := range row {
            fmt.Printf("%3d", val)
            _ = i
            _ = j
        }
        fmt.Println()
    }

    // เข้าถึง element
    fmt.Printf("grid[1][2] = %d\n", grid[1][2])

    // ตัวอย่าง: Game Board
    board := [8][8]rune{}
    // วาง pieces
    for i := range board {
        for j := range board[i] {
            if (i+j)%2 == 0 {
                board[i][j] = '█'
            } else {
                board[i][j] = '░'
            }
        }
    }

    fmt.Println("Chess board:")
    for _, row := range board {
        for _, cell := range row {
            fmt.Printf("%c", cell)
        }
        fmt.Println()
    }

    // 3D array
    cube := [2][3][4]int{}
    for i := range cube {
        for j := range cube[i] {
            for k := range cube[i][j] {
                cube[i][j][k] = i*12 + j*4 + k
            }
        }
    }
    fmt.Printf("cube[1][2][3] = %d\n", cube[1][2][3])
}
```

---

## 7.4 Slices: พื้นฐาน

Slice เป็น dynamic array - ขนาดสามารถเปลี่ยนได้

```go
package main

import "fmt"

func main() {
    // ประกาศ slice (nil slice)
    var s []int
    fmt.Printf("nil slice: %v, len=%d, cap=%d, nil=%v\n", s, len(s), cap(s), s == nil)

    // สร้างด้วย literal
    s2 := []int{1, 2, 3, 4, 5}
    fmt.Printf("literal: %v, len=%d, cap=%d\n", s2, len(s2), cap(s2))

    // สร้างด้วย make(type, len, cap)
    s3 := make([]int, 5)
    fmt.Printf("make(5): %v, len=%d, cap=%d\n", s3, len(s3), cap(s3))

    s4 := make([]int, 3, 10)
    fmt.Printf("make(3,10): %v, len=%d, cap=%d\n", s4, len(s4), cap(s4))

    // Slice ของ array
    arr := [5]int{10, 20, 30, 40, 50}
    sl := arr[1:4] // [20, 30, 40]
    fmt.Printf("arr[1:4]: %v, len=%d, cap=%d\n", sl, len(sl), cap(sl))

    // Slice expressions
    a := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
    fmt.Printf("a[2:5]  = %v\n", a[2:5])   // [2, 3, 4]
    fmt.Printf("a[:3]   = %v\n", a[:3])    // [0, 1, 2]
    fmt.Printf("a[7:]   = %v\n", a[7:])    // [7, 8, 9]
    fmt.Printf("a[:]    = %v\n", a[:])     // ทั้งหมด
    fmt.Printf("a[2:5:7]= %v (cap=%d)\n", a[2:5:7], cap(a[2:5:7])) // full slice expression

    // Slice เป็น reference type (ชี้ไป backing array เดียวกัน)
    original := []int{1, 2, 3, 4, 5}
    ref := original[1:4]
    ref[0] = 100
    fmt.Printf("original: %v (ถูกแก้ไข!)\n", original)
}
```

---

## 7.5 append และ copy

```go
package main

import "fmt"

func main() {
    // append
    var s []int
    fmt.Printf("start: %v (len=%d, cap=%d)\n", s, len(s), cap(s))

    for i := 1; i <= 10; i++ {
        s = append(s, i*10)
        fmt.Printf("append %2d: len=%d, cap=%d\n", i*10, len(s), cap(s))
    }
    fmt.Printf("final: %v\n", s)

    // append หลายค่าพร้อมกัน
    s2 := []int{1, 2, 3}
    s2 = append(s2, 4, 5, 6)
    fmt.Printf("append multiple: %v\n", s2)

    // append slice เข้า slice (ใช้ ...)
    s3 := []int{7, 8, 9}
    s2 = append(s2, s3...)
    fmt.Printf("append slice: %v\n", s2)

    // copy
    src := []int{1, 2, 3, 4, 5}
    dst := make([]int, len(src))
    n := copy(dst, src)
    fmt.Printf("copy %d elements: %v\n", n, dst)

    // copy บางส่วน
    dst2 := make([]int, 3)
    copy(dst2, src)
    fmt.Printf("partial copy: %v\n", dst2)

    // แก้ไข dst ไม่กระทบ src
    dst[0] = 999
    fmt.Printf("src: %v\n", src)
    fmt.Printf("dst: %v\n", dst)
}
```

---

## 7.6 Slice Capacity และ Growth

```go
package main

import "fmt"

func main() {
    // ดู capacity growth pattern
    var s []int
    prevCap := 0
    for i := 0; i < 32; i++ {
        s = append(s, i)
        if cap(s) != prevCap {
            fmt.Printf("len=%2d, cap=%2d (grown from %d)\n", len(s), cap(s), prevCap)
            prevCap = cap(s)
        }
    }

    // Pre-allocate เพื่อประสิทธิภาพ
    // ถ้ารู้ขนาดล่วงหน้า ควร make ด้วย capacity ที่เหมาะสม
    n := 1000
    s2 := make([]int, 0, n) // len=0, cap=n
    for i := 0; i < n; i++ {
        s2 = append(s2, i)
    }
    fmt.Printf("pre-allocated: len=%d, cap=%d\n", len(s2), cap(s2))

    // ระวัง: slice header vs backing array
    a := make([]int, 5, 10)
    b := a[2:4] // share backing array
    b[0] = 100
    fmt.Printf("a: %v\n", a) // a[2] เปลี่ยนเป็น 100
    fmt.Printf("b: %v\n", b)

    // append ที่ใช้ capacity ที่เหลืออยู่ยังแชร์ array
    c := append(b, 999)
    fmt.Printf("a: %v\n", a) // a[4] เปลี่ยนเป็น 999!
    fmt.Printf("b: %v\n", b)
    fmt.Printf("c: %v\n", c)

    // ถ้า append เกิน capacity จะสร้าง backing array ใหม่
    d := make([]int, 3, 3)
    d[0], d[1], d[2] = 1, 2, 3
    e := append(d, 4) // เกิน cap ต้องสร้างใหม่
    e[0] = 100        // ไม่กระทบ d
    fmt.Printf("d: %v\n", d)
    fmt.Printf("e: %v\n", e)
}
```

---

## 7.7 Slice Tricks

```go
package main

import "fmt"

// Delete element at index i
func deleteAt(s []int, i int) []int {
    return append(s[:i], s[i+1:]...)
}

// Delete โดยไม่รักษาลำดับ (เร็วกว่า)
func deleteAtFast(s []int, i int) []int {
    s[i] = s[len(s)-1]
    return s[:len(s)-1]
}

// Insert element at index i
func insertAt(s []int, i, val int) []int {
    s = append(s, 0)
    copy(s[i+1:], s[i:])
    s[i] = val
    return s
}

// Filter ด้วย in-place
func filterInPlace(s []int, keep func(int) bool) []int {
    n := 0
    for _, x := range s {
        if keep(x) {
            s[n] = x
            n++
        }
    }
    return s[:n]
}

// Reverse slice
func reverse(s []int) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}

// Dedup: ลบค่าซ้ำ (assumes sorted)
func dedup(s []int) []int {
    if len(s) == 0 {
        return s
    }
    n := 1
    for i := 1; i < len(s); i++ {
        if s[i] != s[i-1] {
            s[n] = s[i]
            n++
        }
    }
    return s[:n]
}

// Flatten 2D slice
func flatten(ss [][]int) []int {
    var result []int
    for _, s := range ss {
        result = append(result, s...)
    }
    return result
}

func main() {
    // Delete
    s := []int{1, 2, 3, 4, 5}
    fmt.Printf("original: %v\n", s)
    s = deleteAt(s, 2)
    fmt.Printf("delete at 2: %v\n", s)

    s2 := []int{1, 2, 3, 4, 5}
    s2 = deleteAtFast(s2, 2)
    fmt.Printf("delete fast at 2: %v (ลำดับเปลี่ยน)\n", s2)

    // Insert
    s3 := []int{1, 2, 4, 5}
    s3 = insertAt(s3, 2, 3)
    fmt.Printf("insert 3 at 2: %v\n", s3)

    // Filter
    s4 := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    s4 = filterInPlace(s4, func(n int) bool { return n%2 == 0 })
    fmt.Printf("filter even: %v\n", s4)

    // Reverse
    s5 := []int{1, 2, 3, 4, 5}
    reverse(s5)
    fmt.Printf("reversed: %v\n", s5)

    // Dedup
    s6 := []int{1, 1, 2, 3, 3, 3, 4, 5, 5}
    s6 = dedup(s6)
    fmt.Printf("dedup: %v\n", s6)

    // Flatten
    ss := [][]int{{1, 2, 3}, {4, 5}, {6, 7, 8, 9}}
    flat := flatten(ss)
    fmt.Printf("flatten: %v\n", flat)
}
```

---

## 7.8 2D Slices

```go
package main

import "fmt"

// สร้าง 2D slice
func make2D(rows, cols int) [][]int {
    grid := make([][]int, rows)
    for i := range grid {
        grid[i] = make([]int, cols)
    }
    return grid
}

// สร้าง 2D slice จาก flat slice
func reshape(flat []int, rows, cols int) [][]int {
    if len(flat) != rows*cols {
        return nil
    }
    grid := make([][]int, rows)
    for i := range grid {
        grid[i] = flat[i*cols : (i+1)*cols]
    }
    return grid
}

func printGrid(grid [][]int) {
    for _, row := range grid {
        for j, val := range row {
            if j > 0 {
                fmt.Print(" ")
            }
            fmt.Printf("%3d", val)
        }
        fmt.Println()
    }
}

// Matrix transpose
func transpose(m [][]int) [][]int {
    if len(m) == 0 {
        return nil
    }
    rows, cols := len(m), len(m[0])
    t := make([][]int, cols)
    for i := range t {
        t[i] = make([]int, rows)
    }
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            t[j][i] = m[i][j]
        }
    }
    return t
}

func main() {
    // สร้างและกำหนดค่า
    grid := make2D(3, 4)
    for i := range grid {
        for j := range grid[i] {
            grid[i][j] = i*4 + j + 1
        }
    }
    fmt.Println("3x4 grid:")
    printGrid(grid)

    // Reshape
    flat := []int{1, 2, 3, 4, 5, 6}
    m := reshape(flat, 2, 3)
    fmt.Println("\n2x3 matrix:")
    printGrid(m)

    // Transpose
    fmt.Println("Transpose (3x2):")
    printGrid(transpose(m))

    // Jagged array (ขนาดแต่ละแถวไม่เท่ากัน)
    jagged := make([][]int, 5)
    for i := range jagged {
        jagged[i] = make([]int, i+1)
        for j := range jagged[i] {
            jagged[i][j] = j + 1
        }
    }
    fmt.Println("\nJagged array:")
    for _, row := range jagged {
        fmt.Println(row)
    }
}
```

---

## 7.9 Common Slice Patterns

### Pattern: Stack (LIFO)

```go
package main

import (
    "errors"
    "fmt"
)

type Stack struct {
    data []int
}

func (s *Stack) Push(v int) {
    s.data = append(s.data, v)
}

func (s *Stack) Pop() (int, error) {
    if s.IsEmpty() {
        return 0, errors.New("stack ว่างเปล่า")
    }
    n := len(s.data) - 1
    v := s.data[n]
    s.data = s.data[:n]
    return v, nil
}

func (s *Stack) Peek() (int, error) {
    if s.IsEmpty() {
        return 0, errors.New("stack ว่างเปล่า")
    }
    return s.data[len(s.data)-1], nil
}

func (s *Stack) IsEmpty() bool { return len(s.data) == 0 }
func (s *Stack) Size() int     { return len(s.data) }

func main() {
    stack := &Stack{}
    for i := 1; i <= 5; i++ {
        stack.Push(i * 10)
        fmt.Printf("Push %d: size=%d\n", i*10, stack.Size())
    }

    for !stack.IsEmpty() {
        v, _ := stack.Pop()
        fmt.Printf("Pop: %d\n", v)
    }

    _, err := stack.Pop()
    fmt.Printf("Pop จาก stack ว่าง: %v\n", err)
}
```

### Pattern: Queue (FIFO)

```go
package main

import (
    "errors"
    "fmt"
)

type Queue struct {
    data []int
}

func (q *Queue) Enqueue(v int) {
    q.data = append(q.data, v)
}

func (q *Queue) Dequeue() (int, error) {
    if q.IsEmpty() {
        return 0, errors.New("queue ว่างเปล่า")
    }
    v := q.data[0]
    q.data = q.data[1:]
    return v, nil
}

func (q *Queue) IsEmpty() bool { return len(q.data) == 0 }
func (q *Queue) Size() int     { return len(q.data) }

func main() {
    q := &Queue{}
    q.Enqueue(1)
    q.Enqueue(2)
    q.Enqueue(3)

    for !q.IsEmpty() {
        v, _ := q.Dequeue()
        fmt.Printf("Dequeue: %d\n", v)
    }
}
```

### Pattern: Sliding Window

```go
package main

import "fmt"

// Max sum ของ subarray ขนาด k
func maxSumWindow(nums []int, k int) int {
    if len(nums) < k {
        return 0
    }
    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += nums[i]
    }
    maxSum := windowSum
    for i := k; i < len(nums); i++ {
        windowSum += nums[i] - nums[i-k]
        if windowSum > maxSum {
            maxSum = windowSum
        }
    }
    return maxSum
}

// Moving average
func movingAverage(data []float64, window int) []float64 {
    if len(data) < window {
        return nil
    }
    result := make([]float64, len(data)-window+1)
    sum := 0.0
    for i := 0; i < window; i++ {
        sum += data[i]
    }
    result[0] = sum / float64(window)
    for i := window; i < len(data); i++ {
        sum += data[i] - data[i-window]
        result[i-window+1] = sum / float64(window)
    }
    return result
}

func main() {
    nums := []int{1, 4, 2, 10, 23, 3, 1, 0, 20}
    fmt.Printf("Max sum (k=4): %d\n", maxSumWindow(nums, 4))

    prices := []float64{10, 12, 11, 13, 15, 14, 16, 18}
    ma := movingAverage(prices, 3)
    fmt.Printf("Prices: %v\n", prices)
    fmt.Printf("MA(3):  %v\n", ma)
}
```

---

## 7.10 ตัวอย่างรวม: Sorting Algorithms

```go
package main

import "fmt"

// Bubble Sort
func bubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
}

// Selection Sort
func selectionSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if arr[j] < arr[minIdx] {
                minIdx = j
            }
        }
        arr[i], arr[minIdx] = arr[minIdx], arr[i]
    }
}

// Merge Sort
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }
    mid := len(arr) / 2
    left := mergeSort(arr[:mid])
    right := mergeSort(arr[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}

// Quick Sort
func quickSort(arr []int, low, high int) {
    if low < high {
        pivot := partition(arr, low, high)
        quickSort(arr, low, pivot-1)
        quickSort(arr, pivot+1, high)
    }
}

func partition(arr []int, low, high int) int {
    pivot := arr[high]
    i := low - 1
    for j := low; j < high; j++ {
        if arr[j] <= pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[high] = arr[high], arr[i+1]
    return i + 1
}

func main() {
    original := []int{64, 34, 25, 12, 22, 11, 90}

    // Bubble Sort
    arr1 := make([]int, len(original))
    copy(arr1, original)
    bubbleSort(arr1)
    fmt.Printf("Bubble Sort:    %v\n", arr1)

    // Selection Sort
    arr2 := make([]int, len(original))
    copy(arr2, original)
    selectionSort(arr2)
    fmt.Printf("Selection Sort: %v\n", arr2)

    // Merge Sort
    arr3 := make([]int, len(original))
    copy(arr3, original)
    sorted := mergeSort(arr3)
    fmt.Printf("Merge Sort:     %v\n", sorted)

    // Quick Sort
    arr4 := make([]int, len(original))
    copy(arr4, original)
    quickSort(arr4, 0, len(arr4)-1)
    fmt.Printf("Quick Sort:     %v\n", arr4)
}
```

---

## Workshop: แบบฝึกหัด Part 7

### แบบฝึกหัดที่ 1: Array Statistics

สร้าง function รับ `[]float64` และ return:
- Mean (ค่าเฉลี่ย)
- Median (ค่ากลาง)
- Mode (ค่าที่พบบ่อยสุด)
- Variance (ความแปรปรวน)
- Standard Deviation

### แบบฝึกหัดที่ 2: Two Sum Problem

เขียน function `twoSum(nums []int, target int) (int, int)`:
- หาคู่ index ที่ค่ารวมกัน = target
- Return (-1, -1) ถ้าไม่พบ
- ต้องมีประสิทธิภาพ O(n)

### แบบฝึกหัดที่ 3: Matrix Operations

สร้าง functions สำหรับ:
- `AddMatrix(a, b [][]int) [][]int`
- `MultiplyMatrix(a, b [][]int) [][]int`
- `RotateMatrix90(m [][]int) [][]int` (หมุน 90 องศาตามเข็มนาฬิกา)

### แบบฝึกหัดที่ 4: Sliding Window Maximum

เขียน function `maxSlidingWindow(nums []int, k int) []int`:
- Return slice ของค่า max ในแต่ละ window ขนาด k
- เช่น nums=[1,3,-1,-3,5,3,6,7], k=3 -> [3,3,5,5,6,7]

### เฉลยแบบฝึกหัดที่ 2

```go
package main

import "fmt"

func twoSum(nums []int, target int) (int, int) {
    seen := make(map[int]int) // value -> index
    for i, n := range nums {
        complement := target - n
        if j, ok := seen[complement]; ok {
            return j, i
        }
        seen[n] = i
    }
    return -1, -1
}

func main() {
    tests := []struct {
        nums   []int
        target int
    }{
        {[]int{2, 7, 11, 15}, 9},
        {[]int{3, 2, 4}, 6},
        {[]int{3, 3}, 6},
        {[]int{1, 2, 3}, 10},
    }

    for _, t := range tests {
        i, j := twoSum(t.nums, t.target)
        if i == -1 {
            fmt.Printf("nums=%v, target=%d: ไม่พบ\n", t.nums, t.target)
        } else {
            fmt.Printf("nums=%v, target=%d: [%d]+[%d]=%d\n",
                t.nums, t.target, i, j, t.nums[i]+t.nums[j])
        }
    }
}
```

---

## สรุป Part 7

| หัวข้อ | Array | Slice |
|--------|-------|-------|
| ขนาด | คงที่ (compile time) | ยืดหยุ่น (runtime) |
| Type | `[5]int` | `[]int` |
| สร้าง | `[n]T{...}` หรือ `[...]T{...}` | `make([]T, len, cap)` หรือ `[]T{...}` |
| Pass to function | Copy (value type) | Reference (แชร์ backing array) |
| Zero value | `[n]T{}` (all zeros) | `nil` |

**Slice Operations:**

| Operation | Code |
|-----------|------|
| Append | `s = append(s, v)` |
| Copy | `copy(dst, src)` |
| Delete at i | `append(s[:i], s[i+1:]...)` |
| Insert at i | `append(s[:i], append([]T{v}, s[i:]...)...)` |
| Reverse | swap from both ends |
| Slice of slice | `s[low:high]` หรือ `s[low:high:max]` |

**Key Takeaways:**
- Array เป็น value type, slice เป็น reference type
- Slice = pointer + len + cap (ชี้ไป backing array)
- `append` อาจสร้าง backing array ใหม่ถ้า cap ไม่พอ
- Pre-allocate slice ด้วย `make([]T, 0, expectedSize)` เพื่อประสิทธิภาพ

## Resources

- [Go Spec - Slice types](https://go.dev/ref/spec#Slice_types)
- [Go Blog - Slices: usage and internals](https://go.dev/blog/slices-intro)
- [Go Blog - Arrays, slices, and strings](https://go.dev/blog/slices)
- [Go by Example - Slices](https://gobyexample.com/slices)
- [Go by Example - Arrays](https://gobyexample.com/arrays)
