# Part 73: Real-time Systems

## เป้าหมายการเรียนรู้
- Real-time System Requirements
- WebSocket at Scale
- Server-Sent Events (SSE)
- Long Polling
- Real-time กับ Redis Pub/Sub
- NATS for Real-time Messaging
- Operational Transformation
- CRDTs

---

## 1. Real-time System Requirements

```
Latency Requirements:
├── Ultra-low latency (<1ms): Gaming, Trading, Control systems
├── Low latency (<10ms): Video/Audio streaming, Real-time chat
├── Near real-time (<100ms): Notifications, Live updates
└── Soft real-time (<1s): Email, Batch notifications
```

```go
// realtime/metrics.go
package realtime

import (
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    messageLatency = promauto.NewHistogram(prometheus.HistogramOpts{
        Name:    "realtime_message_latency_ms",
        Help:    "Message delivery latency",
        Buckets: []float64{1, 5, 10, 25, 50, 100, 250, 500, 1000},
    })
    
    activeConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "realtime_active_connections",
        Help: "Number of active real-time connections",
    })
    
    messagesPerSecond = promauto.NewCounter(prometheus.CounterOpts{
        Name: "realtime_messages_total",
        Help: "Total messages delivered",
    })
)

func trackLatency(start time.Time) {
    latency := time.Since(start).Milliseconds()
    messageLatency.Observe(float64(latency))
    messagesPerSecond.Inc()
}
```

---

## 2. WebSocket at Scale

```go
// websocket/hub.go
package websocket

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10
    maxMessageSize = 512 * 1024 // 512KB
)

type Message struct {
    Type      string      `json:"type"`
    Payload   interface{} `json:"payload"`
    Timestamp time.Time   `json:"timestamp"`
    From      string      `json:"from,omitempty"`
    To        string      `json:"to,omitempty"`
    Room      string      `json:"room,omitempty"`
}

type Client struct {
    ID     string
    UserID string
    Room   string
    hub    *Hub
    conn   *websocket.Conn
    send   chan []byte
    done   chan struct{}
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    
    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })
    
    for {
        _, data, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("WebSocket error (client %s): %v", c.ID, err)
            }
            break
        }
        
        var msg Message
        if err := json.Unmarshal(data, &msg); err != nil {
            log.Printf("Invalid message from %s: %v", c.ID, err)
            continue
        }
        
        msg.From = c.UserID
        msg.Timestamp = time.Now()
        
        c.hub.processMessage(c, &msg)
    }
}

func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()
    
    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            
            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)
            
            // Batch pending messages
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'})
                w.Write(<-c.send)
            }
            
            if err := w.Close(); err != nil {
                return
            }
            
        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
            
        case <-c.done:
            return
        }
    }
}

// Hub จัดการ connections ทั้งหมด
type Hub struct {
    clients    map[string]*Client           // client ID -> client
    rooms      map[string]map[string]*Client // room -> client ID -> client
    register   chan *Client
    unregister chan *Client
    broadcast  chan *roomMessage
    mu         sync.RWMutex
    
    // Distributed pub/sub
    pubsub     PubSub
}

type roomMessage struct {
    room    string
    message []byte
}

func NewHub(pubsub PubSub) *Hub {
    h := &Hub{
        clients:    make(map[string]*Client),
        rooms:      make(map[string]map[string]*Client),
        register:   make(chan *Client, 256),
        unregister: make(chan *Client, 256),
        broadcast:  make(chan *roomMessage, 1024),
        pubsub:     pubsub,
    }
    
    go h.run()
    return h
}

func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client.ID] = client
            if _, ok := h.rooms[client.Room]; !ok {
                h.rooms[client.Room] = make(map[string]*Client)
            }
            h.rooms[client.Room][client.ID] = client
            h.mu.Unlock()
            
            activeConnections.Inc()
            log.Printf("Client %s joined room %s", client.ID, client.Room)
            
        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client.ID]; ok {
                delete(h.clients, client.ID)
                if room, ok := h.rooms[client.Room]; ok {
                    delete(room, client.ID)
                }
                close(client.send)
            }
            h.mu.Unlock()
            
            activeConnections.Dec()
            
        case msg := <-h.broadcast:
            h.mu.RLock()
            if room, ok := h.rooms[msg.room]; ok {
                for _, client := range room {
                    select {
                    case client.send <- msg.message:
                    default:
                        // Client's buffer full - disconnect
                        go func(c *Client) {
                            h.unregister <- c
                        }(client)
                    }
                }
            }
            h.mu.RUnlock()
        }
    }
}

func (h *Hub) processMessage(from *Client, msg *Message) {
    data, _ := json.Marshal(msg)
    
    switch msg.Type {
    case "chat":
        // Broadcast ไปทุกคนใน room
        h.broadcast <- &roomMessage{room: from.Room, message: data}
        
        // Publish ไป other servers (สำหรับ horizontal scaling)
        h.pubsub.Publish(context.Background(), "room:"+from.Room, data)
        
    case "private":
        // ส่งให้เฉพาะ user ที่ระบุ
        h.mu.RLock()
        if target, ok := h.clients[msg.To]; ok {
            target.send <- data
        }
        h.mu.RUnlock()
        
    case "typing":
        // Broadcast typing indicator (ไม่เก็บใน history)
        h.broadcast <- &roomMessage{room: from.Room, message: data}
    }
}

// HTTP Upgrader
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true // ควรตรวจสอบ origin ใน production
    },
}

func (h *Hub) ServeWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket upgrade error: %v", err)
        return
    }
    
    // ดึง user info จาก JWT
    userID := r.Context().Value("user_id").(string)
    room := r.URL.Query().Get("room")
    if room == "" {
        room = "general"
    }
    
    client := &Client{
        ID:     generateID(),
        UserID: userID,
        Room:   room,
        hub:    h,
        conn:   conn,
        send:   make(chan []byte, 256),
        done:   make(chan struct{}),
    }
    
    h.register <- client
    
    // ส่ง welcome message
    welcome, _ := json.Marshal(Message{
        Type:      "welcome",
        Payload:   map[string]string{"room": room, "client_id": client.ID},
        Timestamp: time.Now(),
    })
    client.send <- welcome
    
    go client.writePump()
    go client.readPump()
}
```

### Horizontal Scaling กับ Redis

```go
// websocket/distributed.go
package websocket

import (
    "context"
    "encoding/json"
    
    "github.com/redis/go-redis/v9"
)

type RedisPubSub struct {
    client *redis.Client
    hub    *Hub
}

func NewRedisPubSub(client *redis.Client, hub *Hub) *RedisPubSub {
    ps := &RedisPubSub{client: client, hub: hub}
    return ps
}

func (ps *RedisPubSub) Publish(ctx context.Context, channel string, message []byte) error {
    return ps.client.Publish(ctx, channel, message).Err()
}

// Subscribe สำหรับ messages จาก servers อื่น
func (ps *RedisPubSub) Subscribe(ctx context.Context, channels ...string) {
    sub := ps.client.Subscribe(ctx, channels...)
    
    go func() {
        defer sub.Close()
        
        for msg := range sub.Channel() {
            var wsMsg Message
            if err := json.Unmarshal([]byte(msg.Payload), &wsMsg); err != nil {
                continue
            }
            
            // ส่งไปยัง local clients ใน room
            data, _ := json.Marshal(wsMsg)
            roomName := extractRoom(msg.Channel)
            ps.hub.broadcast <- &roomMessage{room: roomName, message: data}
        }
    }()
}

func extractRoom(channel string) string {
    // "room:general" -> "general"
    if len(channel) > 5 {
        return channel[5:]
    }
    return ""
}
```

---

## 3. Server-Sent Events (SSE)

SSE ดีกว่า WebSocket สำหรับ one-way streaming จาก server ไป client

```go
// sse/server.go
package sse

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

type Event struct {
    ID      string
    Event   string
    Data    interface{}
    Retry   int
}

func (e *Event) Format() string {
    data, _ := json.Marshal(e.Data)
    
    result := ""
    if e.ID != "" {
        result += fmt.Sprintf("id: %s\n", e.ID)
    }
    if e.Event != "" {
        result += fmt.Sprintf("event: %s\n", e.Event)
    }
    if e.Retry > 0 {
        result += fmt.Sprintf("retry: %d\n", e.Retry)
    }
    result += fmt.Sprintf("data: %s\n\n", string(data))
    
    return result
}

type SSEBroker struct {
    clients    map[string]chan *Event
    register   chan *sseClient
    unregister chan string
    publish    chan *topicEvent
    mu         sync.RWMutex
}

type sseClient struct {
    id      string
    topic   string
    channel chan *Event
}

type topicEvent struct {
    topic string
    event *Event
}

func NewSSEBroker() *SSEBroker {
    broker := &SSEBroker{
        clients:    make(map[string]chan *Event),
        register:   make(chan *sseClient, 100),
        unregister: make(chan string, 100),
        publish:    make(chan *topicEvent, 1000),
    }
    
    go broker.run()
    return broker
}

func (b *SSEBroker) run() {
    topics := make(map[string]map[string]chan *Event) // topic -> clientID -> channel
    
    for {
        select {
        case client := <-b.register:
            if _, ok := topics[client.topic]; !ok {
                topics[client.topic] = make(map[string]chan *Event)
            }
            topics[client.topic][client.id] = client.channel
            
        case id := <-b.unregister:
            for _, clients := range topics {
                delete(clients, id)
            }
            
        case te := <-b.publish:
            if clients, ok := topics[te.topic]; ok {
                for _, ch := range clients {
                    select {
                    case ch <- te.event:
                    default:
                        // Client too slow - skip event
                    }
                }
            }
        }
    }
}

func (b *SSEBroker) Publish(topic string, event *Event) {
    b.publish <- &topicEvent{topic: topic, event: event}
}

func (b *SSEBroker) Handler(w http.ResponseWriter, r *http.Request) {
    topic := r.URL.Query().Get("topic")
    if topic == "" {
        topic = "general"
    }
    
    // Set SSE headers
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("X-Accel-Buffering", "no") // สำหรับ Nginx
    
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "SSE not supported", http.StatusInternalServerError)
        return
    }
    
    // ส่ง retry config ให้ client
    fmt.Fprintf(w, "retry: 3000\n\n")
    flusher.Flush()
    
    // Register client
    clientID := generateID()
    ch := make(chan *Event, 100)
    
    b.register <- &sseClient{
        id:      clientID,
        topic:   topic,
        channel: ch,
    }
    
    defer func() {
        b.unregister <- clientID
    }()
    
    // Handle Last-Event-ID (reconnection)
    lastEventID := r.Header.Get("Last-Event-ID")
    if lastEventID != "" {
        // ส่ง events ที่ client พลาดไป
        go b.replayMissedEvents(clientID, lastEventID, ch)
    }
    
    // Send heartbeat
    heartbeat := time.NewTicker(15 * time.Second)
    defer heartbeat.Stop()
    
    for {
        select {
        case event, ok := <-ch:
            if !ok {
                return
            }
            fmt.Fprint(w, event.Format())
            flusher.Flush()
            
        case <-heartbeat.C:
            // Heartbeat เพื่อ keep-alive
            fmt.Fprintf(w, ": heartbeat\n\n")
            flusher.Flush()
            
        case <-r.Context().Done():
            return
        }
    }
}

func (b *SSEBroker) replayMissedEvents(clientID, lastEventID string, ch chan *Event) {
    // ดึง events จาก event store ที่มี ID > lastEventID
    // และส่งไปยัง channel
}

// ตัวอย่างการใช้งาน: Live Stock Prices
func stockPriceStreamer(broker *SSEBroker) {
    go func() {
        for {
            time.Sleep(500 * time.Millisecond)
            
            event := &Event{
                ID:    fmt.Sprintf("%d", time.Now().UnixNano()),
                Event: "price-update",
                Data: map[string]interface{}{
                    "symbol": "AAPL",
                    "price":  150.25 + float64(rand.Intn(100))/10,
                    "change": (rand.Float64() - 0.5) * 2,
                },
            }
            
            broker.Publish("stocks", event)
        }
    }()
}
```

---

## 4. Long Polling

```go
// longpolling/server.go
package longpolling

import (
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

type Notification struct {
    ID        string    `json:"id"`
    Type      string    `json:"type"`
    Message   string    `json:"message"`
    Timestamp time.Time `json:"timestamp"`
}

type WaitingClient struct {
    userID   string
    since    time.Time
    response chan []Notification
}

type LongPollServer struct {
    waiting      map[string][]*WaitingClient
    notifications map[string][]Notification
    mu           sync.Mutex
    timeout      time.Duration
}

func NewLongPollServer() *LongPollServer {
    return &LongPollServer{
        waiting:       make(map[string][]*WaitingClient),
        notifications: make(map[string][]Notification),
        timeout:       30 * time.Second,
    }
}

func (s *LongPollServer) Poll(w http.ResponseWriter, r *http.Request) {
    userID := r.Context().Value("user_id").(string)
    
    sinceStr := r.URL.Query().Get("since")
    since, err := time.Parse(time.RFC3339, sinceStr)
    if err != nil {
        since = time.Now().Add(-1 * time.Minute)
    }
    
    // ตรวจสอบ notifications ที่มีอยู่แล้ว
    s.mu.Lock()
    notifications := s.getNotificationsSince(userID, since)
    
    if len(notifications) > 0 {
        s.mu.Unlock()
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(notifications)
        return
    }
    
    // ถ้าไม่มี notifications - รอ
    ch := make(chan []Notification, 1)
    client := &WaitingClient{
        userID:   userID,
        since:    since,
        response: ch,
    }
    
    s.waiting[userID] = append(s.waiting[userID], client)
    s.mu.Unlock()
    
    defer func() {
        s.mu.Lock()
        clients := s.waiting[userID]
        for i, c := range clients {
            if c == client {
                s.waiting[userID] = append(clients[:i], clients[i+1:]...)
                break
            }
        }
        s.mu.Unlock()
    }()
    
    select {
    case notifications := <-ch:
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(notifications)
    case <-time.After(s.timeout):
        // Timeout - return empty response
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode([]Notification{})
    case <-r.Context().Done():
        return
    }
}

func (s *LongPollServer) Push(userID string, notification Notification) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    // เก็บ notification
    s.notifications[userID] = append(s.notifications[userID], notification)
    
    // ตัด notifications เก่า (เก็บแค่ 1000 รายการล่าสุด)
    if len(s.notifications[userID]) > 1000 {
        s.notifications[userID] = s.notifications[userID][len(s.notifications[userID])-1000:]
    }
    
    // ตอบกลับ waiting clients
    for _, client := range s.waiting[userID] {
        notifications := s.getNotificationsSince(userID, client.since)
        if len(notifications) > 0 {
            select {
            case client.response <- notifications:
            default:
            }
        }
    }
}

func (s *LongPollServer) getNotificationsSince(userID string, since time.Time) []Notification {
    var result []Notification
    for _, n := range s.notifications[userID] {
        if n.Timestamp.After(since) {
            result = append(result, n)
        }
    }
    return result
}
```

---

## 5. NATS สำหรับ Real-time Messaging

```go
// nats/client.go
package nats

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
    
    "github.com/nats-io/nats.go"
)

type NATSClient struct {
    nc *nats.Conn
    js nats.JetStreamContext
}

func NewNATSClient(url string) (*NATSClient, error) {
    nc, err := nats.Connect(url,
        nats.MaxReconnects(-1),
        nats.ReconnectWait(2*time.Second),
        nats.ErrorHandler(func(nc *nats.Conn, sub *nats.Subscription, err error) {
            log.Printf("NATS error: %v", err)
        }),
        nats.DisconnectErrHandler(func(nc *nats.Conn, err error) {
            log.Printf("NATS disconnected: %v", err)
        }),
        nats.ReconnectHandler(func(nc *nats.Conn) {
            log.Printf("NATS reconnected to %s", nc.ConnectedUrl())
        }),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to connect to NATS: %w", err)
    }
    
    // JetStream สำหรับ persistent messaging
    js, err := nc.JetStream()
    if err != nil {
        return nil, fmt.Errorf("failed to create JetStream context: %w", err)
    }
    
    return &NATSClient{nc: nc, js: js}, nil
}

// Publish message
func (c *NATSClient) Publish(subject string, data interface{}) error {
    payload, err := json.Marshal(data)
    if err != nil {
        return err
    }
    
    return c.nc.Publish(subject, payload)
}

// Subscribe with handler
func (c *NATSClient) Subscribe(subject string, handler func(data []byte)) (*nats.Subscription, error) {
    return c.nc.Subscribe(subject, func(msg *nats.Msg) {
        handler(msg.Data)
    })
}

// Queue Subscribe - load balance ระหว่าง instances
func (c *NATSClient) QueueSubscribe(subject, queue string, handler func(data []byte)) (*nats.Subscription, error) {
    return c.nc.QueueSubscribe(subject, queue, func(msg *nats.Msg) {
        handler(msg.Data)
    })
}

// Request-Reply pattern
func (c *NATSClient) Request(subject string, data interface{}, timeout time.Duration) ([]byte, error) {
    payload, _ := json.Marshal(data)
    
    msg, err := c.nc.Request(subject, payload, timeout)
    if err != nil {
        return nil, err
    }
    
    return msg.Data, nil
}

// JetStream - persistent streaming
func (c *NATSClient) SetupStream(name string, subjects []string) error {
    _, err := c.js.AddStream(&nats.StreamConfig{
        Name:     name,
        Subjects: subjects,
        MaxAge:   24 * time.Hour,
        Storage:  nats.FileStorage,
        Replicas: 3,
    })
    
    if err != nil && err != nats.ErrStreamNameAlreadyInUse {
        return fmt.Errorf("failed to setup stream: %w", err)
    }
    
    return nil
}

func (c *NATSClient) PublishPersistent(subject string, data interface{}) (*nats.PubAck, error) {
    payload, _ := json.Marshal(data)
    return c.js.Publish(subject, payload)
}

func (c *NATSClient) ConsumePersistent(
    stream, consumer, subject string,
    handler func([]byte) bool,
) error {
    sub, err := c.js.PushSubscribeSync(subject,
        nats.Durable(consumer),
        nats.AckExplicit(),
    )
    if err != nil {
        return err
    }
    
    go func() {
        for {
            msg, err := sub.NextMsg(30 * time.Second)
            if err != nil {
                if err == nats.ErrTimeout {
                    continue
                }
                log.Printf("Consumer error: %v", err)
                return
            }
            
            if handler(msg.Data) {
                msg.Ack()
            } else {
                msg.Nak()
            }
        }
    }()
    
    return nil
}

// ตัวอย่าง: Real-time Order Updates
type OrderUpdate struct {
    OrderID string `json:"order_id"`
    Status  string `json:"status"`
    Message string `json:"message"`
}

func orderUpdateService(nc *NATSClient) {
    // สร้าง stream สำหรับ order updates
    nc.SetupStream("ORDERS", []string{"orders.>"})
    
    // Publish order update
    nc.PublishPersistent("orders.123", OrderUpdate{
        OrderID: "123",
        Status:  "shipped",
        Message: "Your order is on the way!",
    })
    
    // Subscribe สำหรับ real-time updates
    nc.Subscribe("orders.123", func(data []byte) {
        var update OrderUpdate
        json.Unmarshal(data, &update)
        fmt.Printf("Order %s: %s\n", update.OrderID, update.Status)
    })
}
```

---

## 6. Operational Transformation (OT)

OT ใช้สำหรับ collaborative editing (เช่น Google Docs)

```go
// ot/operation.go
package ot

import "fmt"

// Operations สำหรับ text editing
type OpType int

const (
    Insert OpType = iota
    Delete
    Retain
)

type Operation struct {
    Type    OpType
    Count   int    // สำหรับ Retain และ Delete
    Content string // สำหรับ Insert
}

type ChangeSet struct {
    Operations []Operation
    BaseLength int    // ความยาว document ก่อน apply
    ResultLength int  // ความยาว document หลัง apply
}

// Apply operation กับ document
func (cs *ChangeSet) Apply(doc string) (string, error) {
    if len(doc) != cs.BaseLength {
        return "", fmt.Errorf("document length mismatch: expected %d, got %d", 
            cs.BaseLength, len(doc))
    }
    
    var result []rune
    runes := []rune(doc)
    pos := 0
    
    for _, op := range cs.Operations {
        switch op.Type {
        case Retain:
            if pos+op.Count > len(runes) {
                return "", fmt.Errorf("retain out of bounds")
            }
            result = append(result, runes[pos:pos+op.Count]...)
            pos += op.Count
            
        case Insert:
            result = append(result, []rune(op.Content)...)
            
        case Delete:
            if pos+op.Count > len(runes) {
                return "", fmt.Errorf("delete out of bounds")
            }
            pos += op.Count
        }
    }
    
    return string(result), nil
}

// Transform สอง operations ที่ concurrent กัน
// Assumes OP1 ถูก apply ก่อน OP2
func Transform(op1, op2 *ChangeSet) (*ChangeSet, *ChangeSet, error) {
    // Simple implementation สำหรับ demonstration
    // Production ควรใช้ library ที่ mature แล้ว
    
    if op1.BaseLength != op2.BaseLength {
        return nil, nil, fmt.Errorf("base lengths don't match")
    }
    
    op1Transformed := &ChangeSet{BaseLength: op2.ResultLength}
    op2Transformed := &ChangeSet{BaseLength: op1.ResultLength}
    
    i1, i2 := 0, 0
    ops1, ops2 := op1.Operations, op2.Operations
    
    for i1 < len(ops1) || i2 < len(ops2) {
        var a, b Operation
        
        if i1 < len(ops1) {
            a = ops1[i1]
        }
        if i2 < len(ops2) {
            b = ops2[i2]
        }
        
        if a.Type == Insert {
            op1Transformed.Operations = append(op1Transformed.Operations, a)
            op2Transformed.Operations = append(op2Transformed.Operations, 
                Operation{Type: Retain, Count: len([]rune(a.Content))})
            i1++
            continue
        }
        
        if b.Type == Insert {
            op1Transformed.Operations = append(op1Transformed.Operations,
                Operation{Type: Retain, Count: len([]rune(b.Content))})
            op2Transformed.Operations = append(op2Transformed.Operations, b)
            i2++
            continue
        }
        
        // Both retain or delete
        // Simplified - full implementation handles all cases
        break
    }
    
    return op1Transformed, op2Transformed, nil
}
```

---

## 7. CRDTs (Conflict-free Replicated Data Types)

```go
// crdt/types.go
package crdt

import (
    "sync"
    "time"
)

// G-Counter: increment-only counter
type GCounter struct {
    nodeID  string
    counts  map[string]int
    mu      sync.RWMutex
}

func NewGCounter(nodeID string) *GCounter {
    return &GCounter{
        nodeID: nodeID,
        counts: make(map[string]int),
    }
}

func (c *GCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counts[c.nodeID]++
}

func (c *GCounter) Value() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    total := 0
    for _, count := range c.counts {
        total += count
    }
    return total
}

func (c *GCounter) Merge(other *GCounter) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    other.mu.RLock()
    defer other.mu.RUnlock()
    
    for nodeID, count := range other.counts {
        if c.counts[nodeID] < count {
            c.counts[nodeID] = count
        }
    }
}

// PN-Counter: increment and decrement counter
type PNCounter struct {
    positive *GCounter
    negative *GCounter
}

func NewPNCounter(nodeID string) *PNCounter {
    return &PNCounter{
        positive: NewGCounter(nodeID),
        negative: NewGCounter(nodeID),
    }
}

func (c *PNCounter) Increment() {
    c.positive.Increment()
}

func (c *PNCounter) Decrement() {
    c.negative.Increment()
}

func (c *PNCounter) Value() int {
    return c.positive.Value() - c.negative.Value()
}

func (c *PNCounter) Merge(other *PNCounter) {
    c.positive.Merge(other.positive)
    c.negative.Merge(other.negative)
}

// LWW-Register: Last-Write-Wins Register
type LWWRegister struct {
    value     interface{}
    timestamp time.Time
    nodeID    string
    mu        sync.RWMutex
}

func NewLWWRegister(nodeID string) *LWWRegister {
    return &LWWRegister{nodeID: nodeID}
}

func (r *LWWRegister) Set(value interface{}) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    r.value = value
    r.timestamp = time.Now()
}

func (r *LWWRegister) Get() interface{} {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return r.value
}

func (r *LWWRegister) Merge(other *LWWRegister) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    other.mu.RLock()
    defer other.mu.RUnlock()
    
    if other.timestamp.After(r.timestamp) {
        r.value = other.value
        r.timestamp = other.timestamp
    }
}

// OR-Set: Observed-Remove Set
type ORSet struct {
    nodeID  string
    added   map[interface{}]map[string]time.Time // item -> nodeID -> timestamp
    removed map[interface{}]map[string]time.Time
    mu      sync.RWMutex
}

func NewORSet(nodeID string) *ORSet {
    return &ORSet{
        nodeID:  nodeID,
        added:   make(map[interface{}]map[string]time.Time),
        removed: make(map[interface{}]map[string]time.Time),
    }
}

func (s *ORSet) Add(item interface{}) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if _, ok := s.added[item]; !ok {
        s.added[item] = make(map[string]time.Time)
    }
    s.added[item][s.nodeID] = time.Now()
}

func (s *ORSet) Remove(item interface{}) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if entries, ok := s.added[item]; ok {
        if _, ok := s.removed[item]; !ok {
            s.removed[item] = make(map[string]time.Time)
        }
        for nodeID, ts := range entries {
            s.removed[item][nodeID] = ts
        }
    }
}

func (s *ORSet) Contains(item interface{}) bool {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    added, ok := s.added[item]
    if !ok {
        return false
    }
    
    removed := s.removed[item]
    
    for nodeID, addedAt := range added {
        removedAt, wasRemoved := removed[nodeID]
        if !wasRemoved || addedAt.After(removedAt) {
            return true
        }
    }
    
    return false
}

func (s *ORSet) Merge(other *ORSet) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    other.mu.RLock()
    defer other.mu.RUnlock()
    
    for item, entries := range other.added {
        if _, ok := s.added[item]; !ok {
            s.added[item] = make(map[string]time.Time)
        }
        for nodeID, ts := range entries {
            if existing, ok := s.added[item][nodeID]; !ok || ts.After(existing) {
                s.added[item][nodeID] = ts
            }
        }
    }
    
    for item, entries := range other.removed {
        if _, ok := s.removed[item]; !ok {
            s.removed[item] = make(map[string]time.Time)
        }
        for nodeID, ts := range entries {
            if existing, ok := s.removed[item][nodeID]; !ok || ts.After(existing) {
                s.removed[item][nodeID] = ts
            }
        }
    }
}
```

---

## Workshop: Real-time Collaboration System

```go
// workshop/collab/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
)

type CollabDocument struct {
    ID       string
    Content  string
    Version  int
    mu       sync.RWMutex
    clients  map[string]*CollabClient
    history  []DocumentOperation
}

type DocumentOperation struct {
    Version    int        `json:"version"`
    Operations []interface{} `json:"operations"`
    UserID     string     `json:"user_id"`
    Timestamp  int64      `json:"timestamp"`
}

type CollabClient struct {
    ID      string
    UserID  string
    doc     *CollabDocument
    send    chan []byte
}

type CollabServer struct {
    docs    map[string]*CollabDocument
    hub     *Hub
    nats    *NATSClient
    mu      sync.RWMutex
}

func NewCollabServer(natsURL string) (*CollabServer, error) {
    natsClient, err := NewNATSClient(natsURL)
    if err != nil {
        return nil, err
    }
    
    srv := &CollabServer{
        docs:  make(map[string]*CollabDocument),
        nats:  natsClient,
    }
    
    return srv, nil
}

func (s *CollabServer) GetOrCreateDoc(docID string) *CollabDocument {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if doc, ok := s.docs[docID]; ok {
        return doc
    }
    
    doc := &CollabDocument{
        ID:      docID,
        Content: "",
        Version: 0,
        clients: make(map[string]*CollabClient),
    }
    
    s.docs[docID] = doc
    return doc
}

func (s *CollabServer) HandleOperation(w http.ResponseWriter, r *http.Request) {
    docID := r.URL.Query().Get("doc_id")
    
    var op DocumentOperation
    if err := json.NewDecoder(r.Body).Decode(&op); err != nil {
        http.Error(w, "Invalid operation", http.StatusBadRequest)
        return
    }
    
    doc := s.GetOrCreateDoc(docID)
    
    doc.mu.Lock()
    defer doc.mu.Unlock()
    
    // Transform operation ถ้า version ไม่ตรง
    if op.Version < doc.Version {
        // Transform against operations since op.Version
        for _, historical := range doc.history[op.Version:] {
            // Transform op.Operations against historical.Operations
            _ = historical
        }
    }
    
    // Apply operation
    doc.Version++
    op.Version = doc.Version
    doc.history = append(doc.history, op)
    
    // Broadcast ไปยัง clients ใน room ผ่าน NATS
    data, _ := json.Marshal(op)
    s.nats.Publish("doc."+docID, data)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]int{
        "version": doc.Version,
    })
}

func main() {
    srv, err := NewCollabServer("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    
    mux := http.NewServeMux()
    mux.HandleFunc("/op", srv.HandleOperation)
    mux.HandleFunc("/ws", srv.hub.ServeWS)
    
    fmt.Println("Collab server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## สรุป

| Technology | Use Case | Latency | Scale |
|-----------|---------|---------|-------|
| WebSocket | Bidirectional, chat | ~5ms | High (กับ Redis) |
| SSE | Server→Client streams | ~5ms | Very high |
| Long Polling | Simple notifications | ~100ms | Medium |
| NATS | Microservice messaging | ~1ms | Very high |
| Kafka | Event streaming | ~10ms | Highest |

### เลือก Pattern ไหนดี?
- **WebSocket**: Chat, gaming, collaborative tools
- **SSE**: Dashboard, news feeds, notifications
- **Long Polling**: Legacy support, firewalls, simple notifications
- **NATS**: Service-to-service real-time messaging

---

*จบ Part 73: Real-time Systems*
