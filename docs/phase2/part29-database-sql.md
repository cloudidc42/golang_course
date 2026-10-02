# Part 29: Database SQL ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `database/sql` package
- เชื่อมต่อ database (PostgreSQL, MySQL, SQLite)
- ตั้งค่า Connection Pool
- ทำ CRUD operations
- ใช้ Prepared Statements
- จัดการ Transactions
- Scan rows ให้เป็น struct
- จัดการ Null values
- ใช้ Context กับ database queries
- เข้าใจ Migration basics

---

## 29.1 database/sql Package

Go มี `database/sql` package ที่เป็น abstraction layer สำหรับ database ต่างๆ โดยต้องใช้คู่กับ database driver

### Driver ที่ใช้บ่อย

| Database | Driver Package |
|----------|---------------|
| PostgreSQL | `github.com/lib/pq` หรือ `github.com/jackc/pgx/v5` |
| MySQL | `github.com/go-sql-driver/mysql` |
| SQLite | `github.com/mattn/go-sqlite3` |

### ติดตั้ง Dependencies (PostgreSQL)

```bash
go get github.com/lib/pq
# หรือ pgx (แนะนำสำหรับ production)
go get github.com/jackc/pgx/v5/stdlib
```

---

## 29.2 การเชื่อมต่อ Database

### Connect PostgreSQL (ตัวอย่างที่ 1)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/lib/pq" // import driver (blank import)
)

func main() {
    // Connection string
    connStr := "host=localhost port=5432 user=postgres password=secret dbname=myapp sslmode=disable"
    
    // เปิด connection (ยังไม่ได้ connect จริง)
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal("Failed to open database:", err)
    }
    defer db.Close()
    
    // ทดสอบ connection จริงๆ
    if err := db.Ping(); err != nil {
        log.Fatal("Failed to ping database:", err)
    }
    
    fmt.Println("Successfully connected to PostgreSQL!")
}
```

### Connect MySQL (ตัวอย่างที่ 2)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    // MySQL DSN: user:password@tcp(host:port)/dbname?options
    dsn := "root:secret@tcp(localhost:3306)/myapp?charset=utf8mb4&parseTime=True&loc=Local"
    
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal("Failed to open database:", err)
    }
    defer db.Close()
    
    if err := db.Ping(); err != nil {
        log.Fatal("Failed to ping database:", err)
    }
    
    fmt.Println("Successfully connected to MySQL!")
}
```

### Connect SQLite (ตัวอย่างที่ 3)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

func main() {
    // SQLite - ใช้ไฟล์ หรือ :memory: สำหรับ in-memory
    db, err := sql.Open("sqlite3", "./myapp.db")
    if err != nil {
        log.Fatal("Failed to open SQLite:", err)
    }
    defer db.Close()
    
    if err := db.Ping(); err != nil {
        log.Fatal("Failed to ping SQLite:", err)
    }
    
    fmt.Println("Connected to SQLite!")
}
```

---

## 29.3 Connection Pool Configuration

### ตั้งค่า Connection Pool (ตัวอย่างที่ 4)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"
    
    _ "github.com/lib/pq"
)

func NewDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening database: %w", err)
    }
    
    // Connection Pool Settings
    db.SetMaxOpenConns(25)                  // จำนวน connections สูงสุด
    db.SetMaxIdleConns(10)                  // จำนวน idle connections
    db.SetConnMaxLifetime(5 * time.Minute)  // อายุสูงสุดของ connection
    db.SetConnMaxIdleTime(2 * time.Minute)  // อายุสูงสุดเมื่อ idle
    
    // ทดสอบ connection
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("pinging database: %w", err)
    }
    
    return db, nil
}

func main() {
    dsn := "host=localhost port=5432 user=postgres password=secret dbname=myapp sslmode=disable"
    
    db, err := NewDB(dsn)
    if err != nil {
        log.Fatal("Database connection error:", err)
    }
    defer db.Close()
    
    // ดูสถิติ connection pool
    stats := db.Stats()
    fmt.Printf("Open connections: %d\n", stats.OpenConnections)
    fmt.Printf("In use: %d\n", stats.InUse)
    fmt.Printf("Idle: %d\n", stats.Idle)
    
    fmt.Println("Database ready!")
}
```

---

## 29.4 สร้าง Table และ CRUD Operations

### สร้าง Table (ตัวอย่างที่ 5)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

const createTableSQL = `
CREATE TABLE IF NOT EXISTS users (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    age        INTEGER,
    active     BOOLEAN DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
`

func main() {
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // สร้าง table
    _, err = db.Exec(createTableSQL)
    if err != nil {
        log.Fatal("Create table error:", err)
    }
    
    fmt.Println("Table created successfully!")
}
```

### INSERT - สร้างข้อมูล (ตัวอย่างที่ 6)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type User struct {
    ID        int64
    Name      string
    Email     string
    Age       int
    Active    bool
}

func insertUser(db *sql.DB, user User) (int64, error) {
    result, err := db.Exec(
        `INSERT INTO users (name, email, age, active) VALUES (?, ?, ?, ?)`,
        user.Name, user.Email, user.Age, user.Active,
    )
    if err != nil {
        return 0, fmt.Errorf("insert user: %w", err)
    }
    
    id, err := result.LastInsertId()
    if err != nil {
        return 0, fmt.Errorf("get last insert id: %w", err)
    }
    
    return id, nil
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    
    db.Exec(`CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL, email TEXT NOT NULL UNIQUE,
        age INTEGER, active BOOLEAN DEFAULT TRUE
    )`)
    
    // Insert user
    id, err := insertUser(db, User{
        Name:   "สมชาย รักเรียน",
        Email:  "somchai@example.com",
        Age:    28,
        Active: true,
    })
    if err != nil {
        log.Fatal("Insert error:", err)
    }
    fmt.Printf("Inserted user with ID: %d\n", id)
    
    // Insert multiple users
    users := []User{
        {Name: "สมหญิง รักสวย", Email: "somying@example.com", Age: 25, Active: true},
        {Name: "สมศักดิ์ ขยันทำ", Email: "somsak@example.com", Age: 35, Active: false},
    }
    
    for _, u := range users {
        id, err := insertUser(db, u)
        if err != nil {
            log.Printf("Error inserting %s: %v", u.Name, err)
            continue
        }
        fmt.Printf("Inserted: %s (ID: %d)\n", u.Name, id)
    }
}
```

### SELECT - อ่านข้อมูล (ตัวอย่างที่ 7)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type User struct {
    ID     int64
    Name   string
    Email  string
    Age    int
    Active bool
}

func getUser(db *sql.DB, id int64) (*User, error) {
    row := db.QueryRow(`SELECT id, name, email, age, active FROM users WHERE id = ?`, id)
    
    var user User
    err := row.Scan(&user.ID, &user.Name, &user.Email, &user.Age, &user.Active)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %d not found", id)
        }
        return nil, fmt.Errorf("scanning user: %w", err)
    }
    
    return &user, nil
}

func getAllUsers(db *sql.DB) ([]User, error) {
    rows, err := db.Query(`SELECT id, name, email, age, active FROM users ORDER BY id`)
    if err != nil {
        return nil, fmt.Errorf("query users: %w", err)
    }
    defer rows.Close() // ต้อง close rows!
    
    var users []User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.Age, &user.Active); err != nil {
            return nil, fmt.Errorf("scanning row: %w", err)
        }
        users = append(users, user)
    }
    
    // ตรวจสอบ error หลัง loop
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("rows error: %w", err)
    }
    
    return users, nil
}

func getUsersByAge(db *sql.DB, minAge, maxAge int) ([]User, error) {
    rows, err := db.Query(
        `SELECT id, name, email, age, active FROM users WHERE age BETWEEN ? AND ? ORDER BY age`,
        minAge, maxAge,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.Age, &u.Active); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}

func setupDB() *sql.DB {
    db, _ := sql.Open("sqlite3", ":memory:")
    db.Exec(`CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL, email TEXT NOT NULL,
        age INTEGER, active BOOLEAN DEFAULT TRUE
    )`)
    db.Exec(`INSERT INTO users (name, email, age, active) VALUES 
        ('Alice', 'alice@example.com', 25, 1),
        ('Bob', 'bob@example.com', 30, 1),
        ('Charlie', 'charlie@example.com', 35, 0)`)
    return db
}

func main() {
    db := setupDB()
    defer db.Close()
    
    // Get single user
    user, err := getUser(db, 1)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User: %+v\n", user)
    
    // Get all users
    users, err := getAllUsers(db)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("\nAll users (%d):\n", len(users))
    for _, u := range users {
        fmt.Printf("  - %s (%s) age=%d active=%v\n", u.Name, u.Email, u.Age, u.Active)
    }
    
    // Get users by age range
    filtered, err := getUsersByAge(db, 25, 32)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("\nUsers aged 25-32: %d\n", len(filtered))
}
```

### UPDATE และ DELETE (ตัวอย่างที่ 8)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

func updateUser(db *sql.DB, id int64, name, email string) error {
    result, err := db.Exec(
        `UPDATE users SET name = ?, email = ? WHERE id = ?`,
        name, email, id,
    )
    if err != nil {
        return fmt.Errorf("update user: %w", err)
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return fmt.Errorf("user %d not found", id)
    }
    
    fmt.Printf("Updated %d row(s)\n", rowsAffected)
    return nil
}

func deleteUser(db *sql.DB, id int64) error {
    result, err := db.Exec(`DELETE FROM users WHERE id = ?`, id)
    if err != nil {
        return fmt.Errorf("delete user: %w", err)
    }
    
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return fmt.Errorf("user %d not found", id)
    }
    
    fmt.Printf("Deleted user ID: %d\n", id)
    return nil
}

func deactivateUser(db *sql.DB, id int64) error {
    _, err := db.Exec(`UPDATE users SET active = FALSE WHERE id = ?`, id)
    return err
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, email TEXT, active BOOLEAN)`)
    db.Exec(`INSERT INTO users (name, email, active) VALUES ('Alice', 'alice@old.com', 1)`)
    
    // Update
    if err := updateUser(db, 1, "Alice Updated", "alice@new.com"); err != nil {
        log.Fatal(err)
    }
    
    // Deactivate
    deactivateUser(db, 1)
    
    // Delete
    if err := deleteUser(db, 1); err != nil {
        log.Fatal(err)
    }
    
    // ลอง delete อีกครั้ง (ไม่มีอยู่แล้ว)
    if err := deleteUser(db, 1); err != nil {
        fmt.Println("Expected error:", err)
    }
}
```

---

## 29.5 Prepared Statements

### ใช้ Prepared Statements (ตัวอย่างที่ 9)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type UserRepository struct {
    db   *sql.DB
    stmts map[string]*sql.Stmt
}

func NewUserRepository(db *sql.DB) (*UserRepository, error) {
    repo := &UserRepository{
        db:    db,
        stmts: make(map[string]*sql.Stmt),
    }
    
    // Prepare statements ล่วงหน้า (ดีสำหรับ performance)
    stmts := map[string]string{
        "getByID":    `SELECT id, name, email FROM users WHERE id = ?`,
        "getByEmail": `SELECT id, name, email FROM users WHERE email = ?`,
        "insert":     `INSERT INTO users (name, email) VALUES (?, ?)`,
        "update":     `UPDATE users SET name = ?, email = ? WHERE id = ?`,
        "delete":     `DELETE FROM users WHERE id = ?`,
    }
    
    for name, query := range stmts {
        stmt, err := db.Prepare(query)
        if err != nil {
            repo.Close()
            return nil, fmt.Errorf("prepare statement %s: %w", name, err)
        }
        repo.stmts[name] = stmt
    }
    
    return repo, nil
}

func (r *UserRepository) Close() {
    for _, stmt := range r.stmts {
        stmt.Close()
    }
}

type User struct {
    ID    int64
    Name  string
    Email string
}

func (r *UserRepository) GetByID(id int64) (*User, error) {
    row := r.stmts["getByID"].QueryRow(id)
    var user User
    if err := row.Scan(&user.ID, &user.Name, &user.Email); err != nil {
        if err == sql.ErrNoRows {
            return nil, nil // not found
        }
        return nil, err
    }
    return &user, nil
}

func (r *UserRepository) Insert(user User) (int64, error) {
    result, err := r.stmts["insert"].Exec(user.Name, user.Email)
    if err != nil {
        return 0, err
    }
    return result.LastInsertId()
}

func (r *UserRepository) Update(user User) error {
    result, err := r.stmts["update"].Exec(user.Name, user.Email, user.ID)
    if err != nil {
        return err
    }
    n, _ := result.RowsAffected()
    if n == 0 {
        return fmt.Errorf("user not found: %d", user.ID)
    }
    return nil
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, email TEXT)`)
    
    repo, err := NewUserRepository(db)
    if err != nil {
        log.Fatal(err)
    }
    defer repo.Close()
    
    // Insert
    id, err := repo.Insert(User{Name: "Alice", Email: "alice@example.com"})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Inserted ID: %d\n", id)
    
    // Get
    user, err := repo.GetByID(id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Found: %+v\n", user)
    
    // Update
    user.Name = "Alice Updated"
    if err := repo.Update(*user); err != nil {
        log.Fatal(err)
    }
    fmt.Println("Updated successfully")
}
```

---

## 29.6 Transactions

### Basic Transaction (ตัวอย่างที่ 10)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type Account struct {
    ID      int64
    Name    string
    Balance float64
}

func transferMoney(db *sql.DB, fromID, toID int64, amount float64) error {
    // เริ่ม transaction
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    
    // defer rollback ถ้าเกิด error (ถ้า commit แล้ว rollback จะไม่ทำอะไร)
    defer tx.Rollback()
    
    // ตรวจสอบยอดเงิน
    var fromBalance float64
    err = tx.QueryRow(`SELECT balance FROM accounts WHERE id = ?`, fromID).Scan(&fromBalance)
    if err != nil {
        return fmt.Errorf("get from balance: %w", err)
    }
    
    if fromBalance < amount {
        return fmt.Errorf("insufficient funds: have %.2f, need %.2f", fromBalance, amount)
    }
    
    // หักเงินจาก account A
    _, err = tx.Exec(`UPDATE accounts SET balance = balance - ? WHERE id = ?`, amount, fromID)
    if err != nil {
        return fmt.Errorf("debit account: %w", err)
    }
    
    // เพิ่มเงินใน account B
    _, err = tx.Exec(`UPDATE accounts SET balance = balance + ? WHERE id = ?`, amount, toID)
    if err != nil {
        return fmt.Errorf("credit account: %w", err)
    }
    
    // บันทึก transaction log
    _, err = tx.Exec(
        `INSERT INTO transfers (from_id, to_id, amount) VALUES (?, ?, ?)`,
        fromID, toID, amount,
    )
    if err != nil {
        return fmt.Errorf("log transfer: %w", err)
    }
    
    // Commit transaction
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    fmt.Printf("Transferred %.2f from account %d to account %d\n", amount, fromID, toID)
    return nil
}

func setupAccountsDB() *sql.DB {
    db, _ := sql.Open("sqlite3", ":memory:")
    db.Exec(`CREATE TABLE accounts (id INTEGER PRIMARY KEY, name TEXT, balance REAL)`)
    db.Exec(`CREATE TABLE transfers (id INTEGER PRIMARY KEY AUTOINCREMENT, from_id INTEGER, to_id INTEGER, amount REAL)`)
    db.Exec(`INSERT INTO accounts VALUES (1, 'Alice', 1000.00), (2, 'Bob', 500.00)`)
    return db
}

func getBalance(db *sql.DB, id int64) float64 {
    var balance float64
    db.QueryRow(`SELECT balance FROM accounts WHERE id = ?`, id).Scan(&balance)
    return balance
}

func main() {
    db := setupAccountsDB()
    defer db.Close()
    
    fmt.Printf("Before: Alice=%.2f, Bob=%.2f\n", getBalance(db, 1), getBalance(db, 2))
    
    // Transfer สำเร็จ
    err := transferMoney(db, 1, 2, 200.00)
    if err != nil {
        log.Println("Transfer error:", err)
    }
    
    fmt.Printf("After: Alice=%.2f, Bob=%.2f\n", getBalance(db, 1), getBalance(db, 2))
    
    // Transfer ที่จะ fail (เงินไม่พอ)
    err = transferMoney(db, 1, 2, 10000.00)
    if err != nil {
        fmt.Println("Expected error:", err)
    }
    
    fmt.Printf("Final: Alice=%.2f, Bob=%.2f\n", getBalance(db, 1), getBalance(db, 2))
}
```

### Transaction Helper (ตัวอย่างที่ 11)

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

// WithTransaction เป็น helper ที่จัดการ begin/commit/rollback ให้
func WithTransaction(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p) // re-panic
        }
    }()
    
    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("rollback error: %v (original: %w)", rbErr, err)
        }
        return err
    }
    
    return tx.Commit()
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    db.Exec(`CREATE TABLE orders (id INTEGER PRIMARY KEY AUTOINCREMENT, product TEXT, quantity INTEGER)`)
    
    ctx := context.Background()
    
    // ใช้ WithTransaction helper
    err := WithTransaction(ctx, db, func(tx *sql.Tx) error {
        // สร้าง order
        result, err := tx.Exec(`INSERT INTO orders (product, quantity) VALUES (?, ?)`, "Go Book", 2)
        if err != nil {
            return fmt.Errorf("create order: %w", err)
        }
        
        orderID, _ := result.LastInsertId()
        fmt.Printf("Created order ID: %d\n", orderID)
        
        // อัพเดท inventory (simulate)
        _, err = tx.Exec(`INSERT INTO orders (product, quantity) VALUES (?, ?)`, "Go Book", -2)
        if err != nil {
            return fmt.Errorf("update inventory: %w", err)
        }
        
        return nil // commit
    })
    
    if err != nil {
        log.Fatal("Transaction failed:", err)
    }
    
    fmt.Println("Transaction completed successfully!")
}
```

---

## 29.7 Null Values

### จัดการ NULL ด้วย sql.NullXxx (ตัวอย่างที่ 12)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type Profile struct {
    ID       int64
    Name     string
    Email    string
    Phone    sql.NullString  // อาจเป็น NULL
    Age      sql.NullInt64   // อาจเป็น NULL
    Score    sql.NullFloat64 // อาจเป็น NULL
    Active   sql.NullBool    // อาจเป็น NULL
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    
    db.Exec(`CREATE TABLE profiles (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT NOT NULL,
        phone TEXT,
        age INTEGER,
        score REAL,
        active BOOLEAN
    )`)
    
    // Insert พร้อม NULL values
    db.Exec(`INSERT INTO profiles (name, email, phone, age, score, active) VALUES 
        ('Alice', 'alice@example.com', '+66-81-234-5678', 25, 95.5, 1),
        ('Bob', 'bob@example.com', NULL, NULL, NULL, NULL)`)
    
    rows, _ := db.Query(`SELECT id, name, email, phone, age, score, active FROM profiles`)
    defer rows.Close()
    
    for rows.Next() {
        var p Profile
        if err := rows.Scan(&p.ID, &p.Name, &p.Email, &p.Phone, &p.Age, &p.Score, &p.Active); err != nil {
            log.Fatal(err)
        }
        
        fmt.Printf("\nProfile ID: %d\n", p.ID)
        fmt.Printf("Name: %s\n", p.Name)
        fmt.Printf("Email: %s\n", p.Email)
        
        if p.Phone.Valid {
            fmt.Printf("Phone: %s\n", p.Phone.String)
        } else {
            fmt.Println("Phone: (not provided)")
        }
        
        if p.Age.Valid {
            fmt.Printf("Age: %d\n", p.Age.Int64)
        } else {
            fmt.Println("Age: (unknown)")
        }
        
        if p.Score.Valid {
            fmt.Printf("Score: %.1f\n", p.Score.Float64)
        } else {
            fmt.Println("Score: (no score)")
        }
        
        if p.Active.Valid {
            fmt.Printf("Active: %v\n", p.Active.Bool)
        } else {
            fmt.Println("Active: (unknown)")
        }
    }
}
```

### ใช้ Pointer สำหรับ Optional Fields (ตัวอย่างที่ 13)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type User struct {
    ID    int64
    Name  string
    Email string
    Phone *string  // pointer แทน sql.NullString
    Age   *int     // pointer แทน sql.NullInt64
}

func scanUserWithPointers(rows *sql.Rows) (*User, error) {
    var u User
    var phone sql.NullString
    var age sql.NullInt64
    
    if err := rows.Scan(&u.ID, &u.Name, &u.Email, &phone, &age); err != nil {
        return nil, err
    }
    
    // แปลง sql.Null เป็น pointer
    if phone.Valid {
        u.Phone = &phone.String
    }
    if age.Valid {
        a := int(age.Int64)
        u.Age = &a
    }
    
    return &u, nil
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    
    db.Exec(`CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT, email TEXT, phone TEXT, age INTEGER
    )`)
    db.Exec(`INSERT INTO users VALUES (1,'Alice','alice@example.com','+66123456',25)`)
    db.Exec(`INSERT INTO users VALUES (2,'Bob','bob@example.com',NULL,NULL)`)
    
    rows, _ := db.Query(`SELECT id, name, email, phone, age FROM users`)
    defer rows.Close()
    
    for rows.Next() {
        user, err := scanUserWithPointers(rows)
        if err != nil {
            log.Fatal(err)
        }
        
        fmt.Printf("User: %s", user.Name)
        if user.Phone != nil {
            fmt.Printf(", Phone: %s", *user.Phone)
        }
        if user.Age != nil {
            fmt.Printf(", Age: %d", *user.Age)
        }
        fmt.Println()
    }
}
```

---

## 29.8 Query with Context

### ใช้ Context กับ Database (ตัวอย่างที่ 14)

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"
    
    _ "github.com/mattn/go-sqlite3"
)

type UserRepo struct {
    db *sql.DB
}

func (r *UserRepo) GetUserWithTimeout(id int64) (*struct{ ID int64; Name string }, error) {
    // กำหนด timeout สำหรับ query นี้
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    var user struct {
        ID   int64
        Name string
    }
    
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name FROM users WHERE id = ?`, id,
    ).Scan(&user.ID, &user.Name)
    
    if err != nil {
        if err == context.DeadlineExceeded {
            return nil, fmt.Errorf("query timed out")
        }
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %d not found", id)
        }
        return nil, err
    }
    
    return &user, nil
}

func (r *UserRepo) GetUsersWithContext(ctx context.Context) ([]string, error) {
    rows, err := r.db.QueryContext(ctx, `SELECT name FROM users`)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var names []string
    for rows.Next() {
        // ตรวจสอบ context ทุก iteration
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }
        
        var name string
        if err := rows.Scan(&name); err != nil {
            return nil, err
        }
        names = append(names, name)
    }
    return names, rows.Err()
}

func main() {
    db, _ := sql.Open("sqlite3", ":memory:")
    defer db.Close()
    db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`)
    db.Exec(`INSERT INTO users VALUES (1,'Alice'),(2,'Bob'),(3,'Charlie')`)
    
    repo := &UserRepo{db: db}
    
    // Query พร้อม timeout
    user, err := repo.GetUserWithTimeout(1)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Found: %+v\n", user)
    
    // Query พร้อม cancellable context
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    names, err := repo.GetUsersWithContext(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Users: %v\n", names)
}
```

---

## 29.9 Migration Basics

### Simple Migration System (ตัวอย่างที่ 15)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/mattn/go-sqlite3"
)

type Migration struct {
    Version int
    Name    string
    Up      string
    Down    string
}

var migrations = []Migration{
    {
        Version: 1,
        Name:    "create_users_table",
        Up: `CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT NOT NULL UNIQUE,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`,
        Down: `DROP TABLE IF EXISTS users`,
    },
    {
        Version: 2,
        Name:    "add_age_to_users",
        Up:      `ALTER TABLE users ADD COLUMN age INTEGER`,
        Down:    `-- SQLite doesn't support DROP COLUMN easily`,
    },
    {
        Version: 3,
        Name:    "create_posts_table",
        Up: `CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            content TEXT,
            published BOOLEAN DEFAULT FALSE,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )`,
        Down: `DROP TABLE IF EXISTS posts`,
    },
}

func runMigrations(db *sql.DB) error {
    // สร้าง migrations table
    _, err := db.Exec(`CREATE TABLE IF NOT EXISTS schema_migrations (
        version INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        applied_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )`)
    if err != nil {
        return fmt.Errorf("create migrations table: %w", err)
    }
    
    // ดู version ที่ apply แล้ว
    applied := make(map[int]bool)
    rows, _ := db.Query(`SELECT version FROM schema_migrations`)
    defer rows.Close()
    for rows.Next() {
        var v int
        rows.Scan(&v)
        applied[v] = true
    }
    
    // Run migrations ที่ยังไม่ได้ apply
    for _, m := range migrations {
        if applied[m.Version] {
            fmt.Printf("Migration %d (%s): already applied\n", m.Version, m.Name)
            continue
        }
        
        fmt.Printf("Applying migration %d: %s\n", m.Version, m.Name)
        
        tx, err := db.Begin()
        if err != nil {
            return err
        }
        
        if _, err := tx.Exec(m.Up); err != nil {
            tx.Rollback()
            return fmt.Errorf("migration %d failed: %w", m.Version, err)
        }
        
        if _, err := tx.Exec(
            `INSERT INTO schema_migrations (version, name) VALUES (?, ?)`,
            m.Version, m.Name,
        ); err != nil {
            tx.Rollback()
            return err
        }
        
        if err := tx.Commit(); err != nil {
            return err
        }
        
        fmt.Printf("Migration %d applied successfully\n", m.Version)
    }
    
    return nil
}

func main() {
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    fmt.Println("Running migrations...")
    if err := runMigrations(db); err != nil {
        log.Fatal("Migration error:", err)
    }
    
    fmt.Println("\nChecking schema...")
    // ตรวจสอบว่า tables ถูกสร้างแล้ว
    tables := []string{"users", "posts", "schema_migrations"}
    for _, table := range tables {
        var name string
        err := db.QueryRow(`SELECT name FROM sqlite_master WHERE type='table' AND name=?`, table).Scan(&name)
        if err != nil {
            fmt.Printf("Table %s: NOT FOUND\n", table)
        } else {
            fmt.Printf("Table %s: OK\n", name)
        }
    }
}
```

---

## Workshop: Complete User Repository

```go
package main

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "log"
    "time"
    
    _ "github.com/mattn/go-sqlite3"
)

var ErrNotFound = errors.New("record not found")
var ErrDuplicateEmail = errors.New("email already exists")

type User struct {
    ID        int64      `json:"id"`
    Name      string     `json:"name"`
    Email     string     `json:"email"`
    Age       *int       `json:"age,omitempty"`
    Active    bool       `json:"active"`
    CreatedAt time.Time  `json:"created_at"`
    UpdatedAt time.Time  `json:"updated_at"`
}

type CreateUserInput struct {
    Name  string
    Email string
    Age   *int
}

type UpdateUserInput struct {
    Name  *string
    Email *string
    Age   *int
}

type UserFilter struct {
    Active  *bool
    MinAge  *int
    MaxAge  *int
    Search  string
    Page    int
    Limit   int
}

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, input CreateUserInput) (*User, error) {
    var ageArg interface{}
    if input.Age != nil {
        ageArg = *input.Age
    }
    
    now := time.Now()
    result, err := r.db.ExecContext(ctx,
        `INSERT INTO users (name, email, age, active, created_at, updated_at) VALUES (?, ?, ?, TRUE, ?, ?)`,
        input.Name, input.Email, ageArg, now, now,
    )
    if err != nil {
        if isUniqueViolation(err) {
            return nil, ErrDuplicateEmail
        }
        return nil, fmt.Errorf("create user: %w", err)
    }
    
    id, _ := result.LastInsertId()
    return r.GetByID(ctx, id)
}

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*User, error) {
    row := r.db.QueryRowContext(ctx,
        `SELECT id, name, email, age, active, created_at, updated_at FROM users WHERE id = ?`, id,
    )
    return r.scanUser(row)
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    row := r.db.QueryRowContext(ctx,
        `SELECT id, name, email, age, active, created_at, updated_at FROM users WHERE email = ?`, email,
    )
    return r.scanUser(row)
}

func (r *UserRepository) List(ctx context.Context, filter UserFilter) ([]*User, int, error) {
    query := `SELECT id, name, email, age, active, created_at, updated_at FROM users WHERE 1=1`
    countQuery := `SELECT COUNT(*) FROM users WHERE 1=1`
    args := []interface{}{}
    
    if filter.Active != nil {
        clause := ` AND active = ?`
        query += clause
        countQuery += clause
        args = append(args, *filter.Active)
    }
    if filter.Search != "" {
        clause := ` AND (name LIKE ? OR email LIKE ?)`
        query += clause
        countQuery += clause
        args = append(args, "%"+filter.Search+"%", "%"+filter.Search+"%")
    }
    
    // Count total
    var total int
    countArgs := args
    r.db.QueryRowContext(ctx, countQuery, countArgs...).Scan(&total)
    
    // Pagination
    if filter.Limit <= 0 {
        filter.Limit = 10
    }
    if filter.Page <= 0 {
        filter.Page = 1
    }
    query += ` ORDER BY created_at DESC LIMIT ? OFFSET ?`
    args = append(args, filter.Limit, (filter.Page-1)*filter.Limit)
    
    rows, err := r.db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, 0, err
    }
    defer rows.Close()
    
    var users []*User
    for rows.Next() {
        u, err := r.scanUser(rows)
        if err != nil {
            return nil, 0, err
        }
        users = append(users, u)
    }
    
    return users, total, rows.Err()
}

func (r *UserRepository) Update(ctx context.Context, id int64, input UpdateUserInput) (*User, error) {
    setParts := []string{"updated_at = ?"}
    args := []interface{}{time.Now()}
    
    if input.Name != nil {
        setParts = append(setParts, "name = ?")
        args = append(args, *input.Name)
    }
    if input.Email != nil {
        setParts = append(setParts, "email = ?")
        args = append(args, *input.Email)
    }
    if input.Age != nil {
        setParts = append(setParts, "age = ?")
        args = append(args, *input.Age)
    }
    
    if len(setParts) == 1 { // เฉพาะ updated_at
        return r.GetByID(ctx, id)
    }
    
    // Build SET clause
    setClause := setParts[0]
    for _, part := range setParts[1:] {
        setClause += ", " + part
    }
    
    args = append(args, id)
    query := fmt.Sprintf(`UPDATE users SET %s WHERE id = ?`, setClause)
    
    result, err := r.db.ExecContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    
    n, _ := result.RowsAffected()
    if n == 0 {
        return nil, ErrNotFound
    }
    
    return r.GetByID(ctx, id)
}

func (r *UserRepository) Delete(ctx context.Context, id int64) error {
    result, err := r.db.ExecContext(ctx, `DELETE FROM users WHERE id = ?`, id)
    if err != nil {
        return err
    }
    n, _ := result.RowsAffected()
    if n == 0 {
        return ErrNotFound
    }
    return nil
}

// scanUser scans from *sql.Row or *sql.Rows
type scanner interface {
    Scan(dest ...interface{}) error
}

func (r *UserRepository) scanUser(s scanner) (*User, error) {
    var u User
    var age sql.NullInt64
    var createdAt, updatedAt string
    
    err := s.Scan(&u.ID, &u.Name, &u.Email, &age, &u.Active, &createdAt, &updatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, ErrNotFound
        }
        return nil, err
    }
    
    if age.Valid {
        a := int(age.Int64)
        u.Age = &a
    }
    
    u.CreatedAt, _ = time.Parse("2006-01-02 15:04:05", createdAt)
    u.UpdatedAt, _ = time.Parse("2006-01-02 15:04:05", updatedAt)
    
    return &u, nil
}

func isUniqueViolation(err error) bool {
    return err != nil && (fmt.Sprintf("%v", err) == "UNIQUE constraint failed: users.email")
}

func setupTestDB() *sql.DB {
    db, _ := sql.Open("sqlite3", ":memory:")
    db.Exec(`CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE,
        age INTEGER,
        active BOOLEAN DEFAULT TRUE,
        created_at TEXT,
        updated_at TEXT
    )`)
    return db
}

func main() {
    db := setupTestDB()
    defer db.Close()
    
    repo := NewUserRepository(db)
    ctx := context.Background()
    
    fmt.Println("=== User Repository Demo ===\n")
    
    // Create users
    age25 := 25
    user1, err := repo.Create(ctx, CreateUserInput{Name: "Alice", Email: "alice@example.com", Age: &age25})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created: %s (ID: %d)\n", user1.Name, user1.ID)
    
    age30 := 30
    user2, _ := repo.Create(ctx, CreateUserInput{Name: "Bob", Email: "bob@example.com", Age: &age30})
    fmt.Printf("Created: %s (ID: %d)\n", user2.Name, user2.ID)
    
    // Duplicate email test
    _, err = repo.Create(ctx, CreateUserInput{Name: "Alice2", Email: "alice@example.com"})
    if errors.Is(err, ErrDuplicateEmail) {
        fmt.Println("Got expected error: duplicate email")
    }
    
    // Get by ID
    found, err := repo.GetByID(ctx, user1.ID)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("\nFound user: %s, Age: %v\n", found.Name, found.Age)
    
    // List
    active := true
    users, total, err := repo.List(ctx, UserFilter{Active: &active, Page: 1, Limit: 10})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("\nList: %d users (total %d)\n", len(users), total)
    for _, u := range users {
        fmt.Printf("  - %s (%s)\n", u.Name, u.Email)
    }
    
    // Update
    newName := "Alice Updated"
    updated, err := repo.Update(ctx, user1.ID, UpdateUserInput{Name: &newName})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("\nUpdated: %s\n", updated.Name)
    
    // Delete
    err = repo.Delete(ctx, user2.ID)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Deleted user: %d\n", user2.ID)
    
    // Get deleted
    _, err = repo.GetByID(ctx, user2.ID)
    if errors.Is(err, ErrNotFound) {
        fmt.Println("Confirmed: user not found after delete")
    }
}
```

---

## สรุป Part 29

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Connection | `sql.Open()` + `db.Ping()` |
| Pool | `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime` |
| Query | `db.Query()` (หลาย rows), `db.QueryRow()` (row เดียว) |
| Exec | `db.Exec()` สำหรับ INSERT/UPDATE/DELETE |
| Scan | `rows.Scan()` - ต้อง match ลำดับกับ SELECT |
| Null | `sql.NullString`, `sql.NullInt64` หรือใช้ pointer |
| Transaction | `db.Begin()`, `tx.Commit()`, `tx.Rollback()` |
| Context | `db.QueryContext()`, `db.ExecContext()` |
| Prepared Stmt | `db.Prepare()` - ดีสำหรับ performance |
| Errors | ตรวจ `sql.ErrNoRows`, ตรวจ `rows.Err()` |

### Resources
- [database/sql package](https://pkg.go.dev/database/sql)
- [Go database/sql tutorial](https://go.dev/doc/database/index)
- [SQLite in Go](https://github.com/mattn/go-sqlite3)
