# Part 86: Go Compiler Internals และ Code Generation

## เป้าหมายของบทเรียน
- เข้าใจโครงสร้าง Go compiler
- ทำงานกับ AST (Abstract Syntax Tree)
- ใช้ go/ast package
- สร้าง code generation tools
- ใช้ go:generate
- Build tags และ compiler directives

---

## 1. Go Compiler Overview

```go
// compiler_overview.go - ภาพรวม Go compiler pipeline
package main

import (
    "fmt"
)

// Go Compiler Pipeline:
// Source Code (.go)
//    ↓ Lexer (tokenize)
// Tokens
//    ↓ Parser
// AST (Abstract Syntax Tree)
//    ↓ Type Checker
// Typed AST
//    ↓ SSA (Static Single Assignment)
// SSA IR
//    ↓ Optimization Passes
// Optimized SSA
//    ↓ Code Generator
// Machine Code (.o)
//    ↓ Linker
// Executable

func main() {
    fmt.Println("=== Go Compiler Pipeline ===")
    
    stages := []struct {
        stage  string
        tool   string
        output string
    }{
        {"Lexical Analysis", "go/scanner", "Tokens"},
        {"Parsing", "go/parser", "AST"},
        {"Type Checking", "go/types", "Typed AST"},
        {"SSA Construction", "golang.org/x/tools/go/ssa", "SSA IR"},
        {"Optimization", "cmd/compile/internal/ssa", "Optimized IR"},
        {"Code Generation", "cmd/compile/internal/gc", "Object Files"},
        {"Linking", "cmd/link", "Executable"},
    }
    
    for i, s := range stages {
        fmt.Printf("%d. %-20s [%-35s] -> %s\n", i+1, s.stage, s.tool, s.output)
    }
}
```

---

## 2. Lexer และ Tokenizer

```go
// lexer_demo.go - ใช้ go/scanner
package main

import (
    "fmt"
    "go/scanner"
    "go/token"
)

func tokenizeSource(src string) {
    fset := token.NewFileSet()
    file := fset.AddFile("example.go", fset.Base(), len(src))
    
    var s scanner.Scanner
    s.Init(file, []byte(src), nil, scanner.ScanComments)
    
    fmt.Printf("Tokenizing: %q\n\n", src)
    fmt.Printf("%-15s %-20s %s\n", "Position", "Token", "Literal")
    fmt.Println(string(make([]byte, 50)))
    
    for {
        pos, tok, lit := s.Scan()
        if tok == token.EOF {
            break
        }
        
        fmt.Printf("%-15s %-20s %q\n",
            fset.Position(pos),
            tok.String(),
            lit,
        )
    }
}

func main() {
    // Example 1: Simple function
    src1 := `package main

func add(a, b int) int {
    return a + b
}`
    tokenizeSource(src1)
    
    fmt.Println("\n---")
    
    // Example 2: Interface
    src2 := `type Stringer interface {
    String() string
}`
    tokenizeSource(src2)
}
```

---

## 3. AST Parsing และ Inspection

```go
// ast_parsing.go - Parse และ inspect AST
package main

import (
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
)

func parseAndInspect(src string) {
    fset := token.NewFileSet()
    
    f, err := parser.ParseFile(fset, "example.go", src, parser.AllErrors|parser.ParseComments)
    if err != nil {
        fmt.Printf("Parse error: %v\n", err)
        return
    }
    
    fmt.Println("=== Package:", f.Name.Name)
    
    // วนดู declarations
    for _, decl := range f.Decls {
        switch d := decl.(type) {
        case *ast.FuncDecl:
            fmt.Printf("\nFunction: %s\n", d.Name.Name)
            
            // Parameters
            if d.Type.Params.NumFields() > 0 {
                fmt.Print("  Params: ")
                for _, field := range d.Type.Params.List {
                    for _, name := range field.Names {
                        fmt.Printf("%s ", name.Name)
                    }
                }
                fmt.Println()
            }
            
            // Returns
            if d.Type.Results != nil && d.Type.Results.NumFields() > 0 {
                fmt.Printf("  Returns: %d values\n", d.Type.Results.NumFields())
            }
            
            // Body statements
            if d.Body != nil {
                fmt.Printf("  Body: %d statements\n", len(d.Body.List))
            }
            
        case *ast.GenDecl:
            fmt.Printf("\nDeclaration: %s\n", d.Tok.String())
            for _, spec := range d.Specs {
                switch s := spec.(type) {
                case *ast.TypeSpec:
                    fmt.Printf("  Type: %s\n", s.Name.Name)
                case *ast.ValueSpec:
                    for _, name := range s.Names {
                        fmt.Printf("  Value: %s\n", name.Name)
                    }
                }
            }
        }
    }
}

// astWalker ใช้ ast.Walk เพื่อ traverse
func astWalker(src string) {
    fset := token.NewFileSet()
    f, _ := parser.ParseFile(fset, "", src, 0)
    
    fmt.Println("\n=== AST Walk ===")
    
    ast.Inspect(f, func(n ast.Node) bool {
        if n == nil {
            return false
        }
        
        pos := fset.Position(n.Pos())
        
        switch x := n.(type) {
        case *ast.Ident:
            fmt.Printf("  Ident '%s' at %s\n", x.Name, pos)
        case *ast.BasicLit:
            fmt.Printf("  Literal %s='%s' at %s\n", x.Kind, x.Value, pos)
        case *ast.CallExpr:
            fmt.Printf("  Call at %s\n", pos)
        }
        
        return true
    })
}

func main() {
    src := `package main

import "fmt"

// add บวกเลขสองตัว
func add(a, b int) int {
    return a + b
}

func greet(name string) string {
    msg := "Hello, " + name
    return msg
}

const Pi = 3.14159

var counter int`

    parseAndInspect(src)
    astWalker(src)
}
```

---

## 4. AST Modification

```go
// ast_modify.go - แก้ไข AST
package main

import (
    "bytes"
    "fmt"
    "go/ast"
    "go/format"
    "go/parser"
    "go/token"
)

// addLogging เพิ่ม logging เข้าทุก function
func addLogging(src string) (string, error) {
    fset := token.NewFileSet()
    f, err := parser.ParseFile(fset, "", src, parser.ParseComments)
    if err != nil {
        return "", err
    }
    
    ast.Inspect(f, func(n ast.Node) bool {
        fn, ok := n.(*ast.FuncDecl)
        if !ok || fn.Body == nil {
            return true
        }
        
        // สร้าง log statement: fmt.Printf("Entering %s\n", "funcName")
        logStmt := &ast.ExprStmt{
            X: &ast.CallExpr{
                Fun: &ast.SelectorExpr{
                    X:   ast.NewIdent("fmt"),
                    Sel: ast.NewIdent("Printf"),
                },
                Args: []ast.Expr{
                    &ast.BasicLit{
                        Kind:  token.STRING,
                        Value: fmt.Sprintf(`"Entering %s\n"`, fn.Name.Name),
                    },
                },
            },
        }
        
        // แทรกที่ต้น body
        fn.Body.List = append([]ast.Stmt{logStmt}, fn.Body.List...)
        
        return true
    })
    
    // Format back to source
    var buf bytes.Buffer
    if err := format.Node(&buf, fset, f); err != nil {
        return "", err
    }
    
    return buf.String(), nil
}

// renameIdentifier เปลี่ยนชื่อ identifier
func renameIdentifier(src, from, to string) (string, error) {
    fset := token.NewFileSet()
    f, err := parser.ParseFile(fset, "", src, 0)
    if err != nil {
        return "", err
    }
    
    ast.Inspect(f, func(n ast.Node) bool {
        ident, ok := n.(*ast.Ident)
        if ok && ident.Name == from {
            ident.Name = to
        }
        return true
    })
    
    var buf bytes.Buffer
    format.Node(&buf, fset, f)
    return buf.String(), nil
}

func main() {
    src := `package main

import "fmt"

func calculate(x, y int) int {
    return x + y
}

func greet(name string) {
    fmt.Println("Hello,", name)
}`

    fmt.Println("=== Original Source ===")
    fmt.Println(src)
    
    fmt.Println("\n=== After Adding Logging ===")
    result, err := addLogging(src)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Println(result)
    }
    
    fmt.Println("\n=== After Renaming 'calculate' to 'compute' ===")
    renamed, err := renameIdentifier(src, "calculate", "compute")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Println(renamed)
    }
}
```

---

## 5. Code Generation ด้วย go:generate

```go
// gen/gen.go - code generator
//go:build ignore

package main

import (
    "bytes"
    "fmt"
    "go/format"
    "os"
    "text/template"
    "time"
)

// EnumDef นิยาม enum ที่จะ generate
type EnumDef struct {
    Package  string
    Name     string
    Values   []string
    Timestamp string
}

// enumTemplate template สำหรับ generate enum
const enumTemplate = `// Code generated by go:generate. DO NOT EDIT.
// Generated at: {{.Timestamp}}

package {{.Package}}

import "fmt"

// {{.Name}} enum type
type {{.Name}} int

const (
{{range $i, $v := .Values}}    {{$.Name}}{{$v}}{{if eq $i 0}} {{$.Name}} = iota{{end}}
{{end}})

// String returns string representation
func (e {{.Name}}) String() string {
    names := []string{
{{range .Values}}        "{{.}}",
{{end}}    }
    if int(e) >= 0 && int(e) < len(names) {
        return names[e]
    }
    return fmt.Sprintf("{{.Name}}(%d)", int(e))
}

// All{{.Name}}s returns all {{.Name}} values
func All{{.Name}}s() []{{.Name}} {
    return []{{.Name}}{
{{range .Values}}        {{$.Name}}{{.}},
{{end}}    }
}

// Parse{{.Name}} parses string to {{.Name}}
func Parse{{.Name}}(s string) ({{.Name}}, error) {
    for _, v := range All{{.Name}}s() {
        if v.String() == s {
            return v, nil
        }
    }
    return 0, fmt.Errorf("unknown {{.Name}}: %q", s)
}
`

func generateEnum(pkg, name string, values []string, outputFile string) error {
    tmpl, err := template.New("enum").Parse(enumTemplate)
    if err != nil {
        return fmt.Errorf("template parse error: %w", err)
    }
    
    def := EnumDef{
        Package:   pkg,
        Name:      name,
        Values:    values,
        Timestamp: time.Now().Format(time.RFC3339),
    }
    
    var buf bytes.Buffer
    if err := tmpl.Execute(&buf, def); err != nil {
        return fmt.Errorf("template execute error: %w", err)
    }
    
    // Format with gofmt
    formatted, err := format.Source(buf.Bytes())
    if err != nil {
        // Return unformatted if format fails
        formatted = buf.Bytes()
    }
    
    return os.WriteFile(outputFile, formatted, 0644)
}

func main() {
    // Generate Status enum
    err := generateEnum(
        "main",
        "Status",
        []string{"Pending", "Active", "Inactive", "Deleted"},
        "status_gen.go",
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    
    fmt.Println("Generated status_gen.go")
    
    // Generate Color enum
    err = generateEnum(
        "main",
        "Color",
        []string{"Red", "Green", "Blue", "Yellow", "Purple"},
        "color_gen.go",
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    
    fmt.Println("Generated color_gen.go")
}
```

---

## 6. Build Tags

```go
// build_tags_demo.go - แสดงการใช้ build tags
package main

import "fmt"

// ไฟล์นี้แสดงตัวอย่าง build tags ต่างๆ

/*
Build Tags Examples:

1. OS-specific:
   //go:build linux
   //go:build windows
   //go:build darwin

2. Architecture-specific:
   //go:build amd64
   //go:build arm64

3. Go version:
   //go:build go1.18

4. Custom tags:
   //go:build integration
   //go:build !production

5. Combined:
   //go:build linux && amd64
   //go:build linux || darwin
   //go:build !windows
*/

// ตัวอย่างการ build:
// go build -tags integration ./...
// go test -tags=production ./...
// GOOS=linux GOARCH=amd64 go build ./...

func main() {
    fmt.Println("Build tag demo")
    fmt.Printf("Current platform info would be set by build constraints\n")
    
    // ตรวจสอบ build environment
    checkBuildEnv()
}

func checkBuildEnv() {
    // ในโค้ดจริง ค่าเหล่านี้จะมาจากไฟล์ที่มี build tags
    envVars := map[string]string{
        "GOOS":   "linux",   // ตัวอย่าง
        "GOARCH": "amd64",   // ตัวอย่าง
        "CGO":    "enabled", // ตัวอย่าง
    }
    
    fmt.Println("\nBuild environment:")
    for k, v := range envVars {
        fmt.Printf("  %s = %s\n", k, v)
    }
}
```

---

## 7. Compiler Directives

```go
// compiler_directives.go - Go compiler directives
package main

import (
    "fmt"
    "unsafe"
)

// go:noinline บอก compiler ไม่ให้ inline function นี้
//
//go:noinline
func expensiveOperation(n int) int {
    sum := 0
    for i := 0; i < n; i++ {
        sum += i
    }
    return sum
}

// go:nosplit บอก compiler ไม่ให้ทำ stack split
//
//go:nosplit
func criticalPath(a, b int) int {
    return a + b
}

// ตัวอย่างการใช้ unsafe package
func unsafeDemo() {
    x := 42
    
    // ดู memory layout
    ptr := unsafe.Pointer(&x)
    fmt.Printf("Value: %d\n", x)
    fmt.Printf("Pointer: %p\n", ptr)
    fmt.Printf("Size: %d bytes\n", unsafe.Sizeof(x))
    fmt.Printf("Alignment: %d bytes\n", unsafe.Alignof(x))
}

// struct memory layout
type Point struct {
    X float64  // 8 bytes
    Y float64  // 8 bytes
}

type MixedStruct struct {
    A bool    // 1 byte
    // 7 bytes padding
    B float64 // 8 bytes
    C bool    // 1 byte
    D int16   // 2 bytes
    // 5 bytes padding
}

type PackedStruct struct {
    B float64 // 8 bytes
    D int16   // 2 bytes
    A bool    // 1 byte
    C bool    // 1 byte
    // 4 bytes padding
}

func memoryLayoutDemo() {
    fmt.Printf("\nMemory Layout:\n")
    fmt.Printf("  Point:        %d bytes\n", unsafe.Sizeof(Point{}))
    fmt.Printf("  MixedStruct:  %d bytes\n", unsafe.Sizeof(MixedStruct{}))
    fmt.Printf("  PackedStruct: %d bytes\n", unsafe.Sizeof(PackedStruct{}))
    
    // Field offsets
    var p MixedStruct
    fmt.Printf("\nMixedStruct offsets:\n")
    fmt.Printf("  A: %d\n", unsafe.Offsetof(p.A))
    fmt.Printf("  B: %d\n", unsafe.Offsetof(p.B))
    fmt.Printf("  C: %d\n", unsafe.Offsetof(p.C))
    fmt.Printf("  D: %d\n", unsafe.Offsetof(p.D))
}

func main() {
    fmt.Println("=== Compiler Directives Demo ===\n")
    
    result := expensiveOperation(100)
    fmt.Printf("expensiveOperation(100) = %d\n", result)
    
    sum := criticalPath(10, 20)
    fmt.Printf("criticalPath(10, 20) = %d\n", sum)
    
    unsafeDemo()
    memoryLayoutDemo()
}
```

---

## สรุป

บทนี้ครอบคลุม:

1. **Go Compiler Pipeline** - ขั้นตอนจาก source code ถึง executable
2. **Lexer/Scanner** - การ tokenize source code
3. **AST** - การ parse และ inspect Abstract Syntax Tree
4. **AST Modification** - การแก้ไข code ผ่าน AST
5. **Code Generation** - ใช้ templates สร้าง Go code อัตโนมัติ
6. **Build Tags** - conditional compilation
7. **Compiler Directives** - `go:noinline`, `go:nosplit`, unsafe

### Key Takeaways

- `go/ast` package ช่วยให้เราทำงานกับ Go code เป็น data structure
- Code generation ช่วยลด boilerplate code
- Build tags ช่วยสร้าง platform-specific code
- Compiler directives ช่วย optimize critical paths
