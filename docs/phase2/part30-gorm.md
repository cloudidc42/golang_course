# Part 30: GORM - Go ORM Library

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ติดตั้งและ setup GORM
- นิยาม Models และ associations
- ใช้ AutoMigrate
- ทำ CRUD operations ด้วย GORM
- Query ขั้นสูง (Where, Order, Limit, Offset)
- Preloading associations
- ใช้ Hooks
- จัดการ Transactions
- เขียน Raw queries
- ทำ GORM migrations

---

## 30.1 GORM Setup

```bash
# ติดตั้ง GORM
go get gorm.io/gorm

# ติดตั้ง driver
go get gorm.io/driver/postgres  # PostgreSQL
go get gorm.io/driver/mysql     # MySQL
go get gorm.io/driver/sqlite    # SQLite
```

### Connect ด้วย GORM (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "log"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func main() {
    // SQLite (ง่ายสำหรับการทดสอบ)
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // แสดง SQL logs
    })
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }
    
    fmt.Println("Connected to SQLite with GORM!")
    
    // ดู underlying *sql.DB
    sqlDB, err := db.DB()
    if err != nil {
        log.Fatal(err)
    }
    
    // ตั้งค่า connection pool
    sqlDB.SetMaxOpenConns(10)
    sqlDB.SetMaxIdleConns(5)
    
    // Ping test
    if err := sqlDB.Ping(); err != nil {
        log.Fatal("Ping error:", err)
    }
    
    fmt.Println("Database connection pool configured!")
}
```

### Connect PostgreSQL (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "log"
    "time"
    
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func ConnectPostgres() (*gorm.DB, error) {
    dsn := "host=localhost user=postgres password=secret dbname=myapp port=5432 sslmode=disable TimeZone=Asia/Bangkok"
    
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, fmt.Errorf("connecting to postgres: %w", err)
    }
    
    sqlDB, _ := db.DB()
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    
    return db, nil
}

func main() {
    db, err := ConnectPostgres()
    if err != nil {
        log.Fatal(err)
    }
    
    sqlDB, _ := db.DB()
    defer sqlDB.Close()
    
    fmt.Println("Connected to PostgreSQL!")
    
    _ = db
}
```

---

## 30.2 Model Definition

### Basic Model (ตัวอย่างที่ 3)

```go
package main

import (
    "time"
    
    "gorm.io/gorm"
)

// gorm.Model มี ID, CreatedAt, UpdatedAt, DeletedAt
type User struct {
    gorm.Model           // embed - ได้ ID, CreatedAt, UpdatedAt, DeletedAt
    Name     string      `gorm:"not null"`
    Email    string      `gorm:"uniqueIndex;not null"`
    Age      int
    Role     string      `gorm:"default:user"`
    Active   bool        `gorm:"default:true"`
}

// Model แบบ custom (ไม่ใช้ gorm.Model)
type Product struct {
    ID          uint      `gorm:"primaryKey;autoIncrement"`
    Name        string    `gorm:"size:200;not null"`
    Description string    `gorm:"type:text"`
    Price       float64   `gorm:"not null;check:price > 0"`
    Stock       int       `gorm:"default:0"`
    SKU         string    `gorm:"uniqueIndex;size:50"`
    CategoryID  uint
    CreatedAt   time.Time
    UpdatedAt   time.Time
    DeletedAt   *time.Time `gorm:"index"` // soft delete
}

// TableName กำหนดชื่อ table เอง
func (Product) TableName() string {
    return "products"
}
```

### Model พร้อม Associations (ตัวอย่างที่ 4)

```go
package main

import (
    "time"
    
    "gorm.io/gorm"
)

type Category struct {
    gorm.Model
    Name        string    `gorm:"uniqueIndex;not null"`
    Description string
    Products    []Product `gorm:"foreignKey:CategoryID"` // Has Many
}

type Product struct {
    gorm.Model
    Name        string    `gorm:"not null"`
    Price       float64
    CategoryID  uint
    Category    Category  `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL"` // Belongs To
    Tags        []Tag     `gorm:"many2many:product_tags"` // Many to Many
}

type Tag struct {
    gorm.Model
    Name     string    `gorm:"uniqueIndex"`
    Products []Product `gorm:"many2many:product_tags"`
}

type Order struct {
    gorm.Model
    UserID     uint
    User       User           `gorm:"constraint:OnUpdate:CASCADE,OnDelete:CASCADE"`
    Items      []OrderItem    `gorm:"foreignKey:OrderID"`
    TotalPrice float64
    Status     string         `gorm:"default:pending"`
}

type OrderItem struct {
    gorm.Model
    OrderID   uint
    ProductID uint
    Product   Product `gorm:"constraint:OnUpdate:CASCADE,OnDelete:RESTRICT"`
    Quantity  int
    Price     float64
}

type User struct {
    gorm.Model
    Name      string    `gorm:"not null"`
    Email     string    `gorm:"uniqueIndex;not null"`
    Profile   *Profile  // Has One
    Orders    []Order   `gorm:"foreignKey:UserID"` // Has Many
}

type Profile struct {
    gorm.Model
    UserID    uint   `gorm:"uniqueIndex"`
    Bio       string `gorm:"type:text"`
    Website   string
    AvatarURL string
    CreatedAt time.Time
}
```

---

## 30.3 AutoMigrate

### AutoMigrate (ตัวอย่างที่ 5)

```go
package main

import (
    "fmt"
    "log"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string `gorm:"not null"`
    Email string `gorm:"uniqueIndex;not null"`
    Age   int
}

type Post struct {
    gorm.Model
    Title     string `gorm:"not null"`
    Content   string `gorm:"type:text"`
    Published bool   `gorm:"default:false"`
    UserID    uint
    User      User
}

func main() {
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    
    // AutoMigrate สร้าง/อัพเดท tables อัตโนมัติ
    // หมายเหตุ: ไม่ drop columns! แค่เพิ่มใหม่เท่านั้น
    err = db.AutoMigrate(
        &User{},
        &Post{},
    )
    if err != nil {
        log.Fatal("AutoMigrate error:", err)
    }
    
    fmt.Println("AutoMigrate completed!")
    
    // ตรวจสอบว่า table มีอยู่
    if db.Migrator().HasTable(&User{}) {
        fmt.Println("users table exists!")
    }
    
    // ตรวจสอบ column
    if db.Migrator().HasColumn(&User{}, "email") {
        fmt.Println("email column exists!")
    }
}
```

---

## 30.4 CRUD Operations

### Create (ตัวอย่างที่ 6)

```go
package main

import (
    "fmt"
    "log"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string `gorm:"uniqueIndex"`
    Age   int
    Role  string `gorm:"default:user"`
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{})
    
    // Create single record
    user := User{Name: "Alice", Email: "alice@example.com", Age: 25}
    result := db.Create(&user)
    if result.Error != nil {
        log.Fatal(result.Error)
    }
    fmt.Printf("Created user ID: %d\n", user.ID)
    
    // Create with selected fields only
    user2 := User{Name: "Bob", Email: "bob@example.com", Age: 30}
    db.Select("Name", "Email").Create(&user2)
    fmt.Printf("Created Bob ID: %d (Age not set)\n", user2.ID)
    
    // Create multiple records
    users := []User{
        {Name: "Charlie", Email: "charlie@example.com", Age: 35},
        {Name: "Diana", Email: "diana@example.com", Age: 28},
    }
    db.Create(&users)
    fmt.Printf("Batch created %d users\n", len(users))
    
    // Create with map
    db.Model(&User{}).Create(map[string]interface{}{
        "Name": "Eve", "Email": "eve@example.com", "Age": 22,
    })
    
    fmt.Println("\nAll created successfully!")
}
```

### Read (ตัวอย่างที่ 7)

```go
package main

import (
    "fmt"
    "log"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string
    Age   int
}

func setupDB() *gorm.DB {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{})
    db.Create(&[]User{
        {Name: "Alice", Email: "alice@example.com", Age: 25},
        {Name: "Bob", Email: "bob@example.com", Age: 30},
        {Name: "Charlie", Email: "charlie@example.com", Age: 35},
        {Name: "Diana", Email: "diana@example.com", Age: 25},
    })
    return db
}

func main() {
    db := setupDB()
    
    // Find by primary key
    var user User
    db.First(&user, 1) // WHERE id = 1 ORDER BY id LIMIT 1
    fmt.Printf("First: %s (ID: %d)\n", user.Name, user.ID)
    
    // Find last record
    var last User
    db.Last(&last)
    fmt.Printf("Last: %s\n", last.Name)
    
    // Find by condition
    var alice User
    db.Where("name = ?", "Alice").First(&alice)
    fmt.Printf("Alice: %s (%s)\n", alice.Name, alice.Email)
    
    // Find by struct condition
    var found User
    db.Where(&User{Email: "bob@example.com"}).First(&found)
    fmt.Printf("Found: %s\n", found.Name)
    
    // Find all
    var allUsers []User
    db.Find(&allUsers)
    fmt.Printf("\nAll users (%d):\n", len(allUsers))
    for _, u := range allUsers {
        fmt.Printf("  - %s (age %d)\n", u.Name, u.Age)
    }
    
    // Select specific columns
    type UserSummary struct {
        Name  string
        Email string
    }
    var summaries []UserSummary
    db.Model(&User{}).Select("name", "email").Scan(&summaries)
    fmt.Printf("\nSummaries: %d\n", len(summaries))
    
    // Count
    var count int64
    db.Model(&User{}).Where("age >= ?", 30).Count(&count)
    fmt.Printf("Users age >= 30: %d\n", count)
    
    // Check if exists
    var exists bool
    db.Model(&User{}).Select("count(*) > 0").Where("email = ?", "alice@example.com").Find(&exists)
    fmt.Printf("Alice exists: %v\n", exists)
}
```

### Update (ตัวอย่างที่ 8)

```go
package main

import (
    "fmt"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string
    Age   int
    Role  string
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{})
    
    user := User{Name: "Alice", Email: "alice@example.com", Age: 25, Role: "user"}
    db.Create(&user)
    
    // Save - อัพเดททุก field (รวม zero values!)
    user.Name = "Alice Updated"
    user.Age = 26
    db.Save(&user)
    
    // Updates - อัพเดทเฉพาะ fields ที่กำหนด (ไม่อัพ zero values)
    db.Model(&user).Updates(User{Name: "Alice v2", Age: 27})
    
    // Update ด้วย map (อัพเดท zero values ได้ด้วย!)
    db.Model(&user).Updates(map[string]interface{}{
        "Name": "Alice v3",
        "Age":  0,  // map สามารถ set 0 ได้
        "Role": "admin",
    })
    
    // Update เฉพาะ column
    db.Model(&user).Update("name", "Alice Final")
    
    // Update หลาย records ตาม condition
    db.Model(&User{}).Where("age < ?", 18).Update("role", "minor")
    
    // Increment/Decrement
    db.Model(&user).UpdateColumn("age", gorm.Expr("age + ?", 1))
    
    // Verify
    var updated User
    db.First(&updated, user.ID)
    fmt.Printf("Final: %s, Age: %d, Role: %s\n", updated.Name, updated.Age, updated.Role)
}
```

### Delete (ตัวอย่างที่ 9)

```go
package main

import (
    "fmt"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model // มี DeletedAt -> Soft Delete
    Name  string
    Email string
}

type Log struct {
    ID      uint `gorm:"primaryKey"`
    Message string
    // ไม่มี DeletedAt -> Hard Delete
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{}, &Log{})
    
    // สร้าง test data
    user := User{Name: "Alice", Email: "alice@example.com"}
    db.Create(&user)
    
    log1 := Log{Message: "log entry 1"}
    db.Create(&log1)
    
    // Soft Delete (เนื่องจาก User มี gorm.Model ซึ่งมี DeletedAt)
    db.Delete(&user)
    
    // หลัง soft delete - ไม่เจอ record แล้ว
    var found User
    result := db.First(&found, user.ID)
    if result.Error != nil {
        fmt.Println("User not found (soft deleted):", result.Error)
    }
    
    // ดู soft deleted records
    var deletedUser User
    db.Unscoped().First(&deletedUser, user.ID)
    fmt.Printf("Soft deleted user: %s (deleted at: %v)\n", deletedUser.Name, deletedUser.DeletedAt)
    
    // Hard Delete (Unscoped)
    db.Unscoped().Delete(&user)
    
    // Delete ตาม condition
    db.Where("name LIKE ?", "test_%").Delete(&User{})
    
    // Delete by primary keys
    db.Delete(&User{}, []uint{1, 2, 3})
    
    // Hard Delete สำหรับ struct ที่ไม่มี DeletedAt
    db.Delete(&log1)
    
    fmt.Println("Delete operations completed!")
}
```

---

## 30.5 Querying

### Where Conditions (ตัวอย่างที่ 10)

```go
package main

import (
    "fmt"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Product struct {
    gorm.Model
    Name     string
    Price    float64
    Stock    int
    Category string
    Active   bool
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&Product{})
    db.Create(&[]Product{
        {Name: "Go Book", Price: 499, Stock: 100, Category: "books", Active: true},
        {Name: "Python Book", Price: 399, Stock: 50, Category: "books", Active: true},
        {Name: "Laptop Stand", Price: 1200, Stock: 30, Category: "accessories", Active: true},
        {Name: "Mouse", Price: 500, Stock: 0, Category: "accessories", Active: false},
        {Name: "Keyboard", Price: 1500, Stock: 20, Category: "accessories", Active: true},
    })
    
    var products []Product
    
    // Basic WHERE
    db.Where("price > ?", 500).Find(&products)
    fmt.Printf("Price > 500: %d products\n", len(products))
    
    // AND condition
    db.Where("price > ? AND active = ?", 400, true).Find(&products)
    fmt.Printf("Price > 400 AND active: %d products\n", len(products))
    
    // OR condition
    db.Where("category = ? OR category = ?", "books", "accessories").Find(&products)
    fmt.Printf("Books OR accessories: %d products\n", len(products))
    
    // IN
    db.Where("category IN ?", []string{"books", "accessories"}).Find(&products)
    fmt.Printf("IN (books, accessories): %d products\n", len(products))
    
    // NOT
    db.Not("category = ?", "books").Find(&products)
    fmt.Printf("NOT books: %d products\n", len(products))
    
    // LIKE
    db.Where("name LIKE ?", "%Book%").Find(&products)
    fmt.Printf("Name LIKE Book: %d products\n", len(products))
    
    // BETWEEN
    db.Where("price BETWEEN ? AND ?", 400, 600).Find(&products)
    fmt.Printf("Price 400-600: %d products\n", len(products))
    
    // Order, Limit, Offset
    db.Where("active = ?", true).
        Order("price desc").
        Limit(3).
        Offset(0).
        Find(&products)
    fmt.Printf("\nTop 3 most expensive active:\n")
    for _, p := range products {
        fmt.Printf("  - %s: %.0f\n", p.Name, p.Price)
    }
    
    // Group by + Having
    type CategoryStats struct {
        Category string
        Count    int
        AvgPrice float64
    }
    var stats []CategoryStats
    db.Model(&Product{}).
        Select("category, count(*) as count, avg(price) as avg_price").
        Group("category").
        Having("count(*) > ?", 1).
        Scan(&stats)
    
    fmt.Printf("\nCategory stats:\n")
    for _, s := range stats {
        fmt.Printf("  - %s: %d products, avg price %.0f\n", s.Category, s.Count, s.AvgPrice)
    }
}
```

### Scopes (ตัวอย่างที่ 11)

```go
package main

import (
    "fmt"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Product struct {
    gorm.Model
    Name     string
    Price    float64
    Active   bool
    Category string
}

// Reusable query scopes
func ActiveScope(db *gorm.DB) *gorm.DB {
    return db.Where("active = ?", true)
}

func PriceRange(min, max float64) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("price BETWEEN ? AND ?", min, max)
    }
}

func InCategory(category string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("category = ?", category)
    }
}

func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&Product{})
    db.Create(&[]Product{
        {Name: "Go Book", Price: 499, Active: true, Category: "books"},
        {Name: "Python Book", Price: 399, Active: true, Category: "books"},
        {Name: "Laptop Stand", Price: 1200, Active: true, Category: "accessories"},
        {Name: "Old Book", Price: 199, Active: false, Category: "books"},
    })
    
    var products []Product
    
    // ใช้ scopes
    db.Scopes(ActiveScope, InCategory("books")).Find(&products)
    fmt.Printf("Active books: %d\n", len(products))
    
    // Combine scopes
    db.Scopes(
        ActiveScope,
        PriceRange(300, 600),
        Paginate(1, 2),
    ).Find(&products)
    fmt.Printf("Active products 300-600 (page 1): %d\n", len(products))
}
```

---

## 30.6 Preloading (Associations)

### Preload Associations (ตัวอย่างที่ 12)

```go
package main

import (
    "fmt"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name    string
    Email   string
    Posts   []Post  `gorm:"foreignKey:UserID"`
    Profile *Profile
}

type Post struct {
    gorm.Model
    Title   string
    Content string
    UserID  uint
    Tags    []Tag `gorm:"many2many:post_tags"`
}

type Profile struct {
    gorm.Model
    UserID  uint
    Bio     string
    Website string
}

type Tag struct {
    gorm.Model
    Name  string
    Posts []Post `gorm:"many2many:post_tags"`
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{}, &Post{}, &Profile{}, &Tag{})
    
    // สร้าง test data
    tag1 := Tag{Name: "golang"}
    tag2 := Tag{Name: "programming"}
    db.Create(&tag1)
    db.Create(&tag2)
    
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Posts: []Post{
            {Title: "Go Basics", Content: "...", Tags: []Tag{tag1, tag2}},
            {Title: "Go Advanced", Content: "...", Tags: []Tag{tag1}},
        },
        Profile: &Profile{Bio: "Go developer", Website: "https://alice.dev"},
    }
    db.Create(&user)
    
    // Preload Posts
    var userWithPosts User
    db.Preload("Posts").First(&userWithPosts, user.ID)
    fmt.Printf("\nUser: %s, Posts: %d\n", userWithPosts.Name, len(userWithPosts.Posts))
    
    // Preload Posts.Tags (nested)
    var userWithAll User
    db.Preload("Posts.Tags").Preload("Profile").First(&userWithAll, user.ID)
    fmt.Printf("\nUser: %s\n", userWithAll.Name)
    fmt.Printf("Bio: %s\n", userWithAll.Profile.Bio)
    for _, post := range userWithAll.Posts {
        tags := []string{}
        for _, tag := range post.Tags {
            tags = append(tags, tag.Name)
        }
        fmt.Printf("  Post: %s, Tags: %v\n", post.Title, tags)
    }
    
    // Preload with conditions
    var userWithPublished User
    db.Preload("Posts", "title LIKE ?", "%Basics%").First(&userWithPublished, user.ID)
    fmt.Printf("\nPosts with 'Basics': %d\n", len(userWithPublished.Posts))
    
    // Preload All (ระวัง! อาจช้าถ้ามี associations เยอะ)
    // db.Preload(clause.Associations).Find(&users)
}
```

---

## 30.7 Hooks

### BeforeCreate, AfterUpdate Hooks (ตัวอย่างที่ 13)

```go
package main

import (
    "errors"
    "fmt"
    "strings"
    "time"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name      string
    Email     string `gorm:"uniqueIndex"`
    Password  string
    LastLogin *time.Time
    LoginCount int
}

// BeforeCreate - เรียกก่อน INSERT
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // Normalize email
    u.Email = strings.ToLower(strings.TrimSpace(u.Email))
    
    // Validate
    if u.Name == "" {
        return errors.New("name is required")
    }
    if u.Email == "" {
        return errors.New("email is required")
    }
    if len(u.Password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    
    // Hash password (simplified - ใน real app ใช้ bcrypt)
    u.Password = "hashed_" + u.Password
    
    fmt.Printf("[Hook] BeforeCreate: %s\n", u.Email)
    return nil
}

// AfterCreate - เรียกหลัง INSERT
func (u *User) AfterCreate(tx *gorm.DB) error {
    fmt.Printf("[Hook] AfterCreate: User %d created\n", u.ID)
    // อาจส่ง email, create log, etc.
    return nil
}

// BeforeUpdate - เรียกก่อน UPDATE
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // Normalize email ถ้ามีการเปลี่ยน
    if tx.Statement.Changed("Email") {
        u.Email = strings.ToLower(u.Email)
    }
    return nil
}

// AfterUpdate - เรียกหลัง UPDATE  
func (u *User) AfterUpdate(tx *gorm.DB) error {
    fmt.Printf("[Hook] AfterUpdate: User %d updated\n", u.ID)
    return nil
}

// BeforeDelete - เรียกก่อน DELETE
func (u *User) BeforeDelete(tx *gorm.DB) error {
    fmt.Printf("[Hook] BeforeDelete: Deleting user %d\n", u.ID)
    return nil
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{})
    
    // Create - hooks ทำงาน
    user := User{
        Name:     "Alice",
        Email:    "ALICE@EXAMPLE.COM", // จะถูก lowercase ใน hook
        Password: "password123",
    }
    
    if err := db.Create(&user).Error; err != nil {
        fmt.Println("Create error:", err)
    } else {
        fmt.Printf("Created: %s, Email: %s\n", user.Name, user.Email)
    }
    
    // Create ที่จะ fail validation
    badUser := User{Name: "", Email: "bad@example.com", Password: "short"}
    if err := db.Create(&badUser).Error; err != nil {
        fmt.Println("Validation error (expected):", err)
    }
    
    // Update
    db.Model(&user).Update("email", "ALICE.UPDATED@EXAMPLE.COM")
    
    fmt.Println("\nAll hooks demonstrated!")
}
```

---

## 30.8 Transactions

### GORM Transactions (ตัวอย่างที่ 14)

```go
package main

import (
    "errors"
    "fmt"
    "log"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Account struct {
    gorm.Model
    Name    string
    Balance float64
}

type TransferRecord struct {
    gorm.Model
    FromID  uint
    ToID    uint
    Amount  float64
    Status  string
}

func transfer(db *gorm.DB, fromID, toID uint, amount float64) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // ดึงข้อมูล account A
        var from Account
        if err := tx.First(&from, fromID).Error; err != nil {
            return fmt.Errorf("from account not found: %w", err)
        }
        
        // ดึงข้อมูล account B
        var to Account
        if err := tx.First(&to, toID).Error; err != nil {
            return fmt.Errorf("to account not found: %w", err)
        }
        
        // ตรวจยอดเงิน
        if from.Balance < amount {
            return errors.New("insufficient funds")
        }
        
        // หักเงิน
        if err := tx.Model(&from).Update("balance", from.Balance-amount).Error; err != nil {
            return err
        }
        
        // เพิ่มเงิน
        if err := tx.Model(&to).Update("balance", to.Balance+amount).Error; err != nil {
            return err
        }
        
        // บันทึก transfer
        record := TransferRecord{
            FromID: fromID,
            ToID:   toID,
            Amount: amount,
            Status: "completed",
        }
        return tx.Create(&record).Error
    })
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&Account{}, &TransferRecord{})
    
    // สร้าง accounts
    alice := Account{Name: "Alice", Balance: 1000}
    bob := Account{Name: "Bob", Balance: 500}
    db.Create(&alice)
    db.Create(&bob)
    
    fmt.Printf("Before: Alice=%.0f, Bob=%.0f\n", alice.Balance, bob.Balance)
    
    // Transfer สำเร็จ
    if err := transfer(db, alice.ID, bob.ID, 200); err != nil {
        log.Fatal("Transfer error:", err)
    }
    
    db.First(&alice, alice.ID)
    db.First(&bob, bob.ID)
    fmt.Printf("After transfer: Alice=%.0f, Bob=%.0f\n", alice.Balance, bob.Balance)
    
    // Transfer ที่จะ fail
    if err := transfer(db, alice.ID, bob.ID, 10000); err != nil {
        fmt.Println("Expected error:", err)
    }
    
    // ยอดเงินต้องเหมือนเดิม
    db.First(&alice, alice.ID)
    fmt.Printf("After failed transfer: Alice=%.0f (unchanged)\n", alice.Balance)
    
    // ดู transfer records
    var records []TransferRecord
    db.Find(&records)
    fmt.Printf("\nTransfer records: %d\n", len(records))
    for _, r := range records {
        fmt.Printf("  - %.0f from %d to %d (%s)\n", r.Amount, r.FromID, r.ToID, r.Status)
    }
}
```

---

## 30.9 Raw Queries

### Raw SQL กับ GORM (ตัวอย่างที่ 15)

```go
package main

import (
    "fmt"
    "time"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string
    Age   int
}

type UserStats struct {
    TotalUsers  int
    AvgAge      float64
    MaxAge      int
    MinAge      int
}

func main() {
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&User{})
    db.Create(&[]User{
        {Name: "Alice", Email: "alice@example.com", Age: 25},
        {Name: "Bob", Email: "bob@example.com", Age: 30},
        {Name: "Charlie", Email: "charlie@example.com", Age: 35},
    })
    
    // Raw query สำหรับ SELECT
    var users []User
    db.Raw("SELECT * FROM users WHERE age > ? ORDER BY age DESC", 20).Scan(&users)
    fmt.Printf("Raw SELECT: %d users\n", len(users))
    
    // Raw query กับ custom struct
    var stats UserStats
    db.Raw(`
        SELECT 
            COUNT(*) as total_users,
            AVG(age) as avg_age,
            MAX(age) as max_age,
            MIN(age) as min_age
        FROM users
        WHERE deleted_at IS NULL
    `).Scan(&stats)
    fmt.Printf("\nStats: total=%d, avg=%.1f, max=%d, min=%d\n",
        stats.TotalUsers, stats.AvgAge, stats.MaxAge, stats.MinAge)
    
    // Raw Exec (INSERT, UPDATE, DELETE)
    db.Exec(`UPDATE users SET name = ? WHERE email = ?`, "Alice Updated", "alice@example.com")
    
    // Row (single row)
    row := db.Raw("SELECT COUNT(*) FROM users").Row()
    var count int
    row.Scan(&count)
    fmt.Printf("\nTotal users: %d\n", count)
    
    // Rows (multiple rows)
    rows, _ := db.Raw("SELECT id, name, created_at FROM users").Rows()
    defer rows.Close()
    
    for rows.Next() {
        var id uint
        var name string
        var createdAt time.Time
        rows.Scan(&id, &name, &createdAt)
        fmt.Printf("Row: %d, %s, %s\n", id, name, createdAt.Format("2006-01-02"))
    }
}
```

---

## Workshop: User & Post API ด้วย GORM

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strconv"
    "time"
    
    "github.com/gin-gonic/gin"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

// Models
type User struct {
    gorm.Model
    Name     string  `gorm:"not null" json:"name"`
    Email    string  `gorm:"uniqueIndex;not null" json:"email"`
    Posts    []Post  `gorm:"foreignKey:AuthorID" json:"posts,omitempty"`
}

type Post struct {
    gorm.Model
    Title     string    `gorm:"not null" json:"title"`
    Content   string    `gorm:"type:text" json:"content"`
    Published bool      `gorm:"default:false" json:"published"`
    AuthorID  uint      `json:"author_id"`
    Author    *User     `gorm:"constraint:OnDelete:CASCADE" json:"author,omitempty"`
    Tags      []Tag     `gorm:"many2many:post_tags" json:"tags,omitempty"`
}

type Tag struct {
    gorm.Model
    Name  string  `gorm:"uniqueIndex" json:"name"`
    Posts []Post  `gorm:"many2many:post_tags" json:"-"`
}

// Request types
type CreateUserReq struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

type CreatePostReq struct {
    Title     string   `json:"title" binding:"required,min=3"`
    Content   string   `json:"content" binding:"required"`
    Published bool     `json:"published"`
    AuthorID  uint     `json:"author_id" binding:"required"`
    Tags      []string `json:"tags"`
}

// Handlers
type App struct {
    db *gorm.DB
}

func (a *App) createUser(c *gin.Context) {
    var req CreateUserReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    user := User{Name: req.Name, Email: req.Email}
    if err := a.db.Create(&user).Error; err != nil {
        c.JSON(http.StatusConflict, gin.H{"error": "Email already exists"})
        return
    }
    
    c.JSON(http.StatusCreated, user)
}

func (a *App) getUser(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    var user User
    if err := a.db.Preload("Posts").First(&user, id).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
        return
    }
    c.JSON(http.StatusOK, user)
}

func (a *App) listUsers(c *gin.Context) {
    var users []User
    a.db.Find(&users)
    c.JSON(http.StatusOK, gin.H{"data": users, "total": len(users)})
}

func (a *App) createPost(c *gin.Context) {
    var req CreatePostReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // ตรวจสอบว่า author มีอยู่
    var author User
    if err := a.db.First(&author, req.AuthorID).Error; err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Author not found"})
        return
    }
    
    // สร้าง tags
    var tags []Tag
    for _, tagName := range req.Tags {
        tag := Tag{Name: tagName}
        a.db.FirstOrCreate(&tag, Tag{Name: tagName})
        tags = append(tags, tag)
    }
    
    post := Post{
        Title:     req.Title,
        Content:   req.Content,
        Published: req.Published,
        AuthorID:  req.AuthorID,
        Tags:      tags,
    }
    
    if err := a.db.Create(&post).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    // Reload with associations
    a.db.Preload("Author").Preload("Tags").First(&post, post.ID)
    c.JSON(http.StatusCreated, post)
}

func (a *App) listPosts(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    limit, _ := strconv.Atoi(c.DefaultQuery("limit", "10"))
    
    var posts []Post
    var total int64
    
    query := a.db.Model(&Post{})
    
    if published := c.Query("published"); published != "" {
        query = query.Where("published = ?", published == "true")
    }
    
    query.Count(&total)
    query.Preload("Author").Preload("Tags").
        Order("created_at DESC").
        Offset((page - 1) * limit).
        Limit(limit).
        Find(&posts)
    
    c.JSON(http.StatusOK, gin.H{
        "data":  posts,
        "total": total,
        "page":  page,
        "limit": limit,
    })
}

func (a *App) getPost(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    var post Post
    if err := a.db.Preload("Author").Preload("Tags").First(&post, id).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }
    c.JSON(http.StatusOK, post)
}

func (a *App) deletePost(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    if err := a.db.Delete(&Post{}, id).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "Post deleted"})
}

func (a *App) searchPosts(c *gin.Context) {
    q := c.Query("q")
    if q == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Search query required"})
        return
    }
    
    var posts []Post
    a.db.Preload("Author").Preload("Tags").
        Where("title LIKE ? OR content LIKE ?", "%"+q+"%", "%"+q+"%").
        Find(&posts)
    
    c.JSON(http.StatusOK, gin.H{
        "data":  posts,
        "query": q,
        "count": len(posts),
    })
}

// Statistics endpoint
func (a *App) getStats(c *gin.Context) {
    var userCount, postCount, publishedCount int64
    a.db.Model(&User{}).Count(&userCount)
    a.db.Model(&Post{}).Count(&postCount)
    a.db.Model(&Post{}).Where("published = ?", true).Count(&publishedCount)
    
    type TagStats struct {
        Name  string
        Count int
    }
    var popularTags []TagStats
    a.db.Raw(`
        SELECT t.name, COUNT(pt.post_id) as count
        FROM tags t
        JOIN post_tags pt ON t.id = pt.tag_id
        GROUP BY t.id
        ORDER BY count DESC
        LIMIT 5
    `).Scan(&popularTags)
    
    c.JSON(http.StatusOK, gin.H{
        "users":          userCount,
        "posts":          postCount,
        "published_posts": publishedCount,
        "popular_tags":   popularTags,
        "generated_at":   time.Now(),
    })
}

func main() {
    // Setup DB
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    
    db.AutoMigrate(&User{}, &Post{}, &Tag{})
    
    // Seed data
    user1 := User{Name: "Alice", Email: "alice@example.com"}
    user2 := User{Name: "Bob", Email: "bob@example.com"}
    db.Create(&user1)
    db.Create(&user2)
    
    tag1, tag2 := Tag{Name: "golang"}, Tag{Name: "tutorial"}
    db.Create(&tag1)
    db.Create(&tag2)
    
    db.Create(&Post{
        Title: "Go GORM Tutorial", Content: "Learning GORM...",
        Published: true, AuthorID: user1.ID, Tags: []Tag{tag1, tag2},
    })
    db.Create(&Post{
        Title: "REST API with Gin", Content: "Building REST API...",
        Published: true, AuthorID: user1.ID, Tags: []Tag{tag1},
    })
    
    app := &App{db: db}
    
    r := gin.Default()
    
    api := r.Group("/api/v1")
    {
        api.GET("/users", app.listUsers)
        api.POST("/users", app.createUser)
        api.GET("/users/:id", app.getUser)
        
        api.GET("/posts", app.listPosts)
        api.POST("/posts", app.createPost)
        api.GET("/posts/search", app.searchPosts)
        api.GET("/posts/:id", app.getPost)
        api.DELETE("/posts/:id", app.deletePost)
        
        api.GET("/stats", app.getStats)
    }
    
    fmt.Println("GORM Blog API running on :8080")
    r.Run(":8080")
}
```

---

## สรุป Part 30

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Setup | `gorm.Open(driver, config)` |
| Model | embed `gorm.Model` หรือกำหนด primary key เอง |
| AutoMigrate | `db.AutoMigrate(&Model{})` - เพิ่ม columns แต่ไม่ drop |
| Create | `db.Create(&record)` |
| Read | `db.First()`, `db.Find()`, `db.Where()` |
| Update | `db.Save()`, `db.Updates()`, `db.Update()` |
| Delete | `db.Delete()` (soft), `db.Unscoped().Delete()` (hard) |
| Preload | `db.Preload("Association")` |
| Hooks | `BeforeCreate`, `AfterUpdate`, etc. |
| Transaction | `db.Transaction(func(tx) error)` |
| Raw | `db.Raw()`, `db.Exec()` |

### Resources
- [GORM documentation](https://gorm.io/docs/)
- [GORM GitHub](https://github.com/go-gorm/gorm)
- [GORM associations](https://gorm.io/docs/has_one.html)
