# Part 58: Event Sourcing

## เป้าหมายการเรียนรู้
- เข้าใจ Event Sourcing concepts
- Event store implementation
- Aggregates
- Projections
- Snapshots
- Replay events
- CQRS + Event Sourcing together

---

## 1. Event Sourcing Concepts

Event Sourcing เก็บ **ทุก event ที่เกิดขึ้น** แทนที่จะเก็บ current state

```
Traditional:  Order { status: "shipped", total: 100 }

Event Sourcing:
  [OrderCreated]   { customer: "alice", items: [...], total: 100 }
  [OrderConfirmed] { confirmed_at: "..." }
  [OrderShipped]   { tracking: "ABC123" }
  
  --> Replay events --> current state
```

### ประโยชน์
1. **Complete audit trail** - รู้ทุกอย่างที่เกิดขึ้น
2. **Time travel** - rebuild state ณ ช่วงเวลาใดก็ได้
3. **Event replay** - rebuild read models ใหม่
4. **Debugging** - เข้าใจว่าเกิดอะไรขึ้น

---

## 2. Domain Events

```go
// eventsourcing/events.go
package eventsourcing

import (
	"encoding/json"
	"time"
)

// Event base type
type Event struct {
	ID            string          `json:"id"`
	Type          string          `json:"type"`
	AggregateID   string          `json:"aggregate_id"`
	AggregateType string          `json:"aggregate_type"`
	Version       int64           `json:"version"`
	Data          json.RawMessage `json:"data"`
	Metadata      Metadata        `json:"metadata"`
	OccurredAt    time.Time       `json:"occurred_at"`
}

type Metadata struct {
	UserID        string            `json:"user_id,omitempty"`
	CorrelationID string            `json:"correlation_id,omitempty"`
	CausationID   string            `json:"causation_id,omitempty"`
	Extra         map[string]string `json:"extra,omitempty"`
}

// Order domain events
type OrderCreatedData struct {
	CustomerID string      `json:"customer_id"`
	Items      []LineItem  `json:"items"`
	TotalAmount float64    `json:"total_amount"`
}

type LineItem struct {
	ProductID   string  `json:"product_id"`
	ProductName string  `json:"product_name"`
	Quantity    int     `json:"quantity"`
	UnitPrice   float64 `json:"unit_price"`
}

type OrderConfirmedData struct {
	ConfirmedAt time.Time `json:"confirmed_at"`
}

type OrderItemAddedData struct {
	Item LineItem `json:"item"`
}

type OrderCancelledData struct {
	Reason      string    `json:"reason"`
	CancelledAt time.Time `json:"cancelled_at"`
}

type OrderShippedData struct {
	TrackingNumber string    `json:"tracking_number"`
	Carrier        string    `json:"carrier"`
	ShippedAt      time.Time `json:"shipped_at"`
}

type OrderDeliveredData struct {
	DeliveredAt time.Time `json:"delivered_at"`
}

// Event type constants
const (
	EventOrderCreated   = "OrderCreated"
	EventOrderConfirmed = "OrderConfirmed"
	EventItemAdded      = "ItemAdded"
	EventOrderCancelled = "OrderCancelled"
	EventOrderShipped   = "OrderShipped"
	EventOrderDelivered = "OrderDelivered"
)
```

---

## 3. Aggregate

```go
// eventsourcing/aggregate.go
package eventsourcing

import (
	"encoding/json"
	"errors"
	"fmt"
	"time"
)

// Aggregate base
type AggregateBase struct {
	id           string
	aggregateType string
	version      int64
	changes      []Event
}

func (a *AggregateBase) ID() string            { return a.id }
func (a *AggregateBase) AggregateType() string { return a.aggregateType }
func (a *AggregateBase) Version() int64        { return a.version }
func (a *AggregateBase) Changes() []Event      { return a.changes }
func (a *AggregateBase) ClearChanges()         { a.changes = nil }

func (a *AggregateBase) raise(eventType string, data interface{}) Event {
	d, _ := json.Marshal(data)
	event := Event{
		ID:            fmt.Sprintf("%s-%d", a.id, a.version+1),
		Type:          eventType,
		AggregateID:   a.id,
		AggregateType: a.aggregateType,
		Version:       a.version + 1,
		Data:          d,
		OccurredAt:    time.Now(),
	}
	a.changes = append(a.changes, event)
	a.version++
	return event
}

// Order Aggregate
type OrderStatus string

const (
	OrderStatusPending   OrderStatus = "pending"
	OrderStatusConfirmed OrderStatus = "confirmed"
	OrderStatusCancelled OrderStatus = "cancelled"
	OrderStatusShipped   OrderStatus = "shipped"
	OrderStatusDelivered OrderStatus = "delivered"
)

type Order struct {
	AggregateBase
	CustomerID string
	Items      []LineItem
	Status     OrderStatus
	TotalAmount float64
	CreatedAt  time.Time
	UpdatedAt  time.Time
}

// NewOrder สร้าง Order ใหม่
func NewOrder(id, customerID string, items []LineItem) (*Order, error) {
	if id == "" {
		return nil, errors.New("id is required")
	}
	if customerID == "" {
		return nil, errors.New("customer_id is required")
	}
	if len(items) == 0 {
		return nil, errors.New("at least one item is required")
	}

	order := &Order{}
	order.id = id
	order.aggregateType = "Order"

	// Calculate total
	total := 0.0
	for _, item := range items {
		total += item.UnitPrice * float64(item.Quantity)
	}

	// Raise event (ไม่ apply state โดยตรง)
	order.raise(EventOrderCreated, OrderCreatedData{
		CustomerID:  customerID,
		Items:       items,
		TotalAmount: total,
	})

	// Apply event to self
	order.apply(order.changes[len(order.changes)-1])

	return order, nil
}

// LoadOrder สร้าง Order จาก event history
func LoadOrder(id string, events []Event) (*Order, error) {
	order := &Order{}
	order.id = id
	order.aggregateType = "Order"

	for _, event := range events {
		if err := order.apply(event); err != nil {
			return nil, fmt.Errorf("applying event %s: %w", event.Type, err)
		}
		order.version = event.Version
	}
	return order, nil
}

// apply อัพเดท state จาก event (ไม่ raise events ใหม่)
func (o *Order) apply(event Event) error {
	switch event.Type {
	case EventOrderCreated:
		var data OrderCreatedData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		o.CustomerID = data.CustomerID
		o.Items = data.Items
		o.TotalAmount = data.TotalAmount
		o.Status = OrderStatusPending
		o.CreatedAt = event.OccurredAt
		o.UpdatedAt = event.OccurredAt

	case EventOrderConfirmed:
		o.Status = OrderStatusConfirmed
		o.UpdatedAt = event.OccurredAt

	case EventItemAdded:
		var data OrderItemAddedData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		o.Items = append(o.Items, data.Item)
		o.TotalAmount += data.Item.UnitPrice * float64(data.Item.Quantity)
		o.UpdatedAt = event.OccurredAt

	case EventOrderCancelled:
		o.Status = OrderStatusCancelled
		o.UpdatedAt = event.OccurredAt

	case EventOrderShipped:
		o.Status = OrderStatusShipped
		o.UpdatedAt = event.OccurredAt

	case EventOrderDelivered:
		o.Status = OrderStatusDelivered
		o.UpdatedAt = event.OccurredAt

	default:
		// Unknown events are ignored (forward compatibility)
	}
	return nil
}

// Business operations - raise events
func (o *Order) Confirm() error {
	if o.Status != OrderStatusPending {
		return fmt.Errorf("cannot confirm order in status: %s", o.Status)
	}

	event := o.raise(EventOrderConfirmed, OrderConfirmedData{
		ConfirmedAt: time.Now(),
	})
	return o.apply(event)
}

func (o *Order) AddItem(item LineItem) error {
	if o.Status != OrderStatusPending {
		return fmt.Errorf("cannot add items to order in status: %s", o.Status)
	}
	if item.Quantity <= 0 {
		return errors.New("quantity must be positive")
	}

	event := o.raise(EventItemAdded, OrderItemAddedData{Item: item})
	return o.apply(event)
}

func (o *Order) Cancel(reason string) error {
	if o.Status == OrderStatusShipped || o.Status == OrderStatusDelivered {
		return fmt.Errorf("cannot cancel order in status: %s", o.Status)
	}

	event := o.raise(EventOrderCancelled, OrderCancelledData{
		Reason:      reason,
		CancelledAt: time.Now(),
	})
	return o.apply(event)
}

func (o *Order) Ship(trackingNumber, carrier string) error {
	if o.Status != OrderStatusConfirmed {
		return fmt.Errorf("cannot ship order in status: %s", o.Status)
	}

	event := o.raise(EventOrderShipped, OrderShippedData{
		TrackingNumber: trackingNumber,
		Carrier:        carrier,
		ShippedAt:      time.Now(),
	})
	return o.apply(event)
}

func (o *Order) Deliver() error {
	if o.Status != OrderStatusShipped {
		return fmt.Errorf("cannot deliver order in status: %s", o.Status)
	}

	event := o.raise(EventOrderDelivered, OrderDeliveredData{
		DeliveredAt: time.Now(),
	})
	return o.apply(event)
}
```

---

## 4. Event Store Implementation

```go
// eventsourcing/store.go
package eventsourcing

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"time"
)

// EventStore interface
type EventStore interface {
	AppendToStream(ctx context.Context, aggregateID string, expectedVersion int64, events []Event) error
	ReadStream(ctx context.Context, aggregateID string) ([]Event, error)
	ReadStreamFromVersion(ctx context.Context, aggregateID string, fromVersion int64) ([]Event, error)
	ReadAllEvents(ctx context.Context, fromPosition int64, limit int) ([]Event, error)
}

// PostgresEventStore
type PostgresEventStore struct {
	db *sql.DB
}

func NewPostgresEventStore(db *sql.DB) *PostgresEventStore {
	return &PostgresEventStore{db: db}
}

func (s *PostgresEventStore) CreateTable(ctx context.Context) error {
	_, err := s.db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS events (
			position BIGSERIAL,
			id VARCHAR(255) PRIMARY KEY,
			aggregate_id VARCHAR(255) NOT NULL,
			aggregate_type VARCHAR(255) NOT NULL,
			version BIGINT NOT NULL,
			type VARCHAR(255) NOT NULL,
			data JSONB NOT NULL,
			metadata JSONB,
			occurred_at TIMESTAMP NOT NULL DEFAULT NOW(),
			
			UNIQUE(aggregate_id, version)
		);
		
		CREATE INDEX IF NOT EXISTS idx_events_aggregate 
			ON events(aggregate_id, version);
		CREATE INDEX IF NOT EXISTS idx_events_position 
			ON events(position);
		CREATE INDEX IF NOT EXISTS idx_events_type 
			ON events(type);
	`)
	return err
}

func (s *PostgresEventStore) AppendToStream(ctx context.Context, aggregateID string, expectedVersion int64, events []Event) error {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("starting transaction: %w", err)
	}
	defer tx.Rollback()

	// Check current version (optimistic locking)
	var currentVersion int64
	err = tx.QueryRowContext(ctx,
		"SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1",
		aggregateID,
	).Scan(&currentVersion)
	if err != nil {
		return fmt.Errorf("checking version: %w", err)
	}

	if expectedVersion != -1 && currentVersion != expectedVersion {
		return fmt.Errorf("concurrency conflict: expected %d, got %d", expectedVersion, currentVersion)
	}

	// Insert events
	stmt, err := tx.PrepareContext(ctx, `
		INSERT INTO events (id, aggregate_id, aggregate_type, version, type, data, metadata, occurred_at)
		VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
	`)
	if err != nil {
		return fmt.Errorf("preparing statement: %w", err)
	}
	defer stmt.Close()

	for _, event := range events {
		metadata, _ := json.Marshal(event.Metadata)
		_, err = stmt.ExecContext(ctx,
			event.ID,
			event.AggregateID,
			event.AggregateType,
			event.Version,
			event.Type,
			event.Data,
			metadata,
			event.OccurredAt,
		)
		if err != nil {
			return fmt.Errorf("inserting event %s: %w", event.ID, err)
		}
	}

	return tx.Commit()
}

func (s *PostgresEventStore) ReadStream(ctx context.Context, aggregateID string) ([]Event, error) {
	rows, err := s.db.QueryContext(ctx, `
		SELECT id, aggregate_id, aggregate_type, version, type, data, metadata, occurred_at
		FROM events
		WHERE aggregate_id = $1
		ORDER BY version ASC
	`, aggregateID)
	if err != nil {
		return nil, fmt.Errorf("querying events: %w", err)
	}
	defer rows.Close()

	return scanEvents(rows)
}

func (s *PostgresEventStore) ReadStreamFromVersion(ctx context.Context, aggregateID string, fromVersion int64) ([]Event, error) {
	rows, err := s.db.QueryContext(ctx, `
		SELECT id, aggregate_id, aggregate_type, version, type, data, metadata, occurred_at
		FROM events
		WHERE aggregate_id = $1 AND version >= $2
		ORDER BY version ASC
	`, aggregateID, fromVersion)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	return scanEvents(rows)
}

func (s *PostgresEventStore) ReadAllEvents(ctx context.Context, fromPosition int64, limit int) ([]Event, error) {
	rows, err := s.db.QueryContext(ctx, `
		SELECT id, aggregate_id, aggregate_type, version, type, data, metadata, occurred_at
		FROM events
		WHERE position > $1
		ORDER BY position ASC
		LIMIT $2
	`, fromPosition, limit)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	return scanEvents(rows)
}

func scanEvents(rows *sql.Rows) ([]Event, error) {
	var events []Event
	for rows.Next() {
		var e Event
		var metadata []byte
		var occurred time.Time

		if err := rows.Scan(
			&e.ID, &e.AggregateID, &e.AggregateType, &e.Version,
			&e.Type, &e.Data, &metadata, &occurred,
		); err != nil {
			return nil, err
		}

		if metadata != nil {
			json.Unmarshal(metadata, &e.Metadata)
		}
		e.OccurredAt = occurred
		events = append(events, e)
	}
	return events, rows.Err()
}
```

---

## 5. Snapshots

```go
// eventsourcing/snapshot.go
package eventsourcing

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"time"
)

// Snapshot เก็บ state ณ เวลาหนึ่ง เพื่อ replay ได้เร็วขึ้น
type Snapshot struct {
	AggregateID   string          `json:"aggregate_id"`
	AggregateType string          `json:"aggregate_type"`
	Version       int64           `json:"version"`
	State         json.RawMessage `json:"state"`
	CreatedAt     time.Time       `json:"created_at"`
}

// SnapshotStore interface
type SnapshotStore interface {
	Get(ctx context.Context, aggregateID string) (*Snapshot, error)
	Save(ctx context.Context, snapshot Snapshot) error
}

// PostgresSnapshotStore
type PostgresSnapshotStore struct {
	db *sql.DB
}

func NewPostgresSnapshotStore(db *sql.DB) *PostgresSnapshotStore {
	return &PostgresSnapshotStore{db: db}
}

func (s *PostgresSnapshotStore) CreateTable(ctx context.Context) error {
	_, err := s.db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS snapshots (
			aggregate_id VARCHAR(255) PRIMARY KEY,
			aggregate_type VARCHAR(255) NOT NULL,
			version BIGINT NOT NULL,
			state JSONB NOT NULL,
			created_at TIMESTAMP NOT NULL DEFAULT NOW()
		)
	`)
	return err
}

func (s *PostgresSnapshotStore) Get(ctx context.Context, aggregateID string) (*Snapshot, error) {
	snap := &Snapshot{}
	err := s.db.QueryRowContext(ctx, `
		SELECT aggregate_id, aggregate_type, version, state, created_at
		FROM snapshots
		WHERE aggregate_id = $1
	`, aggregateID).Scan(
		&snap.AggregateID, &snap.AggregateType, &snap.Version, &snap.State, &snap.CreatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, err
	}
	return snap, nil
}

func (s *PostgresSnapshotStore) Save(ctx context.Context, snap Snapshot) error {
	state, err := json.Marshal(snap.State)
	if err != nil {
		return err
	}

	_, err = s.db.ExecContext(ctx, `
		INSERT INTO snapshots (aggregate_id, aggregate_type, version, state, created_at)
		VALUES ($1, $2, $3, $4, $5)
		ON CONFLICT (aggregate_id) DO UPDATE SET
			version = $3, state = $4, created_at = $5
	`, snap.AggregateID, snap.AggregateType, snap.Version, state, snap.CreatedAt)
	return err
}

// OrderSnapshot state
type OrderSnapshotState struct {
	CustomerID  string      `json:"customer_id"`
	Items       []LineItem  `json:"items"`
	Status      OrderStatus `json:"status"`
	TotalAmount float64     `json:"total_amount"`
	CreatedAt   time.Time   `json:"created_at"`
	UpdatedAt   time.Time   `json:"updated_at"`
}

// Repository with snapshot support
type OrderRepository struct {
	eventStore    EventStore
	snapshotStore SnapshotStore
	snapshotEvery int // take snapshot every N events
}

func NewOrderRepository(eventStore EventStore, snapshotStore SnapshotStore, snapshotEvery int) *OrderRepository {
	return &OrderRepository{
		eventStore:    eventStore,
		snapshotStore: snapshotStore,
		snapshotEvery: snapshotEvery,
	}
}

func (r *OrderRepository) Load(ctx context.Context, orderID string) (*Order, error) {
	// Try to load snapshot first
	snap, err := r.snapshotStore.Get(ctx, orderID)
	if err != nil {
		return nil, fmt.Errorf("loading snapshot: %w", err)
	}

	var order *Order
	var fromVersion int64 = 0

	if snap != nil {
		// Restore from snapshot
		var state OrderSnapshotState
		if err := json.Unmarshal(snap.State, &state); err != nil {
			return nil, fmt.Errorf("unmarshaling snapshot: %w", err)
		}

		order = &Order{}
		order.id = orderID
		order.aggregateType = "Order"
		order.version = snap.Version
		order.CustomerID = state.CustomerID
		order.Items = state.Items
		order.Status = state.Status
		order.TotalAmount = state.TotalAmount
		order.CreatedAt = state.CreatedAt
		order.UpdatedAt = state.UpdatedAt

		fromVersion = snap.Version + 1
	}

	// Load events after snapshot
	events, err := r.eventStore.ReadStreamFromVersion(ctx, orderID, fromVersion)
	if err != nil {
		return nil, fmt.Errorf("reading events: %w", err)
	}

	if order == nil && len(events) == 0 {
		return nil, fmt.Errorf("order not found: %s", orderID)
	}

	if order == nil {
		order = &Order{}
		order.id = orderID
		order.aggregateType = "Order"
	}

	// Apply events
	for _, event := range events {
		if err := order.apply(event); err != nil {
			return nil, fmt.Errorf("applying event: %w", err)
		}
		order.version = event.Version
	}

	return order, nil
}

func (r *OrderRepository) Save(ctx context.Context, order *Order) error {
	if len(order.changes) == 0 {
		return nil
	}

	// Append events
	expectedVersion := order.Version() - int64(len(order.Changes()))
	if err := r.eventStore.AppendToStream(ctx, order.ID(), expectedVersion, order.Changes()); err != nil {
		return fmt.Errorf("appending events: %w", err)
	}

	// Take snapshot if needed
	if r.snapshotEvery > 0 && order.Version()%int64(r.snapshotEvery) == 0 {
		if err := r.takeSnapshot(ctx, order); err != nil {
			// Log but don't fail
			fmt.Printf("Warning: failed to take snapshot: %v\n", err)
		}
	}

	order.ClearChanges()
	return nil
}

func (r *OrderRepository) takeSnapshot(ctx context.Context, order *Order) error {
	state := OrderSnapshotState{
		CustomerID:  order.CustomerID,
		Items:       order.Items,
		Status:      order.Status,
		TotalAmount: order.TotalAmount,
		CreatedAt:   order.CreatedAt,
		UpdatedAt:   order.UpdatedAt,
	}

	stateJSON, err := json.Marshal(state)
	if err != nil {
		return err
	}

	return r.snapshotStore.Save(ctx, Snapshot{
		AggregateID:   order.ID(),
		AggregateType: "Order",
		Version:       order.Version(),
		State:         stateJSON,
		CreatedAt:     time.Now(),
	})
}
```

---

## 6. Event Projection with Replay

```go
// eventsourcing/projection.go
package eventsourcing

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"sync"
	"time"
)

// Projection builds read models from events
type Projection interface {
	Name() string
	HandleEvent(ctx context.Context, event Event) error
	Reset(ctx context.Context) error
}

// OrderSummaryProjection
type OrderSummaryProjection struct {
	mu      sync.RWMutex
	summary map[string]*CustomerSummary // customerID -> summary
}

type CustomerSummary struct {
	CustomerID   string    `json:"customer_id"`
	TotalOrders  int       `json:"total_orders"`
	TotalSpent   float64   `json:"total_spent"`
	LastOrderAt  time.Time `json:"last_order_at"`
	ActiveOrders int       `json:"active_orders"`
}

func NewOrderSummaryProjection() *OrderSummaryProjection {
	return &OrderSummaryProjection{
		summary: make(map[string]*CustomerSummary),
	}
}

func (p *OrderSummaryProjection) Name() string { return "order-summary" }

func (p *OrderSummaryProjection) HandleEvent(ctx context.Context, event Event) error {
	switch event.Type {
	case EventOrderCreated:
		var data OrderCreatedData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}

		p.mu.Lock()
		defer p.mu.Unlock()

		summary := p.getOrCreate(data.CustomerID)
		summary.TotalOrders++
		summary.ActiveOrders++
		summary.LastOrderAt = event.OccurredAt
		return nil

	case EventOrderCancelled:
		var data OrderCancelledData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		_ = data

		// Need to know customer ID - would need to look up order
		// In real implementation, events would contain denormalized data
		return nil

	case EventOrderDelivered:
		// Order completed, update total spent
		// In real implementation would include amount
		return nil
	}
	return nil
}

func (p *OrderSummaryProjection) Reset(ctx context.Context) error {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.summary = make(map[string]*CustomerSummary)
	return nil
}

func (p *OrderSummaryProjection) GetSummary(customerID string) (*CustomerSummary, bool) {
	p.mu.RLock()
	defer p.mu.RUnlock()
	s, ok := p.summary[customerID]
	return s, ok
}

func (p *OrderSummaryProjection) getOrCreate(customerID string) *CustomerSummary {
	if s, ok := p.summary[customerID]; ok {
		return s
	}
	s := &CustomerSummary{CustomerID: customerID}
	p.summary[customerID] = s
	return s
}

// ProjectionRunner manages projections and replays
type ProjectionRunner struct {
	eventStore  EventStore
	projections []Projection
	positions   map[string]int64
	mu          sync.Mutex
}

func NewProjectionRunner(eventStore EventStore, projections ...Projection) *ProjectionRunner {
	positions := make(map[string]int64)
	for _, p := range projections {
		positions[p.Name()] = 0
	}
	return &ProjectionRunner{
		eventStore:  eventStore,
		projections: projections,
		positions:   positions,
	}
}

func (r *ProjectionRunner) Run(ctx context.Context) {
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			r.process(ctx)
		}
	}
}

func (r *ProjectionRunner) process(ctx context.Context) {
	r.mu.Lock()
	defer r.mu.Unlock()

	// Find minimum position
	var minPos int64
	for _, pos := range r.positions {
		if pos < minPos || minPos == 0 {
			minPos = pos
		}
	}

	events, err := r.eventStore.ReadAllEvents(ctx, minPos, 100)
	if err != nil {
		log.Printf("Error reading events: %v", err)
		return
	}

	for _, event := range events {
		for _, p := range r.projections {
			pos := r.positions[p.Name()]
			// Simple position tracking (in real implementation, track per projection)
			if err := p.HandleEvent(ctx, event); err != nil {
				log.Printf("Projection %s error for event %s: %v", p.Name(), event.ID, err)
			}
			r.positions[p.Name()] = pos + 1
		}
	}
}

// Replay all events from beginning
func (r *ProjectionRunner) Replay(ctx context.Context) error {
	// Reset all projections
	for _, p := range r.projections {
		if err := p.Reset(ctx); err != nil {
			return fmt.Errorf("resetting projection %s: %w", p.Name(), err)
		}
		r.positions[p.Name()] = 0
	}

	// Read all events from start
	var position int64
	for {
		events, err := r.eventStore.ReadAllEvents(ctx, position, 1000)
		if err != nil {
			return fmt.Errorf("reading events: %w", err)
		}

		if len(events) == 0 {
			break
		}

		for _, event := range events {
			for _, p := range r.projections {
				if err := p.HandleEvent(ctx, event); err != nil {
					log.Printf("Replay error for projection %s: %v", p.Name(), err)
				}
			}
		}
		position += int64(len(events))
	}

	log.Printf("Replay completed, processed %d events", position)
	return nil
}
```

---

## 7. Complete Event Sourcing Example

```go
// example/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"sync"
	"time"
)

// Simple in-memory event store for demo
type InMemEventStore struct {
	mu     sync.RWMutex
	events []Event
	byID   map[string][]Event
}

func NewInMemEventStore() *InMemEventStore {
	return &InMemEventStore{byID: make(map[string][]Event)}
}

func (s *InMemEventStore) AppendToStream(ctx context.Context, aggregateID string, expectedVersion int64, events []Event) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	currentVersion := int64(len(s.byID[aggregateID]))
	if expectedVersion != -1 && currentVersion != expectedVersion {
		return fmt.Errorf("concurrency conflict: expected %d, got %d", expectedVersion, currentVersion)
	}

	s.byID[aggregateID] = append(s.byID[aggregateID], events...)
	s.events = append(s.events, events...)
	return nil
}

func (s *InMemEventStore) ReadStream(ctx context.Context, aggregateID string) ([]Event, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	events := s.byID[aggregateID]
	result := make([]Event, len(events))
	copy(result, events)
	return result, nil
}

func (s *InMemEventStore) ReadStreamFromVersion(ctx context.Context, aggregateID string, fromVersion int64) ([]Event, error) {
	events, err := s.ReadStream(ctx, aggregateID)
	if err != nil {
		return nil, err
	}
	var result []Event
	for _, e := range events {
		if e.Version >= fromVersion {
			result = append(result, e)
		}
	}
	return result, nil
}

func (s *InMemEventStore) ReadAllEvents(ctx context.Context, fromPosition int64, limit int) ([]Event, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if fromPosition >= int64(len(s.events)) {
		return nil, nil
	}
	end := fromPosition + int64(limit)
	if end > int64(len(s.events)) {
		end = int64(len(s.events))
	}
	result := make([]Event, end-fromPosition)
	copy(result, s.events[fromPosition:end])
	return result, nil
}

// Minimal Event and Order types for demo
type Event struct {
	ID            string          `json:"id"`
	Type          string          `json:"type"`
	AggregateID   string          `json:"aggregate_id"`
	AggregateType string          `json:"aggregate_type"`
	Version       int64           `json:"version"`
	Data          json.RawMessage `json:"data"`
	OccurredAt    time.Time       `json:"occurred_at"`
}

type LineItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	UnitPrice float64 `json:"unit_price"`
}

type OrderStatus string

type Order struct {
	id          string
	version     int64
	changes     []Event
	CustomerID  string
	Items       []LineItem
	Status      OrderStatus
	TotalAmount float64
	CreatedAt   time.Time
}

func (o *Order) Version() int64   { return o.version }
func (o *Order) ID() string       { return o.id }
func (o *Order) Changes() []Event { return o.changes }

func createOrder(id, customerID string, items []LineItem) *Order {
	o := &Order{id: id}
	total := 0.0
	for _, item := range items {
		total += item.UnitPrice * float64(item.Quantity)
	}
	data, _ := json.Marshal(map[string]interface{}{
		"customer_id": customerID,
		"items":       items,
		"total":       total,
	})
	event := Event{
		ID:          fmt.Sprintf("%s-1", id),
		Type:        "OrderCreated",
		AggregateID: id,
		Version:     1,
		Data:        data,
		OccurredAt:  time.Now(),
	}
	o.changes = append(o.changes, event)
	o.CustomerID = customerID
	o.Items = items
	o.Status = "pending"
	o.TotalAmount = total
	o.CreatedAt = event.OccurredAt
	o.version = 1
	return o
}

func loadOrder(id string, events []Event) *Order {
	o := &Order{id: id}
	for _, e := range events {
		switch e.Type {
		case "OrderCreated":
			var data map[string]interface{}
			json.Unmarshal(e.Data, &data)
			o.CustomerID = fmt.Sprint(data["customer_id"])
			o.Status = "pending"
			o.TotalAmount = data["total"].(float64)
			o.CreatedAt = e.OccurredAt
		case "OrderConfirmed":
			o.Status = "confirmed"
		case "OrderShipped":
			o.Status = "shipped"
		}
		o.version = e.Version
	}
	return o
}

func main() {
	ctx := context.Background()
	store := NewInMemEventStore()

	// Create order
	order := createOrder("ord-001", "cust-001", []LineItem{
		{ProductID: "prod-001", Quantity: 2, UnitPrice: 99.99},
	})

	log.Printf("Created order: %s, status: %s, total: %.2f", order.ID(), order.Status, order.TotalAmount)

	// Save to event store
	store.AppendToStream(ctx, order.ID(), 0, order.Changes())

	// Reload from events
	events, _ := store.ReadStream(ctx, order.ID())
	loaded := loadOrder(order.ID(), events)
	log.Printf("Loaded order: %s, status: %s, customer: %s", loaded.ID(), loaded.Status, loaded.CustomerID)

	// Confirm order
	confirmData, _ := json.Marshal(map[string]interface{}{"confirmed_at": time.Now()})
	confirmEvent := Event{
		ID:          fmt.Sprintf("%s-2", order.ID()),
		Type:        "OrderConfirmed",
		AggregateID: order.ID(),
		Version:     loaded.version + 1,
		Data:        confirmData,
		OccurredAt:  time.Now(),
	}
	store.AppendToStream(ctx, order.ID(), loaded.version, []Event{confirmEvent})

	// Reload again
	events, _ = store.ReadStream(ctx, order.ID())
	confirmed := loadOrder(order.ID(), events)
	log.Printf("After confirm: %s, status: %s", confirmed.ID(), confirmed.Status)

	// Show event history
	log.Printf("\n=== Event History for %s ===", order.ID())
	for _, e := range events {
		log.Printf("  v%d [%s] %s", e.Version, e.OccurredAt.Format(time.RFC3339), e.Type)
	}
}
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|---------|
| Event Store | เก็บ events ทั้งหมดตามลำดับ |
| Aggregate | สร้าง current state จาก events |
| Snapshot | เก็บ state ณ เวลาหนึ่ง เพื่อ replay เร็วขึ้น |
| Projection | สร้าง read model จาก events |
| Replay | รัน events ซ้ำเพื่อ rebuild state |

---

**ต่อไป**: Part 59 - Domain-Driven Design
