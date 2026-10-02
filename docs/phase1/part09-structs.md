# Part 9: Structs ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ประกาศและใช้งาน struct ได้อย่างถูกต้อง
- เข้าใจวิธีการ initialize struct แบบต่างๆ
- ใช้ nested structs และ anonymous structs
- เข้าใจ struct embedding สำหรับ composition
- ใช้ struct tags สำหรับ JSON, database, และ validation
- เขียน constructor functions
- เข้าใจความแตกต่างระหว่าง value และ pointer ใน struct

---

## 9.1 Struct คืออะไร?

**Struct** (Structure) คือ data type ที่ใช้รวบรวมข้อมูลที่เกี่ยวข้องกันไว้ในที่เดียว คล้ายกับ class ในภาษาอื่นๆ แต่ Go ไม่มี class - ใช้ struct แทน

Struct ช่วยให้เราสร้าง **custom types** ที่แสดงถึงสิ่งต่างๆ ในโลกจริงได้

```go
// ตัวอย่าง: แทนที่จะเก็บข้อมูล user แยกกัน
name := "สมชาย"
age := 25
email := "somchai@example.com"

// ใช้ struct รวมกัน
type User struct {
    Name  string
    Age   int
    Email string
}
```

---

## 9.2 การประกาศ Struct

### รูปแบบพื้นฐาน

```go
package main

import "fmt"

// ประกาศ struct นอก function (package level)
type Person struct {
    Name string
    Age  int
    City string
}

func main() {
    // ใช้งาน struct
    var p Person
    p.Name = "สมชาย"
    p.Age = 25
    p.City = "กรุงเทพฯ"
    
    fmt.Println(p)
    fmt.Printf("ชื่อ: %s, อายุ: %d, เมือง: %s\n", p.Name, p.Age, p.City)
}
```

### ตัวอย่าง: Product Struct

```go
package main

import "fmt"

type Product struct {
    ID       int
    Name     string
    Price    float64
    InStock  bool
    Category string
}

func main() {
    var product Product
    product.ID = 1
    product.Name = "MacBook Pro"
    product.Price = 59900.00
    product.InStock = true
    product.Category = "Electronics"
    
    fmt.Printf("สินค้า: %s\n", product.Name)
    fmt.Printf("ราคา: %.2f บาท\n", product.Price)
    fmt.Printf("มีสินค้า: %v\n", product.InStock)
}
```

---

## 9.3 Struct Initialization แบบต่างๆ

### 1. Zero Value Initialization

```go
package main

import "fmt"

type Rectangle struct {
    Width  float64
    Height float64
}

func main() {
    // Zero value - ทุก field เป็น zero value ของ type นั้น
    var r Rectangle
    fmt.Printf("Width: %f, Height: %f\n", r.Width, r.Height)
    // Output: Width: 0.000000, Height: 0.000000
}
```

### 2. Named Initialization (แนะนำ)

```go
package main

import "fmt"

type Student struct {
    Name   string
    Grade  int
    Score  float64
    Active bool
}

func main() {
    // Named initialization - ระบุชื่อ field
    s := Student{
        Name:   "สมหญิง ใจดี",
        Grade:  3,
        Score:  85.5,
        Active: true,
    }
    
    fmt.Printf("นักเรียน: %s\n", s.Name)
    fmt.Printf("ชั้น: %d, คะแนน: %.1f\n", s.Grade, s.Score)
}
```

### 3. Positional Initialization

```go
package main

import "fmt"

type Color struct {
    R, G, B uint8
}

func main() {
    // Positional - ต้องระบุครบทุก field ตามลำดับ
    red := Color{255, 0, 0}
    green := Color{0, 255, 0}
    blue := Color{0, 0, 255}
    
    fmt.Println("Red:", red)
    fmt.Println("Green:", green)
    fmt.Println("Blue:", blue)
}
```

### 4. Partial Initialization

```go
package main

import "fmt"

type Config struct {
    Host     string
    Port     int
    Debug    bool
    MaxConns int
}

func main() {
    // ระบุแค่บาง field - field ที่เหลือเป็น zero value
    cfg := Config{
        Host: "localhost",
        Port: 8080,
        // Debug และ MaxConns จะเป็น false และ 0
    }
    
    fmt.Printf("Host: %s, Port: %d\n", cfg.Host, cfg.Port)
    fmt.Printf("Debug: %v, MaxConns: %d\n", cfg.Debug, cfg.MaxConns)
}
```

---

## 9.4 การเข้าถึงและแก้ไข Fields

```go
package main

import "fmt"

type BankAccount struct {
    AccountNumber string
    Owner         string
    Balance       float64
    IsActive      bool
}

func main() {
    account := BankAccount{
        AccountNumber: "1234567890",
        Owner:         "นายสมศักดิ์ เจริญ",
        Balance:       50000.00,
        IsActive:      true,
    }
    
    // อ่านค่า
    fmt.Printf("เลขบัญชี: %s\n", account.AccountNumber)
    fmt.Printf("เจ้าของ: %s\n", account.Owner)
    fmt.Printf("ยอดเงิน: %.2f บาท\n", account.Balance)
    
    // แก้ไขค่า
    account.Balance += 10000.00
    fmt.Printf("ยอดเงินใหม่: %.2f บาท\n", account.Balance)
    
    // ปิดบัญชี
    account.IsActive = false
    fmt.Printf("สถานะบัญชี: %v\n", account.IsActive)
}
```

---

## 9.5 Nested Structs

Struct ซ้อนใน struct ได้ ใช้สำหรับแสดงความสัมพันธ์ที่ซับซ้อน

```go
package main

import "fmt"

type Address struct {
    Street   string
    District string
    Province string
    PostCode string
}

type ContactInfo struct {
    Phone   string
    Email   string
    Line    string
}

type Employee struct {
    ID          int
    FirstName   string
    LastName    string
    Address     Address
    Contact     ContactInfo
    Department  string
    Salary      float64
}

func main() {
    emp := Employee{
        ID:        1001,
        FirstName: "สมชาย",
        LastName:  "รักไทย",
        Address: Address{
            Street:   "123 ถนนสุขุมวิท",
            District: "วัฒนา",
            Province: "กรุงเทพฯ",
            PostCode: "10110",
        },
        Contact: ContactInfo{
            Phone: "081-234-5678",
            Email: "somchai@company.com",
            Line:  "@somchai",
        },
        Department: "IT",
        Salary:     45000.00,
    }
    
    // เข้าถึง nested fields
    fmt.Printf("พนักงาน: %s %s\n", emp.FirstName, emp.LastName)
    fmt.Printf("แผนก: %s\n", emp.Department)
    fmt.Printf("ที่อยู่: %s, %s, %s %s\n",
        emp.Address.Street,
        emp.Address.District,
        emp.Address.Province,
        emp.Address.PostCode)
    fmt.Printf("โทรศัพท์: %s\n", emp.Contact.Phone)
    fmt.Printf("อีเมล: %s\n", emp.Contact.Email)
    
    // แก้ไข nested field
    emp.Address.Province = "เชียงใหม่"
    emp.Contact.Phone = "089-999-8888"
    fmt.Printf("\nอัปเดตที่อยู่: %s\n", emp.Address.Province)
}
```

### ตัวอย่าง: Order System

```go
package main

import (
    "fmt"
    "time"
)

type Customer struct {
    ID    int
    Name  string
    Email string
}

type Item struct {
    ProductID int
    Name      string
    Quantity  int
    Price     float64
}

type Order struct {
    OrderID    string
    Customer   Customer
    Items      []Item
    OrderDate  time.Time
    TotalPrice float64
    Status     string
}

func (o *Order) CalculateTotal() {
    total := 0.0
    for _, item := range o.Items {
        total += float64(item.Quantity) * item.Price
    }
    o.TotalPrice = total
}

func main() {
    order := Order{
        OrderID: "ORD-2024-001",
        Customer: Customer{
            ID:    1,
            Name:  "สมหญิง ดีใจ",
            Email: "somying@example.com",
        },
        Items: []Item{
            {ProductID: 101, Name: "หนังสือ Go Programming", Quantity: 2, Price: 350.00},
            {ProductID: 102, Name: "USB Hub", Quantity: 1, Price: 890.00},
            {ProductID: 103, Name: "Mouse Pad", Quantity: 3, Price: 150.00},
        },
        OrderDate: time.Now(),
        Status:    "pending",
    }
    
    order.CalculateTotal()
    
    fmt.Printf("คำสั่งซื้อ: %s\n", order.OrderID)
    fmt.Printf("ลูกค้า: %s\n", order.Customer.Name)
    fmt.Printf("วันที่: %s\n", order.OrderDate.Format("02/01/2006"))
    fmt.Println("\nรายการสินค้า:")
    for i, item := range order.Items {
        fmt.Printf("  %d. %s x%d = %.2f บาท\n",
            i+1, item.Name, item.Quantity,
            float64(item.Quantity)*item.Price)
    }
    fmt.Printf("\nยอดรวม: %.2f บาท\n", order.TotalPrice)
    fmt.Printf("สถานะ: %s\n", order.Status)
}
```

---

## 9.6 Anonymous Structs

Anonymous struct คือ struct ที่ไม่มีชื่อ type ใช้สำหรับสร้าง struct ชั่วคราว

```go
package main

import "fmt"

func main() {
    // Anonymous struct
    person := struct {
        Name string
        Age  int
    }{
        Name: "สมชาย",
        Age:  30,
    }
    
    fmt.Printf("ชื่อ: %s, อายุ: %d\n", person.Name, person.Age)
    
    // Slice of anonymous structs
    cities := []struct {
        Name       string
        Population int
    }{
        {"กรุงเทพฯ", 10539000},
        {"เชียงใหม่", 1650000},
        {"ขอนแก่น", 1990000},
        {"สงขลา", 1440000},
    }
    
    fmt.Println("\nประชากรในเมืองใหญ่:")
    for _, city := range cities {
        fmt.Printf("  %s: %,d คน\n", city.Name, city.Population)
    }
}
```

### Anonymous Struct ใน Testing

```go
package main

import "fmt"

func Add(a, b int) int {
    return a + b
}

func main() {
    // Test cases เป็น anonymous struct slice - รูปแบบที่นิยมใน Go testing
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"mixed", -5, 10, 5},
        {"zeros", 0, 0, 0},
    }
    
    for _, tt := range tests {
        result := Add(tt.a, tt.b)
        status := "PASS"
        if result != tt.expected {
            status = "FAIL"
        }
        fmt.Printf("[%s] %s: Add(%d, %d) = %d (expected %d)\n",
            status, tt.name, tt.a, tt.b, result, tt.expected)
    }
}
```

---

## 9.7 Struct Embedding (Composition)

Go ใช้ **embedding** แทน inheritance ทำให้ struct หนึ่งสามารถ "สืบทอด" fields และ methods จาก struct อื่นได้

### Basic Embedding

```go
package main

import "fmt"

// Base struct
type Animal struct {
    Name string
    Age  int
}

func (a Animal) Speak() string {
    return a.Name + " พูดว่าอะไรบางอย่าง"
}

func (a Animal) Info() string {
    return fmt.Sprintf("ชื่อ: %s, อายุ: %d ปี", a.Name, a.Age)
}

// Dog embeds Animal
type Dog struct {
    Animal       // embedded struct
    Breed string
}

func (d Dog) Speak() string {
    return d.Name + " โห่งๆ!" // override method
}

// Cat embeds Animal
type Cat struct {
    Animal
    Indoor bool
}

func (c Cat) Speak() string {
    return c.Name + " เมี้ยวๆ!"
}

func main() {
    dog := Dog{
        Animal: Animal{Name: "บักหมา", Age: 3},
        Breed:  "Golden Retriever",
    }
    
    cat := Cat{
        Animal: Animal{Name: "แมวส้ม", Age: 2},
        Indoor: true,
    }
    
    // เข้าถึง embedded fields โดยตรง
    fmt.Println(dog.Name)  // เหมือนกับ dog.Animal.Name
    fmt.Println(dog.Age)
    fmt.Println(dog.Breed)
    
    // เรียก methods
    fmt.Println(dog.Speak())  // override method
    fmt.Println(dog.Info())   // inherited method
    
    fmt.Println()
    fmt.Println(cat.Name)
    fmt.Println(cat.Speak())
    fmt.Printf("แมวในบ้าน: %v\n", cat.Indoor)
}
```

### Multiple Embedding

```go
package main

import "fmt"

type Timestamped struct {
    CreatedAt string
    UpdatedAt string
}

type Identifiable struct {
    ID   int
    UUID string
}

type SoftDelete struct {
    DeletedAt string
    IsDeleted bool
}

// Post embeds multiple structs
type Post struct {
    Timestamped
    Identifiable
    SoftDelete
    Title   string
    Content string
    Author  string
}

func main() {
    post := Post{
        Identifiable: Identifiable{
            ID:   1,
            UUID: "abc-123-def",
        },
        Timestamped: Timestamped{
            CreatedAt: "2024-01-15",
            UpdatedAt: "2024-01-20",
        },
        Title:   "บทความ Go Programming",
        Content: "Go เป็นภาษาที่ยอดเยี่ยม...",
        Author:  "สมชาย เขียนดี",
    }
    
    // เข้าถึง fields จาก embedded structs
    fmt.Printf("ID: %d, UUID: %s\n", post.ID, post.UUID)
    fmt.Printf("Title: %s\n", post.Title)
    fmt.Printf("สร้างเมื่อ: %s\n", post.CreatedAt)
    fmt.Printf("แก้ไขล่าสุด: %s\n", post.UpdatedAt)
    fmt.Printf("ลบแล้ว: %v\n", post.IsDeleted)
}
```

### Embedding Interface

```go
package main

import "fmt"

type Logger interface {
    Log(message string)
}

type ConsoleLogger struct {
    Prefix string
}

func (cl ConsoleLogger) Log(message string) {
    fmt.Printf("[%s] %s\n", cl.Prefix, message)
}

type Service struct {
    Logger        // embed interface
    Name   string
}

func main() {
    svc := Service{
        Logger: ConsoleLogger{Prefix: "SERVICE"},
        Name:   "UserService",
    }
    
    svc.Log("เริ่มต้น service")
    svc.Log(fmt.Sprintf("Service %s พร้อมทำงาน", svc.Name))
}
```

---

## 9.8 Struct Tags

Struct tags คือ metadata ที่แนบกับ struct fields ใช้โดย library ต่างๆ เช่น JSON encoding, ORM, validation

### JSON Tags

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID        int    `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Email     string `json:"email"`
    Password  string `json:"-"`          // ไม่ส่ง password ใน JSON
    Age       int    `json:"age,omitempty"` // ถ้าเป็น 0 จะไม่ส่ง
    Admin     bool   `json:"is_admin"`
}

func main() {
    user := User{
        ID:        1,
        FirstName: "สมชาย",
        LastName:  "ใจดี",
        Email:     "somchai@example.com",
        Password:  "secret123",
        Age:       0,      // จะไม่ปรากฏใน JSON เพราะ omitempty
        Admin:     false,
    }
    
    // Marshal (struct -> JSON)
    jsonData, err := json.MarshalIndent(user, "", "  ")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("JSON output:")
    fmt.Println(string(jsonData))
    
    // Unmarshal (JSON -> struct)
    jsonStr := `{
        "id": 2,
        "first_name": "สมหญิง",
        "last_name": "รักเรียน",
        "email": "somying@example.com",
        "age": 25,
        "is_admin": true
    }`
    
    var user2 User
    err = json.Unmarshal([]byte(jsonStr), &user2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("\nอ่านจาก JSON: %+v\n", user2)
}
```

### Database Tags

```go
package main

import "fmt"

// ตัวอย่าง struct tags สำหรับ database (เช่น GORM)
type Article struct {
    ID        uint   `gorm:"primaryKey;autoIncrement"`
    Title     string `gorm:"type:varchar(200);not null"`
    Content   string `gorm:"type:text"`
    AuthorID  uint   `gorm:"index"`
    Published bool   `gorm:"default:false"`
    ViewCount int    `gorm:"default:0"`
    CreatedAt string `gorm:"autoCreateTime"`
    UpdatedAt string `gorm:"autoUpdateTime"`
}

// ตัวอย่าง struct tags สำหรับ validation
type RegisterForm struct {
    Username string `json:"username" validate:"required,min=3,max=20,alphanum"`
    Email    string `json:"email"    validate:"required,email"`
    Password string `json:"password" validate:"required,min=8,max=100"`
    Age      int    `json:"age"      validate:"required,min=18,max=120"`
    Phone    string `json:"phone"    validate:"required,e164"`
}

func main() {
    article := Article{
        Title:   "บทความ Go",
        Content: "เนื้อหาบทความ...",
    }
    
    fmt.Printf("Article: %+v\n", article)
    
    form := RegisterForm{
        Username: "somchai123",
        Email:    "somchai@example.com",
        Password: "mypassword123",
        Age:      25,
        Phone:    "+66812345678",
    }
    
    fmt.Printf("Form: %+v\n", form)
}
```

### Custom Struct Tags

```go
package main

import (
    "fmt"
    "reflect"
)

type FormField struct {
    Name     string `label:"ชื่อ" required:"true" maxlen:"50"`
    Email    string `label:"อีเมล" required:"true" format:"email"`
    Age      int    `label:"อายุ" required:"false" min:"0" max:"150"`
}

// อ่าน struct tags ด้วย reflection
func printFieldInfo(v interface{}) {
    t := reflect.TypeOf(v)
    
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        label := field.Tag.Get("label")
        required := field.Tag.Get("required")
        
        fmt.Printf("Field: %-10s | Label: %-10s | Required: %s\n",
            field.Name, label, required)
    }
}

func main() {
    form := FormField{
        Name:  "สมชาย",
        Email: "somchai@example.com",
        Age:   25,
    }
    
    fmt.Println("ข้อมูล Form Fields:")
    fmt.Println("===================")
    printFieldInfo(form)
}
```

---

## 9.9 การเปรียบเทียบ Structs

Structs ใน Go สามารถเปรียบเทียบได้ด้วย `==` และ `!=` ถ้า fields ทั้งหมดสามารถเปรียบเทียบได้

```go
package main

import "fmt"

type Point struct {
    X, Y int
}

type Rectangle struct {
    TopLeft     Point
    BottomRight Point
}

func main() {
    // เปรียบเทียบ structs
    p1 := Point{1, 2}
    p2 := Point{1, 2}
    p3 := Point{3, 4}
    
    fmt.Printf("p1 == p2: %v\n", p1 == p2) // true
    fmt.Printf("p1 == p3: %v\n", p1 == p3) // false
    fmt.Printf("p1 != p3: %v\n", p1 != p3) // true
    
    // Nested struct comparison
    r1 := Rectangle{Point{0, 0}, Point{10, 10}}
    r2 := Rectangle{Point{0, 0}, Point{10, 10}}
    r3 := Rectangle{Point{0, 0}, Point{20, 20}}
    
    fmt.Printf("\nr1 == r2: %v\n", r1 == r2) // true
    fmt.Printf("r1 == r3: %v\n", r1 == r3) // false
}
```

### Structs ที่ไม่สามารถเปรียบเทียบได้

```go
package main

import "fmt"

// Struct ที่มี slice ไม่สามารถใช้ == ได้
type Team struct {
    Name    string
    Members []string // slice ทำให้เปรียบเทียบไม่ได้
}

// ต้องใช้ reflect.DeepEqual หรือเขียน function เอง
import "reflect"

func TeamsEqual(t1, t2 Team) bool {
    if t1.Name != t2.Name {
        return false
    }
    return reflect.DeepEqual(t1.Members, t2.Members)
}

func main() {
    t1 := Team{Name: "Dev", Members: []string{"สมชาย", "สมหญิง"}}
    t2 := Team{Name: "Dev", Members: []string{"สมชาย", "สมหญิง"}}
    
    // t1 == t2 จะ compile error!
    // แต่ใช้ function ได้
    fmt.Println("Teams equal:", TeamsEqual(t1, t2))
}
```

---

## 9.10 Pointer to Struct

```go
package main

import "fmt"

type Counter struct {
    Value int
    Name  string
}

// ฟังก์ชันรับ pointer to struct - แก้ไขค่าได้
func Increment(c *Counter) {
    c.Value++
}

// ฟังก์ชันรับ value - แก้ไขค่าไม่ได้ (copy)
func IncrementCopy(c Counter) {
    c.Value++ // แก้แค่ copy ไม่กระทบต้นฉบับ
}

func main() {
    c := Counter{Value: 0, Name: "ตัวนับ"}
    
    // แบบ value
    IncrementCopy(c)
    fmt.Printf("หลัง IncrementCopy: %d\n", c.Value) // ยังเป็น 0
    
    // แบบ pointer
    Increment(&c)
    Increment(&c)
    Increment(&c)
    fmt.Printf("หลัง Increment x3: %d\n", c.Value) // เป็น 3
    
    // สร้าง pointer to struct โดยตรง
    p := &Counter{Value: 100, Name: "pointer counter"}
    fmt.Printf("\nPointer counter: %+v\n", *p)
    
    // Go อนุญาตให้ใช้ . กับ pointer ได้เลย (auto-dereference)
    p.Value = 200  // เหมือนกับ (*p).Value = 200
    fmt.Printf("Updated: %d\n", p.Value)
}
```

---

## 9.11 Constructor Functions

Go ไม่มี constructor แบบ OOP แต่เราสร้าง function ที่ทำหน้าที่เหมือน constructor ได้

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

type User struct {
    ID        int
    Username  string
    Email     string
    CreatedAt time.Time
    IsActive  bool
}

// Constructor function - ชื่อขึ้นต้นด้วย New
func NewUser(id int, username, email string) *User {
    return &User{
        ID:        id,
        Username:  strings.ToLower(username),
        Email:     strings.ToLower(email),
        CreatedAt: time.Now(),
        IsActive:  true,
    }
}

type Config struct {
    Host     string
    Port     int
    Debug    bool
    Timeout  int
    MaxConns int
}

// Constructor with options pattern
type Option func(*Config)

func WithHost(host string) Option {
    return func(c *Config) {
        c.Host = host
    }
}

func WithPort(port int) Option {
    return func(c *Config) {
        c.Port = port
    }
}

func WithDebug(debug bool) Option {
    return func(c *Config) {
        c.Debug = debug
    }
}

func NewConfig(opts ...Option) *Config {
    // Default values
    cfg := &Config{
        Host:     "localhost",
        Port:     8080,
        Debug:    false,
        Timeout:  30,
        MaxConns: 100,
    }
    
    // Apply options
    for _, opt := range opts {
        opt(cfg)
    }
    
    return cfg
}

func main() {
    // Basic constructor
    user := NewUser(1, "SOMCHAI", "SOMCHAI@EXAMPLE.COM")
    fmt.Printf("User: %+v\n", user)
    fmt.Printf("Created: %s\n", user.CreatedAt.Format("2006-01-02 15:04:05"))
    
    // Options pattern
    cfg1 := NewConfig() // default config
    cfg2 := NewConfig(
        WithHost("production.server.com"),
        WithPort(443),
        WithDebug(false),
    )
    
    fmt.Printf("\nDefault Config: %+v\n", cfg1)
    fmt.Printf("Production Config: %+v\n", cfg2)
}
```

---

## 9.12 Struct Methods

```go
package main

import (
    "fmt"
    "math"
)

type Circle struct {
    Radius float64
    Color  string
}

// Value receiver method
func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

func (c Circle) String() string {
    return fmt.Sprintf("Circle(radius=%.2f, color=%s)", c.Radius, c.Color)
}

// Pointer receiver method (แก้ไข struct ได้)
func (c *Circle) Scale(factor float64) {
    c.Radius *= factor
}

func main() {
    c := Circle{Radius: 5.0, Color: "red"}
    
    fmt.Println(c)
    fmt.Printf("พื้นที่: %.2f\n", c.Area())
    fmt.Printf("เส้นรอบวง: %.2f\n", c.Perimeter())
    
    c.Scale(2)
    fmt.Printf("\nหลังขยาย 2 เท่า:\n")
    fmt.Println(c)
    fmt.Printf("พื้นที่ใหม่: %.2f\n", c.Area())
}
```

---

## 9.13 Struct Copying

```go
package main

import "fmt"

type Settings struct {
    Theme    string
    FontSize int
    Language string
}

func main() {
    original := Settings{
        Theme:    "dark",
        FontSize: 14,
        Language: "th",
    }
    
    // Copy struct (shallow copy)
    copy1 := original
    copy1.Theme = "light"  // ไม่กระทบ original
    
    fmt.Printf("Original: %+v\n", original)
    fmt.Printf("Copy1: %+v\n", copy1)
    
    // Pointer - ชี้ไปที่เดียวกัน
    ptr := &original
    ptr.Theme = "system"  // กระทบ original!
    
    fmt.Printf("\nหลังแก้ผ่าน pointer:\n")
    fmt.Printf("Original: %+v\n", original)
    fmt.Printf("Ptr: %+v\n", *ptr)
}
```

---

## 9.14 Struct Patterns ที่ใช้บ่อย

### Value Object Pattern

```go
package main

import (
    "fmt"
    "strings"
)

type Email struct {
    value string
}

func NewEmail(email string) (Email, error) {
    email = strings.TrimSpace(strings.ToLower(email))
    if !strings.Contains(email, "@") {
        return Email{}, fmt.Errorf("invalid email: %s", email)
    }
    return Email{value: email}, nil
}

func (e Email) String() string {
    return e.value
}

func (e Email) Domain() string {
    parts := strings.Split(e.value, "@")
    if len(parts) != 2 {
        return ""
    }
    return parts[1]
}

type Money struct {
    amount   int64 // ใน satang (0.01 บาท)
    currency string
}

func NewMoney(baht float64, currency string) Money {
    return Money{
        amount:   int64(baht * 100),
        currency: currency,
    }
}

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, fmt.Errorf("currency mismatch: %s vs %s", m.currency, other.currency)
    }
    return Money{amount: m.amount + other.amount, currency: m.currency}, nil
}

func (m Money) String() string {
    return fmt.Sprintf("%.2f %s", float64(m.amount)/100, m.currency)
}

func main() {
    email, err := NewEmail("  SOMCHAI@EXAMPLE.COM  ")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Email: %s, Domain: %s\n", email, email.Domain())
    
    price := NewMoney(150.50, "THB")
    tax := NewMoney(10.54, "THB")
    
    total, err := price.Add(tax)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Price: %s\n", price)
    fmt.Printf("Tax: %s\n", tax)
    fmt.Printf("Total: %s\n", total)
}
```

---

## Workshop: User Profile System

สร้างระบบจัดการ User Profile ที่ครบครัน

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
    "time"
)

// Types
type Role string

const (
    RoleAdmin   Role = "admin"
    RoleUser    Role = "user"
    RoleMod     Role = "moderator"
)

type Address struct {
    Street   string `json:"street"`
    District string `json:"district"`
    Province string `json:"province"`
    PostCode string `json:"post_code"`
    Country  string `json:"country"`
}

type SocialLinks struct {
    Facebook  string `json:"facebook,omitempty"`
    Twitter   string `json:"twitter,omitempty"`
    LinkedIn  string `json:"linkedin,omitempty"`
    GitHub    string `json:"github,omitempty"`
    Website   string `json:"website,omitempty"`
}

type Preferences struct {
    Language     string `json:"language"`
    Theme        string `json:"theme"`
    Timezone     string `json:"timezone"`
    Notifications bool  `json:"notifications"`
}

type Profile struct {
    ID          int         `json:"id"`
    Username    string      `json:"username"`
    Email       string      `json:"email"`
    FirstName   string      `json:"first_name"`
    LastName    string      `json:"last_name"`
    Bio         string      `json:"bio,omitempty"`
    AvatarURL   string      `json:"avatar_url,omitempty"`
    Role        Role        `json:"role"`
    Address     Address     `json:"address"`
    Social      SocialLinks `json:"social"`
    Preferences Preferences `json:"preferences"`
    CreatedAt   time.Time   `json:"created_at"`
    UpdatedAt   time.Time   `json:"updated_at"`
    IsActive    bool        `json:"is_active"`
    IsVerified  bool        `json:"is_verified"`
}

// Constructor
func NewProfile(id int, username, email, firstName, lastName string) *Profile {
    now := time.Now()
    return &Profile{
        ID:        id,
        Username:  strings.ToLower(username),
        Email:     strings.ToLower(email),
        FirstName: firstName,
        LastName:  lastName,
        Role:      RoleUser,
        Preferences: Preferences{
            Language:     "th",
            Theme:        "light",
            Timezone:     "Asia/Bangkok",
            Notifications: true,
        },
        CreatedAt: now,
        UpdatedAt: now,
        IsActive:  true,
    }
}

// Methods
func (p *Profile) FullName() string {
    return fmt.Sprintf("%s %s", p.FirstName, p.LastName)
}

func (p *Profile) UpdateAddress(addr Address) {
    p.Address = addr
    p.UpdatedAt = time.Now()
}

func (p *Profile) AddSocialLink(platform, url string) {
    switch strings.ToLower(platform) {
    case "facebook":
        p.Social.Facebook = url
    case "twitter":
        p.Social.Twitter = url
    case "linkedin":
        p.Social.LinkedIn = url
    case "github":
        p.Social.GitHub = url
    case "website":
        p.Social.Website = url
    }
    p.UpdatedAt = time.Now()
}

func (p *Profile) Promote(role Role) {
    p.Role = role
    p.UpdatedAt = time.Now()
}

func (p *Profile) Verify() {
    p.IsVerified = true
    p.UpdatedAt = time.Now()
}

func (p *Profile) Deactivate() {
    p.IsActive = false
    p.UpdatedAt = time.Now()
}

func (p Profile) String() string {
    return fmt.Sprintf("@%s (%s) - %s", p.Username, p.Role, p.FullName())
}

func (p Profile) ToJSON() string {
    data, _ := json.MarshalIndent(p, "", "  ")
    return string(data)
}

// Profile Repository (in-memory)
type ProfileRepository struct {
    profiles map[int]*Profile
    nextID   int
}

func NewProfileRepository() *ProfileRepository {
    return &ProfileRepository{
        profiles: make(map[int]*Profile),
        nextID:   1,
    }
}

func (r *ProfileRepository) Create(username, email, firstName, lastName string) *Profile {
    profile := NewProfile(r.nextID, username, email, firstName, lastName)
    r.profiles[r.nextID] = profile
    r.nextID++
    return profile
}

func (r *ProfileRepository) FindByID(id int) (*Profile, bool) {
    profile, exists := r.profiles[id]
    return profile, exists
}

func (r *ProfileRepository) FindByUsername(username string) (*Profile, bool) {
    for _, p := range r.profiles {
        if p.Username == strings.ToLower(username) {
            return p, true
        }
    }
    return nil, false
}

func (r *ProfileRepository) ListActive() []*Profile {
    var active []*Profile
    for _, p := range r.profiles {
        if p.IsActive {
            active = append(active, p)
        }
    }
    return active
}

func (r *ProfileRepository) Delete(id int) bool {
    if _, exists := r.profiles[id]; exists {
        delete(r.profiles, id)
        return true
    }
    return false
}

func main() {
    repo := NewProfileRepository()
    
    // สร้าง profiles
    admin := repo.Create("admin_thai", "admin@example.com", "ผู้ดูแล", "ระบบ")
    user1 := repo.Create("somchai_dev", "somchai@example.com", "สมชาย", "เขียนโค้ด")
    user2 := repo.Create("somying_art", "somying@example.com", "สมหญิง", "ชอบวาด")
    
    // ตั้งค่า admin
    admin.Promote(RoleAdmin)
    admin.Verify()
    admin.Bio = "ผู้ดูแลระบบ"
    
    // ตั้งค่า user1
    user1.Verify()
    user1.Bio = "นักพัฒนา Go ตัวยง"
    user1.UpdateAddress(Address{
        Street:   "123 ถนนสุขุมวิท",
        District: "วัฒนา",
        Province: "กรุงเทพฯ",
        PostCode: "10110",
        Country:  "Thailand",
    })
    user1.AddSocialLink("github", "https://github.com/somchai-dev")
    user1.AddSocialLink("twitter", "@somchai_dev")
    user1.Preferences.Theme = "dark"
    
    // ตั้งค่า user2
    user2.AddSocialLink("website", "https://somying.art")
    
    // แสดงผล
    fmt.Println("=== ระบบ User Profile ===\n")
    
    fmt.Println("รายชื่อ users ทั้งหมด:")
    for _, p := range repo.ListActive() {
        verified := ""
        if p.IsVerified {
            verified = " ✓"
        }
        fmt.Printf("  - %s%s\n", p, verified)
    }
    
    fmt.Println("\n--- Profile Detail: somchai_dev ---")
    if profile, exists := repo.FindByUsername("somchai_dev"); exists {
        fmt.Printf("ชื่อเต็ม: %s\n", profile.FullName())
        fmt.Printf("Email: %s\n", profile.Email)
        fmt.Printf("Bio: %s\n", profile.Bio)
        fmt.Printf("ที่อยู่: %s, %s, %s\n",
            profile.Address.Street,
            profile.Address.Province,
            profile.Address.Country)
        fmt.Printf("GitHub: %s\n", profile.Social.GitHub)
        fmt.Printf("Theme: %s\n", profile.Preferences.Theme)
        fmt.Printf("Verified: %v\n", profile.IsVerified)
    }
    
    fmt.Println("\n--- JSON Output: admin_thai ---")
    if profile, exists := repo.FindByID(1); exists {
        fmt.Println(profile.ToJSON())
    }
    
    // Deactivate user
    user2.Deactivate()
    
    fmt.Printf("\n\nActive users หลัง deactivate somying: %d คน\n",
        len(repo.ListActive()))
}
```

---

## สรุป

| Concept | รายละเอียด |
|---------|-----------|
| Struct | Custom type รวบรวม fields ที่เกี่ยวข้อง |
| Zero Value | Fields เป็น zero value ถ้าไม่ initialize |
| Named Init | แนะนำ - ระบุชื่อ field ชัดเจน |
| Nested Struct | Struct ซ้อนใน struct สำหรับโครงสร้างซับซ้อน |
| Anonymous Struct | Struct ชั่วคราว ไม่มีชื่อ type |
| Embedding | Composition แทน inheritance |
| Struct Tags | Metadata สำหรับ JSON, DB, validation |
| Constructor | Function สร้าง struct พร้อม default values |
| Pointer Receiver | Method ที่แก้ไข struct ได้ |

## Resources

- [Go Spec: Struct types](https://go.dev/ref/spec#Struct_types)
- [Effective Go: Struct embedding](https://go.dev/doc/effective_go#embedding)
- [Go by Example: Structs](https://gobyexample.com/structs)
- [Go Blog: JSON and Go](https://go.dev/blog/json)

---

*Part 9 จบแล้ว! ต่อไป Part 10: Pointers*
