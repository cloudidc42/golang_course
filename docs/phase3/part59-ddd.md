# Part 59: Domain-Driven Design (DDD)

## เป้าหมายการเรียนรู้
- เข้าใจ Domain-Driven Design overview
- Ubiquitous Language
- Bounded Contexts
- Entities, Value Objects, Aggregates
- Domain Events
- Repositories
- Domain Services
- Application Services
- Go DDD implementation

---

## 1. DDD Overview

Domain-Driven Design (DDD) เป็น approach สำหรับออกแบบ software ที่ซับซ้อน โดยเน้นที่ **domain** (business logic) เป็นหลัก

### Strategic Design
- **Ubiquitous Language**: ภาษากลางระหว่าง developers และ domain experts
- **Bounded Context**: ขอบเขตของ domain model แต่ละส่วน
- **Context Map**: ความสัมพันธ์ระหว่าง Bounded Contexts

### Tactical Design
- **Entity**: object ที่มี identity
- **Value Object**: object ที่ไม่มี identity (immutable)
- **Aggregate**: กลุ่มของ Entities ที่เกี่ยวข้องกัน
- **Repository**: ดึงและบันทึก Aggregates
- **Domain Service**: business logic ที่ไม่เป็น natural ของ Entity ใด
- **Application Service**: orchestrates use cases

---

## 2. Ubiquitous Language

```go
// domain/language.go
// ตัวอย่างการใช้ Ubiquitous Language ใน code

package domain

import (
	"errors"
	"time"
)

// ใช้ภาษาจาก domain โดยตรง ไม่ใช่ technical terms

// OrderStatus ใช้ภาษาที่ business experts ใช้
type OrderStatus string

const (
	OrderStatusDraft     OrderStatus = "Draft"    // ร่าง
	OrderStatusPlaced    OrderStatus = "Placed"   // วางออเดอร์แล้ว
	OrderStatusPaid      OrderStatus = "Paid"     // จ่ายเงินแล้ว
	OrderStatusFulfilled OrderStatus = "Fulfilled" // จัดส่งแล้ว
	OrderStatusCancelled OrderStatus = "Cancelled" // ยกเลิก
)

// CustomerTier ระดับลูกค้า
type CustomerTier string

const (
	CustomerTierStandard CustomerTier = "Standard"
	CustomerTierSilver   CustomerTier = "Silver"
	CustomerTierGold     CustomerTier = "Gold"
	CustomerTierPlatinum CustomerTier = "Platinum"
)

// DiscountPolicy นโยบายส่วนลด
type DiscountPolicy interface {
	CalculateDiscount(subtotal Money) Money
}

// Money - value object สำหรับเงิน
type Money struct {
	Amount   float64
	Currency string
}

func NewMoney(amount float64, currency string) (Money, error) {
	if amount < 0 {
		return Money{}, errors.New("amount cannot be negative")
	}
	if currency == "" {
		return Money{}, errors.New("currency is required")
	}
	return Money{Amount: amount, Currency: currency}, nil
}

func (m Money) Add(other Money) (Money, error) {
	if m.Currency != other.Currency {
		return Money{}, errors.New("cannot add different currencies")
	}
	return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

func (m Money) Subtract(other Money) (Money, error) {
	if m.Currency != other.Currency {
		return Money{}, errors.New("cannot subtract different currencies")
	}
	if m.Amount < other.Amount {
		return Money{}, errors.New("insufficient funds")
	}
	return Money{Amount: m.Amount - other.Amount, Currency: m.Currency}, nil
}

func (m Money) Multiply(factor float64) Money {
	return Money{Amount: m.Amount * factor, Currency: m.Currency}
}

func (m Money) IsGreaterThan(other Money) bool {
	return m.Currency == other.Currency && m.Amount > other.Amount
}

func (m Money) IsZero() bool {
	return m.Amount == 0
}

// Quantity - value object สำหรับจำนวน
type Quantity struct {
	Value int
	Unit  string
}

func NewQuantity(value int, unit string) (Quantity, error) {
	if value <= 0 {
		return Quantity{}, errors.New("quantity must be positive")
	}
	return Quantity{Value: value, Unit: unit}, nil
}

func (q Quantity) Add(other Quantity) (Quantity, error) {
	if q.Unit != other.Unit {
		return Quantity{}, errors.New("cannot add different units")
	}
	return Quantity{Value: q.Value + other.Value, Unit: q.Unit}, nil
}

// Address - value object
type Address struct {
	Street     string
	City       string
	State      string
	PostalCode string
	Country    string
}

func (a Address) Validate() error {
	if a.Street == "" || a.City == "" || a.Country == "" {
		return errors.New("street, city, and country are required")
	}
	return nil
}

// DateRange - value object
type DateRange struct {
	Start time.Time
	End   time.Time
}

func NewDateRange(start, end time.Time) (DateRange, error) {
	if end.Before(start) {
		return DateRange{}, errors.New("end date must be after start date")
	}
	return DateRange{Start: start, End: end}, nil
}

func (d DateRange) Contains(t time.Time) bool {
	return !t.Before(d.Start) && !t.After(d.End)
}

func (d DateRange) Duration() time.Duration {
	return d.End.Sub(d.Start)
}
```

---

## 3. Entities

```go
// domain/customer.go
package domain

import (
	"errors"
	"regexp"
	"time"
)

// Customer Entity - มี identity (CustomerID)
type CustomerID string

type Customer struct {
	id        CustomerID
	name      PersonName
	email     Email
	phone     *Phone
	address   *Address
	tier      CustomerTier
	createdAt time.Time
	updatedAt time.Time
}

// PersonName - value object
type PersonName struct {
	FirstName string
	LastName  string
}

func (n PersonName) Full() string {
	return n.FirstName + " " + n.LastName
}

func (n PersonName) Validate() error {
	if n.FirstName == "" || n.LastName == "" {
		return errors.New("first name and last name are required")
	}
	return nil
}

// Email - value object
type Email struct {
	Value string
}

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func NewEmail(email string) (Email, error) {
	if !emailRegex.MatchString(email) {
		return Email{}, errors.New("invalid email format")
	}
	return Email{Value: email}, nil
}

// Phone - value object
type Phone struct {
	CountryCode string
	Number      string
}

func NewPhone(countryCode, number string) (Phone, error) {
	if number == "" {
		return Phone{}, errors.New("phone number is required")
	}
	return Phone{CountryCode: countryCode, Number: number}, nil
}

// NewCustomer factory function
func NewCustomer(id CustomerID, name PersonName, email Email) (*Customer, error) {
	if err := name.Validate(); err != nil {
		return nil, err
	}

	now := time.Now()
	return &Customer{
		id:        id,
		name:      name,
		email:     email,
		tier:      CustomerTierStandard,
		createdAt: now,
		updatedAt: now,
	}, nil
}

// Accessors
func (c *Customer) ID() CustomerID        { return c.id }
func (c *Customer) Name() PersonName      { return c.name }
func (c *Customer) Email() Email          { return c.email }
func (c *Customer) Tier() CustomerTier    { return c.tier }
func (c *Customer) CreatedAt() time.Time  { return c.createdAt }
func (c *Customer) UpdatedAt() time.Time  { return c.updatedAt }

// Domain operations
func (c *Customer) ChangeName(name PersonName) error {
	if err := name.Validate(); err != nil {
		return err
	}
	c.name = name
	c.updatedAt = time.Now()
	return nil
}

func (c *Customer) UpdateEmail(email Email) {
	c.email = email
	c.updatedAt = time.Now()
}

func (c *Customer) SetPhone(phone Phone) {
	c.phone = &phone
	c.updatedAt = time.Now()
}

func (c *Customer) SetAddress(address Address) error {
	if err := address.Validate(); err != nil {
		return err
	}
	c.address = &address
	c.updatedAt = time.Now()
	return nil
}

func (c *Customer) Upgrade(tier CustomerTier) {
	c.tier = tier
	c.updatedAt = time.Now()
}

func (c *Customer) IsEligibleForPromotion() bool {
	return c.tier == CustomerTierGold || c.tier == CustomerTierPlatinum
}
```

---

## 4. Aggregates

```go
// domain/order.go
package domain

import (
	"errors"
	"fmt"
	"time"
)

// OrderID - value object เป็น identity ของ Order
type OrderID string

// ProductID - value object
type ProductID string

// OrderLine เป็น entity ภายใน Order aggregate
type OrderLine struct {
	id        OrderLineID
	productID ProductID
	name      string
	quantity  Quantity
	unitPrice Money
}

type OrderLineID string

func NewOrderLine(id OrderLineID, productID ProductID, name string, quantity Quantity, unitPrice Money) (*OrderLine, error) {
	if name == "" {
		return nil, errors.New("product name is required")
	}
	return &OrderLine{
		id:        id,
		productID: productID,
		name:      name,
		quantity:  quantity,
		unitPrice: unitPrice,
	}, nil
}

func (l *OrderLine) Subtotal() Money {
	return l.unitPrice.Multiply(float64(l.quantity.Value))
}

func (l *OrderLine) ProductID() ProductID { return l.productID }
func (l *OrderLine) Quantity() Quantity   { return l.quantity }
func (l *OrderLine) UnitPrice() Money     { return l.unitPrice }

// Order Aggregate Root
type Order struct {
	id           OrderID
	customerID   CustomerID
	lines        []*OrderLine
	status       OrderStatus
	shippingAddr Address
	discount     Money
	createdAt    time.Time
	updatedAt    time.Time
	events       []DomainEvent
}

// NewOrder - factory function
func NewOrder(id OrderID, customerID CustomerID, shippingAddr Address) (*Order, error) {
	if err := shippingAddr.Validate(); err != nil {
		return nil, fmt.Errorf("invalid shipping address: %w", err)
	}

	now := time.Now()
	order := &Order{
		id:           id,
		customerID:   customerID,
		shippingAddr: shippingAddr,
		status:       OrderStatusDraft,
		discount:     Money{Amount: 0, Currency: "THB"},
		createdAt:    now,
		updatedAt:    now,
	}

	order.raise(&OrderDraftedEvent{
		OrderID:    id,
		CustomerID: customerID,
		DraftedAt:  now,
	})

	return order, nil
}

// Accessors
func (o *Order) ID() OrderID          { return o.id }
func (o *Order) CustomerID() CustomerID { return o.customerID }
func (o *Order) Status() OrderStatus  { return o.status }
func (o *Order) Lines() []*OrderLine  { return o.lines }
func (o *Order) Events() []DomainEvent { return o.events }

// CalculateSubtotal คำนวณยอดรวมก่อนส่วนลด
func (o *Order) CalculateSubtotal() Money {
	total := Money{Amount: 0, Currency: "THB"}
	for _, line := range o.lines {
		total.Amount += line.Subtotal().Amount
	}
	return total
}

// CalculateTotal คำนวณยอดรวมหลังส่วนลด
func (o *Order) CalculateTotal() (Money, error) {
	subtotal := o.CalculateSubtotal()
	return subtotal.Subtract(o.discount)
}

// AddLine เพิ่มสินค้าในออเดอร์
func (o *Order) AddLine(line *OrderLine) error {
	if o.status != OrderStatusDraft {
		return fmt.Errorf("cannot add items to order in status: %s", o.status)
	}

	// Check if product already in order
	for _, existing := range o.lines {
		if existing.productID == line.productID {
			return errors.New("product already in order, use UpdateQuantity instead")
		}
	}

	o.lines = append(o.lines, line)
	o.updatedAt = time.Now()
	return nil
}

// RemoveLine ลบสินค้าออกจากออเดอร์
func (o *Order) RemoveLine(productID ProductID) error {
	if o.status != OrderStatusDraft {
		return errors.New("cannot remove items from placed order")
	}

	for i, line := range o.lines {
		if line.productID == productID {
			o.lines = append(o.lines[:i], o.lines[i+1:]...)
			o.updatedAt = time.Now()
			return nil
		}
	}
	return fmt.Errorf("product %s not found in order", productID)
}

// ApplyDiscount ใช้ส่วนลด
func (o *Order) ApplyDiscount(discount Money) error {
	if discount.Currency != "THB" {
		return errors.New("discount must be in THB")
	}
	subtotal := o.CalculateSubtotal()
	if discount.Amount > subtotal.Amount {
		return errors.New("discount cannot exceed subtotal")
	}
	o.discount = discount
	o.updatedAt = time.Now()
	return nil
}

// Place วางออเดอร์
func (o *Order) Place() error {
	if o.status != OrderStatusDraft {
		return fmt.Errorf("cannot place order in status: %s", o.status)
	}
	if len(o.lines) == 0 {
		return errors.New("cannot place empty order")
	}

	o.status = OrderStatusPlaced
	o.updatedAt = time.Now()

	total, _ := o.CalculateTotal()
	o.raise(&OrderPlacedEvent{
		OrderID:    o.id,
		CustomerID: o.customerID,
		Total:      total,
		PlacedAt:   o.updatedAt,
	})

	return nil
}

// MarkAsPaid บันทึกว่าจ่ายเงินแล้ว
func (o *Order) MarkAsPaid(transactionID string) error {
	if o.status != OrderStatusPlaced {
		return fmt.Errorf("cannot mark as paid order in status: %s", o.status)
	}

	o.status = OrderStatusPaid
	o.updatedAt = time.Now()

	o.raise(&OrderPaidEvent{
		OrderID:       o.id,
		TransactionID: transactionID,
		PaidAt:        o.updatedAt,
	})

	return nil
}

// Cancel ยกเลิกออเดอร์
func (o *Order) Cancel(reason string) error {
	if o.status == OrderStatusFulfilled {
		return errors.New("cannot cancel fulfilled order")
	}

	o.status = OrderStatusCancelled
	o.updatedAt = time.Now()

	o.raise(&OrderCancelledEvent{
		OrderID:     o.id,
		Reason:      reason,
		CancelledAt: o.updatedAt,
	})

	return nil
}

func (o *Order) raise(event DomainEvent) {
	o.events = append(o.events, event)
}

func (o *Order) ClearEvents() {
	o.events = nil
}
```

---

## 5. Domain Events

```go
// domain/events.go
package domain

import "time"

// DomainEvent interface
type DomainEvent interface {
	EventType() string
	OccurredAt() time.Time
}

type OrderDraftedEvent struct {
	OrderID    OrderID
	CustomerID CustomerID
	DraftedAt  time.Time
}

func (e *OrderDraftedEvent) EventType() string   { return "OrderDrafted" }
func (e *OrderDraftedEvent) OccurredAt() time.Time { return e.DraftedAt }

type OrderPlacedEvent struct {
	OrderID    OrderID
	CustomerID CustomerID
	Total      Money
	PlacedAt   time.Time
}

func (e *OrderPlacedEvent) EventType() string   { return "OrderPlaced" }
func (e *OrderPlacedEvent) OccurredAt() time.Time { return e.PlacedAt }

type OrderPaidEvent struct {
	OrderID       OrderID
	TransactionID string
	PaidAt        time.Time
}

func (e *OrderPaidEvent) EventType() string   { return "OrderPaid" }
func (e *OrderPaidEvent) OccurredAt() time.Time { return e.PaidAt }

type OrderCancelledEvent struct {
	OrderID     OrderID
	Reason      string
	CancelledAt time.Time
}

func (e *OrderCancelledEvent) EventType() string   { return "OrderCancelled" }
func (e *OrderCancelledEvent) OccurredAt() time.Time { return e.CancelledAt }
```

---

## 6. Repositories

```go
// domain/repositories.go
package domain

import "context"

// OrderRepository - repository interface (defined in domain layer)
type OrderRepository interface {
	NextID(ctx context.Context) (OrderID, error)
	FindByID(ctx context.Context, id OrderID) (*Order, error)
	FindByCustomer(ctx context.Context, customerID CustomerID) ([]*Order, error)
	Save(ctx context.Context, order *Order) error
	Delete(ctx context.Context, id OrderID) error
}

// CustomerRepository
type CustomerRepository interface {
	NextID(ctx context.Context) (CustomerID, error)
	FindByID(ctx context.Context, id CustomerID) (*Customer, error)
	FindByEmail(ctx context.Context, email Email) (*Customer, error)
	Save(ctx context.Context, customer *Customer) error
	ExistsByEmail(ctx context.Context, email Email) (bool, error)
}

// ProductRepository
type ProductRepository interface {
	FindByID(ctx context.Context, id ProductID) (*Product, error)
	FindByIDs(ctx context.Context, ids []ProductID) ([]*Product, error)
}

// Product entity (minimal)
type Product struct {
	id    ProductID
	name  string
	price Money
	stock int
}

func (p *Product) ID() ProductID    { return p.id }
func (p *Product) Name() string     { return p.name }
func (p *Product) Price() Money     { return p.price }
func (p *Product) IsAvailable() bool { return p.stock > 0 }
func (p *Product) HasEnough(qty int) bool { return p.stock >= qty }
```

---

## 7. Domain Services

```go
// domain/services.go
package domain

import (
	"context"
	"errors"
	"fmt"
)

// PricingService - domain service สำหรับคำนวณราคา
type PricingService struct {
	discountRepo DiscountRepository
}

type DiscountRepository interface {
	FindApplicable(ctx context.Context, customerTier CustomerTier, total Money) (*Discount, error)
}

type Discount struct {
	ID         string
	Type       string // percentage, fixed
	Value      float64
	MinAmount  Money
	MaxDiscount *Money
}

func NewPricingService(repo DiscountRepository) *PricingService {
	return &PricingService{discountRepo: repo}
}

func (s *PricingService) CalculateDiscount(ctx context.Context, customer *Customer, order *Order) (Money, error) {
	subtotal := order.CalculateSubtotal()
	
	discount, err := s.discountRepo.FindApplicable(ctx, customer.Tier(), subtotal)
	if err != nil {
		return Money{}, fmt.Errorf("finding discount: %w", err)
	}
	if discount == nil {
		return Money{Amount: 0, Currency: "THB"}, nil
	}

	var discountAmount float64
	switch discount.Type {
	case "percentage":
		discountAmount = subtotal.Amount * discount.Value / 100
	case "fixed":
		discountAmount = discount.Value
	default:
		return Money{}, fmt.Errorf("unknown discount type: %s", discount.Type)
	}

	// Apply max discount if set
	if discount.MaxDiscount != nil && discountAmount > discount.MaxDiscount.Amount {
		discountAmount = discount.MaxDiscount.Amount
	}

	return Money{Amount: discountAmount, Currency: "THB"}, nil
}

// OrderFulfillmentService - domain service สำหรับ fulfillment
type OrderFulfillmentService struct {
	inventoryService InventoryService
}

type InventoryService interface {
	CheckAvailability(ctx context.Context, productID ProductID, qty int) error
	Reserve(ctx context.Context, productID ProductID, qty int) error
	Release(ctx context.Context, productID ProductID, qty int) error
}

func NewOrderFulfillmentService(inventory InventoryService) *OrderFulfillmentService {
	return &OrderFulfillmentService{inventoryService: inventory}
}

func (s *OrderFulfillmentService) CheckOrderFulfillable(ctx context.Context, order *Order) error {
	for _, line := range order.Lines() {
		if err := s.inventoryService.CheckAvailability(ctx, line.ProductID(), line.Quantity().Value); err != nil {
			return fmt.Errorf("product %s: %w", line.ProductID(), err)
		}
	}
	return nil
}

func (s *OrderFulfillmentService) ReserveInventory(ctx context.Context, order *Order) error {
	reserved := make([]struct{ id ProductID; qty int }, 0)

	for _, line := range order.Lines() {
		if err := s.inventoryService.Reserve(ctx, line.ProductID(), line.Quantity().Value); err != nil {
			// Compensate - release already reserved
			for _, r := range reserved {
				s.inventoryService.Release(ctx, r.id, r.qty)
			}
			return fmt.Errorf("reserving %s: %w", line.ProductID(), err)
		}
		reserved = append(reserved, struct{ id ProductID; qty int }{line.ProductID(), line.Quantity().Value})
	}
	return nil
}

// TransferService - domain service สำหรับ money transfer
type MoneyTransferService struct{}

func (s *MoneyTransferService) Transfer(from, to *Account, amount Money) error {
	if err := from.Debit(amount); err != nil {
		return fmt.Errorf("debiting from account: %w", err)
	}
	if err := to.Credit(amount); err != nil {
		// Compensate
		from.Credit(amount)
		return fmt.Errorf("crediting to account: %w", err)
	}
	return nil
}

// Account entity (simplified)
type Account struct {
	id      string
	balance Money
}

func (a *Account) Debit(amount Money) error {
	balance, err := a.balance.Subtract(amount)
	if err != nil {
		return errors.New("insufficient balance")
	}
	a.balance = balance
	return nil
}

func (a *Account) Credit(amount Money) error {
	balance, err := a.balance.Add(amount)
	if err != nil {
		return err
	}
	a.balance = balance
	return nil
}
```

---

## 8. Application Services

```go
// application/order_service.go
package application

import (
	"context"
	"fmt"
	"time"
)

// PlaceOrderCommand input DTO
type PlaceOrderCommand struct {
	CustomerID string
	Items      []PlaceOrderItem
	ShippingAddress ShippingAddressDTO
}

type PlaceOrderItem struct {
	ProductID string
	Quantity  int
}

type ShippingAddressDTO struct {
	Street     string
	City       string
	State      string
	PostalCode string
	Country    string
}

// PlaceOrderResult output DTO
type PlaceOrderResult struct {
	OrderID   string
	Total     float64
	Currency  string
	Status    string
	CreatedAt time.Time
}

// PlaceOrderUseCase application service
type PlaceOrderUseCase struct {
	customerRepo    domain.CustomerRepository
	orderRepo       domain.OrderRepository
	productRepo     domain.ProductRepository
	pricingService  *domain.PricingService
	fulfillService  *domain.OrderFulfillmentService
	eventPublisher  EventPublisher
}

type EventPublisher interface {
	Publish(ctx context.Context, events []domain.DomainEvent) error
}

func NewPlaceOrderUseCase(
	customerRepo domain.CustomerRepository,
	orderRepo domain.OrderRepository,
	productRepo domain.ProductRepository,
	pricingService *domain.PricingService,
	fulfillService *domain.OrderFulfillmentService,
	eventPublisher EventPublisher,
) *PlaceOrderUseCase {
	return &PlaceOrderUseCase{
		customerRepo:   customerRepo,
		orderRepo:      orderRepo,
		productRepo:    productRepo,
		pricingService: pricingService,
		fulfillService: fulfillService,
		eventPublisher: eventPublisher,
	}
}

func (uc *PlaceOrderUseCase) Execute(ctx context.Context, cmd PlaceOrderCommand) (*PlaceOrderResult, error) {
	// 1. Load customer
	customer, err := uc.customerRepo.FindByID(ctx, domain.CustomerID(cmd.CustomerID))
	if err != nil {
		return nil, fmt.Errorf("customer not found: %w", err)
	}

	// 2. Validate and load products
	var productIDs []domain.ProductID
	for _, item := range cmd.Items {
		productIDs = append(productIDs, domain.ProductID(item.ProductID))
	}
	products, err := uc.productRepo.FindByIDs(ctx, productIDs)
	if err != nil {
		return nil, fmt.Errorf("loading products: %w", err)
	}

	// Create product map
	productMap := make(map[domain.ProductID]*domain.Product)
	for _, p := range products {
		productMap[p.ID()] = p
	}

	// 3. Create order
	orderID, err := uc.orderRepo.NextID(ctx)
	if err != nil {
		return nil, fmt.Errorf("generating order ID: %w", err)
	}

	shippingAddr := domain.Address{
		Street:     cmd.ShippingAddress.Street,
		City:       cmd.ShippingAddress.City,
		State:      cmd.ShippingAddress.State,
		PostalCode: cmd.ShippingAddress.PostalCode,
		Country:    cmd.ShippingAddress.Country,
	}

	order, err := domain.NewOrder(orderID, customer.ID(), shippingAddr)
	if err != nil {
		return nil, fmt.Errorf("creating order: %w", err)
	}

	// 4. Add order lines
	for i, item := range cmd.Items {
		product, ok := productMap[domain.ProductID(item.ProductID)]
		if !ok {
			return nil, fmt.Errorf("product %s not found", item.ProductID)
		}

		qty, err := domain.NewQuantity(item.Quantity, "unit")
		if err != nil {
			return nil, fmt.Errorf("invalid quantity for %s: %w", item.ProductID, err)
		}

		line, err := domain.NewOrderLine(
			domain.OrderLineID(fmt.Sprintf("%s-line-%d", orderID, i+1)),
			product.ID(),
			product.Name(),
			qty,
			product.Price(),
		)
		if err != nil {
			return nil, err
		}

		if err := order.AddLine(line); err != nil {
			return nil, err
		}
	}

	// 5. Calculate and apply discount
	discount, err := uc.pricingService.CalculateDiscount(ctx, customer, order)
	if err != nil {
		return nil, fmt.Errorf("calculating discount: %w", err)
	}
	if !discount.IsZero() {
		order.ApplyDiscount(discount)
	}

	// 6. Check inventory
	if err := uc.fulfillService.CheckOrderFulfillable(ctx, order); err != nil {
		return nil, fmt.Errorf("insufficient inventory: %w", err)
	}

	// 7. Place order
	if err := order.Place(); err != nil {
		return nil, fmt.Errorf("placing order: %w", err)
	}

	// 8. Save order
	if err := uc.orderRepo.Save(ctx, order); err != nil {
		return nil, fmt.Errorf("saving order: %w", err)
	}

	// 9. Publish domain events
	if err := uc.eventPublisher.Publish(ctx, order.Events()); err != nil {
		// Log but don't fail the use case
		// Events will be retried
		fmt.Printf("Warning: failed to publish events: %v\n", err)
	}
	order.ClearEvents()

	// 10. Return result
	total, _ := order.CalculateTotal()
	return &PlaceOrderResult{
		OrderID:   string(order.ID()),
		Total:     total.Amount,
		Currency:  total.Currency,
		Status:    string(order.Status()),
		CreatedAt: time.Now(),
	}, nil
}
```

---

## 9. Bounded Context Integration

```go
// bounded_context/context_map.go
package bounded_context

// Context Map แสดงความสัมพันธ์ระหว่าง Bounded Contexts

// Ordering Context
// - Order Aggregate
// - Customer (read-only reference)
// - Product (read-only reference)

// Catalog Context (upstream)
// - Product Aggregate
// - Category Aggregate

// Identity Context (upstream)
// - User Aggregate

// Integration ระหว่าง contexts ผ่าน Anti-Corruption Layer
// ACL แปลง model จาก upstream context มาเป็น model ของ context เรา

// CatalogProductDTO - model จาก Catalog context
type CatalogProductDTO struct {
	ID          string  `json:"id"`
	SKU         string  `json:"sku"`
	Title       string  `json:"title"`
	ListPrice   float64 `json:"list_price"`
	Currency    string  `json:"currency"`
	IsAvailable bool    `json:"is_available"`
}

// ProductACL - Anti-Corruption Layer แปลง Catalog model -> Ordering model
type ProductACL struct {
	catalogClient CatalogClient
}

type CatalogClient interface {
	GetProduct(id string) (*CatalogProductDTO, error)
}

func NewProductACL(client CatalogClient) *ProductACL {
	return &ProductACL{catalogClient: client}
}

// GetProductForOrdering แปลง Catalog product เป็น Ordering domain model
func (acl *ProductACL) GetProductForOrdering(id string) (*OrderingProduct, error) {
	catalogProduct, err := acl.catalogClient.GetProduct(id)
	if err != nil {
		return nil, fmt.Errorf("fetching from catalog: %w", err)
	}
	if !catalogProduct.IsAvailable {
		return nil, fmt.Errorf("product %s is not available", id)
	}

	return &OrderingProduct{
		ID:    id,
		Name:  catalogProduct.Title,
		Price: Money{Amount: catalogProduct.ListPrice, Currency: catalogProduct.Currency},
	}, nil
}

type OrderingProduct struct {
	ID    string
	Name  string
	Price Money
}
```

---

## Workshop: Complete DDD Order System

```go
// workshop/ddd_demo.go
package main

import (
	"context"
	"fmt"
	"log"
)

func main() {
	// Setup
	customerRepo := NewInMemCustomerRepo()
	orderRepo := NewInMemOrderRepo()
	productRepo := NewInMemProductRepo()

	// Seed data
	ctx := context.Background()
	
	// Create customer
	name, _ := NewPersonName("Alice", "Smith")
	email, _ := NewEmail("alice@example.com")
	customer, _ := NewCustomer("cust-001", name, email)
	customerRepo.Save(ctx, customer)

	// Create products
	productRepo.Add("prod-001", "MacBook Pro", Money{Amount: 59900, Currency: "THB"})
	productRepo.Add("prod-002", "iPhone 15", Money{Amount: 32900, Currency: "THB"})

	// Place order
	orderID, _ := orderRepo.NextID(ctx)
	shippingAddr := Address{
		Street: "123 Main St",
		City:   "Bangkok",
		Country: "Thailand",
	}
	
	order, err := NewOrder(orderID, customer.ID(), shippingAddr)
	if err != nil {
		log.Fatalf("Creating order: %v", err)
	}

	// Add items
	product1, _ := productRepo.FindByID(ctx, "prod-001")
	qty1, _ := NewQuantity(1, "unit")
	line1, _ := NewOrderLine("line-001", product1.id, product1.name, qty1, product1.price)
	order.AddLine(line1)

	// Apply 10% discount
	subtotal := order.CalculateSubtotal()
	discount := Money{Amount: subtotal.Amount * 0.1, Currency: "THB"}
	order.ApplyDiscount(discount)

	total, _ := order.CalculateTotal()
	fmt.Printf("Order total: %.2f THB (after %.2f THB discount)\n", total.Amount, discount.Amount)

	// Place order
	if err := order.Place(); err != nil {
		log.Fatalf("Placing order: %v", err)
	}
	fmt.Printf("Order status: %s\n", order.Status())

	// Events raised
	for _, event := range order.Events() {
		fmt.Printf("Event: %s at %v\n", event.EventType(), event.OccurredAt())
	}

	// Save
	orderRepo.Save(ctx, order)
	fmt.Println("Order saved successfully")
}

// Simple in-memory implementations
type InMemCustomerRepo struct{ customers map[CustomerID]*Customer }
func NewInMemCustomerRepo() *InMemCustomerRepo { return &InMemCustomerRepo{make(map[CustomerID]*Customer)} }
func (r *InMemCustomerRepo) Save(ctx context.Context, c *Customer) error { r.customers[c.ID()] = c; return nil }
func (r *InMemCustomerRepo) FindByID(ctx context.Context, id CustomerID) (*Customer, error) {
	if c, ok := r.customers[id]; ok { return c, nil }
	return nil, fmt.Errorf("customer not found: %s", id)
}

type InMemOrderRepo struct{ orders map[OrderID]*Order; counter int }
func NewInMemOrderRepo() *InMemOrderRepo { return &InMemOrderRepo{orders: make(map[OrderID]*Order)} }
func (r *InMemOrderRepo) NextID(ctx context.Context) (OrderID, error) {
	r.counter++
	return OrderID(fmt.Sprintf("ord-%03d", r.counter)), nil
}
func (r *InMemOrderRepo) Save(ctx context.Context, o *Order) error { r.orders[o.ID()] = o; return nil }

type InMemProductRepo struct{ products map[ProductID]*simpleProduct }
type simpleProduct struct{ id ProductID; name string; price Money }
func NewInMemProductRepo() *InMemProductRepo { return &InMemProductRepo{make(map[ProductID]*simpleProduct)} }
func (r *InMemProductRepo) Add(id, name string, price Money) {
	r.products[ProductID(id)] = &simpleProduct{ProductID(id), name, price}
}
func (r *InMemProductRepo) FindByID(ctx context.Context, id ProductID) (*simpleProduct, error) {
	if p, ok := r.products[id]; ok { return p, nil }
	return nil, fmt.Errorf("product not found: %s", id)
}

func NewPersonName(first, last string) (PersonName, error) {
	n := PersonName{FirstName: first, LastName: last}
	return n, n.Validate()
}

func NewEmail(e string) (Email, error) { return Email{Value: e}, nil }

func NewCustomer(id CustomerID, name PersonName, email Email) (*Customer, error) {
	return &Customer{id: id, name: name, email: email, tier: CustomerTierStandard}, nil
}
```

---

## สรุป

| Concept | ความหมาย | Go Implementation |
|---------|----------|------------------|
| Entity | Object ที่มี identity | struct + private fields |
| Value Object | Object ที่ immutable | struct ที่ copy by value |
| Aggregate | กลุ่ม Entities | Root entity ที่ control access |
| Repository | interface สำหรับ persistence | interface ใน domain |
| Domain Service | Business logic ที่ไม่ fit ใน entity | struct ที่ orchestrates entities |
| Application Service | Use case orchestration | Use case struct |
| Domain Event | สิ่งที่เกิดขึ้นใน domain | interface + implementations |

---

**ต่อไป**: Part 60 - Hexagonal Architecture
