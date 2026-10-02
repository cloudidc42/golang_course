# Part 62: GraphQL ใน Go

## เป้าหมายการเรียนรู้
- GraphQL overview และ concepts
- gqlgen setup และการใช้งาน
- Schema definition language (SDL)
- Resolvers, Mutations, Subscriptions
- DataLoader แก้ปัญหา N+1
- Authentication และ Authorization
- Pagination (cursor-based)
- ตัวอย่างโค้ด 20+ ตัวอย่าง

---

## 1. GraphQL Overview

GraphQL คือ query language สำหรับ API ที่ช่วยให้ client กำหนดได้เองว่าต้องการข้อมูลอะไร

```
# GraphQL vs REST
REST:
  GET /users/1          → ได้ทุก field ของ user
  GET /users/1/posts    → ได้ posts ของ user (2 requests)

GraphQL:
  query {
    user(id: "1") {
      id
      name
      posts {          ← nested query ใน 1 request
        title
      }
    }
  }
```

---

## 2. Schema Definition

```graphql
# schema/schema.graphql

scalar Time
scalar Upload

enum UserRole {
  ADMIN
  USER
  MODERATOR
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

type User {
  id: ID!
  email: String!
  name: String!
  role: UserRole!
  createdAt: Time!
  orders: [Order!]!
  profile: UserProfile
}

type UserProfile {
  bio: String
  avatar: String
  website: String
}

type Order {
  id: ID!
  status: OrderStatus!
  total: Float!
  items: [OrderItem!]!
  createdAt: Time!
  user: User!
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  price: Float!
}

type Product {
  id: ID!
  name: String!
  description: String
  price: Float!
  stock: Int!
  category: Category!
}

type Category {
  id: ID!
  name: String!
  products: [Product!]!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  cursor: String!
  node: User!
}

input CreateUserInput {
  email: String!
  name: String!
  password: String!
  role: UserRole = USER
}

input UpdateUserInput {
  name: String
  bio: String
  website: String
}

input CreateOrderInput {
  items: [OrderItemInput!]!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}

type Query {
  user(id: ID!): User
  users(first: Int, after: String, last: Int, before: String): UserConnection!
  me: User
  order(id: ID!): Order
  orders: [Order!]!
  product(id: ID!): Product
  products(categoryId: ID): [Product!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createOrder(input: CreateOrderInput!): Order!
  updateOrderStatus(id: ID!, status: OrderStatus!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
  newOrder: Order!
}
```

---

## 3. gqlgen Setup

```go
// tools.go
//go:build tools
// +build tools

package tools

import (
	_ "github.com/99designs/gqlgen"
	_ "github.com/99designs/gqlgen/graphql/introspection"
)
```

```yaml
# gqlgen.yml
schema:
  - schema/*.graphql

exec:
  filename: graph/generated.go
  package: graph

model:
  filename: graph/model/models_gen.go
  package: model

resolver:
  layout: follow-schema
  dir: graph
  package: graph
  filename_template: "{name}.resolvers.go"

autobind:
  - "github.com/example/graphql/internal/model"

models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql/introspection.ID
      - github.com/99designs/gqlgen/graphql/builtin.ID
  Time:
    model: github.com/99designs/gqlgen/graphql/builtin.Time
```

---

## 4. Model Definitions

```go
// internal/model/user.go
package model

import "time"

type User struct {
	ID        string    `json:"id" db:"id"`
	Email     string    `json:"email" db:"email"`
	Name      string    `json:"name" db:"name"`
	Password  string    `json:"-" db:"password_hash"`
	Role      UserRole  `json:"role" db:"role"`
	CreatedAt time.Time `json:"createdAt" db:"created_at"`
}

type UserRole string

const (
	RoleAdmin     UserRole = "ADMIN"
	RoleUser      UserRole = "USER"
	RoleModerator UserRole = "MODERATOR"
)

type Order struct {
	ID        string      `json:"id" db:"id"`
	UserID    string      `json:"userId" db:"user_id"`
	Status    OrderStatus `json:"status" db:"status"`
	Total     float64     `json:"total" db:"total"`
	CreatedAt time.Time   `json:"createdAt" db:"created_at"`
}

type OrderStatus string

const (
	StatusPending    OrderStatus = "PENDING"
	StatusProcessing OrderStatus = "PROCESSING"
	StatusShipped    OrderStatus = "SHIPPED"
	StatusDelivered  OrderStatus = "DELIVERED"
	StatusCancelled  OrderStatus = "CANCELLED"
)

type Product struct {
	ID          string  `json:"id" db:"id"`
	Name        string  `json:"name" db:"name"`
	Description string  `json:"description" db:"description"`
	Price       float64 `json:"price" db:"price"`
	Stock       int     `json:"stock" db:"stock"`
	CategoryID  string  `json:"categoryId" db:"category_id"`
}
```

---

## 5. Resolvers

```go
// graph/query.resolvers.go
package graph

import (
	"context"
	"fmt"

	"github.com/example/graphql/graph/model"
	"github.com/example/graphql/internal/auth"
)

func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
	return r.UserService.GetByID(ctx, id)
}

func (r *queryResolver) Me(ctx context.Context) (*model.User, error) {
	// Get user from context (set by auth middleware)
	userID, ok := auth.GetUserID(ctx)
	if !ok {
		return nil, fmt.Errorf("not authenticated")
	}
	return r.UserService.GetByID(ctx, userID)
}

func (r *queryResolver) Users(ctx context.Context, first *int, after *string, last *int, before *string) (*model.UserConnection, error) {
	// Cursor-based pagination
	var limit int = 20
	if first != nil {
		limit = *first
	}

	users, hasNext, err := r.UserService.ListWithCursor(ctx, limit, after)
	if err != nil {
		return nil, err
	}

	edges := make([]*model.UserEdge, len(users))
	for i, u := range users {
		edges[i] = &model.UserEdge{
			Cursor: encodeCursor(u.ID),
			Node:   u,
		}
	}

	conn := &model.UserConnection{
		Edges: edges,
		PageInfo: &model.PageInfo{
			HasNextPage:     hasNext,
			HasPreviousPage: after != nil,
		},
	}

	if len(edges) > 0 {
		conn.PageInfo.StartCursor = &edges[0].Cursor
		conn.PageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}

	return conn, nil
}

func encodeCursor(id string) string {
	return fmt.Sprintf("cursor:%s", id)
}

func decodeCursor(cursor string) string {
	var id string
	fmt.Sscanf(cursor, "cursor:%s", &id)
	return id
}
```

---

## 6. Mutation Resolvers

```go
// graph/mutation.resolvers.go
package graph

import (
	"context"
	"fmt"

	"github.com/example/graphql/graph/model"
	"github.com/example/graphql/internal/auth"
)

func (r *mutationResolver) CreateUser(ctx context.Context, input model.CreateUserInput) (*model.User, error) {
	// Validate input
	if len(input.Password) < 8 {
		return nil, fmt.Errorf("password must be at least 8 characters")
	}

	user, err := r.UserService.Create(ctx, &CreateUserParams{
		Email:    input.Email,
		Name:     input.Name,
		Password: input.Password,
		Role:     model.UserRole(input.Role),
	})
	if err != nil {
		return nil, fmt.Errorf("creating user: %w", err)
	}

	return user, nil
}

func (r *mutationResolver) UpdateUser(ctx context.Context, id string, input model.UpdateUserInput) (*model.User, error) {
	// Check authorization
	currentUserID, ok := auth.GetUserID(ctx)
	if !ok {
		return nil, fmt.Errorf("not authenticated")
	}

	if currentUserID != id {
		// Check if admin
		if !auth.HasRole(ctx, model.RoleAdmin) {
			return nil, fmt.Errorf("cannot update other user's profile")
		}
	}

	return r.UserService.Update(ctx, id, input)
}

func (r *mutationResolver) CreateOrder(ctx context.Context, input model.CreateOrderInput) (*model.Order, error) {
	userID, ok := auth.GetUserID(ctx)
	if !ok {
		return nil, fmt.Errorf("not authenticated")
	}

	order, err := r.OrderService.Create(ctx, userID, input)
	if err != nil {
		return nil, fmt.Errorf("creating order: %w", err)
	}

	// Publish to subscribers
	r.OrderEvents <- order

	return order, nil
}

type CreateUserParams struct {
	Email    string
	Name     string
	Password string
	Role     model.UserRole
}
```

---

## 7. Subscriptions

```go
// graph/subscription.resolvers.go
package graph

import (
	"context"
	"log"

	"github.com/example/graphql/graph/model"
)

func (r *subscriptionResolver) OrderStatusChanged(ctx context.Context, orderID string) (<-chan *model.Order, error) {
	// Create a channel for this subscriber
	ch := make(chan *model.Order, 1)

	// Subscribe to order events
	unsubscribe := r.EventBus.Subscribe(func(order *model.Order) {
		if order.ID == orderID {
			select {
			case ch <- order:
			default:
				log.Printf("subscriber channel full for order %s", orderID)
			}
		}
	})

	// Cleanup when context is done
	go func() {
		<-ctx.Done()
		unsubscribe()
		close(ch)
	}()

	return ch, nil
}

func (r *subscriptionResolver) NewOrder(ctx context.Context) (<-chan *model.Order, error) {
	ch := make(chan *model.Order, 10)

	// Only admins can subscribe to all new orders
	if !isAdmin(ctx) {
		return nil, errForbidden
	}

	unsubscribe := r.EventBus.Subscribe(func(order *model.Order) {
		select {
		case ch <- order:
		case <-ctx.Done():
		}
	})

	go func() {
		<-ctx.Done()
		unsubscribe()
		close(ch)
	}()

	return ch, nil
}
```

---

## 8. DataLoader (แก้ N+1 Problem)

```go
// dataloader/dataloader.go
package dataloader

import (
	"context"
	"sync"
	"time"
)

// UserLoader batches individual user loads into bulk database queries
type UserLoader struct {
	wait     time.Duration
	maxBatch int
	fetch    func(keys []string) ([]*User, []error)
	
	mu     sync.Mutex
	batch  *userBatch
}

type userBatch struct {
	keys    []string
	data    []*User
	errors  []error
	closing bool
	done    chan struct{}
}

type User struct {
	ID    string
	Email string
	Name  string
}

func NewUserLoader(fetchFn func(keys []string) ([]*User, []error)) *UserLoader {
	return &UserLoader{
		wait:     2 * time.Millisecond,
		maxBatch: 100,
		fetch:    fetchFn,
	}
}

// Load a single user by key, batching with other loads
func (l *UserLoader) Load(key string) (*User, error) {
	return l.LoadThunk(key)()
}

func (l *UserLoader) LoadThunk(key string) func() (*User, error) {
	l.mu.Lock()
	if l.batch == nil {
		l.batch = &userBatch{done: make(chan struct{})}
	}
	batch := l.batch
	pos := batch.keyIndex(l, key)
	l.mu.Unlock()

	return func() (*User, error) {
		<-batch.done

		if batch.errors[pos] != nil {
			return nil, batch.errors[pos]
		}
		return batch.data[pos], nil
	}
}

func (b *userBatch) keyIndex(l *UserLoader, key string) int {
	for i, k := range b.keys {
		if k == key {
			return i
		}
	}

	pos := len(b.keys)
	b.keys = append(b.keys, key)
	b.data = append(b.data, nil)
	b.errors = append(b.errors, nil)

	if pos == 0 {
		go b.startTimer(l)
	}
	if l.maxBatch != 0 && pos >= l.maxBatch-1 {
		if !b.closing {
			b.closing = true
			l.batch = nil
			go b.end(l)
		}
	}

	return pos
}

func (b *userBatch) startTimer(l *UserLoader) {
	time.Sleep(l.wait)
	l.mu.Lock()
	if b == l.batch {
		l.batch = nil
	}
	l.mu.Unlock()
	b.end(l)
}

func (b *userBatch) end(l *UserLoader) {
	b.data, b.errors = l.fetch(b.keys)
	close(b.done)
}

// LoadMany batches multiple users at once
func (l *UserLoader) LoadMany(keys []string) ([]*User, []error) {
	results := make([]*User, len(keys))
	errors := make([]error, len(keys))

	var wg sync.WaitGroup
	for i, key := range keys {
		wg.Add(1)
		go func(i int, key string) {
			defer wg.Done()
			results[i], errors[i] = l.Load(key)
		}(i, key)
	}
	wg.Wait()

	return results, errors
}
```

```go
// dataloader/middleware.go
package dataloader

import (
	"context"
	"net/http"
)

type contextKey string

const loadersKey contextKey = "dataloaders"

type Loaders struct {
	UserByID    *UserLoader
	ProductByID *ProductLoader
}

type ProductLoader struct {
	// similar to UserLoader
}

type DB interface {
	GetUsersByIDs(ctx context.Context, ids []string) ([]*User, error)
}

// Middleware injects DataLoaders into context
func Middleware(db DB) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx := r.Context()

			loaders := &Loaders{
				UserByID: NewUserLoader(func(ids []string) ([]*User, []error) {
					users, err := db.GetUsersByIDs(ctx, ids)
					if err != nil {
						errors := make([]error, len(ids))
						for i := range errors {
							errors[i] = err
						}
						return nil, errors
					}

					// Map by ID
					userMap := make(map[string]*User, len(users))
					for _, u := range users {
						userMap[u.ID] = u
					}

					// Return in same order as ids
					result := make([]*User, len(ids))
					errs := make([]error, len(ids))
					for i, id := range ids {
						if u, ok := userMap[id]; ok {
							result[i] = u
						} else {
							errs[i] = ErrNotFound
						}
					}
					return result, errs
				}),
			}

			ctx = context.WithValue(ctx, loadersKey, loaders)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

var ErrNotFound = errNotFound("not found")

type errNotFound string

func (e errNotFound) Error() string { return string(e) }

func For(ctx context.Context) *Loaders {
	return ctx.Value(loadersKey).(*Loaders)
}
```

---

## 9. Authentication Middleware

```go
// middleware/auth.go
package middleware

import (
	"context"
	"net/http"
	"strings"
)

type contextKey string

const (
	userIDKey contextKey = "userID"
	rolesKey  contextKey = "roles"
)

type TokenValidator interface {
	ValidateToken(token string) (userID string, roles []string, err error)
}

func AuthMiddleware(validator TokenValidator) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			authHeader := r.Header.Get("Authorization")
			if authHeader == "" {
				// Anonymous request - still proceed
				next.ServeHTTP(w, r)
				return
			}

			parts := strings.SplitN(authHeader, " ", 2)
			if len(parts) != 2 || parts[0] != "Bearer" {
				http.Error(w, "invalid authorization header", http.StatusUnauthorized)
				return
			}

			userID, roles, err := validator.ValidateToken(parts[1])
			if err != nil {
				http.Error(w, "invalid token", http.StatusUnauthorized)
				return
			}

			ctx := context.WithValue(r.Context(), userIDKey, userID)
			ctx = context.WithValue(ctx, rolesKey, roles)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

func GetUserID(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(userIDKey).(string)
	return id, ok && id != ""
}

func HasRole(ctx context.Context, role string) bool {
	roles, ok := ctx.Value(rolesKey).([]string)
	if !ok {
		return false
	}
	for _, r := range roles {
		if r == role {
			return true
		}
	}
	return false
}
```

---

## 10. Server Setup

```go
// main.go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/99designs/gqlgen/graphql/handler"
	"github.com/99designs/gqlgen/graphql/handler/extension"
	"github.com/99designs/gqlgen/graphql/handler/transport"
	"github.com/99designs/gqlgen/graphql/playground"
	"github.com/gorilla/websocket"

	"github.com/example/graphql/graph"
	"github.com/example/graphql/graph/generated"
	"github.com/example/graphql/middleware"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	// Build resolver
	resolver := &graph.Resolver{
		// inject dependencies
	}

	// Create GraphQL handler
	srv := handler.New(generated.NewExecutableSchema(generated.Config{
		Resolvers: resolver,
		Directives: generated.DirectiveRoot{},
		Complexity: generated.ComplexityRoot{},
	}))

	// Add transports
	srv.AddTransport(transport.Options{})
	srv.AddTransport(transport.GET{})
	srv.AddTransport(transport.POST{})
	srv.AddTransport(transport.MultipartForm{})
	srv.AddTransport(&transport.Websocket{
		Upgrader: websocket.Upgrader{
			CheckOrigin: func(r *http.Request) bool {
				return true
			},
		},
	})

	// Add extensions
	srv.Use(extension.Introspection{})
	srv.Use(extension.AutomaticPersistedQuery{
		Cache: extension.NewLRUCache(100),
	})

	// Routes
	mux := http.NewServeMux()
	
	// GraphQL endpoint with auth middleware
	mux.Handle("/graphql", 
		middleware.AuthMiddleware(tokenValidator)(srv),
	)

	// GraphQL Playground (development only)
	if os.Getenv("ENV") != "production" {
		mux.Handle("/playground", playground.Handler("GraphQL Playground", "/graphql"))
	}

	log.Printf("GraphQL server starting on :%s", port)
	log.Printf("Playground: http://localhost:%s/playground", port)
	if err := http.ListenAndServe(":"+port, mux); err != nil {
		log.Fatalf("server error: %v", err)
	}
}

// Mock token validator
type mockTokenValidator struct{}

var tokenValidator = &mockTokenValidator{}

func (m *mockTokenValidator) ValidateToken(token string) (string, []string, error) {
	return "user-1", []string{"USER"}, nil
}
```

---

## 11. Resolver เชื่อม DataLoader

```go
// graph/resolver.go
package graph

import (
	"context"

	"github.com/example/graphql/dataloader"
	"github.com/example/graphql/graph/model"
	"github.com/example/graphql/internal/service"
)

type Resolver struct {
	UserService  service.UserService
	OrderService service.OrderService
	EventBus     *EventBus
	OrderEvents  chan *model.Order
}

// User.orders resolver - ใช้ DataLoader เพื่อหลีกเลี่ยง N+1
func (r *userResolver) Orders(ctx context.Context, obj *model.User) ([]*model.Order, error) {
	// Without DataLoader: 1 query per user (N+1 problem)
	// With DataLoader: batched into 1 query for all users
	orders, err := dataloader.For(ctx).OrdersByUserID.Load(obj.ID)
	if err != nil {
		return nil, err
	}
	return orders, nil
}

// Order.user resolver
func (r *orderResolver) User(ctx context.Context, obj *model.Order) (*model.User, error) {
	// DataLoader batches these into a single DB query
	user, err := dataloader.For(ctx).UserByID.Load(obj.UserID)
	if err != nil {
		return nil, err
	}
	return &model.User{
		ID:    user.ID,
		Email: user.Email,
		Name:  user.Name,
	}, nil
}
```

---

## 12. Complex Query Example

```graphql
# ตัวอย่าง query ที่ซับซ้อน
query GetDashboard {
  me {
    id
    name
    role
    orders {
      id
      status
      total
      createdAt
      items {
        quantity
        price
        product {
          name
          category {
            name
          }
        }
      }
    }
  }
}

# Mutation with variables
mutation CreateOrder($items: [OrderItemInput!]!) {
  createOrder(input: { items: $items }) {
    id
    status
    total
    items {
      product {
        name
      }
      quantity
      price
    }
  }
}

# Subscription
subscription WatchOrder($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    updatedAt
  }
}
```

---

## สรุป

| Feature | Package/Approach |
|---------|-----------------|
| Code generation | `gqlgen` |
| DataLoader | Custom batching with goroutines |
| Subscriptions | WebSocket transport |
| Auth | Middleware + context |
| Pagination | Cursor-based (Relay spec) |
| N+1 prevention | DataLoader pattern |

---

**ต่อไป**: Part 63 - Distributed Tracing
