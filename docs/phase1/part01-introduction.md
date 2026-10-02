# Part 1: แนะนำ Go (Golang) - Installation & Hello World

## 🎯 เป้าหมายการเรียนรู้
- เข้าใจว่า Go คืออะไรและทำไมถึงนิยมใช้
- ติดตั้ง Go บน macOS, Linux, Windows
- เขียนโปรแกรม Go แรก
- เข้าใจโครงสร้างพื้นฐานของ Go program
- ใช้ Go CLI commands พื้นฐาน

---

## 1.1 Go คืออะไร?

Go (หรือ Golang) คือภาษาโปรแกรมที่พัฒนาโดย Google ในปี 2009 โดย Robert Griesemer, Rob Pike และ Ken Thompson

### จุดเด่นของ Go
```
✅ Compiled Language - รวดเร็วมาก
✅ Statically Typed - ป้องกัน bugs ตั้งแต่ compile time
✅ Garbage Collection - จัดการ memory อัตโนมัติ
✅ Built-in Concurrency - goroutines & channels
✅ Simple Syntax - เรียนรู้ง่าย เขียนได้เร็ว
✅ Fast Compilation - compile ไว
✅ Cross-platform - รันได้ทุก OS
✅ Strong Standard Library - มี library ครบ
```

### Go ใช้ทำอะไร?
- **Backend APIs** - REST, gRPC, GraphQL
- **Microservices** - Netflix, Uber, Dropbox ใช้
- **DevOps Tools** - Docker, Kubernetes เขียนด้วย Go
- **System Programming** - CLI tools, daemons
- **Cloud Services** - Google Cloud, AWS Lambda
- **Blockchain** - Ethereum tools, Hyperledger

### Go vs ภาษาอื่น
| ภาษา | ความเร็ว | ความง่าย | Concurrency | Use Case |
|------|---------|---------|-------------|---------|
| Go | ⚡⚡⚡⚡ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Backend, Systems |
| Python | ⚡⚡ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ML, Scripting |
| Java | ⚡⚡⚡ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Enterprise |
| Node.js | ⚡⚡⚡ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Web, Real-time |
| Rust | ⚡⚡⚡⚡⚡ | ⭐⭐ | ⭐⭐⭐⭐ | Systems, Safety |
| C++ | ⚡⚡⚡⚡⚡ | ⭐ | ⭐⭐⭐ | Games, Systems |

---

## 1.2 ติดตั้ง Go

### macOS
```bash
# วิธีที่ 1: ใช้ Homebrew (แนะนำ)
brew install go

# วิธีที่ 2: ดาวน์โหลดจาก go.dev
# ไปที่ https://go.dev/dl/
# ดาวน์โหลด go1.22.x.darwin-amd64.pkg หรือ arm64 สำหรับ M1/M2
# ติดตั้งตามขั้นตอน

# ตรวจสอบ
go version
# Output: go version go1.22.0 darwin/arm64
```

### Linux (Ubuntu/Debian)
```bash
# ดาวน์โหลด Go
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz

# ลบเวอร์ชันเก่า (ถ้ามี) และติดตั้งใหม่
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz

# เพิ่ม PATH ใน ~/.bashrc หรือ ~/.zshrc
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc

# reload
source ~/.bashrc

# ตรวจสอบ
go version
```

### Windows
```powershell
# วิธีที่ 1: ใช้ Chocolatey
choco install golang

# วิธีที่ 2: ดาวน์โหลด installer จาก go.dev
# ดาวน์โหลด go1.22.x.windows-amd64.msi
# รัน installer และทำตามขั้นตอน

# ตรวจสอบใน PowerShell
go version
```

### ตั้งค่า GOPATH และ Go Environment
```bash
# ดูค่า environment ทั้งหมด
go env

# ค่าสำคัญ:
# GOPATH    - ที่เก็บ packages ที่ download มา (default: ~/go)
# GOROOT    - ที่ติดตั้ง Go
# GOMODCACHE - cache ของ modules
# GOOS      - operating system เป้าหมาย
# GOARCH    - architecture เป้าหมาย

# ตัวอย่าง output
GOPATH="/home/user/go"
GOROOT="/usr/local/go"
GOMODCACHE="/home/user/go/pkg/mod"
GOOS="linux"
GOARCH="amd64"
```

### ติดตั้ง VS Code Extension (แนะนำ)
```
1. เปิด VS Code
2. กด Ctrl+P (หรือ Cmd+P บน Mac)
3. พิมพ์: ext install golang.go
4. กด Enter และ Install
5. VS Code จะแนะนำให้ติดตั้ง tools เพิ่มเติม - กด "Install All"
```

---

## 1.3 โปรแกรมแรก: Hello, World!

### สร้าง Project
```bash
# สร้าง folder
mkdir hello-world
cd hello-world

# เริ่ม Go module (สำคัญมาก!)
go mod init hello-world

# จะสร้างไฟล์ go.mod
cat go.mod
# module hello-world
# 
# go 1.22.0
```

### เขียน Hello World
```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

### รันโปรแกรม
```bash
# รันโดยตรง (ไม่ต้อง compile)
go run main.go
# Output: Hello, World!

# Compile เป็น executable
go build -o hello main.go
# สร้างไฟล์ hello (หรือ hello.exe บน Windows)

# รัน executable
./hello
# Output: Hello, World!

# Build สำหรับ OS อื่น (Cross Compilation)
GOOS=windows GOARCH=amd64 go build -o hello.exe main.go
GOOS=linux GOARCH=amd64 go build -o hello-linux main.go
GOOS=darwin GOARCH=arm64 go build -o hello-mac main.go
```

---

## 1.4 โครงสร้างของ Go Program

### องค์ประกอบพื้นฐาน
```go
// 1. Package Declaration - บอกว่าไฟล์นี้อยู่ใน package อะไร
package main

// 2. Import - นำ package อื่นมาใช้
import (
    "fmt"     // format และ print
    "os"      // operating system
    "strings" // string operations
)

// 3. Constants - ค่าคงที่
const AppName = "My Go App"
const Version = "1.0.0"

// 4. Variables - ตัวแปร global
var author = "Go Developer"

// 5. Functions - ฟังก์ชัน (main เป็น entry point)
func main() {
    // 6. Statements - คำสั่งต่างๆ
    fmt.Println("App:", AppName)
    fmt.Println("Version:", Version)
    fmt.Println("Author:", author)
    
    // ใช้ os package
    fmt.Println("OS Args:", os.Args)
    
    // ใช้ strings package
    message := "Hello, Go!"
    fmt.Println(strings.ToUpper(message))
}
```

### Package main vs Package library
```go
// Package main - โปรแกรมหลัก (สร้าง executable)
package main

func main() {
    // entry point ของโปรแกรม
}

// Package อื่น - library (ไม่มี main function)
package utils

func Add(a, b int) int {
    return a + b
}
```

### Naming Conventions ใน Go
```go
// ✅ Exported (Public) - ขึ้นต้นด้วยตัวพิมพ์ใหญ่
// สามารถใช้ได้จาก package อื่น
func PublicFunction() {}
var PublicVariable = "public"
type PublicStruct struct{}

// ✅ Unexported (Private) - ขึ้นต้นด้วยตัวพิมพ์เล็ก
// ใช้ได้เฉพาะใน package เดียวกัน
func privateFunction() {}
var privateVariable = "private"
type privateStruct struct{}

// ✅ camelCase สำหรับ variables และ functions
var userName string
func getUserName() string { return "" }

// ✅ PascalCase สำหรับ exported names
type UserProfile struct{}
func GetUserProfile() {}

// ✅ ALL_CAPS สำหรับ constants ที่เป็น global (บางครั้ง)
// แต่ Go style ปกติใช้ PascalCase
const MaxRetries = 3
const DEFAULT_TIMEOUT = 30 // ไม่ค่อยนิยม
```

---

## 1.5 Go CLI Commands ที่สำคัญ

### คำสั่งพื้นฐาน
```bash
# รันโปรแกรม
go run main.go
go run .  # รันทุก .go files ใน directory นี้

# Build
go build                    # build เป็นชื่อ directory
go build -o myapp          # กำหนดชื่อ output
go build ./...             # build ทุก packages

# Test
go test                    # รัน tests ใน directory นี้
go test ./...              # รัน tests ทุก packages
go test -v                 # verbose mode
go test -run TestName      # รัน test เฉพาะชื่อ
go test -cover             # แสดง test coverage

# Install (build & install ลง $GOPATH/bin)
go install

# Clean
go clean                   # ลบ build files
go clean -cache            # ลบ build cache

# Format code
go fmt ./...               # format ทุกไฟล์

# Lint / vet
go vet ./...               # ตรวจสอบ code issues

# Documentation
go doc fmt.Println         # ดู doc ของ function
go doc -all fmt            # ดู doc ทั้ง package
```

### Module Commands
```bash
# เริ่ม module ใหม่
go mod init module-name

# เพิ่ม dependencies
go get github.com/some/package
go get github.com/some/package@v1.2.3    # เวอร์ชันเฉพาะ
go get github.com/some/package@latest    # เวอร์ชันล่าสุด

# อัพเดท dependencies
go get -u ./...            # update ทั้งหมด
go get -u github.com/some/package  # update เฉพาะ package

# ทำความสะอาด dependencies
go mod tidy                # เพิ่ม missing, ลบ unused

# Download dependencies
go mod download

# ดู dependency graph
go mod graph

# Vendor mode (เก็บ deps ใน vendor folder)
go mod vendor
go build -mod=vendor
```

---

## 1.6 โปรแกรมแรก: พัฒนาต่อ

### โปรแกรมที่ซับซ้อนขึ้น
```go
// main.go
package main

import (
    "fmt"
    "os"
    "strings"
    "time"
)

// Constant
const (
    AppName    = "Go Hello World"
    AppVersion = "1.0.0"
)

// Function
func greet(name string) string {
    return fmt.Sprintf("สวัสดี, %s! ยินดีต้อนรับสู่ Go", name)
}

// Function with multiple returns
func getSystemInfo() (string, string) {
    return runtime_goos(), time.Now().Format("2006-01-02 15:04:05")
}

// helper (ในโปรแกรมจริงจะ import "runtime")
func runtime_goos() string {
    // ใช้ os package แทน
    info := "Unknown OS"
    if strings.Contains(os.Getenv("HOME"), "/home") {
        info = "Linux"
    } else if strings.Contains(os.Getenv("HOME"), "/Users") {
        info = "macOS"
    }
    return info
}

func main() {
    // Print header
    fmt.Println("=================================")
    fmt.Printf("  %s v%s\n", AppName, AppVersion)
    fmt.Println("=================================")
    
    // รับ arguments จาก command line
    args := os.Args
    var name string
    
    if len(args) > 1 {
        name = args[1]
    } else {
        name = "World"
    }
    
    // Greeting
    message := greet(name)
    fmt.Println(message)
    
    // System info
    osInfo, currentTime := getSystemInfo()
    fmt.Printf("ระบบ: %s\n", osInfo)
    fmt.Printf("เวลา: %s\n", currentTime)
    
    // Environment
    goVersion := os.Getenv("GOVERSION")
    if goVersion == "" {
        goVersion = "ไม่ทราบ"
    }
    fmt.Printf("Go Version: %s\n", goVersion)
}
```

รันโปรแกรม:
```bash
go run main.go
go run main.go "สมชาย"
go run main.go "Go Developer"
```

---

## 1.7 Go Workspace Structure

### โครงสร้าง Project แบบ Standard
```
my-go-project/
├── go.mod              # module definition
├── go.sum              # checksum database
├── main.go             # entry point
├── README.md           # documentation
├── .gitignore          # git ignore
│
├── cmd/                # command-line applications
│   ├── api/
│   │   └── main.go     # API server entry point
│   └── cli/
│       └── main.go     # CLI tool entry point
│
├── internal/           # private packages (ใช้ได้เฉพาะ module นี้)
│   ├── config/
│   │   └── config.go
│   └── database/
│       └── db.go
│
├── pkg/                # public packages (ใช้ได้จากภายนอก)
│   ├── models/
│   │   └── user.go
│   └── utils/
│       └── helper.go
│
├── api/                # API definitions (OpenAPI, proto files)
│   └── openapi.yaml
│
├── web/                # web assets
│   ├── templates/
│   └── static/
│
├── scripts/            # build, deploy scripts
│   ├── build.sh
│   └── deploy.sh
│
├── tests/              # integration tests
│   └── integration_test.go
│
└── docs/               # documentation
    └── architecture.md
```

### go.mod ไฟล์
```go
module github.com/username/my-go-project

go 1.22.0

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-sql-driver/mysql v1.7.1
    golang.org/x/crypto v0.18.0
)

require (
    // indirect dependencies
    github.com/some/dep v1.0.0 // indirect
)
```

### go.sum ไฟล์
```
# go.sum เก็บ checksums ของ dependencies
# ไม่ต้องแก้ไขเอง - go mod tidy จัดการให้
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

---

## 1.8 Go Playground

สามารถทดลองเขียน Go ออนไลน์ได้ที่ https://go.dev/play/

```go
// ตัวอย่างใน Go Playground
package main

import "fmt"

func main() {
    // ทดลองโค้ดต่างๆ ที่นี่ได้เลย
    for i := 1; i <= 5; i++ {
        fmt.Printf("นับ: %d\n", i)
    }
}
```

---

## 1.9 กฎและ Best Practices ของ Go

### Gofmt - Code Formatting
```bash
# Go มี formatter ในตัว - ต้อง format ทุกครั้ง
gofmt -w main.go      # format ไฟล์เดียว
go fmt ./...          # format ทุกไฟล์ใน project

# ตัวอย่าง code ก่อน format:
func main(){fmt.Println("hello")
x:=1
   y :=2
}

# หลัง format:
func main() {
    fmt.Println("hello")
    x := 1
    y := 2
}
```

### Goimports - Import Management
```bash
# ติดตั้ง
go install golang.org/x/tools/cmd/goimports@latest

# ใช้งาน
goimports -w main.go
```

### Go Vet - Static Analysis
```bash
# ตรวจสอบ code ที่ถูกต้องแต่อาจมี bugs
go vet ./...

# ตัวอย่าง issue ที่ go vet จับได้
fmt.Printf("%d", "string")  // wrong format specifier
```

### golangci-lint - Comprehensive Linting
```bash
# ติดตั้ง
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# ใช้งาน
golangci-lint run ./...
```

---

## 1.10 การจัดการ Error ตั้งแต่เริ่มต้น

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func readFile(filename string) (string, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return "", fmt.Errorf("ไม่สามารถอ่านไฟล์ %s: %w", filename, err)
    }
    return string(data), nil
}

func main() {
    // การ handle error แบบ Go idiom
    content, err := readFile("test.txt")
    if err != nil {
        // จัดการ error
        var pathErr *os.PathError
        if errors.As(err, &pathErr) {
            fmt.Printf("ไม่พบไฟล์: %s\n", pathErr.Path)
        } else {
            fmt.Printf("Error: %v\n", err)
        }
        os.Exit(1)
    }
    
    fmt.Println("เนื้อหาไฟล์:", content)
}
```

---

## 1.11 Workshop: โปรแกรม Calculator อย่างง่าย

```go
// calculator.go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
)

func add(a, b float64) float64 { return a + b }
func subtract(a, b float64) float64 { return a - b }
func multiply(a, b float64) float64 { return a * b }
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("ไม่สามารถหารด้วย 0 ได้")
    }
    return a / b, nil
}

func calculate(a float64, op string, b float64) (float64, error) {
    switch op {
    case "+":
        return add(a, b), nil
    case "-":
        return subtract(a, b), nil
    case "*":
        return multiply(a, b), nil
    case "/":
        return divide(a, b)
    default:
        return 0, fmt.Errorf("ไม่รู้จัก operator: %s", op)
    }
}

func main() {
    fmt.Println("=== Go Calculator ===")
    fmt.Println("รูปแบบ: <ตัวเลข> <+|-|*|/> <ตัวเลข>")
    fmt.Println("พิมพ์ 'quit' เพื่อออก")
    fmt.Println()
    
    scanner := bufio.NewScanner(os.Stdin)
    
    for {
        fmt.Print(">>> ")
        scanner.Scan()
        input := strings.TrimSpace(scanner.Text())
        
        if input == "quit" || input == "exit" {
            fmt.Println("ลาก่อน!")
            break
        }
        
        if input == "" {
            continue
        }
        
        parts := strings.Fields(input)
        if len(parts) != 3 {
            fmt.Println("Error: รูปแบบไม่ถูกต้อง ต้องการ: <ตัวเลข> <operator> <ตัวเลข>")
            continue
        }
        
        a, err := strconv.ParseFloat(parts[0], 64)
        if err != nil {
            fmt.Printf("Error: '%s' ไม่ใช่ตัวเลข\n", parts[0])
            continue
        }
        
        b, err := strconv.ParseFloat(parts[2], 64)
        if err != nil {
            fmt.Printf("Error: '%s' ไม่ใช่ตัวเลข\n", parts[2])
            continue
        }
        
        result, err := calculate(a, parts[1], b)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            continue
        }
        
        fmt.Printf("= %.4g\n", result)
    }
}
```

รันและทดสอบ:
```bash
go run calculator.go

# ทดสอบ:
# >>> 10 + 5
# = 15
# >>> 100 / 4
# = 25
# >>> 7 * 8
# = 56
# >>> 10 / 0
# Error: ไม่สามารถหารด้วย 0 ได้
# >>> quit
# ลาก่อน!
```

---

## 1.12 สรุป

ในบทนี้เราได้เรียนรู้:
- ✅ Go คืออะไร และข้อดีของมัน
- ✅ ติดตั้ง Go บนทุก OS
- ✅ เขียนโปรแกรม Hello World แรก
- ✅ โครงสร้างพื้นฐานของ Go program
- ✅ Go CLI commands สำคัญ
- ✅ โครงสร้าง project แบบ standard
- ✅ Best practices เบื้องต้น

## 📝 Exercise

1. ติดตั้ง Go และ VS Code + Go extension
2. สร้าง project `hello-world` และรันได้สำเร็จ
3. แก้ไข Hello World ให้รับ name จาก command line argument
4. เพิ่ม flag `-upper` เพื่อแสดงข้อความเป็นตัวพิมพ์ใหญ่
5. Build เป็น executable และรันได้

## 🔗 Resources
- [Go Official Website](https://go.dev)
- [Go Tour](https://go.dev/tour)
- [Go Playground](https://go.dev/play)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go by Example](https://gobyexample.com)

---
*Part 1 จาก 100 | Phase 1: พื้นฐาน Go*
