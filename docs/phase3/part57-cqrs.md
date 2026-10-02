# Part 57: CQRS Pattern

## เป้าหมายการเรียนรู้
- เข้าใจ CQRS pattern
- Command side vs Query side
- Command handlers
- Query handlers
- Event store สำหรับ CQRS
- Read model projection
- Go implementation

---

## 1. CQRS Overview

CQRS (Command Query Responsibility Segregation) แยก **การอ่านข้อมูล** (Queries) ออกจาก **การเขียนข้อมูล** (Commands)

```
Command Side                    Query Side
-----------                    ----------
User --> Command --> Handler    User --> Query --> Handler
              |                                      |
              v                                      v
         Write DB                              Read DB (Optimized)
              |
              v
          Events --> Projections --> Read DB
```

### ประโยชน์ของ CQRS
1. **Scale independently**: Read และ Write scale แยกกัน
2. **Optimize separately**: Read model optimize สำหรับ query pattern
3. **Clear separation of concerns**: business logic แยกชัดเจน
4. **Audit trail**: Commands เก็บ intention ของ user

---

## 2. Commands

```go
// cqrs/command/commands.go
package command

import (
	"time"
)

// Command interface - ทุก command ต้อง implement
type Command interface {
	CommandType() string
	AggregateID() string
}

// Base command
type BaseCommand struct {
	ID          string    `json:"id"`
	Type        string    `json:"type"`
	AggregateID string    `json:"aggregate_id"`
	IssuedAt    time.Time `json:"issued_at"`
	IssuedBy    string    `json:"issued_by"`
}

func (c BaseCommand) CommandType() string  { return c.Type }
func (c BaseCommand) AggregateID() string  { return c.AggregateID }

// Order Commands
type CreateOrderCommand struct {
	BaseCommand
	CustomerID string      `json:"customer_id"`
	Items      []OrderItem `json:"items"`
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type ConfirmOrderCommand struct {
	BaseCommand
	OrderID string `json:"order_id"`
}

type CancelOrderCommand struct {
	BaseCommand
	OrderID string `json:"order_id"`
	Reason  string `json:"reason"`
}

type ShipOrderCommand struct {
	BaseCommand
	OrderID        string `json:"order_id"`
	TrackingNumber string `json:"tracking_number"`
	Carrier        string `json:"carrier"`
}

// Product Commands
type CreateProductCommand struct {
	BaseCommand
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	Stock       int     `json:"stock"`
}

type UpdatePriceCommand struct {
	BaseCommand
	ProductID string  `json:"product_id"`
	NewPrice  float64 `json:"new_price"`
}

type AddStockCommand struct {
	BaseCommand
	ProductID string `json:"product_id"`
	Quantity  int    `json:"quantity"`
}

// Command factory
func NewCreateOrder(customerID string, items []OrderItem, issuedBy string) CreateOrderCommand {
	return CreateOrderCommand{
		BaseCommand: BaseCommand{
			ID:       generateID(),
			Type:     "CreateOrder",
			IssuedAt: time.Now(),
			IssuedBy: issuedBy,
		},
		CustomerID: customerID,
		Items:      items,
	}
}

func generateID() string {
	return fmt.Sprintf("%d", time.Now().UnixNano())
}
```

---

## 3. Command Handlers

```go
// cqrs/command/handlers.go
package command

import (
	"context"
	"errors"
	"fmt"
	"log"
	"time"
)

// CommandHandler interface
type CommandHandler interface {
	Handle(ctx context.Context, cmd Command) error
}

// CommandBus routes commands to handlers
type CommandBus struct {
	handlers map[string]CommandHandler
}

func NewCommandBus() *CommandBus {
	return &CommandBus{
		handlers: make(map[string]CommandHandler),
	}
}

func (b *CommandBus) Register(cmdType string, handler CommandHandler) {
	b.handlers[cmdType] = handler
}

func (b *CommandBus) Dispatch(ctx context.Context, cmd Command) error {
	handler, ok := b.handlers[cmd.CommandType()]
	if !ok {
		return fmt.Errorf("no handler registered for command: %s", cmd.CommandType())
	}

	log.Printf("Dispatching command: %s (aggregate: %s)", cmd.CommandType(), cmd.AggregateID())
	return handler.Handle(ctx, cmd)
}

// Command validation middleware
type ValidationMiddleware struct {
	inner     CommandHandler
	validator CommandValidator
}

type CommandValidator interface {
	Validate(cmd Command) error
}

func WithValidation(handler CommandHandler, validator CommandValidator) CommandHandler {
	return &ValidationMiddleware{inner: handler, validator: validator}
}

func (m *ValidationMiddleware) Handle(ctx context.Context, cmd Command) error {
	if err := m.validator.Validate(cmd); err != nil {
		return fmt.Errorf("validation error: %w", err)
	}
	return m.inner.Handle(ctx, cmd)
}

// Logging middleware
type LoggingMiddleware struct {
	inner CommandHandler
}

func WithLogging(handler CommandHandler) CommandHandler {
	return &LoggingMiddleware{inner: handler}
}

func (m *LoggingMiddleware) Handle(ctx context.Context, cmd Command) error {
	start := time.Now()
	log.Printf("Handling command: %s", cmd.CommandType())
	
	err := m.inner.Handle(ctx, cmd)
	
	if err != nil {
		log.Printf("Command %s failed: %v (duration: %v)", cmd.CommandType(), err, time.Since(start))
	} else {
		log.Printf("Command %s succeeded (duration: %v)", cmd.CommandType(), time.Since(start))
	}
	return err
}

// Order command handler
type OrderCommandHandler struct {
	orderRepo OrderRepository
	eventBus  EventPublisher
}

type OrderRepository interface {
	GetByID(ctx context.Context, id string) (*Order, error)
	Save(ctx context.Context, order *Order) error
}

type EventPublisher interface {
	Publish(ctx context.Context, eventType string, data interface{}) error
}

type Order struct {
	ID         string
	CustomerID string
	Items      []OrderItem
	Status     string
	Total      float64
	CreatedAt  time.Time
	UpdatedAt  time.Time
	Events     []interface{}
}

func (o *Order) CalculateTotal() float64 {
	total := 0.0
	for _, item := range o.Items {
		total += item.Price * float64(item.Quantity)
	}
	return total
}

func NewOrderCommandHandler(repo OrderRepository, eventBus EventPublisher) *OrderCommandHandler {
	return &OrderCommandHandler{orderRepo: repo, eventBus: eventBus}
}

func (h *OrderCommandHandler) Handle(ctx context.Context, cmd Command) error {
	switch c := cmd.(type) {
	case CreateOrderCommand:
		return h.handleCreateOrder(ctx, c)
	case ConfirmOrderCommand:
		return h.handleConfirmOrder(ctx, c)
	case CancelOrderCommand:
		return h.handleCancelOrder(ctx, c)
	case ShipOrderCommand:
		return h.handleShipOrder(ctx, c)
	default:
		return fmt.Errorf("unknown command type: %T", cmd)
	}
}

func (h *OrderCommandHandler) handleCreateOrder(ctx context.Context, cmd CreateOrderCommand) error {
	// Validate
	if cmd.CustomerID == "" {
		return errors.New("customer_id is required")
	}
	if len(cmd.Items) == 0 {
		return errors.New("at least one item is required")
	}

	// Create order
	order := &Order{
		ID:         generateID(),
		CustomerID: cmd.CustomerID,
		Items:      cmd.Items,
		Status:     "pending",
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}
	order.Total = order.CalculateTotal()

	// Save to repository
	if err := h.orderRepo.Save(ctx, order); err != nil {
		return fmt.Errorf("saving order: %w", err)
	}

	// Publish event
	return h.eventBus.Publish(ctx, "OrderCreated", map[string]interface{}{
		"order_id":    order.ID,
		"customer_id": order.CustomerID,
		"total":       order.Total,
		"created_at":  order.CreatedAt,
	})
}

func (h *OrderCommandHandler) handleConfirmOrder(ctx context.Context, cmd ConfirmOrderCommand) error {
	order, err := h.orderRepo.GetByID(ctx, cmd.OrderID)
	if err != nil {
		return fmt.Errorf("getting order: %w", err)
	}

	if order.Status != "pending" {
		return fmt.Errorf("cannot confirm order in status: %s", order.Status)
	}

	order.Status = "confirmed"
	order.UpdatedAt = time.Now()

	if err := h.orderRepo.Save(ctx, order); err != nil {
		return fmt.Errorf("saving order: %w", err)
	}

	return h.eventBus.Publish(ctx, "OrderConfirmed", map[string]interface{}{
		"order_id":     order.ID,
		"confirmed_at": order.UpdatedAt,
	})
}

func (h *OrderCommandHandler) handleCancelOrder(ctx context.Context, cmd CancelOrderCommand) error {
	order, err := h.orderRepo.GetByID(ctx, cmd.OrderID)
	if err != nil {
		return fmt.Errorf("getting order: %w", err)
	}

	if order.Status == "shipped" || order.Status == "delivered" {
		return fmt.Errorf("cannot cancel order in status: %s", order.Status)
	}

	order.Status = "cancelled"
	order.UpdatedAt = time.Now()

	if err := h.orderRepo.Save(ctx, order); err != nil {
		return fmt.Errorf("saving order: %w", err)
	}

	return h.eventBus.Publish(ctx, "OrderCancelled", map[string]interface{}{
		"order_id":     order.ID,
		"reason":       cmd.Reason,
		"cancelled_at": order.UpdatedAt,
	})
}

func (h *OrderCommandHandler) handleShipOrder(ctx context.Context, cmd ShipOrderCommand) error {
	order, err := h.orderRepo.GetByID(ctx, cmd.OrderID)
	if err != nil {
		return fmt.Errorf("getting order: %w", err)
	}

	if order.Status != "confirmed" {
		return fmt.Errorf("cannot ship order in status: %s", order.Status)
	}

	order.Status = "shipped"
	order.UpdatedAt = time.Now()

	if err := h.orderRepo.Save(ctx, order); err != nil {
		return fmt.Errorf("saving order: %w", err)
	}

	return h.eventBus.Publish(ctx, "OrderShipped", map[string]interface{}{
		"order_id":        order.ID,
		"tracking_number": cmd.TrackingNumber,
		"carrier":         cmd.Carrier,
		"shipped_at":      order.UpdatedAt,
	})
}
```

---

## 4. Queries

```go
// cqrs/query/queries.go
package query

import (
	"time"
)

// Query interface
type Query interface {
	QueryType() string
}

// Order Queries
type GetOrderByIDQuery struct {
	OrderID string `json:"order_id"`
}

func (q GetOrderByIDQuery) QueryType() string { return "GetOrderByID" }

type ListOrdersQuery struct {
	CustomerID string     `json:"customer_id,omitempty"`
	Status     string     `json:"status,omitempty"`
	From       *time.Time `json:"from,omitempty"`
	To         *time.Time `json:"to,omitempty"`
	Page       int        `json:"page"`
	PageSize   int        `json:"page_size"`
}

func (q ListOrdersQuery) QueryType() string { return "ListOrders" }

type GetOrderSummaryQuery struct {
	CustomerID string `json:"customer_id"`
}

func (q GetOrderSummaryQuery) QueryType() string { return "GetOrderSummary" }

// Product Queries
type GetProductByIDQuery struct {
	ProductID string `json:"product_id"`
}

func (q GetProductByIDQuery) QueryType() string { return "GetProductByID" }

type SearchProductsQuery struct {
	Term     string   `json:"term"`
	Category string   `json:"category"`
	MinPrice float64  `json:"min_price,omitempty"`
	MaxPrice float64  `json:"max_price,omitempty"`
	Tags     []string `json:"tags,omitempty"`
	Page     int      `json:"page"`
	PageSize int      `json:"page_size"`
}

func (q SearchProductsQuery) QueryType() string { return "SearchProducts" }
```

---

## 5. Query Handlers

```go
// cqrs/query/handlers.go
package query

import (
	"context"
	"fmt"
	"time"
)

// QueryHandler interface
type QueryHandler interface {
	Handle(ctx context.Context, query Query) (interface{}, error)
}

// QueryBus routes queries to handlers
type QueryBus struct {
	handlers map[string]QueryHandler
}

func NewQueryBus() *QueryBus {
	return &QueryBus{handlers: make(map[string]QueryHandler)}
}

func (b *QueryBus) Register(queryType string, handler QueryHandler) {
	b.handlers[queryType] = handler
}

func (b *QueryBus) Dispatch(ctx context.Context, query Query) (interface{}, error) {
	handler, ok := b.handlers[query.QueryType()]
	if !ok {
		return nil, fmt.Errorf("no handler for query: %s", query.QueryType())
	}
	return handler.Handle(ctx, query)
}

// Read models (Denormalized for fast reads)
type OrderView struct {
	OrderID      string         `json:"order_id"`
	CustomerID   string         `json:"customer_id"`
	CustomerName string         `json:"customer_name"`
	Items        []OrderItemView `json:"items"`
	Total        float64        `json:"total"`
	Status       string         `json:"status"`
	CreatedAt    time.Time      `json:"created_at"`
	UpdatedAt    time.Time      `json:"updated_at"`
}

type OrderItemView struct {
	ProductID   string  `json:"product_id"`
	ProductName string  `json:"product_name"`
	Quantity    int     `json:"quantity"`
	Price       float64 `json:"price"`
	Subtotal    float64 `json:"subtotal"`
}

type OrderSummary struct {
	CustomerID    string  `json:"customer_id"`
	TotalOrders   int     `json:"total_orders"`
	TotalSpent    float64 `json:"total_spent"`
	PendingOrders int     `json:"pending_orders"`
	LastOrderDate *time.Time `json:"last_order_date"`
}

type PagedResult struct {
	Data       interface{} `json:"data"`
	Total      int64       `json:"total"`
	Page       int         `json:"page"`
	PageSize   int         `json:"page_size"`
	TotalPages int         `json:"total_pages"`
}

// OrderReadRepository (read-optimized database)
type OrderReadRepository interface {
	GetByID(ctx context.Context, orderID string) (*OrderView, error)
	List(ctx context.Context, params ListOrdersQuery) (*PagedResult, error)
	GetSummary(ctx context.Context, customerID string) (*OrderSummary, error)
}

// Order Query Handler
type OrderQueryHandler struct {
	repo OrderReadRepository
}

func NewOrderQueryHandler(repo OrderReadRepository) *OrderQueryHandler {
	return &OrderQueryHandler{repo: repo}
}

func (h *OrderQueryHandler) Handle(ctx context.Context, query Query) (interface{}, error) {
	switch q := query.(type) {
	case GetOrderByIDQuery:
		return h.repo.GetByID(ctx, q.OrderID)
	case ListOrdersQuery:
		return h.repo.List(ctx, q)
	case GetOrderSummaryQuery:
		return h.repo.GetSummary(ctx, q.CustomerID)
	default:
		return nil, fmt.Errorf("unknown query type: %T", query)
	}
}

// Cached Query Handler
type CachedOrderQueryHandler struct {
	inner OrderReadRepository
	cache CacheStore
}

type CacheStore interface {
	Get(ctx context.Context, key string) (interface{}, bool)
	Set(ctx context.Context, key string, value interface{}, ttl time.Duration)
}

func NewCachedHandler(inner OrderReadRepository, cache CacheStore) *CachedOrderQueryHandler {
	return &CachedOrderQueryHandler{inner: inner, cache: cache}
}

func (h *CachedOrderQueryHandler) Handle(ctx context.Context, query Query) (interface{}, error) {
	cacheKey := fmt.Sprintf("query:%s:%v", query.QueryType(), query)
	
	if cached, ok := h.cache.Get(ctx, cacheKey); ok {
		return cached, nil
	}

	handler := &OrderQueryHandler{repo: h.inner}
	result, err := handler.Handle(ctx, query)
	if err != nil {
		return nil, err
	}

	h.cache.Set(ctx, cacheKey, result, 30*time.Second)
	return result, nil
}
```

---

## 6. Projections (Read Model Builder)

```go
// cqrs/projection/projector.go
package projection

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"
)

// Projector สร้าง read model จาก events
type Projector interface {
	Project(ctx context.Context, event StoredEvent) error
}

type StoredEvent struct {
	ID        string          `json:"id"`
	Type      string          `json:"type"`
	Data      json.RawMessage `json:"data"`
	CreatedAt time.Time       `json:"created_at"`
}

// OrderProjection สร้าง OrderView จาก events
type OrderProjection struct {
	db OrderViewDB
}

type OrderViewDB interface {
	UpsertOrder(ctx context.Context, view OrderView) error
	UpdateOrderStatus(ctx context.Context, orderID, status string, updatedAt time.Time) error
	IncrementCustomerStats(ctx context.Context, customerID string, amount float64) error
}

type OrderView struct {
	OrderID    string     `json:"order_id"`
	CustomerID string     `json:"customer_id"`
	Items      []ItemView `json:"items"`
	Total      float64    `json:"total"`
	Status     string     `json:"status"`
	CreatedAt  time.Time  `json:"created_at"`
	UpdatedAt  time.Time  `json:"updated_at"`
}

type ItemView struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

func NewOrderProjection(db OrderViewDB) *OrderProjection {
	return &OrderProjection{db: db}
}

func (p *OrderProjection) Project(ctx context.Context, event StoredEvent) error {
	switch event.Type {
	case "OrderCreated":
		return p.onOrderCreated(ctx, event)
	case "OrderConfirmed":
		return p.onOrderConfirmed(ctx, event)
	case "OrderCancelled":
		return p.onOrderCancelled(ctx, event)
	case "OrderShipped":
		return p.onOrderShipped(ctx, event)
	default:
		// Ignore events we don't care about
		return nil
	}
}

func (p *OrderProjection) onOrderCreated(ctx context.Context, event StoredEvent) error {
	var data struct {
		OrderID    string     `json:"order_id"`
		CustomerID string     `json:"customer_id"`
		Items      []ItemView `json:"items"`
		Total      float64    `json:"total"`
		CreatedAt  time.Time  `json:"created_at"`
	}
	if err := json.Unmarshal(event.Data, &data); err != nil {
		return fmt.Errorf("unmarshal OrderCreated: %w", err)
	}

	view := OrderView{
		OrderID:    data.OrderID,
		CustomerID: data.CustomerID,
		Items:      data.Items,
		Total:      data.Total,
		Status:     "pending",
		CreatedAt:  data.CreatedAt,
		UpdatedAt:  data.CreatedAt,
	}

	return p.db.UpsertOrder(ctx, view)
}

func (p *OrderProjection) onOrderConfirmed(ctx context.Context, event StoredEvent) error {
	var data struct {
		OrderID     string    `json:"order_id"`
		ConfirmedAt time.Time `json:"confirmed_at"`
	}
	if err := json.Unmarshal(event.Data, &data); err != nil {
		return fmt.Errorf("unmarshal OrderConfirmed: %w", err)
	}

	return p.db.UpdateOrderStatus(ctx, data.OrderID, "confirmed", data.ConfirmedAt)
}

func (p *OrderProjection) onOrderCancelled(ctx context.Context, event StoredEvent) error {
	var data struct {
		OrderID     string    `json:"order_id"`
		CancelledAt time.Time `json:"cancelled_at"`
	}
	if err := json.Unmarshal(event.Data, &data); err != nil {
		return fmt.Errorf("unmarshal OrderCancelled: %w", err)
	}

	return p.db.UpdateOrderStatus(ctx, data.OrderID, "cancelled", data.CancelledAt)
}

func (p *OrderProjection) onOrderShipped(ctx context.Context, event StoredEvent) error {
	var data struct {
		OrderID   string    `json:"order_id"`
		ShippedAt time.Time `json:"shipped_at"`
	}
	if err := json.Unmarshal(event.Data, &data); err != nil {
		return fmt.Errorf("unmarshal OrderShipped: %w", err)
	}

	return p.db.UpdateOrderStatus(ctx, data.OrderID, "shipped", data.ShippedAt)
}

// ProjectionManager manages multiple projectors
type ProjectionManager struct {
	projectors []Projector
	eventStore EventStore
	position   int64
}

type EventStore interface {
	ReadAll(ctx context.Context, fromPosition int64) ([]StoredEvent, error)
}

func NewProjectionManager(eventStore EventStore, projectors ...Projector) *ProjectionManager {
	return &ProjectionManager{
		projectors: projectors,
		eventStore: eventStore,
	}
}

func (m *ProjectionManager) Start(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
		}

		events, err := m.eventStore.ReadAll(ctx, m.position)
		if err != nil {
			log.Printf("Error reading events: %v", err)
			time.Sleep(time.Second)
			continue
		}

		for _, event := range events {
			for _, projector := range m.projectors {
				if err := projector.Project(ctx, event); err != nil {
					log.Printf("Projection error for event %s: %v", event.ID, err)
				}
			}
			m.position++
		}

		if len(events) == 0 {
			time.Sleep(100 * time.Millisecond)
		}
	}
}
```

---

## 7. Complete CQRS Application

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/gorilla/mux"
)

// Simple in-memory implementations for demo

// In-memory event store
type MemEventStore struct {
	mu     sync.RWMutex
	events []StoredEvent
}

func NewMemEventStore() *MemEventStore {
	return &MemEventStore{}
}

func (s *MemEventStore) Append(event StoredEvent) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.events = append(s.events, event)
}

func (s *MemEventStore) ReadAll(ctx context.Context, from int64) ([]StoredEvent, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if from >= int64(len(s.events)) {
		return nil, nil
	}
	return s.events[from:], nil
}

// In-memory order write repository
type MemOrderWriteRepo struct {
	mu     sync.RWMutex
	orders map[string]*Order
}

func NewMemOrderWriteRepo() *MemOrderWriteRepo {
	return &MemOrderWriteRepo{orders: make(map[string]*Order)}
}

func (r *MemOrderWriteRepo) GetByID(ctx context.Context, id string) (*Order, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	order, ok := r.orders[id]
	if !ok {
		return nil, fmt.Errorf("order not found: %s", id)
	}
	return order, nil
}

func (r *MemOrderWriteRepo) Save(ctx context.Context, order *Order) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.orders[order.ID] = order
	return nil
}

// In-memory order read repository (denormalized)
type MemOrderReadRepo struct {
	mu     sync.RWMutex
	orders map[string]*OrderView
}

func NewMemOrderReadRepo() *MemOrderReadRepo {
	return &MemOrderReadRepo{orders: make(map[string]*OrderView)}
}

func (r *MemOrderReadRepo) Upsert(view *OrderView) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.orders[view.OrderID] = view
}

func (r *MemOrderReadRepo) GetByID(id string) (*OrderView, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	view, ok := r.orders[id]
	return view, ok
}

func (r *MemOrderReadRepo) ListAll() []*OrderView {
	r.mu.RLock()
	defer r.mu.RUnlock()
	var result []*OrderView
	for _, v := range r.orders {
		result = append(result, v)
	}
	return result
}

// Simple event publisher
type MemEventPublisher struct {
	store *MemEventStore
	readRepo *MemOrderReadRepo
}

func NewMemEventPublisher(store *MemEventStore, readRepo *MemOrderReadRepo) *MemEventPublisher {
	return &MemEventPublisher{store: store, readRepo: readRepo}
}

func (p *MemEventPublisher) Publish(ctx context.Context, eventType string, data interface{}) error {
	d, _ := json.Marshal(data)
	event := StoredEvent{
		ID:        fmt.Sprintf("evt-%d", time.Now().UnixNano()),
		Type:      eventType,
		Data:      d,
		CreatedAt: time.Now(),
	}
	p.store.Append(event)

	// Update read model (sync for demo)
	p.updateReadModel(ctx, event)
	return nil
}

func (p *MemEventPublisher) updateReadModel(ctx context.Context, event StoredEvent) {
	switch event.Type {
	case "OrderCreated":
		var data map[string]interface{}
		json.Unmarshal(event.Data, &data)
		
		view := &OrderView{
			OrderID:    fmt.Sprint(data["order_id"]),
			CustomerID: fmt.Sprint(data["customer_id"]),
			Total:      data["total"].(float64),
			Status:     "pending",
			CreatedAt:  time.Now(),
			UpdatedAt:  time.Now(),
		}
		p.readRepo.Upsert(view)
		log.Printf("Read model updated: OrderCreated %s", view.OrderID)

	case "OrderConfirmed":
		var data map[string]interface{}
		json.Unmarshal(event.Data, &data)
		orderID := fmt.Sprint(data["order_id"])
		if view, ok := p.readRepo.GetByID(orderID); ok {
			view.Status = "confirmed"
			view.UpdatedAt = time.Now()
			p.readRepo.Upsert(view)
		}
	}
}

// HTTP Handlers
type CQRSAPI struct {
	cmdBus   *CommandBus
	readRepo *MemOrderReadRepo
}

func NewCQRSAPI(cmdBus *CommandBus, readRepo *MemOrderReadRepo) *CQRSAPI {
	return &CQRSAPI{cmdBus: cmdBus, readRepo: readRepo}
}

func (api *CQRSAPI) CreateOrder(w http.ResponseWriter, r *http.Request) {
	var req struct {
		CustomerID string          `json:"customer_id"`
		Items      []OrderItem     `json:"items"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	cmd := CreateOrderCommand{
		BaseCommand: BaseCommand{
			ID:       fmt.Sprintf("cmd-%d", time.Now().UnixNano()),
			Type:     "CreateOrder",
			IssuedAt: time.Now(),
		},
		CustomerID: req.CustomerID,
		Items:      req.Items,
	}

	if err := api.cmdBus.Dispatch(r.Context(), cmd); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusAccepted)
	json.NewEncoder(w).Encode(map[string]string{"status": "accepted"})
}

func (api *CQRSAPI) GetOrder(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]
	view, ok := api.readRepo.GetByID(id)
	if !ok {
		http.Error(w, "order not found", http.StatusNotFound)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(view)
}

func (api *CQRSAPI) ListOrders(w http.ResponseWriter, r *http.Request) {
	orders := api.readRepo.ListAll()
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(orders)
}

func main() {
	// Setup
	eventStore := NewMemEventStore()
	readRepo := NewMemOrderReadRepo()
	writeRepo := NewMemOrderWriteRepo()
	publisher := NewMemEventPublisher(eventStore, readRepo)

	// Create handler
	orderHandler := NewOrderCommandHandler(writeRepo, publisher)
	cmdBus := NewCommandBus()
	cmdBus.Register("CreateOrder", WithLogging(orderHandler))
	cmdBus.Register("ConfirmOrder", WithLogging(orderHandler))
	cmdBus.Register("CancelOrder", WithLogging(orderHandler))

	// Create API
	api := NewCQRSAPI(cmdBus, readRepo)

	// Routes
	r := mux.NewRouter()
	r.HandleFunc("/orders", api.CreateOrder).Methods("POST")
	r.HandleFunc("/orders", api.ListOrders).Methods("GET")
	r.HandleFunc("/orders/{id}", api.GetOrder).Methods("GET")

	log.Println("CQRS API starting on :8080")

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	_ = ctx

	log.Fatal(http.ListenAndServe(":8080", r))
}

// Shared types between packages (for demo - normally in separate packages)
type StoredEvent struct {
	ID        string          `json:"id"`
	Type      string          `json:"type"`
	Data      []byte          `json:"data"`
	CreatedAt time.Time       `json:"created_at"`
}

type OrderView struct {
	OrderID    string     `json:"order_id"`
	CustomerID string     `json:"customer_id"`
	Items      []ItemView `json:"items"`
	Total      float64    `json:"total"`
	Status     string     `json:"status"`
	CreatedAt  time.Time  `json:"created_at"`
	UpdatedAt  time.Time  `json:"updated_at"`
}

type ItemView struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type Order struct {
	ID         string
	CustomerID string
	Items      []OrderItem
	Status     string
	Total      float64
	CreatedAt  time.Time
	UpdatedAt  time.Time
}

func (o *Order) CalculateTotal() float64 {
	total := 0.0
	for _, item := range o.Items {
		total += item.Price * float64(item.Quantity)
	}
	return total
}

type BaseCommand struct {
	ID          string    `json:"id"`
	Type        string    `json:"type"`
	AggregateID string    `json:"aggregate_id"`
	IssuedAt    time.Time `json:"issued_at"`
}

func (c BaseCommand) CommandType() string { return c.Type }
func (c BaseCommand) AggregateID_() string { return c.AggregateID }

type CreateOrderCommand struct {
	BaseCommand
	CustomerID string
	Items      []OrderItem
}

type ConfirmOrderCommand struct {
	BaseCommand
	OrderID string
}

type CancelOrderCommand struct {
	BaseCommand
	OrderID string
	Reason  string
}

type Command interface {
	CommandType() string
}

type CommandHandler interface {
	Handle(ctx context.Context, cmd Command) error
}

type LoggingMW struct{ inner CommandHandler }

func WithLogging(h CommandHandler) CommandHandler {
	return &LoggingMW{inner: h}
}

func (m *LoggingMW) Handle(ctx context.Context, cmd Command) error {
	start := time.Now()
	err := m.inner.Handle(ctx, cmd)
	log.Printf("Command %s: %v (%v)", cmd.CommandType(), err, time.Since(start))
	return err
}

type CommandBus struct {
	handlers map[string]CommandHandler
}

func NewCommandBus() *CommandBus {
	return &CommandBus{handlers: make(map[string]CommandHandler)}
}

func (b *CommandBus) Register(t string, h CommandHandler) {
	b.handlers[t] = h
}

func (b *CommandBus) Dispatch(ctx context.Context, cmd Command) error {
	h, ok := b.handlers[cmd.CommandType()]
	if !ok {
		return fmt.Errorf("no handler for %s", cmd.CommandType())
	}
	return h.Handle(ctx, cmd)
}

type EventPublisher interface {
	Publish(ctx context.Context, eventType string, data interface{}) error
}

type OrderRepository interface {
	GetByID(ctx context.Context, id string) (*Order, error)
	Save(ctx context.Context, order *Order) error
}

type OrderCommandHandler struct {
	orderRepo OrderRepository
	eventBus  EventPublisher
}

func NewOrderCommandHandler(repo OrderRepository, ep EventPublisher) *OrderCommandHandler {
	return &OrderCommandHandler{orderRepo: repo, eventBus: ep}
}

func (h *OrderCommandHandler) Handle(ctx context.Context, cmd Command) error {
	switch c := cmd.(type) {
	case CreateOrderCommand:
		order := &Order{
			ID:         fmt.Sprintf("ord-%d", time.Now().UnixNano()),
			CustomerID: c.CustomerID,
			Items:      c.Items,
			Status:     "pending",
			CreatedAt:  time.Now(),
			UpdatedAt:  time.Now(),
		}
		order.Total = order.CalculateTotal()
		h.orderRepo.Save(ctx, order)
		return h.eventBus.Publish(ctx, "OrderCreated", map[string]interface{}{
			"order_id":    order.ID,
			"customer_id": order.CustomerID,
			"total":       order.Total,
		})
	case ConfirmOrderCommand:
		order, err := h.orderRepo.GetByID(ctx, c.OrderID)
		if err != nil {
			return err
		}
		order.Status = "confirmed"
		order.UpdatedAt = time.Now()
		h.orderRepo.Save(ctx, order)
		return h.eventBus.Publish(ctx, "OrderConfirmed", map[string]interface{}{
			"order_id": order.ID,
		})
	default:
		return fmt.Errorf("unknown command: %T", cmd)
	}
}
```

---

## สรุป

| Component | Command Side | Query Side |
|-----------|-------------|-----------|
| Purpose | เปลี่ยนสถานะของระบบ | อ่านสถานะของระบบ |
| Database | Write-optimized | Read-optimized (denormalized) |
| Consistency | Strong | Eventual |
| Complexity | Higher | Lower |
| Scalability | Write scale | Read scale (typically higher) |

---

**ต่อไป**: Part 58 - Event Sourcing
