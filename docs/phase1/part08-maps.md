# Part 8: Maps

## เป้าหมายการเรียนรู้

หลังจากศึกษา Part นี้จบแล้ว คุณจะสามารถ:
- ประกาศและสร้าง maps ด้วยวิธีต่างๆ
- ทำ CRUD operations บน maps ได้อย่างถูกต้อง
- ตรวจสอบการมีอยู่ของ key ด้วย two-value assignment
- วนซ้ำ map ด้วย for range
- ใช้ maps ของ slices และ structs
- ใช้ map patterns ที่พบบ่อย (counting, grouping, caching)
- ใช้ sync.Map สำหรับ concurrent access

---

## 8.1 Map Declaration และ Initialization

Map คือ data structure แบบ key-value (hash table)

```go
package main

import "fmt"

func main() {
    // nil map (ไม่สามารถ write ได้!)
    var m1 map[string]int
    fmt.Printf("nil map: %v, nil=%v\n", m1, m1 == nil)

    // สร้างด้วย make
    m2 := make(map[string]int)
    m2["one"] = 1
    m2["two"] = 2
    fmt.Printf("make: %v\n", m2)

    // Map literal
    m3 := map[string]int{
        "apple":  5,
        "banana": 3,
        "cherry": 8,
    }
    fmt.Printf("literal: %v\n", m3)

    // Map กับ type ต่างๆ
    scores := map[string]float64{
        "สมชาย":  85.5,
        "สมหญิง": 92.0,
        "สมศักดิ์": 78.3,
    }
    fmt.Printf("scores: %v\n", scores)

    // Map with struct values
    type Point struct{ X, Y int }
    points := map[string]Point{
        "origin": {0, 0},
        "a":      {1, 2},
        "b":      {3, 4},
    }
    fmt.Printf("points: %v\n", points)

    // Map กับ int key
    romanNumerals := map[int]string{
        1: "I", 4: "IV", 5: "V",
        9: "IX", 10: "X", 40: "XL",
        50: "L", 90: "XC", 100: "C",
    }
    fmt.Printf("roman: %v\n", romanNumerals)
}
```

---

## 8.2 CRUD Operations

```go
package main

import "fmt"

func main() {
    inventory := make(map[string]int)

    // CREATE: เพิ่ม/อัปเดตค่า
    inventory["apple"] = 100
    inventory["banana"] = 50
    inventory["cherry"] = 75
    inventory["durian"] = 20
    fmt.Printf("After create: %v\n", inventory)

    // READ: อ่านค่า
    appleCount := inventory["apple"]
    fmt.Printf("Apple count: %d\n", appleCount)

    // อ่าน key ที่ไม่มี -> zero value (ไม่ panic)
    mangoCount := inventory["mango"]
    fmt.Printf("Mango count: %d (zero value)\n", mangoCount)

    // UPDATE: แก้ไขค่า (ใช้ syntax เดียวกับ create)
    inventory["apple"] = 120
    inventory["banana"] += 30 // เพิ่มเข้าไป
    fmt.Printf("After update: %v\n", inventory)

    // DELETE: ลบ key
    delete(inventory, "durian")
    fmt.Printf("After delete: %v\n", inventory)

    // delete key ที่ไม่มี -> ไม่ panic
    delete(inventory, "notexist")
    fmt.Println("delete non-existent key: no panic")

    // ขนาด map
    fmt.Printf("len(inventory) = %d\n", len(inventory))
}
```

---

## 8.3 Checking Key Existence

```go
package main

import "fmt"

func main() {
    cache := map[string]string{
        "user:1": "สมชาย",
        "user:2": "สมหญิง",
    }

    // Two-value assignment: value, ok
    keys := []string{"user:1", "user:2", "user:3", "user:4"}

    for _, key := range keys {
        if value, ok := cache[key]; ok {
            fmt.Printf("%-10s -> found: %s\n", key, value)
        } else {
            fmt.Printf("%-10s -> not found\n", key)
        }
    }

    // ตรวจสอบเฉพาะ existence
    if _, ok := cache["user:1"]; ok {
        fmt.Println("\nuser:1 มีอยู่ใน cache")
    }

    // Default value pattern
    config := map[string]string{
        "host": "localhost",
        "port": "5432",
    }

    getConfig := func(key, defaultVal string) string {
        if val, ok := config[key]; ok {
            return val
        }
        return defaultVal
    }

    fmt.Printf("\nhost:     %s\n", getConfig("host", "127.0.0.1"))
    fmt.Printf("port:     %s\n", getConfig("port", "3306"))
    fmt.Printf("timeout:  %s (default)\n", getConfig("timeout", "30s"))
    fmt.Printf("maxConns: %s (default)\n", getConfig("maxConns", "10"))
}
```

---

## 8.4 Iterating Maps

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    population := map[string]int{
        "กรุงเทพ":   10500000,
        "เชียงใหม่": 131111,
        "ภูเก็ต":    76032,
        "ขอนแก่น":  106591,
        "ชลบุรี":    61680,
    }

    // for range (ลำดับสุ่ม!)
    fmt.Println("=== ลำดับสุ่ม ===")
    for city, pop := range population {
        fmt.Printf("  %-15s: %,d\n", city, pop)
    }

    // เรียงตาม key
    fmt.Println("\n=== เรียงตาม key ===")
    keys := make([]string, 0, len(population))
    for k := range population {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    for _, k := range keys {
        fmt.Printf("  %-15s: %d\n", k, population[k])
    }

    // เรียงตาม value
    fmt.Println("\n=== เรียงตาม population (มากไปน้อย) ===")
    type cityPop struct {
        Name string
        Pop  int
    }
    cities := make([]cityPop, 0, len(population))
    for name, pop := range population {
        cities = append(cities, cityPop{name, pop})
    }
    sort.Slice(cities, func(i, j int) bool {
        return cities[i].Pop > cities[j].Pop
    })
    for rank, c := range cities {
        fmt.Printf("  %d. %-15s: %d\n", rank+1, c.Name, c.Pop)
    }

    // วนซ้ำเฉพาะ keys
    fmt.Println("\n=== เฉพาะ keys ===")
    for k := range population {
        fmt.Printf("  %s\n", k)
    }
}
```

---

## 8.5 Maps of Slices

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    // Map ของ slices: grouping
    studentsByGrade := map[string][]string{}

    students := []struct {
        Name  string
        Grade string
    }{
        {"สมชาย", "A"}, {"สมหญิง", "B"}, {"สมศักดิ์", "A"},
        {"สมใจ", "C"}, {"สมพร", "B"}, {"สมคิด", "A"},
        {"สมนึก", "C"}, {"สมบัติ", "B"},
    }

    // Group by grade
    for _, s := range students {
        studentsByGrade[s.Grade] = append(studentsByGrade[s.Grade], s.Name)
    }

    // แสดงผลตามเกรด
    grades := []string{"A", "B", "C", "D", "F"}
    for _, grade := range grades {
        if names, ok := studentsByGrade[grade]; ok {
            sort.Strings(names)
            fmt.Printf("เกรด %s: %v\n", grade, names)
        }
    }

    // Map ของ slices: adjacency list (graph)
    graph := map[string][]string{
        "กรุงเทพ":   {"เชียงใหม่", "ขอนแก่น", "ชลบุรี"},
        "เชียงใหม่": {"กรุงเทพ", "ลำพูน"},
        "ขอนแก่น":  {"กรุงเทพ", "อุดรธานี"},
        "ชลบุรี":    {"กรุงเทพ"},
        "ลำพูน":    {"เชียงใหม่"},
        "อุดรธานี":  {"ขอนแก่น"},
    }

    fmt.Println("\nGraph adjacency list:")
    for city, neighbors := range graph {
        sort.Strings(neighbors)
        fmt.Printf("  %-12s -> %v\n", city, neighbors)
    }

    // BFS traversal
    visited := make(map[string]bool)
    queue := []string{"กรุงเทพ"}
    var order []string

    for len(queue) > 0 {
        city := queue[0]
        queue = queue[1:]
        if visited[city] {
            continue
        }
        visited[city] = true
        order = append(order, city)
        neighbors := graph[city]
        sort.Strings(neighbors)
        for _, n := range neighbors {
            if !visited[n] {
                queue = append(queue, n)
            }
        }
    }
    fmt.Printf("\nBFS จาก กรุงเทพ: %v\n", order)
}
```

---

## 8.6 Maps of Structs

```go
package main

import (
    "fmt"
    "time"
)

type Employee struct {
    Name       string
    Department string
    Salary     float64
    JoinDate   time.Time
}

func main() {
    employees := map[string]Employee{
        "EMP001": {
            Name:       "สมชาย จารึก",
            Department: "Engineering",
            Salary:     65000,
            JoinDate:   time.Date(2020, 3, 15, 0, 0, 0, 0, time.UTC),
        },
        "EMP002": {
            Name:       "สมหญิง ดีใจ",
            Department: "Marketing",
            Salary:     55000,
            JoinDate:   time.Date(2021, 7, 1, 0, 0, 0, 0, time.UTC),
        },
        "EMP003": {
            Name:       "สมศักดิ์ รักดี",
            Department: "Engineering",
            Salary:     72000,
            JoinDate:   time.Date(2019, 11, 20, 0, 0, 0, 0, time.UTC),
        },
    }

    // อ่านข้อมูล
    if emp, ok := employees["EMP001"]; ok {
        fmt.Printf("พนักงาน: %s (%s)\n", emp.Name, emp.Department)
    }

    // ระวัง: ไม่สามารถแก้ไข struct field โดยตรงใน map
    // employees["EMP001"].Salary = 70000 // compile error!

    // ต้องทำแบบนี้
    emp := employees["EMP001"]
    emp.Salary = 70000
    employees["EMP001"] = emp
    fmt.Printf("Salary ใหม่: %.0f\n", employees["EMP001"].Salary)

    // หรือใช้ pointer
    employees2 := map[string]*Employee{
        "EMP001": {Name: "สมชาย", Salary: 65000},
    }
    employees2["EMP001"].Salary = 70000 // OK ด้วย pointer
    fmt.Printf("Salary (ptr): %.0f\n", employees2["EMP001"].Salary)

    // Group by department
    byDept := make(map[string][]Employee)
    for _, emp := range employees {
        byDept[emp.Department] = append(byDept[emp.Department], emp)
    }

    fmt.Println("\nพนักงานตามแผนก:")
    for dept, emps := range byDept {
        fmt.Printf("  %s:\n", dept)
        for _, e := range emps {
            fmt.Printf("    - %s (%.0f)\n", e.Name, e.Salary)
        }
    }

    // สรุปเงินเดือนตาม department
    fmt.Println("\nเงินเดือนรวมตามแผนก:")
    for dept, emps := range byDept {
        total := 0.0
        for _, e := range emps {
            total += e.Salary
        }
        fmt.Printf("  %-15s: %.0f\n", dept, total)
    }
}
```

---

## 8.7 Map Patterns

### Pattern 1: Word Counting

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

func wordCount(text string) map[string]int {
    counts := make(map[string]int)
    // normalize และแยกคำ
    text = strings.ToLower(text)
    for _, word := range strings.Fields(text) {
        // ลบ punctuation
        word = strings.Trim(word, ".,!?;:\"'()")
        if word != "" {
            counts[word]++
        }
    }
    return counts
}

func topN(counts map[string]int, n int) []struct {
    Word  string
    Count int
} {
    type wc struct {
        Word  string
        Count int
    }
    pairs := make([]wc, 0, len(counts))
    for word, count := range counts {
        pairs = append(pairs, wc{word, count})
    }
    sort.Slice(pairs, func(i, j int) bool {
        if pairs[i].Count != pairs[j].Count {
            return pairs[i].Count > pairs[j].Count
        }
        return pairs[i].Word < pairs[j].Word
    })
    if n > len(pairs) {
        n = len(pairs)
    }
    result := make([]struct{ Word string; Count int }, n)
    for i := 0; i < n; i++ {
        result[i].Word = pairs[i].Word
        result[i].Count = pairs[i].Count
    }
    return result
}

func main() {
    text := `Go is an open source programming language that makes it easy
    to build simple reliable and efficient software Go is great
    for building web servers and command line tools Go is easy to learn`

    counts := wordCount(text)

    fmt.Println("=== Word Frequency ===")
    top := topN(counts, 10)
    for i, wc := range top {
        fmt.Printf("%2d. %-15s: %d\n", i+1, wc.Word, wc.Count)
    }

    fmt.Printf("\nจำนวนคำทั้งหมด: %d unique words\n", len(counts))
}
```

### Pattern 2: Grouping

```go
package main

import (
    "fmt"
    "sort"
)

type Order struct {
    ID         int
    CustomerID string
    Product    string
    Amount     float64
    Status     string
}

func groupBy[K comparable, V any](items []V, keyFn func(V) K) map[K][]V {
    result := make(map[K][]V)
    for _, item := range items {
        key := keyFn(item)
        result[key] = append(result[key], item)
    }
    return result
}

func main() {
    orders := []Order{
        {1, "C001", "Go Book", 350, "completed"},
        {2, "C002", "Laptop", 35000, "pending"},
        {3, "C001", "Mouse", 890, "completed"},
        {4, "C003", "Keyboard", 1500, "cancelled"},
        {5, "C002", "Monitor", 8900, "completed"},
        {6, "C001", "Headset", 2500, "pending"},
        {7, "C003", "Webcam", 1200, "completed"},
    }

    // Group by customer
    byCustomer := groupBy(orders, func(o Order) string { return o.CustomerID })
    fmt.Println("=== Orders by Customer ===")
    customers := []string{"C001", "C002", "C003"}
    for _, cid := range customers {
        total := 0.0
        for _, o := range byCustomer[cid] {
            total += o.Amount
        }
        fmt.Printf("%s: %d orders, total=%.0f\n", cid, len(byCustomer[cid]), total)
    }

    // Group by status
    byStatus := groupBy(orders, func(o Order) string { return o.Status })
    fmt.Println("\n=== Orders by Status ===")
    statuses := []string{"completed", "pending", "cancelled"}
    for _, status := range statuses {
        orders := byStatus[status]
        total := 0.0
        for _, o := range orders {
            total += o.Amount
        }
        fmt.Printf("%-12s: %d orders, total=%.0f\n", status, len(orders), total)
    }

    // Find biggest spender
    customerTotal := make(map[string]float64)
    for _, o := range orders {
        if o.Status == "completed" {
            customerTotal[o.CustomerID] += o.Amount
        }
    }
    type cs struct{ ID string; Total float64 }
    var ranked []cs
    for id, total := range customerTotal {
        ranked = append(ranked, cs{id, total})
    }
    sort.Slice(ranked, func(i, j int) bool { return ranked[i].Total > ranked[j].Total })
    fmt.Println("\n=== Customer Ranking ===")
    for i, c := range ranked {
        fmt.Printf("%d. %s: %.0f\n", i+1, c.ID, c.Total)
    }
}
```

### Pattern 3: Caching/Memoization

```go
package main

import (
    "fmt"
    "time"
)

type Cache struct {
    data    map[string]cacheEntry
    ttl     time.Duration
}

type cacheEntry struct {
    value     interface{}
    expiresAt time.Time
}

func NewCache(ttl time.Duration) *Cache {
    return &Cache{
        data: make(map[string]cacheEntry),
        ttl:  ttl,
    }
}

func (c *Cache) Set(key string, value interface{}) {
    c.data[key] = cacheEntry{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    entry, ok := c.data[key]
    if !ok {
        return nil, false
    }
    if time.Now().After(entry.expiresAt) {
        delete(c.data, key)
        return nil, false
    }
    return entry.value, true
}

func (c *Cache) Delete(key string) {
    delete(c.data, key)
}

func (c *Cache) Size() int {
    return len(c.data)
}

// Fibonacci with memoization
func makeFibMemo() func(int) int {
    memo := map[int]int{}
    var fib func(int) int
    fib = func(n int) int {
        if n <= 1 {
            return n
        }
        if v, ok := memo[n]; ok {
            return v
        }
        result := fib(n-1) + fib(n-2)
        memo[n] = result
        return result
    }
    return fib
}

func main() {
    // Cache
    cache := NewCache(100 * time.Millisecond)
    cache.Set("user:1", map[string]string{"name": "สมชาย"})
    cache.Set("user:2", map[string]string{"name": "สมหญิง"})

    if v, ok := cache.Get("user:1"); ok {
        fmt.Printf("cache hit: %v\n", v)
    }

    time.Sleep(150 * time.Millisecond) // รอให้ expire

    if _, ok := cache.Get("user:1"); !ok {
        fmt.Println("cache expired")
    }

    // Fibonacci memo
    fib := makeFibMemo()
    fmt.Println("\nFibonacci (memoized):")
    for i := 0; i <= 15; i++ {
        fmt.Printf("fib(%2d) = %d\n", i, fib(i))
    }
}
```

### Pattern 4: Frequency Counter

```go
package main

import (
    "fmt"
    "sort"
)

func frequency[T comparable](items []T) map[T]int {
    freq := make(map[T]int)
    for _, item := range items {
        freq[item]++
    }
    return freq
}

func mostCommon[T comparable](freq map[T]int, n int) []T {
    type pair struct {
        Key   T
        Count int
    }
    pairs := make([]pair, 0, len(freq))
    for k, v := range freq {
        pairs = append(pairs, pair{k, v})
    }
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].Count > pairs[j].Count
    })
    result := make([]T, 0, n)
    for i := 0; i < n && i < len(pairs); i++ {
        result = append(result, pairs[i].Key)
    }
    return result
}

func main() {
    // Int frequency
    nums := []int{1, 3, 2, 3, 4, 3, 1, 2, 1, 5, 4, 3}
    numFreq := frequency(nums)
    fmt.Println("Number frequency:")
    // เรียงตาม key เพื่อแสดงผลที่สม่ำเสมอ
    keys := make([]int, 0)
    for k := range numFreq {
        keys = append(keys, k)
    }
    sort.Ints(keys)
    for _, k := range keys {
        fmt.Printf("  %d: %s\n", k, repeatChar('*', numFreq[k]))
    }

    // String frequency
    words := []string{"go", "python", "go", "java", "go", "python", "rust"}
    wordFreq := frequency(words)
    top := mostCommon(wordFreq, 3)
    fmt.Printf("\nTop languages: %v\n", top)

    // Anagram detection
    isAnagram := func(s, t string) bool {
        if len(s) != len(t) {
            return false
        }
        freq := make(map[rune]int)
        for _, c := range s {
            freq[c]++
        }
        for _, c := range t {
            freq[c]--
            if freq[c] < 0 {
                return false
            }
        }
        return true
    }
    pairs := [][2]string{
        {"anagram", "nagaram"},
        {"rat", "car"},
        {"listen", "silent"},
        {"hello", "world"},
    }
    for _, p := range pairs {
        fmt.Printf("isAnagram(%q, %q) = %v\n", p[0], p[1], isAnagram(p[0], p[1]))
    }
}

func repeatChar(c rune, n int) string {
    s := make([]rune, n)
    for i := range s {
        s[i] = c
    }
    return string(s)
}
```

---

## 8.8 Set ด้วย Map

Go ไม่มี built-in set type แต่สามารถใช้ `map[T]struct{}` แทนได้

```go
package main

import "fmt"

type Set[T comparable] struct {
    data map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{data: make(map[T]struct{})}
}

func (s *Set[T]) Add(v T) {
    s.data[v] = struct{}{}
}

func (s *Set[T]) Remove(v T) {
    delete(s.data, v)
}

func (s *Set[T]) Contains(v T) bool {
    _, ok := s.data[v]
    return ok
}

func (s *Set[T]) Size() int {
    return len(s.data)
}

func (s *Set[T]) Items() []T {
    result := make([]T, 0, len(s.data))
    for k := range s.data {
        result = append(result, k)
    }
    return result
}

func Intersection[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range a.data {
        if b.Contains(k) {
            result.Add(k)
        }
    }
    return result
}

func Union[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range a.data {
        result.Add(k)
    }
    for k := range b.data {
        result.Add(k)
    }
    return result
}

func Difference[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range a.data {
        if !b.Contains(k) {
            result.Add(k)
        }
    }
    return result
}

func main() {
    a := NewSet[int]()
    b := NewSet[int]()

    for _, v := range []int{1, 2, 3, 4, 5} {
        a.Add(v)
    }
    for _, v := range []int{3, 4, 5, 6, 7} {
        b.Add(v)
    }

    fmt.Printf("A: %v\n", a.Items())
    fmt.Printf("B: %v\n", b.Items())
    fmt.Printf("A ∩ B: %v\n", Intersection(a, b).Items())
    fmt.Printf("A ∪ B: %v\n", Union(a, b).Items())
    fmt.Printf("A - B: %v\n", Difference(a, b).Items())
    fmt.Printf("B - A: %v\n", Difference(b, a).Items())

    // Dedup ด้วย set
    data := []string{"go", "python", "go", "java", "python", "go"}
    seen := NewSet[string]()
    var unique []string
    for _, v := range data {
        if !seen.Contains(v) {
            seen.Add(v)
            unique = append(unique, v)
        }
    }
    fmt.Printf("\noriginal: %v\n", data)
    fmt.Printf("unique:   %v\n", unique)
}
```

---

## 8.9 sync.Map สำหรับ Concurrent Access

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var sm sync.Map

    // เก็บข้อมูล
    sm.Store("key1", "value1")
    sm.Store("key2", 42)
    sm.Store("key3", []int{1, 2, 3})

    // อ่านข้อมูล
    if v, ok := sm.Load("key1"); ok {
        fmt.Printf("key1: %v\n", v)
    }

    // อ่านหรือสร้างใหม่ถ้าไม่มี
    actual, loaded := sm.LoadOrStore("key4", "new value")
    fmt.Printf("key4: %v (loaded=%v)\n", actual, loaded)

    // อ่านซ้ำ
    actual2, loaded2 := sm.LoadOrStore("key4", "another value")
    fmt.Printf("key4: %v (loaded=%v)\n", actual2, loaded2)

    // ลบ
    sm.Delete("key2")

    // วนซ้ำ
    fmt.Println("\nAll entries:")
    sm.Range(func(key, value interface{}) bool {
        fmt.Printf("  %v: %v\n", key, value)
        return true // return false เพื่อหยุด
    })

    // Concurrent example
    var wg sync.WaitGroup
    counter := &sync.Map{}

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            key := fmt.Sprintf("worker-%d", n%5) // 5 workers
            for {
                v, _ := counter.LoadOrStore(key, 0)
                if counter.CompareAndSwap(key, v, v.(int)+1) {
                    break
                }
            }
        }(i)
    }

    wg.Wait()

    fmt.Println("\nConcurrent counts:")
    total := 0
    counter.Range(func(k, v interface{}) bool {
        fmt.Printf("  %s: %d\n", k, v)
        total += v.(int)
        return true
    })
    fmt.Printf("Total: %d (should be 100)\n", total)
}
```

---

## 8.10 ตัวอย่างรวม: LRU Cache

```go
package main

import "fmt"

// LRU Cache ด้วย map + doubly linked list
type Node struct {
    key, value int
    prev, next *Node
}

type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node // Most recently used
    tail     *Node // Least recently used
}

func NewLRUCache(capacity int) *LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    return &LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}

func (c *LRUCache) remove(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) addToFront(node *Node) {
    node.next = c.head.next
    node.prev = c.head
    c.head.next.prev = node
    c.head.next = node
}

func (c *LRUCache) Get(key int) int {
    if node, ok := c.cache[key]; ok {
        c.remove(node)
        c.addToFront(node)
        return node.value
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    if node, ok := c.cache[key]; ok {
        node.value = value
        c.remove(node)
        c.addToFront(node)
        return
    }
    if len(c.cache) >= c.capacity {
        lru := c.tail.prev
        c.remove(lru)
        delete(c.cache, lru.key)
    }
    node := &Node{key: key, value: value}
    c.cache[key] = node
    c.addToFront(node)
}

func (c *LRUCache) Size() int { return len(c.cache) }

func main() {
    cache := NewLRUCache(3)

    cache.Put(1, 10)
    cache.Put(2, 20)
    cache.Put(3, 30)

    fmt.Printf("Get(1) = %d\n", cache.Get(1)) // 10
    fmt.Printf("Get(2) = %d\n", cache.Get(2)) // 20

    cache.Put(4, 40) // ไล่ key=3 ออก (LRU)
    fmt.Printf("Get(3) = %d\n", cache.Get(3)) // -1 (ถูกไล่ออก)
    fmt.Printf("Get(4) = %d\n", cache.Get(4)) // 40

    cache.Put(5, 50) // ไล่ key=1 ออก
    fmt.Printf("Get(1) = %d\n", cache.Get(1)) // -1
    fmt.Printf("Get(2) = %d\n", cache.Get(2)) // 20
    fmt.Printf("Size  = %d\n", cache.Size())   // 3
}
```

---

## Workshop: แบบฝึกหัด Part 8

### แบบฝึกหัดที่ 1: Phone Book

สร้าง phone book ด้วย map:
- `Add(name, phone string)`
- `Remove(name string)`
- `Lookup(name string) (string, bool)`
- `Update(name, newPhone string) bool`
- `ListAll() []string` (เรียงตามชื่อ)

### แบบฝึกหัดที่ 2: Inventory System

สร้างระบบ inventory:
- Map ของ products (id -> Product{name, price, qty})
- `AddProduct`, `UpdateStock`, `SellProduct`
- `LowStock(threshold int) []Product`
- `TotalValue() float64`

### แบบฝึกหัดที่ 3: Text Analyzer

วิเคราะห์ข้อความ:
- นับความถี่ของแต่ละคำ
- หา top-10 คำที่พบบ่อยสุด
- นับจำนวนคำที่ unique
- คำนวณ average word length

### แบบฝึกหัดที่ 4: Graph BFS/DFS

สร้าง graph ด้วย `map[string][]string` แล้วเขียน:
- `BFS(start string) []string`
- `DFS(start string) []string`
- `HasPath(from, to string) bool`
- `ShortestPath(from, to string) []string`

### เฉลยแบบฝึกหัดที่ 1

```go
package main

import (
    "fmt"
    "sort"
)

type PhoneBook struct {
    data map[string]string
}

func NewPhoneBook() *PhoneBook {
    return &PhoneBook{data: make(map[string]string)}
}

func (pb *PhoneBook) Add(name, phone string) {
    pb.data[name] = phone
}

func (pb *PhoneBook) Remove(name string) bool {
    if _, ok := pb.data[name]; !ok {
        return false
    }
    delete(pb.data, name)
    return true
}

func (pb *PhoneBook) Lookup(name string) (string, bool) {
    phone, ok := pb.data[name]
    return phone, ok
}

func (pb *PhoneBook) Update(name, newPhone string) bool {
    if _, ok := pb.data[name]; !ok {
        return false
    }
    pb.data[name] = newPhone
    return true
}

func (pb *PhoneBook) ListAll() []string {
    names := make([]string, 0, len(pb.data))
    for name := range pb.data {
        names = append(names, name)
    }
    sort.Strings(names)
    return names
}

func main() {
    pb := NewPhoneBook()
    pb.Add("สมชาย", "081-234-5678")
    pb.Add("สมหญิง", "089-876-5432")
    pb.Add("สมศักดิ์", "062-111-2222")

    if phone, ok := pb.Lookup("สมชาย"); ok {
        fmt.Printf("สมชาย: %s\n", phone)
    }

    pb.Update("สมชาย", "081-999-0000")
    fmt.Printf("Updated: %s\n", func() string {
        p, _ := pb.Lookup("สมชาย")
        return p
    }())

    fmt.Println("\nสมุดโทรศัพท์:")
    for _, name := range pb.ListAll() {
        phone, _ := pb.Lookup(name)
        fmt.Printf("  %-12s: %s\n", name, phone)
    }
}
```

---

## สรุป Part 8

| Operation | Code | หมายเหตุ |
|-----------|------|---------|
| Create | `m := make(map[K]V)` หรือ `m := map[K]V{}` | |
| Add/Update | `m[key] = value` | ถ้า key มีอยู่จะ update |
| Read | `v := m[key]` | ถ้าไม่มี key คืน zero value |
| Check existence | `v, ok := m[key]` | ok=true ถ้ามี key |
| Delete | `delete(m, key)` | ไม่ panic ถ้า key ไม่มี |
| Iterate | `for k, v := range m` | ลำดับสุ่ม! |
| Size | `len(m)` | |
| Concurrent | `sync.Map` | thread-safe |

**Key Takeaways:**
- Map เป็น reference type เหมือน slice
- ไม่สามารถ write ไปยัง nil map ได้ (panic) ต้องใช้ make ก่อน
- for range บน map ไม่ guarantee ลำดับ
- ไม่สามารถแก้ไข struct field ใน map โดยตรง ต้องทำผ่าน pointer หรือ copy
- ใช้ `struct{}` เป็น value สำหรับ Set (ประหยัด memory)

## Resources

- [Go Spec - Map types](https://go.dev/ref/spec#Map_types)
- [Go Blog - Maps in action](https://go.dev/blog/maps)
- [Go by Example - Maps](https://gobyexample.com/maps)
- [sync.Map documentation](https://pkg.go.dev/sync#Map)
- [Effective Go - Maps](https://go.dev/doc/effective_go#maps)
