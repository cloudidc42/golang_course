# Part 61: gRPC Advanced

## เป้าหมายการเรียนรู้
- gRPC interceptors
- Authentication (JWT, mTLS)
- gRPC health checking
- gRPC reflection
- gRPC gateway (REST + gRPC)
- Load balancing with gRPC
- Error handling best practices
- gRPC with Kubernetes

---

## 1. gRPC Interceptors

Interceptors ใน gRPC คล้ายกับ Middleware ใน HTTP

```proto
// proto/user.proto
syntax = "proto3";
package user;
option go_package = "github.com/example/grpc/proto/user;userpb";

import "google/protobuf/timestamp.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc StreamUsers(ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  UserResponse user = 1;
}

message UserResponse {
  string id = 1;
  string email = 2;
  string name = 3;
  string role = 4;
  google.protobuf.Timestamp created_at = 5;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string role = 3;
}

message CreateUserResponse {
  UserResponse user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string role_filter = 3;
}

message ListUsersResponse {
  repeated UserResponse users = 1;
  int32 total = 2;
}
```

```go
// interceptors/logging.go
package interceptors

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// UnaryLoggingInterceptor logs all unary RPC calls
func UnaryLoggingInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()
	
	// Get request ID from metadata
	reqID := getRequestID(ctx)
	
	log.Printf("[%s] gRPC call: %s started", reqID, info.FullMethod)
	
	resp, err := handler(ctx, req)
	
	duration := time.Since(start)
	code := status.Code(err)
	
	log.Printf("[%s] gRPC call: %s finished | code: %s | duration: %v | error: %v",
		reqID, info.FullMethod, code, duration, err)
	
	return resp, err
}

// StreamLoggingInterceptor logs all streaming RPC calls
func StreamLoggingInterceptor(
	srv interface{},
	ss grpc.ServerStream,
	info *grpc.StreamServerInfo,
	handler grpc.StreamHandler,
) error {
	start := time.Now()
	log.Printf("gRPC stream started: %s", info.FullMethod)
	
	err := handler(srv, ss)
	
	log.Printf("gRPC stream finished: %s | duration: %v | error: %v",
		info.FullMethod, time.Since(start), err)
	
	return err
}

func getRequestID(ctx context.Context) string {
	// Extract from metadata
	return "req-id"
}

// RecoveryInterceptor recovers from panics
func UnaryRecoveryInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (resp interface{}, err error) {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("Panic recovered in %s: %v", info.FullMethod, r)
			err = status.Errorf(codes.Internal, "internal server error")
		}
	}()
	return handler(ctx, req)
}

// MetricsInterceptor records metrics
func UnaryMetricsInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()
	
	resp, err := handler(ctx, req)
	
	duration := time.Since(start)
	code := status.Code(err)
	
	// Record metrics (prometheus, etc.)
	_ = duration
	_ = code
	// grpcRequestsTotal.WithLabelValues(info.FullMethod, code.String()).Inc()
	// grpcRequestDuration.WithLabelValues(info.FullMethod).Observe(duration.Seconds())
	
	return resp, err
}
```

---

## 2. Authentication Interceptors

```go
// interceptors/auth.go
package interceptors

import (
	"context"
	"strings"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

type Claims struct {
	UserID string
	Email  string
	Roles  []string
}

type JWTValidator interface {
	Validate(token string) (*Claims, error)
}

type claimsKey struct{}

// UnaryAuthInterceptor validates JWT tokens
func UnaryAuthInterceptor(validator JWTValidator) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		// Skip auth for health checks
		if info.FullMethod == "/grpc.health.v1.Health/Check" {
			return handler(ctx, req)
		}

		claims, err := extractAndValidateToken(ctx, validator)
		if err != nil {
			return nil, err
		}

		// Add claims to context
		ctx = context.WithValue(ctx, claimsKey{}, claims)
		return handler(ctx, req)
	}
}

func extractAndValidateToken(ctx context.Context, validator JWTValidator) (*Claims, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "no metadata provided")
	}

	authHeader := md.Get("authorization")
	if len(authHeader) == 0 {
		return nil, status.Error(codes.Unauthenticated, "no authorization header")
	}

	parts := strings.SplitN(authHeader[0], " ", 2)
	if len(parts) != 2 || parts[0] != "Bearer" {
		return nil, status.Error(codes.Unauthenticated, "invalid authorization format")
	}

	claims, err := validator.Validate(parts[1])
	if err != nil {
		return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
	}

	return claims, nil
}

func GetClaims(ctx context.Context) (*Claims, bool) {
	claims, ok := ctx.Value(claimsKey{}).(*Claims)
	return claims, ok
}

// RBACInterceptor checks roles
func RBACInterceptor(methodRoles map[string][]string) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		requiredRoles, ok := methodRoles[info.FullMethod]
		if !ok {
			// No roles required
			return handler(ctx, req)
		}

		claims, ok := GetClaims(ctx)
		if !ok {
			return nil, status.Error(codes.PermissionDenied, "no auth context")
		}

		roleMap := make(map[string]bool)
		for _, r := range claims.Roles {
			roleMap[r] = true
		}

		for _, required := range requiredRoles {
			if !roleMap[required] {
				return nil, status.Errorf(codes.PermissionDenied, "role %s required", required)
			}
		}

		return handler(ctx, req)
	}
}
```

---

## 3. mTLS Authentication

```go
// mtls/server.go
package mtls

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"os"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)

// LoadServerTLSCredentials โหลด TLS credentials สำหรับ server
func LoadServerTLSCredentials(certFile, keyFile, caFile string) (credentials.TransportCredentials, error) {
	// Load server cert and key
	serverCert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, fmt.Errorf("loading server cert/key: %w", err)
	}

	// Load CA cert for client verification
	caCert, err := os.ReadFile(caFile)
	if err != nil {
		return nil, fmt.Errorf("reading CA cert: %w", err)
	}

	certPool := x509.NewCertPool()
	if !certPool.AppendCertsFromPEM(caCert) {
		return nil, fmt.Errorf("failed to add CA cert to pool")
	}

	config := &tls.Config{
		Certificates: []tls.Certificate{serverCert},
		ClientAuth:   tls.RequireAndVerifyClientCert, // require client cert
		ClientCAs:    certPool,
		MinVersion:   tls.VersionTLS13,
	}

	return credentials.NewTLS(config), nil
}

// LoadClientTLSCredentials โหลด TLS credentials สำหรับ client
func LoadClientTLSCredentials(certFile, keyFile, caFile string) (credentials.TransportCredentials, error) {
	clientCert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, fmt.Errorf("loading client cert/key: %w", err)
	}

	caCert, err := os.ReadFile(caFile)
	if err != nil {
		return nil, fmt.Errorf("reading CA cert: %w", err)
	}

	certPool := x509.NewCertPool()
	if !certPool.AppendCertsFromPEM(caCert) {
		return nil, fmt.Errorf("failed to add CA cert to pool")
	}

	config := &tls.Config{
		Certificates: []tls.Certificate{clientCert},
		RootCAs:      certPool,
		MinVersion:   tls.VersionTLS13,
	}

	return credentials.NewTLS(config), nil
}

// NewMTLSServer สร้าง gRPC server ที่ต้องการ mTLS
func NewMTLSServer(certFile, keyFile, caFile string, opts ...grpc.ServerOption) (*grpc.Server, error) {
	creds, err := LoadServerTLSCredentials(certFile, keyFile, caFile)
	if err != nil {
		return nil, err
	}

	allOpts := append([]grpc.ServerOption{grpc.Creds(creds)}, opts...)
	return grpc.NewServer(allOpts...), nil
}
```

---

## 4. gRPC Health Checking

```go
// health/health_server.go
package health

import (
	"context"
	"sync"

	"google.golang.org/grpc/health/grpc_health_v1"
)

// HealthServer implements gRPC health checking protocol
type HealthServer struct {
	grpc_health_v1.UnimplementedHealthServer
	mu       sync.RWMutex
	services map[string]grpc_health_v1.HealthCheckResponse_ServingStatus
}

func NewHealthServer() *HealthServer {
	return &HealthServer{
		services: make(map[string]grpc_health_v1.HealthCheckResponse_ServingStatus),
	}
}

func (s *HealthServer) Check(ctx context.Context, req *grpc_health_v1.HealthCheckRequest) (*grpc_health_v1.HealthCheckResponse, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	status, ok := s.services[req.Service]
	if !ok {
		// Default to serving if not explicitly set
		return &grpc_health_v1.HealthCheckResponse{
			Status: grpc_health_v1.HealthCheckResponse_SERVING,
		}, nil
	}

	return &grpc_health_v1.HealthCheckResponse{Status: status}, nil
}

func (s *HealthServer) Watch(req *grpc_health_v1.HealthCheckRequest, stream grpc_health_v1.Health_WatchServer) error {
	// Simplified - production should send updates when status changes
	status := grpc_health_v1.HealthCheckResponse_SERVING
	return stream.Send(&grpc_health_v1.HealthCheckResponse{Status: status})
}

func (s *HealthServer) SetServingStatus(service string, status grpc_health_v1.HealthCheckResponse_ServingStatus) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.services[service] = status
}

func (s *HealthServer) SetHealthy(service string) {
	s.SetServingStatus(service, grpc_health_v1.HealthCheckResponse_SERVING)
}

func (s *HealthServer) SetUnhealthy(service string) {
	s.SetServingStatus(service, grpc_health_v1.HealthCheckResponse_NOT_SERVING)
}
```

---

## 5. gRPC Error Handling

```go
// errors/errors.go
package errors

import (
	"errors"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/anypb"
)

// Domain errors to gRPC status mapping
func ToGRPCError(err error) error {
	if err == nil {
		return nil
	}

	// Check for specific error types
	var notFound *NotFoundError
	if errors.As(err, &notFound) {
		return status.Errorf(codes.NotFound, "%s", err.Error())
	}

	var validation *ValidationError
	if errors.As(err, &validation) {
		st, _ := status.New(codes.InvalidArgument, err.Error()).
			WithDetails(&anypb.Any{}) // Add field violations
		return st.Err()
	}

	var unauthorized *UnauthorizedError
	if errors.As(err, &unauthorized) {
		return status.Errorf(codes.Unauthenticated, "%s", err.Error())
	}

	var forbidden *ForbiddenError
	if errors.As(err, &forbidden) {
		return status.Errorf(codes.PermissionDenied, "%s", err.Error())
	}

	var conflict *ConflictError
	if errors.As(err, &conflict) {
		return status.Errorf(codes.AlreadyExists, "%s", err.Error())
	}

	// Default: internal error
	return status.Errorf(codes.Internal, "internal error: %v", err)
}

type NotFoundError struct{ Message string }
func (e *NotFoundError) Error() string { return e.Message }

type ValidationError struct{ Message string }
func (e *ValidationError) Error() string { return e.Message }

type UnauthorizedError struct{ Message string }
func (e *UnauthorizedError) Error() string { return e.Message }

type ForbiddenError struct{ Message string }
func (e *ForbiddenError) Error() string { return e.Message }

type ConflictError struct{ Message string }
func (e *ConflictError) Error() string { return e.Message }

// FromGRPCError แปลง gRPC error กลับเป็น domain error (client side)
func FromGRPCError(err error) error {
	if err == nil {
		return nil
	}

	st, ok := status.FromError(err)
	if !ok {
		return err
	}

	switch st.Code() {
	case codes.NotFound:
		return &NotFoundError{Message: st.Message()}
	case codes.InvalidArgument:
		return &ValidationError{Message: st.Message()}
	case codes.Unauthenticated:
		return &UnauthorizedError{Message: st.Message()}
	case codes.PermissionDenied:
		return &ForbiddenError{Message: st.Message()}
	case codes.AlreadyExists:
		return &ConflictError{Message: st.Message()}
	default:
		return errors.New(st.Message())
	}
}
```

---

## 6. gRPC Gateway (REST + gRPC)

```proto
// proto/user.proto (with gateway annotations)
syntax = "proto3";
package user.v1;

import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/api/v1/users/{user_id}"
    };
  }
  
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/api/v1/users"
      body: "*"
    };
  }
  
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {
    option (google.api.http) = {
      get: "/api/v1/users"
    };
  }
}
```

```go
// gateway/main.go
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"net/http"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/reflection"

	userpb "github.com/example/grpc/proto/user"
)

func main() {
	// Start gRPC server
	go startGRPCServer()

	// Start HTTP gateway
	if err := startHTTPGateway(); err != nil {
		log.Fatalf("HTTP gateway error: %v", err)
	}
}

func startGRPCServer() {
	lis, err := net.Listen("tcp", ":9090")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	s := grpc.NewServer()
	userpb.RegisterUserServiceServer(s, &userServer{})
	reflection.Register(s) // enable reflection for grpcurl

	log.Println("gRPC server starting on :9090")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("gRPC server error: %v", err)
	}
}

func startHTTPGateway() error {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux(
		runtime.WithErrorHandler(customErrorHandler),
		runtime.WithForwardResponseOption(addCORSHeaders),
	)

	opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	
	if err := userpb.RegisterUserServiceHandlerFromEndpoint(ctx, mux, "localhost:9090", opts); err != nil {
		return fmt.Errorf("registering gateway: %w", err)
	}

	log.Println("HTTP gateway starting on :8080")
	return http.ListenAndServe(":8080", mux)
}

func customErrorHandler(
	ctx context.Context,
	mux *runtime.ServeMux,
	marshaler runtime.Marshaler,
	w http.ResponseWriter,
	r *http.Request,
	err error,
) {
	// Custom error format
	runtime.DefaultHTTPErrorHandler(ctx, mux, marshaler, w, r, err)
}

func addCORSHeaders(ctx context.Context, w http.ResponseWriter, resp interface{}) error {
	w.Header().Set("Access-Control-Allow-Origin", "*")
	return nil
}

// Server implementation
type userServer struct {
	userpb.UnimplementedUserServiceServer
}

func (s *userServer) GetUser(ctx context.Context, req *userpb.GetUserRequest) (*userpb.GetUserResponse, error) {
	return &userpb.GetUserResponse{
		User: &userpb.UserResponse{
			Id:    req.UserId,
			Email: "user@example.com",
			Name:  "Test User",
			Role:  "user",
		},
	}, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *userpb.CreateUserRequest) (*userpb.CreateUserResponse, error) {
	return &userpb.CreateUserResponse{
		User: &userpb.UserResponse{
			Id:    "usr-001",
			Email: req.Email,
			Name:  req.Name,
			Role:  req.Role,
		},
	}, nil
}

func (s *userServer) ListUsers(ctx context.Context, req *userpb.ListUsersRequest) (*userpb.ListUsersResponse, error) {
	return &userpb.ListUsersResponse{
		Users: []*userpb.UserResponse{
			{Id: "1", Email: "alice@example.com", Name: "Alice"},
		},
		Total: 1,
	}, nil
}
```

---

## 7. gRPC Load Balancing

```go
// client/loadbalanced_client.go
package client

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/balancer/roundrobin"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/resolver"

	userpb "github.com/example/grpc/proto/user"
)

// Custom resolver สำหรับ service discovery
type serviceResolver struct {
	cc resolver.ClientConn
}

func (r *serviceResolver) start() {
	// Provide initial addresses
	r.cc.UpdateState(resolver.State{
		Addresses: []resolver.Address{
			{Addr: "localhost:9091"},
			{Addr: "localhost:9092"},
			{Addr: "localhost:9093"},
		},
	})
}

func (r *serviceResolver) ResolveNow(opts resolver.ResolveNowOptions) {
	// Re-resolve (e.g., after connection failure)
	r.start()
}

func (r *serviceResolver) Close() {}

type serviceResolverBuilder struct{}

func (b *serviceResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	r := &serviceResolver{cc: cc}
	r.start()
	return r, nil
}

func (b *serviceResolverBuilder) Scheme() string { return "discovery" }

func init() {
	resolver.Register(&serviceResolverBuilder{})
}

// NewLoadBalancedClient สร้าง gRPC client ที่ใช้ load balancing
func NewLoadBalancedClient(serviceName string) (userpb.UserServiceClient, *grpc.ClientConn, error) {
	target := fmt.Sprintf("discovery:///%s", serviceName)

	conn, err := grpc.Dial(
		target,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithDefaultServiceConfig(fmt.Sprintf(`{
			"loadBalancingConfig": [{"%s": {}}],
			"methodConfig": [{
				"name": [{"service": "user.UserService"}],
				"retryPolicy": {
					"maxAttempts": 3,
					"initialBackoff": "0.1s",
					"maxBackoff": "1s",
					"backoffMultiplier": 2,
					"retryableStatusCodes": ["UNAVAILABLE"]
				},
				"timeout": "10s"
			}]
		}`, roundrobin.Name)),
	)
	if err != nil {
		return nil, nil, fmt.Errorf("dialing: %w", err)
	}

	return userpb.NewUserServiceClient(conn), conn, nil
}

// Client with retry
func callWithRetry(ctx context.Context, client userpb.UserServiceClient, userID string) (*userpb.UserResponse, error) {
	resp, err := client.GetUser(ctx, &userpb.GetUserRequest{UserId: userID})
	if err != nil {
		return nil, fmt.Errorf("GetUser failed: %w", err)
	}
	return resp.User, nil
}
```

---

## 8. Streaming

```go
// server/streaming.go
package server

import (
	"fmt"
	"time"

	userpb "github.com/example/grpc/proto/user"
)

// Server streaming
func (s *userServer) StreamUsers(req *userpb.ListUsersRequest, stream userpb.UserService_StreamUsersServer) error {
	// Simulate streaming users
	users := []struct {
		id    string
		email string
		name  string
	}{
		{"1", "alice@example.com", "Alice"},
		{"2", "bob@example.com", "Bob"},
		{"3", "charlie@example.com", "Charlie"},
	}

	for _, u := range users {
		if err := stream.Send(&userpb.UserResponse{
			Id:    u.id,
			Email: u.email,
			Name:  u.name,
		}); err != nil {
			return fmt.Errorf("sending user: %w", err)
		}
		// Simulate processing time
		time.Sleep(100 * time.Millisecond)
	}
	return nil
}

// Client streaming example (if we had chat-like service)
// Bidirectional streaming
```

---

## สรุป

| Feature | Implementation |
|---------|---------------|
| Interceptors | `grpc.UnaryInterceptor` / `grpc.StreamInterceptor` |
| JWT Auth | Extract from metadata, validate, add to context |
| mTLS | `tls.Config` with `ClientAuth: RequireAndVerify` |
| Health Check | `google.golang.org/grpc/health` package |
| Error Handling | `google.golang.org/grpc/status` package |
| REST Gateway | `grpc-gateway` library |
| Load Balancing | Custom `resolver.Builder` + `balancer` |

---

**ต่อไป**: Part 62 - GraphQL
