# Part 89: Plugin System ใน Go

## เป้าหมายของบทเรียน
- Go plugin package สำหรับ dynamic loading
- RPC-based plugin architecture
- hashicorp/go-plugin framework
- Hot reloading plugins
- Plugin isolation และ security

---

## 1. Go Plugin Package

```go
// host/main.go - ตัวอย่าง plugin host
package main

import (
    "fmt"
    "plugin"
)

/*
Go plugin package:
- รองรับเฉพาะ Linux, macOS (ไม่รองรับ Windows)
- Compile: go build -buildmode=plugin -o myplugin.so ./plugin/
- Load ด้วย plugin.Open()

ข้อจำกัด:
- ต้อง compile ด้วย Go version เดียวกัน
- ต้องมี import paths ตรงกัน
- ไม่สามารถ unload plugin ได้
- Race conditions ถ้าไม่ระวัง
*/

// GreetFunc function signature ที่ plugin ต้อง implement
type GreetFunc func(name string) string

// MathPlugin interface สำหรับ math operations
type MathPlugin interface {
    Add(a, b float64) float64
    Multiply(a, b float64) float64
    Name() string
}

func loadPlugin(path string) {
    fmt.Printf("Loading plugin: %s\n", path)
    
    p, err := plugin.Open(path)
    if err != nil {
        fmt.Printf("Error opening plugin: %v\n", err)
        return
    }
    
    // Lookup symbol
    greetSym, err := p.Lookup("Greet")
    if err != nil {
        fmt.Printf("Error finding Greet: %v\n", err)
        return
    }
    
    // Type assert
    greet, ok := greetSym.(func(string) string)
    if !ok {
        fmt.Printf("Greet has wrong type\n")
        return
    }
    
    result := greet("สมชาย")
    fmt.Printf("Plugin result: %s\n", result)
}

func main() {
    fmt.Println("=== Go Plugin Demo ===")
    fmt.Println("Note: Go plugins require Linux/macOS and same Go version")
    fmt.Println()
    
    // ในโค้ดจริง:
    // loadPlugin("./plugins/hello.so")
    
    fmt.Println("Plugin loading concepts:")
    fmt.Println("  1. Compile plugin: go build -buildmode=plugin -o plugin.so ./")
    fmt.Println("  2. Load: plugin.Open(\"plugin.so\")")
    fmt.Println("  3. Lookup symbol: p.Lookup(\"FunctionName\")")
    fmt.Println("  4. Type assert and call")
}
```

---

## 2. Plugin Interface Design

```go
// plugin_interface.go - ออกแบบ plugin interface
package main

import (
    "fmt"
)

// Plugin interface หลัก
type Plugin interface {
    Name() string
    Version() string
    Initialize(config map[string]string) error
    Execute(input interface{}) (interface{}, error)
    Shutdown() error
}

// PluginMetadata ข้อมูล plugin
type PluginMetadata struct {
    Name        string
    Version     string
    Description string
    Author      string
    Tags        []string
}

// PluginRegistry เก็บ plugins ที่ register แล้ว
type PluginRegistry struct {
    plugins map[string]Plugin
    meta    map[string]PluginMetadata
}

// NewPluginRegistry สร้าง registry
func NewPluginRegistry() *PluginRegistry {
    return &PluginRegistry{
        plugins: make(map[string]Plugin),
        meta:    make(map[string]PluginMetadata),
    }
}

// Register ลงทะเบียน plugin
func (r *PluginRegistry) Register(meta PluginMetadata, p Plugin) error {
    if _, exists := r.plugins[meta.Name]; exists {
        return fmt.Errorf("plugin %q already registered", meta.Name)
    }
    
    r.plugins[meta.Name] = p
    r.meta[meta.Name] = meta
    return nil
}

// Get ดึง plugin ตามชื่อ
func (r *PluginRegistry) Get(name string) (Plugin, bool) {
    p, ok := r.plugins[name]
    return p, ok
}

// List แสดง plugins ทั้งหมด
func (r *PluginRegistry) List() []PluginMetadata {
    result := make([]PluginMetadata, 0, len(r.meta))
    for _, m := range r.meta {
        result = append(result, m)
    }
    return result
}

// Unregister ลบ plugin
func (r *PluginRegistry) Unregister(name string) error {
    p, ok := r.plugins[name]
    if !ok {
        return fmt.Errorf("plugin %q not found", name)
    }
    
    if err := p.Shutdown(); err != nil {
        return fmt.Errorf("shutdown error: %w", err)
    }
    
    delete(r.plugins, name)
    delete(r.meta, name)
    return nil
}

// GreetPlugin ตัวอย่าง plugin implementation
type GreetPlugin struct {
    language string
    count    int
}

func (g *GreetPlugin) Name() string    { return "greet" }
func (g *GreetPlugin) Version() string { return "1.0.0" }

func (g *GreetPlugin) Initialize(config map[string]string) error {
    if lang, ok := config["language"]; ok {
        g.language = lang
    } else {
        g.language = "en"
    }
    fmt.Printf("GreetPlugin initialized (language=%s)\n", g.language)
    return nil
}

func (g *GreetPlugin) Execute(input interface{}) (interface{}, error) {
    name, ok := input.(string)
    if !ok {
        return nil, fmt.Errorf("input must be string")
    }
    
    g.count++
    
    switch g.language {
    case "th":
        return fmt.Sprintf("สวัสดีครับ, %s! (call #%d)", name, g.count), nil
    case "ja":
        return fmt.Sprintf("こんにちは, %s! (call #%d)", name, g.count), nil
    default:
        return fmt.Sprintf("Hello, %s! (call #%d)", name, g.count), nil
    }
}

func (g *GreetPlugin) Shutdown() error {
    fmt.Printf("GreetPlugin shutdown (total calls: %d)\n", g.count)
    return nil
}

// MathPlugin ตัวอย่าง plugin อีกตัว
type MathPluginImpl struct {
    operations int
}

func (m *MathPluginImpl) Name() string    { return "math" }
func (m *MathPluginImpl) Version() string { return "2.0.0" }

func (m *MathPluginImpl) Initialize(config map[string]string) error {
    fmt.Println("MathPlugin initialized")
    return nil
}

func (m *MathPluginImpl) Execute(input interface{}) (interface{}, error) {
    op, ok := input.(map[string]interface{})
    if !ok {
        return nil, fmt.Errorf("input must be map[string]interface{}")
    }
    
    m.operations++
    
    operation, _ := op["op"].(string)
    a, _ := op["a"].(float64)
    b, _ := op["b"].(float64)
    
    switch operation {
    case "add":
        return a + b, nil
    case "subtract":
        return a - b, nil
    case "multiply":
        return a * b, nil
    case "divide":
        if b == 0 {
            return nil, fmt.Errorf("division by zero")
        }
        return a / b, nil
    default:
        return nil, fmt.Errorf("unknown operation: %s", operation)
    }
}

func (m *MathPluginImpl) Shutdown() error {
    fmt.Printf("MathPlugin shutdown (total operations: %d)\n", m.operations)
    return nil
}

func main() {
    fmt.Println("=== Plugin Registry Demo ===\n")
    
    registry := NewPluginRegistry()
    
    // Register plugins
    greetPlugin := &GreetPlugin{}
    registry.Register(PluginMetadata{
        Name:        "greet",
        Version:     "1.0.0",
        Description: "Greeting plugin",
        Tags:        []string{"utility", "greeting"},
    }, greetPlugin)
    
    mathPlugin := &MathPluginImpl{}
    registry.Register(PluginMetadata{
        Name:        "math",
        Version:     "2.0.0",
        Description: "Math operations plugin",
        Tags:        []string{"math", "compute"},
    }, mathPlugin)
    
    // List plugins
    fmt.Println("Registered plugins:")
    for _, meta := range registry.List() {
        fmt.Printf("  - %s v%s: %s\n", meta.Name, meta.Version, meta.Description)
    }
    
    // Use greet plugin
    p, _ := registry.Get("greet")
    p.Initialize(map[string]string{"language": "th"})
    
    result, _ := p.Execute("สมชาย")
    fmt.Printf("\nGreet result: %v\n", result)
    
    result, _ = p.Execute("World")
    fmt.Printf("Greet result: %v\n", result)
    
    // Use math plugin
    mp, _ := registry.Get("math")
    mp.Initialize(nil)
    
    mathResult, _ := mp.Execute(map[string]interface{}{
        "op": "add", "a": 10.0, "b": 20.0,
    })
    fmt.Printf("\nMath result: %v\n", mathResult)
    
    // Unregister
    registry.Unregister("greet")
    registry.Unregister("math")
}
```

---

## 3. RPC-based Plugin System (hashicorp/go-plugin style)

```go
// rpc_plugin.go - RPC-based plugins สำหรับ isolation
package main

import (
    "encoding/gob"
    "fmt"
    "net"
    "net/rpc"
    "os"
    "os/exec"
    "sync"
    "time"
)

/*
hashicorp/go-plugin approach:
- Plugin รันเป็น subprocess แยกต่างหาก
- Communication ผ่าน RPC (gRPC หรือ net/rpc)
- Isolation: crash plugin ไม่ crash host
- Versioning ผ่าน protocol negotiation

Architecture:
  Host Process         Plugin Process
  [Plugin Client] <--> [Plugin Server]
  (net/rpc or gRPC)
*/

// PluginRequest สำหรับ RPC
type PluginRequest struct {
    Method string
    Args   map[string]interface{}
}

// PluginResponse สำหรับ RPC
type PluginResponse struct {
    Result interface{}
    Error  string
}

// PluginServer RPC server ที่รันใน plugin process
type PluginServer struct {
    mu      sync.Mutex
    callCnt int
}

// Execute RPC method
func (s *PluginServer) Execute(req *PluginRequest, resp *PluginResponse) error {
    s.mu.Lock()
    s.callCnt++
    callNum := s.callCnt
    s.mu.Unlock()
    
    fmt.Printf("[Plugin] Execute called #%d: method=%s\n", callNum, req.Method)
    
    switch req.Method {
    case "greet":
        name, _ := req.Args["name"].(string)
        resp.Result = fmt.Sprintf("Hello from plugin, %s!", name)
    case "add":
        a, _ := req.Args["a"].(float64)
        b, _ := req.Args["b"].(float64)
        resp.Result = a + b
    default:
        resp.Error = fmt.Sprintf("unknown method: %s", req.Method)
    }
    
    return nil
}

// PluginClient เชื่อมต่อกับ plugin
type PluginClient struct {
    client *rpc.Client
    conn   net.Conn
}

// NewPluginClient สร้าง client
func NewPluginClient(addr string) (*PluginClient, error) {
    conn, err := net.DialTimeout("tcp", addr, 5*time.Second)
    if err != nil {
        return nil, fmt.Errorf("connect error: %w", err)
    }
    
    return &PluginClient{
        client: rpc.NewClient(conn),
        conn:   conn,
    }, nil
}

// Call เรียก method ใน plugin
func (c *PluginClient) Call(method string, args map[string]interface{}) (interface{}, error) {
    req := &PluginRequest{
        Method: method,
        Args:   args,
    }
    
    resp := &PluginResponse{}
    
    if err := c.client.Call("PluginServer.Execute", req, resp); err != nil {
        return nil, err
    }
    
    if resp.Error != "" {
        return nil, fmt.Errorf(resp.Error)
    }
    
    return resp.Result, nil
}

// Close ปิด connection
func (c *PluginClient) Close() {
    c.client.Close()
    c.conn.Close()
}

// startPluginServer เริ่ม RPC server (รันใน plugin process)
func startPluginServer(addr string) error {
    gob.Register(map[string]interface{}{})
    
    server := rpc.NewServer()
    if err := server.Register(&PluginServer{}); err != nil {
        return err
    }
    
    listener, err := net.Listen("tcp", addr)
    if err != nil {
        return fmt.Errorf("listen error: %w", err)
    }
    
    fmt.Printf("[Plugin Server] Listening on %s\n", addr)
    
    for {
        conn, err := listener.Accept()
        if err != nil {
            return err
        }
        go server.ServeConn(conn)
    }
}

// PluginProcess จำลอง plugin process
func runPluginProcess(addr string) {
    if err := startPluginServer(addr); err != nil {
        fmt.Fprintf(os.Stderr, "Plugin error: %v\n", err)
        os.Exit(1)
    }
}

// PluginManager จัดการ plugin processes
type PluginManager struct {
    plugins map[string]*PluginClient
    procs   map[string]*exec.Cmd
    mu      sync.Mutex
}

func NewPluginManager() *PluginManager {
    return &PluginManager{
        plugins: make(map[string]*PluginClient),
        procs:   make(map[string]*exec.Cmd),
    }
}

func (pm *PluginManager) LoadPlugin(name, addr string) error {
    pm.mu.Lock()
    defer pm.mu.Unlock()
    
    client, err := NewPluginClient(addr)
    if err != nil {
        return fmt.Errorf("load plugin %s: %w", name, err)
    }
    
    pm.plugins[name] = client
    fmt.Printf("Plugin %q loaded from %s\n", name, addr)
    return nil
}

func (pm *PluginManager) Call(pluginName, method string, args map[string]interface{}) (interface{}, error) {
    pm.mu.Lock()
    client, ok := pm.plugins[pluginName]
    pm.mu.Unlock()
    
    if !ok {
        return nil, fmt.Errorf("plugin %q not found", pluginName)
    }
    
    return client.Call(method, args)
}

func (pm *PluginManager) UnloadPlugin(name string) {
    pm.mu.Lock()
    defer pm.mu.Unlock()
    
    if client, ok := pm.plugins[name]; ok {
        client.Close()
        delete(pm.plugins, name)
    }
    
    if cmd, ok := pm.procs[name]; ok {
        cmd.Process.Kill()
        delete(pm.procs, name)
    }
}

func main() {
    fmt.Println("=== RPC Plugin System Demo ===\n")
    
    addr := "127.0.0.1:19999"
    
    // Start plugin server in goroutine (จำลอง subprocess)
    go func() {
        if err := startPluginServer(addr); err != nil {
            fmt.Printf("Server error: %v\n", err)
        }
    }()
    
    time.Sleep(100 * time.Millisecond)
    
    // Connect client
    manager := NewPluginManager()
    if err := manager.LoadPlugin("demo", addr); err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer manager.UnloadPlugin("demo")
    
    // Call methods
    result, err := manager.Call("demo", "greet", map[string]interface{}{
        "name": "สมชาย",
    })
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("greet result: %v\n", result)
    }
    
    result, err = manager.Call("demo", "add", map[string]interface{}{
        "a": 15.0, "b": 27.0,
    })
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("add result: %v\n", result)
    }
    
    fmt.Println("\nFor production: use github.com/hashicorp/go-plugin")
    fmt.Println("  - gRPC-based communication")
    fmt.Println("  - Protocol versioning")
    fmt.Println("  - Auto-reconnect")
    fmt.Println("  - TLS support")
}
```

---

## สรุป

บทนี้ครอบคลุม Plugin System ใน Go:

1. **Go plugin package** - native dynamic loading (Linux/macOS)
2. **Plugin Interface Design** - ออกแบบ interface ที่ดีสำหรับ extensibility
3. **RPC-based Plugins** - isolation ผ่าน subprocess + RPC
4. **hashicorp/go-plugin** - production-ready plugin framework

### Key Takeaways

- Go plugin package มีข้อจำกัดด้าน OS และ version compatibility
- RPC-based plugins ให้ isolation ที่ดีกว่า แต่ overhead สูงกว่า
- hashicorp/go-plugin เป็น standard สำหรับ production plugin systems
- Interface-based design ทำให้ swap implementations ได้ง่าย
- ออกแบบ plugin protocol ให้ backward compatible เสมอ
