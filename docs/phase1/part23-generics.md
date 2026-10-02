# Part 23: Generics ใน Go (Go 1.18+)

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Type Parameters และ Type Constraints
- ใช้ Built-in constraints (`comparable`, `any`)
- เขียน Generic Functions
- สร้าง Generic Types (structs, interfaces)
- เข้าใจ Type Inference
- สร้าง Generic Data Structures (Stack, Queue, Set)
- รู้ว่าเมื่อไหรควรใช้ Generics

---

## 23.1 Generics Overview

### 23.1.1 ทำไมต้องมี Generics?

```go
package main

import "fmt"

// ก่อน Generics - ต้องเขียน function แยกสำหรับแต่ละ type
func sumInts(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func sumFloats(nums []float64) float64 {
    var total float64
    for _, n := range nums {
        total += n
    }
    return total
}

func sumInt32s(nums []int32) int32 {
    var total int32
    for _, n := range nums {
        total += n
    }
    return total
}

// หรือใช้ interface{} แต่ไม่ type safe และต้อง type assertion
func sumAny(nums []interface{}) interface{} {
    // ไม่สะดวกและ error prone
    return nil
}

// With Generics (Go 1.18+)
// type constraint ที่กำหนด types ที่อนุญาต
type Number interface {
    int | int8 | int16 | int32 | int64 |
        float32 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    ints := []int{1, 2, 3, 4, 5}
    floats := []float64{1.1, 2.2, 3.3}
    int32s := []int32{10, 20, 30}
    
    fmt.Println("Sum ints:", Sum(ints))
    fmt.Println("Sum floats:", Sum(floats))
    fmt.Println("Sum int32s:", Sum(int32s))
    
    // Type inference - Go ดูจาก argument โดยอัตโนมัติ
    // Sum[int](ints) ก็ได้ แต่ไม่จำเป็น
}
```

---

## 23.2 Type Parameters

### 23.2.1 Syntax ของ Type Parameters

```go
package main

import "fmt"

// Function กับ type parameter
// func Name[T constraint](params) returnType
func First[T any](slice []T) (T, bool) {
    if len(slice) == 0 {
        var zero T
        return zero, false
    }
    return slice[0], true
}

func Last[T any](slice []T) (T, bool) {
    if len(slice) == 0 {
        var zero T
        return zero, false
    }
    return slice[len(slice)-1], true
}

// Multiple type parameters
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

func Filter[T any](slice []T, pred func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func Reduce[T, U any](slice []T, initial U, f func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = f(result, v)
    }
    return result
}

// Zip - รวม 2 slices เข้าด้วยกัน
func Zip[T, U any](a []T, b []U) []struct{ First T; Second U } {
    length := len(a)
    if len(b) < length {
        length = len(b)
    }
    
    result := make([]struct{ First T; Second U }, length)
    for i := 0; i < length; i++ {
        result[i].First = a[i]
        result[i].Second = b[i]
    }
    return result
}

func main() {
    // First/Last
    nums := []int{1, 2, 3, 4, 5}
    strs := []string{"a", "b", "c"}
    
    if first, ok := First(nums); ok {
        fmt.Println("First num:", first)
    }
    if last, ok := Last(strs); ok {
        fmt.Println("Last str:", last)
    }
    
    // Map
    doubled := Map(nums, func(n int) int { return n * 2 })
    fmt.Println("Doubled:", doubled)
    
    strsToLen := Map(strs, func(s string) int { return len(s) })
    fmt.Println("Lengths:", strsToLen)
    
    // Filter
    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println("Evens:", evens)
    
    // Reduce
    sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })
    fmt.Println("Sum:", sum)
    
    concat := Reduce(strs, "", func(acc, s string) string { return acc + s })
    fmt.Println("Concat:", concat)
    
    // Zip
    names := []string{"Alice", "Bob", "Charlie"}
    ages := []int{30, 25, 35}
    
    pairs := Zip(names, ages)
    for _, p := range pairs {
        fmt.Printf("%s: %d\n", p.First, p.Second)
    }
}
```

---

## 23.3 Type Constraints

### 23.3.1 การสร้าง Constraints

```go
package main

import (
    "fmt"
    "golang.org/x/exp/constraints" // ต้อง go get
)

// Constraint แบบ union
type Integer interface {
    int | int8 | int16 | int32 | int64
}

type Float interface {
    float32 | float64
}

type Number interface {
    Integer | Float
}

// Constraint แบบ ~T (underlying type)
// ~int หมายความว่า int และ types ที่มี underlying type เป็น int
type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

// Named int type
type Celsius float64
type Fahrenheit float64

// ทำงานได้กับ Celsius และ Fahrenheit เพราะ underlying type เป็น float64
type Temperature interface {
    ~float64
}

func ToFloat[T Temperature](v T) float64 {
    return float64(v)
}

// Constraint กับ methods
type Stringer interface {
    String() string
}

type Orderable interface {
    comparable
    Less(other interface{}) bool
}

// Constraint หลายๆ ข้อ
type NumberOrString interface {
    ~int | ~float64 | ~string
}

func printValue[T NumberOrString](v T) {
    fmt.Printf("value: %v (type: %T)\n", v, v)
}

// ใช้ constraints package
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func Clamp[T constraints.Ordered](value, min, max T) T {
    if value < min {
        return min
    }
    if value > max {
        return max
    }
    return value
}

func main() {
    // Temperature
    c := Celsius(100.0)
    f := Fahrenheit(212.0)
    
    fmt.Printf("Celsius as float64: %.1f\n", ToFloat(c))
    fmt.Printf("Fahrenheit as float64: %.1f\n", ToFloat(f))
    
    // printValue
    printValue(42)
    printValue(3.14)
    printValue("hello")
    
    // Min/Max
    fmt.Println("\nMin/Max:")
    fmt.Println("Min(3, 5):", Min(3, 5))
    fmt.Println("Max(3, 5):", Max(3, 5))
    fmt.Println("Min(\"a\", \"b\"):", Min("a", "b"))
    
    // Clamp
    fmt.Println("\nClamp:")
    fmt.Println("Clamp(10, 0, 5):", Clamp(10, 0, 5))
    fmt.Println("Clamp(-1, 0, 5):", Clamp(-1, 0, 5))
    fmt.Println("Clamp(3, 0, 5):", Clamp(3, 0, 5))
}
```

### 23.3.2 Built-in Constraints

```go
package main

import "fmt"

// comparable - ทุก type ที่ใช้ == และ != ได้
// ใช้สำหรับ map keys และ contains functions
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

func IndexOf[T comparable](slice []T, target T) int {
    for i, v := range slice {
        if v == target {
            return i
        }
    }
    return -1
}

func Unique[T comparable](slice []T) []T {
    seen := make(map[T]bool)
    var result []T
    for _, v := range slice {
        if !seen[v] {
            seen[v] = true
            result = append(result, v)
        }
    }
    return result
}

func GroupBy[T any, K comparable](slice []T, keyFn func(T) K) map[K][]T {
    result := make(map[K][]T)
    for _, v := range slice {
        key := keyFn(v)
        result[key] = append(result[key], v)
    }
    return result
}

func CountBy[T any, K comparable](slice []T, keyFn func(T) K) map[K]int {
    result := make(map[K]int)
    for _, v := range slice {
        result[keyFn(v)]++
    }
    return result
}

// any - ทุก type (alias ของ interface{})
func ToString[T any](v T) string {
    return fmt.Sprintf("%v", v)
}

func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func Values[K comparable, V any](m map[K]V) []V {
    vals := make([]V, 0, len(m))
    for _, v := range m {
        vals = append(vals, v)
    }
    return vals
}

func main() {
    // Contains
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println("Contains 3:", Contains(nums, 3))
    fmt.Println("Contains 9:", Contains(nums, 9))
    
    words := []string{"hello", "world", "go"}
    fmt.Println("Contains 'go':", Contains(words, "go"))
    
    // IndexOf
    fmt.Println("IndexOf 3:", IndexOf(nums, 3))
    fmt.Println("IndexOf 9:", IndexOf(nums, 9))
    
    // Unique
    dup := []int{1, 2, 2, 3, 3, 3, 4}
    fmt.Println("Unique:", Unique(dup))
    
    dupStr := []string{"a", "b", "a", "c", "b"}
    fmt.Println("Unique strings:", Unique(dupStr))
    
    // GroupBy
    type Person struct {
        Name string
        Dept string
    }
    
    people := []Person{
        {"Alice", "Engineering"},
        {"Bob", "Marketing"},
        {"Charlie", "Engineering"},
        {"Dave", "HR"},
        {"Eve", "Engineering"},
    }
    
    byDept := GroupBy(people, func(p Person) string { return p.Dept })
    for dept, members := range byDept {
        names := Map(members, func(p Person) string { return p.Name })
        fmt.Printf("%s: %v\n", dept, names)
    }
    
    // CountBy
    counts := CountBy(people, func(p Person) string { return p.Dept })
    for dept, count := range counts {
        fmt.Printf("%s: %d people\n", dept, count)
    }
    
    // Keys/Values
    m := map[string]int{"a": 1, "b": 2, "c": 3}
    fmt.Println("Keys:", Keys(m))
    fmt.Println("Values:", Values(m))
}

func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}
```

---

## 23.4 Generic Types

### 23.4.1 Generic Stack

```go
package main

import (
    "errors"
    "fmt"
)

var ErrEmptyStack = errors.New("stack is empty")

// Generic Stack
type Stack[T any] struct {
    items []T
}

func NewStack[T any]() *Stack[T] {
    return &Stack[T]{}
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, error) {
    if s.IsEmpty() {
        var zero T
        return zero, ErrEmptyStack
    }
    
    last := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return last, nil
}

func (s *Stack[T]) Peek() (T, error) {
    if s.IsEmpty() {
        var zero T
        return zero, ErrEmptyStack
    }
    return s.items[len(s.items)-1], nil
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

func (s *Stack[T]) Size() int {
    return len(s.items)
}

func (s *Stack[T]) Items() []T {
    result := make([]T, len(s.items))
    copy(result, s.items)
    return result
}

func main() {
    // Integer stack
    intStack := NewStack[int]()
    intStack.Push(1)
    intStack.Push(2)
    intStack.Push(3)
    
    fmt.Println("Stack:", intStack.Items())
    fmt.Println("Size:", intStack.Size())
    
    if top, err := intStack.Peek(); err == nil {
        fmt.Println("Top:", top)
    }
    
    for !intStack.IsEmpty() {
        if v, err := intStack.Pop(); err == nil {
            fmt.Printf("Popped: %d\n", v)
        }
    }
    
    _, err := intStack.Pop()
    fmt.Println("Pop empty:", err)
    
    // String stack
    strStack := NewStack[string]()
    strStack.Push("hello")
    strStack.Push("world")
    strStack.Push("go")
    
    fmt.Println("\nString Stack:", strStack.Items())
    
    // Struct stack
    type Task struct {
        ID   int
        Name string
    }
    
    taskStack := NewStack[Task]()
    taskStack.Push(Task{1, "Setup"})
    taskStack.Push(Task{2, "Develop"})
    taskStack.Push(Task{3, "Test"})
    
    fmt.Println("\nTask Stack:")
    for !taskStack.IsEmpty() {
        if task, err := taskStack.Pop(); err == nil {
            fmt.Printf("  Task %d: %s\n", task.ID, task.Name)
        }
    }
}
```

### 23.4.2 Generic Queue

```go
package main

import (
    "errors"
    "fmt"
)

var ErrEmptyQueue = errors.New("queue is empty")

// Generic Queue (FIFO)
type Queue[T any] struct {
    items []T
}

func NewQueue[T any]() *Queue[T] {
    return &Queue[T]{}
}

func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item)
}

func (q *Queue[T]) Dequeue() (T, error) {
    if q.IsEmpty() {
        var zero T
        return zero, ErrEmptyQueue
    }
    
    front := q.items[0]
    q.items = q.items[1:]
    return front, nil
}

func (q *Queue[T]) Front() (T, error) {
    if q.IsEmpty() {
        var zero T
        return zero, ErrEmptyQueue
    }
    return q.items[0], nil
}

func (q *Queue[T]) IsEmpty() bool {
    return len(q.items) == 0
}

func (q *Queue[T]) Size() int {
    return len(q.items)
}

// Priority Queue
type PriorityItem[T any] struct {
    Value    T
    Priority int
}

type PriorityQueue[T any] struct {
    items []PriorityItem[T]
}

func NewPriorityQueue[T any]() *PriorityQueue[T] {
    return &PriorityQueue[T]{}
}

func (pq *PriorityQueue[T]) Enqueue(item T, priority int) {
    pq.items = append(pq.items, PriorityItem[T]{Value: item, Priority: priority})
    // Sort by priority (higher = first)
    for i := len(pq.items) - 1; i > 0; i-- {
        if pq.items[i].Priority > pq.items[i-1].Priority {
            pq.items[i], pq.items[i-1] = pq.items[i-1], pq.items[i]
        } else {
            break
        }
    }
}

func (pq *PriorityQueue[T]) Dequeue() (T, error) {
    if len(pq.items) == 0 {
        var zero T
        return zero, errors.New("priority queue is empty")
    }
    item := pq.items[0]
    pq.items = pq.items[1:]
    return item.Value, nil
}

func (pq *PriorityQueue[T]) IsEmpty() bool {
    return len(pq.items) == 0
}

func main() {
    // Regular Queue
    fmt.Println("=== Regular Queue ===")
    q := NewQueue[string]()
    q.Enqueue("first")
    q.Enqueue("second")
    q.Enqueue("third")
    
    fmt.Println("Size:", q.Size())
    
    for !q.IsEmpty() {
        if v, err := q.Dequeue(); err == nil {
            fmt.Printf("Dequeued: %s\n", v)
        }
    }
    
    // Priority Queue
    fmt.Println("\n=== Priority Queue ===")
    pq := NewPriorityQueue[string]()
    pq.Enqueue("low priority task", 1)
    pq.Enqueue("high priority task", 10)
    pq.Enqueue("medium priority task", 5)
    pq.Enqueue("critical task", 100)
    
    fmt.Println("Processing tasks by priority:")
    for !pq.IsEmpty() {
        if task, err := pq.Dequeue(); err == nil {
            fmt.Printf("  - %s\n", task)
        }
    }
}
```

### 23.4.3 Generic Set

```go
package main

import "fmt"

// Generic Set
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
    s := &Set[T]{items: make(map[T]struct{})}
    for _, item := range items {
        s.Add(item)
    }
    return s
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
    delete(s.items, item)
}

func (s *Set[T]) Contains(item T) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set[T]) Size() int {
    return len(s.items)
}

func (s *Set[T]) Items() []T {
    result := make([]T, 0, len(s.items))
    for item := range s.items {
        result = append(result, item)
    }
    return result
}

// Set operations
func Union[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range a.items {
        result.Add(item)
    }
    for item := range b.items {
        result.Add(item)
    }
    return result
}

func Intersection[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range a.items {
        if b.Contains(item) {
            result.Add(item)
        }
    }
    return result
}

func Difference[T comparable](a, b *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range a.items {
        if !b.Contains(item) {
            result.Add(item)
        }
    }
    return result
}

func IsSubset[T comparable](sub, super *Set[T]) bool {
    for item := range sub.items {
        if !super.Contains(item) {
            return false
        }
    }
    return true
}

func main() {
    fmt.Println("=== Integer Set ===")
    s1 := NewSet(1, 2, 3, 4, 5)
    s2 := NewSet(3, 4, 5, 6, 7)
    
    fmt.Println("s1:", s1.Items())
    fmt.Println("s2:", s2.Items())
    fmt.Println("Union:", Union(s1, s2).Items())
    fmt.Println("Intersection:", Intersection(s1, s2).Items())
    fmt.Println("Difference (s1-s2):", Difference(s1, s2).Items())
    
    s3 := NewSet(3, 4)
    fmt.Printf("s3 is subset of s1: %v\n", IsSubset(s3, s1))
    fmt.Printf("s1 is subset of s2: %v\n", IsSubset(s1, s2))
    
    fmt.Println("\n=== String Set ===")
    fruits1 := NewSet("apple", "banana", "cherry")
    fruits2 := NewSet("banana", "cherry", "date", "elderberry")
    
    fmt.Println("Common fruits:", Intersection(fruits1, fruits2).Items())
    fmt.Println("All fruits:", Union(fruits1, fruits2).Items())
    fmt.Println("Only in fruits1:", Difference(fruits1, fruits2).Items())
    
    fmt.Println("\n=== Operations ===")
    s := NewSet[int]()
    s.Add(1)
    s.Add(2)
    s.Add(2) // duplicate - ignored
    fmt.Println("Size after adding 1,2,2:", s.Size())
    
    s.Remove(1)
    fmt.Println("Size after removing 1:", s.Size())
    fmt.Println("Contains 2:", s.Contains(2))
    fmt.Println("Contains 1:", s.Contains(1))
}
```

---

## 23.5 Generic Data Structures

### 23.5.1 Generic LinkedList

```go
package main

import "fmt"

type Node[T any] struct {
    Value T
    Next  *Node[T]
}

type LinkedList[T any] struct {
    head *Node[T]
    size int
}

func NewLinkedList[T any]() *LinkedList[T] {
    return &LinkedList[T]{}
}

func (l *LinkedList[T]) Prepend(value T) {
    l.head = &Node[T]{Value: value, Next: l.head}
    l.size++
}

func (l *LinkedList[T]) Append(value T) {
    newNode := &Node[T]{Value: value}
    
    if l.head == nil {
        l.head = newNode
    } else {
        curr := l.head
        for curr.Next != nil {
            curr = curr.Next
        }
        curr.Next = newNode
    }
    l.size++
}

func (l *LinkedList[T]) RemoveFirst() (T, bool) {
    if l.head == nil {
        var zero T
        return zero, false
    }
    
    value := l.head.Value
    l.head = l.head.Next
    l.size--
    return value, true
}

func (l *LinkedList[T]) Size() int {
    return l.size
}

func (l *LinkedList[T]) ToSlice() []T {
    result := make([]T, 0, l.size)
    curr := l.head
    for curr != nil {
        result = append(result, curr.Value)
        curr = curr.Next
    }
    return result
}

func (l *LinkedList[T]) ForEach(f func(T)) {
    curr := l.head
    for curr != nil {
        f(curr.Value)
        curr = curr.Next
    }
}

// Generic BinaryTree
type TreeNode[T any] struct {
    Value T
    Left  *TreeNode[T]
    Right *TreeNode[T]
}

type BST[T any] struct {
    root *TreeNode[T]
    less func(a, b T) bool
}

func NewBST[T any](less func(a, b T) bool) *BST[T] {
    return &BST[T]{less: less}
}

func (t *BST[T]) Insert(value T) {
    t.root = t.insert(t.root, value)
}

func (t *BST[T]) insert(node *TreeNode[T], value T) *TreeNode[T] {
    if node == nil {
        return &TreeNode[T]{Value: value}
    }
    
    if t.less(value, node.Value) {
        node.Left = t.insert(node.Left, value)
    } else if t.less(node.Value, value) {
        node.Right = t.insert(node.Right, value)
    }
    // equal → ignore
    
    return node
}

func (t *BST[T]) InOrder() []T {
    var result []T
    t.inOrder(t.root, &result)
    return result
}

func (t *BST[T]) inOrder(node *TreeNode[T], result *[]T) {
    if node == nil {
        return
    }
    t.inOrder(node.Left, result)
    *result = append(*result, node.Value)
    t.inOrder(node.Right, result)
}

func main() {
    // LinkedList
    fmt.Println("=== LinkedList ===")
    list := NewLinkedList[int]()
    list.Append(1)
    list.Append(2)
    list.Append(3)
    list.Prepend(0)
    
    fmt.Println("List:", list.ToSlice())
    fmt.Println("Size:", list.Size())
    
    v, _ := list.RemoveFirst()
    fmt.Println("Removed first:", v)
    fmt.Println("After remove:", list.ToSlice())
    
    // String LinkedList
    strList := NewLinkedList[string]()
    for _, s := range []string{"hello", "world", "go"} {
        strList.Append(s)
    }
    strList.ForEach(func(s string) {
        fmt.Printf("  %s\n", s)
    })
    
    // BST
    fmt.Println("\n=== BST (Integer) ===")
    bst := NewBST[int](func(a, b int) bool { return a < b })
    for _, n := range []int{5, 3, 7, 1, 4, 6, 8} {
        bst.Insert(n)
    }
    fmt.Println("In-order:", bst.InOrder()) // sorted
    
    // BST with string
    fmt.Println("\n=== BST (String) ===")
    strBST := NewBST[string](func(a, b string) bool { return a < b })
    for _, s := range []string{"banana", "apple", "cherry", "date"} {
        strBST.Insert(s)
    }
    fmt.Println("In-order:", strBST.InOrder())
}
```

---

## 23.6 Type Inference

### 23.6.1 Type Inference Rules

```go
package main

import "fmt"

func Identity[T any](v T) T {
    return v
}

func Pair[T, U any](t T, u U) (T, U) {
    return t, u
}

func MakeSlice[T any](items ...T) []T {
    return items
}

type Pair2[T, U any] struct {
    First  T
    Second U
}

func main() {
    // Type inference จาก arguments
    n := Identity(42)       // T = int
    s := Identity("hello")  // T = string
    b := Identity(true)     // T = bool
    
    fmt.Printf("n: %v (%T)\n", n, n)
    fmt.Printf("s: %v (%T)\n", s, s)
    fmt.Printf("b: %v (%T)\n", b, b)
    
    // Multiple type params
    a, c := Pair(1, "one")
    fmt.Printf("Pair: %v, %v\n", a, c)
    
    x, y := Pair(3.14, true)
    fmt.Printf("Pair: %v, %v\n", x, y)
    
    // MakeSlice
    ints := MakeSlice(1, 2, 3, 4, 5)
    strs := MakeSlice("a", "b", "c")
    
    fmt.Println("Ints:", ints)
    fmt.Println("Strs:", strs)
    
    // Struct generic type ต้องระบุ type parameter
    p := Pair2[string, int]{"Alice", 30}
    fmt.Printf("Pair2: %+v\n", p)
    
    // บางครั้ง Go ไม่สามารถ infer type ได้ ต้องระบุเอง
    // เช่น return type
    nums := MakeSlice[int]()  // explicit type
    fmt.Println("Empty ints:", nums)
}
```

---

## 23.7 เมื่อไหรควรใช้ Generics

### 23.7.1 Guidelines

```go
package main

import "fmt"

// ควรใช้ Generics เมื่อ:

// 1. Code ที่ทำงานเหมือนกันกับหลาย types
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// 2. Data structures ที่ type-agnostic
type Optional[T any] struct {
    value    T
    hasValue bool
}

func Some[T any](v T) Optional[T] {
    return Optional[T]{value: v, hasValue: true}
}

func None[T any]() Optional[T] {
    return Optional[T]{}
}

func (o Optional[T]) Get() (T, bool) {
    return o.value, o.hasValue
}

func (o Optional[T]) OrElse(defaultVal T) T {
    if o.hasValue {
        return o.value
    }
    return defaultVal
}

// 3. Result type สำหรับ error handling
type Result[T any] struct {
    value T
    err   error
}

func OK[T any](v T) Result[T] {
    return Result[T]{value: v}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOK() bool { return r.err == nil }
func (r Result[T]) Value() T   { return r.value }
func (r Result[T]) Error() error { return r.err }

func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic(r.err)
    }
    return r.value
}

// ไม่ควรใช้ Generics เมื่อ:
// 1. ใช้ interface พอเพียง
// 2. Code ทำงานกับ type เดียว
// 3. Logic แตกต่างกันตาม type (ควรใช้ interface กับ method แทน)

// ตัวอย่าง: ไม่ควรใช้ generic
type Stringer interface {
    String() string
}

func PrintAll(items []Stringer) { // ใช้ interface ดีกว่า
    for _, item := range items {
        fmt.Println(item.String())
    }
}

func main() {
    // Contains
    fmt.Println("Contains:")
    fmt.Println(Contains([]int{1, 2, 3}, 2))
    fmt.Println(Contains([]string{"a", "b", "c"}, "b"))
    
    // Optional
    fmt.Println("\nOptional:")
    opt1 := Some(42)
    opt2 := None[int]()
    
    if v, ok := opt1.Get(); ok {
        fmt.Println("Got:", v)
    }
    
    fmt.Println("OrElse:", opt1.OrElse(0))
    fmt.Println("OrElse empty:", opt2.OrElse(-1))
    
    // Result
    fmt.Println("\nResult:")
    r1 := OK(42)
    r2 := Err[int](fmt.Errorf("something failed"))
    
    fmt.Println("r1 ok:", r1.IsOK(), "value:", r1.Value())
    fmt.Println("r2 ok:", r2.IsOK(), "err:", r2.Error())
    
    // Map over Result
    doubled := func(r Result[int]) Result[int] {
        if !r.IsOK() {
            return r
        }
        return OK(r.Value() * 2)
    }
    
    fmt.Println("Doubled r1:", doubled(r1).Value())
}
```

---

## 23.8 Workshop: Generic Utils Library

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

// Slice utilities
func Chunk[T any](slice []T, size int) [][]T {
    if size <= 0 {
        return nil
    }
    
    var chunks [][]T
    for i := 0; i < len(slice); i += size {
        end := i + size
        if end > len(slice) {
            end = len(slice)
        }
        chunks = append(chunks, slice[i:end])
    }
    return chunks
}

func Flatten[T any](slices [][]T) []T {
    var result []T
    for _, s := range slices {
        result = append(result, s...)
    }
    return result
}

func Reverse[T any](slice []T) []T {
    result := make([]T, len(slice))
    for i, v := range slice {
        result[len(slice)-1-i] = v
    }
    return result
}

func Take[T any](slice []T, n int) []T {
    if n > len(slice) {
        n = len(slice)
    }
    return slice[:n]
}

func Drop[T any](slice []T, n int) []T {
    if n > len(slice) {
        return nil
    }
    return slice[n:]
}

func TakeWhile[T any](slice []T, pred func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if !pred(v) {
            break
        }
        result = append(result, v)
    }
    return result
}

func DropWhile[T any](slice []T, pred func(T) bool) []T {
    for i, v := range slice {
        if !pred(v) {
            return slice[i:]
        }
    }
    return nil
}

func ForEach[T any](slice []T, f func(T)) {
    for _, v := range slice {
        f(v)
    }
}

func Partition[T any](slice []T, pred func(T) bool) ([]T, []T) {
    var yes, no []T
    for _, v := range slice {
        if pred(v) {
            yes = append(yes, v)
        } else {
            no = append(no, v)
        }
    }
    return yes, no
}

func All[T any](slice []T, pred func(T) bool) bool {
    for _, v := range slice {
        if !pred(v) {
            return false
        }
    }
    return true
}

func Any[T any](slice []T, pred func(T) bool) bool {
    for _, v := range slice {
        if pred(v) {
            return true
        }
    }
    return false
}

func None2[T any](slice []T, pred func(T) bool) bool {
    return !Any(slice, pred)
}

func SortBy[T any](slice []T, less func(a, b T) bool) []T {
    result := make([]T, len(slice))
    copy(result, slice)
    sort.Slice(result, func(i, j int) bool {
        return less(result[i], result[j])
    })
    return result
}

// Map utilities
func MergeMap[K comparable, V any](maps ...map[K]V) map[K]V {
    result := make(map[K]V)
    for _, m := range maps {
        for k, v := range m {
            result[k] = v
        }
    }
    return result
}

func FilterMap[K comparable, V any](m map[K]V, pred func(K, V) bool) map[K]V {
    result := make(map[K]V)
    for k, v := range m {
        if pred(k, v) {
            result[k] = v
        }
    }
    return result
}

func MapValues[K comparable, V, U any](m map[K]V, f func(V) U) map[K]U {
    result := make(map[K]U, len(m))
    for k, v := range m {
        result[k] = f(v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    // Chunk
    chunks := Chunk(nums, 3)
    fmt.Println("Chunks:", chunks)
    
    // Flatten
    nested := [][]int{{1, 2}, {3, 4}, {5, 6}}
    flat := Flatten(nested)
    fmt.Println("Flatten:", flat)
    
    // Reverse
    fmt.Println("Reverse:", Reverse(nums))
    
    // Take/Drop
    fmt.Println("Take 3:", Take(nums, 3))
    fmt.Println("Drop 7:", Drop(nums, 7))
    
    // TakeWhile/DropWhile
    fmt.Println("TakeWhile <5:", TakeWhile(nums, func(n int) bool { return n < 5 }))
    fmt.Println("DropWhile <5:", DropWhile(nums, func(n int) bool { return n < 5 }))
    
    // Partition
    evens, odds := Partition(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println("Evens:", evens)
    fmt.Println("Odds:", odds)
    
    // All/Any/None
    fmt.Println("All >0:", All(nums, func(n int) bool { return n > 0 }))
    fmt.Println("Any >9:", Any(nums, func(n int) bool { return n > 9 }))
    fmt.Println("None <0:", None2(nums, func(n int) bool { return n < 0 }))
    
    // SortBy
    words := []string{"banana", "apple", "cherry", "date"}
    byLen := SortBy(words, func(a, b string) bool { return len(a) < len(b) })
    fmt.Println("Sort by length:", byLen)
    
    // Map utils
    m1 := map[string]int{"a": 1, "b": 2}
    m2 := map[string]int{"b": 20, "c": 3}
    merged := MergeMap(m1, m2)
    fmt.Println("Merged:", merged)
    
    // FilterMap
    bigNums := FilterMap(merged, func(k string, v int) bool { return v > 2 })
    fmt.Println("Filter >2:", bigNums)
    
    // MapValues
    doubled := MapValues(merged, func(v int) int { return v * 2 })
    fmt.Println("Doubled values:", doubled)
    
    // Complex example: process student data
    type Student struct {
        Name  string
        Score int
        Grade string
    }
    
    students := []Student{
        {"Alice", 95, "A"},
        {"Bob", 72, "B"},
        {"Charlie", 85, "A"},
        {"Dave", 60, "C"},
        {"Eve", 88, "A"},
    }
    
    // Get A students sorted by score
    aStudents := SortBy(
        Filter(students, func(s Student) bool { return s.Grade == "A" }),
        func(a, b Student) bool { return a.Score > b.Score },
    )
    
    fmt.Println("\n=== A Students (sorted) ===")
    ForEach(aStudents, func(s Student) {
        fmt.Printf("  %s: %d\n", s.Name, s.Score)
    })
    
    // Stats
    scores := Map(students, func(s Student) int { return s.Score })
    total := Reduce(scores, 0, func(acc, n int) int { return acc + n })
    avg := float64(total) / float64(len(scores))
    fmt.Printf("\nAverage score: %.1f\n", avg)
    fmt.Printf("Passing (>=70): %d students\n",
        len(Filter(students, func(s Student) bool { return s.Score >= 70 })))
    
    _ = strings.Join // suppress import
}

func Filter[T any](slice []T, pred func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

func Reduce[T, U any](slice []T, initial U, f func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = f(result, v)
    }
    return result
}
```

---

## สรุป

| Concept | Syntax | การใช้งาน |
|---------|--------|----------|
| Type Parameter | `[T any]` | Generic function/type |
| Type Constraint | `T comparable` | จำกัด types ที่อนุญาต |
| Union Constraint | `int \| float64` | หลาย types |
| Underlying Constraint | `~int` | int + types based on int |
| Type Inference | อัตโนมัติ | ไม่ต้องระบุ type |

### เมื่อไหรใช้ Generics vs Interface

| ใช้ Generics | ใช้ Interface |
|-------------|--------------|
| Data structures | Behavior abstraction |
| Algorithm (sort, filter) | Plugin pattern |
| Type-safe containers | Polymorphism |
| Math operations | Event handling |

## Resources

- [Go Generics Tutorial](https://go.dev/doc/tutorial/generics)
- [Type Parameters Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)
- [golang.org/x/exp/constraints](https://pkg.go.dev/golang.org/x/exp/constraints)
- [When to use generics](https://go.dev/blog/when-generics)
