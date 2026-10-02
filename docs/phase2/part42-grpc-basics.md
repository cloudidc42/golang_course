# Part 42: gRPC Basics ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ gRPC และความแตกต่างจาก REST
- ใช้ Protocol Buffers กับ Go
- สร้าง gRPC server และ client
- ใช้ Unary, Server Streaming, Client Streaming, Bidirectional Streaming
- จัดการ Error Handling ใน gRPC

---

## 1. gRPC Overview

gRPC คือ framework สำหรับ Remote Procedure Call (RPC) ที่พัฒนาโดย Google
- ใช้ HTTP/2 (เร็วกว่า HTTP/1.1)
- ใช้ Protocol Buffers (compact binary format)
- รองรับ streaming
- Type-safe interface
- Code generation

**เปรียบเทียบ gRPC vs REST:**

| Feature | REST | gRPC |
|---------|------|------|
| Protocol | HTTP/1.1 | HTTP/2 |
| Format | JSON (text) | Protobuf (binary) |
| Streaming | Limited | Full support |
| Type safety | None (manual) | Generated code |
| Performance | Good | Excellent |
| Browser support | ✓ | Limited |

---

## 2. Setup

```bash
# Install protoc compiler
# macOS: brew install protobuf
# Linux: apt-get install protobuf-compiler

# Install Go protobuf plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Install gRPC packages
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

---

## 3. Proto File Definition

```protobuf
// proto/hello/hello.proto
syntax = "proto3";

package hello;

option go_package = "github.com/yourorg/app/proto/hello";

// ตัวอย่าง 1: Simple message types
message HelloRequest {
  string name = 1;
  string language = 2;
}

message HelloResponse {
  string greeting = 1;
  int64 timestamp = 2;
}

// ตัวอย่าง 2: User service proto
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  repeated string roles = 4;
  UserStatus status = 5;
  
  enum UserStatus {
    UNKNOWN = 0;
    ACTIVE = 1;
    INACTIVE = 2;
    BANNED = 3;
  }
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  int32 id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string search = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  int32 page = 3;
}

// ตัวอย่าง 3: Service definition
service HelloService {
  // Unary RPC
  rpc SayHello(HelloRequest) returns (HelloResponse);
}

service UserService {
  // Unary RPCs
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  
  // Server streaming - stream all updates
  rpc WatchUser(GetUserRequest) returns (stream User);
  
  // Client streaming - bulk create
  rpc BulkCreateUsers(stream CreateUserRequest) returns (CreateUserResponse);
  
  // Bidirectional streaming - chat
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string user = 1;
  string text = 2;
  int64 timestamp = 3;
}
```

```bash
# Generate Go code
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/hello/hello.proto
```

---

## 4. Unary RPC

```go
// ตัวอย่าง 4: Hello Service - Server implementation

package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"time"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	
	pb "github.com/yourorg/app/proto/hello"
)

// Server implementation
type helloServer struct {
	pb.UnimplementedHelloServiceServer
}

func (s *helloServer) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	// Validate input
	if req.Name == "" {
		return nil, status.Error(codes.InvalidArgument, "name cannot be empty")
	}
	
	var greeting string
	switch req.Language {
	case "th":
		greeting = fmt.Sprintf("สวัสดี, %s!", req.Name)
	case "ja":
		greeting = fmt.Sprintf("こんにちは、%s！", req.Name)
	default:
		greeting = fmt.Sprintf("Hello, %s!", req.Name)
	}
	
	return &pb.HelloResponse{
		Greeting:  greeting,
		Timestamp: time.Now().Unix(),
	}, nil
}

// ตัวอย่าง 5: Starting gRPC server
func startServer() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}
	
	// Create gRPC server with options
	s := grpc.NewServer(
		grpc.UnaryInterceptor(loggingInterceptor),
		grpc.MaxRecvMsgSize(16*1024*1024), // 16MB max message
		grpc.MaxSendMsgSize(16*1024*1024),
	)
	
	pb.RegisterHelloServiceServer(s, &helloServer{})
	
	fmt.Printf("gRPC server listening on :50051\n")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}

// ตัวอย่าง 6: Logging interceptor
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	start := time.Now()
	
	resp, err := handler(ctx, req)
	
	duration := time.Since(start)
	statusCode := codes.OK
	if err != nil {
		if st, ok := status.FromError(err); ok {
			statusCode = st.Code()
		}
	}
	
	log.Printf("gRPC %s | %v | %v | %v",
		info.FullMethod, statusCode, duration, err)
	
	return resp, err
}

// ตัวอย่าง 7: gRPC Client
func grpcClient() {
	// Establish connection
	conn, err := grpc.Dial("localhost:50051",
		grpc.WithInsecure(), // use TLS in production
		grpc.WithBlock(),
		grpc.WithTimeout(5*time.Second),
	)
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}
	defer conn.Close()
	
	client := pb.NewHelloServiceClient(conn)
	
	// Make RPC call
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	
	resp, err := client.SayHello(ctx, &pb.HelloRequest{
		Name:     "Somchai",
		Language: "th",
	})
	
	if err != nil {
		// Handle gRPC errors
		st, ok := status.FromError(err)
		if ok {
			fmt.Printf("gRPC error: code=%v, message=%v\n", st.Code(), st.Message())
		} else {
			fmt.Printf("Error: %v\n", err)
		}
		return
	}
	
	fmt.Printf("Response: %s (at %v)\n", resp.Greeting, resp.Timestamp)
}

func main() {
	// Run server in goroutine
	go startServer()
	time.Sleep(100 * time.Millisecond) // wait for server to start
	
	// Run client
	grpcClient()
}
```

---

## 5. Server Streaming RPC

```go
// ตัวอย่าง 8: Server streaming - price feed

// proto definition (conceptual):
// rpc GetPriceFeed(PriceRequest) returns (stream PriceUpdate);

package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

// Simulated types (would be generated from proto)
type PriceRequest struct {
	Symbols []string
}

type PriceUpdate struct {
	Symbol    string
	Price     float64
	Timestamp int64
	Change    float64
}

// Server-side streaming implementation
type PriceServiceServer interface {
	GetPriceFeed(*PriceRequest, PriceFeedStream) error
}

type PriceFeedStream interface {
	Send(*PriceUpdate) error
	Context() context.Context
}

type priceServer struct{}

func (s *priceServer) GetPriceFeed(req *PriceRequest, stream PriceFeedStream) error {
	prices := map[string]float64{
		"AAPL": 150.0,
		"GOOG": 2800.0,
		"BTC":  45000.0,
	}
	
	ticker := time.NewTicker(500 * time.Millisecond)
	defer ticker.Stop()
	
	for {
		select {
		case <-stream.Context().Done():
			fmt.Println("Client disconnected")
			return nil
		case <-ticker.C:
			for _, symbol := range req.Symbols {
				if _, ok := prices[symbol]; !ok {
					continue
				}
				
				// Simulate price change
				change := (rand.Float64() - 0.5) * 10
				prices[symbol] += change
				
				if err := stream.Send(&PriceUpdate{
					Symbol:    symbol,
					Price:     prices[symbol],
					Timestamp: time.Now().Unix(),
					Change:    change,
				}); err != nil {
					return fmt.Errorf("failed to send: %w", err)
				}
			}
		}
	}
}

// ตัวอย่าง 9: Client consuming server stream
func consumePriceFeed() {
	fmt.Println("=== Server Streaming Demo ===")
	
	// Simulated stream consumption
	updates := []PriceUpdate{
		{Symbol: "AAPL", Price: 152.3, Change: 2.3},
		{Symbol: "BTC", Price: 45500, Change: 500},
		{Symbol: "AAPL", Price: 151.8, Change: -0.5},
	}
	
	for _, update := range updates {
		fmt.Printf("Price update: %s = %.2f (change: %+.2f)\n",
			update.Symbol, update.Price, update.Change)
	}
}

// ตัวอย่าง 10: Actual gRPC server streaming code
/*
// Server implementation
func (s *stockServer) WatchPrices(req *pb.WatchRequest, stream pb.StockService_WatchPricesServer) error {
	ctx := stream.Context()
	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()
	
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			for _, symbol := range req.Symbols {
				update := &pb.PriceUpdate{
					Symbol: symbol,
					Price:  getPrice(symbol),
					Ts:     time.Now().Unix(),
				}
				if err := stream.Send(update); err != nil {
					return err
				}
			}
		}
	}
}

// Client implementation
func watchPrices(client pb.StockServiceClient, symbols []string) {
	stream, err := client.WatchPrices(context.Background(), &pb.WatchRequest{
		Symbols: symbols,
	})
	if err != nil {
		log.Fatal(err)
	}
	
	for {
		update, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Printf("Error: %v", err)
			break
		}
		fmt.Printf("Received: %s = %.2f\n", update.Symbol, update.Price)
	}
}
*/

func main() {
	rand.Seed(time.Now().UnixNano())
	consumePriceFeed()
}
```

---

## 6. Client Streaming RPC

```go
// ตัวอย่าง 11: Client streaming - batch upload

package main

import (
	"fmt"
	"time"
)

// Conceptual client streaming implementation
// In real gRPC:
/*
// Server receives stream of files
func (s *fileServer) UploadFile(stream pb.FileService_UploadFileServer) error {
	var filename string
	var data []byte
	
	for {
		chunk, err := stream.Recv()
		if err == io.EOF {
			// All chunks received
			break
		}
		if err != nil {
			return fmt.Errorf("receive error: %w", err)
		}
		
		if filename == "" {
			filename = chunk.Filename
		}
		data = append(data, chunk.Content...)
	}
	
	// Save file
	if err := os.WriteFile(filename, data, 0644); err != nil {
		return status.Errorf(codes.Internal, "failed to save file: %v", err)
	}
	
	// Send single response
	return stream.SendAndClose(&pb.UploadResponse{
		Filename:  filename,
		Size:      int64(len(data)),
		Checksum:  calculateChecksum(data),
	})
}

// Client sends stream of file chunks
func uploadFile(client pb.FileServiceClient, filePath string) error {
	stream, err := client.UploadFile(context.Background())
	if err != nil {
		return err
	}
	
	file, err := os.Open(filePath)
	if err != nil {
		return err
	}
	defer file.Close()
	
	buf := make([]byte, 64*1024) // 64KB chunks
	filename := filepath.Base(filePath)
	
	for {
		n, err := file.Read(buf)
		if err == io.EOF {
			break
		}
		if err != nil {
			return err
		}
		
		if err := stream.Send(&pb.FileChunk{
			Filename: filename,
			Content:  buf[:n],
		}); err != nil {
			return err
		}
	}
	
	resp, err := stream.CloseAndRecv()
	if err != nil {
		return err
	}
	
	fmt.Printf("Uploaded: %s (%d bytes, checksum: %s)\n",
		resp.Filename, resp.Size, resp.Checksum)
	return nil
}
*/

// ตัวอย่าง 12: Stats collector (client streaming)
type StatsRequest struct {
	Metric string
	Value  float64
}

type StatsResponse struct {
	Count   int
	Sum     float64
	Average float64
	Min     float64
	Max     float64
}

type statsAccumulator struct {
	count int
	sum   float64
	min   float64
	max   float64
}

func (s *statsAccumulator) Add(value float64) {
	s.count++
	s.sum += value
	if s.count == 1 || value < s.min {
		s.min = value
	}
	if s.count == 1 || value > s.max {
		s.max = value
	}
}

func (s *statsAccumulator) Result() StatsResponse {
	avg := 0.0
	if s.count > 0 {
		avg = s.sum / float64(s.count)
	}
	return StatsResponse{
		Count:   s.count,
		Sum:     s.sum,
		Average: avg,
		Min:     s.min,
		Max:     s.max,
	}
}

func clientStreamingDemo() {
	fmt.Println("=== Client Streaming Demo ===")
	
	acc := &statsAccumulator{}
	
	metrics := []float64{10, 25, 7, 42, 18, 33, 5, 67, 14, 29}
	
	fmt.Println("Sending metrics:")
	for _, v := range metrics {
		acc.Add(v)
		fmt.Printf("  Sent: %.0f\n", v)
	}
	
	result := acc.Result()
	fmt.Printf("\nServer response:\n")
	fmt.Printf("  Count:   %d\n", result.Count)
	fmt.Printf("  Sum:     %.2f\n", result.Sum)
	fmt.Printf("  Average: %.2f\n", result.Average)
	fmt.Printf("  Min:     %.2f\n", result.Min)
	fmt.Printf("  Max:     %.2f\n", result.Max)
	
	_ = time.Now()
}

func main() {
	clientStreamingDemo()
}
```

---

## 7. Bidirectional Streaming RPC

```go
// ตัวอย่าง 13: Bidirectional streaming - chat

package main

import (
	"fmt"
	"sync"
	"time"
)

// Simulated bidirectional streaming chat
type ChatMessage struct {
	User      string
	Text      string
	Timestamp time.Time
}

type ChatStream struct {
	incoming chan ChatMessage
	outgoing chan ChatMessage
	done     chan struct{}
}

func NewChatStream() *ChatStream {
	return &ChatStream{
		incoming: make(chan ChatMessage, 100),
		outgoing: make(chan ChatMessage, 100),
		done:     make(chan struct{}),
	}
}

func (s *ChatStream) Send(msg ChatMessage) {
	select {
	case s.outgoing <- msg:
	case <-s.done:
	}
}

func (s *ChatStream) Recv() (ChatMessage, bool) {
	select {
	case msg := <-s.incoming:
		return msg, true
	case <-s.done:
		return ChatMessage{}, false
	}
}

// Server handles bidirectional stream
type ChatServer struct {
	mu       sync.Mutex
	clients  map[string]*ChatStream
	messages []ChatMessage
}

func NewChatServer() *ChatServer {
	return &ChatServer{
		clients: make(map[string]*ChatStream),
	}
}

func (cs *ChatServer) HandleStream(username string, stream *ChatStream) {
	cs.mu.Lock()
	cs.clients[username] = stream
	cs.mu.Unlock()
	
	defer func() {
		cs.mu.Lock()
		delete(cs.clients, username)
		cs.mu.Unlock()
	}()
	
	// Receive messages and broadcast
	for {
		msg, ok := stream.Recv()
		if !ok {
			fmt.Printf("%s disconnected\n", username)
			return
		}
		
		fmt.Printf("[Server] Received from %s: %s\n", msg.User, msg.Text)
		
		cs.mu.Lock()
		// Broadcast to all other clients
		for clientName, clientStream := range cs.clients {
			if clientName != username {
				clientStream.Send(msg)
			}
		}
		cs.mu.Unlock()
	}
}

// ตัวอย่าง 14: Bidirectional chat simulation
func biDirectionalStreamingDemo() {
	fmt.Println("=== Bidirectional Streaming Demo ===")
	
	server := NewChatServer()
	
	aliceStream := NewChatStream()
	bobStream := NewChatStream()
	
	// Start server handlers
	go server.HandleStream("Alice", aliceStream)
	go server.HandleStream("Bob", bobStream)
	
	time.Sleep(50 * time.Millisecond)
	
	// Alice sends to Bob via server
	aliceStream.incoming <- ChatMessage{User: "Alice", Text: "Hello Bob!", Timestamp: time.Now()}
	aliceStream.incoming <- ChatMessage{User: "Alice", Text: "How are you?", Timestamp: time.Now()}
	
	// Bob sends to Alice
	bobStream.incoming <- ChatMessage{User: "Bob", Text: "Hi Alice! I'm great.", Timestamp: time.Now()}
	
	time.Sleep(100 * time.Millisecond)
	
	// Bob receives Alice's messages
	fmt.Println("\nBob's received messages:")
	for {
		select {
		case msg := <-bobStream.outgoing:
			fmt.Printf("  [%s] %s\n", msg.User, msg.Text)
		default:
			goto done
		}
	}
done:
	
	// Close streams
	close(aliceStream.done)
	close(bobStream.done)
}

// ตัวอย่าง 15: Real gRPC bidirectional streaming code
/*
// Server
func (s *chatServer) Chat(stream pb.ChatService_ChatServer) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		
		// Process and broadcast
		response := &pb.ChatMessage{
			User:      msg.User,
			Text:      msg.Text,
			Timestamp: time.Now().Unix(),
		}
		
		// Broadcast to others (simplified)
		if err := stream.Send(response); err != nil {
			return err
		}
	}
}

// Client
func chatSession(client pb.ChatServiceClient, username string) {
	stream, err := client.Chat(context.Background())
	if err != nil {
		log.Fatal(err)
	}
	
	// Receive goroutine
	go func() {
		for {
			msg, err := stream.Recv()
			if err != nil {
				return
			}
			fmt.Printf("[%s] %s\n", msg.User, msg.Text)
		}
	}()
	
	// Send messages
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		stream.Send(&pb.ChatMessage{
			User: username,
			Text: scanner.Text(),
		})
	}
}
*/

func main() {
	biDirectionalStreamingDemo()
}
```

---

## 8. Error Handling in gRPC

```go
// ตัวอย่าง 16: gRPC error handling

package main

import (
	"context"
	"fmt"
	
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/anypb"
	
	// errdetails "google.golang.org/genproto/googleapis/rpc/errdetails"
)

// ตัวอย่าง 17: Creating rich error responses
func createValidationError(field, reason string) error {
	st := status.New(codes.InvalidArgument, "validation failed")
	
	// In production, use errdetails.BadRequest
	// For now, demonstrate the pattern
	details := fmt.Sprintf("field '%s': %s", field, reason)
	_ = details
	
	return st.Err()
}

func handleGRPCError(err error) {
	if err == nil {
		return
	}
	
	st, ok := status.FromError(err)
	if !ok {
		fmt.Printf("Non-gRPC error: %v\n", err)
		return
	}
	
	switch st.Code() {
	case codes.NotFound:
		fmt.Printf("Resource not found: %s\n", st.Message())
	case codes.InvalidArgument:
		fmt.Printf("Invalid argument: %s\n", st.Message())
		// In production, parse st.Details() for field errors
	case codes.Unauthenticated:
		fmt.Printf("Not authenticated: %s\n", st.Message())
	case codes.PermissionDenied:
		fmt.Printf("Permission denied: %s\n", st.Message())
	case codes.Unavailable:
		fmt.Printf("Service unavailable: %s\n", st.Message())
	case codes.DeadlineExceeded:
		fmt.Printf("Deadline exceeded: %s\n", st.Message())
	case codes.ResourceExhausted:
		fmt.Printf("Resource exhausted (rate limited): %s\n", st.Message())
	default:
		fmt.Printf("Unknown error (%v): %s\n", st.Code(), st.Message())
	}
}

// ตัวอย่าง 18: Server with proper error handling
type userService struct{}

func (s *userService) getUser(ctx context.Context, id int32) (interface{}, error) {
	if id <= 0 {
		return nil, status.Errorf(codes.InvalidArgument, 
			"user ID must be positive, got %d", id)
	}
	
	if id > 1000 {
		return nil, status.Errorf(codes.NotFound,
			"user with ID %d not found", id)
	}
	
	// Simulate permission check
	if id == 666 {
		return nil, status.Error(codes.PermissionDenied,
			"access to this user is restricted")
	}
	
	return map[string]interface{}{"id": id, "name": "User"}, nil
}

// ตัวอย่าง 19: Client-side error handling with retry
func grpcWithRetry(ctx context.Context, fn func() error) error {
	maxRetries := 3
	
	for attempt := 1; attempt <= maxRetries; attempt++ {
		err := fn()
		if err == nil {
			return nil
		}
		
		st, ok := status.FromError(err)
		if !ok {
			return err
		}
		
		switch st.Code() {
		case codes.Unavailable, codes.DeadlineExceeded, codes.ResourceExhausted:
			// Retryable errors
			if attempt < maxRetries {
				fmt.Printf("Attempt %d failed (%v), retrying...\n", attempt, st.Code())
				continue
			}
		default:
			// Non-retryable
			return err
		}
	}
	
	return fmt.Errorf("max retries exceeded")
}

// ตัวอย่าง 20: Custom error codes mapping
type AppError struct {
	Code    string
	Message string
	Details map[string]string
}

func (e *AppError) ToGRPCError() error {
	var grpcCode codes.Code
	
	switch e.Code {
	case "USER_NOT_FOUND":
		grpcCode = codes.NotFound
	case "INVALID_EMAIL":
		grpcCode = codes.InvalidArgument
	case "UNAUTHORIZED":
		grpcCode = codes.Unauthenticated
	case "DATABASE_ERROR":
		grpcCode = codes.Internal
	case "QUOTA_EXCEEDED":
		grpcCode = codes.ResourceExhausted
	default:
		grpcCode = codes.Unknown
	}
	
	return status.Errorf(grpcCode, "%s: %s", e.Code, e.Message)
}

func errorHandlingDemo() {
	fmt.Println("=== gRPC Error Handling Demo ===")
	
	svc := &userService{}
	ctx := context.Background()
	
	testCases := []struct {
		id  int32
		desc string
	}{
		{0, "invalid ID"},
		{42, "valid user"},
		{1001, "not found"},
		{666, "permission denied"},
	}
	
	for _, tc := range testCases {
		fmt.Printf("\nTest: %s (id=%d)\n", tc.desc, tc.id)
		user, err := svc.getUser(ctx, tc.id)
		if err != nil {
			handleGRPCError(err)
		} else {
			fmt.Printf("Success: %v\n", user)
		}
	}
	
	// Test custom error
	appErr := &AppError{
		Code:    "USER_NOT_FOUND",
		Message: "user with email test@example.com not found",
	}
	
	grpcErr := appErr.ToGRPCError()
	fmt.Printf("\nCustom error: %v\n", grpcErr)
	handleGRPCError(grpcErr)
	
	_ = anypb.New
}

func main() {
	errorHandlingDemo()
}
```

---

## 9. gRPC Interceptors (Middleware)

```go
// ตัวอย่าง 21: Middleware/Interceptors

package main

import (
	"context"
	"fmt"
	"time"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

// ตัวอย่าง 22: Authentication interceptor
func authInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	// Skip auth for health check
	if info.FullMethod == "/health.HealthService/Check" {
		return handler(ctx, req)
	}
	
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "missing metadata")
	}
	
	authHeader := md.Get("authorization")
	if len(authHeader) == 0 {
		return nil, status.Error(codes.Unauthenticated, "missing authorization token")
	}
	
	token := authHeader[0]
	if token != "Bearer valid-token" { // Simplified check
		return nil, status.Error(codes.Unauthenticated, "invalid token")
	}
	
	// Add user info to context
	ctx = context.WithValue(ctx, "user_id", "user-123")
	
	return handler(ctx, req)
}

// ตัวอย่าง 23: Rate limiting interceptor
type rateLimiter struct {
	tokens chan struct{}
}

func newRateLimiter(rps int) *rateLimiter {
	rl := &rateLimiter{
		tokens: make(chan struct{}, rps),
	}
	// Fill tokens
	for i := 0; i < rps; i++ {
		rl.tokens <- struct{}{}
	}
	// Refill
	go func() {
		ticker := time.NewTicker(time.Second / time.Duration(rps))
		for range ticker.C {
			select {
			case rl.tokens <- struct{}{}:
			default:
			}
		}
	}()
	return rl
}

func (rl *rateLimiter) interceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	select {
	case <-rl.tokens:
		return handler(ctx, req)
	default:
		return nil, status.Error(codes.ResourceExhausted, "too many requests")
	}
}

// ตัวอย่าง 24: Recovery interceptor
func recoveryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("Recovered from panic in %s: %v\n", info.FullMethod, r)
			err = status.Errorf(codes.Internal, "internal server error")
		}
	}()
	return handler(ctx, req)
}

// ตัวอย่าง 25: Chaining interceptors
func chainInterceptors(interceptors ...grpc.UnaryServerInterceptor) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		chain := handler
		for i := len(interceptors) - 1; i >= 0; i-- {
			interceptor := interceptors[i]
			next := chain
			chain = func(ctx context.Context, req interface{}) (interface{}, error) {
				return interceptor(ctx, req, info, next)
			}
		}
		return chain(ctx, req)
	}
}

func interceptorDemo() {
	fmt.Println("=== Interceptor Demo ===")
	
	rl := newRateLimiter(5) // 5 rps
	
	// Create server with chained interceptors
	chainedInterceptor := chainInterceptors(
		recoveryInterceptor,
		authInterceptor,
		rl.interceptor,
		loggingInterceptor,
	)
	
	_ = grpc.NewServer(
		grpc.UnaryInterceptor(chainedInterceptor),
	)
	
	fmt.Println("Server configured with interceptors:")
	fmt.Println("  1. Recovery (panic handler)")
	fmt.Println("  2. Authentication")
	fmt.Println("  3. Rate limiting (5 rps)")
	fmt.Println("  4. Logging")
}

func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	start := time.Now()
	resp, err := handler(ctx, req)
	fmt.Printf("[%s] %s | %v\n", time.Since(start), info.FullMethod, err)
	return resp, err
}

func main() {
	interceptorDemo()
}
```

---

## Workshop: Building a Complete gRPC Service

```go
// ตัวอย่าง: Product Service with full gRPC implementation

package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// Simulated proto-generated types
type Product struct {
	ID          string
	Name        string
	Description string
	Price       float64
	Stock       int32
	CreatedAt   time.Time
}

type CreateProductRequest struct {
	Name        string
	Description string
	Price       float64
	InitialStock int32
}

type GetProductRequest struct {
	ID string
}

type UpdateStockRequest struct {
	ProductID string
	Delta     int32 // positive = add, negative = remove
}

// ProductStore - simulates database
type ProductStore struct {
	mu       sync.RWMutex
	products map[string]*Product
}

func NewProductStore() *ProductStore {
	return &ProductStore{
		products: make(map[string]*Product),
	}
}

func (ps *ProductStore) Create(p *Product) error {
	ps.mu.Lock()
	defer ps.mu.Unlock()
	ps.products[p.ID] = p
	return nil
}

func (ps *ProductStore) Get(id string) (*Product, error) {
	ps.mu.RLock()
	defer ps.mu.RUnlock()
	p, ok := ps.products[id]
	if !ok {
		return nil, fmt.Errorf("product %s not found", id)
	}
	return p, nil
}

func (ps *ProductStore) UpdateStock(id string, delta int32) (*Product, error) {
	ps.mu.Lock()
	defer ps.mu.Unlock()
	
	p, ok := ps.products[id]
	if !ok {
		return nil, fmt.Errorf("product %s not found", id)
	}
	
	newStock := p.Stock + delta
	if newStock < 0 {
		return nil, fmt.Errorf("insufficient stock: have %d, need %d", p.Stock, -delta)
	}
	
	p.Stock = newStock
	return p, nil
}

// ProductServiceServer implementation
type ProductServiceServer struct {
	store  *ProductStore
	nextID int
	mu     sync.Mutex
	
	// For server streaming - watchers
	watchers map[string]chan *Product
}

func NewProductServiceServer() *ProductServiceServer {
	return &ProductServiceServer{
		store:    NewProductStore(),
		watchers: make(map[string]chan *Product),
	}
}

func (s *ProductServiceServer) generateID() string {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.nextID++
	return fmt.Sprintf("PROD-%04d", s.nextID)
}

func (s *ProductServiceServer) CreateProduct(ctx context.Context, req *CreateProductRequest) (*Product, error) {
	if req.Name == "" {
		return nil, fmt.Errorf("product name is required")
	}
	if req.Price <= 0 {
		return nil, fmt.Errorf("price must be positive")
	}
	
	p := &Product{
		ID:          s.generateID(),
		Name:        req.Name,
		Description: req.Description,
		Price:       req.Price,
		Stock:       req.InitialStock,
		CreatedAt:   time.Now(),
	}
	
	if err := s.store.Create(p); err != nil {
		return nil, err
	}
	
	return p, nil
}

func (s *ProductServiceServer) GetProduct(ctx context.Context, req *GetProductRequest) (*Product, error) {
	return s.store.Get(req.ID)
}

func (s *ProductServiceServer) UpdateStock(ctx context.Context, req *UpdateStockRequest) (*Product, error) {
	p, err := s.store.UpdateStock(req.ProductID, req.Delta)
	if err != nil {
		return nil, err
	}
	
	// Notify watchers
	s.mu.Lock()
	if watcher, ok := s.watchers[req.ProductID]; ok {
		select {
		case watcher <- p:
		default:
		}
	}
	s.mu.Unlock()
	
	return p, nil
}

// Simulate server streaming - watch product
func (s *ProductServiceServer) WatchProduct(productID string, updates chan<- *Product, done <-chan struct{}) error {
	watcher := make(chan *Product, 10)
	
	s.mu.Lock()
	s.watchers[productID] = watcher
	s.mu.Unlock()
	
	defer func() {
		s.mu.Lock()
		delete(s.watchers, productID)
		s.mu.Unlock()
	}()
	
	for {
		select {
		case update := <-watcher:
			updates <- update
		case <-done:
			return nil
		}
	}
}

func productServiceDemo() {
	fmt.Println("=== Product Service Workshop ===")
	
	svc := NewProductServiceServer()
	ctx := context.Background()
	
	// Create products
	laptop, err := svc.CreateProduct(ctx, &CreateProductRequest{
		Name:         "MacBook Pro",
		Description:  "16-inch laptop",
		Price:        79900,
		InitialStock: 10,
	})
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Created: %s - %s (stock: %d)\n", laptop.ID, laptop.Name, laptop.Stock)
	
	phone, err := svc.CreateProduct(ctx, &CreateProductRequest{
		Name:         "iPhone 15",
		Price:        32900,
		InitialStock: 50,
	})
	fmt.Printf("Created: %s - %s (stock: %d)\n", phone.ID, phone.Name, phone.Stock)
	
	// Watch for stock changes (server streaming)
	updates := make(chan *Product, 10)
	done := make(chan struct{})
	
	go svc.WatchProduct(laptop.ID, updates, done)
	
	go func() {
		for update := range updates {
			fmt.Printf("[Watch] %s stock: %d\n", update.Name, update.Stock)
		}
	}()
	
	// Update stock
	time.Sleep(50 * time.Millisecond)
	svc.UpdateStock(ctx, &UpdateStockRequest{ProductID: laptop.ID, Delta: -2})
	svc.UpdateStock(ctx, &UpdateStockRequest{ProductID: laptop.ID, Delta: 5})
	svc.UpdateStock(ctx, &UpdateStockRequest{ProductID: laptop.ID, Delta: -1})
	
	time.Sleep(100 * time.Millisecond)
	close(done)
	
	// Get product
	retrieved, _ := svc.GetProduct(ctx, &GetProductRequest{ID: laptop.ID})
	fmt.Printf("\nFinal state: %s, stock=%d, price=%.0f\n",
		retrieved.Name, retrieved.Stock, retrieved.Price)
	
	// Try to oversell
	_, err = svc.UpdateStock(ctx, &UpdateStockRequest{ProductID: laptop.ID, Delta: -100})
	if err != nil {
		fmt.Printf("Expected error: %v\n", err)
	}
}

func main() {
	productServiceDemo()
}
```

---

## สรุป

ใน Part 42 เราได้เรียนรู้:

1. **gRPC Overview**: HTTP/2, Protocol Buffers, code generation
2. **Unary RPC**: request-response pattern
3. **Server Streaming**: server sends stream of responses
4. **Client Streaming**: client sends stream of requests
5. **Bidirectional Streaming**: both sides stream simultaneously
6. **Error Handling**: gRPC status codes and error details
7. **Interceptors**: middleware for cross-cutting concerns
8. **Workshop**: Complete product service implementation

---

## Resources

- [gRPC Go Documentation](https://grpc.io/docs/languages/go/)
- [Protocol Buffers](https://protobuf.dev/)
- [gRPC Status Codes](https://grpc.io/docs/guides/status-codes/)
- [gRPC Interceptors](https://github.com/grpc-ecosystem/go-grpc-middleware)
