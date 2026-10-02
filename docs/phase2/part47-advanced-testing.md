# Part 47: Advanced Testing ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- สร้าง Mocks, Stubs, Fakes
- ใช้ gomock และ testify/mock
- ทำ HTTP testing ด้วย httptest
- ทำ Database testing ด้วย sqlmock
- เขียน Integration tests
- เขียน End-to-End tests
- Property-based testing
- Fuzz testing

---

## 1. Mocks, Stubs, Fakes

```go
// ตัวอย่าง 1: Understanding test doubles

package testing_examples

import (
	"context"
	"errors"
	"fmt"
)

// Interface to test
type UserRepository interface {
	GetByID(ctx context.Context, id int) (*User, error)
	Save(ctx context.Context, user *User) error
	Delete(ctx context.Context, id int) error
	List(ctx context.Context) ([]*User, error)
}

type User struct {
	ID    int
	Name  string
	Email string
}

// Stub: Returns hardcoded data
type StubUserRepo struct{}

func (s *StubUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
	return &User{ID: id, Name: "Test User", Email: "test@example.com"}, nil
}

func (s *StubUserRepo) Save(ctx context.Context, user *User) error { return nil }
func (s *StubUserRepo) Delete(ctx context.Context, id int) error   { return nil }
func (s *StubUserRepo) List(ctx context.Context) ([]*User, error) {
	return []*User{{ID: 1, Name: "Test"}}, nil
}

// Fake: Working implementation (in-memory)
type FakeUserRepo struct {
	users map[int]*User
	nextID int
}

func NewFakeUserRepo() *FakeUserRepo {
	return &FakeUserRepo{
		users:  make(map[int]*User),
		nextID: 1,
	}
}

func (f *FakeUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
	user, ok := f.users[id]
	if !ok {
		return nil, fmt.Errorf("user not found: %d", id)
	}
	return user, nil
}

func (f *FakeUserRepo) Save(ctx context.Context, user *User) error {
	if user.ID == 0 {
		user.ID = f.nextID
		f.nextID++
	}
	f.users[user.ID] = user
	return nil
}

func (f *FakeUserRepo) Delete(ctx context.Context, id int) error {
	delete(f.users, id)
	return nil
}

func (f *FakeUserRepo) List(ctx context.Context) ([]*User, error) {
	users := make([]*User, 0, len(f.users))
	for _, u := range f.users {
		users = append(users, u)
	}
	return users, nil
}

// Service to test
type UserService struct {
	repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}

func (s *UserService) GetUser(ctx context.Context, id int) (*User, error) {
	if id <= 0 {
		return nil, errors.New("invalid user ID")
	}
	return s.repo.GetByID(ctx, id)
}

func (s *UserService) CreateUser(ctx context.Context, name, email string) (*User, error) {
	if name == "" {
		return nil, errors.New("name is required")
	}
	if email == "" {
		return nil, errors.New("email is required")
	}
	
	user := &User{Name: name, Email: email}
	if err := s.repo.Save(ctx, user); err != nil {
		return nil, fmt.Errorf("saving user: %w", err)
	}
	return user, nil
}
```

---

## 2. Manual Mock Implementation

```go
// ตัวอย่าง 2: Manual mock with call tracking

package testing_examples

import (
	"context"
	"sync"
	"testing"
)

type MockUserRepo struct {
	mu sync.Mutex
	
	// Track calls
	GetByIDCalls []int
	SaveCalls    []*User
	DeleteCalls  []int
	ListCalls    int
	
	// Configure returns
	GetByIDFunc func(ctx context.Context, id int) (*User, error)
	SaveFunc    func(ctx context.Context, user *User) error
	DeleteFunc  func(ctx context.Context, id int) error
	ListFunc    func(ctx context.Context) ([]*User, error)
}

func (m *MockUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
	m.mu.Lock()
	m.GetByIDCalls = append(m.GetByIDCalls, id)
	m.mu.Unlock()
	
	if m.GetByIDFunc != nil {
		return m.GetByIDFunc(ctx, id)
	}
	return nil, nil
}

func (m *MockUserRepo) Save(ctx context.Context, user *User) error {
	m.mu.Lock()
	m.SaveCalls = append(m.SaveCalls, user)
	m.mu.Unlock()
	
	if m.SaveFunc != nil {
		return m.SaveFunc(ctx, user)
	}
	return nil
}

func (m *MockUserRepo) Delete(ctx context.Context, id int) error {
	m.mu.Lock()
	m.DeleteCalls = append(m.DeleteCalls, id)
	m.mu.Unlock()
	
	if m.DeleteFunc != nil {
		return m.DeleteFunc(ctx, id)
	}
	return nil
}

func (m *MockUserRepo) List(ctx context.Context) ([]*User, error) {
	m.mu.Lock()
	m.ListCalls++
	m.mu.Unlock()
	
	if m.ListFunc != nil {
		return m.ListFunc(ctx)
	}
	return nil, nil
}

func TestUserService_GetUser_WithMock(t *testing.T) {
	mock := &MockUserRepo{
		GetByIDFunc: func(ctx context.Context, id int) (*User, error) {
			return &User{ID: id, Name: "Alice", Email: "alice@test.com"}, nil
		},
	}
	
	service := NewUserService(mock)
	
	user, err := service.GetUser(context.Background(), 1)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	
	if user.ID != 1 {
		t.Errorf("expected ID 1, got %d", user.ID)
	}
	
	if len(mock.GetByIDCalls) != 1 || mock.GetByIDCalls[0] != 1 {
		t.Error("expected GetByID to be called with ID 1")
	}
}

func TestUserService_CreateUser(t *testing.T) {
	tests := []struct {
		name      string
		userName  string
		email     string
		repoErr   error
		wantErr   bool
		errMsg    string
	}{
		{
			name:     "success",
			userName: "Alice",
			email:    "alice@test.com",
			wantErr:  false,
		},
		{
			name:    "empty name",
			email:   "alice@test.com",
			wantErr: true,
			errMsg:  "name is required",
		},
		{
			name:     "empty email",
			userName: "Alice",
			wantErr:  true,
			errMsg:   "email is required",
		},
		{
			name:     "repo error",
			userName: "Alice",
			email:    "alice@test.com",
			repoErr:  errors.New("db error"),
			wantErr:  true,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			mock := &MockUserRepo{
				SaveFunc: func(ctx context.Context, user *User) error {
					return tt.repoErr
				},
			}
			
			service := NewUserService(mock)
			user, err := service.CreateUser(context.Background(), tt.userName, tt.email)
			
			if tt.wantErr {
				if err == nil {
					t.Error("expected error but got none")
				}
				if tt.errMsg != "" && !strings.Contains(err.Error(), tt.errMsg) {
					t.Errorf("expected error containing %q, got %q", tt.errMsg, err.Error())
				}
				return
			}
			
			if err != nil {
				t.Fatalf("unexpected error: %v", err)
			}
			if user == nil {
				t.Fatal("expected user, got nil")
			}
		})
	}
}
```

---

## 3. testify/mock

```go
// ตัวอย่าง 3: Using testify/mock

package testing_examples

import (
	"context"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"
)

// MockRepo using testify/mock
type TestifyMockUserRepo struct {
	mock.Mock
}

func (m *TestifyMockUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
	args := m.Called(ctx, id)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*User), args.Error(1)
}

func (m *TestifyMockUserRepo) Save(ctx context.Context, user *User) error {
	args := m.Called(ctx, user)
	return args.Error(0)
}

func (m *TestifyMockUserRepo) Delete(ctx context.Context, id int) error {
	args := m.Called(ctx, id)
	return args.Error(0)
}

func (m *TestifyMockUserRepo) List(ctx context.Context) ([]*User, error) {
	args := m.Called(ctx)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).([]*User), args.Error(1)
}

func TestWithTestifyMock(t *testing.T) {
	mockRepo := &TestifyMockUserRepo{}
	
	// Setup expectations
	expectedUser := &User{ID: 1, Name: "Alice", Email: "alice@test.com"}
	mockRepo.On("GetByID", mock.Anything, 1).Return(expectedUser, nil)
	mockRepo.On("GetByID", mock.Anything, 999).Return(nil, errors.New("not found"))
	
	service := NewUserService(mockRepo)
	
	// Test success case
	user, err := service.GetUser(context.Background(), 1)
	require.NoError(t, err)
	assert.Equal(t, expectedUser, user)
	
	// Test not found case
	_, err = service.GetUser(context.Background(), 999)
	assert.Error(t, err)
	
	// Verify all expectations were met
	mockRepo.AssertExpectations(t)
}

// ตัวอย่าง 4: Mock with Times and Once

func TestMockCallCount(t *testing.T) {
	mockRepo := &TestifyMockUserRepo{}
	
	// Expect called exactly twice
	mockRepo.On("GetByID", mock.Anything, mock.AnythingOfType("int")).
		Return(&User{ID: 1, Name: "Test"}, nil).
		Times(2)
	
	// Expect called once with specific args
	mockRepo.On("Delete", mock.Anything, 1).
		Return(nil).
		Once()
	
	service := NewUserService(mockRepo)
	ctx := context.Background()
	
	service.GetUser(ctx, 1)
	service.GetUser(ctx, 2)
	mockRepo.Delete(ctx, 1)
	
	mockRepo.AssertExpectations(t)
	mockRepo.AssertNumberOfCalls(t, "GetByID", 2)
}
```

---

## 4. HTTP Testing with httptest

```go
// ตัวอย่าง 5: Testing HTTP handlers

package handlers_test

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

// Handler to test
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

type CreateUserResponse struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
	var req CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request", http.StatusBadRequest)
		return
	}
	
	if req.Name == "" {
		http.Error(w, "name required", http.StatusBadRequest)
		return
	}
	
	resp := CreateUserResponse{
		ID:    1,
		Name:  req.Name,
		Email: req.Email,
	}
	
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(resp)
}

func TestCreateUserHandler(t *testing.T) {
	tests := []struct {
		name           string
		body           interface{}
		expectedStatus int
		expectedName   string
	}{
		{
			name:           "success",
			body:           CreateUserRequest{Name: "Alice", Email: "alice@test.com"},
			expectedStatus: http.StatusCreated,
			expectedName:   "Alice",
		},
		{
			name:           "empty name",
			body:           CreateUserRequest{Email: "alice@test.com"},
			expectedStatus: http.StatusBadRequest,
		},
		{
			name:           "invalid json",
			body:           "not json",
			expectedStatus: http.StatusBadRequest,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			body, _ := json.Marshal(tt.body)
			req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewReader(body))
			req.Header.Set("Content-Type", "application/json")
			
			w := httptest.NewRecorder()
			createUserHandler(w, req)
			
			resp := w.Result()
			assert.Equal(t, tt.expectedStatus, resp.StatusCode)
			
			if tt.expectedName != "" {
				var result CreateUserResponse
				require.NoError(t, json.NewDecoder(resp.Body).Decode(&result))
				assert.Equal(t, tt.expectedName, result.Name)
			}
		})
	}
}

// ตัวอย่าง 6: Testing with real HTTP server

func TestWithHTTPServer(t *testing.T) {
	// Create a real test server
	mux := http.NewServeMux()
	mux.HandleFunc("/users", createUserHandler)
	
	server := httptest.NewServer(mux)
	defer server.Close()
	
	// Make real HTTP requests
	body := `{"name":"Bob","email":"bob@test.com"}`
	resp, err := http.Post(server.URL+"/users", "application/json",
		bytes.NewBufferString(body))
	require.NoError(t, err)
	defer resp.Body.Close()
	
	assert.Equal(t, http.StatusCreated, resp.StatusCode)
	
	var result CreateUserResponse
	require.NoError(t, json.NewDecoder(resp.Body).Decode(&result))
	assert.Equal(t, "Bob", result.Name)
}
```

---

## 5. Database Testing with sqlmock

```go
// ตัวอย่าง 7: SQL mock testing

package db_test

import (
	"context"
	"database/sql"
	"testing"
	"time"
	
	"github.com/DATA-DOG/go-sqlmock"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

type PostgresUserRepo struct {
	db *sql.DB
}

func (r *PostgresUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
	query := "SELECT id, name, email, created_at FROM users WHERE id = $1"
	
	var user User
	var createdAt time.Time
	
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&user.ID, &user.Name, &user.Email, &createdAt,
	)
	if err == sql.ErrNoRows {
		return nil, fmt.Errorf("user not found: %d", id)
	}
	if err != nil {
		return nil, fmt.Errorf("querying user: %w", err)
	}
	
	return &user, nil
}

func (r *PostgresUserRepo) Save(ctx context.Context, user *User) error {
	query := `
		INSERT INTO users (name, email) VALUES ($1, $2)
		ON CONFLICT (email) DO UPDATE SET name = $1
		RETURNING id`
	
	return r.db.QueryRowContext(ctx, query, user.Name, user.Email).Scan(&user.ID)
}

func (r *PostgresUserRepo) Delete(ctx context.Context, id int) error {
	_, err := r.db.ExecContext(ctx, "DELETE FROM users WHERE id = $1", id)
	return err
}

func TestPostgresUserRepo_GetByID(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()
	
	repo := &PostgresUserRepo{db: db}
	
	t.Run("success", func(t *testing.T) {
		rows := sqlmock.NewRows([]string{"id", "name", "email", "created_at"}).
			AddRow(1, "Alice", "alice@test.com", time.Now())
		
		mock.ExpectQuery("SELECT id, name, email, created_at FROM users").
			WithArgs(1).
			WillReturnRows(rows)
		
		user, err := repo.GetByID(context.Background(), 1)
		require.NoError(t, err)
		assert.Equal(t, 1, user.ID)
		assert.Equal(t, "Alice", user.Name)
	})
	
	t.Run("not found", func(t *testing.T) {
		mock.ExpectQuery("SELECT id, name, email, created_at FROM users").
			WithArgs(999).
			WillReturnError(sql.ErrNoRows)
		
		_, err := repo.GetByID(context.Background(), 999)
		assert.Error(t, err)
		assert.Contains(t, err.Error(), "not found")
	})
	
	// Verify all expectations met
	assert.NoError(t, mock.ExpectationsWereMet())
}

func TestPostgresUserRepo_Save(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()
	
	repo := &PostgresUserRepo{db: db}
	
	rows := sqlmock.NewRows([]string{"id"}).AddRow(42)
	
	mock.ExpectQuery(`INSERT INTO users`).
		WithArgs("Alice", "alice@test.com").
		WillReturnRows(rows)
	
	user := &User{Name: "Alice", Email: "alice@test.com"}
	err = repo.Save(context.Background(), user)
	require.NoError(t, err)
	assert.Equal(t, 42, user.ID)
	
	assert.NoError(t, mock.ExpectationsWereMet())
}

// ตัวอย่าง 8: Testing transactions

func TestTransactionRollback(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()
	
	mock.ExpectBegin()
	mock.ExpectExec("INSERT INTO orders").
		WithArgs(1, 100.0).
		WillReturnResult(sqlmock.NewResult(1, 1))
	mock.ExpectExec("UPDATE inventory").
		WithArgs(1).
		WillReturnError(errors.New("not enough stock"))
	mock.ExpectRollback()
	
	// Run function that uses transaction
	err = createOrder(context.Background(), db, 1, 100.0)
	assert.Error(t, err)
	assert.Contains(t, err.Error(), "not enough stock")
	
	assert.NoError(t, mock.ExpectationsWereMet())
}

func createOrder(ctx context.Context, db *sql.DB, userID int, amount float64) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	
	_, err = tx.ExecContext(ctx, "INSERT INTO orders VALUES($1, $2)", userID, amount)
	if err != nil {
		return err
	}
	
	_, err = tx.ExecContext(ctx, "UPDATE inventory SET stock = stock - 1 WHERE id = $1", userID)
	if err != nil {
		return fmt.Errorf("not enough stock: %w", err)
	}
	
	return tx.Commit()
}
```

---

## 6. Integration Tests

```go
// ตัวอย่าง 9: Integration test with real database

//go:build integration
// +build integration

package integration_test

import (
	"context"
	"database/sql"
	"os"
	"testing"
	
	_ "github.com/lib/pq"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func setupTestDB(t *testing.T) *sql.DB {
	t.Helper()
	
	dsn := os.Getenv("TEST_DATABASE_URL")
	if dsn == "" {
		dsn = "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
	}
	
	db, err := sql.Open("postgres", dsn)
	require.NoError(t, err)
	require.NoError(t, db.Ping())
	
	// Run migrations
	_, err = db.Exec(`
		CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			name VARCHAR(255) NOT NULL,
			email VARCHAR(255) UNIQUE NOT NULL,
			created_at TIMESTAMP DEFAULT NOW()
		)
	`)
	require.NoError(t, err)
	
	// Cleanup after test
	t.Cleanup(func() {
		db.Exec("TRUNCATE TABLE users RESTART IDENTITY")
		db.Close()
	})
	
	return db
}

func TestUserRepo_Integration(t *testing.T) {
	db := setupTestDB(t)
	repo := &PostgresUserRepo{db: db}
	ctx := context.Background()
	
	t.Run("create and get", func(t *testing.T) {
		user := &User{Name: "Alice", Email: "alice@test.com"}
		err := repo.Save(ctx, user)
		require.NoError(t, err)
		assert.Greater(t, user.ID, 0)
		
		fetched, err := repo.GetByID(ctx, user.ID)
		require.NoError(t, err)
		assert.Equal(t, user.Name, fetched.Name)
		assert.Equal(t, user.Email, fetched.Email)
	})
	
	t.Run("delete", func(t *testing.T) {
		user := &User{Name: "Bob", Email: "bob@test.com"}
		require.NoError(t, repo.Save(ctx, user))
		
		require.NoError(t, repo.Delete(ctx, user.ID))
		
		_, err := repo.GetByID(ctx, user.ID)
		assert.Error(t, err)
	})
}
```

---

## 7. Test Helpers and Fixtures

```go
// ตัวอย่าง 10: Test helpers

package testing_examples

import (
	"testing"
	"time"
)

// Test builder pattern
type UserBuilder struct {
	user User
}

func NewUserBuilder() *UserBuilder {
	return &UserBuilder{
		user: User{
			Name:  "Default User",
			Email: "default@test.com",
		},
	}
}

func (b *UserBuilder) WithName(name string) *UserBuilder {
	b.user.Name = name
	return b
}

func (b *UserBuilder) WithEmail(email string) *UserBuilder {
	b.user.Email = email
	return b
}

func (b *UserBuilder) WithID(id int) *UserBuilder {
	b.user.ID = id
	return b
}

func (b *UserBuilder) Build() *User {
	u := b.user
	return &u
}

func TestWithBuilder(t *testing.T) {
	user := NewUserBuilder().
		WithName("Alice").
		WithEmail("alice@test.com").
		WithID(1).
		Build()
	
	assert.Equal(t, "Alice", user.Name)
}

// Test clock for time-dependent tests
type Clock interface {
	Now() time.Time
}

type RealClock struct{}

func (c RealClock) Now() time.Time { return time.Now() }

type MockClock struct {
	current time.Time
}

func NewMockClock(t time.Time) *MockClock {
	return &MockClock{current: t}
}

func (m *MockClock) Now() time.Time { return m.current }

func (m *MockClock) Advance(d time.Duration) {
	m.current = m.current.Add(d)
}

// Service using clock
type TokenService struct {
	clock  Clock
	expiry time.Duration
}

func (s *TokenService) IsExpired(createdAt time.Time) bool {
	return s.clock.Now().After(createdAt.Add(s.expiry))
}

func TestTokenExpiry(t *testing.T) {
	fixedTime := time.Date(2024, 1, 1, 12, 0, 0, 0, time.UTC)
	clock := NewMockClock(fixedTime)
	
	svc := &TokenService{clock: clock, expiry: time.Hour}
	
	createdAt := fixedTime.Add(-30 * time.Minute)
	assert.False(t, svc.IsExpired(createdAt), "should not be expired after 30 min")
	
	clock.Advance(time.Hour)
	assert.True(t, svc.IsExpired(createdAt), "should be expired after 90 min")
}
```

---

## 8. End-to-End Tests

```go
// ตัวอย่าง 11: E2E test with real HTTP server

//go:build e2e
// +build e2e

package e2e_test

import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

var baseURL string

func TestMain(m *testing.M) {
	baseURL = os.Getenv("E2E_BASE_URL")
	if baseURL == "" {
		baseURL = "http://localhost:8080"
	}
	
	// Wait for server to be ready
	if err := waitForServer(baseURL+"/health", 30); err != nil {
		fmt.Println("Server not ready:", err)
		os.Exit(1)
	}
	
	os.Exit(m.Run())
}

func waitForServer(url string, attempts int) error {
	for i := 0; i < attempts; i++ {
		resp, err := http.Get(url)
		if err == nil && resp.StatusCode == 200 {
			return nil
		}
		time.Sleep(time.Second)
	}
	return fmt.Errorf("server not ready after %d attempts", attempts)
}

func TestUserCRUD_E2E(t *testing.T) {
	client := &http.Client{}
	
	// Create user
	createBody, _ := json.Marshal(map[string]string{
		"name":  "E2E Test User",
		"email": "e2e@test.com",
	})
	
	req, _ := http.NewRequest(http.MethodPost, baseURL+"/api/users",
		bytes.NewReader(createBody))
	req.Header.Set("Content-Type", "application/json")
	
	resp, err := client.Do(req)
	require.NoError(t, err)
	assert.Equal(t, http.StatusCreated, resp.StatusCode)
	
	var created struct {
		ID int `json:"id"`
	}
	json.NewDecoder(resp.Body).Decode(&created)
	resp.Body.Close()
	
	require.Greater(t, created.ID, 0)
	
	// Get user
	resp, err = http.Get(fmt.Sprintf("%s/api/users/%d", baseURL, created.ID))
	require.NoError(t, err)
	assert.Equal(t, http.StatusOK, resp.StatusCode)
	resp.Body.Close()
	
	// Delete user
	req, _ = http.NewRequest(http.MethodDelete,
		fmt.Sprintf("%s/api/users/%d", baseURL, created.ID), nil)
	resp, err = client.Do(req)
	require.NoError(t, err)
	assert.Equal(t, http.StatusNoContent, resp.StatusCode)
	resp.Body.Close()
}
```

---

## 9. Property-based Testing

```go
// ตัวอย่าง 12: Property-based testing with gopter

package testing_examples

import (
	"testing"
	"unicode/utf8"
	
	"github.com/leanovate/gopter"
	"github.com/leanovate/gopter/gen"
	"github.com/leanovate/gopter/prop"
)

// Function to test
func reverseString(s string) string {
	runes := []rune(s)
	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
		runes[i], runes[j] = runes[j], runes[i]
	}
	return string(runes)
}

func TestReverseString_Properties(t *testing.T) {
	properties := gopter.NewProperties(nil)
	
	// Property 1: Double reverse = original
	properties.Property("double reverse equals original", prop.ForAll(
		func(s string) bool {
			return reverseString(reverseString(s)) == s
		},
		gen.AnyString(),
	))
	
	// Property 2: Length preserved
	properties.Property("length preserved", prop.ForAll(
		func(s string) bool {
			return utf8.RuneCountInString(reverseString(s)) == utf8.RuneCountInString(s)
		},
		gen.AnyString(),
	))
	
	// Property 3: Empty string reversed is empty
	properties.Property("empty string", prop.ForAll(
		func() bool {
			return reverseString("") == ""
		},
	))
	
	properties.TestingRun(t)
}

// ตัวอย่าง 13: Property testing for sort
func TestSort_Properties(t *testing.T) {
	properties := gopter.NewProperties(nil)
	
	properties.Property("sorted list is ordered", prop.ForAll(
		func(nums []int) bool {
			sorted := sortInts(nums)
			for i := 1; i < len(sorted); i++ {
				if sorted[i] < sorted[i-1] {
					return false
				}
			}
			return true
		},
		gen.SliceOf(gen.Int()),
	))
	
	properties.Property("sorted has same length", prop.ForAll(
		func(nums []int) bool {
			return len(sortInts(nums)) == len(nums)
		},
		gen.SliceOf(gen.Int()),
	))
	
	properties.TestingRun(t)
}

func sortInts(nums []int) []int {
	result := make([]int, len(nums))
	copy(result, nums)
	sort.Ints(result)
	return result
}
```

---

## 10. Fuzz Testing

```go
// ตัวอย่าง 14: Fuzz testing (Go 1.18+)

package testing_examples

import (
	"encoding/json"
	"testing"
	"unicode/utf8"
)

// Function to fuzz
func parseUserInput(input string) (string, error) {
	if !utf8.ValidString(input) {
		return "", fmt.Errorf("invalid UTF-8")
	}
	if len(input) > 1000 {
		return "", fmt.Errorf("input too long")
	}
	// Process input
	return strings.TrimSpace(input), nil
}

func FuzzParseUserInput(f *testing.F) {
	// Seed corpus
	f.Add("")
	f.Add("hello")
	f.Add("hello world")
	f.Add("   spaces   ")
	f.Add("\n\t\r")
	f.Add("unicode: 日本語")
	
	f.Fuzz(func(t *testing.T, input string) {
		result, err := parseUserInput(input)
		
		if err != nil {
			// Errors are allowed, but should not panic
			return
		}
		
		// Properties that should always hold
		if len(result) > len(input) {
			t.Errorf("result longer than input: %d > %d", len(result), len(input))
		}
		
		if !utf8.ValidString(result) {
			t.Error("result is not valid UTF-8")
		}
		
		// Idempotent: applying trim twice gives same result
		if result2, _ := parseUserInput(result); result2 != result {
			t.Errorf("not idempotent: %q -> %q -> %q", input, result, result2)
		}
	})
}

// ตัวอย่าง 15: Fuzzing JSON parser

func FuzzJSONRoundTrip(f *testing.F) {
	f.Add(`{"name":"Alice","age":30}`)
	f.Add(`{}`)
	f.Add(`{"nested":{"key":"value"}}`)
	
	f.Fuzz(func(t *testing.T, data string) {
		// Parse JSON
		var m map[string]interface{}
		if err := json.Unmarshal([]byte(data), &m); err != nil {
			return // Invalid JSON is OK
		}
		
		// Marshal back
		encoded, err := json.Marshal(m)
		if err != nil {
			t.Errorf("failed to marshal valid JSON: %v", err)
			return
		}
		
		// Parse again
		var m2 map[string]interface{}
		if err := json.Unmarshal(encoded, &m2); err != nil {
			t.Errorf("failed to parse re-encoded JSON: %v", err)
		}
	})
}
```

---

## 11. Test Organization

```go
// ตัวอย่าง 16: Test organization with subtests

package testing_examples

import (
	"testing"
)

func TestUserService(t *testing.T) {
	// Shared setup
	repo := NewFakeUserRepo()
	service := NewUserService(repo)
	ctx := context.Background()
	
	t.Run("CreateUser", func(t *testing.T) {
		t.Run("success", func(t *testing.T) {
			user, err := service.CreateUser(ctx, "Alice", "alice@test.com")
			require.NoError(t, err)
			assert.Equal(t, "Alice", user.Name)
		})
		
		t.Run("validation", func(t *testing.T) {
			t.Run("empty name", func(t *testing.T) {
				_, err := service.CreateUser(ctx, "", "alice@test.com")
				assert.ErrorContains(t, err, "name is required")
			})
			
			t.Run("empty email", func(t *testing.T) {
				_, err := service.CreateUser(ctx, "Alice", "")
				assert.ErrorContains(t, err, "email is required")
			})
		})
	})
	
	t.Run("GetUser", func(t *testing.T) {
		// Create test user
		user, _ := service.CreateUser(ctx, "Bob", "bob@test.com")
		
		t.Run("success", func(t *testing.T) {
			fetched, err := service.GetUser(ctx, user.ID)
			require.NoError(t, err)
			assert.Equal(t, user.ID, fetched.ID)
		})
		
		t.Run("invalid id", func(t *testing.T) {
			_, err := service.GetUser(ctx, -1)
			assert.ErrorContains(t, err, "invalid user ID")
		})
	})
}

// ตัวอย่าง 17: Parallel tests

func TestParallel(t *testing.T) {
	tests := []struct {
		name  string
		input int
		want  int
	}{
		{"zero", 0, 0},
		{"positive", 5, 25},
		{"negative", -3, 9},
	}
	
	for _, tt := range tests {
		tt := tt // Capture for parallel
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel() // Run subtests in parallel
			
			result := square(tt.input)
			assert.Equal(t, tt.want, result)
		})
	}
}

func square(n int) int {
	return n * n
}
```

---

## 12. Test Coverage Analysis

```bash
# ตัวอย่าง 18: Coverage commands

# Run tests with coverage
go test -coverprofile=coverage.out ./...

# Show coverage by function
go tool cover -func=coverage.out

# Generate HTML report
go tool cover -html=coverage.out -o coverage.html

# Run specific package with coverage
go test -cover ./internal/...

# Show coverage in terminal
go test -v -coverprofile=coverage.out ./... && go tool cover -func=coverage.out | tail -1
```

---

## สรุป

ใน Part 47 เราได้เรียนรู้:

1. **Test Doubles**: Stub, Fake, Mock — ความแตกต่างและการใช้งาน
2. **Manual Mocks**: สร้าง mock ด้วยตัวเอง
3. **testify/mock**: Library สำหรับ mocking
4. **httptest**: ทดสอบ HTTP handlers
5. **sqlmock**: ทดสอบ database queries
6. **Integration Tests**: ทดสอบกับ real database
7. **Test Helpers**: Builder pattern, Mock Clock
8. **E2E Tests**: ทดสอบ end-to-end ด้วย real server
9. **Property-based Testing**: หา edge cases อัตโนมัติ
10. **Fuzz Testing**: Go 1.18+ built-in fuzzing
11. **Test Organization**: subtests, parallel tests

---

## Resources

- [testify](https://github.com/stretchr/testify)
- [gomock](https://github.com/uber-go/mock)
- [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)
- [gopter](https://github.com/leanovate/gopter)
- [Go Fuzzing](https://go.dev/doc/fuzz/)
- [Testing in Go - Official](https://pkg.go.dev/testing)
