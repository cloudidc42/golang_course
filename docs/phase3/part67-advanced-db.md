# Part 67: Advanced Database Patterns

## เป้าหมายการเรียนรู้
- เข้าใจ Connection Pooling ขั้นสูง
- เรียนรู้ Database Sharding และ Read Replicas
- ใช้งาน CQRS, Multi-tenancy
- จัดการ Database Migrations ด้วย golang-migrate
- Query Optimization เทคนิคต่างๆ

---

## 1. Connection Pooling ขั้นสูง

### ปัญหาของ Connection ที่ไม่ได้ optimize

```go
// ❌ แบบผิด - สร้าง connection ใหม่ทุกครั้ง
package main

import (
    "database/sql"
    "log"
    _ "github.com/lib/pq"
)

func badExample() {
    // สร้าง connection ใหม่ทุก request - ช้ามาก!
    db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // ทำงาน
    db.QueryRow("SELECT 1")
}
```

```go
// ✅ แบบถูก - ใช้ Connection Pool
package database

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"
    
    _ "github.com/lib/pq"
)

type DBConfig struct {
    Host            string
    Port            int
    User            string
    Password        string
    DBName          string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

func NewDB(cfg DBConfig) (*sql.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=disable",
        cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName,
    )
    
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }
    
    // Connection Pool Settings
    db.SetMaxOpenConns(cfg.MaxOpenConns)       // สูงสุด N connections พร้อมกัน
    db.SetMaxIdleConns(cfg.MaxIdleConns)       // เก็บ idle connections ไว้ N ตัว
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime) // connection ใช้ได้นาน
    db.SetConnMaxIdleTime(cfg.ConnMaxIdleTime) // idle connection expire

    // ทดสอบ connection
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }
    
    log.Printf("Database connected (pool: max=%d, idle=%d)", 
        cfg.MaxOpenConns, cfg.MaxIdleConns)
    return db, nil
}

// ตัวอย่างการ config สำหรับ production
func DefaultConfig() DBConfig {
    return DBConfig{
        MaxOpenConns:    25,               // จำนวน CPUs * 4
        MaxIdleConns:    10,               // MaxOpenConns / 2
        ConnMaxLifetime: 5 * time.Minute,  // ต่ำกว่า DB timeout (8h default postgres)
        ConnMaxIdleTime: 2 * time.Minute,  // release idle connections
    }
}
```

### Connection Pool Monitoring

```go
// pool_monitor.go
package database

import (
    "context"
    "database/sql"
    "log"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    dbOpenConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "db_open_connections",
        Help: "Number of open database connections",
    })
    dbIdleConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "db_idle_connections",
        Help: "Number of idle database connections",
    })
    dbWaitCount = promauto.NewCounter(prometheus.CounterOpts{
        Name: "db_wait_count_total",
        Help: "Total number of connections waited for",
    })
    dbWaitDuration = promauto.NewHistogram(prometheus.HistogramOpts{
        Name:    "db_wait_duration_seconds",
        Help:    "Time waited for a connection",
        Buckets: prometheus.DefBuckets,
    })
)

func MonitorPool(ctx context.Context, db *sql.DB, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    
    var lastWaitCount int64
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            stats := db.Stats()
            
            dbOpenConnections.Set(float64(stats.OpenConnections))
            dbIdleConnections.Set(float64(stats.Idle))
            
            waitDiff := stats.WaitCount - lastWaitCount
            if waitDiff > 0 {
                dbWaitCount.Add(float64(waitDiff))
                log.Printf("Pool warning: %d new connection waits", waitDiff)
            }
            lastWaitCount = stats.WaitCount
            
            if stats.WaitDuration > 0 {
                dbWaitDuration.Observe(stats.WaitDuration.Seconds())
            }
            
            log.Printf("DB Pool - Open: %d, Idle: %d, InUse: %d, WaitTotal: %d",
                stats.OpenConnections,
                stats.Idle,
                stats.InUse,
                stats.WaitCount,
            )
        }
    }
}
```

### pgx Connection Pool (ประสิทธิภาพสูงกว่า)

```go
// pgx_pool.go
package database

import (
    "context"
    "fmt"
    
    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPgxPool(connString string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(connString)
    if err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    
    // ปรับแต่ง pool
    config.MaxConns = 20
    config.MinConns = 5
    config.MaxConnLifetime = 5 * 60 * 1000000000  // 5 minutes in nanoseconds
    config.MaxConnIdleTime = 2 * 60 * 1000000000  // 2 minutes
    
    pool, err := pgxpool.NewWithConfig(context.Background(), config)
    if err != nil {
        return nil, fmt.Errorf("failed to create pool: %w", err)
    }
    
    return pool, nil
}

// Batch operations กับ pgx
func BatchInsert(ctx context.Context, pool *pgxpool.Pool, users []User) error {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return err
    }
    defer conn.Release()
    
    batch := &pgx.Batch{}
    for _, user := range users {
        batch.Queue(
            "INSERT INTO users (name, email, created_at) VALUES ($1, $2, $3)",
            user.Name, user.Email, user.CreatedAt,
        )
    }
    
    results := conn.SendBatch(ctx, batch)
    defer results.Close()
    
    for range users {
        if _, err := results.Exec(); err != nil {
            return fmt.Errorf("batch insert failed: %w", err)
        }
    }
    
    return nil
}
```

---

## 2. Database Sharding

Sharding คือการแบ่ง data ออกเป็น partitions บน servers หลายตัว

```
┌─────────────────────────────────────────────┐
│              Application                     │
│                   │                          │
│              Shard Router                    │
│            /      |      \                   │
│       Shard1   Shard2   Shard3              │
│    (users A-I)(users J-R)(users S-Z)        │
└─────────────────────────────────────────────┘
```

### Hash-based Sharding

```go
// sharding/shard_manager.go
package sharding

import (
    "context"
    "crypto/md5"
    "database/sql"
    "fmt"
    "hash/fnv"
    "log"
    "sync"
)

type Shard struct {
    ID int
    DB *sql.DB
}

type ShardManager struct {
    shards []*Shard
    mu     sync.RWMutex
}

func NewShardManager(configs []DBConfig) (*ShardManager, error) {
    sm := &ShardManager{
        shards: make([]*Shard, len(configs)),
    }
    
    for i, cfg := range configs {
        db, err := NewDB(cfg)
        if err != nil {
            return nil, fmt.Errorf("failed to connect shard %d: %w", i, err)
        }
        sm.shards[i] = &Shard{ID: i, DB: db}
        log.Printf("Shard %d connected: %s", i, cfg.Host)
    }
    
    return sm, nil
}

// Hash-based shard selection
func (sm *ShardManager) GetShardByID(id int64) *Shard {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    
    shardIndex := int(id) % len(sm.shards)
    return sm.shards[shardIndex]
}

// String-based shard selection (เช่น user ID)
func (sm *ShardManager) GetShardByString(key string) *Shard {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    
    h := fnv.New32a()
    h.Write([]byte(key))
    shardIndex := int(h.Sum32()) % len(sm.shards)
    return sm.shards[shardIndex]
}

// Consistent hashing สำหรับ shard rebalancing
type ConsistentHashRouter struct {
    ring     map[uint32]*Shard
    sorted   []uint32
    replicas int
    mu       sync.RWMutex
}

func NewConsistentHashRouter(replicas int) *ConsistentHashRouter {
    return &ConsistentHashRouter{
        ring:     make(map[uint32]*Shard),
        replicas: replicas,
    }
}

func (chr *ConsistentHashRouter) AddShard(shard *Shard) {
    chr.mu.Lock()
    defer chr.mu.Unlock()
    
    for i := 0; i < chr.replicas; i++ {
        key := chr.hashKey(fmt.Sprintf("shard-%d-%d", shard.ID, i))
        chr.ring[key] = shard
        chr.sorted = append(chr.sorted, key)
    }
    
    // Sort ring
    sort.Slice(chr.sorted, func(i, j int) bool {
        return chr.sorted[i] < chr.sorted[j]
    })
}

func (chr *ConsistentHashRouter) GetShard(key string) *Shard {
    chr.mu.RLock()
    defer chr.mu.RUnlock()
    
    if len(chr.ring) == 0 {
        return nil
    }
    
    hash := chr.hashKey(key)
    
    // Binary search หา shard ที่ใกล้ที่สุด
    idx := sort.Search(len(chr.sorted), func(i int) bool {
        return chr.sorted[i] >= hash
    })
    
    if idx == len(chr.sorted) {
        idx = 0
    }
    
    return chr.ring[chr.sorted[idx]]
}

func (chr *ConsistentHashRouter) hashKey(key string) uint32 {
    h := md5.New()
    h.Write([]byte(key))
    b := h.Sum(nil)
    return uint32(b[3])<<24 | uint32(b[2])<<16 | uint32(b[1])<<8 | uint32(b[0])
}

// User Repository กับ Sharding
type UserRepository struct {
    shardManager *ShardManager
}

func (r *UserRepository) CreateUser(ctx context.Context, user *User) error {
    shard := r.shardManager.GetShardByString(user.Email)
    
    _, err := shard.DB.ExecContext(ctx,
        "INSERT INTO users (id, name, email, created_at) VALUES ($1, $2, $3, $4)",
        user.ID, user.Name, user.Email, user.CreatedAt,
    )
    return err
}

func (r *UserRepository) GetUser(ctx context.Context, userID int64) (*User, error) {
    shard := r.shardManager.GetShardByID(userID)
    
    var user User
    err := shard.DB.QueryRowContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1",
        userID,
    ).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    
    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }
    return &user, err
}

// Cross-shard query (ต้อง query ทุก shard แล้ว merge)
func (r *UserRepository) FindUsersByName(ctx context.Context, name string) ([]*User, error) {
    var (
        mu      sync.Mutex
        wg      sync.WaitGroup
        allUsers []*User
        errors  []error
    )
    
    for _, shard := range r.shardManager.shards {
        wg.Add(1)
        go func(s *Shard) {
            defer wg.Done()
            
            rows, err := s.DB.QueryContext(ctx,
                "SELECT id, name, email, created_at FROM users WHERE name ILIKE $1",
                "%"+name+"%",
            )
            if err != nil {
                mu.Lock()
                errors = append(errors, err)
                mu.Unlock()
                return
            }
            defer rows.Close()
            
            var shardUsers []*User
            for rows.Next() {
                var u User
                rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
                shardUsers = append(shardUsers, &u)
            }
            
            mu.Lock()
            allUsers = append(allUsers, shardUsers...)
            mu.Unlock()
        }(shard)
    }
    
    wg.Wait()
    
    if len(errors) > 0 {
        return allUsers, fmt.Errorf("some shards failed: %v", errors)
    }
    
    return allUsers, nil
}
```

---

## 3. Read Replicas

```go
// read_replicas.go
package database

import (
    "context"
    "database/sql"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

type ReplicaSet struct {
    primary  *sql.DB
    replicas []*sql.DB
    mu       sync.RWMutex
    rr       int // round-robin index
}

func NewReplicaSet(primaryDSN string, replicaDSNs []string) (*ReplicaSet, error) {
    primary, err := sql.Open("postgres", primaryDSN)
    if err != nil {
        return nil, fmt.Errorf("failed to connect primary: %w", err)
    }
    configurePool(primary)
    
    replicas := make([]*sql.DB, 0, len(replicaDSNs))
    for _, dsn := range replicaDSNs {
        db, err := sql.Open("postgres", dsn)
        if err != nil {
            return nil, fmt.Errorf("failed to connect replica: %w", err)
        }
        configurePool(db)
        replicas = append(replicas, db)
    }
    
    return &ReplicaSet{
        primary:  primary,
        replicas: replicas,
    }, nil
}

func configurePool(db *sql.DB) {
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)
}

// ส่ง write operations ไป primary
func (rs *ReplicaSet) Primary() *sql.DB {
    return rs.primary
}

// ส่ง read operations ไป replica (round-robin)
func (rs *ReplicaSet) Replica() *sql.DB {
    rs.mu.Lock()
    defer rs.mu.Unlock()
    
    if len(rs.replicas) == 0 {
        return rs.primary  // fallback ไป primary
    }
    
    replica := rs.replicas[rs.rr%len(rs.replicas)]
    rs.rr++
    return replica
}

// Random replica selection
func (rs *ReplicaSet) RandomReplica() *sql.DB {
    rs.mu.RLock()
    defer rs.mu.RUnlock()
    
    if len(rs.replicas) == 0 {
        return rs.primary
    }
    
    return rs.replicas[rand.Intn(len(rs.replicas))]
}

// Repository pattern กับ Read Replicas
type ProductRepository struct {
    db *ReplicaSet
}

func (r *ProductRepository) Create(ctx context.Context, p *Product) error {
    // Write ไป primary
    _, err := r.db.Primary().ExecContext(ctx,
        "INSERT INTO products (id, name, price, stock) VALUES ($1, $2, $3, $4)",
        p.ID, p.Name, p.Price, p.Stock,
    )
    return err
}

func (r *ProductRepository) GetByID(ctx context.Context, id string) (*Product, error) {
    // Read จาก replica
    var p Product
    err := r.db.Replica().QueryRowContext(ctx,
        "SELECT id, name, price, stock FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
    
    return &p, err
}

func (r *ProductRepository) List(ctx context.Context, limit, offset int) ([]*Product, error) {
    // Read จาก replica
    rows, err := r.db.Replica().QueryContext(ctx,
        "SELECT id, name, price, stock FROM products ORDER BY created_at DESC LIMIT $1 OFFSET $2",
        limit, offset,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var products []*Product
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
        products = append(products, &p)
    }
    
    return products, rows.Err()
}
```

---

## 4. Database Partitioning

```sql
-- PostgreSQL Range Partitioning
-- แบ่ง orders table ตามปี
CREATE TABLE orders (
    id          BIGSERIAL,
    user_id     BIGINT NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- สร้าง partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Index บน partition key
CREATE INDEX ON orders_2024 (user_id);
CREATE INDEX ON orders_2024 (created_at);
```

```go
// partition_manager.go
package database

import (
    "context"
    "database/sql"
    "fmt"
    "time"
)

type PartitionManager struct {
    db *sql.DB
}

// สร้าง partition อัตโนมัติสำหรับเดือนถัดไป
func (pm *PartitionManager) EnsureNextMonthPartition(ctx context.Context) error {
    nextMonth := time.Now().AddDate(0, 1, 0)
    partitionName := fmt.Sprintf("orders_%s", nextMonth.Format("200601"))
    
    startDate := fmt.Sprintf("%d-%02d-01", nextMonth.Year(), nextMonth.Month())
    endDate := fmt.Sprintf("%d-%02d-01", nextMonth.AddDate(0, 1, 0).Year(), 
        nextMonth.AddDate(0, 1, 0).Month())
    
    query := fmt.Sprintf(`
        CREATE TABLE IF NOT EXISTS %s 
        PARTITION OF orders
        FOR VALUES FROM ('%s') TO ('%s')
    `, partitionName, startDate, endDate)
    
    _, err := pm.db.ExecContext(ctx, query)
    if err != nil {
        return fmt.Errorf("failed to create partition %s: %w", partitionName, err)
    }
    
    // สร้าง index บน partition ใหม่
    indexQuery := fmt.Sprintf(
        "CREATE INDEX IF NOT EXISTS idx_%s_user_id ON %s (user_id)",
        partitionName, partitionName,
    )
    _, err = pm.db.ExecContext(ctx, indexQuery)
    return err
}

// Drop partition เก่าที่ไม่ต้องการแล้ว
func (pm *PartitionManager) DropOldPartitions(ctx context.Context, keepMonths int) error {
    cutoff := time.Now().AddDate(0, -keepMonths, 0)
    partitionName := fmt.Sprintf("orders_%s", cutoff.Format("200601"))
    
    // Detach ก่อน drop (ป้องกัน lock ทั้ง table)
    detachQuery := fmt.Sprintf(
        "ALTER TABLE orders DETACH PARTITION %s", partitionName,
    )
    if _, err := pm.db.ExecContext(ctx, detachQuery); err != nil {
        return fmt.Errorf("failed to detach partition: %w", err)
    }
    
    // Archive ข้อมูลก่อน drop (ถ้าต้องการ)
    dropQuery := fmt.Sprintf("DROP TABLE IF EXISTS %s", partitionName)
    _, err := pm.db.ExecContext(ctx, dropQuery)
    return err
}
```

---

## 5. CQRS กับ Separate Databases

CQRS = Command Query Responsibility Segregation

```go
// cqrs/command.go - Write side
package cqrs

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "time"
)

// Commands (Write)
type CreateOrderCommand struct {
    UserID    string
    Items     []OrderItem
    PaymentID string
}

type CommandHandler interface {
    Handle(ctx context.Context, cmd interface{}) error
}

type OrderCommandHandler struct {
    writeDB   *sql.DB
    eventBus  EventBus
}

func (h *OrderCommandHandler) Handle(ctx context.Context, cmd interface{}) error {
    switch c := cmd.(type) {
    case CreateOrderCommand:
        return h.handleCreateOrder(ctx, c)
    default:
        return fmt.Errorf("unknown command type: %T", cmd)
    }
}

func (h *OrderCommandHandler) handleCreateOrder(ctx context.Context, cmd CreateOrderCommand) error {
    tx, err := h.writeDB.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    orderID := generateID()
    total := calculateTotal(cmd.Items)
    
    // Write ไป write database
    _, err = tx.ExecContext(ctx, `
        INSERT INTO orders (id, user_id, total, status, created_at)
        VALUES ($1, $2, $3, 'pending', $4)
    `, orderID, cmd.UserID, total, time.Now())
    if err != nil {
        return err
    }
    
    // Save event
    event := OrderCreatedEvent{
        OrderID: orderID,
        UserID:  cmd.UserID,
        Items:   cmd.Items,
        Total:   total,
    }
    
    eventData, _ := json.Marshal(event)
    _, err = tx.ExecContext(ctx, `
        INSERT INTO events (aggregate_id, event_type, payload, occurred_at)
        VALUES ($1, 'OrderCreated', $2, $3)
    `, orderID, eventData, time.Now())
    if err != nil {
        return err
    }
    
    if err := tx.Commit(); err != nil {
        return err
    }
    
    // Publish event ไป read side
    h.eventBus.Publish(event)
    
    return nil
}
```

```go
// cqrs/query.go - Read side
package cqrs

import (
    "context"
    "github.com/elastic/go-elasticsearch/v8"
)

// Queries (Read)
type GetOrderQuery struct {
    OrderID string
}

type ListOrdersQuery struct {
    UserID string
    Limit  int
    Offset int
}

// Read model - optimized for queries
type OrderReadModel struct {
    ID         string  `json:"id"`
    UserID     string  `json:"user_id"`
    UserName   string  `json:"user_name"`  // denormalized
    UserEmail  string  `json:"user_email"` // denormalized
    Items      []Item  `json:"items"`
    Total      float64 `json:"total"`
    Status     string  `json:"status"`
    CreatedAt  string  `json:"created_at"`
}

type OrderQueryHandler struct {
    esClient *elasticsearch.Client
    readDB   *sql.DB // Read-optimized database
}

// Query ไปยัง Elasticsearch (read-optimized store)
func (h *OrderQueryHandler) GetOrder(ctx context.Context, query GetOrderQuery) (*OrderReadModel, error) {
    res, err := h.esClient.Get("orders", query.OrderID,
        h.esClient.Get.WithContext(ctx),
    )
    if err != nil {
        return nil, err
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return nil, fmt.Errorf("ES error: %s", res.Status())
    }
    
    var result struct {
        Source OrderReadModel `json:"_source"`
    }
    json.NewDecoder(res.Body).Decode(&result)
    
    return &result.Source, nil
}
```

```go
// cqrs/projector.go - Event handler สำหรับ update read models
package cqrs

type OrderProjector struct {
    esClient *elasticsearch.Client
    readDB   *sql.DB
}

func (p *OrderProjector) HandleOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    // Build denormalized read model
    user, _ := p.getUserInfo(ctx, event.UserID)
    
    readModel := OrderReadModel{
        ID:        event.OrderID,
        UserID:    event.UserID,
        UserName:  user.Name,
        UserEmail: user.Email,
        Items:     event.Items,
        Total:     event.Total,
        Status:    "pending",
        CreatedAt: event.OccurredAt.Format(time.RFC3339),
    }
    
    // Save ไปยัง read store (Elasticsearch)
    data, _ := json.Marshal(readModel)
    _, err := p.esClient.Index("orders",
        bytes.NewReader(data),
        p.esClient.Index.WithDocumentID(event.OrderID),
        p.esClient.Index.WithContext(ctx),
    )
    
    return err
}
```

---

## 6. Multi-tenancy

```go
// multitenancy/tenant.go
package multitenancy

import (
    "context"
    "database/sql"
    "fmt"
)

type TenantID string

type contextKey string

const tenantKey contextKey = "tenant_id"

// ใส่ tenant ID ใน context
func WithTenant(ctx context.Context, tenantID TenantID) context.Context {
    return context.WithValue(ctx, tenantKey, tenantID)
}

// ดึง tenant ID จาก context
func GetTenant(ctx context.Context) (TenantID, error) {
    tenantID, ok := ctx.Value(tenantKey).(TenantID)
    if !ok || tenantID == "" {
        return "", fmt.Errorf("tenant ID not found in context")
    }
    return tenantID, nil
}

// Strategy 1: Schema-based multi-tenancy
type SchemaMultiTenancy struct {
    db *sql.DB
}

func (m *SchemaMultiTenancy) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
    tenantID, err := GetTenant(ctx)
    if err != nil {
        return nil, err
    }
    
    // Set search_path ก่อน query
    conn, err := m.db.Conn(ctx)
    if err != nil {
        return nil, err
    }
    defer conn.Close()
    
    _, err = conn.ExecContext(ctx, fmt.Sprintf("SET search_path = '%s'", string(tenantID)))
    if err != nil {
        return nil, fmt.Errorf("failed to set schema: %w", err)
    }
    
    return conn.ExecContext(ctx, query, args...)
}

// Strategy 2: Row-level multi-tenancy (tenant_id column)
type RowMultiTenancy struct {
    db *sql.DB
}

type TenantRepository[T any] struct {
    db        *RowMultiTenancy
    tableName string
}

func (r *TenantRepository[T]) FindAll(ctx context.Context) ([]*T, error) {
    tenantID, err := GetTenant(ctx)
    if err != nil {
        return nil, err
    }
    
    // Automatically add tenant_id filter
    query := fmt.Sprintf(
        "SELECT * FROM %s WHERE tenant_id = $1 AND deleted_at IS NULL",
        r.tableName,
    )
    
    rows, err := r.db.db.QueryContext(ctx, query, string(tenantID))
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // Scan rows...
    return nil, nil
}

// Strategy 3: Database-per-tenant
type DatabasePerTenantManager struct {
    connections map[TenantID]*sql.DB
    configFn    func(TenantID) string // function to get DSN for tenant
}

func (m *DatabasePerTenantManager) GetDB(ctx context.Context) (*sql.DB, error) {
    tenantID, err := GetTenant(ctx)
    if err != nil {
        return nil, err
    }
    
    if db, ok := m.connections[tenantID]; ok {
        return db, nil
    }
    
    // Lazy connect
    dsn := m.configFn(tenantID)
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("failed to connect tenant DB: %w", err)
    }
    
    m.connections[tenantID] = db
    return db, nil
}

// HTTP Middleware สำหรับ extract tenant ID
func TenantMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ดึง tenant จาก subdomain: tenant1.app.com
        host := r.Host
        parts := strings.Split(host, ".")
        if len(parts) < 3 {
            http.Error(w, "Invalid tenant", http.StatusBadRequest)
            return
        }
        
        tenantID := TenantID(parts[0])
        ctx := WithTenant(r.Context(), tenantID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 7. Database Migrations กับ golang-migrate

```go
// migrations/migrator.go
package migrations

import (
    "database/sql"
    "embed"
    "fmt"
    "log"
    
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed sql/*.sql
var sqlFiles embed.FS

type Migrator struct {
    db  *sql.DB
    dsn string
}

func NewMigrator(db *sql.DB, dsn string) *Migrator {
    return &Migrator{db: db, dsn: dsn}
}

func (m *Migrator) Up() error {
    driver, err := postgres.WithInstance(m.db, &postgres.Config{})
    if err != nil {
        return fmt.Errorf("failed to create migration driver: %w", err)
    }
    
    sourceDriver, err := iofs.New(sqlFiles, "sql")
    if err != nil {
        return fmt.Errorf("failed to create source driver: %w", err)
    }
    
    migrator, err := migrate.NewWithInstance("iofs", sourceDriver, "postgres", driver)
    if err != nil {
        return fmt.Errorf("failed to create migrator: %w", err)
    }
    
    if err := migrator.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("migration failed: %w", err)
    }
    
    version, dirty, _ := migrator.Version()
    log.Printf("Migration complete - version: %d, dirty: %v", version, dirty)
    
    return nil
}

func (m *Migrator) Down(steps int) error {
    driver, err := postgres.WithInstance(m.db, &postgres.Config{})
    if err != nil {
        return err
    }
    
    sourceDriver, err := iofs.New(sqlFiles, "sql")
    if err != nil {
        return err
    }
    
    migrator, err := migrate.NewWithInstance("iofs", sourceDriver, "postgres", driver)
    if err != nil {
        return err
    }
    
    return migrator.Steps(-steps)
}
```

```sql
-- migrations/sql/000001_create_users.up.sql
CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    tenant_id   VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at  TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_created_at ON users(created_at);

-- migrations/sql/000001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

```sql
-- migrations/sql/000002_create_orders.up.sql
CREATE TABLE IF NOT EXISTS orders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id),
    status      VARCHAR(50) NOT NULL DEFAULT 'pending',
    total       DECIMAL(12,2) NOT NULL,
    tenant_id   VARCHAR(100) NOT NULL,
    metadata    JSONB,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE INDEX idx_orders_user_id ON orders(user_id, created_at);
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id, created_at);
CREATE INDEX idx_orders_status ON orders(status) WHERE status != 'completed';
```

---

## 8. Query Optimization

```go
// query_optimizer.go
package database

import (
    "context"
    "database/sql"
    "fmt"
    "strings"
    "time"
)

// N+1 Problem - แบบผิด
func badGetUsersWithOrders(ctx context.Context, db *sql.DB) ([]UserWithOrders, error) {
    // Query 1: ดึง users
    rows, _ := db.QueryContext(ctx, "SELECT id, name FROM users LIMIT 100")
    
    var users []UserWithOrders
    for rows.Next() {
        var u UserWithOrders
        rows.Scan(&u.ID, &u.Name)
        
        // N+1: Query เพิ่มอีก 1 ครั้งต่อ user!
        orderRows, _ := db.QueryContext(ctx,
            "SELECT id, total FROM orders WHERE user_id = $1", u.ID,
        )
        for orderRows.Next() {
            var o Order
            orderRows.Scan(&o.ID, &o.Total)
            u.Orders = append(u.Orders, o)
        }
        
        users = append(users, u)
    }
    return users, nil
}

// ✅ แก้ N+1 ด้วย JOIN
func goodGetUsersWithOrders(ctx context.Context, db *sql.DB) ([]UserWithOrders, error) {
    rows, err := db.QueryContext(ctx, `
        SELECT 
            u.id, 
            u.name,
            o.id AS order_id,
            o.total
        FROM users u
        LEFT JOIN orders o ON o.user_id = u.id
        WHERE u.deleted_at IS NULL
        ORDER BY u.id, o.created_at DESC
        LIMIT 1000
    `)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    userMap := make(map[string]*UserWithOrders)
    var userOrder []string
    
    for rows.Next() {
        var (
            userID    string
            userName  string
            orderID   sql.NullString
            orderTotal sql.NullFloat64
        )
        rows.Scan(&userID, &userName, &orderID, &orderTotal)
        
        if _, ok := userMap[userID]; !ok {
            userMap[userID] = &UserWithOrders{ID: userID, Name: userName}
            userOrder = append(userOrder, userID)
        }
        
        if orderID.Valid {
            userMap[userID].Orders = append(userMap[userID].Orders, Order{
                ID:    orderID.String,
                Total: orderTotal.Float64,
            })
        }
    }
    
    result := make([]UserWithOrders, 0, len(userOrder))
    for _, id := range userOrder {
        result = append(result, *userMap[id])
    }
    
    return result, nil
}

// Cursor-based Pagination (ดีกว่า OFFSET สำหรับ large datasets)
type CursorPagination struct {
    Limit  int
    Cursor string // ID ของ record สุดท้ายที่ได้รับ
}

func getUsersWithCursor(ctx context.Context, db *sql.DB, p CursorPagination) ([]User, string, error) {
    query := `
        SELECT id, name, email, created_at
        FROM users
        WHERE deleted_at IS NULL
        AND ($1 = '' OR id > $1)
        ORDER BY id ASC
        LIMIT $2
    `
    
    rows, err := db.QueryContext(ctx, query, p.Cursor, p.Limit+1)
    if err != nil {
        return nil, "", err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var u User
        rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
        users = append(users, u)
    }
    
    var nextCursor string
    if len(users) > p.Limit {
        nextCursor = users[p.Limit].ID
        users = users[:p.Limit]
    }
    
    return users, nextCursor, nil
}

// Query Builder สำหรับ Dynamic Queries
type QueryBuilder struct {
    table      string
    conditions []string
    args       []interface{}
    orderBy    string
    limit      int
    offset     int
}

func NewQueryBuilder(table string) *QueryBuilder {
    return &QueryBuilder{table: table}
}

func (qb *QueryBuilder) Where(condition string, args ...interface{}) *QueryBuilder {
    qb.conditions = append(qb.conditions, condition)
    qb.args = append(qb.args, args...)
    return qb
}

func (qb *QueryBuilder) OrderBy(column, direction string) *QueryBuilder {
    qb.orderBy = fmt.Sprintf("%s %s", column, direction)
    return qb
}

func (qb *QueryBuilder) Limit(n int) *QueryBuilder {
    qb.limit = n
    return qb
}

func (qb *QueryBuilder) Build() (string, []interface{}) {
    query := fmt.Sprintf("SELECT * FROM %s", qb.table)
    
    if len(qb.conditions) > 0 {
        // แก้ parameter index
        for i, cond := range qb.conditions {
            cond = strings.Replace(cond, "?", fmt.Sprintf("$%d", i+1), 1)
            qb.conditions[i] = cond
        }
        query += " WHERE " + strings.Join(qb.conditions, " AND ")
    }
    
    if qb.orderBy != "" {
        query += " ORDER BY " + qb.orderBy
    }
    
    if qb.limit > 0 {
        query += fmt.Sprintf(" LIMIT %d", qb.limit)
    }
    
    return query, qb.args
}

// การใช้งาน Query Builder
func searchUsers(ctx context.Context, db *sql.DB, filters UserFilters) ([]User, error) {
    qb := NewQueryBuilder("users")
    
    if filters.Name != "" {
        qb.Where("name ILIKE ?", "%"+filters.Name+"%")
    }
    if filters.Email != "" {
        qb.Where("email = ?", filters.Email)
    }
    if !filters.CreatedAfter.IsZero() {
        qb.Where("created_at > ?", filters.CreatedAfter)
    }
    
    qb.OrderBy("created_at", "DESC").Limit(50)
    
    query, args := qb.Build()
    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var u User
        // scan...
        users = append(users, u)
    }
    return users, nil
}
```

### Explain Analyze

```go
// explain.go
package database

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "strings"
    "time"
)

// Auto-explain slow queries
func WrapWithExplain(db *sql.DB, threshold time.Duration) *ExplainDB {
    return &ExplainDB{db: db, threshold: threshold}
}

type ExplainDB struct {
    db        *sql.DB
    threshold time.Duration
}

func (e *ExplainDB) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
    start := time.Now()
    rows, err := e.db.QueryContext(ctx, query, args...)
    elapsed := time.Since(start)
    
    if elapsed > e.threshold {
        log.Printf("SLOW QUERY (%v): %s", elapsed, query)
        go e.explainQuery(context.Background(), query, args...)
    }
    
    return rows, err
}

func (e *ExplainDB) explainQuery(ctx context.Context, query string, args ...interface{}) {
    // ไม่ run EXPLAIN บน DML
    lq := strings.ToLower(strings.TrimSpace(query))
    if !strings.HasPrefix(lq, "select") {
        return
    }
    
    explainQuery := "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " + query
    rows, err := e.db.QueryContext(ctx, explainQuery, args...)
    if err != nil {
        log.Printf("EXPLAIN failed: %v", err)
        return
    }
    defer rows.Close()
    
    if rows.Next() {
        var plan string
        rows.Scan(&plan)
        log.Printf("QUERY PLAN:\n%s", plan)
    }
}
```

---

## Workshop: Multi-tenant SaaS Database Layer

```go
// workshop/saas_db.go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "os"
    "strings"
    "time"
    
    _ "github.com/lib/pq"
)

type TenantDB struct {
    writeDB  *sql.DB
    readDB   *sql.DB
    tenantID string
}

type MultiTenantPool struct {
    tenants map[string]*TenantDB
    primary *sql.DB // shared primary สำหรับ tenant management
}

func NewMultiTenantPool(adminDSN string) (*MultiTenantPool, error) {
    primary, err := sql.Open("postgres", adminDSN)
    if err != nil {
        return nil, err
    }
    
    return &MultiTenantPool{
        tenants: make(map[string]*TenantDB),
        primary: primary,
    }, nil
}

func (p *MultiTenantPool) GetTenantDB(tenantID string) (*TenantDB, error) {
    if db, ok := p.tenants[tenantID]; ok {
        return db, nil
    }
    
    // Load tenant config from admin DB
    var writeDSN, readDSN string
    err := p.primary.QueryRow(
        "SELECT write_dsn, read_dsn FROM tenants WHERE id = $1 AND active = true",
        tenantID,
    ).Scan(&writeDSN, &readDSN)
    if err != nil {
        return nil, fmt.Errorf("tenant not found: %s", tenantID)
    }
    
    writeDB, err := sql.Open("postgres", writeDSN)
    if err != nil {
        return nil, err
    }
    writeDB.SetMaxOpenConns(10)
    
    readDB, err := sql.Open("postgres", readDSN)
    if err != nil {
        return nil, err
    }
    readDB.SetMaxOpenConns(20)
    
    tdb := &TenantDB{
        writeDB:  writeDB,
        readDB:   readDB,
        tenantID: tenantID,
    }
    
    p.tenants[tenantID] = tdb
    return tdb, nil
}

// Middleware
func TenantDBMiddleware(pool *MultiTenantPool) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract tenant from subdomain
            host := r.Host
            parts := strings.Split(host, ".")
            if len(parts) < 2 {
                http.Error(w, "Invalid request", http.StatusBadRequest)
                return
            }
            
            tenantID := parts[0]
            tdb, err := pool.GetTenantDB(tenantID)
            if err != nil {
                log.Printf("Tenant not found: %s", tenantID)
                http.Error(w, "Tenant not found", http.StatusNotFound)
                return
            }
            
            ctx := context.WithValue(r.Context(), "tenant_db", tdb)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func GetTenantDBFromContext(ctx context.Context) *TenantDB {
    return ctx.Value("tenant_db").(*TenantDB)
}

func main() {
    pool, err := NewMultiTenantPool(os.Getenv("ADMIN_DB_URL"))
    if err != nil {
        log.Fatal(err)
    }
    
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
        tdb := GetTenantDBFromContext(r.Context())
        
        rows, err := tdb.readDB.QueryContext(r.Context(),
            "SELECT id, name FROM users ORDER BY created_at DESC LIMIT 50",
        )
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        defer rows.Close()
        
        // process rows...
        w.WriteHeader(http.StatusOK)
    })
    
    handler := TenantDBMiddleware(pool)(mux)
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

---

## สรุป

| Pattern | Use Case | Tradeoff |
|---------|----------|----------|
| Connection Pool | ทุก application | ต้อง tune ให้เหมาะสม |
| Sharding | Data > 1TB, high write | Complex cross-shard queries |
| Read Replicas | Read-heavy workloads | Replication lag |
| Partitioning | Time-series data | Partition pruning |
| CQRS | Complex domains | Eventual consistency |
| Multi-tenancy | SaaS applications | Isolation vs efficiency |

### Best Practices
1. เริ่มต้นด้วย single database แล้วค่อย scale
2. Monitor connection pool metrics
3. ใช้ migrations สำหรับทุก schema change
4. Test ด้วย production-like data volume
5. Index ทุก column ที่ใช้ใน WHERE, JOIN, ORDER BY

---

*จบ Part 67: Advanced Database Patterns*
