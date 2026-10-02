# Part 44: WebSockets ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ WebSocket protocol
- สร้าง WebSocket server และ client ด้วย gorilla/websocket
- สร้าง Broadcasting system
- จัดการ Rooms/Groups
- ใช้ Ping/Pong heartbeat
- ทำ Authentication ใน WebSocket
- สร้าง Real-time chat (Workshop)

---

## 1. WebSocket Overview

WebSocket เป็น protocol ที่ให้ full-duplex communication channel ผ่าน TCP connection เดียว

**เปรียบเทียบกับ HTTP:**
| Feature | HTTP | WebSocket |
|---------|------|-----------|
| Connection | Request-Response | Persistent |
| Direction | Unidirectional | Bidirectional |
| Overhead | Headers every request | Once at handshake |
| Use case | REST API | Real-time apps |

```bash
# Install gorilla/websocket
go get github.com/gorilla/websocket
```

---

## 2. Basic WebSocket Server

```go
// ตัวอย่าง 1: Basic WebSocket server

package main

import (
	"fmt"
	"log"
	"net/http"
	
	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		// In production, validate origin
		return true
	},
}

func echoHandler(w http.ResponseWriter, r *http.Request) {
	// Upgrade HTTP to WebSocket
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Printf("Upgrade error: %v", err)
		return
	}
	defer conn.Close()
	
	fmt.Println("Client connected:", conn.RemoteAddr())
	
	// Echo loop
	for {
		messageType, message, err := conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("Read error: %v", err)
			}
			break
		}
		
		fmt.Printf("Received (%d): %s\n", messageType, message)
		
		// Echo back
		if err := conn.WriteMessage(messageType, message); err != nil {
			log.Printf("Write error: %v", err)
			break
		}
	}
	
	fmt.Println("Client disconnected:", conn.RemoteAddr())
}

func main() {
	http.HandleFunc("/ws", echoHandler)
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		http.ServeFile(w, r, "index.html")
	})
	
	fmt.Println("WebSocket server running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 3. WebSocket Client

```go
// ตัวอย่าง 2: WebSocket client

package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
	
	"github.com/gorilla/websocket"
)

func wsClient() {
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt, syscall.SIGTERM)
	
	// Connect to server
	conn, _, err := websocket.DefaultDialer.Dial("ws://localhost:8080/ws", nil)
	if err != nil {
		log.Fatalf("Dial error: %v", err)
	}
	defer conn.Close()
	
	fmt.Println("Connected to WebSocket server")
	
	done := make(chan struct{})
	
	// Receive goroutine
	go func() {
		defer close(done)
		for {
			messageType, message, err := conn.ReadMessage()
			if err != nil {
				log.Println("Read error:", err)
				return
			}
			fmt.Printf("Received (%d): %s\n", messageType, message)
		}
	}()
	
	// Send messages
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	
	for {
		select {
		case <-done:
			return
		case t := <-ticker.C:
			msg := fmt.Sprintf("Hello at %v", t.Format("15:04:05"))
			err := conn.WriteMessage(websocket.TextMessage, []byte(msg))
			if err != nil {
				log.Println("Write error:", err)
				return
			}
			fmt.Printf("Sent: %s\n", msg)
		case <-interrupt:
			fmt.Println("Closing connection...")
			// Send close message
			err := conn.WriteMessage(websocket.CloseMessage,
				websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
			if err != nil {
				log.Println("Close error:", err)
				return
			}
			select {
			case <-done:
			case <-time.After(time.Second):
			}
			return
		}
	}
}

// ตัวอย่าง 3: Client with reconnection
type ReconnectingClient struct {
	url           string
	conn          *websocket.Conn
	reconnectWait time.Duration
	maxReconnects int
	
	onMessage func([]byte)
	onConnect func()
	onDisconnect func(error)
}

func NewReconnectingClient(url string) *ReconnectingClient {
	return &ReconnectingClient{
		url:           url,
		reconnectWait: 5 * time.Second,
		maxReconnects: 10,
	}
}

func (c *ReconnectingClient) Connect() error {
	attempts := 0
	
	for {
		conn, _, err := websocket.DefaultDialer.Dial(c.url, nil)
		if err != nil {
			attempts++
			if attempts > c.maxReconnects {
				return fmt.Errorf("max reconnects exceeded: %w", err)
			}
			fmt.Printf("Connection failed, retrying in %v (attempt %d/%d)\n",
				c.reconnectWait, attempts, c.maxReconnects)
			time.Sleep(c.reconnectWait)
			continue
		}
		
		c.conn = conn
		attempts = 0
		
		if c.onConnect != nil {
			c.onConnect()
		}
		
		// Read loop
		err = c.readLoop()
		c.conn.Close()
		
		if c.onDisconnect != nil {
			c.onDisconnect(err)
		}
		
		if err == nil {
			return nil // graceful close
		}
		
		fmt.Printf("Disconnected: %v, reconnecting...\n", err)
		time.Sleep(c.reconnectWait)
	}
}

func (c *ReconnectingClient) readLoop() error {
	for {
		_, message, err := c.conn.ReadMessage()
		if err != nil {
			return err
		}
		if c.onMessage != nil {
			c.onMessage(message)
		}
	}
}

func (c *ReconnectingClient) Send(message []byte) error {
	if c.conn == nil {
		return fmt.Errorf("not connected")
	}
	return c.conn.WriteMessage(websocket.TextMessage, message)
}

func main() {
	wsClient()
}
```

---

## 4. Broadcasting Messages

```go
// ตัวอย่าง 4: Hub pattern for broadcasting

package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
	
	"github.com/gorilla/websocket"
)

// Client represents a WebSocket connection
type Client struct {
	hub  *Hub
	conn *websocket.Conn
	send chan []byte
	id   string
}

const (
	writeWait      = 10 * time.Second
	pongWait       = 60 * time.Second
	pingPeriod     = (pongWait * 9) / 10
	maxMessageSize = 512
)

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
		_, message, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("error: %v", err)
			}
			break
		}
		c.hub.broadcast <- message
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
			
			// Batch queued messages
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
		}
	}
}

// Hub maintains active connections and broadcasts messages
type Hub struct {
	clients    map[*Client]bool
	broadcast  chan []byte
	register   chan *Client
	unregister chan *Client
	mu         sync.RWMutex
}

func NewHub() *Hub {
	return &Hub{
		broadcast:  make(chan []byte),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		clients:    make(map[*Client]bool),
	}
}

func (h *Hub) Run() {
	for {
		select {
		case client := <-h.register:
			h.mu.Lock()
			h.clients[client] = true
			h.mu.Unlock()
			fmt.Printf("Client %s connected (total: %d)\n", client.id, len(h.clients))
			
		case client := <-h.unregister:
			h.mu.Lock()
			if _, ok := h.clients[client]; ok {
				delete(h.clients, client)
				close(client.send)
			}
			h.mu.Unlock()
			fmt.Printf("Client %s disconnected (total: %d)\n", client.id, len(h.clients))
			
		case message := <-h.broadcast:
			h.mu.RLock()
			for client := range h.clients {
				select {
				case client.send <- message:
				default:
					close(client.send)
					delete(h.clients, client)
				}
			}
			h.mu.RUnlock()
		}
	}
}

func (h *Hub) ClientCount() int {
	h.mu.RLock()
	defer h.mu.RUnlock()
	return len(h.clients)
}

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}
	
	clientID := fmt.Sprintf("client-%d", time.Now().UnixNano())
	client := &Client{
		hub:  hub,
		conn: conn,
		send: make(chan []byte, 256),
		id:   clientID,
	}
	client.hub.register <- client
	
	go client.writePump()
	go client.readPump()
}

func broadcastDemo() {
	hub := NewHub()
	go hub.Run()
	
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWs(hub, w, r)
	})
	
	// Broadcast server time every second
	go func() {
		ticker := time.NewTicker(time.Second)
		for t := range ticker.C {
			msg := fmt.Sprintf(`{"type":"time","data":"%s","clients":%d}`,
				t.Format("15:04:05"), hub.ClientCount())
			hub.broadcast <- []byte(msg)
		}
	}()
	
	fmt.Println("Server with broadcasting running on :8080")
	// log.Fatal(http.ListenAndServe(":8080", nil))
}

func main() {
	broadcastDemo()
}
```

---

## 5. Rooms and Groups

```go
// ตัวอย่าง 5: Room-based WebSocket server

package main

import (
	"encoding/json"
	"fmt"
	"sync"
	"time"
	
	"github.com/gorilla/websocket"
)

type RoomMessage struct {
	Type    string          `json:"type"`
	Room    string          `json:"room,omitempty"`
	User    string          `json:"user,omitempty"`
	Content json.RawMessage `json:"content,omitempty"`
}

// RoomClient represents a client in a room
type RoomClient struct {
	id    string
	name  string
	conn  *websocket.Conn
	rooms map[string]bool
	send  chan []byte
	hub   *RoomHub
}

// RoomHub manages rooms
type RoomHub struct {
	mu         sync.RWMutex
	clients    map[string]*RoomClient    // clientID -> client
	rooms      map[string][]*RoomClient  // roomName -> clients
	
	join       chan joinRequest
	leave      chan leaveRequest
	broadcast  chan broadcastMsg
	register   chan *RoomClient
	unregister chan *RoomClient
}

type joinRequest struct {
	client *RoomClient
	room   string
}

type leaveRequest struct {
	client *RoomClient
	room   string
}

type broadcastMsg struct {
	room    string
	sender  string
	message []byte
}

func NewRoomHub() *RoomHub {
	return &RoomHub{
		clients:    make(map[string]*RoomClient),
		rooms:      make(map[string][]*RoomClient),
		join:       make(chan joinRequest),
		leave:      make(chan leaveRequest),
		broadcast:  make(chan broadcastMsg),
		register:   make(chan *RoomClient),
		unregister: make(chan *RoomClient),
	}
}

func (h *RoomHub) Run() {
	for {
		select {
		case client := <-h.register:
			h.mu.Lock()
			h.clients[client.id] = client
			h.mu.Unlock()
			
		case client := <-h.unregister:
			h.mu.Lock()
			// Remove from all rooms
			for room := range client.rooms {
				h.removeFromRoom(client, room)
			}
			delete(h.clients, client.id)
			close(client.send)
			h.mu.Unlock()
			
		case req := <-h.join:
			h.mu.Lock()
			h.rooms[req.room] = append(h.rooms[req.room], req.client)
			req.client.rooms[req.room] = true
			h.mu.Unlock()
			
			// Notify room members
			notification, _ := json.Marshal(RoomMessage{
				Type: "user_joined",
				Room: req.room,
				User: req.client.name,
			})
			h.broadcastToRoom(req.room, req.client.id, notification)
			
		case req := <-h.leave:
			h.mu.Lock()
			h.removeFromRoom(req.client, req.room)
			h.mu.Unlock()
			
			// Notify room members
			notification, _ := json.Marshal(RoomMessage{
				Type: "user_left",
				Room: req.room,
				User: req.client.name,
			})
			h.broadcastToRoom(req.room, req.client.id, notification)
			
		case msg := <-h.broadcast:
			h.mu.RLock()
			h.broadcastToRoom(msg.room, msg.sender, msg.message)
			h.mu.RUnlock()
		}
	}
}

func (h *RoomHub) removeFromRoom(client *RoomClient, room string) {
	clients := h.rooms[room]
	for i, c := range clients {
		if c.id == client.id {
			h.rooms[room] = append(clients[:i], clients[i+1:]...)
			break
		}
	}
	delete(client.rooms, room)
	if len(h.rooms[room]) == 0 {
		delete(h.rooms, room)
	}
}

func (h *RoomHub) broadcastToRoom(room, senderID string, message []byte) {
	for _, client := range h.rooms[room] {
		if client.id != senderID {
			select {
			case client.send <- message:
			default:
				close(client.send)
				delete(h.clients, client.id)
			}
		}
	}
}

func (h *RoomHub) JoinRoom(client *RoomClient, room string) {
	h.join <- joinRequest{client: client, room: room}
}

func (h *RoomHub) LeaveRoom(client *RoomClient, room string) {
	h.leave <- leaveRequest{client: client, room: room}
}

func (h *RoomHub) SendToRoom(room, senderID string, msg []byte) {
	h.broadcast <- broadcastMsg{room: room, sender: senderID, message: msg}
}

func (h *RoomHub) RoomList() []string {
	h.mu.RLock()
	defer h.mu.RUnlock()
	rooms := make([]string, 0, len(h.rooms))
	for room := range h.rooms {
		rooms = append(rooms, room)
	}
	return rooms
}

func (h *RoomHub) RoomMemberCount(room string) int {
	h.mu.RLock()
	defer h.mu.RUnlock()
	return len(h.rooms[room])
}

func roomDemo() {
	fmt.Println("=== Room-based WebSocket Demo ===")
	
	hub := NewRoomHub()
	go hub.Run()
	
	// Simulate clients
	// In real code, clients connect via HTTP upgrade
	
	time.Sleep(100 * time.Millisecond)
	
	fmt.Println("Room management setup complete")
	fmt.Println("Clients can join/leave specific rooms")
	fmt.Println("Messages are isolated to rooms")
}

func main() {
	roomDemo()
}
```

---

## 6. Ping/Pong Heartbeat

```go
// ตัวอย่าง 6: Ping/Pong heartbeat mechanism

package main

import (
	"fmt"
	"sync/atomic"
	"time"
	
	"github.com/gorilla/websocket"
)

type HeartbeatConn struct {
	conn     *websocket.Conn
	alive    atomic.Bool
	latency  atomic.Int64 // microseconds
	
	pingInterval time.Duration
	pongTimeout  time.Duration
	
	done chan struct{}
}

func NewHeartbeatConn(conn *websocket.Conn, pingInterval time.Duration) *HeartbeatConn {
	hc := &HeartbeatConn{
		conn:         conn,
		pingInterval: pingInterval,
		pongTimeout:  pingInterval * 2,
		done:         make(chan struct{}),
	}
	hc.alive.Store(true)
	
	go hc.heartbeat()
	return hc
}

func (hc *HeartbeatConn) heartbeat() {
	ticker := time.NewTicker(hc.pingInterval)
	defer ticker.Stop()
	
	hc.conn.SetPongHandler(func(appData string) error {
		// Calculate latency
		if appData != "" {
			var pingTime int64
			fmt.Sscanf(appData, "%d", &pingTime)
			latency := time.Now().UnixMicro() - pingTime
			hc.latency.Store(latency)
		}
		
		hc.conn.SetReadDeadline(time.Now().Add(hc.pongTimeout))
		return nil
	})
	
	for {
		select {
		case <-hc.done:
			return
		case <-ticker.C:
			pingTime := fmt.Sprintf("%d", time.Now().UnixMicro())
			
			hc.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
			if err := hc.conn.WriteMessage(websocket.PingMessage, []byte(pingTime)); err != nil {
				fmt.Printf("Ping failed: %v\n", err)
				hc.alive.Store(false)
				return
			}
		}
	}
}

func (hc *HeartbeatConn) Latency() time.Duration {
	return time.Duration(hc.latency.Load()) * time.Microsecond
}

func (hc *HeartbeatConn) IsAlive() bool {
	return hc.alive.Load()
}

func (hc *HeartbeatConn) Close() {
	close(hc.done)
	hc.conn.Close()
}

// ตัวอย่าง 7: Connection health monitoring
type ConnectionMonitor struct {
	conns   map[string]*HeartbeatConn
	mu      sync.RWMutex
	
	onDead func(clientID string)
}

func NewConnectionMonitor(onDead func(clientID string)) *ConnectionMonitor {
	cm := &ConnectionMonitor{
		conns:  make(map[string]*HeartbeatConn),
		onDead: onDead,
	}
	go cm.monitor()
	return cm
}

func (cm *ConnectionMonitor) Add(clientID string, conn *HeartbeatConn) {
	cm.mu.Lock()
	defer cm.mu.Unlock()
	cm.conns[clientID] = conn
}

func (cm *ConnectionMonitor) Remove(clientID string) {
	cm.mu.Lock()
	defer cm.mu.Unlock()
	delete(cm.conns, clientID)
}

func (cm *ConnectionMonitor) monitor() {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()
	
	for range ticker.C {
		cm.mu.Lock()
		deadClients := make([]string, 0)
		
		for id, conn := range cm.conns {
			if !conn.IsAlive() {
				deadClients = append(deadClients, id)
			}
		}
		
		for _, id := range deadClients {
			delete(cm.conns, id)
			if cm.onDead != nil {
				go cm.onDead(id)
			}
		}
		cm.mu.Unlock()
	}
}

import "sync"

func heartbeatDemo() {
	fmt.Println("=== Ping/Pong Heartbeat Demo ===")
	fmt.Println("Heartbeat keeps WebSocket connections alive")
	fmt.Println("Detects dead connections without sending data")
	fmt.Println()
	fmt.Println("Configuration:")
	fmt.Printf("  Ping interval: %v\n", 30*time.Second)
	fmt.Printf("  Pong timeout:  %v\n", 60*time.Second)
	fmt.Printf("  Write timeout: %v\n", 10*time.Second)
}

func main() {
	heartbeatDemo()
}
```

---

## 7. Authentication in WebSocket

```go
// ตัวอย่าง 8: JWT Authentication in WebSocket

package main

import (
	"fmt"
	"net/http"
	"strings"
	"time"
	
	"github.com/golang-jwt/jwt/v5"
	"github.com/gorilla/websocket"
)

type Claims struct {
	UserID   int    `json:"user_id"`
	UserName string `json:"user_name"`
	Role     string `json:"role"`
	jwt.RegisteredClaims
}

var jwtSecret = []byte("your-secret-key")

func generateToken(userID int, userName, role string) (string, error) {
	claims := Claims{
		UserID:   userID,
		UserName: userName,
		Role:     role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}
	
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

func validateToken(tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return jwtSecret, nil
	})
	
	if err != nil {
		return nil, err
	}
	
	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	
	return claims, nil
}

var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

// Method 1: Token in query parameter (less secure)
func authenticatedWsHandlerQuery(w http.ResponseWriter, r *http.Request) {
	tokenStr := r.URL.Query().Get("token")
	if tokenStr == "" {
		http.Error(w, "missing token", http.StatusUnauthorized)
		return
	}
	
	claims, err := validateToken(tokenStr)
	if err != nil {
		http.Error(w, "invalid token", http.StatusUnauthorized)
		return
	}
	
	conn, err := wsUpgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()
	
	fmt.Printf("User %s (role=%s) connected via query token\n",
		claims.UserName, claims.Role)
	
	// Handle connection...
}

// Method 2: Token in Authorization header (more secure)
func authenticatedWsHandlerHeader(w http.ResponseWriter, r *http.Request) {
	authHeader := r.Header.Get("Authorization")
	
	if !strings.HasPrefix(authHeader, "Bearer ") {
		http.Error(w, "missing or invalid authorization header", http.StatusUnauthorized)
		return
	}
	
	tokenStr := strings.TrimPrefix(authHeader, "Bearer ")
	claims, err := validateToken(tokenStr)
	if err != nil {
		http.Error(w, "invalid token", http.StatusUnauthorized)
		return
	}
	
	conn, err := wsUpgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()
	
	fmt.Printf("User %s connected via header token\n", claims.UserName)
}

// Method 3: Initial message authentication
func authenticatedWsHandlerMessage(w http.ResponseWriter, r *http.Request) {
	conn, err := wsUpgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()
	
	// Set short timeout for auth message
	conn.SetReadDeadline(time.Now().Add(10 * time.Second))
	
	// Wait for auth message
	_, message, err := conn.ReadMessage()
	if err != nil {
		conn.WriteMessage(websocket.CloseMessage,
			websocket.FormatCloseMessage(websocket.CloseProtocolError, "auth timeout"))
		return
	}
	
	// Reset deadline after auth
	conn.SetReadDeadline(time.Time{})
	
	claims, err := validateToken(string(message))
	if err != nil {
		conn.WriteMessage(websocket.CloseMessage,
			websocket.FormatCloseMessage(websocket.CloseProtocolError, "invalid token"))
		return
	}
	
	// Send auth success
	conn.WriteMessage(websocket.TextMessage, []byte(`{"type":"auth","status":"ok"}`))
	
	fmt.Printf("User %s authenticated via message\n", claims.UserName)
	
	// Handle connection normally from here
}

func authDemo() {
	fmt.Println("=== WebSocket Authentication Demo ===")
	
	// Generate token for demo
	token, err := generateToken(1, "alice", "admin")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Generated token: %s...\n", token[:30])
	
	// Validate token
	claims, err := validateToken(token)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	
	fmt.Printf("Validated: user=%s, role=%s\n", claims.UserName, claims.Role)
	fmt.Println()
	fmt.Println("Authentication methods:")
	fmt.Println("1. Query param: ws://host/ws?token=<jwt>")
	fmt.Println("2. Header: Authorization: Bearer <jwt>")
	fmt.Println("3. First message: send token as first WebSocket message")
}

func main() {
	authDemo()
}
```

---

## Workshop: Real-time Chat Room

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
	
	"github.com/gorilla/websocket"
)

// Workshop: Complete Chat Room Implementation

type MessageType string

const (
	MsgTypeChat      MessageType = "chat"
	MsgTypeJoin      MessageType = "join"
	MsgTypeLeave     MessageType = "leave"
	MsgTypeUserList  MessageType = "user_list"
	MsgTypeError     MessageType = "error"
	MsgTypeSystem    MessageType = "system"
)

type ChatMsg struct {
	Type      MessageType `json:"type"`
	Room      string      `json:"room,omitempty"`
	From      string      `json:"from,omitempty"`
	Content   string      `json:"content,omitempty"`
	Timestamp time.Time   `json:"timestamp"`
	Users     []string    `json:"users,omitempty"`
}

// ChatClient manages a WebSocket connection
type ChatClient struct {
	id   string
	name string
	conn *websocket.Conn
	send chan []byte
	hub  *ChatHub
	
	rooms map[string]bool
	mu    sync.Mutex
}

func newChatClient(id, name string, conn *websocket.Conn, hub *ChatHub) *ChatClient {
	return &ChatClient{
		id:    id,
		name:  name,
		conn:  conn,
		send:  make(chan []byte, 256),
		hub:   hub,
		rooms: make(map[string]bool),
	}
}

func (c *ChatClient) readPump() {
	defer func() {
		c.hub.unregister <- c
		c.conn.Close()
	}()
	
	c.conn.SetReadLimit(4096)
	c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
		return nil
	})
	
	for {
		_, msg, err := c.conn.ReadMessage()
		if err != nil {
			break
		}
		
		var chatMsg ChatMsg
		if err := json.Unmarshal(msg, &chatMsg); err != nil {
			c.sendError("Invalid message format")
			continue
		}
		
		chatMsg.From = c.name
		chatMsg.Timestamp = time.Now()
		
		c.hub.handleMessage(c, &chatMsg)
	}
}

func (c *ChatClient) writePump() {
	ticker := time.NewTicker(54 * time.Second)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()
	
	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
				return
			}
		case <-ticker.C:
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}

func (c *ChatClient) sendMsg(msg *ChatMsg) {
	data, _ := json.Marshal(msg)
	select {
	case c.send <- data:
	default:
		close(c.send)
	}
}

func (c *ChatClient) sendError(errMsg string) {
	c.sendMsg(&ChatMsg{
		Type:      MsgTypeError,
		Content:   errMsg,
		Timestamp: time.Now(),
	})
}

// ChatHub manages all chat rooms and clients
type ChatHub struct {
	mu      sync.RWMutex
	clients map[string]*ChatClient         // clientID -> client
	rooms   map[string]map[string]*ChatClient // roomName -> clientID -> client
	
	register   chan *ChatClient
	unregister chan *ChatClient
}

func NewChatHub() *ChatHub {
	return &ChatHub{
		clients:    make(map[string]*ChatClient),
		rooms:      make(map[string]map[string]*ChatClient),
		register:   make(chan *ChatClient),
		unregister: make(chan *ChatClient),
	}
}

func (h *ChatHub) Run() {
	for {
		select {
		case client := <-h.register:
			h.mu.Lock()
			h.clients[client.id] = client
			h.mu.Unlock()
			
			// Send welcome message
			client.sendMsg(&ChatMsg{
				Type:      MsgTypeSystem,
				Content:   fmt.Sprintf("Welcome, %s! Use 'join <room>' to join a room.", client.name),
				Timestamp: time.Now(),
			})
			
			fmt.Printf("[+] %s connected (total: %d)\n", client.name, len(h.clients))
			
		case client := <-h.unregister:
			h.mu.Lock()
			// Leave all rooms
			for room := range client.rooms {
				h.removeFromRoom(client, room)
			}
			delete(h.clients, client.id)
			h.mu.Unlock()
			
			fmt.Printf("[-] %s disconnected\n", client.name)
		}
	}
}

func (h *ChatHub) handleMessage(client *ChatClient, msg *ChatMsg) {
	switch msg.Type {
	case MsgTypeJoin:
		h.joinRoom(client, msg.Room)
	case MsgTypeLeave:
		h.leaveRoom(client, msg.Room)
	case MsgTypeChat:
		h.sendToRoom(client, msg)
	}
}

func (h *ChatHub) joinRoom(client *ChatClient, room string) {
	if room == "" {
		client.sendError("Room name required")
		return
	}
	
	h.mu.Lock()
	if h.rooms[room] == nil {
		h.rooms[room] = make(map[string]*ChatClient)
	}
	h.rooms[room][client.id] = client
	client.rooms[room] = true
	
	// Get user list
	users := make([]string, 0, len(h.rooms[room]))
	for _, c := range h.rooms[room] {
		users = append(users, c.name)
	}
	h.mu.Unlock()
	
	// Send join confirmation
	client.sendMsg(&ChatMsg{
		Type:      MsgTypeJoin,
		Room:      room,
		Users:     users,
		Content:   fmt.Sprintf("Joined room: %s", room),
		Timestamp: time.Now(),
	})
	
	// Notify others
	h.broadcastToRoom(room, client.id, &ChatMsg{
		Type:      MsgTypeSystem,
		Room:      room,
		Content:   fmt.Sprintf("%s joined the room", client.name),
		Timestamp: time.Now(),
	})
	
	fmt.Printf("[Room:%s] %s joined\n", room, client.name)
}

func (h *ChatHub) leaveRoom(client *ChatClient, room string) {
	h.mu.Lock()
	h.removeFromRoom(client, room)
	h.mu.Unlock()
	
	client.sendMsg(&ChatMsg{
		Type:      MsgTypeLeave,
		Room:      room,
		Content:   fmt.Sprintf("Left room: %s", room),
		Timestamp: time.Now(),
	})
	
	h.broadcastToRoom(room, client.id, &ChatMsg{
		Type:      MsgTypeSystem,
		Room:      room,
		Content:   fmt.Sprintf("%s left the room", client.name),
		Timestamp: time.Now(),
	})
}

func (h *ChatHub) removeFromRoom(client *ChatClient, room string) {
	if h.rooms[room] != nil {
		delete(h.rooms[room], client.id)
		if len(h.rooms[room]) == 0 {
			delete(h.rooms, room)
		}
	}
	delete(client.rooms, room)
}

func (h *ChatHub) sendToRoom(sender *ChatClient, msg *ChatMsg) {
	if msg.Room == "" {
		sender.sendError("Room required for chat message")
		return
	}
	
	h.mu.RLock()
	_, inRoom := sender.rooms[msg.Room]
	h.mu.RUnlock()
	
	if !inRoom {
		sender.sendError(fmt.Sprintf("You are not in room: %s", msg.Room))
		return
	}
	
	h.broadcastToRoom(msg.Room, "", msg)
	fmt.Printf("[Room:%s] %s: %s\n", msg.Room, msg.From, msg.Content)
}

func (h *ChatHub) broadcastToRoom(room, excludeID string, msg *ChatMsg) {
	h.mu.RLock()
	defer h.mu.RUnlock()
	
	clients := h.rooms[room]
	for _, client := range clients {
		if client.id != excludeID {
			client.sendMsg(msg)
		}
	}
}

var chatUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func setupChatServer() {
	hub := NewChatHub()
	go hub.Run()
	
	clientCounter := 0
	
	http.HandleFunc("/ws/chat", func(w http.ResponseWriter, r *http.Request) {
		conn, err := chatUpgrader.Upgrade(w, r, nil)
		if err != nil {
			log.Println(err)
			return
		}
		
		clientCounter++
		clientID := fmt.Sprintf("client-%d", clientCounter)
		userName := r.URL.Query().Get("name")
		if userName == "" {
			userName = fmt.Sprintf("User%d", clientCounter)
		}
		
		client := newChatClient(clientID, userName, conn, hub)
		hub.register <- client
		
		go client.writePump()
		go client.readPump()
	})
	
	// Chat statistics endpoint
	http.HandleFunc("/stats", func(w http.ResponseWriter, r *http.Request) {
		hub.mu.RLock()
		stats := map[string]interface{}{
			"total_clients": len(hub.clients),
			"total_rooms":   len(hub.rooms),
		}
		hub.mu.RUnlock()
		
		json.NewEncoder(w).Encode(stats)
	})
	
	fmt.Println("Chat server running on :8080")
	fmt.Println("Connect: ws://localhost:8080/ws/chat?name=YourName")
	fmt.Println("Stats: http://localhost:8080/stats")
}

// Demo without actual WebSocket connection
func chatWorkshopDemo() {
	fmt.Println("=== Chat Room Workshop ===")
	
	hub := NewChatHub()
	go hub.Run()
	
	time.Sleep(50 * time.Millisecond)
	
	// Create mock clients (without real WebSocket)
	client1 := &ChatClient{
		id: "c1", name: "Alice",
		rooms: make(map[string]bool),
		send:  make(chan []byte, 256),
		hub:   hub,
	}
	client2 := &ChatClient{
		id: "c2", name: "Bob",
		rooms: make(map[string]bool),
		send:  make(chan []byte, 256),
		hub:   hub,
	}
	
	// Register clients
	hub.register <- client1
	hub.register <- client2
	time.Sleep(50 * time.Millisecond)
	
	// Join room
	hub.joinRoom(client1, "general")
	hub.joinRoom(client2, "general")
	time.Sleep(50 * time.Millisecond)
	
	// Send chat message
	hub.sendToRoom(client1, &ChatMsg{
		Type:      MsgTypeChat,
		Room:      "general",
		From:      "Alice",
		Content:   "Hello everyone!",
		Timestamp: time.Now(),
	})
	
	hub.sendToRoom(client2, &ChatMsg{
		Type:      MsgTypeChat,
		Room:      "general",
		From:      "Bob",
		Content:   "Hi Alice!",
		Timestamp: time.Now(),
	})
	
	time.Sleep(50 * time.Millisecond)
	
	// Drain messages for demo
	fmt.Println("\nMessages received by Alice:")
	for {
		select {
		case msg := <-client1.send:
			var chatMsg ChatMsg
			json.Unmarshal(msg, &chatMsg)
			fmt.Printf("  [%s] %s: %s\n", chatMsg.Type, chatMsg.From, chatMsg.Content)
		default:
			goto done
		}
	}
done:
	
	fmt.Printf("\nHub stats: clients=%d, rooms=%d\n",
		len(hub.clients), len(hub.rooms))
}

func main() {
	chatWorkshopDemo()
}
```

---

## สรุป

ใน Part 44 เราได้เรียนรู้:

1. **WebSocket Overview**: full-duplex communication over HTTP
2. **Basic Server**: upgrading HTTP to WebSocket
3. **WebSocket Client**: connecting and sending/receiving
4. **Reconnecting Client**: automatic reconnection on disconnect
5. **Broadcasting**: Hub pattern for sending to all clients
6. **Rooms/Groups**: isolated messaging spaces
7. **Ping/Pong Heartbeat**: keep connections alive, detect dead ones
8. **Authentication**: JWT via query param, header, or first message
9. **Workshop**: Complete real-time chat room implementation

---

## Resources

- [gorilla/websocket](https://github.com/gorilla/websocket)
- [WebSocket RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Gorilla WebSocket Chat Example](https://github.com/gorilla/websocket/tree/main/examples/chat)
- [WebSocket Security](https://portswigger.net/web-security/websockets)
