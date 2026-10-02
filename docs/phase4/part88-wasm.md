# Part 88: WebAssembly กับ Go

## เป้าหมายของบทเรียน
- เข้าใจ WebAssembly (WASM) basics
- Compile Go ไปเป็น WASM
- ใช้ syscall/js เพื่อ interact กับ browser
- WASI สำหรับ server-side WASM
- TinyGo สำหรับ small binaries

---

## 1. WebAssembly Basics

```go
// wasm_basics.go - ความรู้พื้นฐาน WASM
package main

import (
    "fmt"
)

/*
WebAssembly (WASM):
- Binary instruction format สำหรับ stack-based VM
- Design goals: fast, safe, portable, compact
- รันใน browsers และ server environments

Go → WASM Pipeline:
1. เขียน Go code
2. Compile: GOOS=js GOARCH=wasm go build -o main.wasm .
3. Load ใน browser ด้วย wasm_exec.js (จาก Go distribution)
4. JavaScript calls Go functions และ vice versa

File sizes:
- Standard Go: ~2-5 MB (includes runtime)
- TinyGo: ~100-500 KB
- Compressed: ลด 70-80% ด้วย gzip/brotli

WASM Data Types:
- i32, i64: integers
- f32, f64: floats
- v128: SIMD vectors (WASM SIMD)

Memory:
- Linear memory model
- Shared แบบ optional (SharedArrayBuffer)
*/

func main() {
    fmt.Println("=== WebAssembly Concepts ===")
    
    concepts := map[string]string{
        "Module":    "Binary WASM file (.wasm)",
        "Instance":  "Instantiated module with memory",
        "Memory":    "Linear memory (growable ArrayBuffer)",
        "Table":     "Array of function references",
        "Import":    "Functions provided by host (JS/WASI)",
        "Export":    "Functions exposed to host",
        "Trap":      "Runtime error (divide by zero, OOB access)",
    }
    
    fmt.Println("WASM Concepts:")
    for term, desc := range concepts {
        fmt.Printf("  %-12s: %s\n", term, desc)
    }
    
    compileTargets := []struct {
        target string
        goos   string
        goarch string
        useFor string
    }{
        {"Browser WASM", "js", "wasm", "Web apps"},
        {"WASI", "wasip1", "wasm", "Server-side, edge computing"},
        {"TinyGo Browser", "js", "wasm", "Tiny web components"},
        {"TinyGo WASI", "wasip1", "wasm", "Microcontrollers, edge"},
    }
    
    fmt.Println("\nCompile targets:")
    for _, t := range compileTargets {
        fmt.Printf("  GOOS=%-8s GOARCH=%-6s -> %s (%s)\n",
            t.goos, t.goarch, t.target, t.useFor)
    }
}
```

---

## 2. syscall/js - Browser Interaction

```go
//go:build js && wasm

// browser_wasm.go - รันใน browser
package main

import (
    "fmt"
    "syscall/js"
)

// registerFunctions ลงทะเบียน Go functions ให้ JavaScript เรียกได้
func registerFunctions() {
    // Register Go function เป็น JS function
    js.Global().Set("goAdd", js.FuncOf(add))
    js.Global().Set("goGreet", js.FuncOf(greet))
    js.Global().Set("goFibonacci", js.FuncOf(fibonacci))
    js.Global().Set("goSort", js.FuncOf(sortArray))
    
    fmt.Println("Go functions registered in JavaScript global scope")
}

// add รับ JavaScript arguments และ return value
func add(this js.Value, args []js.Value) interface{} {
    if len(args) < 2 {
        return map[string]interface{}{
            "error": "need 2 arguments",
        }
    }
    
    a := args[0].Float()
    b := args[1].Float()
    result := a + b
    
    return result
}

// greet สร้าง greeting message
func greet(this js.Value, args []js.Value) interface{} {
    name := "World"
    if len(args) > 0 {
        name = args[0].String()
    }
    
    return fmt.Sprintf("สวัสดี, %s! จาก Go WASM", name)
}

// fibonacci คำนวณ Fibonacci
func fibonacci(this js.Value, args []js.Value) interface{} {
    if len(args) == 0 {
        return 0
    }
    
    n := args[0].Int()
    return fibCalc(n)
}

func fibCalc(n int) int {
    if n <= 1 {
        return n
    }
    return fibCalc(n-1) + fibCalc(n-2)
}

// sortArray sort JavaScript array
func sortArray(this js.Value, args []js.Value) interface{} {
    if len(args) == 0 {
        return nil
    }
    
    jsArr := args[0]
    length := jsArr.Length()
    
    // Copy to Go slice
    arr := make([]float64, length)
    for i := 0; i < length; i++ {
        arr[i] = jsArr.Index(i).Float()
    }
    
    // Sort
    for i := 0; i < len(arr); i++ {
        for j := i + 1; j < len(arr); j++ {
            if arr[j] < arr[i] {
                arr[i], arr[j] = arr[j], arr[i]
            }
        }
    }
    
    // Convert back to JS array
    result := js.Global().Get("Array").New(length)
    for i, v := range arr {
        result.SetIndex(i, v)
    }
    
    return result
}

// domManipulation แก้ไข DOM จาก Go
func domManipulation() {
    document := js.Global().Get("document")
    
    // สร้าง element
    div := document.Call("createElement", "div")
    div.Set("id", "go-output")
    div.Set("innerHTML", "<h2>Hello from Go WASM!</h2>")
    
    // เพิ่ม style
    div.Get("style").Set("color", "blue")
    div.Get("style").Set("fontFamily", "monospace")
    
    // Append ไปที่ body
    body := document.Get("body")
    body.Call("appendChild", div)
    
    fmt.Println("DOM manipulation complete")
}

// fetchData เรียก fetch API จาก Go
func fetchData(url string) {
    // Promise-based async
    fetchPromise := js.Global().Call("fetch", url)
    
    fetchPromise.Call("then", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        response := args[0]
        return response.Call("json")
    })).Call("then", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        data := args[0]
        fmt.Printf("Fetched data: %v\n", data)
        return nil
    }))
}

// eventListener ตั้ง event listener
func setupEventListeners() {
    document := js.Global().Get("document")
    
    // Click handler
    clickHandler := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        event := args[0]
        target := event.Get("target")
        fmt.Printf("Clicked: %s\n", target.Get("id").String())
        return nil
    })
    
    document.Call("addEventListener", "click", clickHandler)
}

func main() {
    fmt.Println("Go WASM initialized")
    
    // Register functions
    registerFunctions()
    
    // DOM manipulation
    domManipulation()
    
    // Setup events
    setupEventListeners()
    
    // Keep WASM alive
    select {}
}
```

---

## 3. HTML สำหรับรัน WASM

```html
<!-- index.html - สำหรับใช้กับ Go WASM -->
<!DOCTYPE html>
<html>
<head>
    <title>Go WASM Demo</title>
</head>
<body>
    <h1>Go WebAssembly Demo</h1>
    
    <div>
        <h2>Add Numbers</h2>
        <input id="num1" type="number" value="10">
        <input id="num2" type="number" value="20">
        <button onclick="performAdd()">Add</button>
        <span id="result"></span>
    </div>
    
    <div>
        <h2>Fibonacci</h2>
        <input id="fibInput" type="number" value="10">
        <button onclick="calcFib()">Calculate</button>
        <span id="fibResult"></span>
    </div>
    
    <!-- Load Go WASM support -->
    <script src="wasm_exec.js"></script>
    <script>
        // Initialize Go WASM
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject).then(result => {
            go.run(result.instance);
        });
        
        function performAdd() {
            const a = parseFloat(document.getElementById('num1').value);
            const b = parseFloat(document.getElementById('num2').value);
            const result = goAdd(a, b);  // เรียก Go function
            document.getElementById('result').textContent = ` = ${result}`;
        }
        
        function calcFib() {
            const n = parseInt(document.getElementById('fibInput').value);
            const result = goFibonacci(n);  // เรียก Go function
            document.getElementById('fibResult').textContent = ` = ${result}`;
        }
    </script>
</body>
</html>
```

---

## 4. WASI - Server-side WASM

```go
//go:build wasip1

// wasi_example.go - WASI target
package main

import (
    "bufio"
    "fmt"
    "os"
)

/*
WASI (WebAssembly System Interface):
- Standard API สำหรับ WASM นอก browser
- System calls: file, network, clock, random
- Sandbox: ต้องได้รับ permission ก่อน access resources

WASI Runtimes:
- wasmtime
- wasmer
- wazero (pure Go)
- Node.js (experimental)

Compile:
  GOOS=wasip1 GOARCH=wasm go build -o app.wasm .

Run:
  wasmtime app.wasm
  wasmer app.wasm
*/

func main() {
    fmt.Println("Hello from WASI!")
    fmt.Printf("Running on WASI\n")
    
    // Read stdin
    scanner := bufio.NewScanner(os.Stdin)
    fmt.Print("Enter your name: ")
    if scanner.Scan() {
        name := scanner.Text()
        fmt.Printf("Hello, %s!\n", name)
    }
    
    // Write file
    err := os.WriteFile("/tmp/hello.txt", []byte("Hello from WASI\n"), 0644)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    
    fmt.Println("File written successfully")
    
    // Read file back
    data, err := os.ReadFile("/tmp/hello.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    
    fmt.Printf("File content: %s", data)
}
```

---

## 5. TinyGo สำหรับ WASM

```go
// tinygo_wasm.go - TinyGo สำหรับขนาดเล็ก
//go:build tinygo

package main

import (
    "fmt"
    "syscall/js"
)

/*
TinyGo:
- Go compiler ที่ออกแบบสำหรับ embedded/small environments
- ใช้ LLVM แทน gc compiler
- ขนาดเล็กกว่ามาก (100-500KB แทน 2-5MB)
- รองรับ: microcontrollers, WASM, embedded Linux

Install:
  brew install tinygo
  apt-get install tinygo

Compile to WASM:
  tinygo build -o app.wasm -target wasm ./
  tinygo build -o app.wasm -target wasi ./

Limitations:
- ไม่รองรับทุก features ของ Go standard library
- reflect package support จำกัด
- CGO ไม่รองรับ
*/

//export add
func add(a, b int) int {
    return a + b
}

//export multiply
func multiply(a, b int) int {
    return a * b
}

//export fibonacci
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    // Register ใน JS global
    js.Global().Set("tinyAdd", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        a := args[0].Int()
        b := args[1].Int()
        return add(a, b)
    }))
    
    js.Global().Set("tinyFib", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        n := args[0].Int()
        return fibonacci(n)
    }))
    
    fmt.Println("TinyGo WASM loaded")
    select {}
}
```

---

## 6. Wazero - Pure Go WASM Runtime

```go
// wazero_host.go - รัน WASM ใน Go ด้วย wazero
package main

import (
    "context"
    "fmt"
    "os"
)

/*
Wazero คือ pure Go WASM runtime
ไม่ต้องมี CGO หรือ external dependencies

Install:
  go get github.com/tetratelabs/wazero

ตัวอย่างนี้แสดง concept โดยไม่ import wazero จริง
เพราะต้องมี .wasm file จริงๆ
*/

// WASMRunner interface สำหรับ WASM execution
type WASMRunner interface {
    Run(ctx context.Context, wasmBytes []byte, funcName string, args ...uint64) ([]uint64, error)
    Close() error
}

// MockWASMRunner จำลอง WASM runner
type MockWASMRunner struct {
    functions map[string]func(...uint64) []uint64
}

func NewMockWASMRunner() *MockWASMRunner {
    r := &MockWASMRunner{
        functions: make(map[string]func(...uint64) []uint64),
    }
    
    // Register mock functions
    r.functions["add"] = func(args ...uint64) []uint64 {
        if len(args) < 2 {
            return []uint64{0}
        }
        return []uint64{args[0] + args[1]}
    }
    
    r.functions["fibonacci"] = func(args ...uint64) []uint64 {
        if len(args) == 0 {
            return []uint64{0}
        }
        n := args[0]
        return []uint64{fibUint64(n)}
    }
    
    return r
}

func fibUint64(n uint64) uint64 {
    if n <= 1 {
        return n
    }
    return fibUint64(n-1) + fibUint64(n-2)
}

func (r *MockWASMRunner) Run(ctx context.Context, funcName string, args ...uint64) ([]uint64, error) {
    fn, ok := r.functions[funcName]
    if !ok {
        return nil, fmt.Errorf("function %q not found", funcName)
    }
    
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    return fn(args...), nil
}

func (r *MockWASMRunner) Close() error {
    return nil
}

// ในโค้ดจริง จะใช้ wazero:
/*
import (
    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

func realWazeroExample() {
    ctx := context.Background()
    
    // Create runtime
    r := wazero.NewRuntime(ctx)
    defer r.Close(ctx)
    
    // WASI support
    wasi_snapshot_preview1.MustInstantiate(ctx, r)
    
    // Load WASM
    wasmBytes, _ := os.ReadFile("app.wasm")
    mod, _ := r.Instantiate(ctx, wasmBytes)
    defer mod.Close(ctx)
    
    // Call function
    add := mod.ExportedFunction("add")
    results, _ := add.Call(ctx, 1, 2)
    fmt.Printf("add(1, 2) = %d\n", results[0])
}
*/

func main() {
    fmt.Println("=== Wazero WASM Runtime Demo ===\n")
    
    runner := NewMockWASMRunner()
    defer runner.Close()
    
    ctx := context.Background()
    
    // Test add
    results, err := runner.Run(ctx, "add", 10, 20)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("add(10, 20) = %d\n", results[0])
    
    // Test fibonacci
    for _, n := range []uint64{0, 1, 5, 10, 15} {
        results, _ = runner.Run(ctx, "fibonacci", n)
        fmt.Printf("fibonacci(%d) = %d\n", n, results[0])
    }
    
    fmt.Println("\nReal wazero usage:")
    fmt.Println("  go get github.com/tetratelabs/wazero")
    fmt.Println("  See: https://github.com/tetratelabs/wazero/tree/main/examples")
}
```

---

## สรุป

บทนี้ครอบคลุม WebAssembly กับ Go:

1. **WASM Basics** - format, runtime model, use cases
2. **syscall/js** - Go ↔ JavaScript interop ใน browser
3. **WASI** - server-side WASM สำหรับ edge/cloud
4. **TinyGo** - compile ขนาดเล็กสำหรับ WASM และ embedded
5. **Wazero** - pure Go WASM runtime สำหรับ hosting

### Key Takeaways

- Standard Go WASM ขนาดใหญ่ (~2-5MB) แต่ TinyGo เล็กกว่ามาก
- `syscall/js` ช่วย interact กับ browser DOM และ APIs
- WASI ทำให้ WASM รันนอก browser ได้อย่างปลอดภัย
- Wazero ให้ embed WASM runtime ใน Go apps ได้
- ใช้ compression (gzip/brotli) เพื่อลดขนาด WASM downloads
