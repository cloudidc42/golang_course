# Part 32: Logging ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ standard `log` package
- ใช้ zerolog สำหรับ high-performance logging
- ใช้ zap จาก Uber
- ใช้ slog (Go 1.21+) สำหรับ structured logging
- จัดการ Log Levels
- ทำ Structured Logging แบบ JSON
- ใช้ Correlation IDs
- ทำ Log Rotation

---

## 32.1 Standard log Package

### Basic Logging (ตัวอย่างที่ 1)

```go
package main

import (
    "log"
    "os"
)

func main() {
    // Default logger
    log.Println("Application started")
    log.Printf("Server listening on port %d", 8080)
    
    // Log แล้ว exit (ไม่ใช้ panic)
    // log.Fatal("Critical error!") // เหมือน log + os.Exit(1)
    // log.Fatalln("Fatal error")
    
    // Log แล้ว panic
    // log.Panic("Panic error!")
    
    // กำหนด flags
    log.SetFlags(log.LstdFlags | log.Lshortfile)
    log.Println("With file and line number")
    
    log.SetFlags(log.LstdFlags | log.Lmicroseconds)
    log.Println("With microseconds")
    
    // กำหนด prefix
    log.SetPrefix("[APP] ")
    log.Println("With prefix")
    
    // Log ไปยัง custom writer
    log.SetOutput(os.Stderr) // default คือ stderr
    log.Println("To stderr")
}
```

### Custom Logger (ตัวอย่างที่ 2)

```go
package main

import (
    "io"
    "log"
    "os"
)

type Logger struct {
    info  *log.Logger
    warn  *log.Logger
    error *log.Logger
    debug *log.Logger
}

func NewLogger(out io.Writer, debug bool) *Logger {
    flags := log.LstdFlags | log.Lshortfile
    l := &Logger{
        info:  log.New(out, "INFO  ", flags),
        warn:  log.New(out, "WARN  ", flags),
        error: log.New(os.Stderr, "ERROR ", flags),
    }
    if debug {
        l.debug = log.New(out, "DEBUG ", flags)
    }
    return l
}

func (l *Logger) Info(format string, v ...interface{}) {
    l.info.Printf(format, v...)
}

func (l *Logger) Warn(format string, v ...interface{}) {
    l.warn.Printf(format, v...)
}

func (l *Logger) Error(format string, v ...interface{}) {
    l.error.Printf(format, v...)
}

func (l *Logger) Debug(format string, v ...interface{}) {
    if l.debug != nil {
        l.debug.Printf(format, v...)
    }
}

func main() {
    logger := NewLogger(os.Stdout, true)
    
    logger.Info("Application starting")
    logger.Debug("Debug info: port=%d", 8080)
    logger.Warn("Memory usage is high: %d%%", 85)
    logger.Error("Failed to connect to database: %v", "connection refused")
    
    // Logger ที่ log ไปยังไฟล์
    logFile, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
    if err == nil {
        defer logFile.Close()
        fileLogger := NewLogger(io.MultiWriter(os.Stdout, logFile), false)
        fileLogger.Info("This goes to both stdout and file")
    }
}
```

---

## 32.2 zerolog

### ติดตั้ง zerolog

```bash
go get github.com/rs/zerolog
go get github.com/rs/zerolog/log
```

### Basic zerolog (ตัวอย่างที่ 3)

```go
package main

import (
    "os"
    "time"
    
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func main() {
    // ตั้งค่า global logger
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    
    // Pretty printing สำหรับ development
    log.Logger = log.Output(zerolog.ConsoleWriter{Out: os.Stderr})
    
    // Basic logging
    log.Info().Msg("Application started")
    log.Debug().Msg("This won't show (below Info level)")
    log.Warn().Msg("This is a warning")
    log.Error().Msg("This is an error")
    
    // Structured logging
    log.Info().
        Str("service", "api").
        Str("version", "1.0.0").
        Int("port", 8080).
        Msg("Server starting")
    
    // Log with fields
    log.Info().
        Str("user_id", "123").
        Str("action", "login").
        Bool("success", true).
        Dur("duration", 100*time.Millisecond).
        Msg("User action logged")
    
    // JSON output (production)
    jsonLogger := zerolog.New(os.Stdout).With().Timestamp().Logger()
    jsonLogger.Info().
        Str("event", "request").
        Str("method", "GET").
        Str("path", "/api/users").
        Int("status", 200).
        Msg("HTTP Request")
}
```

### zerolog สำหรับ HTTP Server (ตัวอย่างที่ 4)

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "time"
    
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func main() {
    // Production logger
    logger := zerolog.New(os.Stdout).
        With().
        Timestamp().
        Str("service", "api").
        Logger()
    
    // Middleware logging
    loggingMiddleware := func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            // สร้าง request logger พร้อม request ID
            requestID := r.Header.Get("X-Request-ID")
            if requestID == "" {
                requestID = fmt.Sprintf("req-%d", time.Now().UnixNano())
            }
            
            requestLogger := logger.With().
                Str("request_id", requestID).
                Str("method", r.Method).
                Str("path", r.URL.Path).
                Str("remote_addr", r.RemoteAddr).
                Logger()
            
            requestLogger.Info().Msg("Request started")
            
            next(w, r)
            
            requestLogger.Info().
                Dur("duration", time.Since(start)).
                Msg("Request completed")
        }
    }
    
    mux := http.NewServeMux()
    mux.HandleFunc("/", loggingMiddleware(func(w http.ResponseWriter, r *http.Request) {
        logger.Debug().Str("path", r.URL.Path).Msg("Handling request")
        fmt.Fprintf(w, "Hello World!")
    }))
    
    log.Info().Int("port", 8080).Msg("Server starting")
    http.ListenAndServe(":8080", mux)
}
```

### zerolog Levels และ Context (ตัวอย่างที่ 5)

```go
package main

import (
    "context"
    "os"
    
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

type contextKey string

const loggerKey contextKey = "logger"

// WithLogger เพิ่ม logger เข้า context
func WithLogger(ctx context.Context, logger zerolog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

// FromContext ดึง logger จาก context
func FromContext(ctx context.Context) zerolog.Logger {
    if logger, ok := ctx.Value(loggerKey).(zerolog.Logger); ok {
        return logger
    }
    return log.Logger
}

func processOrder(ctx context.Context, orderID string) error {
    logger := FromContext(ctx).With().
        Str("order_id", orderID).
        Logger()
    
    logger.Info().Msg("Processing order")
    
    // Simulate work
    logger.Debug().Str("step", "validate").Msg("Validating order")
    logger.Debug().Str("step", "payment").Msg("Processing payment")
    logger.Info().Msg("Order processed successfully")
    
    return nil
}

func main() {
    // Setup
    zerolog.SetGlobalLevel(zerolog.DebugLevel)
    logger := zerolog.New(os.Stdout).
        With().
        Timestamp().
        Logger()
    
    // สร้าง context พร้อม logger และ fields
    ctx := context.Background()
    ctx = WithLogger(ctx, logger.With().
        Str("user_id", "user-123").
        Str("session_id", "sess-456").
        Logger())
    
    if err := processOrder(ctx, "ord-789"); err != nil {
        log.Error().Err(err).Msg("Order processing failed")
    }
    
    // Log ด้วย error
    someErr := fmt.Errorf("connection refused")
    logger.Error().
        Err(someErr).
        Str("host", "localhost:5432").
        Msg("Database connection failed")
}

// ต้องเพิ่ม import fmt
import "fmt"
```

---

## 32.3 Zap (Uber's Logger)

### ติดตั้ง zap

```bash
go get go.uber.org/zap
```

### Basic Zap (ตัวอย่างที่ 6)

```go
package main

import (
    "time"
    
    "go.uber.org/zap"
)

func main() {
    // Production logger (JSON output, Info level)
    logger, _ := zap.NewProduction()
    defer logger.Sync() // flush buffered logs
    
    logger.Info("Application started",
        zap.String("version", "1.0.0"),
        zap.Int("port", 8080),
    )
    
    logger.Warn("High memory usage",
        zap.Int("usage_percent", 85),
    )
    
    // Development logger (pretty print)
    devLogger, _ := zap.NewDevelopment()
    defer devLogger.Sync()
    
    devLogger.Info("Development mode",
        zap.String("level", "debug"),
        zap.Bool("pretty", true),
    )
    
    // SugaredLogger - less type-safe แต่ flexible กว่า
    sugar := logger.Sugar()
    sugar.Infof("Server starting on port %d", 8080)
    sugar.Infow("Structured with sugar",
        "user_id", "123",
        "action", "login",
    )
    
    // Log with duration
    start := time.Now()
    time.Sleep(10 * time.Millisecond)
    logger.Info("Operation completed",
        zap.Duration("duration", time.Since(start)),
    )
}
```

### Custom Zap Logger (ตัวอย่างที่ 7)

```go
package main

import (
    "os"
    
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func NewLogger(env string) (*zap.Logger, error) {
    var config zap.Config
    
    if env == "production" {
        config = zap.NewProductionConfig()
        config.EncoderConfig.TimeKey = "timestamp"
        config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    } else {
        config = zap.NewDevelopmentConfig()
        config.EncoderConfig.EncodeLevel = zapcore.CapitalColorLevelEncoder
    }
    
    // Level
    config.Level = zap.NewAtomicLevelAt(zapcore.DebugLevel)
    
    return config.Build(
        zap.AddCaller(),
        zap.AddStacktrace(zapcore.ErrorLevel),
    )
}

func NewCustomLogger(level zapcore.Level) *zap.Logger {
    encoderConfig := zapcore.EncoderConfig{
        TimeKey:        "ts",
        LevelKey:       "level",
        NameKey:        "logger",
        CallerKey:      "caller",
        MessageKey:     "msg",
        StacktraceKey:  "stacktrace",
        LineEnding:     zapcore.DefaultLineEnding,
        EncodeLevel:    zapcore.LowercaseLevelEncoder,
        EncodeTime:     zapcore.ISO8601TimeEncoder,
        EncodeDuration: zapcore.MillisDurationEncoder,
        EncodeCaller:   zapcore.ShortCallerEncoder,
    }
    
    core := zapcore.NewCore(
        zapcore.NewJSONEncoder(encoderConfig),
        zapcore.AddSync(os.Stdout),
        level,
    )
    
    return zap.New(core, zap.AddCaller())
}

func main() {
    // Development
    devLogger, _ := NewLogger("development")
    defer devLogger.Sync()
    
    devLogger.Info("Development logger ready",
        zap.String("env", "development"),
    )
    
    // Custom
    customLogger := NewCustomLogger(zapcore.DebugLevel)
    defer customLogger.Sync()
    
    customLogger.Debug("Debug message",
        zap.String("key", "value"),
        zap.Int("number", 42),
    )
    
    customLogger.Info("Info message with multiple fields",
        zap.String("user_id", "123"),
        zap.String("request_id", "req-abc"),
        zap.Int("status", 200),
    )
    
    // Named logger
    userLogger := customLogger.Named("user-service")
    userLogger.Info("User created",
        zap.String("email", "user@example.com"),
    )
}
```

---

## 32.4 slog (Go 1.21+)

### Basic slog (ตัวอย่างที่ 8)

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // Default logger (writes to stderr)
    slog.Info("Application started")
    slog.Debug("Debug message")     // ไม่แสดง (default level = Info)
    slog.Warn("Warning message")
    slog.Error("Error occurred")
    
    // Structured logging
    slog.Info("User action",
        "user_id", "123",
        "action", "login",
        "ip", "192.168.1.1",
    )
    
    // JSON Handler
    jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))
    
    jsonLogger.Info("JSON log",
        "method", "GET",
        "path", "/api/users",
        "status", 200,
    )
    
    // Text Handler
    textLogger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
        AddSource: true, // เพิ่ม file:line
    }))
    
    textLogger.Debug("Text log with source",
        "key", "value",
    )
    
    // Set as default logger
    slog.SetDefault(jsonLogger)
    slog.Info("Now using JSON by default")
}
```

### slog กับ Attributes (ตัวอย่างที่ 9)

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))
    
    // slog.Attr types
    logger.Info("Request processed",
        slog.String("method", "POST"),
        slog.String("path", "/api/users"),
        slog.Int("status", 201),
        slog.Duration("latency", 150*time.Millisecond),
        slog.Bool("cached", false),
        slog.Time("timestamp", time.Now()),
    )
    
    // Group attributes
    logger.Info("Database operation",
        slog.Group("db",
            slog.String("driver", "postgres"),
            slog.String("host", "localhost"),
            slog.Int("port", 5432),
            slog.Duration("query_time", 50*time.Millisecond),
        ),
    )
    
    // With - เพิ่ม default fields
    requestLogger := logger.With(
        slog.String("request_id", "req-123"),
        slog.String("user_id", "user-456"),
    )
    
    requestLogger.Info("Handler called")
    requestLogger.Info("Handler completed", slog.Int("status", 200))
    
    // Context-aware logging
    ctx := context.Background()
    logger.InfoContext(ctx, "Context logging",
        slog.String("trace_id", "trace-789"),
    )
    
    // Log level check (performance optimization)
    if logger.Enabled(ctx, slog.LevelDebug) {
        // Expensive computation only if debug is enabled
        value := computeExpensiveDebugValue()
        logger.Debug("Computed value", slog.Any("value", value))
    }
}

func computeExpensiveDebugValue() interface{} {
    return map[string]interface{}{
        "memory_usage": "256MB",
        "goroutines":   42,
    }
}
```

### Custom slog Handler (ตัวอย่างที่ 10)

```go
package main

import (
    "context"
    "io"
    "log/slog"
    "os"
    "time"
)

// ColorHandler เป็น custom handler ที่ใส่สี
type ColorHandler struct {
    writer io.Writer
    opts   slog.HandlerOptions
}

func NewColorHandler(w io.Writer, opts *slog.HandlerOptions) *ColorHandler {
    h := &ColorHandler{writer: w}
    if opts != nil {
        h.opts = *opts
    }
    return h
}

var levelColors = map[slog.Level]string{
    slog.LevelDebug: "\033[36m", // Cyan
    slog.LevelInfo:  "\033[32m", // Green
    slog.LevelWarn:  "\033[33m", // Yellow
    slog.LevelError: "\033[31m", // Red
}

const resetColor = "\033[0m"

func (h *ColorHandler) Enabled(_ context.Context, level slog.Level) bool {
    return level >= h.opts.Level.Level()
}

func (h *ColorHandler) Handle(_ context.Context, r slog.Record) error {
    color := levelColors[r.Level]
    
    msg := color + r.Level.String() + resetColor
    msg += " " + r.Time.Format(time.RFC3339)
    msg += " " + r.Message
    
    r.Attrs(func(a slog.Attr) bool {
        msg += " " + a.Key + "=" + a.Value.String()
        return true
    })
    
    _, err := io.WriteString(h.writer, msg+"\n")
    return err
}

func (h *ColorHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return h // simplified
}

func (h *ColorHandler) WithGroup(name string) slog.Handler {
    return h // simplified
}

func main() {
    colorLogger := slog.New(NewColorHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))
    
    colorLogger.Debug("Debug message")
    colorLogger.Info("Info message")
    colorLogger.Warn("Warning message")
    colorLogger.Error("Error message")
    colorLogger.Info("With fields",
        slog.String("user", "alice"),
        slog.Int("age", 30),
    )
}
```

---

## 32.5 Structured Logging

### Log Structure ที่ดี (ตัวอย่างที่ 11)

```go
package main

import (
    "log/slog"
    "os"
    "time"
)

// Standard log fields
type LogFields struct {
    Service   string
    Version   string
    Env       string
    RequestID string
    UserID    string
    TraceID   string
}

func main() {
    // สร้าง base logger พร้อม service info
    baseLogger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })).With(
        slog.String("service", "user-api"),
        slog.String("version", "1.2.3"),
        slog.String("env", "production"),
    )
    
    // HTTP Request log
    baseLogger.Info("http_request",
        slog.Group("request",
            slog.String("method", "GET"),
            slog.String("path", "/api/users/123"),
            slog.String("query", "include_posts=true"),
            slog.String("ip", "192.168.1.100"),
            slog.String("user_agent", "Mozilla/5.0"),
        ),
        slog.Group("response",
            slog.Int("status", 200),
            slog.Int("size", 1234),
            slog.Duration("latency", 45*time.Millisecond),
        ),
        slog.String("trace_id", "trace-abc-123"),
        slog.String("request_id", "req-xyz-456"),
    )
    
    // Database query log
    baseLogger.Info("db_query",
        slog.Group("query",
            slog.String("operation", "SELECT"),
            slog.String("table", "users"),
            slog.Duration("duration", 12*time.Millisecond),
            slog.Int("rows_affected", 1),
        ),
        slog.String("trace_id", "trace-abc-123"),
    )
    
    // Business event log
    baseLogger.Info("user_created",
        slog.Group("user",
            slog.String("id", "user-789"),
            slog.String("email", "new@example.com"),
            slog.String("plan", "pro"),
        ),
        slog.String("actor_id", "admin-001"),
        slog.Time("timestamp", time.Now()),
    )
    
    // Error log
    baseLogger.Error("payment_failed",
        slog.Group("payment",
            slog.String("order_id", "ord-123"),
            slog.Float64("amount", 1999.99),
            slog.String("currency", "THB"),
            slog.String("provider", "stripe"),
        ),
        slog.String("error", "card_declined"),
        slog.String("error_code", "insufficient_funds"),
        slog.String("trace_id", "trace-abc-123"),
    )
}
```

---

## 32.6 Correlation IDs

### Correlation ID Middleware (ตัวอย่างที่ 12)

```go
package main

import (
    "context"
    "fmt"
    "log/slog"
    "math/rand"
    "net/http"
    "os"
    "time"
)

type contextKey string

const (
    correlationIDKey contextKey = "correlation_id"
    loggerKey        contextKey = "logger"
)

func generateCorrelationID() string {
    return fmt.Sprintf("%d-%06d", time.Now().UnixNano(), rand.Intn(1000000))
}

func withCorrelationID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, correlationIDKey, id)
}

func getCorrelationID(ctx context.Context) string {
    if id, ok := ctx.Value(correlationIDKey).(string); ok {
        return id
    }
    return ""
}

func withLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func getLogger(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

// CorrelationMiddleware เพิ่ม correlation ID ทุก request
func CorrelationMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // ใช้ existing หรือสร้างใหม่
            correlationID := r.Header.Get("X-Correlation-ID")
            if correlationID == "" {
                correlationID = generateCorrelationID()
            }
            
            // เพิ่มใน response header
            w.Header().Set("X-Correlation-ID", correlationID)
            
            // สร้าง request logger
            requestLogger := logger.With(
                slog.String("correlation_id", correlationID),
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
            )
            
            // เก็บใน context
            ctx := withCorrelationID(r.Context(), correlationID)
            ctx = withLogger(ctx, requestLogger)
            
            start := time.Now()
            requestLogger.Info("Request started")
            
            next.ServeHTTP(w, r.WithContext(ctx))
            
            requestLogger.Info("Request completed",
                slog.Duration("duration", time.Since(start)),
            )
        })
    }
}

func processUserData(ctx context.Context, userID string) error {
    logger := getLogger(ctx)
    logger.Info("Processing user data",
        slog.String("user_id", userID),
    )
    
    // Simulate sub-operations
    if err := fetchUserFromDB(ctx, userID); err != nil {
        return err
    }
    
    logger.Info("User data processed successfully",
        slog.String("user_id", userID),
    )
    return nil
}

func fetchUserFromDB(ctx context.Context, userID string) error {
    logger := getLogger(ctx)
    start := time.Now()
    
    // Simulate DB query
    time.Sleep(10 * time.Millisecond)
    
    logger.Debug("DB query executed",
        slog.String("operation", "SELECT"),
        slog.String("user_id", userID),
        slog.Duration("duration", time.Since(start)),
    )
    return nil
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))
    
    mux := http.NewServeMux()
    mux.HandleFunc("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        userID := r.PathValue("id")
        if userID == "" {
            userID = "123" // fallback for older Go versions
        }
        
        if err := processUserData(r.Context(), userID); err != nil {
            http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            return
        }
        
        fmt.Fprintf(w, `{"user_id": "%s"}`, userID)
    })
    
    handler := CorrelationMiddleware(logger)(mux)
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", handler)
}
```

---

## 32.7 Log Rotation

### ใช้ lumberjack สำหรับ Log Rotation (ตัวอย่างที่ 13)

```bash
go get gopkg.in/natefinish.lumberjack.v2
# หรือ
go get gopkg.in/lumberjack.v2
```

```go
package main

import (
    "io"
    "log/slog"
    "os"
    
    "gopkg.in/lumberjack.v2"
)

func NewRotatingLogger(logFile string, level slog.Level) *slog.Logger {
    // ตั้งค่า log rotation
    rotator := &lumberjack.Logger{
        Filename:   logFile,  // ชื่อไฟล์
        MaxSize:    100,      // MB ก่อน rotate
        MaxBackups: 5,        // เก็บ backup ไว้กี่ไฟล์
        MaxAge:     30,       // วัน ก่อนลบ
        Compress:   true,     // gzip compressed backups
    }
    
    // Write ไปทั้ง file และ stdout
    multiWriter := io.MultiWriter(os.Stdout, rotator)
    
    handler := slog.NewJSONHandler(multiWriter, &slog.HandlerOptions{
        Level: level,
    })
    
    return slog.New(handler)
}

func main() {
    logger := NewRotatingLogger("logs/app.log", slog.LevelInfo)
    
    // Log messages จะไปทั้ง stdout และ logs/app.log
    logger.Info("Application started with log rotation")
    logger.Warn("This is a warning")
    logger.Error("This is an error",
        slog.String("error", "something went wrong"),
    )
    
    // ทำ many logs เพื่อทดสอบ rotation
    for i := 0; i < 100; i++ {
        logger.Info("Log message",
            slog.Int("index", i),
            slog.String("data", "some data payload"),
        )
    }
    
    slog.SetDefault(logger)
    slog.Info("Using rotated logger as default")
}
```

---

## Workshop: Production Logger

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log/slog"
    "os"
    "time"
)

// ProductionLogger wraps slog with helpful features
type ProductionLogger struct {
    logger *slog.Logger
    level  slog.Level
}

type LoggerConfig struct {
    Level      string // debug, info, warn, error
    Format     string // json, text
    Output     io.Writer
    Service    string
    Version    string
    Env        string
}

func NewProductionLogger(cfg LoggerConfig) *ProductionLogger {
    // Parse level
    var level slog.Level
    switch cfg.Level {
    case "debug":
        level = slog.LevelDebug
    case "warn":
        level = slog.LevelWarn
    case "error":
        level = slog.LevelError
    default:
        level = slog.LevelInfo
    }
    
    if cfg.Output == nil {
        cfg.Output = os.Stdout
    }
    
    opts := &slog.HandlerOptions{
        Level:     level,
        AddSource: cfg.Level == "debug",
    }
    
    var handler slog.Handler
    if cfg.Format == "json" {
        handler = slog.NewJSONHandler(cfg.Output, opts)
    } else {
        handler = slog.NewTextHandler(cfg.Output, opts)
    }
    
    logger := slog.New(handler).With(
        slog.String("service", cfg.Service),
        slog.String("version", cfg.Version),
        slog.String("env", cfg.Env),
    )
    
    return &ProductionLogger{
        logger: logger,
        level:  level,
    }
}

// WithContext สร้าง logger พร้อม context fields
func (l *ProductionLogger) WithContext(ctx context.Context) *ProductionLogger {
    fields := []any{}
    
    if traceID, ok := ctx.Value("trace_id").(string); ok {
        fields = append(fields, slog.String("trace_id", traceID))
    }
    if requestID, ok := ctx.Value("request_id").(string); ok {
        fields = append(fields, slog.String("request_id", requestID))
    }
    if userID, ok := ctx.Value("user_id").(string); ok {
        fields = append(fields, slog.String("user_id", userID))
    }
    
    return &ProductionLogger{
        logger: l.logger.With(fields...),
        level:  l.level,
    }
}

// HTTP Request logging
func (l *ProductionLogger) LogHTTPRequest(method, path, requestID string, status int, duration time.Duration, size int) {
    level := slog.LevelInfo
    if status >= 500 {
        level = slog.LevelError
    } else if status >= 400 {
        level = slog.LevelWarn
    }
    
    l.logger.Log(context.Background(), level, "http_request",
        slog.String("method", method),
        slog.String("path", path),
        slog.String("request_id", requestID),
        slog.Int("status", status),
        slog.Duration("duration", duration),
        slog.Int("response_size", size),
    )
}

// Database logging
func (l *ProductionLogger) LogDBQuery(operation, table string, duration time.Duration, rowsAffected int64, err error) {
    if err != nil {
        l.logger.Error("db_query_error",
            slog.String("operation", operation),
            slog.String("table", table),
            slog.Duration("duration", duration),
            slog.String("error", err.Error()),
        )
        return
    }
    
    l.logger.Debug("db_query",
        slog.String("operation", operation),
        slog.String("table", table),
        slog.Duration("duration", duration),
        slog.Int64("rows_affected", rowsAffected),
    )
}

// Business event logging
func (l *ProductionLogger) LogEvent(event string, fields ...slog.Attr) {
    attrs := make([]any, len(fields))
    for i, f := range fields {
        attrs[i] = f
    }
    l.logger.Info(event, attrs...)
}

// Error logging
func (l *ProductionLogger) LogError(msg string, err error, fields ...slog.Attr) {
    attrs := []any{slog.String("error", err.Error())}
    for _, f := range fields {
        attrs = append(attrs, f)
    }
    l.logger.Error(msg, attrs...)
}

func main() {
    // สร้าง production logger
    prodLogger := NewProductionLogger(LoggerConfig{
        Level:   "debug",
        Format:  "json",
        Output:  os.Stdout,
        Service: "user-api",
        Version: "2.0.0",
        Env:     "production",
    })
    
    // Simulate HTTP requests
    prodLogger.LogHTTPRequest("GET", "/api/users", "req-001", 200, 45*time.Millisecond, 1234)
    prodLogger.LogHTTPRequest("POST", "/api/users", "req-002", 201, 120*time.Millisecond, 456)
    prodLogger.LogHTTPRequest("GET", "/api/users/999", "req-003", 404, 5*time.Millisecond, 89)
    
    // Simulate DB queries
    prodLogger.LogDBQuery("SELECT", "users", 12*time.Millisecond, 1, nil)
    prodLogger.LogDBQuery("INSERT", "users", 25*time.Millisecond, 1, nil)
    prodLogger.LogDBQuery("UPDATE", "orders", 0, 0, fmt.Errorf("connection timeout"))
    
    // Business events
    prodLogger.LogEvent("user_registered",
        slog.String("user_id", "user-123"),
        slog.String("email", "new@example.com"),
        slog.String("plan", "free"),
    )
    
    prodLogger.LogEvent("payment_processed",
        slog.String("order_id", "ord-456"),
        slog.Float64("amount", 999.00),
        slog.String("currency", "THB"),
    )
    
    // Context-aware logging
    ctx := context.WithValue(context.Background(), "trace_id", "trace-789")
    ctx = context.WithValue(ctx, "user_id", "user-123")
    
    ctxLogger := prodLogger.WithContext(ctx)
    ctxLogger.LogEvent("cart_updated",
        slog.Int("item_count", 3),
        slog.Float64("total", 2999.00),
    )
    
    fmt.Println("\n=== Production Logger Demo Complete ===")
}
```

---

## สรุป Part 32

| Logger | เมื่อใช้ | Performance |
|--------|---------|-------------|
| `log` package | ง่ายๆ, เล็กๆ, internal | ปานกลาง |
| zerolog | Production, JSON, low latency | สูงมาก (zero-alloc) |
| zap | High throughput, Uber-tested | สูงมาก |
| slog | Standard library, Go 1.21+ | ดี |

### Best Practices
- ใช้ Structured Logging (JSON) ใน production
- มี log levels และกรอง level ที่เหมาะสม
- เพิ่ม Correlation/Request ID ทุก request
- Log ที่ boundaries (HTTP, DB, external services)
- อย่า log sensitive data (passwords, tokens, PII)
- ใช้ log rotation ใน production

### Resources
- [log/slog documentation](https://pkg.go.dev/log/slog)
- [zerolog](https://github.com/rs/zerolog)
- [zap](https://github.com/uber-go/zap)
- [lumberjack](https://github.com/natefinish/lumberjack)
