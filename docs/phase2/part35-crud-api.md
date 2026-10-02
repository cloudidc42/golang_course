# Part 35: Complete CRUD API Project

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Complete CRUD API ด้วย Clean Architecture
- แยก layer ออกเป็น Models, Repositories, Services, Handlers
- ทำ Validation อย่างครบถ้วน
- ส่ง Error responses ที่สม่ำเสมอ
- เขียน API documentation ด้วย Swagger
- ทดสอบ API

---

## 35.1 Project Structure

```
ecommerce-api/
├── main.go
├── go.mod
├── go.sum
├── config/
│   └── config.go
├── models/
│   ├── user.go
│   └── product.go
├── repository/
│   ├── user_repository.go
│   └── product_repository.go
├── service/
│   ├── user_service.go
│   └── product_service.go
├── handler/
│   ├── user_handler.go
│   └── product_handler.go
├── middleware/
│   ├── auth.go
│   ├── cors.go
│   └── logger.go
├── router/
│   └── router.go
├── errors/
│   └── errors.go
└── utils/
    ├── response.go
    └── validation.go
```

---

## 35.2 Models

### User Model (ตัวอย่างที่ 1)

```go
// models/user.go
package models

import "time"

// User represents a user in the system
// @Description User account information
type User struct {
    ID           string     `json:"id" gorm:"primaryKey;type:varchar(36)"`
    Name         string     `json:"name" gorm:"not null;size:100"`
    Email        string     `json:"email" gorm:"uniqueIndex;not null;size:255"`
    PasswordHash string     `json:"-" gorm:"not null"`
    Role         string     `json:"role" gorm:"default:user;size:50"`
    Active       bool       `json:"active" gorm:"default:true"`
    CreatedAt    time.Time  `json:"created_at"`
    UpdatedAt    time.Time  `json:"updated_at"`
    DeletedAt    *time.Time `json:"-" gorm:"index"`
}

// UserProfile is the public user profile (no sensitive data)
type UserProfile struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Role      string    `json:"role"`
    Active    bool      `json:"active"`
    CreatedAt time.Time `json:"created_at"`
}

// ToProfile converts User to UserProfile
func (u *User) ToProfile() UserProfile {
    return UserProfile{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Role:      u.Role,
        Active:    u.Active,
        CreatedAt: u.CreatedAt,
    }
}

// Request/Response types
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=100" example:"John Doe"`
    Email    string `json:"email" binding:"required,email" example:"john@example.com"`
    Password string `json:"password" binding:"required,min=8,max=72" example:"SecurePass123"`
}

type UpdateUserRequest struct {
    Name   *string `json:"name,omitempty" binding:"omitempty,min=2,max=100"`
    Email  *string `json:"email,omitempty" binding:"omitempty,email"`
    Active *bool   `json:"active,omitempty"`
}

type ChangePasswordRequest struct {
    CurrentPassword string `json:"current_password" binding:"required"`
    NewPassword     string `json:"new_password" binding:"required,min=8,max=72"`
    ConfirmPassword string `json:"confirm_password" binding:"required"`
}

type UserListResponse struct {
    Users      []UserProfile `json:"users"`
    TotalItems int64         `json:"total_items"`
    Page       int           `json:"page"`
    Limit      int           `json:"limit"`
    TotalPages int           `json:"total_pages"`
}

type UserFilter struct {
    Search string `form:"search"`
    Role   string `form:"role"`
    Active *bool  `form:"active"`
    Page   int    `form:"page,default=1" binding:"min=1"`
    Limit  int    `form:"limit,default=10" binding:"min=1,max=100"`
    SortBy string `form:"sort_by,default=created_at"`
    Order  string `form:"order,default=desc" binding:"oneof=asc desc"`
}
```

### Product Model (ตัวอย่างที่ 2)

```go
// models/product.go
package models

import "time"

// Product represents a product in the store
// @Description Product information
type Product struct {
    ID          string     `json:"id" gorm:"primaryKey;type:varchar(36)"`
    Name        string     `json:"name" gorm:"not null;size:200;index"`
    Description string     `json:"description" gorm:"type:text"`
    Price       float64    `json:"price" gorm:"not null;check:price >= 0"`
    Stock       int        `json:"stock" gorm:"default:0;check:stock >= 0"`
    SKU         string     `json:"sku" gorm:"uniqueIndex;size:50"`
    CategoryID  string     `json:"category_id" gorm:"type:varchar(36);index"`
    ImageURL    string     `json:"image_url" gorm:"size:500"`
    Active      bool       `json:"active" gorm:"default:true;index"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at"`
    DeletedAt   *time.Time `json:"-" gorm:"index"`
}

type CreateProductRequest struct {
    Name        string  `json:"name" binding:"required,min=2,max=200" example:"iPhone 15"`
    Description string  `json:"description" example:"Latest iPhone model"`
    Price       float64 `json:"price" binding:"required,min=0" example:"39900"`
    Stock       int     `json:"stock" binding:"min=0" example:"100"`
    SKU         string  `json:"sku" binding:"required,min=2,max=50" example:"IPHONE-15-BLK"`
    CategoryID  string  `json:"category_id" binding:"required" example:"cat-001"`
    ImageURL    string  `json:"image_url" binding:"omitempty,url"`
}

type UpdateProductRequest struct {
    Name        *string  `json:"name,omitempty" binding:"omitempty,min=2,max=200"`
    Description *string  `json:"description,omitempty"`
    Price       *float64 `json:"price,omitempty" binding:"omitempty,min=0"`
    Stock       *int     `json:"stock,omitempty" binding:"omitempty,min=0"`
    ImageURL    *string  `json:"image_url,omitempty" binding:"omitempty,url"`
    Active      *bool    `json:"active,omitempty"`
}

type ProductFilter struct {
    Search     string   `form:"search"`
    CategoryID string   `form:"category_id"`
    MinPrice   *float64 `form:"min_price"`
    MaxPrice   *float64 `form:"max_price"`
    InStock    *bool    `form:"in_stock"`
    Active     *bool    `form:"active"`
    Page       int      `form:"page,default=1"`
    Limit      int      `form:"limit,default=10"`
    SortBy     string   `form:"sort_by,default=created_at"`
    Order      string   `form:"order,default=desc"`
}

type ProductListResponse struct {
    Products   []Product `json:"products"`
    TotalItems int64     `json:"total_items"`
    Page       int       `json:"page"`
    Limit      int       `json:"limit"`
    TotalPages int       `json:"total_pages"`
}
```

---

## 35.3 Error Handling

### Centralized Errors (ตัวอย่างที่ 3)

```go
// errors/errors.go
package apperrors

import (
    "fmt"
    "net/http"
)

// AppError represents an application error
type AppError struct {
    Code       string `json:"code"`
    Message    string `json:"message"`
    Details    interface{} `json:"details,omitempty"`
    StatusCode int    `json:"-"`
    Err        error  `json:"-"`
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// Constructors
func New(statusCode int, code, message string) *AppError {
    return &AppError{
        StatusCode: statusCode,
        Code:       code,
        Message:    message,
    }
}

func Wrap(err error, statusCode int, code, message string) *AppError {
    return &AppError{
        StatusCode: statusCode,
        Code:       code,
        Message:    message,
        Err:        err,
    }
}

// Predefined errors
var (
    ErrNotFound = func(resource string) *AppError {
        return New(http.StatusNotFound, "NOT_FOUND", fmt.Sprintf("%s not found", resource))
    }
    
    ErrAlreadyExists = func(field string) *AppError {
        return New(http.StatusConflict, "ALREADY_EXISTS", fmt.Sprintf("%s already exists", field))
    }
    
    ErrValidation = func(details interface{}) *AppError {
        return &AppError{
            StatusCode: http.StatusUnprocessableEntity,
            Code:       "VALIDATION_ERROR",
            Message:    "Validation failed",
            Details:    details,
        }
    }
    
    ErrUnauthorized = New(http.StatusUnauthorized, "UNAUTHORIZED", "Authentication required")
    ErrForbidden    = New(http.StatusForbidden, "FORBIDDEN", "Insufficient permissions")
    ErrInternal     = New(http.StatusInternalServerError, "INTERNAL_ERROR", "An unexpected error occurred")
    
    ErrInvalidCredentials = New(http.StatusUnauthorized, "INVALID_CREDENTIALS", "Invalid email or password")
    ErrTokenExpired       = New(http.StatusUnauthorized, "TOKEN_EXPIRED", "Token has expired")
    ErrTokenInvalid       = New(http.StatusUnauthorized, "TOKEN_INVALID", "Invalid token")
)
```

---

## 35.4 Response Utilities

### Standardized Response (ตัวอย่างที่ 4)

```go
// utils/response.go
package utils

import (
    "errors"
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
    
    apperrors "ecommerce-api/errors"
)

type APIResponse struct {
    Success   bool        `json:"success"`
    Data      interface{} `json:"data,omitempty"`
    Error     *ErrorInfo  `json:"error,omitempty"`
    Meta      interface{} `json:"meta,omitempty"`
    Timestamp time.Time   `json:"timestamp"`
}

type ErrorInfo struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Details interface{} `json:"details,omitempty"`
}

func Success(c *gin.Context, statusCode int, data interface{}) {
    c.JSON(statusCode, APIResponse{
        Success:   true,
        Data:      data,
        Timestamp: time.Now(),
    })
}

func SuccessWithMeta(c *gin.Context, statusCode int, data, meta interface{}) {
    c.JSON(statusCode, APIResponse{
        Success:   true,
        Data:      data,
        Meta:      meta,
        Timestamp: time.Now(),
    })
}

func Error(c *gin.Context, err error) {
    var appErr *apperrors.AppError
    if errors.As(err, &appErr) {
        c.JSON(appErr.StatusCode, APIResponse{
            Success: false,
            Error: &ErrorInfo{
                Code:    appErr.Code,
                Message: appErr.Message,
                Details: appErr.Details,
            },
            Timestamp: time.Now(),
        })
        return
    }
    
    // Unknown error
    c.JSON(http.StatusInternalServerError, APIResponse{
        Success: false,
        Error: &ErrorInfo{
            Code:    "INTERNAL_ERROR",
            Message: "An unexpected error occurred",
        },
        Timestamp: time.Now(),
    })
}

// FormatValidationErrors formats validator.ValidationErrors into user-friendly messages
func FormatValidationErrors(err error) interface{} {
    validationErrors, ok := err.(validator.ValidationErrors)
    if !ok {
        return err.Error()
    }
    
    fields := make(map[string]string)
    for _, e := range validationErrors {
        field := toSnakeCase(e.Field())
        switch e.Tag() {
        case "required":
            fields[field] = field + " is required"
        case "email":
            fields[field] = field + " must be a valid email"
        case "min":
            fields[field] = field + " must be at least " + e.Param() + " characters"
        case "max":
            fields[field] = field + " must be at most " + e.Param() + " characters"
        case "oneof":
            fields[field] = field + " must be one of: " + e.Param()
        case "url":
            fields[field] = field + " must be a valid URL"
        default:
            fields[field] = field + " is invalid"
        }
    }
    return fields
}

func toSnakeCase(s string) string {
    var result []byte
    for i, c := range s {
        if c >= 'A' && c <= 'Z' {
            if i > 0 {
                result = append(result, '_')
            }
            result = append(result, byte(c+32))
        } else {
            result = append(result, byte(c))
        }
    }
    return string(result)
}
```

---

## 35.5 Repository Layer

### User Repository (ตัวอย่างที่ 5)

```go
// repository/user_repository.go
package repository

import (
    "context"
    "strings"
    
    "gorm.io/gorm"
    
    apperrors "ecommerce-api/errors"
    "ecommerce-api/models"
)

type UserRepository interface {
    Create(ctx context.Context, user *models.User) error
    GetByID(ctx context.Context, id string) (*models.User, error)
    GetByEmail(ctx context.Context, email string) (*models.User, error)
    Update(ctx context.Context, id string, updates map[string]interface{}) (*models.User, error)
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter models.UserFilter) ([]models.User, int64, error)
    ExistsByEmail(ctx context.Context, email string) (bool, error)
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *models.User) error {
    if err := r.db.WithContext(ctx).Create(user).Error; err != nil {
        if strings.Contains(err.Error(), "unique") || strings.Contains(err.Error(), "UNIQUE") {
            return apperrors.ErrAlreadyExists("email")
        }
        return apperrors.Wrap(err, 500, "DB_ERROR", "Failed to create user")
    }
    return nil
}

func (r *userRepository) GetByID(ctx context.Context, id string) (*models.User, error) {
    var user models.User
    err := r.db.WithContext(ctx).Where("id = ? AND deleted_at IS NULL", id).First(&user).Error
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, apperrors.ErrNotFound("user")
        }
        return nil, apperrors.Wrap(err, 500, "DB_ERROR", "Failed to get user")
    }
    return &user, nil
}

func (r *userRepository) GetByEmail(ctx context.Context, email string) (*models.User, error) {
    var user models.User
    err := r.db.WithContext(ctx).Where("email = ? AND deleted_at IS NULL", strings.ToLower(email)).First(&user).Error
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, apperrors.ErrNotFound("user")
        }
        return nil, apperrors.Wrap(err, 500, "DB_ERROR", "Failed to get user")
    }
    return &user, nil
}

func (r *userRepository) Update(ctx context.Context, id string, updates map[string]interface{}) (*models.User, error) {
    result := r.db.WithContext(ctx).Model(&models.User{}).Where("id = ?", id).Updates(updates)
    if result.Error != nil {
        return nil, apperrors.Wrap(result.Error, 500, "DB_ERROR", "Failed to update user")
    }
    if result.RowsAffected == 0 {
        return nil, apperrors.ErrNotFound("user")
    }
    return r.GetByID(ctx, id)
}

func (r *userRepository) Delete(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Where("id = ?", id).Delete(&models.User{})
    if result.Error != nil {
        return apperrors.Wrap(result.Error, 500, "DB_ERROR", "Failed to delete user")
    }
    if result.RowsAffected == 0 {
        return apperrors.ErrNotFound("user")
    }
    return nil
}

func (r *userRepository) List(ctx context.Context, filter models.UserFilter) ([]models.User, int64, error) {
    query := r.db.WithContext(ctx).Model(&models.User{}).Where("deleted_at IS NULL")
    
    if filter.Search != "" {
        search := "%" + filter.Search + "%"
        query = query.Where("name ILIKE ? OR email ILIKE ?", search, search)
    }
    if filter.Role != "" {
        query = query.Where("role = ?", filter.Role)
    }
    if filter.Active != nil {
        query = query.Where("active = ?", *filter.Active)
    }
    
    var total int64
    query.Count(&total)
    
    sortBy := filter.SortBy
    if sortBy == "" {
        sortBy = "created_at"
    }
    order := filter.Order
    if order == "" {
        order = "desc"
    }
    
    var users []models.User
    offset := (filter.Page - 1) * filter.Limit
    err := query.Order(sortBy + " " + order).
        Offset(offset).
        Limit(filter.Limit).
        Find(&users).Error
    
    if err != nil {
        return nil, 0, apperrors.Wrap(err, 500, "DB_ERROR", "Failed to list users")
    }
    
    return users, total, nil
}

func (r *userRepository) ExistsByEmail(ctx context.Context, email string) (bool, error) {
    var count int64
    err := r.db.WithContext(ctx).Model(&models.User{}).
        Where("email = ? AND deleted_at IS NULL", strings.ToLower(email)).
        Count(&count).Error
    return count > 0, err
}
```

### Product Repository (ตัวอย่างที่ 6)

```go
// repository/product_repository.go
package repository

import (
    "context"
    "strings"
    
    "gorm.io/gorm"
    
    apperrors "ecommerce-api/errors"
    "ecommerce-api/models"
)

type ProductRepository interface {
    Create(ctx context.Context, product *models.Product) error
    GetByID(ctx context.Context, id string) (*models.Product, error)
    GetBySKU(ctx context.Context, sku string) (*models.Product, error)
    Update(ctx context.Context, id string, updates map[string]interface{}) (*models.Product, error)
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter models.ProductFilter) ([]models.Product, int64, error)
    UpdateStock(ctx context.Context, id string, delta int) error
}

type productRepository struct {
    db *gorm.DB
}

func NewProductRepository(db *gorm.DB) ProductRepository {
    return &productRepository{db: db}
}

func (r *productRepository) Create(ctx context.Context, product *models.Product) error {
    if err := r.db.WithContext(ctx).Create(product).Error; err != nil {
        if strings.Contains(err.Error(), "unique") || strings.Contains(err.Error(), "UNIQUE") {
            return apperrors.ErrAlreadyExists("SKU")
        }
        return apperrors.Wrap(err, 500, "DB_ERROR", "Failed to create product")
    }
    return nil
}

func (r *productRepository) GetByID(ctx context.Context, id string) (*models.Product, error) {
    var product models.Product
    err := r.db.WithContext(ctx).Where("id = ? AND deleted_at IS NULL", id).First(&product).Error
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, apperrors.ErrNotFound("product")
        }
        return nil, apperrors.Wrap(err, 500, "DB_ERROR", "Failed to get product")
    }
    return &product, nil
}

func (r *productRepository) GetBySKU(ctx context.Context, sku string) (*models.Product, error) {
    var product models.Product
    err := r.db.WithContext(ctx).Where("sku = ? AND deleted_at IS NULL", sku).First(&product).Error
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, apperrors.ErrNotFound("product")
        }
        return nil, err
    }
    return &product, nil
}

func (r *productRepository) Update(ctx context.Context, id string, updates map[string]interface{}) (*models.Product, error) {
    result := r.db.WithContext(ctx).Model(&models.Product{}).Where("id = ?", id).Updates(updates)
    if result.Error != nil {
        return nil, apperrors.Wrap(result.Error, 500, "DB_ERROR", "Failed to update product")
    }
    if result.RowsAffected == 0 {
        return nil, apperrors.ErrNotFound("product")
    }
    return r.GetByID(ctx, id)
}

func (r *productRepository) Delete(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Where("id = ?", id).Delete(&models.Product{})
    if result.Error != nil {
        return apperrors.Wrap(result.Error, 500, "DB_ERROR", "Failed to delete product")
    }
    if result.RowsAffected == 0 {
        return apperrors.ErrNotFound("product")
    }
    return nil
}

func (r *productRepository) List(ctx context.Context, filter models.ProductFilter) ([]models.Product, int64, error) {
    query := r.db.WithContext(ctx).Model(&models.Product{}).Where("deleted_at IS NULL")
    
    if filter.Search != "" {
        search := "%" + filter.Search + "%"
        query = query.Where("name LIKE ? OR description LIKE ?", search, search)
    }
    if filter.CategoryID != "" {
        query = query.Where("category_id = ?", filter.CategoryID)
    }
    if filter.MinPrice != nil {
        query = query.Where("price >= ?", *filter.MinPrice)
    }
    if filter.MaxPrice != nil {
        query = query.Where("price <= ?", *filter.MaxPrice)
    }
    if filter.InStock != nil && *filter.InStock {
        query = query.Where("stock > 0")
    }
    if filter.Active != nil {
        query = query.Where("active = ?", *filter.Active)
    }
    
    var total int64
    query.Count(&total)
    
    if filter.Limit == 0 {
        filter.Limit = 10
    }
    if filter.Page == 0 {
        filter.Page = 1
    }
    
    var products []models.Product
    err := query.
        Order(filter.SortBy + " " + filter.Order).
        Offset((filter.Page - 1) * filter.Limit).
        Limit(filter.Limit).
        Find(&products).Error
    
    if err != nil {
        return nil, 0, apperrors.Wrap(err, 500, "DB_ERROR", "Failed to list products")
    }
    
    return products, total, nil
}

func (r *productRepository) UpdateStock(ctx context.Context, id string, delta int) error {
    result := r.db.WithContext(ctx).Model(&models.Product{}).
        Where("id = ? AND deleted_at IS NULL", id).
        UpdateColumn("stock", gorm.Expr("stock + ?", delta))
    
    if result.Error != nil {
        return apperrors.Wrap(result.Error, 500, "DB_ERROR", "Failed to update stock")
    }
    if result.RowsAffected == 0 {
        return apperrors.ErrNotFound("product")
    }
    return nil
}
```

---

## 35.6 Service Layer

### User Service (ตัวอย่างที่ 7)

```go
// service/user_service.go
package service

import (
    "context"
    "strings"
    "time"
    
    "golang.org/x/crypto/bcrypt"
    
    apperrors "ecommerce-api/errors"
    "ecommerce-api/models"
    "ecommerce-api/repository"
    "ecommerce-api/utils"
)

type UserService interface {
    Create(ctx context.Context, req models.CreateUserRequest) (*models.UserProfile, error)
    GetByID(ctx context.Context, id string) (*models.UserProfile, error)
    Update(ctx context.Context, id string, req models.UpdateUserRequest) (*models.UserProfile, error)
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter models.UserFilter) (*models.UserListResponse, error)
    ChangePassword(ctx context.Context, id string, req models.ChangePasswordRequest) error
}

type userService struct {
    userRepo repository.UserRepository
}

func NewUserService(userRepo repository.UserRepository) UserService {
    return &userService{userRepo: userRepo}
}

func (s *userService) Create(ctx context.Context, req models.CreateUserRequest) (*models.UserProfile, error) {
    // ตรวจสอบ email ซ้ำ
    exists, err := s.userRepo.ExistsByEmail(ctx, req.Email)
    if err != nil {
        return nil, apperrors.ErrInternal
    }
    if exists {
        return nil, apperrors.ErrAlreadyExists("email")
    }
    
    // Hash password
    hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, apperrors.Wrap(err, 500, "HASH_ERROR", "Failed to hash password")
    }
    
    user := &models.User{
        ID:           utils.GenerateID(),
        Name:         strings.TrimSpace(req.Name),
        Email:        strings.ToLower(strings.TrimSpace(req.Email)),
        PasswordHash: string(hash),
        Role:         "user",
        Active:       true,
        CreatedAt:    time.Now(),
        UpdatedAt:    time.Now(),
    }
    
    if err := s.userRepo.Create(ctx, user); err != nil {
        return nil, err
    }
    
    profile := user.ToProfile()
    return &profile, nil
}

func (s *userService) GetByID(ctx context.Context, id string) (*models.UserProfile, error) {
    user, err := s.userRepo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }
    profile := user.ToProfile()
    return &profile, nil
}

func (s *userService) Update(ctx context.Context, id string, req models.UpdateUserRequest) (*models.UserProfile, error) {
    updates := make(map[string]interface{})
    updates["updated_at"] = time.Now()
    
    if req.Name != nil {
        updates["name"] = strings.TrimSpace(*req.Name)
    }
    if req.Email != nil {
        email := strings.ToLower(strings.TrimSpace(*req.Email))
        // ตรวจสอบ email ใหม่ว่าซ้ำไหม
        exists, err := s.userRepo.ExistsByEmail(ctx, email)
        if err != nil {
            return nil, apperrors.ErrInternal
        }
        if exists {
            return nil, apperrors.ErrAlreadyExists("email")
        }
        updates["email"] = email
    }
    if req.Active != nil {
        updates["active"] = *req.Active
    }
    
    user, err := s.userRepo.Update(ctx, id, updates)
    if err != nil {
        return nil, err
    }
    
    profile := user.ToProfile()
    return &profile, nil
}

func (s *userService) Delete(ctx context.Context, id string) error {
    return s.userRepo.Delete(ctx, id)
}

func (s *userService) List(ctx context.Context, filter models.UserFilter) (*models.UserListResponse, error) {
    users, total, err := s.userRepo.List(ctx, filter)
    if err != nil {
        return nil, err
    }
    
    profiles := make([]models.UserProfile, len(users))
    for i, u := range users {
        profiles[i] = u.ToProfile()
    }
    
    totalPages := int((total + int64(filter.Limit) - 1) / int64(filter.Limit))
    
    return &models.UserListResponse{
        Users:      profiles,
        TotalItems: total,
        Page:       filter.Page,
        Limit:      filter.Limit,
        TotalPages: totalPages,
    }, nil
}

func (s *userService) ChangePassword(ctx context.Context, id string, req models.ChangePasswordRequest) error {
    if req.NewPassword != req.ConfirmPassword {
        return apperrors.New(400, "PASSWORD_MISMATCH", "New password and confirmation do not match")
    }
    
    user, err := s.userRepo.GetByID(ctx, id)
    if err != nil {
        return err
    }
    
    // ตรวจสอบ current password
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.CurrentPassword)); err != nil {
        return apperrors.New(400, "WRONG_PASSWORD", "Current password is incorrect")
    }
    
    // Hash new password
    hash, err := bcrypt.GenerateFromPassword([]byte(req.NewPassword), bcrypt.DefaultCost)
    if err != nil {
        return apperrors.ErrInternal
    }
    
    _, err = s.userRepo.Update(ctx, id, map[string]interface{}{
        "password_hash": string(hash),
        "updated_at":    time.Now(),
    })
    return err
}
```

---

## 35.7 Handler Layer

### User Handler (ตัวอย่างที่ 8)

```go
// handler/user_handler.go
package handler

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
    
    apperrors "ecommerce-api/errors"
    "ecommerce-api/models"
    "ecommerce-api/service"
    "ecommerce-api/utils"
)

type UserHandler struct {
    userService service.UserService
}

func NewUserHandler(userService service.UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

// CreateUser godoc
// @Summary Create a new user
// @Description Create a new user account
// @Tags users
// @Accept json
// @Produce json
// @Param request body models.CreateUserRequest true "User data"
// @Success 201 {object} utils.APIResponse{data=models.UserProfile}
// @Failure 400 {object} utils.APIResponse
// @Failure 409 {object} utils.APIResponse
// @Router /users [post]
func (h *UserHandler) Create(c *gin.Context) {
    var req models.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.Error(c, apperrors.ErrValidation(utils.FormatValidationErrors(err)))
        return
    }
    
    user, err := h.userService.Create(c.Request.Context(), req)
    if err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.Success(c, http.StatusCreated, user)
}

// GetUser godoc
// @Summary Get user by ID
// @Tags users
// @Param id path string true "User ID"
// @Success 200 {object} utils.APIResponse{data=models.UserProfile}
// @Failure 404 {object} utils.APIResponse
// @Security BearerAuth
// @Router /users/{id} [get]
func (h *UserHandler) GetByID(c *gin.Context) {
    id := c.Param("id")
    
    user, err := h.userService.GetByID(c.Request.Context(), id)
    if err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.Success(c, http.StatusOK, user)
}

// ListUsers godoc
// @Summary List users with pagination and filtering
// @Tags users
// @Param page query int false "Page number" default(1)
// @Param limit query int false "Items per page" default(10)
// @Param search query string false "Search term"
// @Param role query string false "Filter by role"
// @Success 200 {object} utils.APIResponse{data=models.UserListResponse}
// @Security BearerAuth
// @Router /users [get]
func (h *UserHandler) List(c *gin.Context) {
    var filter models.UserFilter
    if err := c.ShouldBindQuery(&filter); err != nil {
        utils.Error(c, apperrors.ErrValidation(utils.FormatValidationErrors(err)))
        return
    }
    
    response, err := h.userService.List(c.Request.Context(), filter)
    if err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.SuccessWithMeta(c, http.StatusOK, response.Users, map[string]interface{}{
        "page":        response.Page,
        "limit":       response.Limit,
        "total_items": response.TotalItems,
        "total_pages": response.TotalPages,
    })
}

// UpdateUser godoc
// @Summary Update user
// @Tags users
// @Param id path string true "User ID"
// @Param request body models.UpdateUserRequest true "Update data"
// @Success 200 {object} utils.APIResponse{data=models.UserProfile}
// @Security BearerAuth
// @Router /users/{id} [put]
func (h *UserHandler) Update(c *gin.Context) {
    id := c.Param("id")
    
    var req models.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.Error(c, apperrors.ErrValidation(utils.FormatValidationErrors(err)))
        return
    }
    
    user, err := h.userService.Update(c.Request.Context(), id, req)
    if err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.Success(c, http.StatusOK, user)
}

// DeleteUser godoc
// @Summary Delete user
// @Tags users
// @Param id path string true "User ID"
// @Success 200 {object} utils.APIResponse
// @Failure 404 {object} utils.APIResponse
// @Security BearerAuth
// @Router /users/{id} [delete]
func (h *UserHandler) Delete(c *gin.Context) {
    id := c.Param("id")
    
    if err := h.userService.Delete(c.Request.Context(), id); err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.Success(c, http.StatusOK, map[string]string{"message": "User deleted successfully"})
}

// ChangePassword godoc
// @Summary Change user password
// @Tags users
// @Param id path string true "User ID"
// @Param request body models.ChangePasswordRequest true "Password data"
// @Success 200 {object} utils.APIResponse
// @Security BearerAuth
// @Router /users/{id}/password [put]
func (h *UserHandler) ChangePassword(c *gin.Context) {
    id := c.Param("id")
    
    // ตรวจสอบว่า user กำลัง change password ของตัวเอง
    currentUserID, _ := c.Get("user_id")
    if currentUserID != id {
        utils.Error(c, apperrors.ErrForbidden)
        return
    }
    
    var req models.ChangePasswordRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.Error(c, apperrors.ErrValidation(utils.FormatValidationErrors(err)))
        return
    }
    
    if err := h.userService.ChangePassword(c.Request.Context(), id, req); err != nil {
        utils.Error(c, err)
        return
    }
    
    utils.Success(c, http.StatusOK, map[string]string{"message": "Password changed successfully"})
}
```

---

## 35.8 Router Setup

### Router Configuration (ตัวอย่างที่ 9)

```go
// router/router.go
package router

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
    
    "ecommerce-api/handler"
    "ecommerce-api/middleware"
)

type Router struct {
    engine          *gin.Engine
    userHandler     *handler.UserHandler
    productHandler  *handler.ProductHandler
    authMiddleware  gin.HandlerFunc
}

func NewRouter(
    userHandler *handler.UserHandler,
    productHandler *handler.ProductHandler,
    authMiddleware gin.HandlerFunc,
) *Router {
    engine := gin.New()
    return &Router{
        engine:         engine,
        userHandler:    userHandler,
        productHandler: productHandler,
        authMiddleware: authMiddleware,
    }
}

func (r *Router) Setup() *gin.Engine {
    // Global middleware
    r.engine.Use(gin.Recovery())
    r.engine.Use(middleware.RequestID())
    r.engine.Use(middleware.Logger())
    r.engine.Use(middleware.CORS([]string{"*"}))
    
    // Health check
    r.engine.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":  "healthy",
            "version": "1.0.0",
        })
    })
    
    // API v1
    v1 := r.engine.Group("/api/v1")
    
    // Public routes
    public := v1.Group("")
    {
        public.POST("/auth/register", r.userHandler.Create)
        // public.POST("/auth/login", r.authHandler.Login)
        // public.POST("/auth/refresh", r.authHandler.Refresh)
    }
    
    // Products (public read)
    products := v1.Group("/products")
    {
        products.GET("", r.productHandler.List)
        products.GET("/:id", r.productHandler.GetByID)
        
        // Protected write operations
        protected := products.Group("")
        protected.Use(r.authMiddleware)
        protected.POST("", r.productHandler.Create)
        protected.PUT("/:id", r.productHandler.Update)
        protected.DELETE("/:id", r.productHandler.Delete)
    }
    
    // Protected routes
    protected := v1.Group("")
    protected.Use(r.authMiddleware)
    {
        // User management
        users := protected.Group("/users")
        {
            users.GET("", r.userHandler.List)
            users.GET("/:id", r.userHandler.GetByID)
            users.PUT("/:id", r.userHandler.Update)
            users.DELETE("/:id", r.userHandler.Delete)
            users.PUT("/:id/password", r.userHandler.ChangePassword)
        }
    }
    
    return r.engine
}
```

---

## 35.9 Main Application

### Complete Main (ตัวอย่างที่ 10)

```go
// main.go
package main

import (
    "fmt"
    "log"
    "os"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
    
    "ecommerce-api/handler"
    "ecommerce-api/models"
    "ecommerce-api/repository"
    "ecommerce-api/router"
    "ecommerce-api/service"
)

func main() {
    // Setup database
    db, err := setupDatabase()
    if err != nil {
        log.Fatal("Database setup failed:", err)
    }
    
    // Initialize repositories
    userRepo := repository.NewUserRepository(db)
    productRepo := repository.NewProductRepository(db)
    
    // Initialize services
    userService := service.NewUserService(userRepo)
    productService := service.NewProductService(productRepo)
    
    // Initialize handlers
    userHandler := handler.NewUserHandler(userService)
    productHandler := handler.NewProductHandler(productService)
    
    // Auth middleware (simplified)
    authMiddleware := func(c *gin.Context) {
        // ใน production: ตรวจ JWT จริงๆ
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" || authHeader != "Bearer valid-token" {
            c.AbortWithStatusJSON(401, gin.H{"error": "Unauthorized"})
            return
        }
        c.Set("user_id", "user-001")
        c.Set("user_role", "admin")
        c.Next()
    }
    
    // Setup router
    r := router.NewRouter(userHandler, productHandler, authMiddleware)
    engine := r.Setup()
    
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    
    fmt.Printf("E-commerce API running on port %s\n", port)
    fmt.Println("Endpoints:")
    fmt.Println("  GET    /health")
    fmt.Println("  GET    /api/v1/products")
    fmt.Println("  POST   /api/v1/products")
    fmt.Println("  GET    /api/v1/users")
    fmt.Println("  POST   /auth/register")
    
    log.Fatal(engine.Run(":" + port))
}

func setupDatabase() (*gorm.DB, error) {
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    if err != nil {
        return nil, err
    }
    
    // AutoMigrate
    if err := db.AutoMigrate(
        &models.User{},
        &models.Product{},
    ); err != nil {
        return nil, fmt.Errorf("migration failed: %w", err)
    }
    
    // Seed data
    seedData(db)
    
    return db, nil
}

func seedData(db *gorm.DB) {
    // Add sample products
    products := []models.Product{
        {
            ID: "prod-001", Name: "MacBook Pro",
            Description: "Powerful laptop for developers",
            Price: 89000, Stock: 50,
            SKU: "MBP-M3-2024", CategoryID: "cat-laptops",
            Active: true,
        },
        {
            ID: "prod-002", Name: "iPhone 15 Pro",
            Description: "Latest iPhone with titanium design",
            Price: 42000, Stock: 200,
            SKU: "IP15PRO-BLK", CategoryID: "cat-phones",
            Active: true,
        },
        {
            ID: "prod-003", Name: "AirPods Pro",
            Description: "Noise-canceling wireless earbuds",
            Price: 9000, Stock: 300,
            SKU: "APP-2023", CategoryID: "cat-audio",
            Active: true,
        },
    }
    
    for _, p := range products {
        db.FirstOrCreate(&p, models.Product{ID: p.ID})
    }
    
    log.Printf("Seeded %d products\n", len(products))
}
```

---

## 35.10 Utilities

### Generate ID (ตัวอย่างที่ 11)

```go
// utils/utils.go
package utils

import (
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "strings"
)

// GenerateID generates a random unique ID
func GenerateID() string {
    b := make([]byte, 8)
    rand.Read(b)
    return hex.EncodeToString(b)
}

// GenerateUUID generates a UUID v4
func GenerateUUID() string {
    b := make([]byte, 16)
    rand.Read(b)
    b[6] = (b[6] & 0x0f) | 0x40
    b[8] = (b[8] & 0x3f) | 0x80
    return fmt.Sprintf("%x-%x-%x-%x-%x", b[0:4], b[4:6], b[6:8], b[8:10], b[10:])
}

// Paginate calculates pagination values
func Paginate(page, limit int, total int64) map[string]interface{} {
    totalPages := int((total + int64(limit) - 1) / int64(limit))
    return map[string]interface{}{
        "page":        page,
        "limit":       limit,
        "total_items": total,
        "total_pages": totalPages,
        "has_next":    page < totalPages,
        "has_prev":    page > 1,
    }
}

// ToSnakeCase converts PascalCase or camelCase to snake_case
func ToSnakeCase(s string) string {
    var result []byte
    for i, c := range s {
        if c >= 'A' && c <= 'Z' {
            if i > 0 {
                result = append(result, '_')
            }
            result = append(result, byte(c+32))
        } else {
            result = append(result, byte(c))
        }
    }
    return string(result)
}

// Ptr returns a pointer to the given value
func Ptr[T any](v T) *T {
    return &v
}

// SanitizeSearch removes special characters from search string
func SanitizeSearch(s string) string {
    s = strings.TrimSpace(s)
    // Remove SQL injection patterns
    s = strings.ReplaceAll(s, "'", "")
    s = strings.ReplaceAll(s, "\"", "")
    s = strings.ReplaceAll(s, ";", "")
    s = strings.ReplaceAll(s, "--", "")
    return s
}
```

---

## 35.11 Swagger Documentation

### Setup Swagger (ตัวอย่างที่ 12)

```bash
# ติดตั้ง swag CLI
go install github.com/swaggo/swag/cmd/swag@latest

# ติดตั้ง gin-swagger
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
go get github.com/swaggo/swag

# Generate docs
swag init
```

### Swagger Setup ใน main.go (ตัวอย่างที่ 13)

```go
package main

// @title           E-Commerce API
// @version         1.0
// @description     A comprehensive e-commerce API with CRUD operations
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.email  support@ecommerce.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

import (
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"
    
    _ "ecommerce-api/docs" // ต้อง import docs package ที่ swag สร้าง
)

func setupSwagger(r *gin.Engine) {
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

---

## Workshop: ทดสอบ API ด้วย curl

```bash
# 1. Health check
curl http://localhost:8080/health

# 2. List products
curl http://localhost:8080/api/v1/products

# 3. Get product
curl http://localhost:8080/api/v1/products/prod-001

# 4. Search products
curl "http://localhost:8080/api/v1/products?search=mac&limit=5"

# 5. Create product (ต้องมี auth)
curl -X POST http://localhost:8080/api/v1/products \
  -H "Authorization: Bearer valid-token" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPad Pro",
    "description": "Powerful tablet",
    "price": 29900,
    "stock": 100,
    "sku": "IPAD-PRO-2024",
    "category_id": "cat-tablets"
  }'

# 6. Update product
curl -X PUT http://localhost:8080/api/v1/products/prod-001 \
  -H "Authorization: Bearer valid-token" \
  -H "Content-Type: application/json" \
  -d '{"price": 85000, "stock": 45}'

# 7. Delete product
curl -X DELETE http://localhost:8080/api/v1/products/prod-003 \
  -H "Authorization: Bearer valid-token"

# 8. Register user
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "password": "SecurePass123"
  }'

# 9. List users (ต้องมี auth)
curl http://localhost:8080/api/v1/users \
  -H "Authorization: Bearer valid-token"

# 10. Filter users
curl "http://localhost:8080/api/v1/users?search=john&page=1&limit=5" \
  -H "Authorization: Bearer valid-token"
```

---

## สรุป Part 35

ใน Part นี้เราได้สร้าง Complete CRUD API ที่มี:

| Layer | ความรับผิดชอบ |
|-------|--------------|
| Models | Data structures, validation tags, request/response types |
| Repository | Database access, SQL queries, error mapping |
| Service | Business logic, validation, data transformation |
| Handler | HTTP handling, request binding, response formatting |
| Router | Route registration, middleware application |
| Middleware | Cross-cutting concerns (auth, logging, CORS) |
| Utils | Shared helpers (response, validation, ID generation) |
| Errors | Centralized error types และ codes |

### Clean Architecture Principles

1. **Dependency Inversion** - Layers depend on interfaces, not implementations
2. **Single Responsibility** - Each layer has one job
3. **Open/Closed** - Open for extension, closed for modification
4. **Testability** - Each layer can be tested independently

### Next Steps

1. เพิ่ม Unit Tests สำหรับทุก layer
2. เพิ่ม Integration Tests
3. Setup CI/CD pipeline
4. Deploy ด้วย Docker
5. เพิ่ม Caching (Redis)
6. เพิ่ม API Rate Limiting per user
7. เพิ่ม Audit logs

### Resources
- [Clean Architecture in Go](https://github.com/bxcodec/go-clean-arch)
- [Gin framework](https://gin-gonic.com)
- [GORM](https://gorm.io)
- [Swagger/OpenAPI](https://swagger.io)
- [Go best practices](https://go.dev/doc/effective_go)
