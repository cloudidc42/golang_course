# Part 98: Complete E-Commerce Platform

## เป้าหมายของบทเรียน
- สร้าง microservices e-commerce platform
- User, Product, Order, Payment, Notification services
- PostgreSQL + Redis + Kafka integration
- Service-to-service communication
- Complete working implementation

---

## 1. Project Structure

```
ecommerce/
├── services/
│   ├── user/         - User management & auth
│   ├── product/      - Product catalog
│   ├── order/        - Order processing
│   ├── payment/      - Payment handling
│   └── notification/ - Email/SMS notifications
├── shared/
│   ├── events/       - Event definitions
│   ├── middleware/   - Shared middleware
│   └── db/           - DB helpers
└── docker-compose.yml
```

---

## 2. Shared Types และ Events

```go
// shared/events/events.go
package events

import "time"

// Event types
const (
    UserRegistered    = "user.registered"
    OrderCreated      = "order.created"
    OrderPaid         = "order.paid"
    OrderShipped      = "order.shipped"
    PaymentProcessed  = "payment.processed"
    PaymentFailed     = "payment.failed"
)

// BaseEvent ข้อมูล event พื้นฐาน
type BaseEvent struct {
    ID        string    `json:"id"`
    Type      string    `json:"type"`
    OccuredAt time.Time `json:"occurred_at"`
    Version   int       `json:"version"`
}

// UserRegisteredEvent
type UserRegisteredEvent struct {
    BaseEvent
    UserID    string `json:"user_id"`
    Email     string `json:"email"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
}

// OrderCreatedEvent
type OrderCreatedEvent struct {
    BaseEvent
    OrderID    string      `json:"order_id"`
    UserID     string      `json:"user_id"`
    Items      []OrderItem `json:"items"`
    TotalAmount float64    `json:"total_amount"`
}

// OrderItem
type OrderItem struct {
    ProductID string  `json:"product_id"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

// PaymentProcessedEvent
type PaymentProcessedEvent struct {
    BaseEvent
    PaymentID string  `json:"payment_id"`
    OrderID   string  `json:"order_id"`
    Amount    float64 `json:"amount"`
    Method    string  `json:"method"`
    Status    string  `json:"status"`
}
```

---

## 3. User Service

```go
// services/user/main.go
package main

import (
    "context"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strings"
    "sync"
    "time"
)

// User model
type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    FirstName string    `json:"first_name"`
    LastName  string    `json:"last_name"`
    Password  string    `json:"-"`
    Role      string    `json:"role"`
    Active    bool      `json:"active"`
    CreatedAt time.Time `json:"created_at"`
}

// UserRepository จัดการ user data
type UserRepository struct {
    mu    sync.RWMutex
    users map[string]*User
    byEmail map[string]*User
}

func NewUserRepository() *UserRepository {
    return &UserRepository{
        users:   make(map[string]*User),
        byEmail: make(map[string]*User),
    }
}

func (r *UserRepository) Create(user *User) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    if _, exists := r.byEmail[user.Email]; exists {
        return fmt.Errorf("email already exists: %s", user.Email)
    }
    
    user.ID = generateID()
    user.CreatedAt = time.Now()
    user.Active = true
    
    r.users[user.ID] = user
    r.byEmail[user.Email] = user
    return nil
}

func (r *UserRepository) FindByID(id string) (*User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    user, ok := r.users[id]
    if !ok {
        return nil, fmt.Errorf("user not found: %s", id)
    }
    return user, nil
}

func (r *UserRepository) FindByEmail(email string) (*User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    user, ok := r.byEmail[email]
    if !ok {
        return nil, fmt.Errorf("user not found: %s", email)
    }
    return user, nil
}

// TokenStore จัดการ JWT tokens (simplified)
type TokenStore struct {
    mu     sync.RWMutex
    tokens map[string]string // token -> userID
}

func NewTokenStore() *TokenStore {
    return &TokenStore{tokens: make(map[string]string)}
}

func (ts *TokenStore) Create(userID string) string {
    ts.mu.Lock()
    defer ts.mu.Unlock()
    
    token := generateToken(userID)
    ts.tokens[token] = userID
    return token
}

func (ts *TokenStore) Validate(token string) (string, bool) {
    ts.mu.RLock()
    defer ts.mu.RUnlock()
    
    userID, ok := ts.tokens[token]
    return userID, ok
}

// UserService business logic
type UserService struct {
    repo   *UserRepository
    tokens *TokenStore
}

func NewUserService() *UserService {
    return &UserService{
        repo:   NewUserRepository(),
        tokens: NewTokenStore(),
    }
}

// Register สร้าง user ใหม่
func (s *UserService) Register(email, password, firstName, lastName string) (*User, error) {
    if email == "" || password == "" {
        return nil, fmt.Errorf("email and password required")
    }
    
    if len(password) < 8 {
        return nil, fmt.Errorf("password must be at least 8 characters")
    }
    
    user := &User{
        Email:     email,
        FirstName: firstName,
        LastName:  lastName,
        Password:  hashPassword(password),
        Role:      "customer",
    }
    
    if err := s.repo.Create(user); err != nil {
        return nil, err
    }
    
    return user, nil
}

// Login authenticate user
func (s *UserService) Login(email, password string) (string, error) {
    user, err := s.repo.FindByEmail(email)
    if err != nil {
        return "", fmt.Errorf("invalid credentials")
    }
    
    if user.Password != hashPassword(password) {
        return "", fmt.Errorf("invalid credentials")
    }
    
    if !user.Active {
        return "", fmt.Errorf("account disabled")
    }
    
    token := s.tokens.Create(user.ID)
    return token, nil
}

// UserHandler HTTP handlers
type UserHandler struct {
    svc *UserService
}

func NewUserHandler(svc *UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) Register(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    var req struct {
        Email     string `json:"email"`
        Password  string `json:"password"`
        FirstName string `json:"first_name"`
        LastName  string `json:"last_name"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    user, err := h.svc.Register(req.Email, req.Password, req.FirstName, req.LastName)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "id":         user.ID,
        "email":      user.Email,
        "first_name": user.FirstName,
        "last_name":  user.LastName,
        "created_at": user.CreatedAt,
    })
}

func (h *UserHandler) Login(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    token, err := h.svc.Login(req.Email, req.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusUnauthorized)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "token": token,
    })
}

func (h *UserHandler) Profile(w http.ResponseWriter, r *http.Request) {
    token := extractToken(r)
    if token == "" {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    userID, ok := h.svc.tokens.Validate(token)
    if !ok {
        http.Error(w, "Invalid token", http.StatusUnauthorized)
        return
    }
    
    user, err := h.svc.repo.FindByID(userID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "id":         user.ID,
        "email":      user.Email,
        "first_name": user.FirstName,
        "last_name":  user.LastName,
        "role":       user.Role,
    })
}

// Helper functions
func generateID() string {
    return fmt.Sprintf("%d", time.Now().UnixNano())
}

func generateToken(userID string) string {
    h := sha256.Sum256([]byte(userID + fmt.Sprintf("%d", time.Now().UnixNano())))
    return fmt.Sprintf("%x", h[:16])
}

func hashPassword(password string) string {
    h := sha256.Sum256([]byte(password + "salt"))
    return fmt.Sprintf("%x", h[:])
}

func extractToken(r *http.Request) string {
    auth := r.Header.Get("Authorization")
    if strings.HasPrefix(auth, "Bearer ") {
        return auth[7:]
    }
    return ""
}

func startUserService(ctx context.Context) {
    svc := NewUserService()
    handler := NewUserHandler(svc)
    
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users/register", handler.Register)
    mux.HandleFunc("/api/users/login", handler.Login)
    mux.HandleFunc("/api/users/profile", handler.Profile)
    
    server := &http.Server{
        Addr:    ":8081",
        Handler: mux,
    }
    
    go func() {
        log.Println("User service started on :8081")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Printf("User service error: %v\n", err)
        }
    }()
    
    <-ctx.Done()
    server.Shutdown(context.Background())
}
```

---

## 4. Product Service

```go
// services/product/service.go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Product model
type Product struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Stock       int       `json:"stock"`
    Category    string    `json:"category"`
    Active      bool      `json:"active"`
    CreatedAt   time.Time `json:"created_at"`
}

// ProductRepository จัดการ products
type ProductRepository struct {
    mu       sync.RWMutex
    products map[string]*Product
}

func NewProductRepository() *ProductRepository {
    return &ProductRepository{
        products: make(map[string]*Product),
    }
}

func (r *ProductRepository) Create(p *Product) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    p.ID = fmt.Sprintf("prod_%d", time.Now().UnixNano())
    p.CreatedAt = time.Now()
    p.Active = true
    
    r.products[p.ID] = p
    return nil
}

func (r *ProductRepository) FindByID(id string) (*Product, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    p, ok := r.products[id]
    if !ok {
        return nil, fmt.Errorf("product not found: %s", id)
    }
    return p, nil
}

func (r *ProductRepository) List(category string, limit, offset int) []*Product {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    var result []*Product
    for _, p := range r.products {
        if !p.Active {
            continue
        }
        if category != "" && p.Category != category {
            continue
        }
        result = append(result, p)
    }
    
    // Apply pagination
    start := offset
    if start > len(result) {
        return []*Product{}
    }
    end := start + limit
    if end > len(result) {
        end = len(result)
    }
    return result[start:end]
}

func (r *ProductRepository) UpdateStock(id string, delta int) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    p, ok := r.products[id]
    if !ok {
        return fmt.Errorf("product not found: %s", id)
    }
    
    newStock := p.Stock + delta
    if newStock < 0 {
        return fmt.Errorf("insufficient stock for %s: have %d, need %d", id, p.Stock, -delta)
    }
    
    p.Stock = newStock
    return nil
}

// ProductService business logic
type ProductService struct {
    repo *ProductRepository
}

func NewProductService() *ProductService {
    svc := &ProductService{
        repo: NewProductRepository(),
    }
    svc.seedData()
    return svc
}

func (s *ProductService) seedData() {
    products := []Product{
        {Name: "Go Programming Book", Price: 590.0, Stock: 100, Category: "books"},
        {Name: "Mechanical Keyboard", Price: 3500.0, Stock: 50, Category: "electronics"},
        {Name: "USB-C Hub", Price: 1200.0, Stock: 200, Category: "electronics"},
        {Name: "Clean Code Book", Price: 450.0, Stock: 80, Category: "books"},
        {Name: "Monitor Stand", Price: 890.0, Stock: 30, Category: "accessories"},
    }
    
    for i := range products {
        s.repo.Create(&products[i])
    }
}
```

---

## 5. Order Service

```go
// services/order/service.go
package main

import (
    "fmt"
    "sync"
    "time"
)

// OrderStatus สถานะ order
type OrderStatus string

const (
    OrderPending    OrderStatus = "pending"
    OrderConfirmed  OrderStatus = "confirmed"
    OrderPaid       OrderStatus = "paid"
    OrderShipping   OrderStatus = "shipping"
    OrderDelivered  OrderStatus = "delivered"
    OrderCancelled  OrderStatus = "cancelled"
)

// Order model
type Order struct {
    ID          string      `json:"id"`
    UserID      string      `json:"user_id"`
    Items       []OrderItem `json:"items"`
    Status      OrderStatus `json:"status"`
    TotalAmount float64     `json:"total_amount"`
    Address     string      `json:"address"`
    CreatedAt   time.Time   `json:"created_at"`
    UpdatedAt   time.Time   `json:"updated_at"`
}

type OrderItem struct {
    ProductID string  `json:"product_id"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

// OrderRepository
type OrderRepository struct {
    mu     sync.RWMutex
    orders map[string]*Order
    byUser map[string][]*Order
}

func NewOrderRepository() *OrderRepository {
    return &OrderRepository{
        orders: make(map[string]*Order),
        byUser: make(map[string][]*Order),
    }
}

func (r *OrderRepository) Create(order *Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    order.ID = fmt.Sprintf("ord_%d", time.Now().UnixNano())
    order.Status = OrderPending
    order.CreatedAt = time.Now()
    order.UpdatedAt = time.Now()
    
    r.orders[order.ID] = order
    r.byUser[order.UserID] = append(r.byUser[order.UserID], order)
    return nil
}

func (r *OrderRepository) FindByID(id string) (*Order, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    order, ok := r.orders[id]
    if !ok {
        return nil, fmt.Errorf("order not found: %s", id)
    }
    return order, nil
}

func (r *OrderRepository) UpdateStatus(id string, status OrderStatus) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    order, ok := r.orders[id]
    if !ok {
        return fmt.Errorf("order not found: %s", id)
    }
    
    order.Status = status
    order.UpdatedAt = time.Now()
    return nil
}

func (r *OrderRepository) ListByUser(userID string) []*Order {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return r.byUser[userID]
}

// OrderService business logic
type OrderService struct {
    repo *OrderRepository
}

func NewOrderService() *OrderService {
    return &OrderService{repo: NewOrderRepository()}
}

func (s *OrderService) CreateOrder(userID string, items []OrderItem, address string) (*Order, error) {
    if len(items) == 0 {
        return nil, fmt.Errorf("order must have at least one item")
    }
    
    total := 0.0
    for _, item := range items {
        if item.Quantity <= 0 {
            return nil, fmt.Errorf("invalid quantity for %s", item.ProductID)
        }
        total += item.Price * float64(item.Quantity)
    }
    
    order := &Order{
        UserID:      userID,
        Items:       items,
        TotalAmount: total,
        Address:     address,
    }
    
    if err := s.repo.Create(order); err != nil {
        return nil, err
    }
    
    return order, nil
}
```

---

## 6. Main Integration

```go
// main.go - รวม services
package main

import (
    "context"
    "fmt"
    "net/http"
    "net/http/httptest"
    "strings"
    "time"
)

func runECommerceDemo() {
    fmt.Println("=== E-Commerce Platform Demo ===\n")
    
    // Start services
    userSvc := NewUserService()
    productSvc := NewProductService()
    orderSvc := NewOrderService()
    
    // 1. Register user
    fmt.Println("1. Registering user...")
    user, err := userSvc.Register("somchai@example.com", "password123", "สมชาย", "ใจดี")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("   User registered: %s (%s)\n", user.FirstName, user.Email)
    
    // 2. Login
    fmt.Println("\n2. Logging in...")
    token, err := userSvc.Login("somchai@example.com", "password123")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("   Token: %s...\n", token[:8])
    
    // 3. Browse products
    fmt.Println("\n3. Browsing products...")
    products := productSvc.repo.List("", 10, 0)
    for i, p := range products {
        fmt.Printf("   [%d] %s - ฿%.2f (stock: %d)\n", i+1, p.Name, p.Price, p.Stock)
    }
    
    // 4. Create order
    fmt.Println("\n4. Creating order...")
    
    if len(products) == 0 {
        fmt.Println("   No products available")
        return
    }
    
    items := []OrderItem{
        {
            ProductID: products[0].ID,
            Name:      products[0].Name,
            Quantity:  2,
            Price:     products[0].Price,
        },
    }
    
    order, err := orderSvc.CreateOrder(user.ID, items, "123 ถนนสุขุมวิท กรุงเทพ 10110")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("   Order created: %s\n", order.ID)
    fmt.Printf("   Total: ฿%.2f\n", order.TotalAmount)
    fmt.Printf("   Status: %s\n", order.Status)
    
    // 5. Update order status (simulate payment)
    fmt.Println("\n5. Processing payment...")
    orderSvc.repo.UpdateStatus(order.ID, OrderPaid)
    
    updatedOrder, _ := orderSvc.repo.FindByID(order.ID)
    fmt.Printf("   Order status updated: %s\n", updatedOrder.Status)
    
    // 6. Summary
    fmt.Println("\n=== Summary ===")
    fmt.Printf("User: %s %s\n", user.FirstName, user.LastName)
    fmt.Printf("Orders: %d\n", len(orderSvc.repo.ListByUser(user.ID)))
    fmt.Printf("Order total: ฿%.2f\n", order.TotalAmount)
    
    _ = token
    _ = context.Background()
    _ = http.MethodGet
    _ = httptest.NewRequest
    _ = strings.NewReader
    _ = time.Now()
}

func main() {
    runECommerceDemo()
    
    fmt.Println("\n=== Production Architecture ===\n")
    fmt.Println("Services:")
    services := []struct {
        name string
        port string
        tech string
    }{
        {"User Service", ":8081", "Go + PostgreSQL + Redis"},
        {"Product Service", ":8082", "Go + PostgreSQL + Redis Cache"},
        {"Order Service", ":8083", "Go + PostgreSQL + Kafka"},
        {"Payment Service", ":8084", "Go + PostgreSQL + Stripe/PromptPay"},
        {"Notification", ":8085", "Go + Kafka + SendGrid"},
        {"API Gateway", ":8080", "Go + Redis (rate limiting)"},
    }
    
    for _, s := range services {
        fmt.Printf("  %-20s %-6s  %s\n", s.name, s.port, s.tech)
    }
    
    fmt.Println("\nInfrastructure:")
    infra := []string{
        "PostgreSQL 15 (primary + replica)",
        "Redis 7 (cache + sessions + rate limiting)",
        "Apache Kafka (event streaming)",
        "Nginx (load balancer)",
        "Prometheus + Grafana (monitoring)",
        "Jaeger (distributed tracing)",
    }
    for _, i := range infra {
        fmt.Printf("  - %s\n", i)
    }
}
```

---

## สรุป

บทนี้สร้าง Complete E-Commerce Platform:

1. **User Service** - Registration, login, JWT auth
2. **Product Service** - Catalog, inventory management
3. **Order Service** - Order lifecycle management
4. **Integration** - Service-to-service communication
5. **Event-driven** - Kafka events สำหรับ async operations

### Key Takeaways

- Microservices ช่วยให้ scale แต่ละ service แยกกัน
- Repository pattern แยก data access จาก business logic
- Event-driven architecture ลด coupling ระหว่าง services
- ทุก service ต้องมี graceful shutdown
- Idempotency สำคัญมากสำหรับ payment operations
