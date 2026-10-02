# Part 56: Event-Driven Architecture

## เป้าหมายการเรียนรู้
- เข้าใจ Event-driven architecture
- Domain events
- Event bus implementation
- Saga pattern
- Outbox pattern
- Idempotency

---

## 1. Event-Driven Architecture Overview

Event-Driven Architecture (EDA) คือ architecture pattern ที่ components สื่อสารกันผ่าน events แทนที่จะ call กันโดยตรง

```
Producer --> Event Bus --> Consumer 1
                      --> Consumer 2
                      --> Consumer 3
```

### ประเภทของ Events
1. **Domain Events**: สิ่งที่เกิดขึ้นใน business domain เช่น OrderPlaced, PaymentReceived
2. **Integration Events**: events ที่ส่งระหว่าง services
3. **System Events**: events เกี่ยวกับ infrastructure เช่น ServiceStarted, HealthCheckFailed

```go
// events/types.go
package events

import (
	"encoding/json"
	"time"

	"github.com/google/uuid"
)

// BaseEvent contains common fields for all events
type BaseEvent struct {
	ID            string                 `json:"id"`
	Type          string                 `json:"type"`
	Version       int                    `json:"version"`
	AggregateID   string                 `json:"aggregate_id"`
	AggregateType string                 `json:"aggregate_type"`
	OccurredAt    time.Time              `json:"occurred_at"`
	Metadata      map[string]interface{} `json:"metadata,omitempty"`
}

func NewBaseEvent(eventType, aggregateID, aggregateType string) BaseEvent {
	return BaseEvent{
		ID:            uuid.New().String(),
		Type:          eventType,
		Version:       1,
		AggregateID:   aggregateID,
		AggregateType: aggregateType,
		OccurredAt:    time.Now(),
	}
}

// DomainEvent interface
type DomainEvent interface {
	EventID() string
	EventType() string
	EventVersion() int
	AggregateID() string
	OccurredAt() time.Time
}

func (e BaseEvent) EventID() string      { return e.ID }
func (e BaseEvent) EventType() string    { return e.Type }
func (e BaseEvent) EventVersion() int    { return e.Version }
func (e BaseEvent) AggregateID() string  { return e.AggregateID }
func (e BaseEvent) OccurredAt() time.Time { return e.OccurredAt }

// Order domain events
type OrderPlaced struct {
	BaseEvent
	CustomerID string      `json:"customer_id"`
	Items      []OrderItem `json:"items"`
	TotalAmount float64    `json:"total_amount"`
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type OrderConfirmed struct {
	BaseEvent
	ConfirmedAt time.Time `json:"confirmed_at"`
}

type OrderCancelled struct {
	BaseEvent
	Reason      string    `json:"reason"`
	CancelledAt time.Time `json:"cancelled_at"`
}

type OrderShipped struct {
	BaseEvent
	TrackingNumber string    `json:"tracking_number"`
	Carrier        string    `json:"carrier"`
	ShippedAt      time.Time `json:"shipped_at"`
}

// Payment domain events
type PaymentInitiated struct {
	BaseEvent
	OrderID    string  `json:"order_id"`
	Amount     float64 `json:"amount"`
	Currency   string  `json:"currency"`
	Method     string  `json:"method"`
}

type PaymentCompleted struct {
	BaseEvent
	OrderID         string    `json:"order_id"`
	TransactionID   string    `json:"transaction_id"`
	CompletedAt     time.Time `json:"completed_at"`
}

type PaymentFailed struct {
	BaseEvent
	OrderID string `json:"order_id"`
	Reason  string `json:"reason"`
}

// Serialize event to JSON
func MarshalEvent(event DomainEvent) ([]byte, error) {
	return json.Marshal(event)
}

// Message envelope for event bus
type EventMessage struct {
	Event     json.RawMessage `json:"event"`
	EventType string          `json:"event_type"`
	Topic     string          `json:"topic"`
}
```

---

## 2. Event Bus Implementation

```go
// eventbus/bus.go
package eventbus

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"reflect"
	"sync"
)

// Handler function type
type Handler func(ctx context.Context, event interface{}) error

// EventBus interface
type EventBus interface {
	Publish(ctx context.Context, event interface{}) error
	Subscribe(eventType string, handler Handler) error
	Unsubscribe(eventType string, handler Handler)
}

// Subscription represents a handler subscription
type Subscription struct {
	eventType string
	handler   Handler
	id        string
}

// InMemoryEventBus - in-process event bus
type InMemoryEventBus struct {
	mu           sync.RWMutex
	handlers     map[string][]Subscription
	errorHandler func(err error)
}

func NewInMemoryEventBus() *InMemoryEventBus {
	return &InMemoryEventBus{
		handlers: make(map[string][]Subscription),
		errorHandler: func(err error) {
			log.Printf("Event handler error: %v", err)
		},
	}
}

func (b *InMemoryEventBus) WithErrorHandler(h func(err error)) *InMemoryEventBus {
	b.errorHandler = h
	return b
}

func (b *InMemoryEventBus) Subscribe(eventType string, handler Handler) error {
	b.mu.Lock()
	defer b.mu.Unlock()

	sub := Subscription{
		eventType: eventType,
		handler:   handler,
		id:        fmt.Sprintf("%p", handler),
	}

	b.handlers[eventType] = append(b.handlers[eventType], sub)
	return nil
}

func (b *InMemoryEventBus) Unsubscribe(eventType string, handler Handler) {
	b.mu.Lock()
	defer b.mu.Unlock()

	handlerID := fmt.Sprintf("%p", handler)
	subs := b.handlers[eventType]
	for i, sub := range subs {
		if sub.id == handlerID {
			b.handlers[eventType] = append(subs[:i], subs[i+1:]...)
			return
		}
	}
}

func (b *InMemoryEventBus) Publish(ctx context.Context, event interface{}) error {
	eventType := reflect.TypeOf(event).Name()
	if reflect.TypeOf(event).Kind() == reflect.Ptr {
		eventType = reflect.TypeOf(event).Elem().Name()
	}

	b.mu.RLock()
	subs := make([]Subscription, len(b.handlers[eventType]))
	copy(subs, b.handlers[eventType])
	b.mu.RUnlock()

	for _, sub := range subs {
		go func(s Subscription) {
			if err := s.handler(ctx, event); err != nil {
				b.errorHandler(fmt.Errorf("handler for %s: %w", s.eventType, err))
			}
		}(sub)
	}

	return nil
}

// TypedEventBus ใช้ type-safe handler
type TypedEventBus struct {
	bus *InMemoryEventBus
}

func NewTypedEventBus() *TypedEventBus {
	return &TypedEventBus{bus: NewInMemoryEventBus()}
}

func SubscribeTyped[T any](bus *TypedEventBus, handler func(ctx context.Context, event T) error) {
	eventType := reflect.TypeOf(*new(T)).Name()
	bus.bus.Subscribe(eventType, func(ctx context.Context, raw interface{}) error {
		event, ok := raw.(T)
		if !ok {
			return fmt.Errorf("type assertion failed: expected %T", *new(T))
		}
		return handler(ctx, event)
	})
}

func PublishTyped[T any](ctx context.Context, bus *TypedEventBus, event T) error {
	return bus.bus.Publish(ctx, event)
}
```

---

## 3. Event Store

```go
// eventstore/store.go
package eventstore

import (
	"context"
	"encoding/json"
	"fmt"
	"sync"
	"time"
)

// StoredEvent เก็บ event ใน store
type StoredEvent struct {
	ID            string          `json:"id"`
	StreamID      string          `json:"stream_id"`
	StreamVersion int64           `json:"stream_version"`
	Type          string          `json:"type"`
	Data          json.RawMessage `json:"data"`
	Metadata      json.RawMessage `json:"metadata,omitempty"`
	CreatedAt     time.Time       `json:"created_at"`
}

// EventStore interface
type EventStore interface {
	AppendToStream(ctx context.Context, streamID string, expectedVersion int64, events []EventData) error
	ReadStream(ctx context.Context, streamID string, fromVersion int64) ([]StoredEvent, error)
	ReadAllStreams(ctx context.Context, fromPosition int64) ([]StoredEvent, error)
}

type EventData struct {
	Type     string
	Data     interface{}
	Metadata map[string]interface{}
}

// InMemoryEventStore
type InMemoryEventStore struct {
	mu      sync.RWMutex
	streams map[string][]StoredEvent
	all     []StoredEvent
	nextPos int64
}

func NewInMemoryEventStore() *InMemoryEventStore {
	return &InMemoryEventStore{
		streams: make(map[string][]StoredEvent),
	}
}

func (s *InMemoryEventStore) AppendToStream(ctx context.Context, streamID string, expectedVersion int64, events []EventData) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	stream := s.streams[streamID]
	currentVersion := int64(len(stream))

	// Optimistic concurrency check
	if expectedVersion != -1 && currentVersion != expectedVersion {
		return fmt.Errorf("concurrency conflict: expected version %d, got %d",
			expectedVersion, currentVersion)
	}

	for i, ed := range events {
		data, err := json.Marshal(ed.Data)
		if err != nil {
			return fmt.Errorf("marshaling event data: %w", err)
		}

		var metadata json.RawMessage
		if ed.Metadata != nil {
			metadata, _ = json.Marshal(ed.Metadata)
		}

		stored := StoredEvent{
			ID:            fmt.Sprintf("%s-%d", streamID, currentVersion+int64(i)+1),
			StreamID:      streamID,
			StreamVersion: currentVersion + int64(i) + 1,
			Type:          ed.Type,
			Data:          data,
			Metadata:      metadata,
			CreatedAt:     time.Now(),
		}

		s.streams[streamID] = append(s.streams[streamID], stored)
		s.all = append(s.all, stored)
		s.nextPos++
	}

	return nil
}

func (s *InMemoryEventStore) ReadStream(ctx context.Context, streamID string, fromVersion int64) ([]StoredEvent, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	stream, ok := s.streams[streamID]
	if !ok {
		return nil, fmt.Errorf("stream not found: %s", streamID)
	}

	var result []StoredEvent
	for _, e := range stream {
		if e.StreamVersion >= fromVersion {
			result = append(result, e)
		}
	}
	return result, nil
}

func (s *InMemoryEventStore) ReadAllStreams(ctx context.Context, fromPosition int64) ([]StoredEvent, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	if fromPosition >= int64(len(s.all)) {
		return nil, nil
	}
	return s.all[fromPosition:], nil
}
```

---

## 4. Saga Pattern

```go
// saga/choreography.go
package saga

import (
	"context"
	"fmt"
	"log"
)

// Choreography-based Saga
// แต่ละ service react ต่อ events และ publish events ของตัวเอง

type OrderSagaEvent int

const (
	EventOrderCreated OrderSagaEvent = iota
	EventPaymentProcessed
	EventPaymentFailed
	EventInventoryReserved
	EventInventoryFailed
	EventOrderCompleted
	EventOrderFailed
)

// Order Saga State
type OrderSagaState struct {
	OrderID    string
	CustomerID string
	Amount     float64
	Status     string
	Steps      []SagaStep
}

type SagaStep struct {
	Name   string
	Status string // pending, completed, failed, compensated
	Error  string
}

// OrderService reacts to payment and inventory events
type OrderService struct {
	eventBus EventPublisher
}

type EventPublisher interface {
	Publish(ctx context.Context, topic string, event interface{}) error
}

func (s *OrderService) HandlePaymentProcessed(ctx context.Context, event PaymentProcessedEvent) error {
	log.Printf("Order %s: Payment processed, confirming order", event.OrderID)
	
	// Update order status
	// In real implementation, update database
	
	// Check if inventory is also reserved
	// If both done, complete the order
	return s.eventBus.Publish(ctx, "order-confirmed", OrderConfirmedEvent{
		OrderID: event.OrderID,
	})
}

func (s *OrderService) HandlePaymentFailed(ctx context.Context, event PaymentFailedEvent) error {
	log.Printf("Order %s: Payment failed, cancelling order", event.OrderID)
	return s.eventBus.Publish(ctx, "order-cancelled", OrderCancelledEvent{
		OrderID: event.OrderID,
		Reason:  "payment_failed: " + event.Reason,
	})
}

type PaymentProcessedEvent struct {
	OrderID       string
	TransactionID string
	Amount        float64
}

type PaymentFailedEvent struct {
	OrderID string
	Reason  string
}

type OrderConfirmedEvent struct {
	OrderID string
}

type OrderCancelledEvent struct {
	OrderID string
	Reason  string
}

// Orchestration-based Saga
// Central coordinator manages the saga

type CreateOrderSaga struct {
	orderID    string
	state      *SagaState
	eventBus   EventPublisher
}

type SagaState struct {
	CurrentStep int
	Completed   bool
	Failed      bool
	Steps       []SagaStepState
}

type SagaStepState struct {
	Name      string
	Status    string
	Error     string
}

func NewCreateOrderSaga(orderID string, bus EventPublisher) *CreateOrderSaga {
	return &CreateOrderSaga{
		orderID:  orderID,
		eventBus: bus,
		state: &SagaState{
			Steps: []SagaStepState{
				{Name: "reserve_inventory", Status: "pending"},
				{Name: "process_payment", Status: "pending"},
				{Name: "confirm_order", Status: "pending"},
			},
		},
	}
}

func (s *CreateOrderSaga) Start(ctx context.Context) error {
	log.Printf("Starting CreateOrder saga for order %s", s.orderID)
	return s.executeStep(ctx, 0)
}

func (s *CreateOrderSaga) executeStep(ctx context.Context, stepIdx int) error {
	if stepIdx >= len(s.state.Steps) {
		s.state.Completed = true
		log.Printf("Saga completed for order %s", s.orderID)
		return nil
	}

	step := &s.state.Steps[stepIdx]
	step.Status = "executing"

	var err error
	switch step.Name {
	case "reserve_inventory":
		err = s.reserveInventory(ctx)
	case "process_payment":
		err = s.processPayment(ctx)
	case "confirm_order":
		err = s.confirmOrder(ctx)
	}

	if err != nil {
		step.Status = "failed"
		step.Error = err.Error()
		s.state.Failed = true
		return s.compensate(ctx, stepIdx-1)
	}

	step.Status = "completed"
	s.state.CurrentStep = stepIdx + 1
	return s.executeStep(ctx, stepIdx+1)
}

func (s *CreateOrderSaga) compensate(ctx context.Context, fromStep int) error {
	log.Printf("Compensating saga for order %s from step %d", s.orderID, fromStep)
	
	for i := fromStep; i >= 0; i-- {
		step := s.state.Steps[i]
		if step.Status != "completed" {
			continue
		}

		switch step.Name {
		case "reserve_inventory":
			if err := s.releaseInventory(ctx); err != nil {
				log.Printf("Compensation failed for %s: %v", step.Name, err)
			} else {
				s.state.Steps[i].Status = "compensated"
			}
		case "process_payment":
			if err := s.refundPayment(ctx); err != nil {
				log.Printf("Compensation failed for %s: %v", step.Name, err)
			} else {
				s.state.Steps[i].Status = "compensated"
			}
		}
	}
	return fmt.Errorf("saga failed and compensated for order %s", s.orderID)
}

func (s *CreateOrderSaga) reserveInventory(ctx context.Context) error {
	log.Printf("Reserving inventory for order %s", s.orderID)
	return s.eventBus.Publish(ctx, "inventory.reserve", map[string]string{
		"order_id": s.orderID,
	})
}

func (s *CreateOrderSaga) processPayment(ctx context.Context) error {
	log.Printf("Processing payment for order %s", s.orderID)
	return s.eventBus.Publish(ctx, "payment.process", map[string]string{
		"order_id": s.orderID,
	})
}

func (s *CreateOrderSaga) confirmOrder(ctx context.Context) error {
	log.Printf("Confirming order %s", s.orderID)
	return nil
}

func (s *CreateOrderSaga) releaseInventory(ctx context.Context) error {
	log.Printf("Releasing inventory for order %s", s.orderID)
	return nil
}

func (s *CreateOrderSaga) refundPayment(ctx context.Context) error {
	log.Printf("Refunding payment for order %s", s.orderID)
	return nil
}
```

---

## 5. Outbox Pattern

```go
// outbox/outbox.go
package outbox

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"log"
	"time"
)

// OutboxEvent เก็บ event ที่รอส่ง
type OutboxEvent struct {
	ID          string          `json:"id"`
	AggregateID string          `json:"aggregate_id"`
	EventType   string          `json:"event_type"`
	Payload     json.RawMessage `json:"payload"`
	Status      string          `json:"status"` // pending, sent, failed
	CreatedAt   time.Time       `json:"created_at"`
	SentAt      *time.Time      `json:"sent_at,omitempty"`
	Attempts    int             `json:"attempts"`
	Error       string          `json:"error,omitempty"`
}

// OutboxRepository เก็บ outbox events ใน database
type OutboxRepository struct {
	db *sql.DB
}

func NewOutboxRepository(db *sql.DB) *OutboxRepository {
	return &OutboxRepository{db: db}
}

func (r *OutboxRepository) CreateTable(ctx context.Context) error {
	_, err := r.db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS outbox_events (
			id VARCHAR(36) PRIMARY KEY,
			aggregate_id VARCHAR(255) NOT NULL,
			event_type VARCHAR(255) NOT NULL,
			payload JSONB NOT NULL,
			status VARCHAR(50) NOT NULL DEFAULT 'pending',
			created_at TIMESTAMP NOT NULL DEFAULT NOW(),
			sent_at TIMESTAMP,
			attempts INT NOT NULL DEFAULT 0,
			error TEXT
		);
		CREATE INDEX IF NOT EXISTS idx_outbox_status ON outbox_events(status, created_at);
	`)
	return err
}

// SaveWithTransaction บันทึก business data และ outbox event ใน transaction เดียว
func (r *OutboxRepository) SaveWithTransaction(ctx context.Context, tx *sql.Tx, event OutboxEvent) error {
	payload, err := json.Marshal(event.Payload)
	if err != nil {
		return fmt.Errorf("marshaling payload: %w", err)
	}

	_, err = tx.ExecContext(ctx, `
		INSERT INTO outbox_events (id, aggregate_id, event_type, payload, status, created_at)
		VALUES ($1, $2, $3, $4, $5, $6)
	`, event.ID, event.AggregateID, event.EventType, payload, "pending", event.CreatedAt)

	return err
}

// GetPending ดึง events ที่ยังไม่ได้ส่ง
func (r *OutboxRepository) GetPending(ctx context.Context, limit int) ([]OutboxEvent, error) {
	rows, err := r.db.QueryContext(ctx, `
		SELECT id, aggregate_id, event_type, payload, status, created_at, attempts
		FROM outbox_events
		WHERE status = 'pending' AND attempts < 3
		ORDER BY created_at ASC
		LIMIT $1
		FOR UPDATE SKIP LOCKED
	`, limit)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var events []OutboxEvent
	for rows.Next() {
		var e OutboxEvent
		if err := rows.Scan(&e.ID, &e.AggregateID, &e.EventType, &e.Payload, &e.Status, &e.CreatedAt, &e.Attempts); err != nil {
			return nil, err
		}
		events = append(events, e)
	}
	return events, rows.Err()
}

// MarkSent อัพเดทสถานะเป็น sent
func (r *OutboxRepository) MarkSent(ctx context.Context, id string) error {
	now := time.Now()
	_, err := r.db.ExecContext(ctx, `
		UPDATE outbox_events
		SET status = 'sent', sent_at = $1
		WHERE id = $2
	`, now, id)
	return err
}

// MarkFailed อัพเดทสถานะเป็น failed
func (r *OutboxRepository) MarkFailed(ctx context.Context, id string, errMsg string) error {
	_, err := r.db.ExecContext(ctx, `
		UPDATE outbox_events
		SET attempts = attempts + 1,
		    error = $1,
		    status = CASE WHEN attempts + 1 >= 3 THEN 'failed' ELSE 'pending' END
		WHERE id = $2
	`, errMsg, id)
	return err
}

// OutboxPublisher ส่ง events จาก outbox ไปยัง message broker
type OutboxPublisher struct {
	repo      *OutboxRepository
	publisher MessagePublisher
	interval  time.Duration
}

type MessagePublisher interface {
	Publish(ctx context.Context, topic string, payload []byte) error
}

func NewOutboxPublisher(repo *OutboxRepository, publisher MessagePublisher, interval time.Duration) *OutboxPublisher {
	return &OutboxPublisher{
		repo:      repo,
		publisher: publisher,
		interval:  interval,
	}
}

func (p *OutboxPublisher) Start(ctx context.Context) {
	ticker := time.NewTicker(p.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			p.process(ctx)
		}
	}
}

func (p *OutboxPublisher) process(ctx context.Context) {
	events, err := p.repo.GetPending(ctx, 100)
	if err != nil {
		log.Printf("Error getting pending events: %v", err)
		return
	}

	for _, event := range events {
		if err := p.publisher.Publish(ctx, event.EventType, event.Payload); err != nil {
			log.Printf("Error publishing event %s: %v", event.ID, err)
			p.repo.MarkFailed(ctx, event.ID, err.Error())
			continue
		}

		if err := p.repo.MarkSent(ctx, event.ID); err != nil {
			log.Printf("Error marking event %s as sent: %v", event.ID, err)
		}
	}
}
```

---

## 6. Idempotency

```go
// idempotency/handler.go
package idempotency

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
	"time"
)

// IdempotencyStore เก็บ processed requests
type IdempotencyStore interface {
	Get(ctx context.Context, key string) (*IdempotentResponse, error)
	Set(ctx context.Context, key string, resp *IdempotentResponse, ttl time.Duration) error
}

type IdempotentResponse struct {
	StatusCode int             `json:"status_code"`
	Body       json.RawMessage `json:"body"`
	Headers    map[string]string `json:"headers"`
	CreatedAt  time.Time       `json:"created_at"`
}

// InMemoryIdempotencyStore
type InMemoryIdempotencyStore struct {
	mu    sync.RWMutex
	store map[string]storeItem
}

type storeItem struct {
	resp      *IdempotentResponse
	expiresAt time.Time
}

func NewInMemoryIdempotencyStore() *InMemoryIdempotencyStore {
	s := &InMemoryIdempotencyStore{
		store: make(map[string]storeItem),
	}
	go s.cleanup()
	return s
}

func (s *InMemoryIdempotencyStore) Get(ctx context.Context, key string) (*IdempotentResponse, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	item, ok := s.store[key]
	if !ok || time.Now().After(item.expiresAt) {
		return nil, nil
	}
	return item.resp, nil
}

func (s *InMemoryIdempotencyStore) Set(ctx context.Context, key string, resp *IdempotentResponse, ttl time.Duration) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.store[key] = storeItem{
		resp:      resp,
		expiresAt: time.Now().Add(ttl),
	}
	return nil
}

func (s *InMemoryIdempotencyStore) cleanup() {
	ticker := time.NewTicker(time.Minute)
	defer ticker.Stop()
	for range ticker.C {
		s.mu.Lock()
		now := time.Now()
		for k, v := range s.store {
			if now.After(v.expiresAt) {
				delete(s.store, k)
			}
		}
		s.mu.Unlock()
	}
}

// Idempotency Middleware
type IdempotencyMiddleware struct {
	store  IdempotencyStore
	ttl    time.Duration
	keyFn  func(r *http.Request) string
}

func NewIdempotencyMiddleware(store IdempotencyStore, ttl time.Duration) *IdempotencyMiddleware {
	return &IdempotencyMiddleware{
		store: store,
		ttl:   ttl,
		keyFn: defaultKeyFn,
	}
}

func defaultKeyFn(r *http.Request) string {
	// Use Idempotency-Key header if present
	if key := r.Header.Get("Idempotency-Key"); key != "" {
		return key
	}
	// Fall back to content hash
	hash := sha256.Sum256([]byte(r.URL.String()))
	return hex.EncodeToString(hash[:])
}

// responseRecorder captures the response
type responseRecorder struct {
	http.ResponseWriter
	statusCode int
	body       []byte
	headers    http.Header
}

func newResponseRecorder(w http.ResponseWriter) *responseRecorder {
	return &responseRecorder{
		ResponseWriter: w,
		statusCode:     http.StatusOK,
		headers:        make(http.Header),
	}
}

func (r *responseRecorder) WriteHeader(code int) {
	r.statusCode = code
	r.ResponseWriter.WriteHeader(code)
}

func (r *responseRecorder) Write(b []byte) (int, error) {
	r.body = append(r.body, b...)
	return r.ResponseWriter.Write(b)
}

func (m *IdempotencyMiddleware) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Only apply to mutating methods
		if r.Method != "POST" && r.Method != "PUT" && r.Method != "PATCH" {
			next.ServeHTTP(w, r)
			return
		}

		key := m.keyFn(r)
		ctx := r.Context()

		// Check for existing response
		existing, err := m.store.Get(ctx, key)
		if err != nil {
			// Log but continue
			fmt.Printf("Idempotency check error: %v\n", err)
		}

		if existing != nil {
			// Return cached response
			for k, v := range existing.Headers {
				w.Header().Set(k, v)
			}
			w.Header().Set("X-Idempotency-Replayed", "true")
			w.WriteHeader(existing.StatusCode)
			w.Write(existing.Body)
			return
		}

		// Record response
		rec := newResponseRecorder(w)
		next.ServeHTTP(rec, r)

		// Store response for future idempotent requests
		headers := make(map[string]string)
		for k := range w.Header() {
			headers[k] = w.Header().Get(k)
		}

		resp := &IdempotentResponse{
			StatusCode: rec.statusCode,
			Body:       rec.body,
			Headers:    headers,
			CreatedAt:  time.Now(),
		}

		if err := m.store.Set(ctx, key, resp, m.ttl); err != nil {
			fmt.Printf("Failed to store idempotent response: %v\n", err)
		}
	})
}

// Event Idempotency - ป้องกัน duplicate event processing
type EventIdempotencyChecker struct {
	store     IdempotencyStore
	processed map[string]bool
	mu        sync.RWMutex
}

func NewEventIdempotencyChecker(store IdempotencyStore) *EventIdempotencyChecker {
	return &EventIdempotencyChecker{
		store:     store,
		processed: make(map[string]bool),
	}
}

func (c *EventIdempotencyChecker) IsProcessed(ctx context.Context, eventID string) bool {
	c.mu.RLock()
	if c.processed[eventID] {
		c.mu.RUnlock()
		return true
	}
	c.mu.RUnlock()

	resp, _ := c.store.Get(ctx, "event:"+eventID)
	return resp != nil
}

func (c *EventIdempotencyChecker) MarkProcessed(ctx context.Context, eventID string) {
	c.mu.Lock()
	c.processed[eventID] = true
	c.mu.Unlock()

	c.store.Set(ctx, "event:"+eventID, &IdempotentResponse{
		CreatedAt: time.Now(),
	}, 24*time.Hour)
}
```

---

## 7. Complete Event-Driven Example

```go
// example/ecommerce_eda.go
package main

import (
	"context"
	"fmt"
	"log"
	"time"
)

// Simple event-driven e-commerce
type Event struct {
	ID        string
	Type      string
	Data      map[string]interface{}
	Timestamp time.Time
}

type EventHandler func(ctx context.Context, event Event) error

type SimpleEventBus struct {
	handlers map[string][]EventHandler
	ch       chan Event
}

func NewSimpleEventBus() *SimpleEventBus {
	return &SimpleEventBus{
		handlers: make(map[string][]EventHandler),
		ch:       make(chan Event, 100),
	}
}

func (b *SimpleEventBus) On(eventType string, handler EventHandler) {
	b.handlers[eventType] = append(b.handlers[eventType], handler)
}

func (b *SimpleEventBus) Emit(ctx context.Context, event Event) {
	select {
	case b.ch <- event:
	case <-ctx.Done():
	}
}

func (b *SimpleEventBus) Start(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case event := <-b.ch:
			handlers := b.handlers[event.Type]
			for _, h := range handlers {
				go func(handler EventHandler, e Event) {
					if err := handler(ctx, e); err != nil {
						log.Printf("Handler error for %s: %v", e.Type, err)
					}
				}(h, event)
			}
		}
	}
}

func main() {
	bus := NewSimpleEventBus()
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Inventory service
	bus.On("order.created", func(ctx context.Context, e Event) error {
		fmt.Printf("[Inventory] Reserving stock for order %s\n", e.Data["order_id"])
		// Reserve stock...
		bus.Emit(ctx, Event{
			ID:        "evt-inv-001",
			Type:      "inventory.reserved",
			Data:      map[string]interface{}{"order_id": e.Data["order_id"]},
			Timestamp: time.Now(),
		})
		return nil
	})

	// Payment service
	bus.On("inventory.reserved", func(ctx context.Context, e Event) error {
		fmt.Printf("[Payment] Processing payment for order %s\n", e.Data["order_id"])
		// Process payment...
		bus.Emit(ctx, Event{
			ID:        "evt-pay-001",
			Type:      "payment.processed",
			Data:      map[string]interface{}{"order_id": e.Data["order_id"]},
			Timestamp: time.Now(),
		})
		return nil
	})

	// Notification service
	bus.On("payment.processed", func(ctx context.Context, e Event) error {
		fmt.Printf("[Notification] Sending confirmation for order %s\n", e.Data["order_id"])
		return nil
	})

	// Fulfillment service
	bus.On("payment.processed", func(ctx context.Context, e Event) error {
		fmt.Printf("[Fulfillment] Preparing shipment for order %s\n", e.Data["order_id"])
		return nil
	})

	// Start event processing
	go bus.Start(ctx)

	// Simulate order creation
	bus.Emit(ctx, Event{
		ID:        "evt-ord-001",
		Type:      "order.created",
		Data:      map[string]interface{}{"order_id": "ORD-001", "amount": 99.99},
		Timestamp: time.Now(),
	})

	// Wait for processing
	time.Sleep(500 * time.Millisecond)
	fmt.Println("Event processing complete")
}
```

---

## สรุป

| Pattern | จุดประสงค์ | ข้อดี |
|---------|-----------|-------|
| Domain Events | สื่อสาร business events | Decoupling, audit trail |
| Event Bus | ส่ง events ระหว่าง components | Loose coupling |
| Saga | Distributed transactions | ไม่ต้อง 2PC |
| Outbox | Reliable event delivery | At-least-once delivery |
| Idempotency | ป้องกัน duplicate processing | Safe retry |

---

**ต่อไป**: Part 57 - CQRS Pattern
