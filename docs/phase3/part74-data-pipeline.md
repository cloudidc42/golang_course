# Part 74: Data Pipeline

## เป้าหมายการเรียนรู้
- Data Pipeline Architecture
- ETL Processes ใน Go
- Apache Kafka Streams
- Batch Processing
- Stream Processing
- Data Transformation
- Error Handling ใน Pipelines

---

## 1. Data Pipeline Architecture

```
Data Sources → Ingestion → Processing → Storage → Consumption
    │              │            │          │           │
  APIs         Kafka       Transform    Data        BI Tools
  Files        Kinesis     Validate    Lake        APIs
  DBs          Pub/Sub     Enrich      DW          ML Models
```

```go
// pipeline/types.go
package pipeline

import (
    "context"
    "time"
)

// Pipeline Stage interface
type Stage interface {
    Name() string
    Process(ctx context.Context, input <-chan Record) (<-chan Record, error)
}

// Record คือหน่วยข้อมูลใน pipeline
type Record struct {
    ID        string
    Source    string
    Timestamp time.Time
    Data      map[string]interface{}
    Metadata  map[string]string
    Error     error
}

// Pipeline รวม stages ต่อกัน
type Pipeline struct {
    stages  []Stage
    metrics *PipelineMetrics
}

type PipelineMetrics struct {
    ProcessedRecords int64
    FailedRecords    int64
    Latency          time.Duration
}

func NewPipeline(stages ...Stage) *Pipeline {
    return &Pipeline{
        stages:  stages,
        metrics: &PipelineMetrics{},
    }
}

func (p *Pipeline) Run(ctx context.Context, input <-chan Record) (<-chan Record, error) {
    current := input
    
    for _, stage := range p.stages {
        output, err := stage.Process(ctx, current)
        if err != nil {
            return nil, fmt.Errorf("stage %s failed: %w", stage.Name(), err)
        }
        current = output
    }
    
    return current, nil
}
```

---

## 2. ETL Pipeline

```go
// etl/pipeline.go
package etl

import (
    "context"
    "database/sql"
    "encoding/csv"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "os"
    "sync"
    "time"
)

// Extract Phase
type CSVExtractor struct {
    filename string
    batchSize int
}

func NewCSVExtractor(filename string, batchSize int) *CSVExtractor {
    return &CSVExtractor{filename: filename, batchSize: batchSize}
}

func (e *CSVExtractor) Name() string { return "csv-extractor" }

func (e *CSVExtractor) Process(ctx context.Context, _ <-chan Record) (<-chan Record, error) {
    f, err := os.Open(e.filename)
    if err != nil {
        return nil, fmt.Errorf("failed to open file: %w", err)
    }
    
    output := make(chan Record, e.batchSize)
    
    go func() {
        defer close(output)
        defer f.Close()
        
        reader := csv.NewReader(f)
        
        // อ่าน header
        headers, err := reader.Read()
        if err != nil {
            log.Printf("Failed to read CSV header: %v", err)
            return
        }
        
        rowNum := 0
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            row, err := reader.Read()
            if err == io.EOF {
                break
            }
            if err != nil {
                log.Printf("Error reading row %d: %v", rowNum, err)
                output <- Record{Error: err}
                continue
            }
            
            data := make(map[string]interface{})
            for i, header := range headers {
                if i < len(row) {
                    data[header] = row[i]
                }
            }
            
            output <- Record{
                ID:        fmt.Sprintf("row-%d", rowNum),
                Source:    e.filename,
                Timestamp: time.Now(),
                Data:      data,
            }
            rowNum++
        }
        
        log.Printf("Extracted %d records from %s", rowNum, e.filename)
    }()
    
    return output, nil
}

// Database Extractor
type DatabaseExtractor struct {
    db        *sql.DB
    query     string
    batchSize int
}

func (e *DatabaseExtractor) Name() string { return "db-extractor" }

func (e *DatabaseExtractor) Process(ctx context.Context, _ <-chan Record) (<-chan Record, error) {
    output := make(chan Record, e.batchSize)
    
    go func() {
        defer close(output)
        
        rows, err := e.db.QueryContext(ctx, e.query)
        if err != nil {
            output <- Record{Error: err}
            return
        }
        defer rows.Close()
        
        columns, err := rows.Columns()
        if err != nil {
            output <- Record{Error: err}
            return
        }
        
        rowNum := 0
        for rows.Next() {
            values := make([]interface{}, len(columns))
            valuePtrs := make([]interface{}, len(columns))
            for i := range values {
                valuePtrs[i] = &values[i]
            }
            
            if err := rows.Scan(valuePtrs...); err != nil {
                output <- Record{Error: err}
                continue
            }
            
            data := make(map[string]interface{})
            for i, col := range columns {
                data[col] = values[i]
            }
            
            output <- Record{
                ID:        fmt.Sprintf("db-row-%d", rowNum),
                Source:    "database",
                Timestamp: time.Now(),
                Data:      data,
            }
            rowNum++
        }
    }()
    
    return output, nil
}

// Transform Phase
type TransformStage struct {
    name        string
    transforms  []TransformFunc
    concurrency int
}

type TransformFunc func(record *Record) error

func NewTransformStage(name string, concurrency int, transforms ...TransformFunc) *TransformStage {
    return &TransformStage{
        name:        name,
        transforms:  transforms,
        concurrency: concurrency,
    }
}

func (t *TransformStage) Name() string { return t.name }

func (t *TransformStage) Process(ctx context.Context, input <-chan Record) (<-chan Record, error) {
    output := make(chan Record, t.concurrency*2)
    
    var wg sync.WaitGroup
    
    for i := 0; i < t.concurrency; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            for record := range input {
                if ctx.Err() != nil {
                    return
                }
                
                r := record // copy
                
                if r.Error != nil {
                    output <- r
                    continue
                }
                
                for _, transform := range t.transforms {
                    if err := transform(&r); err != nil {
                        r.Error = err
                        break
                    }
                }
                
                output <- r
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(output)
    }()
    
    return output, nil
}

// Common transforms
func NormalizeEmailTransform(record *Record) error {
    if email, ok := record.Data["email"].(string); ok {
        record.Data["email"] = strings.ToLower(strings.TrimSpace(email))
    }
    return nil
}

func ValidateRequiredFieldsTransform(fields []string) TransformFunc {
    return func(record *Record) error {
        for _, field := range fields {
            val, ok := record.Data[field]
            if !ok || val == nil || val == "" {
                return fmt.Errorf("required field '%s' is missing or empty", field)
            }
        }
        return nil
    }
}

func ParseDateTransform(field, format string) TransformFunc {
    return func(record *Record) error {
        val, ok := record.Data[field].(string)
        if !ok {
            return nil
        }
        
        parsed, err := time.Parse(format, val)
        if err != nil {
            return fmt.Errorf("invalid date format for field '%s': %v", field, err)
        }
        
        record.Data[field] = parsed
        return nil
    }
}

func EnrichWithLookupTransform(field string, lookup map[string]interface{}) TransformFunc {
    return func(record *Record) error {
        val, ok := record.Data[field].(string)
        if !ok {
            return nil
        }
        
        if enrichment, ok := lookup[val]; ok {
            record.Data[field+"_enriched"] = enrichment
        }
        
        return nil
    }
}

// Load Phase
type DatabaseLoader struct {
    db        *sql.DB
    tableName string
    batchSize int
}

func (l *DatabaseLoader) Name() string { return "db-loader" }

func (l *DatabaseLoader) Process(ctx context.Context, input <-chan Record) (<-chan Record, error) {
    output := make(chan Record, l.batchSize)
    
    go func() {
        defer close(output)
        
        var batch []Record
        
        flush := func() error {
            if len(batch) == 0 {
                return nil
            }
            
            if err := l.insertBatch(ctx, batch); err != nil {
                for i := range batch {
                    batch[i].Error = err
                }
            }
            
            for _, r := range batch {
                output <- r
            }
            batch = batch[:0]
            
            return nil
        }
        
        for record := range input {
            if record.Error != nil {
                output <- record
                continue
            }
            
            batch = append(batch, record)
            
            if len(batch) >= l.batchSize {
                flush()
            }
        }
        
        flush() // Flush remaining
    }()
    
    return output, nil
}

func (l *DatabaseLoader) insertBatch(ctx context.Context, records []Record) error {
    tx, err := l.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    stmt, err := tx.PrepareContext(ctx, fmt.Sprintf(
        "INSERT INTO %s (data, source, created_at) VALUES ($1, $2, $3) ON CONFLICT DO NOTHING",
        l.tableName,
    ))
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    for _, r := range records {
        data, _ := json.Marshal(r.Data)
        if _, err := stmt.ExecContext(ctx, data, r.Source, r.Timestamp); err != nil {
            return err
        }
    }
    
    return tx.Commit()
}

// Complete ETL Pipeline
func RunUserDataETL(ctx context.Context, db *sql.DB, sourceFile string) error {
    extractor := NewCSVExtractor(sourceFile, 100)
    
    transformer := NewTransformStage("user-transformer", 4,
        ValidateRequiredFieldsTransform([]string{"email", "name"}),
        NormalizeEmailTransform,
        ParseDateTransform("created_at", "2006-01-02"),
    )
    
    loader := &DatabaseLoader{
        db:        db,
        tableName: "users",
        batchSize: 100,
    }
    
    pipeline := NewPipeline(extractor, transformer, loader)
    
    // Pipeline source เป็น nil เพราะ Extractor ไม่ต้องการ input
    output, err := pipeline.Run(ctx, nil)
    if err != nil {
        return err
    }
    
    // Process results
    var (
        processed int
        failed    int
    )
    
    for record := range output {
        if record.Error != nil {
            log.Printf("Failed record %s: %v", record.ID, record.Error)
            failed++
        } else {
            processed++
        }
    }
    
    log.Printf("ETL complete: processed=%d, failed=%d", processed, failed)
    return nil
}
```

---

## 3. Kafka Stream Processing

```go
// kafka/stream.go
package kafka

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
    
    "github.com/IBM/sarama"
)

type KafkaConsumer struct {
    client  sarama.ConsumerGroup
    topics  []string
    handler ConsumerGroupHandler
}

type ConsumerGroupHandler struct {
    ready   chan bool
    process func(message *sarama.ConsumerMessage) error
}

func (h ConsumerGroupHandler) Setup(session sarama.ConsumerGroupSession) error {
    close(h.ready)
    return nil
}

func (h ConsumerGroupHandler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h ConsumerGroupHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for {
        select {
        case message, ok := <-claim.Messages():
            if !ok {
                return nil
            }
            
            start := time.Now()
            
            if err := h.process(message); err != nil {
                log.Printf("Message processing error (offset %d): %v", message.Offset, err)
                // ไม่ commit offset เมื่อ error (จะ retry)
                continue
            }
            
            session.MarkMessage(message, "")
            
            log.Printf("Processed message offset=%d latency=%v", 
                message.Offset, time.Since(start))
            
        case <-session.Context().Done():
            return nil
        }
    }
}

func NewKafkaConsumer(brokers []string, groupID string, topics []string, processor func(*sarama.ConsumerMessage) error) (*KafkaConsumer, error) {
    config := sarama.NewConfig()
    config.Version = sarama.V3_0_0_0
    config.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
        sarama.NewBalanceStrategyRoundRobin(),
    }
    config.Consumer.Offsets.Initial = sarama.OffsetOldest
    config.Consumer.Return.Errors = true
    
    client, err := sarama.NewConsumerGroup(brokers, groupID, config)
    if err != nil {
        return nil, fmt.Errorf("failed to create consumer group: %w", err)
    }
    
    return &KafkaConsumer{
        client:  client,
        topics:  topics,
        handler: ConsumerGroupHandler{
            ready:   make(chan bool),
            process: processor,
        },
    }, nil
}

func (kc *KafkaConsumer) Start(ctx context.Context) error {
    go func() {
        for err := range kc.client.Errors() {
            log.Printf("Kafka error: %v", err)
        }
    }()
    
    for {
        if err := kc.client.Consume(ctx, kc.topics, kc.handler); err != nil {
            if err == sarama.ErrClosedConsumerGroup {
                return nil
            }
            return fmt.Errorf("consumer error: %w", err)
        }
        
        if ctx.Err() != nil {
            return nil
        }
        
        // Reset ready channel สำหรับ rebalancing
        kc.handler.ready = make(chan bool)
    }
}

// Kafka Producer
type KafkaProducer struct {
    producer sarama.SyncProducer
    topic    string
}

func NewKafkaProducer(brokers []string, topic string) (*KafkaProducer, error) {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll
    config.Producer.Retry.Max = 5
    config.Producer.Return.Successes = true
    config.Producer.Compression = sarama.CompressionSnappy
    
    producer, err := sarama.NewSyncProducer(brokers, config)
    if err != nil {
        return nil, fmt.Errorf("failed to create producer: %w", err)
    }
    
    return &KafkaProducer{producer: producer, topic: topic}, nil
}

func (p *KafkaProducer) Publish(key string, value interface{}) (int32, int64, error) {
    data, err := json.Marshal(value)
    if err != nil {
        return 0, 0, err
    }
    
    msg := &sarama.ProducerMessage{
        Topic: p.topic,
        Key:   sarama.StringEncoder(key),
        Value: sarama.ByteEncoder(data),
        Headers: []sarama.RecordHeader{
            {
                Key:   []byte("timestamp"),
                Value: []byte(fmt.Sprintf("%d", time.Now().UnixNano())),
            },
        },
    }
    
    return p.producer.SendMessage(msg)
}

// Stream Processing: Count events per window
type WindowedCounter struct {
    counts    map[string]int
    windowSize time.Duration
    mu         sync.Mutex
}

func NewWindowedCounter(windowSize time.Duration) *WindowedCounter {
    wc := &WindowedCounter{
        counts:     make(map[string]int),
        windowSize: windowSize,
    }
    
    go wc.flush()
    return wc
}

func (wc *WindowedCounter) Increment(key string) {
    wc.mu.Lock()
    defer wc.mu.Unlock()
    wc.counts[key]++
}

func (wc *WindowedCounter) flush() {
    ticker := time.NewTicker(wc.windowSize)
    for range ticker.C {
        wc.mu.Lock()
        snapshot := wc.counts
        wc.counts = make(map[string]int)
        wc.mu.Unlock()
        
        for key, count := range snapshot {
            log.Printf("Window count - %s: %d", key, count)
        }
    }
}
```

---

## 4. Batch Processing

```go
// batch/processor.go
package batch

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

type BatchJob struct {
    ID          string
    Name        string
    Schedule    string // cron expression
    BatchSize   int
    Concurrency int
    Processor   BatchProcessor
}

type BatchProcessor interface {
    FetchBatch(ctx context.Context, cursor string, limit int) ([]Record, string, error)
    ProcessRecord(ctx context.Context, record Record) error
    OnComplete(ctx context.Context, stats BatchStats) error
}

type BatchStats struct {
    JobID     string
    StartTime time.Time
    EndTime   time.Time
    Total     int
    Success   int
    Failed    int
    Duration  time.Duration
}

type BatchRunner struct {
    job    *BatchJob
    db     *sql.DB
    redis  *redis.Client
}

func (r *BatchRunner) Run(ctx context.Context) error {
    stats := BatchStats{
        JobID:     r.job.ID,
        StartTime: time.Now(),
    }
    
    log.Printf("Starting batch job: %s", r.job.Name)
    
    // ดึง cursor สุดท้าย (สำหรับ resume)
    cursor := r.getLastCursor(ctx)
    
    // Semaphore สำหรับ concurrency control
    sem := make(chan struct{}, r.job.Concurrency)
    var wg sync.WaitGroup
    var mu sync.Mutex
    
    for {
        if ctx.Err() != nil {
            break
        }
        
        // Fetch batch
        records, nextCursor, err := r.job.Processor.FetchBatch(ctx, cursor, r.job.BatchSize)
        if err != nil {
            log.Printf("Failed to fetch batch: %v", err)
            break
        }
        
        if len(records) == 0 {
            log.Println("No more records to process")
            break
        }
        
        // Process concurrently
        for _, record := range records {
            wg.Add(1)
            sem <- struct{}{} // Acquire
            
            go func(r Record) {
                defer func() {
                    <-sem // Release
                    wg.Done()
                }()
                
                if err := r.job.Processor.ProcessRecord(ctx, r); err != nil {
                    log.Printf("Record %s failed: %v", r.ID, err)
                    mu.Lock()
                    stats.Failed++
                    mu.Unlock()
                } else {
                    mu.Lock()
                    stats.Success++
                    mu.Unlock()
                }
                
                mu.Lock()
                stats.Total++
                mu.Unlock()
            }(record)
        }
        
        wg.Wait()
        
        // Save checkpoint
        cursor = nextCursor
        r.saveCheckpoint(ctx, cursor)
        
        log.Printf("Batch progress: processed=%d, failed=%d", stats.Success, stats.Failed)
        
        if nextCursor == "" {
            break // ไม่มี records เพิ่มเติม
        }
    }
    
    stats.EndTime = time.Now()
    stats.Duration = stats.EndTime.Sub(stats.StartTime)
    
    r.job.Processor.OnComplete(ctx, stats)
    
    log.Printf("Batch job complete: %+v", stats)
    return nil
}

func (r *BatchRunner) getLastCursor(ctx context.Context) string {
    cursor, err := r.redis.Get(ctx, "batch:"+r.job.ID+":cursor").Result()
    if err != nil {
        return ""
    }
    return cursor
}

func (r *BatchRunner) saveCheckpoint(ctx context.Context, cursor string) {
    r.redis.Set(ctx, "batch:"+r.job.ID+":cursor", cursor, 24*time.Hour)
}

// ตัวอย่าง: Batch process invoices
type InvoiceProcessor struct {
    db          *sql.DB
    emailSender EmailSender
}

func (p *InvoiceProcessor) FetchBatch(ctx context.Context, cursor string, limit int) ([]Record, string, error) {
    query := `
        SELECT id, user_id, amount, status
        FROM invoices
        WHERE status = 'pending'
        AND id > $1
        ORDER BY id
        LIMIT $2
    `
    
    rows, err := p.db.QueryContext(ctx, query, cursor, limit)
    if err != nil {
        return nil, "", err
    }
    defer rows.Close()
    
    var records []Record
    var lastID string
    
    for rows.Next() {
        var (
            id, userID, status string
            amount             float64
        )
        rows.Scan(&id, &userID, &amount, &status)
        
        records = append(records, Record{
            ID: id,
            Data: map[string]interface{}{
                "user_id": userID,
                "amount":  amount,
                "status":  status,
            },
        })
        lastID = id
    }
    
    return records, lastID, rows.Err()
}

func (p *InvoiceProcessor) ProcessRecord(ctx context.Context, record Record) error {
    userID := record.Data["user_id"].(string)
    amount := record.Data["amount"].(float64)
    
    // ส่ง invoice email
    if err := p.emailSender.SendInvoice(ctx, userID, amount); err != nil {
        return fmt.Errorf("failed to send invoice: %w", err)
    }
    
    // Update status
    _, err := p.db.ExecContext(ctx,
        "UPDATE invoices SET status = 'sent', sent_at = $1 WHERE id = $2",
        time.Now(), record.ID,
    )
    return err
}

func (p *InvoiceProcessor) OnComplete(ctx context.Context, stats BatchStats) error {
    log.Printf("Invoice batch complete: sent=%d, failed=%d, duration=%v",
        stats.Success, stats.Failed, stats.Duration)
    return nil
}
```

---

## 5. Stream Processing

```go
// stream/processor.go
package stream

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Pipeline operators
type Stream[T any] struct {
    ch <-chan T
}

func FromChannel[T any](ch <-chan T) *Stream[T] {
    return &Stream[T]{ch: ch}
}

func (s *Stream[T]) Filter(predicate func(T) bool) *Stream[T] {
    output := make(chan T)
    
    go func() {
        defer close(output)
        for item := range s.ch {
            if predicate(item) {
                output <- item
            }
        }
    }()
    
    return &Stream[T]{ch: output}
}

func (s *Stream[T]) Map(transform func(T) T) *Stream[T] {
    output := make(chan T)
    
    go func() {
        defer close(output)
        for item := range s.ch {
            output <- transform(item)
        }
    }()
    
    return &Stream[T]{ch: output}
}

func MapTo[T, U any](s *Stream[T], transform func(T) U) *Stream[U] {
    output := make(chan U)
    
    go func() {
        defer close(output)
        for item := range s.ch {
            output <- transform(item)
        }
    }()
    
    return &Stream[U]{ch: output}
}

func (s *Stream[T]) FlatMap(transform func(T) []T) *Stream[T] {
    output := make(chan T)
    
    go func() {
        defer close(output)
        for item := range s.ch {
            for _, v := range transform(item) {
                output <- v
            }
        }
    }()
    
    return &Stream[T]{ch: output}
}

func (s *Stream[T]) Batch(size int) *Stream[[]T] {
    output := make(chan []T)
    
    go func() {
        defer close(output)
        
        var batch []T
        for item := range s.ch {
            batch = append(batch, item)
            if len(batch) >= size {
                output <- batch
                batch = nil
            }
        }
        if len(batch) > 0 {
            output <- batch
        }
    }()
    
    return &Stream[[]T]{ch: output}
}

func (s *Stream[T]) Window(duration time.Duration) *Stream[[]T] {
    output := make(chan []T)
    
    go func() {
        defer close(output)
        
        var (
            window []T
            mu     sync.Mutex
        )
        
        ticker := time.NewTicker(duration)
        defer ticker.Stop()
        
        go func() {
            for item := range s.ch {
                mu.Lock()
                window = append(window, item)
                mu.Unlock()
            }
        }()
        
        for range ticker.C {
            mu.Lock()
            if len(window) > 0 {
                batch := make([]T, len(window))
                copy(batch, window)
                window = nil
                mu.Unlock()
                output <- batch
            } else {
                mu.Unlock()
            }
        }
    }()
    
    return &Stream[[]T]{ch: output}
}

func (s *Stream[T]) Parallel(concurrency int, fn func(T)) {
    var wg sync.WaitGroup
    sem := make(chan struct{}, concurrency)
    
    for item := range s.ch {
        wg.Add(1)
        sem <- struct{}{}
        
        go func(v T) {
            defer func() {
                <-sem
                wg.Done()
            }()
            fn(v)
        }(item)
    }
    
    wg.Wait()
}

func (s *Stream[T]) Sink(fn func(T)) {
    for item := range s.ch {
        fn(item)
    }
}

// ตัวอย่างการใช้งาน
type Event struct {
    Type      string
    UserID    string
    Amount    float64
    Timestamp time.Time
}

func processEventStream(ctx context.Context, events <-chan Event) {
    stream := FromChannel(events)
    
    // Filter, transform, and process
    stream.
        Filter(func(e Event) bool {
            return e.Type == "purchase" && e.Amount > 0
        }).
        Window(time.Minute).
        Sink(func(window []Event) {
            // Aggregate events ใน window
            total := 0.0
            for _, e := range window {
                total += e.Amount
            }
            
            log.Printf("Window aggregate: count=%d, total=%.2f", len(window), total)
        })
}
```

---

## 6. Error Handling ใน Pipelines

```go
// pipeline/errors.go
package pipeline

import (
    "context"
    "fmt"
    "log"
    "time"
)

// Dead Letter Queue สำหรับ failed records
type DeadLetterQueue struct {
    records chan FailedRecord
    storage DLQStorage
}

type FailedRecord struct {
    Record    Record
    Error     string
    Attempts  int
    FailedAt  time.Time
    Source    string
}

func NewDLQ(storage DLQStorage) *DeadLetterQueue {
    dlq := &DeadLetterQueue{
        records: make(chan FailedRecord, 1000),
        storage: storage,
    }
    
    go dlq.process()
    return dlq
}

func (dlq *DeadLetterQueue) Send(record Record, err error, source string) {
    dlq.records <- FailedRecord{
        Record:   record,
        Error:    err.Error(),
        FailedAt: time.Now(),
        Source:   source,
    }
}

func (dlq *DeadLetterQueue) process() {
    for failed := range dlq.records {
        if err := dlq.storage.Save(failed); err != nil {
            log.Printf("Failed to save to DLQ: %v", err)
        }
    }
}

// Retry Policy
type RetryPolicy struct {
    MaxAttempts int
    InitialWait time.Duration
    MaxWait     time.Duration
    Multiplier  float64
}

func (p *RetryPolicy) Execute(ctx context.Context, fn func() error) error {
    wait := p.InitialWait
    
    for attempt := 1; attempt <= p.MaxAttempts; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        if attempt == p.MaxAttempts {
            return fmt.Errorf("max attempts reached: %w", err)
        }
        
        log.Printf("Attempt %d failed: %v, waiting %v", attempt, err, wait)
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(wait):
        }
        
        wait = time.Duration(float64(wait) * p.Multiplier)
        if wait > p.MaxWait {
            wait = p.MaxWait
        }
    }
    
    return fmt.Errorf("all attempts failed")
}

// Error Stage สำหรับ pipeline
type ErrorHandlingStage struct {
    inner     Stage
    dlq       *DeadLetterQueue
    retryPol  *RetryPolicy
}

func (s *ErrorHandlingStage) Name() string { return "error-handler" }

func (s *ErrorHandlingStage) Process(ctx context.Context, input <-chan Record) (<-chan Record, error) {
    innerOutput, err := s.inner.Process(ctx, input)
    if err != nil {
        return nil, err
    }
    
    output := make(chan Record, 100)
    
    go func() {
        defer close(output)
        
        for record := range innerOutput {
            if record.Error != nil {
                // ลอง retry
                var lastErr error
                retryErr := s.retryPol.Execute(ctx, func() error {
                    // Re-process record
                    return nil
                })
                
                if retryErr != nil {
                    // ส่งไป DLQ
                    s.dlq.Send(record, lastErr, s.inner.Name())
                    continue
                }
            }
            
            output <- record
        }
    }()
    
    return output, nil
}
```

---

## Workshop: Analytics Pipeline

```go
// workshop/analytics/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
)

type PageView struct {
    UserID    string    `json:"user_id"`
    Page      string    `json:"page"`
    Duration  int       `json:"duration_seconds"`
    Referrer  string    `json:"referrer"`
    UserAgent string    `json:"user_agent"`
    Timestamp time.Time `json:"timestamp"`
}

type UserAnalytics struct {
    UserID      string  `json:"user_id"`
    TotalViews  int     `json:"total_views"`
    TotalTime   int     `json:"total_time_seconds"`
    TopPages    []string `json:"top_pages"`
    LastSeen    time.Time `json:"last_seen"`
}

// Analytics pipeline
type AnalyticsPipeline struct {
    kafkaConsumer *KafkaConsumer
    db            *sql.DB
    redis         *redis.Client
}

func (p *AnalyticsPipeline) Start(ctx context.Context) error {
    return p.kafkaConsumer.Start(ctx)
}

func processPageView(db *sql.DB, redis *redis.Client) func(msg *sarama.ConsumerMessage) error {
    return func(msg *sarama.ConsumerMessage) error {
        var pv PageView
        if err := json.Unmarshal(msg.Value, &pv); err != nil {
            return fmt.Errorf("invalid message: %w", err)
        }
        
        // Update real-time counters ใน Redis
        ctx := context.Background()
        key := fmt.Sprintf("analytics:user:%s:views", pv.UserID)
        redis.Incr(ctx, key)
        redis.Expire(ctx, key, 24*time.Hour)
        
        // Aggregate ใน DB (batch)
        _, err := db.ExecContext(ctx, `
            INSERT INTO page_views (user_id, page, duration, timestamp)
            VALUES ($1, $2, $3, $4)
        `, pv.UserID, pv.Page, pv.Duration, pv.Timestamp)
        
        return err
    }
}

// Aggregation job ที่ run ทุกชั่วโมง
func runAggregationJob(ctx context.Context, db *sql.DB) error {
    processor := &BatchRunner{
        job: &BatchJob{
            ID:          "analytics-aggregation",
            Name:        "Analytics Aggregation",
            BatchSize:   1000,
            Concurrency: 4,
        },
    }
    
    return processor.Run(ctx)
}

func main() {
    db, _ := sql.Open("postgres", "postgres://user:pass@localhost/analytics")
    
    consumer, err := NewKafkaConsumer(
        []string{"localhost:9092"},
        "analytics-group",
        []string{"page-views"},
        processPageView(db, nil),
    )
    if err != nil {
        log.Fatal(err)
    }
    
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Start consumer
    go func() {
        if err := consumer.Start(ctx); err != nil {
            log.Printf("Consumer error: %v", err)
        }
    }()
    
    // Run aggregation job ทุก 1 ชั่วโมง
    ticker := time.NewTicker(time.Hour)
    for {
        select {
        case <-ticker.C:
            runAggregationJob(ctx, db)
        case <-ctx.Done():
            return
        }
    }
}
```

---

## สรุป

| Pattern | Use Case | Throughput | Latency |
|---------|---------|------------|---------|
| ETL Batch | Bulk data migration | High | High |
| Stream Processing | Real-time analytics | Medium | Low |
| Micro-batching | Near real-time | High | Medium |
| Lambda Architecture | Both batch & stream | Highest | Mixed |

### Best Practices
1. ออกแบบ pipeline ให้ idempotent (run ซ้ำได้ผลเหมือนกัน)
2. ใช้ checkpointing สำหรับ fault tolerance
3. Monitor lag ใน Kafka consumer groups
4. ใช้ Dead Letter Queue สำหรับ failed records
5. Test กับ production-size data
6. ใช้ schema registry สำหรับ data contracts

---

*จบ Part 74: Data Pipeline*
