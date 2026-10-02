# Part 22: Embedding ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Struct Embedding เพื่อนำ fields และ methods มาใช้ซ้ำ
- ใช้ Interface Embedding
- เข้าใจ Method Promotion
- Override promoted methods
- เข้าใจความแตกต่างระหว่าง Embedding กับ Inheritance
- ใช้ Multiple Embedding
- นำ Embedding ไปใช้ใน practical patterns

---

## 22.1 Struct Embedding

### 22.1.1 Embedding พื้นฐาน

```go
package main

import "fmt"

// Base struct
type Animal struct {
    Name string
    Age  int
}

func (a Animal) Speak() string {
    return fmt.Sprintf("I am %s", a.Name)
}

func (a Animal) Info() string {
    return fmt.Sprintf("%s (age: %d)", a.Name, a.Age)
}

// Dog embeds Animal (ไม่ใช่ inheritance - เป็น composition)
type Dog struct {
    Animal  // embedded type (ไม่มีชื่อ field)
    Breed string
}

func (d Dog) Speak() string {
    // Override Speak
    return fmt.Sprintf("Woof! I am %s the %s", d.Name, d.Breed)
}

// Cat embeds Animal
type Cat struct {
    Animal
    Indoor bool
}

func main() {
    dog := Dog{
        Animal: Animal{Name: "Rex", Age: 3},
        Breed:  "Labrador",
    }
    
    // เข้าถึง fields ของ Animal โดยตรง (promoted)
    fmt.Println("Name:", dog.Name)        // promoted field
    fmt.Println("Age:", dog.Age)          // promoted field
    fmt.Println("Breed:", dog.Breed)      // own field
    
    // หรือเข้าถึงผ่าน embedded type
    fmt.Println("Via Animal:", dog.Animal.Name)
    
    // Methods
    fmt.Println(dog.Speak())      // Dog's override
    fmt.Println(dog.Info())       // promoted from Animal
    
    cat := Cat{
        Animal: Animal{Name: "Whiskers", Age: 5},
        Indoor: true,
    }
    
    fmt.Println(cat.Speak())  // Animal's Speak (no override)
    fmt.Println(cat.Indoor)
    
    // Interface satisfaction
    type Speaker interface {
        Speak() string
    }
    
    var s Speaker = dog
    fmt.Println("Interface:", s.Speak())
}
```

### 22.1.2 Multiple Embedding

```go
package main

import (
    "fmt"
    "time"
)

// Multiple embedded structs
type Timestamps struct {
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (t *Timestamps) Touch() {
    t.UpdatedAt = time.Now()
}

func (t Timestamps) Age() time.Duration {
    return time.Since(t.CreatedAt)
}

type SoftDelete struct {
    DeletedAt *time.Time
    IsDeleted bool
}

func (s *SoftDelete) Delete() {
    now := time.Now()
    s.DeletedAt = &now
    s.IsDeleted = true
}

func (s SoftDelete) Restore() {
    s.DeletedAt = nil
    s.IsDeleted = false
}

type BaseModel struct {
    ID uint64
    Timestamps
    SoftDelete
}

// User model
type User struct {
    BaseModel
    Username string
    Email    string
    Password string
}

// Product model
type Product struct {
    BaseModel
    Name  string
    Price float64
    Stock int
}

func main() {
    now := time.Now()
    
    user := User{
        BaseModel: BaseModel{
            ID: 1,
            Timestamps: Timestamps{
                CreatedAt: now,
                UpdatedAt: now,
            },
        },
        Username: "alice",
        Email:    "alice@example.com",
        Password: "hashed_password",
    }
    
    fmt.Printf("User: %+v\n", user.Username)
    fmt.Printf("ID: %d\n", user.ID)
    fmt.Printf("Created: %v\n", user.CreatedAt.Format("2006-01-02"))
    fmt.Printf("Deleted: %v\n", user.IsDeleted)
    
    // Touch (update UpdatedAt)
    user.Touch()
    fmt.Printf("Updated: %v\n", user.UpdatedAt.Format("15:04:05"))
    
    // Soft delete
    user.Delete()
    fmt.Printf("Is deleted: %v\n", user.IsDeleted)
    fmt.Printf("Deleted at: %v\n", user.DeletedAt.Format("15:04:05"))
    
    // Product
    product := Product{
        BaseModel: BaseModel{
            ID: 101,
            Timestamps: Timestamps{
                CreatedAt: now,
                UpdatedAt: now,
            },
        },
        Name:  "Go Book",
        Price: 350.0,
        Stock: 100,
    }
    
    fmt.Printf("\nProduct: %s (฿%.2f)\n", product.Name, product.Price)
    fmt.Printf("Age: %.0f seconds\n", product.Age().Seconds())
}
```

### 22.1.3 Field Conflicts

```go
package main

import "fmt"

type A struct {
    X int
    Y int
}

func (a A) GetX() int { return a.X }
func (a A) Name() string { return "A" }

type B struct {
    X int
    Z int
}

func (b B) GetX() int { return b.X }
func (b B) Name() string { return "B" }

// Ambiguous embedding
type C struct {
    A
    B
    X int // C's own X (ซ่อน A.X และ B.X)
}

func (c C) Name() string { return "C" } // Override ทั้ง A.Name และ B.Name

func main() {
    c := C{
        A: A{X: 10, Y: 20},
        B: B{X: 30, Z: 40},
        X: 50, // C's own X
    }
    
    // C.X → ของ C เอง
    fmt.Println("C.X:", c.X)
    
    // ต้องระบุ path ถ้า ambiguous
    fmt.Println("C.A.X:", c.A.X)
    fmt.Println("C.B.X:", c.B.X)
    fmt.Println("C.Y:", c.Y)  // promote จาก A (ไม่ ambiguous)
    fmt.Println("C.Z:", c.Z)  // promote จาก B (ไม่ ambiguous)
    
    // Method ambiguity
    // c.GetX() → compile error: ambiguous selector
    // ต้องระบุ:
    fmt.Println("A.GetX:", c.A.GetX())
    fmt.Println("B.GetX:", c.B.GetX())
    
    // C.Name() → ของ C เอง (ไม่ ambiguous)
    fmt.Println("Name:", c.Name())
}
```

---

## 22.2 Interface Embedding

### 22.2.1 Interface Embedding พื้นฐาน

```go
package main

import (
    "fmt"
    "io"
)

// Small interfaces
type Reader interface {
    Read() string
}

type Writer interface {
    Write(s string)
}

type Closer interface {
    Close() error
}

// Composed interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    ReadWriter
    Closer
}

// Standard library examples:
// io.Reader, io.Writer, io.Closer
// io.ReadWriter = io.Reader + io.Writer
// io.ReadWriteCloser = io.ReadWriter + io.Closer

// Implementation
type StringBuffer struct {
    data   []byte
    closed bool
}

func (b *StringBuffer) Read() string {
    if b.closed {
        return ""
    }
    return string(b.data)
}

func (b *StringBuffer) Write(s string) {
    if !b.closed {
        b.data = append(b.data, s...)
    }
}

func (b *StringBuffer) Close() error {
    b.closed = true
    fmt.Println("Buffer closed")
    return nil
}

func useReader(r Reader) {
    fmt.Println("Read:", r.Read())
}

func useWriter(w Writer) {
    w.Write("hello ")
    w.Write("world")
}

func useReadWriteCloser(rwc ReadWriteCloser) {
    rwc.Write("test data")
    fmt.Println("Read:", rwc.Read())
    rwc.Close()
}

func main() {
    buf := &StringBuffer{}
    
    // StringBuffer implements ReadWriteCloser
    useReadWriteCloser(buf)
    
    // สร้างใหม่
    buf2 := &StringBuffer{}
    var rw ReadWriter = buf2
    useWriter(rw)
    useReader(rw)
    
    // io package
    var r io.Reader = strings.NewReader("hello")
    var w io.Writer = os.Stdout
    io.Copy(w, r)
    fmt.Println()
}
```

### 22.2.2 Interface Composition Patterns

```go
package main

import (
    "fmt"
    "time"
)

// CRUD interface decomposition
type Creator[T any] interface {
    Create(item T) error
}

type Finder[T any] interface {
    FindByID(id int) (T, error)
    FindAll() ([]T, error)
}

type Updater[T any] interface {
    Update(id int, item T) error
}

type Deleter interface {
    Delete(id int) error
}

type Repository[T any] interface {
    Creator[T]
    Finder[T]
    Updater[T]
    Deleter
}

// Implementation
type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

type UserRepository struct {
    users  map[int]*User
    nextID int
}

func NewUserRepository() *UserRepository {
    return &UserRepository{
        users:  make(map[int]*User),
        nextID: 1,
    }
}

func (r *UserRepository) Create(user User) error {
    user.ID = r.nextID
    user.CreatedAt = time.Now()
    r.users[r.nextID] = &user
    r.nextID++
    fmt.Printf("Created user: %+v\n", user)
    return nil
}

func (r *UserRepository) FindByID(id int) (User, error) {
    user, ok := r.users[id]
    if !ok {
        return User{}, fmt.Errorf("user %d not found", id)
    }
    return *user, nil
}

func (r *UserRepository) FindAll() ([]User, error) {
    users := make([]User, 0, len(r.users))
    for _, u := range r.users {
        users = append(users, *u)
    }
    return users, nil
}

func (r *UserRepository) Update(id int, user User) error {
    if _, ok := r.users[id]; !ok {
        return fmt.Errorf("user %d not found", id)
    }
    user.ID = id
    r.users[id] = &user
    fmt.Printf("Updated user %d\n", id)
    return nil
}

func (r *UserRepository) Delete(id int) error {
    if _, ok := r.users[id]; !ok {
        return fmt.Errorf("user %d not found", id)
    }
    delete(r.users, id)
    fmt.Printf("Deleted user %d\n", id)
    return nil
}

func main() {
    repo := NewUserRepository()
    
    // Create
    repo.Create(User{Name: "Alice", Email: "alice@example.com"})
    repo.Create(User{Name: "Bob", Email: "bob@example.com"})
    repo.Create(User{Name: "Charlie", Email: "charlie@example.com"})
    
    // Find
    user, err := repo.FindByID(2)
    if err != nil {
        fmt.Printf("error: %v\n", err)
    } else {
        fmt.Printf("Found: %+v\n", user)
    }
    
    // FindAll
    all, _ := repo.FindAll()
    fmt.Printf("All users: %d\n", len(all))
    
    // Update
    repo.Update(1, User{Name: "Alice Updated", Email: "alice2@example.com"})
    
    // Delete
    repo.Delete(3)
    
    all2, _ := repo.FindAll()
    fmt.Printf("Remaining users: %d\n", len(all2))
}
```

---

## 22.3 Method Promotion

### 22.3.1 Method Promotion Rules

```go
package main

import "fmt"

type Logger struct {
    Prefix string
}

func (l Logger) Log(msg string) {
    fmt.Printf("[%s] %s\n", l.Prefix, msg)
}

func (l Logger) Warn(msg string) {
    fmt.Printf("[%s] WARN: %s\n", l.Prefix, msg)
}

func (l Logger) Error(msg string) {
    fmt.Printf("[%s] ERROR: %s\n", l.Prefix, msg)
}

type Metrics struct {
    counts map[string]int
}

func NewMetrics() Metrics {
    return Metrics{counts: make(map[string]int)}
}

func (m *Metrics) Increment(key string) {
    m.counts[key]++
}

func (m Metrics) Get(key string) int {
    return m.counts[key]
}

// Service embeds Logger and Metrics
type UserService struct {
    Logger
    Metrics
    db map[int]string
}

func NewUserService() *UserService {
    return &UserService{
        Logger:  Logger{Prefix: "UserService"},
        Metrics: NewMetrics(),
        db:      make(map[int]string),
    }
}

func (s *UserService) CreateUser(id int, name string) {
    s.Log(fmt.Sprintf("Creating user: %d - %s", id, name))
    s.db[id] = name
    s.Increment("users_created")
}

func (s *UserService) GetUser(id int) (string, bool) {
    s.Increment("users_fetched")
    name, ok := s.db[id]
    if !ok {
        s.Warn(fmt.Sprintf("User %d not found", id))
    }
    return name, ok
}

func (s *UserService) Stats() {
    fmt.Printf("Created: %d, Fetched: %d\n",
        s.Get("users_created"),
        s.Get("users_fetched"))
}

func main() {
    svc := NewUserService()
    
    // Promoted methods
    svc.Log("Service started")
    svc.CreateUser(1, "Alice")
    svc.CreateUser(2, "Bob")
    
    if name, ok := svc.GetUser(1); ok {
        fmt.Println("Found:", name)
    }
    
    svc.GetUser(99) // triggers warning
    
    svc.Stats()
}
```

### 22.3.2 Pointer vs Value Embedding

```go
package main

import "fmt"

type Engine struct {
    Power int
    RPM   int
}

func (e *Engine) Start() {
    e.RPM = 1000
    fmt.Printf("Engine started: %d RPM\n", e.RPM)
}

func (e *Engine) Stop() {
    e.RPM = 0
    fmt.Println("Engine stopped")
}

func (e Engine) Status() string {
    if e.RPM > 0 {
        return fmt.Sprintf("Running at %d RPM", e.RPM)
    }
    return "Stopped"
}

// Embed ด้วย value
type CarValue struct {
    Engine // value embedding - ถูก copy เมื่อ copy CarValue
    Model string
}

// Embed ด้วย pointer
type CarPointer struct {
    *Engine // pointer embedding - share ข้อมูลเดียวกัน
    Model string
}

func main() {
    // Value embedding
    cv := CarValue{
        Engine: Engine{Power: 100},
        Model:  "Sedan",
    }
    cv.Start()
    fmt.Println("Car status:", cv.Status())
    
    // Copy
    cv2 := cv
    cv2.Start() // เปลี่ยน RPM ของ cv2 ไม่กระทบ cv
    cv.Stop()
    
    fmt.Printf("cv.RPM = %d\n", cv.RPM)  // 0 (stopped)
    fmt.Printf("cv2.RPM = %d\n", cv2.RPM) // ขึ้นอยู่กับ timing
    
    // Pointer embedding
    engine := &Engine{Power: 200}
    cp1 := CarPointer{Engine: engine, Model: "Sports"}
    cp2 := CarPointer{Engine: engine, Model: "Sports2"} // share engine!
    
    cp1.Start()
    fmt.Printf("cp1.RPM = %d\n", cp1.RPM) // 1000
    fmt.Printf("cp2.RPM = %d\n", cp2.RPM) // 1000 (shared!)
    
    cp2.Stop()
    fmt.Printf("cp1.RPM = %d\n", cp1.RPM) // 0 (shared!)
}
```

---

## 22.4 Embedding vs Inheritance

### 22.4.1 ความแตกต่างสำคัญ

```go
package main

import "fmt"

// Go ไม่มี inheritance - ใช้ composition แทน

// ปัญหาของ inheritance (ถ้า Go มี)
// class Base { void method() { ... } }
// class Child extends Base { void method() { base.method() } }
// ปัญหา: "fragile base class"

// Go approach: composition
type Base struct {
    data string
}

func (b *Base) Method() {
    fmt.Printf("Base.Method called with data: %s\n", b.data)
}

func (b *Base) Helper() string {
    return "helper: " + b.data
}

type Composed struct {
    Base // embed
    extra string
}

// Override - ไม่มี super/parent concept
func (c *Composed) Method() {
    // ถ้าต้องการเรียก Base.Method ต้องเรียกตรงๆ
    c.Base.Method()
    fmt.Printf("Composed.Method also called with extra: %s\n", c.extra)
}

// Delegation pattern
type Worker struct {
    name string
}

func (w *Worker) DoWork() string {
    return fmt.Sprintf("Worker %s doing work", w.name)
}

type Manager struct {
    Worker
    team []string
}

// Manager delegates to Worker
func (m *Manager) DoWork() string {
    // Override แต่ยัง delegate ไปที่ Worker
    result := m.Worker.DoWork()
    return result + fmt.Sprintf(" (managing %d people)", len(m.team))
}

// Interface-based polymorphism (แทน inheritance polymorphism)
type Processor interface {
    Process(data string) string
}

type UpperProcessor struct{}
type LowerProcessor struct{}
type TrimProcessor struct{}

func (p UpperProcessor) Process(data string) string {
    return strings.ToUpper(data)
}
func (p LowerProcessor) Process(data string) string {
    return strings.ToLower(data)
}
func (p TrimProcessor) Process(data string) string {
    return strings.TrimSpace(data)
}

type Pipeline struct {
    processors []Processor
}

func (p *Pipeline) Add(proc Processor) *Pipeline {
    p.processors = append(p.processors, proc)
    return p
}

func (p *Pipeline) Execute(data string) string {
    for _, proc := range p.processors {
        data = proc.Process(data)
    }
    return data
}

func main() {
    // Embedding vs inheritance
    c := &Composed{
        Base:  Base{data: "hello"},
        extra: "world",
    }
    
    c.Method() // calls Composed.Method (which calls Base.Method)
    fmt.Println(c.Helper()) // promoted from Base
    
    // Manager delegation
    m := &Manager{
        Worker: Worker{name: "Alice"},
        team:   []string{"Bob", "Charlie", "Dave"},
    }
    
    fmt.Println(m.DoWork())
    
    // Pipeline (composition over inheritance)
    pipeline := &Pipeline{}
    pipeline.Add(TrimProcessor{}).
        Add(LowerProcessor{})
    
    result := pipeline.Execute("  Hello World!  ")
    fmt.Println("Pipeline result:", result)
}
```

---

## 22.5 Practical Patterns

### 22.5.1 Mixin Pattern

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Observable mixin
type Observable struct {
    mu        sync.RWMutex
    listeners map[string][]func(interface{})
}

func (o *Observable) On(event string, fn func(interface{})) {
    o.mu.Lock()
    defer o.mu.Unlock()
    
    if o.listeners == nil {
        o.listeners = make(map[string][]func(interface{}))
    }
    o.listeners[event] = append(o.listeners[event], fn)
}

func (o *Observable) Emit(event string, data interface{}) {
    o.mu.RLock()
    defer o.mu.RUnlock()
    
    for _, fn := range o.listeners[event] {
        fn(data)
    }
}

// Cacheable mixin
type Cacheable struct {
    mu    sync.RWMutex
    cache map[string]cachedItem
}

type cachedItem struct {
    value   interface{}
    expires time.Time
}

func (c *Cacheable) SetCache(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if c.cache == nil {
        c.cache = make(map[string]cachedItem)
    }
    c.cache[key] = cachedItem{value: value, expires: time.Now().Add(ttl)}
}

func (c *Cacheable) GetCache(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    if c.cache == nil {
        return nil, false
    }
    
    item, ok := c.cache[key]
    if !ok || time.Now().After(item.expires) {
        return nil, false
    }
    return item.value, true
}

// Service ที่ใช้ mixins
type ProductService struct {
    Observable
    Cacheable
    products map[int]string
}

func NewProductService() *ProductService {
    svc := &ProductService{
        products: map[int]string{
            1: "iPhone",
            2: "MacBook",
            3: "iPad",
        },
    }
    return svc
}

func (s *ProductService) GetProduct(id int) (string, error) {
    // ดูใน cache ก่อน
    cacheKey := fmt.Sprintf("product:%d", id)
    if cached, ok := s.GetCache(cacheKey); ok {
        fmt.Printf("[CACHE HIT] product %d\n", id)
        return cached.(string), nil
    }
    
    // ดู database
    product, ok := s.products[id]
    if !ok {
        return "", fmt.Errorf("product %d not found", id)
    }
    
    // เก็บใน cache
    s.SetCache(cacheKey, product, 30*time.Second)
    
    // emit event
    s.Emit("product:fetched", map[string]interface{}{
        "id":   id,
        "name": product,
    })
    
    return product, nil
}

func (s *ProductService) UpdateProduct(id int, name string) {
    s.products[id] = name
    
    // Invalidate cache
    s.SetCache(fmt.Sprintf("product:%d", id), name, 30*time.Second)
    
    s.Emit("product:updated", map[string]interface{}{
        "id":   id,
        "name": name,
    })
}

func main() {
    svc := NewProductService()
    
    // ลงทะเบียน event listeners
    svc.On("product:fetched", func(data interface{}) {
        m := data.(map[string]interface{})
        fmt.Printf("[EVENT] fetched: id=%v, name=%v\n", m["id"], m["name"])
    })
    
    svc.On("product:updated", func(data interface{}) {
        m := data.(map[string]interface{})
        fmt.Printf("[EVENT] updated: id=%v, name=%v\n", m["id"], m["name"])
    })
    
    // ใช้งาน
    fmt.Println("=== Get Products ===")
    for _, id := range []int{1, 2, 99} {
        product, err := svc.GetProduct(id)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
        } else {
            fmt.Printf("Product %d: %s\n", id, product)
        }
    }
    
    // ดึงซ้ำ (จาก cache)
    fmt.Println("\n=== Get Cached ===")
    svc.GetProduct(1)
    svc.GetProduct(2)
    
    // Update
    fmt.Println("\n=== Update ===")
    svc.UpdateProduct(1, "iPhone 15 Pro")
    product, _ := svc.GetProduct(1)
    fmt.Println("Updated:", product)
}
```

---

## 22.6 Workshop: Plugin System

```go
package main

import (
    "fmt"
    "sort"
    "strings"
    "time"
)

// Plugin System ด้วย Embedding

// Base plugin interface
type Plugin interface {
    Name() string
    Version() string
    Init() error
    Execute(input string) (string, error)
    Cleanup() error
}

// Base plugin สำหรับ embed
type BasePlugin struct {
    name    string
    version string
    enabled bool
    stats   PluginStats
}

type PluginStats struct {
    Executions int
    Errors     int
    TotalTime  time.Duration
}

func NewBasePlugin(name, version string) BasePlugin {
    return BasePlugin{
        name:    name,
        version: version,
        enabled: true,
    }
}

func (b *BasePlugin) Name() string    { return b.name }
func (b *BasePlugin) Version() string { return b.version }
func (b *BasePlugin) IsEnabled() bool { return b.enabled }

func (b *BasePlugin) Init() error {
    fmt.Printf("[%s] Initializing...\n", b.name)
    return nil
}

func (b *BasePlugin) Cleanup() error {
    fmt.Printf("[%s] Cleaning up...\n", b.name)
    return nil
}

func (b *BasePlugin) recordExecution(start time.Time, err error) {
    b.stats.Executions++
    b.stats.TotalTime += time.Since(start)
    if err != nil {
        b.stats.Errors++
    }
}

func (b *BasePlugin) Stats() PluginStats {
    return b.stats
}

// Concrete plugins
type UpperCasePlugin struct {
    BasePlugin
}

func NewUpperCasePlugin() *UpperCasePlugin {
    return &UpperCasePlugin{
        BasePlugin: NewBasePlugin("uppercase", "1.0.0"),
    }
}

func (p *UpperCasePlugin) Execute(input string) (string, error) {
    start := time.Now()
    result := strings.ToUpper(input)
    p.recordExecution(start, nil)
    return result, nil
}

type ReversePlugin struct {
    BasePlugin
}

func NewReversePlugin() *ReversePlugin {
    return &ReversePlugin{
        BasePlugin: NewBasePlugin("reverse", "1.1.0"),
    }
}

func (p *ReversePlugin) Execute(input string) (string, error) {
    start := time.Now()
    runes := []rune(input)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    result := string(runes)
    p.recordExecution(start, nil)
    return result, nil
}

type EncryptPlugin struct {
    BasePlugin
    key byte
}

func NewEncryptPlugin(key byte) *EncryptPlugin {
    return &EncryptPlugin{
        BasePlugin: NewBasePlugin("encrypt", "2.0.0"),
        key:        key,
    }
}

func (p *EncryptPlugin) Execute(input string) (string, error) {
    start := time.Now()
    result := make([]byte, len(input))
    for i, c := range []byte(input) {
        result[i] = c ^ p.key
    }
    p.recordExecution(start, nil)
    return string(result), nil
}

type ValidatePlugin struct {
    BasePlugin
    minLen int
    maxLen int
}

func NewValidatePlugin(minLen, maxLen int) *ValidatePlugin {
    return &ValidatePlugin{
        BasePlugin: NewBasePlugin("validate", "1.0.0"),
        minLen:     minLen,
        maxLen:     maxLen,
    }
}

func (p *ValidatePlugin) Execute(input string) (string, error) {
    start := time.Now()
    
    var err error
    if len(input) < p.minLen {
        err = fmt.Errorf("input too short: %d < %d", len(input), p.minLen)
    } else if len(input) > p.maxLen {
        err = fmt.Errorf("input too long: %d > %d", len(input), p.maxLen)
    }
    
    p.recordExecution(start, err)
    
    if err != nil {
        return "", err
    }
    return input, nil
}

// Plugin Registry
type PluginRegistry struct {
    plugins map[string]Plugin
    order   []string
}

func NewPluginRegistry() *PluginRegistry {
    return &PluginRegistry{
        plugins: make(map[string]Plugin),
    }
}

func (r *PluginRegistry) Register(p Plugin) error {
    name := p.Name()
    if _, exists := r.plugins[name]; exists {
        return fmt.Errorf("plugin %q already registered", name)
    }
    
    if err := p.Init(); err != nil {
        return fmt.Errorf("init plugin %q: %w", name, err)
    }
    
    r.plugins[name] = p
    r.order = append(r.order, name)
    return nil
}

func (r *PluginRegistry) Unregister(name string) error {
    p, ok := r.plugins[name]
    if !ok {
        return fmt.Errorf("plugin %q not found", name)
    }
    
    if err := p.Cleanup(); err != nil {
        return fmt.Errorf("cleanup plugin %q: %w", name, err)
    }
    
    delete(r.plugins, name)
    for i, n := range r.order {
        if n == name {
            r.order = append(r.order[:i], r.order[i+1:]...)
            break
        }
    }
    return nil
}

func (r *PluginRegistry) Execute(input string) (string, error) {
    result := input
    for _, name := range r.order {
        p := r.plugins[name]
        var err error
        result, err = p.Execute(result)
        if err != nil {
            return "", fmt.Errorf("plugin %q: %w", name, err)
        }
    }
    return result, nil
}

func (r *PluginRegistry) ExecutePartial(input string, pluginNames ...string) (string, error) {
    result := input
    for _, name := range pluginNames {
        p, ok := r.plugins[name]
        if !ok {
            return "", fmt.Errorf("plugin %q not found", name)
        }
        var err error
        result, err = p.Execute(result)
        if err != nil {
            return "", fmt.Errorf("plugin %q: %w", name, err)
        }
    }
    return result, nil
}

func (r *PluginRegistry) PrintStats() {
    fmt.Println("=== Plugin Statistics ===")
    
    names := make([]string, 0, len(r.plugins))
    for name := range r.plugins {
        names = append(names, name)
    }
    sort.Strings(names)
    
    for _, name := range names {
        p := r.plugins[name]
        
        type statsGetter interface {
            Stats() PluginStats
        }
        
        if sg, ok := p.(statsGetter); ok {
            stats := sg.Stats()
            avgTime := time.Duration(0)
            if stats.Executions > 0 {
                avgTime = stats.TotalTime / time.Duration(stats.Executions)
            }
            fmt.Printf("  %s v%s: executions=%d, errors=%d, avgTime=%v\n",
                name, p.Version(), stats.Executions, stats.Errors, avgTime)
        }
    }
}

func (r *PluginRegistry) List() []string {
    return append([]string{}, r.order...)
}

func main() {
    registry := NewPluginRegistry()
    
    // ลงทะเบียน plugins
    registry.Register(NewValidatePlugin(3, 100))
    registry.Register(NewUpperCasePlugin())
    registry.Register(NewReversePlugin())
    
    fmt.Println("Registered plugins:", registry.List())
    
    // Execute pipeline
    fmt.Println("\n=== Execute Pipeline ===")
    tests := []string{
        "Hello, World!",
        "Go Programming",
        "ab", // too short
    }
    
    for _, test := range tests {
        result, err := registry.Execute(test)
        if err != nil {
            fmt.Printf("Input: %-20s → Error: %v\n", test, err)
        } else {
            fmt.Printf("Input: %-20s → %s\n", test, result)
        }
    }
    
    // Execute partial
    fmt.Println("\n=== Partial Execution ===")
    result, _ := registry.ExecutePartial("hello world", "uppercase")
    fmt.Println("uppercase only:", result)
    
    result, _ = registry.ExecutePartial("hello world", "reverse")
    fmt.Println("reverse only:", result)
    
    // Stats
    fmt.Println()
    registry.PrintStats()
    
    // Unregister
    fmt.Println("\n=== Unregister ===")
    registry.Unregister("reverse")
    fmt.Println("Plugins after unregister:", registry.List())
    
    // Execute without reverse
    result2, _ := registry.Execute("Hello Go!")
    fmt.Println("Result without reverse:", result2)
    
    // Cleanup all
    fmt.Println("\n=== Cleanup ===")
    for _, name := range registry.List() {
        registry.Unregister(name)
    }
}
```

---

## สรุป

| Concept | การใช้งาน |
|---------|----------|
| Struct Embedding | Code reuse, composition |
| Interface Embedding | Interface composition |
| Method Promotion | Automatic delegation |
| Override | Re-define promoted method |
| Multiple Embedding | Mixin pattern |

### Embedding Rules

1. Promoted methods และ fields ใช้ได้เหมือนเป็นของ struct เอง
2. ถ้ามี method/field ซ้ำ (ambiguous) ต้องระบุ path ชัดเจน
3. Outer struct method/field ซ่อน (shadow) inner เสมอ
4. Pointer embedding (`*T`) share state; value embedding (`T`) copy

## Resources

- [Effective Go: Embedding](https://go.dev/doc/effective_go#embedding)
- [Go spec: Struct types](https://go.dev/ref/spec#Struct_types)
- [Composition over inheritance](https://go.dev/doc/faq#inheritance)
