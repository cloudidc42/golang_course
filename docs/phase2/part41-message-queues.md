# Part 41: Message Queues ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ Message Queue concepts
- ใช้ RabbitMQ กับ amqp091-go
- ใช้ Kafka กับ confluent-kafka-go
- ใช้ NATS messaging
- ใช้ Message patterns: pub/sub, request/reply, work queues
- จัดการ Dead Letter Queues
- จัดการ Message Acknowledgment

---

## 1. Message Queue Concepts

Message Queue เป็น middleware ที่ช่วยให้ services สื่อสารกันแบบ asynchronous

```
Producer → [Message Queue] → Consumer
```

**ประโยชน์:**
- Decoupling: producer และ consumer ทำงานอิสระ
- Load balancing: กระจายงานไปหลาย consumers
- Buffering: รองรับ traffic spikes
- Reliability: ข้อความไม่สูญหายแม้ consumer down

```go
package main

import (
	"fmt"
	"time"
)

// ตัวอย่าง 1: Simple in-memory message queue
type Message struct {
	ID        string
	Topic     string
	Body      []byte
	Headers   map[string]string
	Timestamp time.Time
	Retries   int
}

type SimpleQueue struct {
	messages chan Message
	done     chan struct{}
}

func NewSimpleQueue(bufferSize int) *SimpleQueue {
	return &SimpleQueue{
		messages: make(chan Message, bufferSize),
		done:     make(chan struct{}),
	}
}

func (q *SimpleQueue) Publish(msg Message) error {
	select {
	case q.messages <- msg:
		return nil
	case <-time.After(5 * time.Second):
		return fmt.Errorf("queue is full, publish timed out")
	}
}

func (q *SimpleQueue) Subscribe(handler func(Message) error) {
	go func() {
		for {
			select {
			case msg := <-q.messages:
				if err := handler(msg); err != nil {
					fmt.Printf("Handler error: %v\n", err)
					// Retry logic
					if msg.Retries < 3 {
						msg.Retries++
						q.messages <- msg
					} else {
						fmt.Printf("Message %s discarded after %d retries\n", msg.ID, msg.Retries)
					}
				}
			case <-q.done:
				return
			}
		}
	}()
}

func (q *SimpleQueue) Close() {
	close(q.done)
}

func simpleQueueDemo() {
	q := NewSimpleQueue(100)
	defer q.Close()
	
	// Subscribe
	q.Subscribe(func(msg Message) error {
		fmt.Printf("Processing: [%s] %s\n", msg.Topic, string(msg.Body))
		return nil
	})
	
	// Publish messages
	topics := []string{"orders", "payments", "notifications"}
	for i, topic := range topics {
		q.Publish(Message{
			ID:        fmt.Sprintf("msg-%d", i+1),
			Topic:     topic,
			Body:      []byte(fmt.Sprintf("Message %d content", i+1)),
			Headers:   map[string]string{"content-type": "application/json"},
			Timestamp: time.Now(),
		})
	}
	
	time.Sleep(100 * time.Millisecond)
}

func main() {
	simpleQueueDemo()
}
```

---

## 2. RabbitMQ with amqp091-go

```bash
# Install
go get github.com/rabbitmq/amqp091-go
```

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"
	
	amqp "github.com/rabbitmq/amqp091-go"
)

// ตัวอย่าง 2: RabbitMQ connection
func connectRabbitMQ() (*amqp.Connection, *amqp.Channel) {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("Failed to connect to RabbitMQ: %v", err)
	}
	
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Failed to open channel: %v", err)
	}
	
	return conn, ch
}

// ตัวอย่าง 3: Simple producer/consumer (Work Queue)
func workQueueProducer(ch *amqp.Channel, queueName string, messages []string) {
	// Declare queue
	q, err := ch.QueueDeclare(
		queueName, // name
		true,      // durable (survives broker restart)
		false,     // auto-delete
		false,     // exclusive
		false,     // no-wait
		nil,       // arguments
	)
	if err != nil {
		log.Fatalf("Failed to declare queue: %v", err)
	}
	
	// Set QoS - process one message at a time per consumer
	ch.Qos(1, 0, false)
	
	for i, msg := range messages {
		err = ch.PublishWithContext(
			context.Background(),
			"",     // exchange (empty = default)
			q.Name, // routing key
			false,  // mandatory
			false,  // immediate
			amqp.Publishing{
				DeliveryMode: amqp.Persistent, // persist messages
				ContentType:  "application/json",
				MessageId:    fmt.Sprintf("msg-%d", i+1),
				Timestamp:    time.Now(),
				Body:         []byte(msg),
			},
		)
		
		if err != nil {
			log.Printf("Failed to publish: %v\n", err)
			continue
		}
		fmt.Printf("Sent: %s\n", msg)
	}
}

// ตัวอย่าง 4: Work queue consumer
func workQueueConsumer(ch *amqp.Channel, queueName, consumerName string, done <-chan struct{}) {
	msgs, err := ch.Consume(
		queueName,    // queue
		consumerName, // consumer name
		false,        // auto-ack (false = manual ack)
		false,        // exclusive
		false,        // no-local
		false,        // no-wait
		nil,          // arguments
	)
	if err != nil {
		log.Fatalf("Failed to register consumer: %v", err)
	}
	
	go func() {
		for {
			select {
			case msg, ok := <-msgs:
				if !ok {
					return
				}
				
				fmt.Printf("[%s] Processing: %s\n", consumerName, msg.Body)
				
				// Simulate work
				time.Sleep(100 * time.Millisecond)
				
				// Acknowledge message
				if err := msg.Ack(false); err != nil {
					fmt.Printf("Failed to ack: %v\n", err)
				}
				
			case <-done:
				return
			}
		}
	}()
}

// ตัวอย่าง 5: Topic Exchange (Pub/Sub)
func topicExchangeDemo(conn *amqp.Connection) {
	fmt.Println("=== Topic Exchange Demo ===")
	
	producerCh, _ := conn.Channel()
	defer producerCh.Close()
	
	consumerCh, _ := conn.Channel()
	defer consumerCh.Close()
	
	// Declare exchange
	exchangeName := "events"
	producerCh.ExchangeDeclare(exchangeName, "topic", true, false, false, false, nil)
	
	// Setup subscribers
	setupSubscriber := func(ch *amqp.Channel, routingKey, queueName string) <-chan amqp.Delivery {
		ch.ExchangeDeclare(exchangeName, "topic", true, false, false, false, nil)
		
		q, _ := ch.QueueDeclare(queueName, false, true, false, false, nil)
		ch.QueueBind(q.Name, routingKey, exchangeName, false, nil)
		
		msgs, _ := ch.Consume(q.Name, "", true, false, false, false, nil)
		return msgs
	}
	
	// Subscribe to user.* events
	userCh, _ := conn.Channel()
	defer userCh.Close()
	userMsgs := setupSubscriber(userCh, "user.*", "user-events")
	
	// Subscribe to order.* events
	orderCh, _ := conn.Channel()
	defer orderCh.Close()
	orderMsgs := setupSubscriber(orderCh, "order.*", "order-events")
	
	// Subscribe to everything
	allCh, _ := conn.Channel()
	defer allCh.Close()
	allMsgs := setupSubscriber(allCh, "#", "all-events")
	
	// Start consumers
	done := make(chan struct{})
	
	go func() {
		for msg := range userMsgs {
			select {
			case <-done:
				return
			default:
				fmt.Printf("[UserConsumer] %s: %s\n", msg.RoutingKey, msg.Body)
			}
		}
	}()
	
	go func() {
		for msg := range orderMsgs {
			select {
			case <-done:
				return
			default:
				fmt.Printf("[OrderConsumer] %s: %s\n", msg.RoutingKey, msg.Body)
			}
		}
	}()
	
	go func() {
		for msg := range allMsgs {
			select {
			case <-done:
				return
			default:
				fmt.Printf("[AuditLog] %s: %s\n", msg.RoutingKey, msg.Body)
			}
		}
	}()
	
	// Publish events
	time.Sleep(200 * time.Millisecond)
	
	events := []struct {
		routingKey string
		body       string
	}{
		{"user.registered", `{"user_id": 1, "email": "alice@example.com"}`},
		{"user.login", `{"user_id": 1, "ip": "192.168.1.1"}`},
		{"order.created", `{"order_id": 100, "total": 99.99}`},
		{"order.paid", `{"order_id": 100, "method": "credit_card"}`},
		{"user.logout", `{"user_id": 1}`},
	}
	
	for _, event := range events {
		producerCh.PublishWithContext(
			context.Background(),
			exchangeName,
			event.routingKey,
			false, false,
			amqp.Publishing{
				ContentType: "application/json",
				Body:        []byte(event.body),
			},
		)
		time.Sleep(100 * time.Millisecond)
	}
	
	time.Sleep(500 * time.Millisecond)
	close(done)
}

// ตัวอย่าง 6: Dead Letter Queue
func deadLetterQueueDemo(conn *amqp.Connection) {
	fmt.Println("\n=== Dead Letter Queue Demo ===")
	
	ch, _ := conn.Channel()
	defer ch.Close()
	
	dlxName := "dlx"
	dlqName := "dead-letters"
	mainQueueName := "main-queue"
	
	// Declare DLX
	ch.ExchangeDeclare(dlxName, "direct", true, false, false, false, nil)
	
	// Declare DLQ
	ch.QueueDeclare(dlqName, true, false, false, false, nil)
	ch.QueueBind(dlqName, dlqName, dlxName, false, nil)
	
	// Declare main queue with DLX settings
	ch.QueueDeclare(mainQueueName, true, false, false, false, amqp.Table{
		"x-dead-letter-exchange":    dlxName,
		"x-dead-letter-routing-key": dlqName,
		"x-message-ttl":             int32(5000), // 5 second TTL
		"x-max-delivery-count":      3,
	})
	
	fmt.Println("Dead Letter Queue setup:")
	fmt.Println("  main-queue -> (on reject/TTL/max-retries) -> dlx -> dead-letters")
}

func main() {
	fmt.Println("RabbitMQ examples require a running RabbitMQ instance")
	fmt.Println("Start with: docker run -p 5672:5672 rabbitmq:3-management")
	fmt.Println()
	fmt.Println("Key RabbitMQ concepts:")
	fmt.Println("  Exchange types: direct, fanout, topic, headers")
	fmt.Println("  Queue durability: durable queues survive broker restart")
	fmt.Println("  Message persistence: DeliveryMode=2 persists to disk")
	fmt.Println("  ACK/NACK: manual acknowledgment prevents message loss")
	fmt.Println("  Dead Letter Exchange: handles failed messages")
}
```

---

## 3. Kafka with confluent-kafka-go

```bash
# Install
go get github.com/confluentinc/confluent-kafka-go/v2/kafka
```

```go
package main

import (
	"fmt"
	"time"
	
	"github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

// ตัวอย่าง 7: Kafka Producer
func kafkaProducer() {
	p, err := kafka.NewProducer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
		"acks":              "all",              // wait for all replicas
		"retries":           5,
		"linger.ms":         5,                  // batch messages for 5ms
		"batch.size":        16384,              // 16KB batch
		"compression.type":  "snappy",
	})
	if err != nil {
		panic(fmt.Sprintf("Failed to create producer: %v", err))
	}
	defer p.Close()
	
	// Delivery reports channel
	go func() {
		for e := range p.Events() {
			switch ev := e.(type) {
			case *kafka.Message:
				if ev.TopicPartition.Error != nil {
					fmt.Printf("Delivery failed: %v\n", ev.TopicPartition.Error)
				} else {
					fmt.Printf("Delivered to %v[%d]@%v\n",
						*ev.TopicPartition.Topic,
						ev.TopicPartition.Partition,
						ev.TopicPartition.Offset)
				}
			}
		}
	}()
	
	topic := "user-events"
	
	// Produce messages
	events := []struct {
		key   string
		value string
	}{
		{"user:1", `{"event": "login", "user_id": 1, "timestamp": "2025-01-01T10:00:00Z"}`},
		{"user:2", `{"event": "purchase", "user_id": 2, "amount": 99.99}`},
		{"user:1", `{"event": "logout", "user_id": 1}`},
	}
	
	for _, e := range events {
		p.Produce(&kafka.Message{
			TopicPartition: kafka.TopicPartition{
				Topic:     &topic,
				Partition: kafka.PartitionAny,
			},
			Key:       []byte(e.key),
			Value:     []byte(e.value),
			Timestamp: time.Now(),
			Headers: []kafka.Header{
				{Key: "source", Value: []byte("go-service")},
			},
		}, nil)
	}
	
	// Wait for outstanding deliveries
	p.Flush(15 * 1000)
}

// ตัวอย่าง 8: Kafka Consumer
func kafkaConsumer() {
	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers":       "localhost:9092",
		"group.id":               "my-consumer-group",
		"auto.offset.reset":      "earliest",
		"enable.auto.commit":     false,          // manual commit
		"session.timeout.ms":     6000,
		"heartbeat.interval.ms":  2000,
		"max.poll.interval.ms":   300000,
	})
	if err != nil {
		panic(fmt.Sprintf("Failed to create consumer: %v", err))
	}
	defer c.Close()
	
	topics := []string{"user-events", "order-events"}
	c.SubscribeTopics(topics, nil)
	
	fmt.Println("Kafka consumer started, waiting for messages...")
	
	for i := 0; i < 10; i++ {
		msg, err := c.ReadMessage(1 * time.Second)
		if err != nil {
			if err.(kafka.Error).Code() == kafka.ErrTimedOut {
				continue
			}
			fmt.Printf("Consumer error: %v\n", err)
			continue
		}
		
		fmt.Printf("Received from [%s|%d|%d]: key=%s value=%s\n",
			*msg.TopicPartition.Topic,
			msg.TopicPartition.Partition,
			msg.TopicPartition.Offset,
			string(msg.Key),
			string(msg.Value),
		)
		
		// Manual commit after processing
		c.CommitMessage(msg)
	}
}

// ตัวอย่าง 9: Kafka Admin - create topics
func kafkaAdmin() {
	admin, err := kafka.NewAdminClient(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
	})
	if err != nil {
		panic(err)
	}
	defer admin.Close()
	
	topicSpec := kafka.TopicSpecification{
		Topic:             "new-topic",
		NumPartitions:     3,
		ReplicationFactor: 1,
		Config: map[string]string{
			"retention.ms":      "86400000", // 1 day
			"cleanup.policy":    "delete",
		},
	}
	
	results, err := admin.CreateTopics(
		context.Background(),
		[]kafka.TopicSpecification{topicSpec},
		kafka.SetAdminOperationTimeout(10*time.Second),
	)
	
	if err != nil {
		fmt.Printf("Failed to create topics: %v\n", err)
		return
	}
	
	for _, result := range results {
		fmt.Printf("Topic %s: %v\n", result.Topic, result.Error)
	}
}

// ตัวอย่าง 10: Exactly-once semantics with transactions
func kafkaTransactionalProducer() {
	p, err := kafka.NewProducer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
		"transactional.id": "my-transactional-producer",
		"acks":             "all",
	})
	if err != nil {
		panic(err)
	}
	defer p.Close()
	
	// Initialize transactions
	err = p.InitTransactions(context.Background())
	if err != nil {
		panic(err)
	}
	
	// Start transaction
	err = p.BeginTransaction()
	if err != nil {
		panic(err)
	}
	
	topic := "transactional-topic"
	
	// Produce within transaction
	for i := 0; i < 5; i++ {
		p.Produce(&kafka.Message{
			TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
			Value:          []byte(fmt.Sprintf("transactional message %d", i+1)),
		}, nil)
	}
	
	p.Flush(5000)
	
	// Commit transaction
	err = p.CommitTransaction(context.Background())
	if err != nil {
		fmt.Printf("Transaction failed, aborting: %v\n", err)
		p.AbortTransaction(context.Background())
	} else {
		fmt.Println("Transaction committed successfully")
	}
}

func main() {
	fmt.Println("Kafka examples require a running Kafka instance")
	fmt.Println("Start with: docker run -p 9092:9092 confluentinc/cp-kafka")
	fmt.Println()
	fmt.Println("Key Kafka concepts:")
	fmt.Println("  Topics: ordered log of messages")
	fmt.Println("  Partitions: parallelism and ordering within partition")
	fmt.Println("  Consumer Groups: load balancing consumers")
	fmt.Println("  Offsets: track position in topic")
	fmt.Println("  Transactions: exactly-once semantics")
}
```

---

## 4. NATS Messaging

```bash
# Install
go get github.com/nats-io/nats.go
```

```go
package main

import (
	"fmt"
	"sync"
	"time"
	
	"github.com/nats-io/nats.go"
)

// ตัวอย่าง 11: NATS basic pub/sub
func natsBasicPubSub(nc *nats.Conn) {
	fmt.Println("=== NATS Pub/Sub ===")
	
	// Subscribe
	sub, _ := nc.Subscribe("greetings", func(msg *nats.Msg) {
		fmt.Printf("Received: %s\n", string(msg.Data))
	})
	defer sub.Unsubscribe()
	
	// Publish
	nc.Publish("greetings", []byte("Hello, NATS!"))
	nc.Publish("greetings", []byte("สวัสดี NATS!"))
	
	time.Sleep(100 * time.Millisecond)
}

// ตัวอย่าง 12: NATS Request/Reply
func natsRequestReply(nc *nats.Conn) {
	fmt.Println("\n=== NATS Request/Reply ===")
	
	// Responder
	nc.Subscribe("calculate", func(msg *nats.Msg) {
		fmt.Printf("Received request: %s\n", msg.Data)
		// Process and reply
		msg.Respond([]byte("42"))
	})
	
	// Requester
	msg, err := nc.Request("calculate", []byte("6 * 7 = ?"), 2*time.Second)
	if err != nil {
		fmt.Printf("Request error: %v\n", err)
		return
	}
	fmt.Printf("Reply: %s\n", msg.Data)
}

// ตัวอย่าง 13: NATS Queue Groups (load balancing)
func natsQueueGroups(nc *nats.Conn) {
	fmt.Println("\n=== NATS Queue Groups ===")
	
	var wg sync.WaitGroup
	
	// Multiple workers in the same queue group
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		workerID := i
		nc.QueueSubscribe("work", "workers", func(msg *nats.Msg) {
			defer wg.Done()
			fmt.Printf("Worker %d processing: %s\n", workerID, msg.Data)
		})
	}
	
	// Send 6 messages (should be distributed among workers)
	for i := 1; i <= 6; i++ {
		nc.Publish("work", []byte(fmt.Sprintf("task-%d", i)))
	}
	
	time.Sleep(100 * time.Millisecond)
}

// ตัวอย่าง 14: NATS JetStream (persistent messaging)
func natsJetStream(nc *nats.Conn) {
	fmt.Println("\n=== NATS JetStream ===")
	
	// Create JetStream context
	js, err := nc.JetStream()
	if err != nil {
		fmt.Printf("JetStream error: %v\n", err)
		return
	}
	
	// Create stream
	js.AddStream(&nats.StreamConfig{
		Name:       "EVENTS",
		Subjects:   []string{"events.>"},
		MaxMsgs:    1000,
		MaxAge:     24 * time.Hour,
		Storage:    nats.FileStorage,
		Replicas:   1,
	})
	
	// Publish to stream
	js.Publish("events.user.created", []byte(`{"user_id": 1}`))
	js.Publish("events.order.placed", []byte(`{"order_id": 100}`))
	
	// Create consumer
	sub, _ := js.PullSubscribe("events.>", "event-processor",
		nats.PullMaxWaiting(128))
	defer sub.Unsubscribe()
	
	// Pull messages
	msgs, _ := sub.Fetch(10, nats.MaxWait(2*time.Second))
	for _, msg := range msgs {
		fmt.Printf("JetStream: %s -> %s\n", msg.Subject, msg.Data)
		msg.Ack()
	}
}

// ตัวอย่าง 15: NATS Key-Value Store
func natsKVStore(nc *nats.Conn) {
	fmt.Println("\n=== NATS Key-Value Store ===")
	
	js, _ := nc.JetStream()
	
	// Create KV bucket
	kv, err := js.CreateKeyValue(&nats.KeyValueConfig{
		Bucket: "users",
		TTL:    30 * time.Minute,
	})
	if err != nil {
		fmt.Printf("KV error: %v\n", err)
		return
	}
	
	// Put value
	kv.Put("user:1", []byte(`{"name": "Alice", "email": "alice@example.com"}`))
	
	// Get value
	entry, _ := kv.Get("user:1")
	fmt.Printf("KV get: key=%s value=%s rev=%d\n", entry.Key(), entry.Value(), entry.Revision())
	
	// Watch for changes
	watcher, _ := kv.Watch("user.*")
	defer watcher.Stop()
	
	// Update triggers watch
	kv.Put("user:2", []byte(`{"name": "Bob"}`))
	
	select {
	case update := <-watcher.Updates():
		if update != nil {
			fmt.Printf("KV update: %s = %s\n", update.Key(), update.Value())
		}
	case <-time.After(500 * time.Millisecond):
	}
	
	// Delete
	kv.Delete("user:1")
	
	// History
	history, _ := kv.History("user:1")
	fmt.Printf("KV history for user:1: %d entries\n", len(history))
}

func main() {
	// Connect to NATS
	nc, err := nats.Connect("nats://localhost:4222",
		nats.ReconnectWait(2*time.Second),
		nats.MaxReconnects(10),
		nats.DisconnectErrHandler(func(nc *nats.Conn, err error) {
			fmt.Printf("Disconnected: %v\n", err)
		}),
		nats.ReconnectHandler(func(nc *nats.Conn) {
			fmt.Printf("Reconnected to %s\n", nc.ConnectedUrl())
		}),
	)
	
	if err != nil {
		fmt.Println("NATS examples require a running NATS server")
		fmt.Println("Start with: docker run -p 4222:4222 nats:latest -js")
		return
	}
	defer nc.Close()
	
	natsBasicPubSub(nc)
	natsRequestReply(nc)
	natsQueueGroups(nc)
	natsJetStream(nc)
	natsKVStore(nc)
}
```

---

## 5. Message Patterns

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 16: Outbox Pattern - ensure message delivery with database

type OutboxMessage struct {
	ID        int
	EventType string
	Payload   []byte
	CreatedAt time.Time
	SentAt    *time.Time
	Retries   int
}

type OutboxPublisher struct {
	mu       sync.Mutex
	outbox   []OutboxMessage
	nextID   int
	
	publisher func(msg OutboxMessage) error
}

func NewOutboxPublisher(publisher func(msg OutboxMessage) error) *OutboxPublisher {
	op := &OutboxPublisher{publisher: publisher}
	go op.worker()
	return op
}

func (op *OutboxPublisher) Write(ctx context.Context, eventType string, payload []byte) error {
	op.mu.Lock()
	defer op.mu.Unlock()
	
	op.nextID++
	msg := OutboxMessage{
		ID:        op.nextID,
		EventType: eventType,
		Payload:   payload,
		CreatedAt: time.Now(),
	}
	
	// In real implementation: save to DB in same transaction
	op.outbox = append(op.outbox, msg)
	fmt.Printf("Outbox: queued message %d (%s)\n", msg.ID, msg.EventType)
	return nil
}

func (op *OutboxPublisher) worker() {
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()
	
	for range ticker.C {
		op.mu.Lock()
		for i := range op.outbox {
			msg := &op.outbox[i]
			if msg.SentAt != nil {
				continue // already sent
			}
			
			if err := op.publisher(*msg); err != nil {
				msg.Retries++
				fmt.Printf("Outbox: failed to send %d (retry %d): %v\n",
					msg.ID, msg.Retries, err)
			} else {
				now := time.Now()
				msg.SentAt = &now
				fmt.Printf("Outbox: sent message %d\n", msg.ID)
			}
		}
		op.mu.Unlock()
	}
}

// ตัวอย่าง 17: Saga Pattern for distributed transactions
type SagaStep struct {
	Name    string
	Execute func(ctx context.Context) error
	Compensate func(ctx context.Context) error
}

type Saga struct {
	steps    []SagaStep
	executed []SagaStep
}

func NewSaga(steps ...SagaStep) *Saga {
	return &Saga{steps: steps}
}

func (s *Saga) Execute(ctx context.Context) error {
	for _, step := range s.steps {
		fmt.Printf("Saga: executing step '%s'\n", step.Name)
		
		if err := step.Execute(ctx); err != nil {
			fmt.Printf("Saga: step '%s' failed: %v, compensating...\n", step.Name, err)
			
			// Compensate in reverse order
			for i := len(s.executed) - 1; i >= 0; i-- {
				compensateStep := s.executed[i]
				fmt.Printf("Saga: compensating '%s'\n", compensateStep.Name)
				if compErr := compensateStep.Compensate(ctx); compErr != nil {
					fmt.Printf("Saga: compensation error in '%s': %v\n",
						compensateStep.Name, compErr)
				}
			}
			
			return fmt.Errorf("saga failed at step '%s': %w", step.Name, err)
		}
		
		s.executed = append(s.executed, step)
	}
	
	fmt.Println("Saga: all steps completed successfully")
	return nil
}

func sagaDemo() {
	fmt.Println("=== Saga Pattern Demo ===")
	
	// Order processing saga
	saga := NewSaga(
		SagaStep{
			Name: "reserve-inventory",
			Execute: func(ctx context.Context) error {
				fmt.Println("  - Reserving inventory...")
				return nil // success
			},
			Compensate: func(ctx context.Context) error {
				fmt.Println("  - Releasing inventory reservation")
				return nil
			},
		},
		SagaStep{
			Name: "charge-payment",
			Execute: func(ctx context.Context) error {
				fmt.Println("  - Charging payment...")
				return fmt.Errorf("payment declined") // simulate failure
			},
			Compensate: func(ctx context.Context) error {
				fmt.Println("  - Reversing payment charge")
				return nil
			},
		},
		SagaStep{
			Name: "ship-order",
			Execute: func(ctx context.Context) error {
				fmt.Println("  - Creating shipment...")
				return nil
			},
			Compensate: func(ctx context.Context) error {
				fmt.Println("  - Cancelling shipment")
				return nil
			},
		},
	)
	
	if err := saga.Execute(context.Background()); err != nil {
		fmt.Printf("Saga failed: %v\n", err)
	}
}

// ตัวอย่าง 18: Event sourcing pattern
type Event struct {
	ID        int
	Type      string
	Payload   map[string]interface{}
	CreatedAt time.Time
}

type EventStore struct {
	mu     sync.Mutex
	events []Event
	nextID int
}

func (es *EventStore) Append(eventType string, payload map[string]interface{}) Event {
	es.mu.Lock()
	defer es.mu.Unlock()
	
	es.nextID++
	event := Event{
		ID:        es.nextID,
		Type:      eventType,
		Payload:   payload,
		CreatedAt: time.Now(),
	}
	es.events = append(es.events, event)
	return event
}

func (es *EventStore) GetAll() []Event {
	es.mu.Lock()
	defer es.mu.Unlock()
	return append([]Event{}, es.events...)
}

type BankAccount struct {
	ID      string
	Balance float64
	store   *EventStore
}

func (a *BankAccount) Deposit(amount float64) {
	a.store.Append("DEPOSITED", map[string]interface{}{
		"account_id": a.ID,
		"amount":     amount,
	})
	a.Balance += amount
}

func (a *BankAccount) Withdraw(amount float64) error {
	if a.Balance < amount {
		return fmt.Errorf("insufficient funds")
	}
	a.store.Append("WITHDRAWN", map[string]interface{}{
		"account_id": a.ID,
		"amount":     amount,
	})
	a.Balance -= amount
	return nil
}

func (a *BankAccount) Replay(events []Event) {
	a.Balance = 0
	for _, event := range events {
		switch event.Type {
		case "DEPOSITED":
			a.Balance += event.Payload["amount"].(float64)
		case "WITHDRAWN":
			a.Balance -= event.Payload["amount"].(float64)
		}
	}
}

func eventSourcingDemo() {
	fmt.Println("\n=== Event Sourcing Demo ===")
	
	store := &EventStore{}
	account := &BankAccount{ID: "acc-001", store: store}
	
	account.Deposit(1000)
	account.Deposit(500)
	account.Withdraw(200)
	account.Deposit(100)
	account.Withdraw(300)
	
	fmt.Printf("Current balance: %.2f\n", account.Balance)
	
	// Replay events to verify
	newAccount := &BankAccount{ID: "acc-001", store: store}
	newAccount.Replay(store.GetAll())
	fmt.Printf("Replayed balance: %.2f\n", newAccount.Balance)
	
	fmt.Printf("Event history:\n")
	for _, event := range store.GetAll() {
		fmt.Printf("  [%s] %s: amount=%.2f\n",
			event.Type,
			event.CreatedAt.Format("15:04:05"),
			event.Payload["amount"])
	}
}

func main() {
	// Outbox demo
	publisher := NewOutboxPublisher(func(msg OutboxMessage) error {
		fmt.Printf("  Publishing: %s - %s\n", msg.EventType, msg.Payload)
		return nil
	})
	
	ctx := context.Background()
	publisher.Write(ctx, "order.created", []byte(`{"order_id": 1}`))
	publisher.Write(ctx, "payment.processed", []byte(`{"payment_id": 101}`))
	publisher.Write(ctx, "notification.sent", []byte(`{"user_id": 42}`))
	
	time.Sleep(300 * time.Millisecond)
	
	sagaDemo()
	eventSourcingDemo()
}
```

---

## 6. Message Acknowledgment Patterns

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// ตัวอย่าง 19: At-least-once delivery with acknowledgment

type AckableMessage struct {
	ID      string
	Body    []byte
	ack     chan struct{}
	nack    chan error
}

func (m *AckableMessage) Ack() {
	close(m.ack)
}

func (m *AckableMessage) Nack(err error) {
	m.nack <- err
}

type AckableQueue struct {
	mu       sync.Mutex
	inflight map[string]*AckableMessage
	queue    chan *AckableMessage
	deadLetter chan *AckableMessage
}

func NewAckableQueue(bufferSize int) *AckableQueue {
	return &AckableQueue{
		inflight:   make(map[string]*AckableMessage),
		queue:      make(chan *AckableMessage, bufferSize),
		deadLetter: make(chan *AckableMessage, bufferSize),
	}
}

func (q *AckableQueue) Publish(id string, body []byte) {
	msg := &AckableMessage{
		ID:   id,
		Body: body,
		ack:  make(chan struct{}),
		nack: make(chan error, 1),
	}
	
	q.mu.Lock()
	q.inflight[id] = msg
	q.mu.Unlock()
	
	q.queue <- msg
	
	// Monitor ack/nack
	go func() {
		select {
		case <-msg.ack:
			q.mu.Lock()
			delete(q.inflight, id)
			q.mu.Unlock()
			fmt.Printf("Message %s acknowledged\n", id)
		case err := <-msg.nack:
			q.mu.Lock()
			delete(q.inflight, id)
			q.mu.Unlock()
			fmt.Printf("Message %s nacked: %v, sending to DLQ\n", id, err)
			q.deadLetter <- msg
		case <-time.After(30 * time.Second):
			// Visibility timeout - requeue
			fmt.Printf("Message %s timed out, requeuing\n", id)
			q.queue <- msg
		}
	}()
}

func (q *AckableQueue) Receive(ctx context.Context) (*AckableMessage, error) {
	select {
	case msg := <-q.queue:
		return msg, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

func ackableQueueDemo() {
	fmt.Println("=== Acknowledgment Queue Demo ===")
	
	q := NewAckableQueue(100)
	
	// Publish messages
	q.Publish("msg-1", []byte("important task 1"))
	q.Publish("msg-2", []byte("important task 2"))
	q.Publish("msg-3", []byte("important task 3"))
	
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	
	// Process messages
	for i := 0; i < 3; i++ {
		msg, err := q.Receive(ctx)
		if err != nil {
			fmt.Printf("Error receiving: %v\n", err)
			break
		}
		
		fmt.Printf("Processing: %s - %s\n", msg.ID, msg.Body)
		
		// Simulate processing: fail msg-2
		if msg.ID == "msg-2" {
			msg.Nack(fmt.Errorf("processing failed"))
		} else {
			msg.Ack()
		}
	}
	
	// Check dead letter queue
	time.Sleep(100 * time.Millisecond)
	select {
	case dlMsg := <-q.deadLetter:
		fmt.Printf("Dead letter: %s - %s\n", dlMsg.ID, dlMsg.Body)
	default:
		fmt.Println("No dead letter messages")
	}
}

// ตัวอย่าง 20: Message deduplication
type DeduplicatingQueue struct {
	mu        sync.Mutex
	seen      map[string]time.Time
	queue     chan interface{}
	dedupeWindow time.Duration
}

func NewDeduplicatingQueue(bufferSize int, window time.Duration) *DeduplicatingQueue {
	q := &DeduplicatingQueue{
		seen:         make(map[string]time.Time),
		queue:        make(chan interface{}, bufferSize),
		dedupeWindow: window,
	}
	go q.cleanup()
	return q
}

func (q *DeduplicatingQueue) Publish(dedupeKey string, msg interface{}) bool {
	q.mu.Lock()
	defer q.mu.Unlock()
	
	if _, seen := q.seen[dedupeKey]; seen {
		fmt.Printf("Duplicate message dropped: %s\n", dedupeKey)
		return false
	}
	
	q.seen[dedupeKey] = time.Now()
	q.queue <- msg
	return true
}

func (q *DeduplicatingQueue) cleanup() {
	ticker := time.NewTicker(time.Minute)
	for range ticker.C {
		q.mu.Lock()
		cutoff := time.Now().Add(-q.dedupeWindow)
		for key, t := range q.seen {
			if t.Before(cutoff) {
				delete(q.seen, key)
			}
		}
		q.mu.Unlock()
	}
}

func deduplicationDemo() {
	fmt.Println("\n=== Message Deduplication Demo ===")
	
	q := NewDeduplicatingQueue(100, 5*time.Minute)
	
	// Simulate duplicate messages (e.g., from retry logic)
	messages := []struct {
		key  string
		body string
	}{
		{"order:100:created", "Order 100 created"},
		{"order:100:created", "Order 100 created (duplicate)"},
		{"order:101:created", "Order 101 created"},
		{"order:100:created", "Order 100 created (another duplicate)"},
		{"payment:abc:processed", "Payment ABC processed"},
	}
	
	for _, msg := range messages {
		sent := q.Publish(msg.key, msg.body)
		fmt.Printf("Message '%s': sent=%v\n", msg.key, sent)
	}
}

func main() {
	ackableQueueDemo()
	deduplicationDemo()
}
```

---

## สรุป

ใน Part 41 เราได้เรียนรู้:

1. **Message Queue Concepts**: async communication, decoupling, buffering
2. **RabbitMQ**: AMQP protocol, exchanges, queues, DLQ
3. **Kafka**: distributed log, consumer groups, transactions
4. **NATS**: lightweight messaging, JetStream, KV store
5. **Pub/Sub Pattern**: broadcast to multiple subscribers
6. **Work Queues**: distribute tasks among workers
7. **Request/Reply**: synchronous-over-async messaging
8. **Dead Letter Queue**: handle failed messages
9. **Outbox Pattern**: reliable message publishing
10. **Saga Pattern**: distributed transaction management
11. **Event Sourcing**: audit trail and state reconstruction
12. **Message Deduplication**: prevent duplicate processing

---

## Resources

- [RabbitMQ Go Client](https://github.com/rabbitmq/amqp091-go)
- [Confluent Kafka Go](https://github.com/confluentinc/confluent-kafka-go)
- [NATS Go Client](https://github.com/nats-io/nats.go)
- [Message Queue Patterns](https://www.enterpriseintegrationpatterns.com/)
