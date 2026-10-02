# Part 60: Hexagonal Architecture (Ports & Adapters)

## เป้าหมายการเรียนรู้
- เข้าใจ Hexagonal Architecture
- Domain layer
- Application layer
- Infrastructure layer
- Ports (interfaces)
- Adapters (implementations)
- Dependency injection
- Testing with clean architecture
- Workshop: Complete service

---

## 1. Hexagonal Architecture Overview

Hexagonal Architecture (หรือ Ports & Adapters) แยก business logic ออกจาก infrastructure concerns

```
                    ┌─────────────────────────────┐
    HTTP Adapter ──>│                             │
    CLI Adapter  ──>│   Application / Domain       │──> Database Adapter
    gRPC Adapter ──>│                             │──> Email Adapter
                    └─────────────────────────────┘
                          Ports (Interfaces)
```

### 3 Layers หลัก
1. **Domain Layer**: business rules, entities, value objects
2. **Application Layer**: use cases, orchestrates domain
3. **Infrastructure Layer**: implementations (DB, HTTP, etc.)

```
/internal
├── domain/           # Core business logic
│   ├── user.go       # Entity
│   ├── repository.go # Port (interface)
│   └── service.go    # Domain service
├── application/      # Use cases
│   ├── create_user.go
│   └── get_user.go
└── adapters/         # Infrastructure
    ├── primary/      # Driving adapters (HTTP, CLI, gRPC)
    │   └── http/
    └── secondary/    # Driven adapters (DB, Email, etc.)
        ├── postgres/
        └── email/
```

---

## 2. Domain Layer

```go
// internal/domain/user.go
package domain

import (
	"errors"
	"regexp"
	"time"
)

// User entity
type User struct {
	id        UserID
	email     Email
	name      Name
	role      Role
	active    bool
	createdAt time.Time
	updatedAt time.Time
}

type UserID string
type Email string
type Name string
type Role string

const (
	RoleAdmin   Role = "admin"
	RoleUser    Role = "user"
	RoleViewer  Role = "viewer"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func NewUser(id UserID, email Email, name Name, role Role) (*User, error) {
	if id == "" {
		return nil, errors.New("user id is required")
	}
	if !emailRegex.MatchString(string(email)) {
		return nil, errors.New("invalid email format")
	}
	if name == "" {
		return nil, errors.New("name is required")
	}
	if role != RoleAdmin && role != RoleUser && role != RoleViewer {
		return nil, errors.New("invalid role")
	}

	now := time.Now()
	return &User{
		id:        id,
		email:     email,
		name:      name,
		role:      role,
		active:    true,
		createdAt: now,
		updatedAt: now,
	}, nil
}

// Getters
func (u *User) ID() UserID        { return u.id }
func (u *User) Email() Email      { return u.email }
func (u *User) Name() Name        { return u.name }
func (u *User) Role() Role        { return u.role }
func (u *User) IsActive() bool    { return u.active }
func (u *User) CreatedAt() time.Time { return u.createdAt }
func (u *User) UpdatedAt() time.Time { return u.updatedAt }

// Domain methods
func (u *User) Activate() {
	u.active = true
	u.updatedAt = time.Now()
}

func (u *User) Deactivate() {
	u.active = false
	u.updatedAt = time.Now()
}

func (u *User) UpdateName(name Name) error {
	if name == "" {
		return errors.New("name cannot be empty")
	}
	u.name = name
	u.updatedAt = time.Now()
	return nil
}

func (u *User) ChangeRole(role Role) error {
	if role != RoleAdmin && role != RoleUser && role != RoleViewer {
		return errors.New("invalid role")
	}
	u.role = role
	u.updatedAt = time.Now()
	return nil
}

func (u *User) IsAdmin() bool { return u.role == RoleAdmin }

// Domain errors
var (
	ErrUserNotFound      = errors.New("user not found")
	ErrEmailAlreadyTaken = errors.New("email already taken")
	ErrUserInactive      = errors.New("user is inactive")
)
```

```go
// internal/domain/repository.go - Ports (Secondary ports)
package domain

import "context"

// UserRepository - secondary port
type UserRepository interface {
	FindByID(ctx context.Context, id UserID) (*User, error)
	FindByEmail(ctx context.Context, email Email) (*User, error)
	FindAll(ctx context.Context, filter UserFilter) ([]*User, error)
	Save(ctx context.Context, user *User) error
	Delete(ctx context.Context, id UserID) error
	ExistsByEmail(ctx context.Context, email Email) (bool, error)
}

type UserFilter struct {
	Role   *Role
	Active *bool
	Limit  int
	Offset int
}

// IDGenerator - secondary port
type IDGenerator interface {
	Generate() UserID
}

// EventPublisher - secondary port
type EventPublisher interface {
	Publish(ctx context.Context, topic string, event interface{}) error
}

// EmailSender - secondary port
type EmailSender interface {
	SendWelcome(ctx context.Context, to Email, name Name) error
	SendPasswordReset(ctx context.Context, to Email, token string) error
}

// Cache - secondary port
type Cache interface {
	Get(ctx context.Context, key string) (interface{}, bool)
	Set(ctx context.Context, key string, value interface{}, ttl int) error
	Delete(ctx context.Context, key string) error
}
```

---

## 3. Application Layer (Use Cases)

```go
// internal/application/create_user.go
package application

import (
	"context"
	"fmt"
	"time"

	"github.com/example/app/internal/domain"
)

// CreateUserCommand - input
type CreateUserCommand struct {
	Email string
	Name  string
	Role  string
}

// CreateUserResult - output
type CreateUserResult struct {
	ID        string
	Email     string
	Name      string
	Role      string
	Active    bool
	CreatedAt time.Time
}

// CreateUserUseCase
type CreateUserUseCase struct {
	userRepo       domain.UserRepository
	idGenerator    domain.IDGenerator
	emailSender    domain.EmailSender
	eventPublisher domain.EventPublisher
}

func NewCreateUserUseCase(
	userRepo domain.UserRepository,
	idGenerator domain.IDGenerator,
	emailSender domain.EmailSender,
	eventPublisher domain.EventPublisher,
) *CreateUserUseCase {
	return &CreateUserUseCase{
		userRepo:       userRepo,
		idGenerator:    idGenerator,
		emailSender:    emailSender,
		eventPublisher: eventPublisher,
	}
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, cmd CreateUserCommand) (*CreateUserResult, error) {
	// 1. Check email uniqueness
	exists, err := uc.userRepo.ExistsByEmail(ctx, domain.Email(cmd.Email))
	if err != nil {
		return nil, fmt.Errorf("checking email: %w", err)
	}
	if exists {
		return nil, domain.ErrEmailAlreadyTaken
	}

	// 2. Generate ID
	id := uc.idGenerator.Generate()

	// 3. Create user entity
	user, err := domain.NewUser(
		id,
		domain.Email(cmd.Email),
		domain.Name(cmd.Name),
		domain.Role(cmd.Role),
	)
	if err != nil {
		return nil, fmt.Errorf("creating user: %w", err)
	}

	// 4. Save user
	if err := uc.userRepo.Save(ctx, user); err != nil {
		return nil, fmt.Errorf("saving user: %w", err)
	}

	// 5. Send welcome email (non-blocking)
	go func() {
		bgCtx := context.Background()
		if err := uc.emailSender.SendWelcome(bgCtx, user.Email(), user.Name()); err != nil {
			// Log but don't fail
			fmt.Printf("Warning: failed to send welcome email: %v\n", err)
		}
	}()

	// 6. Publish event
	uc.eventPublisher.Publish(ctx, "user.created", map[string]interface{}{
		"user_id": string(user.ID()),
		"email":   string(user.Email()),
	})

	return &CreateUserResult{
		ID:        string(user.ID()),
		Email:     string(user.Email()),
		Name:      string(user.Name()),
		Role:      string(user.Role()),
		Active:    user.IsActive(),
		CreatedAt: user.CreatedAt(),
	}, nil
}
```

```go
// internal/application/get_user.go
package application

import (
	"context"
	"fmt"
	"time"

	"github.com/example/app/internal/domain"
)

type GetUserQuery struct {
	UserID string
}

type UserDTO struct {
	ID        string    `json:"id"`
	Email     string    `json:"email"`
	Name      string    `json:"name"`
	Role      string    `json:"role"`
	Active    bool      `json:"active"`
	CreatedAt time.Time `json:"created_at"`
}

type GetUserUseCase struct {
	userRepo domain.UserRepository
	cache    domain.Cache
}

func NewGetUserUseCase(userRepo domain.UserRepository, cache domain.Cache) *GetUserUseCase {
	return &GetUserUseCase{userRepo: userRepo, cache: cache}
}

func (uc *GetUserUseCase) Execute(ctx context.Context, query GetUserQuery) (*UserDTO, error) {
	cacheKey := fmt.Sprintf("user:%s", query.UserID)

	// Check cache
	if cached, ok := uc.cache.Get(ctx, cacheKey); ok {
		if dto, ok := cached.(*UserDTO); ok {
			return dto, nil
		}
	}

	// Load from repository
	user, err := uc.userRepo.FindByID(ctx, domain.UserID(query.UserID))
	if err != nil {
		return nil, domain.ErrUserNotFound
	}

	dto := &UserDTO{
		ID:        string(user.ID()),
		Email:     string(user.Email()),
		Name:      string(user.Name()),
		Role:      string(user.Role()),
		Active:    user.IsActive(),
		CreatedAt: user.CreatedAt(),
	}

	// Cache result
	uc.cache.Set(ctx, cacheKey, dto, 300) // 5 minutes

	return dto, nil
}

// ListUsersUseCase
type ListUsersQuery struct {
	Role   string
	Active *bool
	Page   int
	Size   int
}

type ListUsersResult struct {
	Users  []UserDTO `json:"users"`
	Total  int       `json:"total"`
	Page   int       `json:"page"`
	Size   int       `json:"size"`
}

type ListUsersUseCase struct {
	userRepo domain.UserRepository
}

func NewListUsersUseCase(userRepo domain.UserRepository) *ListUsersUseCase {
	return &ListUsersUseCase{userRepo: userRepo}
}

func (uc *ListUsersUseCase) Execute(ctx context.Context, query ListUsersQuery) (*ListUsersResult, error) {
	filter := domain.UserFilter{
		Limit:  query.Size,
		Offset: (query.Page - 1) * query.Size,
	}
	if query.Role != "" {
		role := domain.Role(query.Role)
		filter.Role = &role
	}
	if query.Active != nil {
		filter.Active = query.Active
	}

	users, err := uc.userRepo.FindAll(ctx, filter)
	if err != nil {
		return nil, fmt.Errorf("listing users: %w", err)
	}

	var dtos []UserDTO
	for _, u := range users {
		dtos = append(dtos, UserDTO{
			ID:        string(u.ID()),
			Email:     string(u.Email()),
			Name:      string(u.Name()),
			Role:      string(u.Role()),
			Active:    u.IsActive(),
			CreatedAt: u.CreatedAt(),
		})
	}

	return &ListUsersResult{
		Users: dtos,
		Total: len(dtos),
		Page:  query.Page,
		Size:  query.Size,
	}, nil
}
```

---

## 4. Infrastructure - Secondary Adapters

```go
// internal/adapters/secondary/postgres/user_repository.go
package postgres

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"github.com/example/app/internal/domain"
)

type UserRepository struct {
	db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

type userRow struct {
	ID        string
	Email     string
	Name      string
	Role      string
	Active    bool
	CreatedAt time.Time
	UpdatedAt time.Time
}

func (r *UserRepository) FindByID(ctx context.Context, id domain.UserID) (*domain.User, error) {
	row := &userRow{}
	err := r.db.QueryRowContext(ctx, `
		SELECT id, email, name, role, active, created_at, updated_at
		FROM users WHERE id = $1
	`, string(id)).Scan(&row.ID, &row.Email, &row.Name, &row.Role, &row.Active, &row.CreatedAt, &row.UpdatedAt)

	if err == sql.ErrNoRows {
		return nil, domain.ErrUserNotFound
	}
	if err != nil {
		return nil, fmt.Errorf("querying user: %w", err)
	}

	return r.toDomain(row)
}

func (r *UserRepository) FindByEmail(ctx context.Context, email domain.Email) (*domain.User, error) {
	row := &userRow{}
	err := r.db.QueryRowContext(ctx, `
		SELECT id, email, name, role, active, created_at, updated_at
		FROM users WHERE email = $1
	`, string(email)).Scan(&row.ID, &row.Email, &row.Name, &row.Role, &row.Active, &row.CreatedAt, &row.UpdatedAt)

	if err == sql.ErrNoRows {
		return nil, domain.ErrUserNotFound
	}
	if err != nil {
		return nil, fmt.Errorf("querying user by email: %w", err)
	}

	return r.toDomain(row)
}

func (r *UserRepository) FindAll(ctx context.Context, filter domain.UserFilter) ([]*domain.User, error) {
	query := `SELECT id, email, name, role, active, created_at, updated_at FROM users WHERE 1=1`
	args := []interface{}{}
	argIdx := 1

	if filter.Role != nil {
		query += fmt.Sprintf(" AND role = $%d", argIdx)
		args = append(args, string(*filter.Role))
		argIdx++
	}
	if filter.Active != nil {
		query += fmt.Sprintf(" AND active = $%d", argIdx)
		args = append(args, *filter.Active)
		argIdx++
	}

	query += fmt.Sprintf(" LIMIT $%d OFFSET $%d", argIdx, argIdx+1)
	args = append(args, filter.Limit, filter.Offset)

	rows, err := r.db.QueryContext(ctx, query, args...)
	if err != nil {
		return nil, fmt.Errorf("querying users: %w", err)
	}
	defer rows.Close()

	var users []*domain.User
	for rows.Next() {
		row := &userRow{}
		if err := rows.Scan(&row.ID, &row.Email, &row.Name, &row.Role, &row.Active, &row.CreatedAt, &row.UpdatedAt); err != nil {
			return nil, fmt.Errorf("scanning user: %w", err)
		}
		user, err := r.toDomain(row)
		if err != nil {
			return nil, err
		}
		users = append(users, user)
	}
	return users, rows.Err()
}

func (r *UserRepository) Save(ctx context.Context, user *domain.User) error {
	_, err := r.db.ExecContext(ctx, `
		INSERT INTO users (id, email, name, role, active, created_at, updated_at)
		VALUES ($1, $2, $3, $4, $5, $6, $7)
		ON CONFLICT (id) DO UPDATE SET
			email = $2, name = $3, role = $4, active = $5, updated_at = $7
	`,
		string(user.ID()),
		string(user.Email()),
		string(user.Name()),
		string(user.Role()),
		user.IsActive(),
		user.CreatedAt(),
		user.UpdatedAt(),
	)
	return err
}

func (r *UserRepository) Delete(ctx context.Context, id domain.UserID) error {
	_, err := r.db.ExecContext(ctx, "DELETE FROM users WHERE id = $1", string(id))
	return err
}

func (r *UserRepository) ExistsByEmail(ctx context.Context, email domain.Email) (bool, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users WHERE email = $1", string(email)).Scan(&count)
	return count > 0, err
}

func (r *UserRepository) toDomain(row *userRow) (*domain.User, error) {
	user, err := domain.NewUser(
		domain.UserID(row.ID),
		domain.Email(row.Email),
		domain.Name(row.Name),
		domain.Role(row.Role),
	)
	if err != nil {
		return nil, fmt.Errorf("creating domain user: %w", err)
	}
	if !row.Active {
		user.Deactivate()
	}
	return user, nil
}
```

```go
// internal/adapters/secondary/redis/cache.go
package redis

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

type RedisCache struct {
	client *redis.Client
}

func NewRedisCache(client *redis.Client) *RedisCache {
	return &RedisCache{client: client}
}

func (c *RedisCache) Get(ctx context.Context, key string) (interface{}, bool) {
	data, err := c.client.Get(ctx, key).Bytes()
	if err != nil {
		return nil, false
	}
	var value interface{}
	if err := json.Unmarshal(data, &value); err != nil {
		return nil, false
	}
	return value, true
}

func (c *RedisCache) Set(ctx context.Context, key string, value interface{}, ttlSeconds int) error {
	data, err := json.Marshal(value)
	if err != nil {
		return fmt.Errorf("marshaling value: %w", err)
	}
	return c.client.Set(ctx, key, data, time.Duration(ttlSeconds)*time.Second).Err()
}

func (c *RedisCache) Delete(ctx context.Context, key string) error {
	return c.client.Del(ctx, key).Err()
}
```

---

## 5. Primary Adapters (HTTP)

```go
// internal/adapters/primary/http/user_handler.go
package http

import (
	"encoding/json"
	"errors"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/example/app/internal/application"
	"github.com/example/app/internal/domain"
)

type UserHandler struct {
	createUser *application.CreateUserUseCase
	getUser    *application.GetUserUseCase
	listUsers  *application.ListUsersUseCase
}

func NewUserHandler(
	createUser *application.CreateUserUseCase,
	getUser *application.GetUserUseCase,
	listUsers *application.ListUsersUseCase,
) *UserHandler {
	return &UserHandler{
		createUser: createUser,
		getUser:    getUser,
		listUsers:  listUsers,
	}
}

func (h *UserHandler) RegisterRoutes(r *mux.Router) {
	r.HandleFunc("/users", h.CreateUser).Methods("POST")
	r.HandleFunc("/users", h.ListUsers).Methods("GET")
	r.HandleFunc("/users/{id}", h.GetUser).Methods("GET")
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Email string `json:"email"`
		Name  string `json:"name"`
		Role  string `json:"role"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	cmd := application.CreateUserCommand{
		Email: req.Email,
		Name:  req.Name,
		Role:  req.Role,
	}

	result, err := h.createUser.Execute(r.Context(), cmd)
	if err != nil {
		if errors.Is(err, domain.ErrEmailAlreadyTaken) {
			respondError(w, http.StatusConflict, err.Error())
			return
		}
		respondError(w, http.StatusInternalServerError, "internal error")
		return
	}

	respondJSON(w, http.StatusCreated, result)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]

	result, err := h.getUser.Execute(r.Context(), application.GetUserQuery{UserID: id})
	if err != nil {
		if errors.Is(err, domain.ErrUserNotFound) {
			respondError(w, http.StatusNotFound, "user not found")
			return
		}
		respondError(w, http.StatusInternalServerError, "internal error")
		return
	}

	respondJSON(w, http.StatusOK, result)
}

func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	
	page := 1
	size := 20

	result, err := h.listUsers.Execute(r.Context(), application.ListUsersQuery{
		Role: q.Get("role"),
		Page: page,
		Size: size,
	})
	if err != nil {
		respondError(w, http.StatusInternalServerError, "internal error")
		return
	}

	respondJSON(w, http.StatusOK, result)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
	respondJSON(w, status, map[string]string{"error": message})
}
```

---

## 6. Dependency Injection (Wire/Manual)

```go
// cmd/server/main.go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/gorilla/mux"
	_ "github.com/lib/pq"
	"github.com/redis/go-redis/v9"
	
	"github.com/example/app/internal/adapters/primary/httphandler"
	"github.com/example/app/internal/adapters/secondary/memory"
	"github.com/example/app/internal/application"
)

type App struct {
	router *mux.Router
	db     *sql.DB
}

func NewApp() (*App, error) {
	// Connect to database
	db, err := connectDB()
	if err != nil {
		return nil, fmt.Errorf("connecting to DB: %w", err)
	}

	// Connect to Redis
	redisClient := connectRedis()

	// Build dependency graph manually (can use wire or fx for larger apps)

	// Secondary adapters (driven)
	userRepo := memory.NewUserRepository() // or postgres.NewUserRepository(db)
	cache := memory.NewCache()              // or redis_adapter.NewRedisCache(redisClient)
	idGen := memory.NewUUIDGenerator()
	emailSender := memory.NewNoOpEmailSender()
	eventPub := memory.NewNoOpEventPublisher()

	// Use cases (application layer)
	createUserUC := application.NewCreateUserUseCase(userRepo, idGen, emailSender, eventPub)
	getUserUC := application.NewGetUserUseCase(userRepo, cache)
	listUsersUC := application.NewListUsersUseCase(userRepo)

	// Primary adapters (driving)
	userHandler := httphandler.NewUserHandler(createUserUC, getUserUC, listUsersUC)

	// Setup router
	r := mux.NewRouter()
	userHandler.RegisterRoutes(r.PathPrefix("/api/v1").Subrouter())

	_ = redisClient

	return &App{router: r, db: db}, nil
}

func (a *App) Run(port string) error {
	log.Printf("Server starting on :%s", port)
	return http.ListenAndServe(":"+port, a.router)
}

func main() {
	app, err := NewApp()
	if err != nil {
		log.Fatalf("Failed to create app: %v", err)
	}

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	if err := app.Run(port); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}

func connectDB() (*sql.DB, error) {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		dsn = "postgres://postgres:password@localhost/appdb?sslmode=disable"
	}
	return sql.Open("postgres", dsn)
}

func connectRedis() *redis.Client {
	addr := os.Getenv("REDIS_ADDR")
	if addr == "" {
		addr = "localhost:6379"
	}
	return redis.NewClient(&redis.Options{Addr: addr})
}
```

---

## 7. Testing with Clean Architecture

```go
// internal/application/create_user_test.go
package application_test

import (
	"context"
	"testing"

	"github.com/example/app/internal/application"
	"github.com/example/app/internal/domain"
)

// Mock implementations (testdoubles)

type MockUserRepository struct {
	users  map[domain.UserID]*domain.User
	emails map[domain.Email]bool
}

func NewMockUserRepository() *MockUserRepository {
	return &MockUserRepository{
		users:  make(map[domain.UserID]*domain.User),
		emails: make(map[domain.Email]bool),
	}
}

func (m *MockUserRepository) FindByID(ctx context.Context, id domain.UserID) (*domain.User, error) {
	if u, ok := m.users[id]; ok {
		return u, nil
	}
	return nil, domain.ErrUserNotFound
}

func (m *MockUserRepository) FindByEmail(ctx context.Context, email domain.Email) (*domain.User, error) {
	for _, u := range m.users {
		if u.Email() == email {
			return u, nil
		}
	}
	return nil, domain.ErrUserNotFound
}

func (m *MockUserRepository) FindAll(ctx context.Context, filter domain.UserFilter) ([]*domain.User, error) {
	var result []*domain.User
	for _, u := range m.users {
		result = append(result, u)
	}
	return result, nil
}

func (m *MockUserRepository) Save(ctx context.Context, user *domain.User) error {
	m.users[user.ID()] = user
	m.emails[user.Email()] = true
	return nil
}

func (m *MockUserRepository) Delete(ctx context.Context, id domain.UserID) error {
	delete(m.users, id)
	return nil
}

func (m *MockUserRepository) ExistsByEmail(ctx context.Context, email domain.Email) (bool, error) {
	return m.emails[email], nil
}

type MockIDGenerator struct {
	counter int
}

func (m *MockIDGenerator) Generate() domain.UserID {
	m.counter++
	return domain.UserID(fmt.Sprintf("user-%d", m.counter))
}

type MockEmailSender struct {
	SentEmails []string
}

func (m *MockEmailSender) SendWelcome(ctx context.Context, to domain.Email, name domain.Name) error {
	m.SentEmails = append(m.SentEmails, string(to))
	return nil
}

func (m *MockEmailSender) SendPasswordReset(ctx context.Context, to domain.Email, token string) error {
	return nil
}

type MockEventPublisher struct {
	Events []map[string]interface{}
}

func (m *MockEventPublisher) Publish(ctx context.Context, topic string, event interface{}) error {
	if e, ok := event.(map[string]interface{}); ok {
		m.Events = append(m.Events, e)
	}
	return nil
}

type MockCache struct {
	data map[string]interface{}
}

func NewMockCache() *MockCache {
	return &MockCache{data: make(map[string]interface{})}
}

func (m *MockCache) Get(ctx context.Context, key string) (interface{}, bool) {
	v, ok := m.data[key]
	return v, ok
}

func (m *MockCache) Set(ctx context.Context, key string, value interface{}, ttl int) error {
	m.data[key] = value
	return nil
}

func (m *MockCache) Delete(ctx context.Context, key string) error {
	delete(m.data, key)
	return nil
}

// Tests
func TestCreateUser_Success(t *testing.T) {
	// Arrange
	repo := NewMockUserRepository()
	idGen := &MockIDGenerator{}
	emailSender := &MockEmailSender{}
	eventPub := &MockEventPublisher{}

	useCase := application.NewCreateUserUseCase(repo, idGen, emailSender, eventPub)

	cmd := application.CreateUserCommand{
		Email: "alice@example.com",
		Name:  "Alice Smith",
		Role:  "user",
	}

	// Act
	result, err := useCase.Execute(context.Background(), cmd)

	// Assert
	if err != nil {
		t.Fatalf("Expected no error, got: %v", err)
	}
	if result.Email != cmd.Email {
		t.Errorf("Expected email %s, got %s", cmd.Email, result.Email)
	}
	if result.Name != cmd.Name {
		t.Errorf("Expected name %s, got %s", cmd.Name, result.Name)
	}
	if !result.Active {
		t.Error("Expected user to be active")
	}
}

func TestCreateUser_EmailAlreadyTaken(t *testing.T) {
	repo := NewMockUserRepository()
	idGen := &MockIDGenerator{}
	emailSender := &MockEmailSender{}
	eventPub := &MockEventPublisher{}

	useCase := application.NewCreateUserUseCase(repo, idGen, emailSender, eventPub)
	ctx := context.Background()

	cmd := application.CreateUserCommand{
		Email: "alice@example.com",
		Name:  "Alice",
		Role:  "user",
	}

	// Create first user
	useCase.Execute(ctx, cmd)

	// Try to create second user with same email
	_, err := useCase.Execute(ctx, cmd)

	if err == nil {
		t.Fatal("Expected error for duplicate email")
	}
	if err != domain.ErrEmailAlreadyTaken {
		t.Errorf("Expected ErrEmailAlreadyTaken, got: %v", err)
	}
}

func TestGetUser_CacheMiss_LoadsFromDB(t *testing.T) {
	repo := NewMockUserRepository()
	cache := NewMockCache()

	// Seed user in repo
	user, _ := domain.NewUser("user-001", "alice@test.com", "Alice", "user")
	repo.Save(context.Background(), user)

	useCase := application.NewGetUserUseCase(repo, cache)

	result, err := useCase.Execute(context.Background(), application.GetUserQuery{
		UserID: "user-001",
	})

	if err != nil {
		t.Fatalf("Unexpected error: %v", err)
	}
	if result.ID != "user-001" {
		t.Errorf("Expected user-001, got %s", result.ID)
	}
}
```

---

## 8. Workshop: Complete Service

```go
// workshop/user_service/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/gorilla/mux"
)

// Domain
type UserID string
type Email string

type User struct {
	id        UserID
	email     Email
	name      string
	active    bool
	createdAt time.Time
}

var ErrUserNotFound = fmt.Errorf("user not found")
var ErrEmailTaken = fmt.Errorf("email already taken")

// Port: Repository
type UserRepo interface {
	FindByID(ctx context.Context, id UserID) (*User, error)
	FindAll(ctx context.Context) ([]*User, error)
	Save(ctx context.Context, u *User) error
	ExistsByEmail(ctx context.Context, e Email) (bool, error)
}

// Port: ID Generator
type IDGen interface {
	Next() UserID
}

// Adapter: In-memory repository
type MemRepo struct {
	mu    sync.RWMutex
	store map[UserID]*User
	cnt   int
}

func NewMemRepo() *MemRepo {
	return &MemRepo{store: make(map[UserID]*User)}
}

func (r *MemRepo) FindByID(ctx context.Context, id UserID) (*User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if u, ok := r.store[id]; ok {
		return u, nil
	}
	return nil, ErrUserNotFound
}

func (r *MemRepo) FindAll(ctx context.Context) ([]*User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	var result []*User
	for _, u := range r.store {
		result = append(result, u)
	}
	return result, nil
}

func (r *MemRepo) Save(ctx context.Context, u *User) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.store[u.id] = u
	return nil
}

func (r *MemRepo) ExistsByEmail(ctx context.Context, e Email) (bool, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	for _, u := range r.store {
		if u.email == e {
			return true, nil
		}
	}
	return false, nil
}

// Adapter: Simple ID generator
type SimpleIDGen struct{ cnt int }

func (g *SimpleIDGen) Next() UserID {
	g.cnt++
	return UserID(fmt.Sprintf("usr-%06d", g.cnt))
}

// Application: Use cases
type CreateUserUC struct {
	repo UserRepo
	gen  IDGen
}

func (uc *CreateUserUC) Execute(ctx context.Context, email, name string) (*User, error) {
	exists, _ := uc.repo.ExistsByEmail(ctx, Email(email))
	if exists {
		return nil, ErrEmailTaken
	}
	u := &User{
		id:        uc.gen.Next(),
		email:     Email(email),
		name:      name,
		active:    true,
		createdAt: time.Now(),
	}
	return u, uc.repo.Save(ctx, u)
}

// HTTP Handler (Primary adapter)
type Handler struct {
	createUC *CreateUserUC
	repo     UserRepo
}

func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Email string `json:"email"`
		Name  string `json:"name"`
	}
	json.NewDecoder(r.Body).Decode(&req)
	user, err := h.createUC.Execute(r.Context(), req.Email, req.Name)
	if err != nil {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusConflict)
		json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]interface{}{
		"id":         user.id,
		"email":      user.email,
		"name":       user.name,
		"active":     user.active,
		"created_at": user.createdAt,
	})
}

func (h *Handler) ListUsers(w http.ResponseWriter, r *http.Request) {
	users, _ := h.repo.FindAll(r.Context())
	var result []map[string]interface{}
	for _, u := range users {
		result = append(result, map[string]interface{}{
			"id": u.id, "email": u.email, "name": u.name,
		})
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(result)
}

func main() {
	// Wire dependencies
	repo := NewMemRepo()
	gen := &SimpleIDGen{}
	createUC := &CreateUserUC{repo: repo, gen: gen}
	handler := &Handler{createUC: createUC, repo: repo}

	// Setup routes
	r := mux.NewRouter()
	r.HandleFunc("/api/users", handler.CreateUser).Methods("POST")
	r.HandleFunc("/api/users", handler.ListUsers).Methods("GET")
	r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	}).Methods("GET")

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	srv := &http.Server{Addr: ":" + port, Handler: r}

	go func() {
		log.Printf("Hexagonal service starting on port %s", port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	srv.Shutdown(ctx)
	log.Println("Server stopped")
}
```

---

## สรุป

| Layer | ประกอบด้วย | Depends On |
|-------|-----------|-----------|
| Domain | Entities, VOs, Domain Services, Repository interfaces | ไม่ depend ใคร |
| Application | Use Cases, Application Services | Domain only |
| Primary Adapters | HTTP handlers, CLI, gRPC | Application |
| Secondary Adapters | DB implementations, Email, Cache | Domain interfaces |

### Key Principles
1. **Dependency Rule**: dependencies ชี้เข้าหา domain เท่านั้น
2. **Ports**: interfaces ใน domain/application layer
3. **Adapters**: implementations ใน infrastructure layer
4. **Testability**: สามารถ test ทุก layer แยกกันได้

---

**ต่อไป**: Part 61 - gRPC Advanced
