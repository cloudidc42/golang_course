# Part 28: REST API Basics ด้วย Gin Framework

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ REST principles
- ใช้ HTTP Methods อย่างถูกต้อง
- ติดตั้งและใช้งาน Gin framework
- สร้าง routes และจัดการ parameters
- Bind request data (JSON, form, query)
- ส่ง response ในรูปแบบต่างๆ
- จัดการ error อย่างเป็นระบบ

---

## 28.1 REST Principles

REST (Representational State Transfer) เป็น architectural style สำหรับ API design

### 6 Constraints ของ REST
1. **Client-Server** - แยก client กับ server
2. **Stateless** - ทุก request ต้องมีข้อมูลครบ ไม่เก็บ session บน server
3. **Cacheable** - response ต้องบอกว่า cacheable ได้ไหม
4. **Uniform Interface** - มีรูปแบบ interface ที่สม่ำเสมอ
5. **Layered System** - มี layers เช่น load balancer, cache
6. **Code on Demand** (optional) - server ส่ง executable code ได้

### HTTP Methods และการใช้งาน

| Method | ใช้สำหรับ | Idempotent | Safe |
|--------|-----------|------------|------|
| GET | อ่านข้อมูล | ✓ | ✓ |
| POST | สร้างข้อมูล | ✗ | ✗ |
| PUT | อัพเดทแบบ full | ✓ | ✗ |
| PATCH | อัพเดทแบบ partial | ✗ | ✗ |
| DELETE | ลบข้อมูล | ✓ | ✗ |

### URL Design Conventions
```
GET    /posts          # ดึงรายการ posts
GET    /posts/:id      # ดึง post เฉพาะ ID
POST   /posts          # สร้าง post ใหม่
PUT    /posts/:id      # แทนที่ post ทั้งหมด
PATCH  /posts/:id      # อัพเดทบางส่วนของ post
DELETE /posts/:id      # ลบ post

# Nested resources
GET    /users/:id/posts           # ดึง posts ของ user
POST   /users/:id/posts           # สร้าง post สำหรับ user
GET    /users/:id/posts/:postId   # ดึง post เฉพาะของ user
```

---

## 28.2 ติดตั้ง Gin

```bash
# สร้าง project ใหม่
mkdir blog-api && cd blog-api
go mod init github.com/yourname/blog-api

# ติดตั้ง Gin
go get github.com/gin-gonic/gin
```

### Hello World ด้วย Gin (ตัวอย่างที่ 1)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

func main() {
    // สร้าง Gin router พร้อม default middleware (logger + recovery)
    r := gin.Default()
    
    // ลงทะเบียน route
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })
    
    // เริ่ม server
    r.Run(":8080") // default port 8080
}
```

---

## 28.3 Router Setup

### Basic Router (ตัวอย่างที่ 2)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // Simple routes
    r.GET("/", func(c *gin.Context) {
        c.String(http.StatusOK, "Hello World!")
    })
    
    r.GET("/about", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"page": "about"})
    })
    
    // Health check
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":  "healthy",
            "version": "1.0.0",
        })
    })
    
    // รองรับ OPTIONS สำหรับ CORS preflight
    r.OPTIONS("/api/*path", func(c *gin.Context) {
        c.Status(http.StatusNoContent)
    })
    
    r.Run(":8080")
}
```

### Route Groups (ตัวอย่างที่ 3)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // API v1 group
    v1 := r.Group("/api/v1")
    {
        users := v1.Group("/users")
        {
            users.GET("", listUsers)
            users.POST("", createUser)
            users.GET("/:id", getUser)
            users.PUT("/:id", updateUser)
            users.DELETE("/:id", deleteUser)
        }
        
        posts := v1.Group("/posts")
        {
            posts.GET("", listPosts)
            posts.POST("", createPost)
            posts.GET("/:id", getPost)
            posts.PUT("/:id", updatePost)
            posts.DELETE("/:id", deletePost)
        }
    }
    
    // API v2 group
    v2 := r.Group("/api/v2")
    {
        v2.GET("/users", func(c *gin.Context) {
            c.JSON(http.StatusOK, gin.H{"version": "v2", "feature": "new users API"})
        })
    }
    
    r.Run(":8080")
}

// Placeholder handlers
func listUsers(c *gin.Context)   { c.JSON(http.StatusOK, gin.H{"action": "list users"}) }
func createUser(c *gin.Context)  { c.JSON(http.StatusCreated, gin.H{"action": "create user"}) }
func getUser(c *gin.Context)     { c.JSON(http.StatusOK, gin.H{"action": "get user"}) }
func updateUser(c *gin.Context)  { c.JSON(http.StatusOK, gin.H{"action": "update user"}) }
func deleteUser(c *gin.Context)  { c.JSON(http.StatusOK, gin.H{"action": "delete user"}) }
func listPosts(c *gin.Context)   { c.JSON(http.StatusOK, gin.H{"action": "list posts"}) }
func createPost(c *gin.Context)  { c.JSON(http.StatusCreated, gin.H{"action": "create post"}) }
func getPost(c *gin.Context)     { c.JSON(http.StatusOK, gin.H{"action": "get post"}) }
func updatePost(c *gin.Context)  { c.JSON(http.StatusOK, gin.H{"action": "update post"}) }
func deletePost(c *gin.Context)  { c.JSON(http.StatusOK, gin.H{"action": "delete post"}) }
```

---

## 28.4 Route Parameters

### Path Parameters (ตัวอย่างที่ 4)

```go
package main

import (
    "net/http"
    "strconv"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // :id = required path parameter
    r.GET("/users/:id", func(c *gin.Context) {
        idStr := c.Param("id")
        id, err := strconv.Atoi(idStr)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid ID format"})
            return
        }
        c.JSON(http.StatusOK, gin.H{"user_id": id})
    })
    
    // Multiple path params
    r.GET("/users/:userID/posts/:postID", func(c *gin.Context) {
        userID := c.Param("userID")
        postID := c.Param("postID")
        
        c.JSON(http.StatusOK, gin.H{
            "user_id": userID,
            "post_id": postID,
        })
    })
    
    // Wildcard parameter (*) - matches any path
    r.GET("/files/*filepath", func(c *gin.Context) {
        filepath := c.Param("filepath")
        c.JSON(http.StatusOK, gin.H{"filepath": filepath})
    })
    
    r.Run(":8080")
}
```

### Query Parameters (ตัวอย่างที่ 5)

```go
package main

import (
    "net/http"
    "strconv"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    r.GET("/search", func(c *gin.Context) {
        // อ่าน query params
        keyword := c.Query("q")          // ถ้าไม่มีจะได้ ""
        page := c.DefaultQuery("page", "1")      // ถ้าไม่มีจะได้ default "1"
        limit := c.DefaultQuery("limit", "10")   // default 10
        sortBy := c.DefaultQuery("sort", "created_at")
        order := c.DefaultQuery("order", "desc")
        
        // Parse int
        pageNum, _ := strconv.Atoi(page)
        limitNum, _ := strconv.Atoi(limit)
        
        // อ่าน array params: ?tags=go&tags=api&tags=rest
        tags := c.QueryArray("tags")
        
        // อ่าน map params: ?filters[name]=john&filters[age]=25
        filters := c.QueryMap("filters")
        
        c.JSON(http.StatusOK, gin.H{
            "keyword": keyword,
            "page":    pageNum,
            "limit":   limitNum,
            "sort_by": sortBy,
            "order":   order,
            "tags":    tags,
            "filters": filters,
        })
    })
    
    r.Run(":8080")
}
```

---

## 28.5 Request Binding

### Bind JSON (ตัวอย่างที่ 6)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

type CreatePostRequest struct {
    Title   string   `json:"title" binding:"required,min=3,max=200"`
    Content string   `json:"content" binding:"required,min=10"`
    Tags    []string `json:"tags"`
    Published bool  `json:"published"`
}

func main() {
    r := gin.Default()
    
    r.POST("/posts", func(c *gin.Context) {
        var req CreatePostRequest
        
        // ShouldBindJSON จะไม่ตอบ error เอง
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{
                "error":   "Validation failed",
                "details": err.Error(),
            })
            return
        }
        
        // ใช้ข้อมูลที่ bind แล้ว
        c.JSON(http.StatusCreated, gin.H{
            "message": "Post created",
            "post": gin.H{
                "title":     req.Title,
                "content":   req.Content,
                "tags":      req.Tags,
                "published": req.Published,
            },
        })
    })
    
    r.Run(":8080")
}
```

### Bind Form Data (ตัวอย่างที่ 7)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

type LoginRequest struct {
    Username string `form:"username" binding:"required"`
    Password string `form:"password" binding:"required,min=8"`
}

type RegisterRequest struct {
    Name     string `form:"name" binding:"required"`
    Email    string `form:"email" binding:"required,email"`
    Password string `form:"password" binding:"required,min=8"`
    Age      int    `form:"age" binding:"required,min=18"`
}

func main() {
    r := gin.Default()
    
    // Form POST
    r.POST("/login", func(c *gin.Context) {
        var req LoginRequest
        if err := c.ShouldBind(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        // ตรวจสอบ credentials (simplified)
        if req.Username != "admin" || req.Password != "password123" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
            return
        }
        
        c.JSON(http.StatusOK, gin.H{
            "message": "Login successful",
            "token":   "fake-jwt-token",
        })
    })
    
    // Form POST with more fields
    r.POST("/register", func(c *gin.Context) {
        var req RegisterRequest
        if err := c.ShouldBind(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        c.JSON(http.StatusCreated, gin.H{
            "message": "User registered",
            "user": gin.H{
                "name":  req.Name,
                "email": req.Email,
            },
        })
    })
    
    r.Run(":8080")
}
```

### Bind Query Params to Struct (ตัวอย่างที่ 8)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

type PaginationQuery struct {
    Page    int    `form:"page" binding:"min=1"`
    Limit   int    `form:"limit" binding:"min=1,max=100"`
    SortBy  string `form:"sort_by"`
    Order   string `form:"order" binding:"oneof=asc desc"`
    Search  string `form:"search"`
}

func main() {
    r := gin.Default()
    
    r.GET("/items", func(c *gin.Context) {
        var query PaginationQuery
        
        // Set defaults
        query.Page = 1
        query.Limit = 10
        query.Order = "desc"
        
        if err := c.ShouldBindQuery(&query); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        offset := (query.Page - 1) * query.Limit
        
        c.JSON(http.StatusOK, gin.H{
            "pagination": gin.H{
                "page":   query.Page,
                "limit":  query.Limit,
                "offset": offset,
            },
            "sort": gin.H{
                "by":    query.SortBy,
                "order": query.Order,
            },
            "search": query.Search,
            "items":  []string{"item1", "item2"},
        })
    })
    
    r.Run(":8080")
}
```

### Bind URI Parameters (ตัวอย่างที่ 9)

```go
package main

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
)

type URIParams struct {
    UserID int `uri:"userId" binding:"required,min=1"`
    PostID int `uri:"postId" binding:"required,min=1"`
}

func main() {
    r := gin.Default()
    
    r.GET("/users/:userId/posts/:postId", func(c *gin.Context) {
        var params URIParams
        if err := c.ShouldBindUri(&params); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        c.JSON(http.StatusOK, gin.H{
            "user_id": params.UserID,
            "post_id": params.PostID,
        })
    })
    
    r.Run(":8080")
}
```

---

## 28.6 Response Helpers

### Response Types ต่างๆ (ตัวอย่างที่ 10)

```go
package main

import (
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // JSON response
    r.GET("/json", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "name": "John",
            "age":  30,
        })
    })
    
    // String response
    r.GET("/string", func(c *gin.Context) {
        c.String(http.StatusOK, "Hello, %s!", "World")
    })
    
    // HTML response
    r.GET("/html", func(c *gin.Context) {
        c.HTML(http.StatusOK, "index.tmpl", gin.H{
            "title": "My App",
        })
    })
    
    // XML response
    type Person struct {
        Name string `xml:"name"`
        Age  int    `xml:"age"`
    }
    r.GET("/xml", func(c *gin.Context) {
        c.XML(http.StatusOK, Person{Name: "John", Age: 30})
    })
    
    // Redirect
    r.GET("/redirect", func(c *gin.Context) {
        c.Redirect(http.StatusMovedPermanently, "https://google.com")
    })
    
    // YAML response
    r.GET("/yaml", func(c *gin.Context) {
        c.YAML(http.StatusOK, gin.H{
            "name": "John",
            "time": time.Now(),
        })
    })
    
    // File response
    r.GET("/file", func(c *gin.Context) {
        c.File("./static/example.txt")
    })
    
    // Data (raw bytes)
    r.GET("/data", func(c *gin.Context) {
        c.Data(http.StatusOK, "text/plain", []byte("Raw data"))
    })
    
    r.Run(":8080")
}
```

### Custom Response Format (ตัวอย่างที่ 11)

```go
package main

import (
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
)

type Response struct {
    Success   bool        `json:"success"`
    Data      interface{} `json:"data,omitempty"`
    Error     *ErrorInfo  `json:"error,omitempty"`
    Meta      *Meta       `json:"meta,omitempty"`
    Timestamp time.Time   `json:"timestamp"`
}

type ErrorInfo struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details interface{} `json:"details,omitempty"`
}

type Meta struct {
    Page       int `json:"page,omitempty"`
    Limit      int `json:"limit,omitempty"`
    TotalItems int `json:"total_items,omitempty"`
    TotalPages int `json:"total_pages,omitempty"`
}

func successResponse(c *gin.Context, statusCode int, data interface{}, meta *Meta) {
    c.JSON(statusCode, Response{
        Success:   true,
        Data:      data,
        Meta:      meta,
        Timestamp: time.Now(),
    })
}

func errorResponse(c *gin.Context, statusCode int, code, message string, details interface{}) {
    c.JSON(statusCode, Response{
        Success: false,
        Error: &ErrorInfo{
            Code:    code,
            Message: message,
            Details: details,
        },
        Timestamp: time.Now(),
    })
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func main() {
    r := gin.Default()
    
    r.GET("/users", func(c *gin.Context) {
        users := []User{
            {ID: 1, Name: "Alice", Email: "alice@example.com"},
            {ID: 2, Name: "Bob", Email: "bob@example.com"},
        }
        
        successResponse(c, http.StatusOK, users, &Meta{
            Page:       1,
            Limit:      10,
            TotalItems: 2,
            TotalPages: 1,
        })
    })
    
    r.GET("/users/:id", func(c *gin.Context) {
        id := c.Param("id")
        if id != "1" {
            errorResponse(c, http.StatusNotFound, "USER_NOT_FOUND", "User not found", nil)
            return
        }
        successResponse(c, http.StatusOK, User{ID: 1, Name: "Alice", Email: "alice@example.com"}, nil)
    })
    
    r.Run(":8080")
}
```

---

## 28.7 Error Handling

### Centralized Error Handling (ตัวอย่างที่ 12)

```go
package main

import (
    "errors"
    "net/http"
    
    "github.com/gin-gonic/gin"
)

// Custom error types
type AppError struct {
    Code       string
    Message    string
    StatusCode int
    Err        error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return e.Message + ": " + e.Err.Error()
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// Predefined errors
var (
    ErrNotFound   = &AppError{Code: "NOT_FOUND", Message: "Resource not found", StatusCode: http.StatusNotFound}
    ErrBadRequest = &AppError{Code: "BAD_REQUEST", Message: "Invalid request", StatusCode: http.StatusBadRequest}
    ErrUnauthorized = &AppError{Code: "UNAUTHORIZED", Message: "Unauthorized", StatusCode: http.StatusUnauthorized}
    ErrInternal   = &AppError{Code: "INTERNAL_ERROR", Message: "Internal server error", StatusCode: http.StatusInternalServerError}
)

func NewNotFoundError(msg string) *AppError {
    return &AppError{Code: "NOT_FOUND", Message: msg, StatusCode: http.StatusNotFound}
}

// Error handler middleware
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        // จัดการ errors หลังจาก handler ทำงาน
        if len(c.Errors) > 0 {
            err := c.Errors.Last().Err
            
            var appErr *AppError
            if errors.As(err, &appErr) {
                c.JSON(appErr.StatusCode, gin.H{
                    "error": gin.H{
                        "code":    appErr.Code,
                        "message": appErr.Message,
                    },
                })
            } else {
                c.JSON(http.StatusInternalServerError, gin.H{
                    "error": gin.H{
                        "code":    "INTERNAL_ERROR",
                        "message": "An unexpected error occurred",
                    },
                })
            }
        }
    }
}

func getUserByID(id string) (*gin.H, error) {
    if id == "999" {
        return nil, NewNotFoundError("User with ID 999 not found")
    }
    user := gin.H{"id": id, "name": "John Doe"}
    return &user, nil
}

func main() {
    r := gin.Default()
    r.Use(ErrorHandler())
    
    r.GET("/users/:id", func(c *gin.Context) {
        id := c.Param("id")
        
        user, err := getUserByID(id)
        if err != nil {
            c.Error(err)
            return
        }
        
        c.JSON(http.StatusOK, user)
    })
    
    r.Run(":8080")
}
```

### Validation Error Handling (ตัวอย่างที่ 13)

```go
package main

import (
    "net/http"
    "strings"
    
    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
)

type ValidationErrorResponse struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func formatValidationErrors(err error) []ValidationErrorResponse {
    var errors []ValidationErrorResponse
    
    validationErrors, ok := err.(validator.ValidationErrors)
    if !ok {
        return []ValidationErrorResponse{{Field: "unknown", Message: err.Error()}}
    }
    
    for _, e := range validationErrors {
        field := strings.ToLower(e.Field())
        var message string
        
        switch e.Tag() {
        case "required":
            message = field + " is required"
        case "email":
            message = field + " must be a valid email address"
        case "min":
            message = field + " must be at least " + e.Param() + " characters"
        case "max":
            message = field + " must be at most " + e.Param() + " characters"
        case "oneof":
            message = field + " must be one of: " + e.Param()
        default:
            message = field + " is invalid"
        }
        
        errors = append(errors, ValidationErrorResponse{
            Field:   field,
            Message: message,
        })
    }
    
    return errors
}

type CreateArticleRequest struct {
    Title    string `json:"title" binding:"required,min=5,max=200"`
    Content  string `json:"content" binding:"required,min=50"`
    Status   string `json:"status" binding:"required,oneof=draft published archived"`
    AuthorID int    `json:"author_id" binding:"required,min=1"`
}

func main() {
    r := gin.Default()
    
    r.POST("/articles", func(c *gin.Context) {
        var req CreateArticleRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusUnprocessableEntity, gin.H{
                "error":  "Validation failed",
                "errors": formatValidationErrors(err),
            })
            return
        }
        
        c.JSON(http.StatusCreated, gin.H{
            "message": "Article created",
            "data":    req,
        })
    })
    
    r.Run(":8080")
}
```

---

## Workshop: Simple Blog API

### โครงสร้าง Project
```
blog-api/
├── main.go
├── models/
│   └── post.go
├── handlers/
│   └── post_handler.go
└── go.mod
```

### models/post.go

```go
package models

import "time"

type Post struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    Author    string    `json:"author"`
    Tags      []string  `json:"tags"`
    Published bool      `json:"published"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type CreatePostRequest struct {
    Title     string   `json:"title" binding:"required,min=5,max=200"`
    Content   string   `json:"content" binding:"required,min=10"`
    Author    string   `json:"author" binding:"required"`
    Tags      []string `json:"tags"`
    Published bool     `json:"published"`
}

type UpdatePostRequest struct {
    Title     string   `json:"title" binding:"min=5,max=200"`
    Content   string   `json:"content" binding:"min=10"`
    Tags      []string `json:"tags"`
    Published *bool    `json:"published"`
}

type PostFilter struct {
    Author    string `form:"author"`
    Published *bool  `form:"published"`
    Tag       string `form:"tag"`
    Page      int    `form:"page,default=1"`
    Limit     int    `form:"limit,default=10"`
}
```

### handlers/post_handler.go

```go
package handlers

import (
    "net/http"
    "sort"
    "strconv"
    "strings"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
    
    "blog-api/models"
)

type PostHandler struct {
    mu     sync.RWMutex
    posts  map[int]*models.Post
    nextID int
}

func NewPostHandler() *PostHandler {
    h := &PostHandler{
        posts:  make(map[int]*models.Post),
        nextID: 1,
    }
    
    // Seed data
    now := time.Now()
    h.posts[1] = &models.Post{
        ID: 1, Title: "Go Programming Basics",
        Content:   "Go is a statically typed, compiled programming language...",
        Author:    "John",
        Tags:      []string{"go", "programming"},
        Published: true,
        CreatedAt: now.Add(-72 * time.Hour),
        UpdatedAt: now.Add(-24 * time.Hour),
    }
    h.posts[2] = &models.Post{
        ID: 2, Title: "REST API Design",
        Content:   "REST (Representational State Transfer) is an architectural style...",
        Author:    "Jane",
        Tags:      []string{"api", "rest", "design"},
        Published: true,
        CreatedAt: now.Add(-48 * time.Hour),
        UpdatedAt: now.Add(-12 * time.Hour),
    }
    h.nextID = 3
    return h
}

// List posts พร้อม filtering และ pagination
func (h *PostHandler) ListPosts(c *gin.Context) {
    var filter models.PostFilter
    if err := c.ShouldBindQuery(&filter); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    h.mu.RLock()
    defer h.mu.RUnlock()
    
    var posts []*models.Post
    for _, p := range h.posts {
        // กรอง
        if filter.Author != "" && p.Author != filter.Author {
            continue
        }
        if filter.Published != nil && p.Published != *filter.Published {
            continue
        }
        if filter.Tag != "" {
            found := false
            for _, tag := range p.Tags {
                if tag == filter.Tag {
                    found = true
                    break
                }
            }
            if !found {
                continue
            }
        }
        posts = append(posts, p)
    }
    
    // Sort by created_at desc
    sort.Slice(posts, func(i, j int) bool {
        return posts[i].CreatedAt.After(posts[j].CreatedAt)
    })
    
    // Pagination
    total := len(posts)
    start := (filter.Page - 1) * filter.Limit
    end := start + filter.Limit
    if start >= total {
        posts = []*models.Post{}
    } else {
        if end > total {
            end = total
        }
        posts = posts[start:end]
    }
    
    totalPages := (total + filter.Limit - 1) / filter.Limit
    
    c.JSON(http.StatusOK, gin.H{
        "data": posts,
        "meta": gin.H{
            "page":        filter.Page,
            "limit":       filter.Limit,
            "total_items": total,
            "total_pages": totalPages,
        },
    })
}

// Get single post
func (h *PostHandler) GetPost(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid post ID"})
        return
    }
    
    h.mu.RLock()
    post, ok := h.posts[id]
    h.mu.RUnlock()
    
    if !ok {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"data": post})
}

// Create post
func (h *PostHandler) CreatePost(c *gin.Context) {
    var req models.CreatePostRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    h.mu.Lock()
    defer h.mu.Unlock()
    
    now := time.Now()
    post := &models.Post{
        ID:        h.nextID,
        Title:     req.Title,
        Content:   req.Content,
        Author:    req.Author,
        Tags:      req.Tags,
        Published: req.Published,
        CreatedAt: now,
        UpdatedAt: now,
    }
    
    h.posts[post.ID] = post
    h.nextID++
    
    c.JSON(http.StatusCreated, gin.H{
        "message": "Post created successfully",
        "data":    post,
    })
}

// Update post
func (h *PostHandler) UpdatePost(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid post ID"})
        return
    }
    
    var req models.UpdatePostRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    h.mu.Lock()
    defer h.mu.Unlock()
    
    post, ok := h.posts[id]
    if !ok {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }
    
    // Update only provided fields
    if req.Title != "" {
        post.Title = req.Title
    }
    if req.Content != "" {
        post.Content = req.Content
    }
    if req.Tags != nil {
        post.Tags = req.Tags
    }
    if req.Published != nil {
        post.Published = *req.Published
    }
    post.UpdatedAt = time.Now()
    
    c.JSON(http.StatusOK, gin.H{
        "message": "Post updated successfully",
        "data":    post,
    })
}

// Delete post
func (h *PostHandler) DeletePost(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid post ID"})
        return
    }
    
    h.mu.Lock()
    defer h.mu.Unlock()
    
    if _, ok := h.posts[id]; !ok {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }
    
    delete(h.posts, id)
    
    c.JSON(http.StatusOK, gin.H{"message": "Post deleted successfully"})
}

// Search posts
func (h *PostHandler) SearchPosts(c *gin.Context) {
    keyword := strings.ToLower(c.Query("q"))
    if keyword == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Search keyword required"})
        return
    }
    
    h.mu.RLock()
    defer h.mu.RUnlock()
    
    var results []*models.Post
    for _, post := range h.posts {
        if strings.Contains(strings.ToLower(post.Title), keyword) ||
            strings.Contains(strings.ToLower(post.Content), keyword) {
            results = append(results, post)
        }
    }
    
    c.JSON(http.StatusOK, gin.H{
        "data":  results,
        "query": keyword,
        "count": len(results),
    })
}
```

### main.go (Blog API)

```go
package main

import (
    "log"
    "net/http"
    
    "github.com/gin-gonic/gin"
    
    "blog-api/handlers"
)

func main() {
    gin.SetMode(gin.ReleaseMode) // หรือ gin.DebugMode
    
    r := gin.Default()
    
    // Global middleware
    r.Use(gin.Logger())
    r.Use(gin.Recovery())
    
    // CORS middleware (simplified)
    r.Use(func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        
        c.Next()
    })
    
    // Health check
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":  "ok",
            "version": "1.0.0",
        })
    })
    
    // Initialize handlers
    postHandler := handlers.NewPostHandler()
    
    // API routes
    api := r.Group("/api/v1")
    {
        posts := api.Group("/posts")
        {
            posts.GET("", postHandler.ListPosts)
            posts.POST("", postHandler.CreatePost)
            posts.GET("/search", postHandler.SearchPosts)
            posts.GET("/:id", postHandler.GetPost)
            posts.PUT("/:id", postHandler.UpdatePost)
            posts.DELETE("/:id", postHandler.DeletePost)
        }
    }
    
    log.Println("Blog API running on :8080")
    log.Println("Endpoints:")
    log.Println("  GET    /api/v1/posts")
    log.Println("  POST   /api/v1/posts")
    log.Println("  GET    /api/v1/posts/search?q=keyword")
    log.Println("  GET    /api/v1/posts/:id")
    log.Println("  PUT    /api/v1/posts/:id")
    log.Println("  DELETE /api/v1/posts/:id")
    
    r.Run(":8080")
}
```

---

## สรุป Part 28

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| REST Principles | Stateless, Uniform Interface, Client-Server |
| HTTP Methods | GET (read), POST (create), PUT (replace), PATCH (update), DELETE (delete) |
| Gin Setup | `gin.Default()`, `r.Run(":8080")` |
| Route Groups | `r.Group("/api/v1")` สำหรับ namespace |
| Path Params | `c.Param("id")` |
| Query Params | `c.Query("key")`, `c.DefaultQuery("key", "default")` |
| Request Binding | `c.ShouldBindJSON()`, `c.ShouldBind()`, `c.ShouldBindQuery()` |
| Response | `c.JSON()`, `c.String()`, `c.XML()` |
| Validation | binding tags: `required`, `min`, `max`, `email`, `oneof` |
| Error Handling | Custom error types + middleware |

### Resources
- [Gin documentation](https://gin-gonic.com/docs/)
- [RESTful API Design](https://restfulapi.net/)
- [go-playground/validator](https://github.com/go-playground/validator)
