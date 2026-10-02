# Part 51: Microservices Architecture with Go

## เป้าหมายการเรียนรู้
- เข้าใจความแตกต่างระหว่าง Microservices และ Monolith
- รู้จักเมื่อไหร่ควรใช้ Microservices
- เรียนรู้กลยุทธ์การแบ่ง Service
- เข้าใจ Communication patterns ทั้ง sync และ async
- เรียนรู้การจัดการข้อมูลใน Microservices
- ทำความเข้าใจ 12-Factor App principles
- สร้าง Go microservice project structure

---

## 1. Microservices vs Monolith

### Monolithic Architecture

Monolith คือแอปพลิเคชันที่รวมทุกฟังก์ชันไว้ในที่เดียว deploy ด้วยกัน scale ด้วยกัน

```go
// monolith/main.go - ตัวอย่าง Monolith แบบง่าย
package main

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

// โดเมนทุกอย่างอยู่ใน package เดียว
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
	Stock int     `json:"stock"`
}

type Order struct {
	ID        int     `json:"id"`
	UserID    int     `json:"user_id"`
	ProductID int     `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Total     float64 `json:"total"`
}

// handlers ทั้งหมดอยู่ในไฟล์เดียวกัน
func getUsers(w http.ResponseWriter, r *http.Request) {
	users := []User{
		{ID: 1, Name: "Alice", Email: "alice@example.com"},
	}
	json.NewEncoder(w).Encode(users)
}

func getProducts(w http.ResponseWriter, r *http.Request) {
	products := []Product{
		{ID: 1, Name: "Laptop", Price: 999.99, Stock: 10},
	}
	json.NewEncoder(w).Encode(products)
}

func createOrder(w http.ResponseWriter, r *http.Request) {
	// logic ทั้งหมดอยู่ที่นี่
	order := Order{ID: 1, UserID: 1, ProductID: 1, Quantity: 1, Total: 999.99}
	json.NewEncoder(w).Encode(order)
}

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/users", getUsers).Methods("GET")
	r.HandleFunc("/products", getProducts).Methods("GET")
	r.HandleFunc("/orders", createOrder).Methods("POST")
	log.Fatal(http.ListenAndServe(":8080", r))
}
```

### Microservices Architecture

Microservices แบ่งแอปพลิเคชันเป็น services ขนาดเล็กที่ทำงานอิสระ

```go
// user-service/main.go - แยก User service ออกมา
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strconv"

	"github.com/gorilla/mux"
)

type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

type UserService struct {
	users map[int]User
	next  int
}

func NewUserService() *UserService {
	return &UserService{
		users: make(map[int]User),
		next:  1,
	}
}

func (s *UserService) GetUser(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id, err := strconv.Atoi(vars["id"])
	if err != nil {
		http.Error(w, "Invalid user ID", http.StatusBadRequest)
		return
	}

	user, ok := s.users[id]
	if !ok {
		http.Error(w, "User not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(user)
}

func (s *UserService) CreateUser(w http.ResponseWriter, r *http.Request) {
	var user User
	if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
		http.Error(w, "Invalid request body", http.StatusBadRequest)
		return
	}

	user.ID = s.next
	s.next++
	s.users[user.ID] = user

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(user)
}

func (s *UserService) ListUsers(w http.ResponseWriter, r *http.Request) {
	var users []User
	for _, u := range s.users {
		users = append(users, u)
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(users)
}

func main() {
	svc := NewUserService()
	r := mux.NewRouter()

	r.HandleFunc("/users", svc.ListUsers).Methods("GET")
	r.HandleFunc("/users/{id}", svc.GetUser).Methods("GET")
	r.HandleFunc("/users", svc.CreateUser).Methods("POST")

	log.Println("User Service listening on :8081")
	log.Fatal(http.ListenAndServe(":8081", r))
}
```

---

## 2. เมื่อไหร่ควรใช้ Microservices

### การตัดสินใจ

```go
// ตัวอย่าง: การวิเคราะห์ขนาด team และ codebase เพื่อตัดสินใจ

// criteria/analyzer.go
package criteria

import "fmt"

type TeamSize int
type CodebaseSize int

const (
	SmallTeam  TeamSize = iota // < 5 คน
	MediumTeam                 // 5-20 คน
	LargeTeam                  // > 20 คน
)

const (
	SmallCodebase  CodebaseSize = iota // < 10K LOC
	MediumCodebase                     // 10K-100K LOC
	LargeCodebase                      // > 100K LOC
)

type ArchitectureRecommendation struct {
	Architecture string
	Reasons      []string
	Warnings     []string
}

func AnalyzeArchitecture(
	teamSize TeamSize,
	codebaseSize CodebaseSize,
	deployFrequency string,
	scalingNeeds bool,
) ArchitectureRecommendation {
	rec := ArchitectureRecommendation{}

	switch {
	case teamSize == SmallTeam && codebaseSize == SmallCodebase:
		rec.Architecture = "Monolith"
		rec.Reasons = []string{
			"ทีมขนาดเล็กไม่ต้องการ overhead จาก microservices",
			"codebase ขนาดเล็กยังจัดการได้",
			"ประหยัดเวลาและทรัพยากร",
		}
		rec.Warnings = []string{
			"วางแผนสถาปัตยกรรมให้พร้อมขยายในอนาคต",
		}

	case teamSize == LargeTeam || codebaseSize == LargeCodebase:
		rec.Architecture = "Microservices"
		rec.Reasons = []string{
			"ทีมใหญ่ทำงานแยกกันได้โดยไม่กระทบกัน",
			"deploy อิสระในแต่ละ service",
			"scale เฉพาะส่วนที่ต้องการได้",
		}
		rec.Warnings = []string{
			"ต้องการ DevOps expertise สูง",
			"complexity ของ distributed system",
		}

	default:
		rec.Architecture = "Modular Monolith"
		rec.Reasons = []string{
			"เป็น middle ground ที่ดี",
			"แยก module ชัดเจน พร้อมแปลงเป็น microservicesในอนาคต",
		}
	}

	if scalingNeeds {
		rec.Reasons = append(rec.Reasons, fmt.Sprintf("ต้องการ scale ตามโหลด: %s", deployFrequency))
	}

	return rec
}
```

---

## 3. Service Decomposition Strategies

### Decompose by Business Capability

```go
// ตัวอย่าง: E-commerce แบ่ง services ตาม business capability

// services/catalog/service.go
package catalog

import (
	"context"
	"errors"
)

// Product catalog service - จัดการสินค้า
type Product struct {
	ID          string  `json:"id"`
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	CategoryID  string  `json:"category_id"`
}

type CatalogRepository interface {
	FindByID(ctx context.Context, id string) (*Product, error)
	FindAll(ctx context.Context, filter ProductFilter) ([]Product, error)
	Save(ctx context.Context, p *Product) error
	Delete(ctx context.Context, id string) error
}

type ProductFilter struct {
	CategoryID string
	MinPrice   float64
	MaxPrice   float64
	SearchTerm string
}

type CatalogService struct {
	repo CatalogRepository
}

func NewCatalogService(repo CatalogRepository) *CatalogService {
	return &CatalogService{repo: repo}
}

var ErrProductNotFound = errors.New("product not found")

func (s *CatalogService) GetProduct(ctx context.Context, id string) (*Product, error) {
	p, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return nil, ErrProductNotFound
	}
	return p, nil
}

func (s *CatalogService) SearchProducts(ctx context.Context, filter ProductFilter) ([]Product, error) {
	return s.repo.FindAll(ctx, filter)
}
```

```go
// services/inventory/service.go
package inventory

import (
	"context"
	"errors"
	"sync"
)

// Inventory service - จัดการสต็อก (แยกจาก Catalog)
type StockLevel struct {
	ProductID string `json:"product_id"`
	Quantity  int    `json:"quantity"`
	Reserved  int    `json:"reserved"`
}

func (s *StockLevel) Available() int {
	return s.Quantity - s.Reserved
}

type InventoryService struct {
	mu    sync.RWMutex
	stock map[string]*StockLevel
}

func NewInventoryService() *InventoryService {
	return &InventoryService{
		stock: make(map[string]*StockLevel),
	}
}

var ErrInsufficientStock = errors.New("insufficient stock")

func (s *InventoryService) CheckAvailability(ctx context.Context, productID string, qty int) error {
	s.mu.RLock()
	defer s.mu.RUnlock()

	level, ok := s.stock[productID]
	if !ok || level.Available() < qty {
		return ErrInsufficientStock
	}
	return nil
}

func (s *InventoryService) ReserveStock(ctx context.Context, productID string, qty int) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	level, ok := s.stock[productID]
	if !ok || level.Available() < qty {
		return ErrInsufficientStock
	}

	level.Reserved += qty
	return nil
}

func (s *InventoryService) ReleaseStock(ctx context.Context, productID string, qty int) {
	s.mu.Lock()
	defer s.mu.Unlock()

	if level, ok := s.stock[productID]; ok {
		if level.Reserved >= qty {
			level.Reserved -= qty
		}
	}
}

func (s *InventoryService) ConfirmDeduction(ctx context.Context, productID string, qty int) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	level, ok := s.stock[productID]
	if !ok {
		return errors.New("product not found in inventory")
	}

	if level.Reserved < qty || level.Quantity < qty {
		return ErrInsufficientStock
	}

	level.Reserved -= qty
	level.Quantity -= qty
	return nil
}
```

### Decompose by Subdomain (DDD)

```go
// Bounded Context: Ordering
// services/ordering/domain.go
package ordering

import (
	"errors"
	"time"
)

type OrderID string
type CustomerID string
type Money float64

type OrderStatus int

const (
	OrderStatusPending   OrderStatus = iota
	OrderStatusConfirmed
	OrderStatusShipped
	OrderStatusDelivered
	OrderStatusCancelled
)

func (s OrderStatus) String() string {
	switch s {
	case OrderStatusPending:
		return "PENDING"
	case OrderStatusConfirmed:
		return "CONFIRMED"
	case OrderStatusShipped:
		return "SHIPPED"
	case OrderStatusDelivered:
		return "DELIVERED"
	case OrderStatusCancelled:
		return "CANCELLED"
	default:
		return "UNKNOWN"
	}
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Name      string  `json:"name"`
	Quantity  int     `json:"quantity"`
	UnitPrice Money   `json:"unit_price"`
}

func (i OrderItem) Subtotal() Money {
	return Money(float64(i.UnitPrice) * float64(i.Quantity))
}

type Order struct {
	ID         OrderID
	CustomerID CustomerID
	Items      []OrderItem
	Status     OrderStatus
	CreatedAt  time.Time
	UpdatedAt  time.Time
}

func NewOrder(customerID CustomerID, items []OrderItem) (*Order, error) {
	if len(items) == 0 {
		return nil, errors.New("order must have at least one item")
	}
	for _, item := range items {
		if item.Quantity <= 0 {
			return nil, errors.New("item quantity must be positive")
		}
		if item.UnitPrice <= 0 {
			return nil, errors.New("item price must be positive")
		}
	}

	return &Order{
		CustomerID: customerID,
		Items:      items,
		Status:     OrderStatusPending,
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}, nil
}

func (o *Order) Total() Money {
	var total Money
	for _, item := range o.Items {
		total += item.Subtotal()
	}
	return total
}

func (o *Order) Confirm() error {
	if o.Status != OrderStatusPending {
		return errors.New("can only confirm pending orders")
	}
	o.Status = OrderStatusConfirmed
	o.UpdatedAt = time.Now()
	return nil
}

func (o *Order) Cancel() error {
	if o.Status == OrderStatusShipped || o.Status == OrderStatusDelivered {
		return errors.New("cannot cancel shipped or delivered orders")
	}
	o.Status = OrderStatusCancelled
	o.UpdatedAt = time.Now()
	return nil
}
```

---

## 4. Communication Patterns: Sync vs Async

### Synchronous Communication (REST/HTTP)

```go
// communication/sync/client.go
package sync

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

// HTTP client สำหรับ inter-service communication
type ServiceClient struct {
	baseURL    string
	httpClient *http.Client
}

func NewServiceClient(baseURL string) *ServiceClient {
	return &ServiceClient{
		baseURL: baseURL,
		httpClient: &http.Client{
			Timeout: 10 * time.Second,
		},
	}
}

type UserResponse struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func (c *ServiceClient) GetUser(ctx context.Context, userID int) (*UserResponse, error) {
	url := fmt.Sprintf("%s/users/%d", c.baseURL, userID)

	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, fmt.Errorf("creating request: %w", err)
	}

	// propagate trace headers
	if traceID := ctx.Value("trace-id"); traceID != nil {
		req.Header.Set("X-Trace-ID", fmt.Sprint(traceID))
	}

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("executing request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode == http.StatusNotFound {
		return nil, fmt.Errorf("user %d not found", userID)
	}
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
	}

	var user UserResponse
	if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
		return nil, fmt.Errorf("decoding response: %w", err)
	}
	return &user, nil
}
```

### Asynchronous Communication (Message Queue)

```go
// communication/async/publisher.go
package async

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"
)

// Event-based communication
type EventType string

const (
	EventOrderCreated   EventType = "order.created"
	EventOrderConfirmed EventType = "order.confirmed"
	EventOrderCancelled EventType = "order.cancelled"
	EventPaymentSuccess EventType = "payment.success"
	EventPaymentFailed  EventType = "payment.failed"
)

type Event struct {
	ID        string                 `json:"id"`
	Type      EventType              `json:"type"`
	Source    string                 `json:"source"`
	Data      map[string]interface{} `json:"data"`
	Timestamp time.Time              `json:"timestamp"`
}

// Publisher interface
type Publisher interface {
	Publish(ctx context.Context, topic string, event Event) error
}

// Subscriber interface
type Subscriber interface {
	Subscribe(topic string, handler EventHandler) error
	Start(ctx context.Context) error
}

type EventHandler func(ctx context.Context, event Event) error

// In-memory event bus (สำหรับ development/testing)
type InMemoryEventBus struct {
	handlers map[string][]EventHandler
	events   chan struct {
		topic string
		event Event
	}
}

func NewInMemoryEventBus() *InMemoryEventBus {
	return &InMemoryEventBus{
		handlers: make(map[string][]EventHandler),
		events: make(chan struct {
			topic string
			event Event
		}, 100),
	}
}

func (b *InMemoryEventBus) Publish(ctx context.Context, topic string, event Event) error {
	select {
	case b.events <- struct {
		topic string
		event Event
	}{topic, event}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (b *InMemoryEventBus) Subscribe(topic string, handler EventHandler) error {
	b.handlers[topic] = append(b.handlers[topic], handler)
	return nil
}

func (b *InMemoryEventBus) Start(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case msg := <-b.events:
			handlers, ok := b.handlers[msg.topic]
			if !ok {
				continue
			}
			for _, h := range handlers {
				go func(handler EventHandler, evt Event) {
					if err := handler(ctx, evt); err != nil {
						log.Printf("Error handling event %s: %v", evt.Type, err)
					}
				}(h, msg.event)
			}
		}
	}
}

// ตัวอย่างการใช้งาน
func ExampleAsyncCommunication() {
	bus := NewInMemoryEventBus()
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Inventory service subscribe to order events
	bus.Subscribe(string(EventOrderCreated), func(ctx context.Context, event Event) error {
		data, _ := json.Marshal(event.Data)
		fmt.Printf("Inventory: Processing order %s\n", string(data))
		// ลด stock
		return nil
	})

	// Notification service subscribe to order events
	bus.Subscribe(string(EventOrderCreated), func(ctx context.Context, event Event) error {
		fmt.Printf("Notification: Sending email for order\n")
		return nil
	})

	// Start bus
	go bus.Start(ctx)

	// Publish event
	event := Event{
		ID:     "evt-001",
		Type:   EventOrderCreated,
		Source: "order-service",
		Data: map[string]interface{}{
			"order_id":   "ord-001",
			"customer_id": "cust-001",
			"total":      99.99,
		},
		Timestamp: time.Now(),
	}

	bus.Publish(ctx, string(EventOrderCreated), event)
	time.Sleep(100 * time.Millisecond) // รอ async processing
}
```

---

## 5. Data Management in Microservices

### Database per Service Pattern

```go
// data/database_per_service.go
package data

import (
	"database/sql"
	"fmt"
	"log"

	_ "github.com/lib/pq"
)

// แต่ละ service มี database เป็นของตัวเอง

// UserDB - database สำหรับ user service
type UserDB struct {
	db *sql.DB
}

func NewUserDB(dsn string) (*UserDB, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("opening user db: %w", err)
	}
	return &UserDB{db: db}, nil
}

func (u *UserDB) Migrate() error {
	_, err := u.db.Exec(`
		CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			name VARCHAR(255) NOT NULL,
			email VARCHAR(255) UNIQUE NOT NULL,
			created_at TIMESTAMP DEFAULT NOW()
		)
	`)
	return err
}

// OrderDB - database สำหรับ order service (แยกกัน)
type OrderDB struct {
	db *sql.DB
}

func NewOrderDB(dsn string) (*OrderDB, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("opening order db: %w", err)
	}
	return &OrderDB{db: db}, nil
}

func (o *OrderDB) Migrate() error {
	_, err := o.db.Exec(`
		CREATE TABLE IF NOT EXISTS orders (
			id SERIAL PRIMARY KEY,
			customer_id INTEGER NOT NULL,
			status VARCHAR(50) NOT NULL DEFAULT 'pending',
			total DECIMAL(10,2) NOT NULL,
			created_at TIMESTAMP DEFAULT NOW()
		);
		CREATE TABLE IF NOT EXISTS order_items (
			id SERIAL PRIMARY KEY,
			order_id INTEGER REFERENCES orders(id),
			product_id INTEGER NOT NULL,
			quantity INTEGER NOT NULL,
			unit_price DECIMAL(10,2) NOT NULL
		);
	`)
	return err
}

// Saga pattern สำหรับ distributed transaction
type CreateOrderSaga struct {
	userClient     interface{ GetUser(id int) error }
	inventoryClient interface{ ReserveStock(productID string, qty int) error }
	paymentClient  interface{ ProcessPayment(amount float64) error }
	orderDB        *OrderDB
}

type SagaStep struct {
	Name       string
	Execute    func() error
	Compensate func() error
}

func RunSaga(steps []SagaStep) error {
	executed := make([]int, 0)

	for i, step := range steps {
		log.Printf("Executing saga step: %s", step.Name)
		if err := step.Execute(); err != nil {
			log.Printf("Saga step %s failed: %v, compensating...", step.Name, err)
			// compensate ย้อนกลับ steps ที่ทำไปแล้ว
			for j := len(executed) - 1; j >= 0; j-- {
				idx := executed[j]
				if steps[idx].Compensate != nil {
					if cErr := steps[idx].Compensate(); cErr != nil {
						log.Printf("Compensation failed for step %s: %v", steps[idx].Name, cErr)
					}
				}
			}
			return fmt.Errorf("saga failed at step %s: %w", step.Name, err)
		}
		executed = append(executed, i)
	}
	return nil
}
```

---

## 6. 12-Factor App Principles

```go
// twelve_factor/config.go
package twelve_factor

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

// Factor III: Config - เก็บ config ใน environment variables
type Config struct {
	// Server
	Port     int
	Host     string
	
	// Database
	DBHost     string
	DBPort     int
	DBName     string
	DBUser     string
	DBPassword string
	
	// Cache
	RedisAddr     string
	RedisPassword string
	
	// Service URLs
	UserServiceURL    string
	OrderServiceURL   string
	
	// Feature flags
	EnableMetrics bool
	EnableTracing bool
	
	// Timeouts
	RequestTimeout time.Duration
}

func LoadConfig() (*Config, error) {
	cfg := &Config{}
	var errs []string

	// Helper functions
	getEnv := func(key, defaultVal string) string {
		if val := os.Getenv(key); val != "" {
			return val
		}
		return defaultVal
	}

	getEnvInt := func(key string, defaultVal int) int {
		if val := os.Getenv(key); val != "" {
			n, err := strconv.Atoi(val)
			if err != nil {
				errs = append(errs, fmt.Sprintf("%s must be integer: %v", key, err))
				return defaultVal
			}
			return n
		}
		return defaultVal
	}

	getEnvBool := func(key string, defaultVal bool) bool {
		if val := os.Getenv(key); val != "" {
			b, err := strconv.ParseBool(val)
			if err != nil {
				errs = append(errs, fmt.Sprintf("%s must be boolean: %v", key, err))
				return defaultVal
			}
			return b
		}
		return defaultVal
	}

	getRequiredEnv := func(key string) string {
		val := os.Getenv(key)
		if val == "" {
			errs = append(errs, fmt.Sprintf("%s is required", key))
		}
		return val
	}

	// Load config
	cfg.Port = getEnvInt("PORT", 8080)
	cfg.Host = getEnv("HOST", "0.0.0.0")
	
	cfg.DBHost = getRequiredEnv("DB_HOST")
	cfg.DBPort = getEnvInt("DB_PORT", 5432)
	cfg.DBName = getRequiredEnv("DB_NAME")
	cfg.DBUser = getRequiredEnv("DB_USER")
	cfg.DBPassword = getRequiredEnv("DB_PASSWORD")
	
	cfg.RedisAddr = getEnv("REDIS_ADDR", "localhost:6379")
	cfg.RedisPassword = getEnv("REDIS_PASSWORD", "")
	
	cfg.UserServiceURL = getRequiredEnv("USER_SERVICE_URL")
	cfg.OrderServiceURL = getRequiredEnv("ORDER_SERVICE_URL")
	
	cfg.EnableMetrics = getEnvBool("ENABLE_METRICS", true)
	cfg.EnableTracing = getEnvBool("ENABLE_TRACING", false)
	
	timeoutSecs := getEnvInt("REQUEST_TIMEOUT_SECONDS", 30)
	cfg.RequestTimeout = time.Duration(timeoutSecs) * time.Second

	if len(errs) > 0 {
		return nil, fmt.Errorf("configuration errors: %v", errs)
	}

	return cfg, nil
}

func (c *Config) DBConnectionString() string {
	return fmt.Sprintf(
		"host=%s port=%d user=%s password=%s dbname=%s sslmode=disable",
		c.DBHost, c.DBPort, c.DBUser, c.DBPassword, c.DBName,
	)
}
```

```go
// twelve_factor/graceful_shutdown.go
package twelve_factor

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// Factor IX: Disposability - graceful shutdown
type Server struct {
	httpServer *http.Server
}

func NewServer(port string, handler http.Handler) *Server {
	return &Server{
		httpServer: &http.Server{
			Addr:         ":" + port,
			Handler:      handler,
			ReadTimeout:  15 * time.Second,
			WriteTimeout: 15 * time.Second,
			IdleTimeout:  60 * time.Second,
		},
	}
}

func (s *Server) Run() error {
	// Start server
	serverError := make(chan error, 1)
	go func() {
		log.Printf("Server starting on %s", s.httpServer.Addr)
		if err := s.httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			serverError <- err
		}
	}()

	// Wait for shutdown signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-serverError:
		return fmt.Errorf("server error: %w", err)
	case sig := <-quit:
		log.Printf("Received signal: %v, initiating graceful shutdown...", sig)
	}

	// Graceful shutdown with timeout
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := s.httpServer.Shutdown(ctx); err != nil {
		return fmt.Errorf("graceful shutdown failed: %w", err)
	}

	log.Println("Server stopped gracefully")
	return nil
}
```

---

## 7. Go Microservice Project Structure

```
ecommerce/
├── services/
│   ├── user-service/
│   │   ├── cmd/
│   │   │   └── main.go
│   │   ├── internal/
│   │   │   ├── domain/
│   │   │   │   ├── user.go
│   │   │   │   └── repository.go
│   │   │   ├── application/
│   │   │   │   ├── create_user.go
│   │   │   │   └── get_user.go
│   │   │   ├── infrastructure/
│   │   │   │   ├── postgres_repository.go
│   │   │   │   └── redis_cache.go
│   │   │   └── transport/
│   │   │       ├── http_handler.go
│   │   │       └── grpc_handler.go
│   │   ├── Dockerfile
│   │   ├── go.mod
│   │   └── go.sum
│   ├── order-service/
│   │   └── (similar structure)
│   └── catalog-service/
│       └── (similar structure)
├── shared/
│   ├── events/
│   │   └── types.go
│   ├── middleware/
│   │   └── auth.go
│   └── proto/
│       └── user.proto
├── docker-compose.yml
└── Makefile
```

```go
// services/user-service/cmd/main.go
package main

import (
	"context"
	"log"
	"os"

	"github.com/example/user-service/internal/application"
	"github.com/example/user-service/internal/infrastructure"
	"github.com/example/user-service/internal/transport"
)

func main() {
	// Load configuration
	cfg, err := LoadConfig()
	if err != nil {
		log.Fatalf("Failed to load config: %v", err)
	}

	// Initialize dependencies
	db, err := infrastructure.NewPostgresDB(cfg.DBConnectionString())
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer db.Close()

	// Run migrations
	if err := db.Migrate(); err != nil {
		log.Fatalf("Failed to run migrations: %v", err)
	}

	// Initialize repositories
	userRepo := infrastructure.NewPostgresUserRepository(db)

	// Initialize use cases
	createUser := application.NewCreateUserUseCase(userRepo)
	getUser := application.NewGetUserUseCase(userRepo)

	// Initialize HTTP handler
	handler := transport.NewHTTPHandler(createUser, getUser)

	// Start server
	port := os.Getenv("PORT")
	if port == "" {
		port = "8081"
	}

	server := NewServer(port, handler.Router())
	ctx := context.Background()
	_ = ctx

	if err := server.Run(); err != nil {
		log.Fatalf("Server failed: %v", err)
	}
}
```

```go
// services/user-service/internal/application/create_user.go
package application

import (
	"context"
	"errors"

	"github.com/example/user-service/internal/domain"
)

type CreateUserInput struct {
	Name  string
	Email string
}

type CreateUserOutput struct {
	User *domain.User
}

type CreateUserUseCase struct {
	repo domain.UserRepository
}

func NewCreateUserUseCase(repo domain.UserRepository) *CreateUserUseCase {
	return &CreateUserUseCase{repo: repo}
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*CreateUserOutput, error) {
	// Validate input
	if input.Name == "" {
		return nil, errors.New("name is required")
	}
	if input.Email == "" {
		return nil, errors.New("email is required")
	}

	// Check if email already exists
	existing, _ := uc.repo.FindByEmail(ctx, input.Email)
	if existing != nil {
		return nil, domain.ErrEmailAlreadyExists
	}

	// Create user
	user := domain.NewUser(input.Name, input.Email)

	// Save to repository
	if err := uc.repo.Save(ctx, user); err != nil {
		return nil, err
	}

	return &CreateUserOutput{User: user}, nil
}
```

```go
// services/user-service/internal/domain/user.go
package domain

import (
	"context"
	"errors"
	"regexp"
	"time"
)

var ErrEmailAlreadyExists = errors.New("email already exists")
var ErrUserNotFound = errors.New("user not found")

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

type User struct {
	ID        string
	Name      string
	Email     string
	CreatedAt time.Time
	UpdatedAt time.Time
}

func NewUser(name, email string) *User {
	now := time.Now()
	return &User{
		Name:      name,
		Email:     email,
		CreatedAt: now,
		UpdatedAt: now,
	}
}

func (u *User) Validate() error {
	if u.Name == "" {
		return errors.New("name cannot be empty")
	}
	if !emailRegex.MatchString(u.Email) {
		return errors.New("invalid email format")
	}
	return nil
}

type UserRepository interface {
	FindByID(ctx context.Context, id string) (*User, error)
	FindByEmail(ctx context.Context, email string) (*User, error)
	Save(ctx context.Context, user *User) error
	Update(ctx context.Context, user *User) error
	Delete(ctx context.Context, id string) error
}
```

---

## 8. Health Check และ Readiness

```go
// health/health.go
package health

import (
	"context"
	"encoding/json"
	"net/http"
	"sync"
	"time"
)

type Status string

const (
	StatusHealthy   Status = "healthy"
	StatusUnhealthy Status = "unhealthy"
	StatusDegraded  Status = "degraded"
)

type Check struct {
	Name    string
	Checker func(ctx context.Context) error
}

type CheckResult struct {
	Name    string        `json:"name"`
	Status  Status        `json:"status"`
	Message string        `json:"message,omitempty"`
	Latency time.Duration `json:"latency_ms"`
}

type HealthResponse struct {
	Status  Status                 `json:"status"`
	Checks  []CheckResult          `json:"checks"`
	Version string                 `json:"version"`
	Info    map[string]interface{} `json:"info,omitempty"`
}

type HealthHandler struct {
	checks  []Check
	version string
}

func NewHealthHandler(version string) *HealthHandler {
	return &HealthHandler{version: version}
}

func (h *HealthHandler) AddCheck(name string, checker func(ctx context.Context) error) {
	h.checks = append(h.checks, Check{Name: name, Checker: checker})
}

func (h *HealthHandler) LivenessHandler(w http.ResponseWriter, r *http.Request) {
	// Liveness: service ยังทำงานอยู่ไหม
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "alive"})
}

func (h *HealthHandler) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
	// Readiness: service พร้อมรับ traffic ไหม
	ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
	defer cancel()

	var (
		wg      sync.WaitGroup
		mu      sync.Mutex
		results []CheckResult
		overall = StatusHealthy
	)

	for _, check := range h.checks {
		wg.Add(1)
		go func(c Check) {
			defer wg.Done()
			start := time.Now()
			err := c.Checker(ctx)
			latency := time.Since(start)

			result := CheckResult{
				Name:    c.Name,
				Latency: latency / time.Millisecond,
			}

			if err != nil {
				result.Status = StatusUnhealthy
				result.Message = err.Error()
				mu.Lock()
				overall = StatusUnhealthy
				mu.Unlock()
			} else {
				result.Status = StatusHealthy
			}

			mu.Lock()
			results = append(results, result)
			mu.Unlock()
		}(check)
	}

	wg.Wait()

	resp := HealthResponse{
		Status:  overall,
		Checks:  results,
		Version: h.version,
	}

	w.Header().Set("Content-Type", "application/json")
	if overall == StatusUnhealthy {
		w.WriteHeader(http.StatusServiceUnavailable)
	}
	json.NewEncoder(w).Encode(resp)
}

// ตัวอย่างการตรวจสอบ database
func DatabaseCheck(db interface{ PingContext(ctx context.Context) error }) func(context.Context) error {
	return func(ctx context.Context) error {
		return db.PingContext(ctx)
	}
}
```

---

## 9. Service Configuration with Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  user-service:
    build:
      context: ./services/user-service
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
    environment:
      - PORT=8081
      - DB_HOST=user-db
      - DB_PORT=5432
      - DB_NAME=userdb
      - DB_USER=postgres
      - DB_PASSWORD=secret
      - REDIS_ADDR=redis:6379
    depends_on:
      - user-db
      - redis
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8081/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  order-service:
    build:
      context: ./services/order-service
      dockerfile: Dockerfile
    ports:
      - "8082:8082"
    environment:
      - PORT=8082
      - DB_HOST=order-db
      - DB_PORT=5432
      - DB_NAME=orderdb
      - DB_USER=postgres
      - DB_PASSWORD=secret
      - USER_SERVICE_URL=http://user-service:8081
    depends_on:
      - order-db
      - user-service

  user-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=userdb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=secret
    volumes:
      - user-db-data:/var/lib/postgresql/data

  order-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=orderdb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=secret
    volumes:
      - order-db-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  user-db-data:
  order-db-data:
```

---

## 10. Dockerfile สำหรับ Go Microservice

```dockerfile
# Multi-stage build
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Download dependencies first (cache layer)
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
    -o service ./cmd/main.go

# Final stage - minimal image
FROM alpine:3.18

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Install CA certificates
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/service .

# Use non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:${PORT:-8080}/health || exit 1

EXPOSE 8080

CMD ["./service"]
```

---

## Workshop: สร้าง Simple E-commerce Microservices

```go
// workshop/ecommerce/product-service/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"sync"
	"syscall"
	"time"

	"github.com/gorilla/mux"
)

// Domain
type Product struct {
	ID          string    `json:"id"`
	Name        string    `json:"name"`
	Description string    `json:"description"`
	Price       float64   `json:"price"`
	Stock       int       `json:"stock"`
	CreatedAt   time.Time `json:"created_at"`
}

// Repository (in-memory สำหรับ demo)
type ProductRepository struct {
	mu       sync.RWMutex
	products map[string]*Product
	counter  int
}

func NewProductRepository() *ProductRepository {
	repo := &ProductRepository{
		products: make(map[string]*Product),
	}
	// seed data
	repo.products["p001"] = &Product{
		ID: "p001", Name: "MacBook Pro",
		Price: 59900, Stock: 5, CreatedAt: time.Now(),
	}
	repo.products["p002"] = &Product{
		ID: "p002", Name: "iPhone 15",
		Price: 32900, Stock: 20, CreatedAt: time.Now(),
	}
	return repo
}

func (r *ProductRepository) FindAll() []*Product {
	r.mu.RLock()
	defer r.mu.RUnlock()
	var products []*Product
	for _, p := range r.products {
		products = append(products, p)
	}
	return products
}

func (r *ProductRepository) FindByID(id string) (*Product, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	p, ok := r.products[id]
	return p, ok
}

func (r *ProductRepository) Save(p *Product) *Product {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.counter++
	p.ID = fmt.Sprintf("p%03d", r.counter)
	p.CreatedAt = time.Now()
	r.products[p.ID] = p
	return p
}

// HTTP Handler
type ProductHandler struct {
	repo *ProductRepository
}

func NewProductHandler(repo *ProductRepository) *ProductHandler {
	return &ProductHandler{repo: repo}
}

func (h *ProductHandler) ListProducts(w http.ResponseWriter, r *http.Request) {
	products := h.repo.FindAll()
	respondJSON(w, http.StatusOK, products)
}

func (h *ProductHandler) GetProduct(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]
	product, ok := h.repo.FindByID(id)
	if !ok {
		respondError(w, http.StatusNotFound, "product not found")
		return
	}
	respondJSON(w, http.StatusOK, product)
}

func (h *ProductHandler) CreateProduct(w http.ResponseWriter, r *http.Request) {
	var p Product
	if err := json.NewDecoder(r.Body).Decode(&p); err != nil {
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}
	if p.Name == "" || p.Price <= 0 {
		respondError(w, http.StatusBadRequest, "name and price are required")
		return
	}
	created := h.repo.Save(&p)
	respondJSON(w, http.StatusCreated, created)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
	respondJSON(w, status, map[string]string{"error": message})
}

func main() {
	repo := NewProductRepository()
	handler := NewProductHandler(repo)

	r := mux.NewRouter()
	r.Use(loggingMiddleware)
	r.Use(recoveryMiddleware)

	// Routes
	r.HandleFunc("/health", healthHandler).Methods("GET")
	r.HandleFunc("/products", handler.ListProducts).Methods("GET")
	r.HandleFunc("/products/{id}", handler.GetProduct).Methods("GET")
	r.HandleFunc("/products", handler.CreateProduct).Methods("POST")

	port := os.Getenv("PORT")
	if port == "" {
		port = "8083"
	}

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      r,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
	}

	// Graceful shutdown
	go func() {
		log.Printf("Product Service starting on port %s", port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	srv.Shutdown(ctx)
	log.Println("Product Service stopped")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	respondJSON(w, http.StatusOK, map[string]string{
		"status":  "healthy",
		"service": "product-service",
		"time":    time.Now().Format(time.RFC3339),
	})
}

func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
	})
}

func recoveryMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("panic: %v", err)
				respondError(w, http.StatusInternalServerError, "internal server error")
			}
		}()
		next.ServeHTTP(w, r)
	})
}

// suppress unused import warning
var _ = strconv.Itoa
```

---

## สรุป

| หัวข้อ | สิ่งที่ควรจำ |
|--------|------------|
| Monolith vs Microservices | Microservices เหมาะกับ large team, complex domain |
| Decomposition | แบ่งตาม Business Capability หรือ Subdomain |
| Communication | Sync (HTTP/gRPC) vs Async (Message Queue) |
| Data | Database per Service, ไม่ share database |
| 12-Factor | Config จาก env, stateless, graceful shutdown |
| Project Structure | cmd/, internal/(domain/application/infrastructure/transport) |

---

**ต่อไป**: Part 52 - Service Discovery
