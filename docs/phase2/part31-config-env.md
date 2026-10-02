# Part 31: Configuration และ Environment Variables

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `os.Getenv` อ่าน environment variables
- ใช้ `.env` files ด้วย godotenv
- ใช้ Viper configuration library
- สร้าง Configuration structs
- Validate configuration
- จัดการ Multiple environments
- เข้าใจ Secrets management

---

## 31.1 Environment Variables พื้นฐาน

### os.Getenv (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // อ่าน environment variable
    dbHost := os.Getenv("DB_HOST")
    dbPort := os.Getenv("DB_PORT")
    
    fmt.Println("DB_HOST:", dbHost)
    fmt.Println("DB_PORT:", dbPort)
    
    // ถ้าไม่มีจะได้ "" (empty string)
    if dbHost == "" {
        fmt.Println("DB_HOST is not set!")
    }
}
```

### os.LookupEnv - ตรวจสอบว่า set ไว้ไหม (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func getEnvOrDefault(key, defaultValue string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return defaultValue
}

func getEnvAsInt(key string, defaultValue int) int {
    value := os.Getenv(key)
    if value == "" {
        return defaultValue
    }
    
    intValue, err := strconv.Atoi(value)
    if err != nil {
        fmt.Printf("Warning: %s is not a valid integer, using default %d\n", key, defaultValue)
        return defaultValue
    }
    return intValue
}

func getEnvAsBool(key string, defaultValue bool) bool {
    value := os.Getenv(key)
    if value == "" {
        return defaultValue
    }
    
    boolValue, err := strconv.ParseBool(value)
    if err != nil {
        return defaultValue
    }
    return boolValue
}

func main() {
    // ตั้งค่า env สำหรับทดสอบ
    os.Setenv("APP_NAME", "MyGoApp")
    os.Setenv("APP_PORT", "8080")
    os.Setenv("DEBUG", "true")
    
    appName := getEnvOrDefault("APP_NAME", "DefaultApp")
    appPort := getEnvAsInt("APP_PORT", 3000)
    debug := getEnvAsBool("DEBUG", false)
    missingVar := getEnvOrDefault("MISSING_VAR", "default-value")
    
    fmt.Printf("App: %s\n", appName)
    fmt.Printf("Port: %d\n", appPort)
    fmt.Printf("Debug: %v\n", debug)
    fmt.Printf("Missing: %s\n", missingVar)
    
    // Check if variable exists
    if val, ok := os.LookupEnv("APP_NAME"); ok {
        fmt.Printf("APP_NAME is set to: %s\n", val)
    } else {
        fmt.Println("APP_NAME is NOT set")
    }
}
```

### ตั้งค่า Environment Variables (ตัวอย่างที่ 3)

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Set env var
    os.Setenv("MY_VAR", "my-value")
    
    // Get env var
    fmt.Println("MY_VAR:", os.Getenv("MY_VAR"))
    
    // Unset env var
    os.Unsetenv("MY_VAR")
    fmt.Println("After unset:", os.Getenv("MY_VAR"))
    
    // Expand env variables in string
    os.Setenv("HOME", "/home/user")
    os.Setenv("APP_NAME", "myapp")
    
    path := os.ExpandEnv("${HOME}/apps/${APP_NAME}/config.yaml")
    fmt.Println("Path:", path)
    
    // ดูทุก env vars
    fmt.Println("\nAll env vars count:", len(os.Environ()))
}
```

---

## 31.2 .env Files ด้วย godotenv

### ติดตั้ง godotenv

```bash
go get github.com/joho/godotenv
```

### สร้าง .env file
```bash
# .env
APP_NAME=MyGoApp
APP_PORT=8080
APP_ENV=development

DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=postgres
DB_PASSWORD=secret

REDIS_URL=redis://localhost:6379
JWT_SECRET=my-super-secret-key
```

### ใช้ godotenv (ตัวอย่างที่ 4)

```go
package main

import (
    "fmt"
    "log"
    "os"
    
    "github.com/joho/godotenv"
)

func main() {
    // Load .env file
    if err := godotenv.Load(); err != nil {
        log.Println("Warning: .env file not found, using system environment variables")
    }
    
    // อ่าน vars
    appName := os.Getenv("APP_NAME")
    appPort := os.Getenv("APP_PORT")
    dbHost := os.Getenv("DB_HOST")
    
    fmt.Printf("App: %s running on port %s\n", appName, appPort)
    fmt.Printf("Database host: %s\n", dbHost)
}
```

### Load หลาย .env files (ตัวอย่างที่ 5)

```go
package main

import (
    "fmt"
    "log"
    "os"
    
    "github.com/joho/godotenv"
)

func loadConfig(env string) {
    // Load .env ก่อน (defaults)
    godotenv.Load(".env")
    
    // Override ด้วย environment-specific file
    envFile := fmt.Sprintf(".env.%s", env)
    if err := godotenv.Load(envFile); err != nil {
        log.Printf("Note: %s not found\n", envFile)
    }
    
    // .env.local (ไม่ commit ลงใน git - สำหรับ local development)
    godotenv.Load(".env.local")
}

func main() {
    env := os.Getenv("APP_ENV")
    if env == "" {
        env = "development"
    }
    
    loadConfig(env)
    
    fmt.Println("APP_NAME:", os.Getenv("APP_NAME"))
    fmt.Println("APP_ENV:", os.Getenv("APP_ENV"))
}
```

### godotenv.Read - อ่านเป็น map (ตัวอย่างที่ 6)

```go
package main

import (
    "fmt"
    "log"
    
    "github.com/joho/godotenv"
)

func main() {
    // อ่านเป็น map โดยไม่ set ลงใน os.Environ
    envMap, err := godotenv.Read(".env")
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Variables in .env:")
    for key, value := range envMap {
        // ซ่อน secret values
        if key == "DB_PASSWORD" || key == "JWT_SECRET" {
            fmt.Printf("  %s = [REDACTED]\n", key)
        } else {
            fmt.Printf("  %s = %s\n", key, value)
        }
    }
}
```

---

## 31.3 Configuration Struct

### Config Struct Pattern (ตัวอย่างที่ 7)

```go
package main

import (
    "fmt"
    "log"
    "os"
    "strconv"
    "time"
)

// Config เก็บ configuration ทั้งหมดของ application
type Config struct {
    App      AppConfig
    Database DatabaseConfig
    Redis    RedisConfig
    JWT      JWTConfig
}

type AppConfig struct {
    Name    string
    Port    int
    Env     string
    Debug   bool
    Timeout time.Duration
}

type DatabaseConfig struct {
    Host            string
    Port            int
    Name            string
    User            string
    Password        string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

type RedisConfig struct {
    URL      string
    Password string
    DB       int
}

type JWTConfig struct {
    Secret          string
    ExpiresIn       time.Duration
    RefreshExpiresIn time.Duration
}

// LoadConfig อ่าน configuration จาก environment variables
func LoadConfig() (*Config, error) {
    cfg := &Config{}
    
    // App config
    cfg.App.Name = getEnv("APP_NAME", "MyApp")
    cfg.App.Port = getEnvInt("APP_PORT", 8080)
    cfg.App.Env = getEnv("APP_ENV", "development")
    cfg.App.Debug = getEnvBool("APP_DEBUG", cfg.App.Env == "development")
    cfg.App.Timeout = getEnvDuration("APP_TIMEOUT", 30*time.Second)
    
    // Database config
    cfg.Database.Host = getEnv("DB_HOST", "localhost")
    cfg.Database.Port = getEnvInt("DB_PORT", 5432)
    cfg.Database.Name = requireEnv("DB_NAME")
    cfg.Database.User = requireEnv("DB_USER")
    cfg.Database.Password = requireEnv("DB_PASSWORD")
    cfg.Database.MaxOpenConns = getEnvInt("DB_MAX_OPEN_CONNS", 25)
    cfg.Database.MaxIdleConns = getEnvInt("DB_MAX_IDLE_CONNS", 10)
    cfg.Database.ConnMaxLifetime = getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute)
    
    // Redis config
    cfg.Redis.URL = getEnv("REDIS_URL", "redis://localhost:6379")
    cfg.Redis.Password = getEnv("REDIS_PASSWORD", "")
    cfg.Redis.DB = getEnvInt("REDIS_DB", 0)
    
    // JWT config
    cfg.JWT.Secret = requireEnv("JWT_SECRET")
    cfg.JWT.ExpiresIn = getEnvDuration("JWT_EXPIRES_IN", 24*time.Hour)
    cfg.JWT.RefreshExpiresIn = getEnvDuration("JWT_REFRESH_EXPIRES_IN", 7*24*time.Hour)
    
    return cfg, nil
}

// Helper functions
func getEnv(key, defaultValue string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return defaultValue
}

func requireEnv(key string) string {
    value, ok := os.LookupEnv(key)
    if !ok || value == "" {
        log.Fatalf("Required environment variable %s is not set", key)
    }
    return value
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intVal, err := strconv.Atoi(value); err == nil {
            return intVal
        }
        log.Printf("Warning: %s is not a valid int, using default %d\n", key, defaultValue)
    }
    return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if boolVal, err := strconv.ParseBool(value); err == nil {
            return boolVal
        }
    }
    return defaultValue
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if d, err := time.ParseDuration(value); err == nil {
            return d
        }
    }
    return defaultValue
}

func main() {
    // ตั้งค่า env vars สำหรับทดสอบ
    os.Setenv("APP_NAME", "TestApp")
    os.Setenv("DB_NAME", "testdb")
    os.Setenv("DB_USER", "postgres")
    os.Setenv("DB_PASSWORD", "secret")
    os.Setenv("JWT_SECRET", "my-jwt-secret")
    
    cfg, err := LoadConfig()
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("App: %s (port %d, env: %s)\n", cfg.App.Name, cfg.App.Port, cfg.App.Env)
    fmt.Printf("DB: %s@%s:%d/%s\n", cfg.Database.User, cfg.Database.Host, cfg.Database.Port, cfg.Database.Name)
    fmt.Printf("JWT expires in: %v\n", cfg.JWT.ExpiresIn)
}
```

---

## 31.4 Viper Configuration Library

### ติดตั้ง Viper

```bash
go get github.com/spf13/viper
```

### Basic Viper Usage (ตัวอย่างที่ 8)

```go
package main

import (
    "fmt"
    "log"
    
    "github.com/spf13/viper"
)

func main() {
    // ตั้งค่า defaults
    viper.SetDefault("app.name", "MyApp")
    viper.SetDefault("app.port", 8080)
    viper.SetDefault("app.debug", false)
    viper.SetDefault("database.max_open_conns", 25)
    
    // อ่านจาก environment variables
    viper.AutomaticEnv() // อ่าน env vars อัตโนมัติ
    viper.SetEnvPrefix("APP") // prefix: APP_NAME จะ map กับ "name"
    
    // อ่าน config file
    viper.SetConfigName("config")   // ชื่อไฟล์ (ไม่รวม extension)
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")        // current directory
    viper.AddConfigPath("./config")
    viper.AddConfigPath("$HOME/.myapp")
    
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); ok {
            log.Println("Config file not found, using defaults and env vars")
        } else {
            log.Fatal("Config error:", err)
        }
    }
    
    // อ่านค่า
    appName := viper.GetString("app.name")
    appPort := viper.GetInt("app.port")
    debug := viper.GetBool("app.debug")
    maxConns := viper.GetInt("database.max_open_conns")
    
    fmt.Printf("App: %s\n", appName)
    fmt.Printf("Port: %d\n", appPort)
    fmt.Printf("Debug: %v\n", debug)
    fmt.Printf("Max DB connections: %d\n", maxConns)
}
```

### Viper กับ YAML Config File (ตัวอย่างที่ 9)

สร้างไฟล์ `config.yaml`:
```yaml
app:
  name: MyGoApp
  port: 8080
  env: development
  debug: true

database:
  host: localhost
  port: 5432
  name: myapp
  user: postgres
  password: secret
  pool:
    max_open: 25
    max_idle: 10

redis:
  addr: localhost:6379
  db: 0

jwt:
  secret: my-jwt-secret
  expires_in: 24h

features:
  feature_a: true
  feature_b: false
```

```go
package main

import (
    "fmt"
    "log"
    "time"
    
    "github.com/spf13/viper"
)

type AppConfig struct {
    Name  string
    Port  int
    Env   string
    Debug bool
}

type DatabaseConfig struct {
    Host     string
    Port     int
    Name     string
    User     string
    Password string
    Pool     struct {
        MaxOpen int `mapstructure:"max_open"`
        MaxIdle int `mapstructure:"max_idle"`
    }
}

type Config struct {
    App      AppConfig      `mapstructure:"app"`
    Database DatabaseConfig `mapstructure:"database"`
    JWT struct {
        Secret    string
        ExpiresIn time.Duration `mapstructure:"expires_in"`
    } `mapstructure:"jwt"`
}

func LoadViperConfig() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AddConfigPath("./config")
    
    // Env vars override config file
    viper.AutomaticEnv()
    
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, err
        }
    }
    
    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshal config: %w", err)
    }
    
    return &cfg, nil
}

func main() {
    cfg, err := LoadViperConfig()
    if err != nil {
        log.Fatal("Config error:", err)
    }
    
    fmt.Printf("App: %s (port: %d, env: %s)\n", cfg.App.Name, cfg.App.Port, cfg.App.Env)
    fmt.Printf("DB: %s@%s:%d/%s\n", cfg.Database.User, cfg.Database.Host, cfg.Database.Port, cfg.Database.Name)
}
```

---

## 31.5 Config Validation

### ใช้ go-playground/validator (ตัวอย่างที่ 10)

```go
package main

import (
    "fmt"
    "log"
    "os"
    
    "github.com/go-playground/validator/v10"
)

type Config struct {
    App struct {
        Name    string `validate:"required,min=2,max=50"`
        Port    int    `validate:"required,min=1,max=65535"`
        Env     string `validate:"required,oneof=development staging production"`
    }
    Database struct {
        Host     string `validate:"required"`
        Port     int    `validate:"required,min=1,max=65535"`
        Name     string `validate:"required"`
        User     string `validate:"required"`
        Password string `validate:"required,min=8"`
    }
    JWT struct {
        Secret string `validate:"required,min=32"`
    }
}

func loadAndValidateConfig() (*Config, error) {
    cfg := &Config{}
    
    // App
    cfg.App.Name = getEnv("APP_NAME", "MyApp")
    cfg.App.Port = getEnvInt("APP_PORT", 8080)
    cfg.App.Env = getEnv("APP_ENV", "development")
    
    // Database
    cfg.Database.Host = getEnv("DB_HOST", "localhost")
    cfg.Database.Port = getEnvInt("DB_PORT", 5432)
    cfg.Database.Name = os.Getenv("DB_NAME")
    cfg.Database.User = os.Getenv("DB_USER")
    cfg.Database.Password = os.Getenv("DB_PASSWORD")
    
    // JWT
    cfg.JWT.Secret = os.Getenv("JWT_SECRET")
    
    // Validate
    validate := validator.New()
    if err := validate.Struct(cfg); err != nil {
        return nil, fmt.Errorf("config validation failed: %w", err)
    }
    
    return cfg, nil
}

func getEnv(key, defaultVal string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultVal
}

func getEnvInt(key string, defaultVal int) int {
    if v := os.Getenv(key); v != "" {
        var i int
        fmt.Sscanf(v, "%d", &i)
        return i
    }
    return defaultVal
}

func main() {
    // ตั้งค่า env vars ที่ถูกต้อง
    os.Setenv("DB_NAME", "testdb")
    os.Setenv("DB_USER", "postgres")
    os.Setenv("DB_PASSWORD", "securepassword123")
    os.Setenv("JWT_SECRET", "this-is-a-very-long-secret-key-at-least-32-chars")
    
    cfg, err := loadAndValidateConfig()
    if err != nil {
        log.Fatal("Config error:", err)
    }
    
    fmt.Printf("Config loaded and validated!\n")
    fmt.Printf("App: %s (port %d, env %s)\n", cfg.App.Name, cfg.App.Port, cfg.App.Env)
    
    // ทดสอบ validation failure
    os.Setenv("APP_ENV", "invalid-env")
    _, err = loadAndValidateConfig()
    if err != nil {
        fmt.Println("\nExpected validation error:", err)
    }
}
```

---

## 31.6 Multiple Environments

### Environment-based Configuration (ตัวอย่างที่ 11)

```go
package main

import (
    "fmt"
    "os"
    "time"
)

type Config struct {
    App      AppConfig
    Database DatabaseConfig
    Cache    CacheConfig
    Logging  LoggingConfig
}

type AppConfig struct {
    Name    string
    Port    int
    Env     string
    Debug   bool
    Origins []string // CORS origins
}

type DatabaseConfig struct {
    URL             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

type CacheConfig struct {
    URL string
    TTL time.Duration
}

type LoggingConfig struct {
    Level  string
    Format string // json or text
}

// Development config
func developmentConfig() *Config {
    return &Config{
        App: AppConfig{
            Name:    "MyApp",
            Port:    8080,
            Env:     "development",
            Debug:   true,
            Origins: []string{"http://localhost:3000", "http://localhost:5173"},
        },
        Database: DatabaseConfig{
            URL:             "postgres://postgres:secret@localhost:5432/myapp_dev?sslmode=disable",
            MaxOpenConns:    5,
            MaxIdleConns:    2,
            ConnMaxLifetime: time.Minute,
        },
        Cache: CacheConfig{
            URL: "redis://localhost:6379",
            TTL: 5 * time.Minute,
        },
        Logging: LoggingConfig{
            Level:  "debug",
            Format: "text",
        },
    }
}

// Production config
func productionConfig() *Config {
    return &Config{
        App: AppConfig{
            Name:    getEnvRequired("APP_NAME"),
            Port:    8080,
            Env:     "production",
            Debug:   false,
            Origins: []string{getEnvRequired("ALLOWED_ORIGIN")},
        },
        Database: DatabaseConfig{
            URL:             getEnvRequired("DATABASE_URL"),
            MaxOpenConns:    25,
            MaxIdleConns:    10,
            ConnMaxLifetime: 5 * time.Minute,
        },
        Cache: CacheConfig{
            URL: getEnvRequired("REDIS_URL"),
            TTL: 30 * time.Minute,
        },
        Logging: LoggingConfig{
            Level:  "info",
            Format: "json",
        },
    }
}

// Staging config - มักจะคล้าย production แต่ใช้ dev credentials
func stagingConfig() *Config {
    cfg := productionConfig()
    cfg.App.Env = "staging"
    cfg.App.Debug = true
    cfg.Logging.Level = "debug"
    return cfg
}

func LoadConfig() *Config {
    env := os.Getenv("APP_ENV")
    
    switch env {
    case "production":
        return productionConfig()
    case "staging":
        return stagingConfig()
    default: // development
        return developmentConfig()
    }
}

func getEnvRequired(key string) string {
    value := os.Getenv(key)
    if value == "" {
        panic(fmt.Sprintf("Required environment variable %s is not set", key))
    }
    return value
}

func main() {
    cfg := LoadConfig()
    
    fmt.Printf("=== %s Config ===\n", cfg.App.Env)
    fmt.Printf("Port: %d\n", cfg.App.Port)
    fmt.Printf("Debug: %v\n", cfg.App.Debug)
    fmt.Printf("Log Level: %s\n", cfg.Logging.Level)
    fmt.Printf("DB Max Connections: %d\n", cfg.Database.MaxOpenConns)
    fmt.Printf("Cache TTL: %v\n", cfg.Cache.TTL)
}
```

---

## 31.7 Secrets Management

### Best Practices สำหรับ Secrets (ตัวอย่างที่ 12)

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "errors"
    "fmt"
    "io"
    "os"
)

// Secret Manager Interface
type SecretManager interface {
    GetSecret(name string) (string, error)
    SetSecret(name, value string) error
}

// EnvSecretManager - อ่าน secrets จาก environment variables
type EnvSecretManager struct{}

func (m *EnvSecretManager) GetSecret(name string) (string, error) {
    value := os.Getenv(name)
    if value == "" {
        return "", fmt.Errorf("secret %s not found", name)
    }
    return value, nil
}

func (m *EnvSecretManager) SetSecret(name, value string) error {
    return os.Setenv(name, value)
}

// EncryptedSecretManager - เข้ารหัส secrets
type EncryptedSecretManager struct {
    key []byte
    secrets map[string]string
}

func NewEncryptedSecretManager(key string) (*EncryptedSecretManager, error) {
    // key ต้องยาว 32 bytes สำหรับ AES-256
    keyBytes := []byte(key)
    if len(keyBytes) < 32 {
        return nil, errors.New("encryption key must be at least 32 bytes")
    }
    
    return &EncryptedSecretManager{
        key:     keyBytes[:32],
        secrets: make(map[string]string),
    }, nil
}

func (m *EncryptedSecretManager) encrypt(plaintext string) (string, error) {
    block, err := aes.NewCipher(m.key)
    if err != nil {
        return "", err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }
    
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", err
    }
    
    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (m *EncryptedSecretManager) decrypt(ciphertextBase64 string) (string, error) {
    ciphertext, err := base64.StdEncoding.DecodeString(ciphertextBase64)
    if err != nil {
        return "", err
    }
    
    block, err := aes.NewCipher(m.key)
    if err != nil {
        return "", err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }
    
    nonceSize := gcm.NonceSize()
    if len(ciphertext) < nonceSize {
        return "", errors.New("ciphertext too short")
    }
    
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return "", err
    }
    
    return string(plaintext), nil
}

func (m *EncryptedSecretManager) SetSecret(name, value string) error {
    encrypted, err := m.encrypt(value)
    if err != nil {
        return err
    }
    m.secrets[name] = encrypted
    return nil
}

func (m *EncryptedSecretManager) GetSecret(name string) (string, error) {
    encrypted, ok := m.secrets[name]
    if !ok {
        return "", fmt.Errorf("secret %s not found", name)
    }
    return m.decrypt(encrypted)
}

func main() {
    fmt.Println("=== Env Secret Manager ===")
    envManager := &EnvSecretManager{}
    envManager.SetSecret("DB_PASSWORD", "my-secret-password")
    
    secret, err := envManager.GetSecret("DB_PASSWORD")
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("DB_PASSWORD:", secret)
    }
    
    fmt.Println("\n=== Encrypted Secret Manager ===")
    encManager, err := NewEncryptedSecretManager("this-is-a-32-byte-encryption-key!!")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    encManager.SetSecret("api_key", "sk-super-secret-api-key-12345")
    encManager.SetSecret("db_password", "ultra-secure-password")
    
    apiKey, _ := encManager.GetSecret("api_key")
    fmt.Println("API Key:", apiKey)
    
    dbPass, _ := encManager.GetSecret("db_password")
    fmt.Println("DB Password:", dbPass)
    
    // ลอง get secret ที่ไม่มี
    _, err = encManager.GetSecret("missing_secret")
    fmt.Println("Missing secret error:", err)
}
```

### .gitignore สำหรับ Secrets (ตัวอย่างที่ 13)

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)

// ตรวจสอบว่าไฟล์ sensitive ไม่ได้อยู่ใน git
func checkSensitiveFiles() {
    sensitiveFiles := []string{
        ".env",
        ".env.local",
        ".env.production",
        "secrets.yaml",
        "config.local.yaml",
        "*.pem",
        "*.key",
        "service-account.json",
    }
    
    fmt.Println("Checking sensitive files...")
    for _, pattern := range sensitiveFiles {
        matches, _ := filepath.Glob(pattern)
        for _, match := range matches {
            if _, err := os.Stat(match); err == nil {
                fmt.Printf("WARNING: Sensitive file found: %s - Make sure it's in .gitignore!\n", match)
            }
        }
    }
    
    // ตรวจสอบ .gitignore
    gitignore, err := os.ReadFile(".gitignore")
    if err != nil {
        fmt.Println("WARNING: .gitignore not found!")
        return
    }
    
    requiredEntries := []string{".env", ".env.local", "*.key", "*.pem"}
    gitignoreContent := string(gitignore)
    
    for _, entry := range requiredEntries {
        found := false
        for _, line := range []string{gitignoreContent} {
            if len(line) > 0 {
                found = true
                break
            }
        }
        if !found {
            fmt.Printf("WARNING: %s should be in .gitignore\n", entry)
        }
    }
    
    fmt.Println("Security check complete!")
    _ = gitignoreContent
}

func main() {
    checkSensitiveFiles()
    
    // แสดง config สำหรับ debugging (ไม่ให้แสดง secrets)
    fmt.Println("\n=== Application Config ===")
    fmt.Printf("APP_NAME: %s\n", os.Getenv("APP_NAME"))
    fmt.Printf("APP_ENV: %s\n", os.Getenv("APP_ENV"))
    fmt.Printf("APP_PORT: %s\n", os.Getenv("APP_PORT"))
    fmt.Printf("DB_HOST: %s\n", os.Getenv("DB_HOST"))
    fmt.Printf("DB_NAME: %s\n", os.Getenv("DB_NAME"))
    
    // ไม่แสดง secrets!
    if os.Getenv("DB_PASSWORD") != "" {
        fmt.Println("DB_PASSWORD: [SET]")
    } else {
        fmt.Println("DB_PASSWORD: [NOT SET]")
    }
    
    if os.Getenv("JWT_SECRET") != "" {
        fmt.Println("JWT_SECRET: [SET]")
    } else {
        fmt.Println("JWT_SECRET: [NOT SET]")
    }
}
```

---

## Workshop: Complete Config System

```go
package main

import (
    "fmt"
    "log"
    "os"
    "time"
    
    "github.com/joho/godotenv"
    "github.com/go-playground/validator/v10"
)

type Config struct {
    App      AppConfig      `validate:"required"`
    Database DatabaseConfig `validate:"required"`
    Auth     AuthConfig     `validate:"required"`
    Cache    CacheConfig
    Features FeatureFlags
}

type AppConfig struct {
    Name    string        `validate:"required,min=1"`
    Port    int           `validate:"required,min=1,max=65535"`
    Env     string        `validate:"required,oneof=development staging production"`
    Debug   bool
    Timeout time.Duration
}

type DatabaseConfig struct {
    URL          string `validate:"required,url"`
    MaxOpenConns int    `validate:"min=1,max=100"`
    MaxIdleConns int    `validate:"min=1"`
    Migrate      bool
}

type AuthConfig struct {
    JWTSecret        string        `validate:"required,min=32"`
    JWTExpiry        time.Duration `validate:"required"`
    RefreshExpiry    time.Duration `validate:"required"`
    BcryptCost       int           `validate:"min=10,max=14"`
}

type CacheConfig struct {
    Enabled bool
    URL     string
    TTL     time.Duration
}

type FeatureFlags struct {
    NewUserFlow   bool
    BetaFeatures  bool
    MaintenanceMode bool
}

type ConfigLoader struct {
    validate *validator.Validate
}

func NewConfigLoader() *ConfigLoader {
    return &ConfigLoader{
        validate: validator.New(),
    }
}

func (l *ConfigLoader) Load() (*Config, error) {
    // Load .env file (dev only)
    env := os.Getenv("APP_ENV")
    if env == "" || env == "development" {
        if err := godotenv.Load(); err != nil {
            log.Println("No .env file found")
        }
    }
    
    cfg := &Config{}
    
    // App
    cfg.App = AppConfig{
        Name:    getStr("APP_NAME", "MyApp"),
        Port:    getInt("APP_PORT", 8080),
        Env:     getStr("APP_ENV", "development"),
        Debug:   getBool("APP_DEBUG", env != "production"),
        Timeout: getDuration("APP_TIMEOUT", 30*time.Second),
    }
    
    // Database
    cfg.Database = DatabaseConfig{
        URL:          getStr("DATABASE_URL", "postgres://postgres:secret@localhost:5432/myapp?sslmode=disable"),
        MaxOpenConns: getInt("DB_MAX_OPEN_CONNS", 25),
        MaxIdleConns: getInt("DB_MAX_IDLE_CONNS", 10),
        Migrate:      getBool("DB_MIGRATE", true),
    }
    
    // Auth
    cfg.Auth = AuthConfig{
        JWTSecret:     getStr("JWT_SECRET", ""),
        JWTExpiry:     getDuration("JWT_EXPIRY", 24*time.Hour),
        RefreshExpiry: getDuration("REFRESH_EXPIRY", 7*24*time.Hour),
        BcryptCost:    getInt("BCRYPT_COST", 12),
    }
    
    // Cache
    cfg.Cache = CacheConfig{
        Enabled: getBool("CACHE_ENABLED", false),
        URL:     getStr("REDIS_URL", ""),
        TTL:     getDuration("CACHE_TTL", 15*time.Minute),
    }
    
    // Feature Flags
    cfg.Features = FeatureFlags{
        NewUserFlow:     getBool("FEATURE_NEW_USER_FLOW", false),
        BetaFeatures:    getBool("FEATURE_BETA", false),
        MaintenanceMode: getBool("MAINTENANCE_MODE", false),
    }
    
    // Validate
    if err := l.validate.Struct(cfg); err != nil {
        return nil, fmt.Errorf("config validation: %w", err)
    }
    
    return cfg, nil
}

func (c *Config) IsDevelopment() bool { return c.App.Env == "development" }
func (c *Config) IsProduction() bool  { return c.App.Env == "production" }
func (c *Config) IsStaging() bool     { return c.App.Env == "staging" }

func (c *Config) Print() {
    fmt.Println("=== Configuration ===")
    fmt.Printf("App: %s (v1.0) running on :%d [%s]\n", c.App.Name, c.App.Port, c.App.Env)
    fmt.Printf("Debug: %v, Timeout: %v\n", c.App.Debug, c.App.Timeout)
    fmt.Printf("DB Max Connections: %d open, %d idle\n", c.Database.MaxOpenConns, c.Database.MaxIdleConns)
    fmt.Printf("JWT Expiry: %v, Refresh: %v\n", c.Auth.JWTExpiry, c.Auth.RefreshExpiry)
    fmt.Printf("Cache: %v (TTL: %v)\n", c.Cache.Enabled, c.Cache.TTL)
    fmt.Printf("Features: newUserFlow=%v, beta=%v, maintenance=%v\n",
        c.Features.NewUserFlow, c.Features.BetaFeatures, c.Features.MaintenanceMode)
}

// Helper functions
func getStr(key, def string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return def
}

func getInt(key string, def int) int {
    if v := os.Getenv(key); v != "" {
        var i int
        fmt.Sscanf(v, "%d", &i)
        if i > 0 {
            return i
        }
    }
    return def
}

func getBool(key string, def bool) bool {
    v := os.Getenv(key)
    if v == "true" || v == "1" || v == "yes" {
        return true
    }
    if v == "false" || v == "0" || v == "no" {
        return false
    }
    return def
}

func getDuration(key string, def time.Duration) time.Duration {
    if v := os.Getenv(key); v != "" {
        if d, err := time.ParseDuration(v); err == nil {
            return d
        }
    }
    return def
}

func main() {
    // ตั้งค่า env สำหรับทดสอบ
    os.Setenv("APP_NAME", "BlogAPI")
    os.Setenv("JWT_SECRET", "this-is-a-very-secure-jwt-secret-key-32-bytes!!")
    os.Setenv("APP_ENV", "development")
    
    loader := NewConfigLoader()
    cfg, err := loader.Load()
    if err != nil {
        log.Fatal("Failed to load config:", err)
    }
    
    cfg.Print()
    
    fmt.Printf("\nIs Development: %v\n", cfg.IsDevelopment())
    fmt.Printf("Is Production: %v\n", cfg.IsProduction())
    
    if cfg.Features.MaintenanceMode {
        fmt.Println("System is in maintenance mode!")
    }
}
```

---

## สรุป Part 31

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| os.Getenv | อ่าน env var, returns "" ถ้าไม่มี |
| os.LookupEnv | อ่าน env var พร้อมตรวจสอบว่าถูก set ไหม |
| godotenv | โหลด .env file เข้า os.Environ |
| Viper | อ่าน config จาก files, env vars, flags |
| Config Struct | รวม config ทั้งหมดใน struct เดียว |
| Validation | ใช้ validator tags เพื่อ validate config |
| Multiple Envs | Development/Staging/Production configs |
| Secrets | ไม่ commit secrets, ใช้ env vars |

### Resources
- [godotenv](https://github.com/joho/godotenv)
- [Viper](https://github.com/spf13/viper)
- [The 12-Factor App - Config](https://12factor.net/config)
- [Go validator](https://github.com/go-playground/validator)
