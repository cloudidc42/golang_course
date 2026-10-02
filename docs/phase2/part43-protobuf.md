# Part 43: Protocol Buffers ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เข้าใจ Protocol Buffers syntax
- ใช้ Message types, Field types, Field numbers
- สร้าง Nested messages, Enums, Oneof, Maps
- กำหนด Services ใน proto files
- Generate code ด้วย protoc
- จัดการ Proto versioning

---

## 1. Protocol Buffers Overview

Protocol Buffers (Protobuf) คือ language-neutral, platform-neutral format สำหรับ serializing structured data

**ประโยชน์:**
- Binary format (เล็กกว่า JSON 3-10x)
- เร็วกว่า JSON ในการ serialize/deserialize
- Type-safe
- Backward/Forward compatible
- Auto-generate code

---

## 2. Basic Syntax

```protobuf
// ตัวอย่าง 1: Basic proto3 syntax
syntax = "proto3";

// Package declaration
package myapp.v1;

// Go package path (required for Go generation)
option go_package = "github.com/yourorg/myapp/gen/myapp/v1;myappv1";

// Java package (if targeting Java)
option java_package = "com.example.myapp";
option java_multiple_files = true;

// Simple message
message Person {
  // Field: <type> <name> = <field_number>;
  string name = 1;
  int32 age = 2;
  string email = 3;
}
```

---

## 3. Field Types

```protobuf
// ตัวอย่าง 2: All scalar field types
message AllTypes {
  // Integers
  int32    int32_val    = 1;   // -2^31 to 2^31-1
  int64    int64_val    = 2;   // -2^63 to 2^63-1
  uint32   uint32_val   = 3;   // 0 to 2^32-1
  uint64   uint64_val   = 4;   // 0 to 2^64-1
  sint32   sint32_val   = 5;   // signed int32 (more efficient for negative)
  sint64   sint64_val   = 6;   // signed int64
  fixed32  fixed32_val  = 7;   // always 4 bytes (better for > 2^28)
  fixed64  fixed64_val  = 8;   // always 8 bytes (better for > 2^56)
  sfixed32 sfixed32_val = 9;   // signed fixed32
  sfixed64 sfixed64_val = 10;  // signed fixed64
  
  // Floating point
  float  float_val  = 11;  // 32-bit
  double double_val = 12;  // 64-bit
  
  // Boolean
  bool bool_val = 13;
  
  // String (UTF-8)
  string string_val = 14;
  
  // Bytes (arbitrary binary)
  bytes bytes_val = 15;
}
```

```go
// ตัวอย่าง 3: Using generated Go code

package main

import (
	"fmt"
	"google.golang.org/protobuf/proto"
)

// Simulated generated types (normally from protoc)
// In real code, import from generated package

func protoBasicsDemo() {
	// Create a message
	p := &Person{
		Name:  "Alice",
		Age:   30,
		Email: "alice@example.com",
	}
	
	// Serialize to binary
	data, err := proto.Marshal(p)
	if err != nil {
		panic(err)
	}
	fmt.Printf("Serialized size: %d bytes\n", len(data))
	
	// Deserialize
	var decoded Person
	if err := proto.Unmarshal(data, &decoded); err != nil {
		panic(err)
	}
	fmt.Printf("Decoded: name=%s, age=%d, email=%s\n",
		decoded.Name, decoded.Age, decoded.Email)
	
	// Compare JSON size
	import "encoding/json"
	jsonData, _ := json.Marshal(map[string]interface{}{
		"name": p.Name, "age": p.Age, "email": p.Email,
	})
	fmt.Printf("JSON size: %d bytes\n", len(jsonData))
	fmt.Printf("Protobuf is %.1fx smaller\n", float64(len(jsonData))/float64(len(data)))
}
```

---

## 4. Field Numbers and Reserved Fields

```protobuf
// ตัวอย่าง 4: Field numbers and reservation

message UserProfile {
  // Field numbers 1-15 use 1 byte (use for frequently accessed fields)
  int32  id    = 1;
  string name  = 2;
  string email = 3;
  
  // Field numbers 16-2047 use 2 bytes
  string address = 16;
  string phone   = 17;
  
  // Never use: 19000-19999 (reserved for protocol buffers)
  
  // Reserve fields that are removed to prevent accidental reuse
  reserved 4, 5, 6;
  reserved "old_field", "deprecated_field";
}
```

---

## 5. Nested Messages

```protobuf
// ตัวอย่าง 5: Nested messages

message Order {
  int32 id = 1;
  string status = 2;
  
  // Nested message type
  Customer customer = 3;
  
  // Repeated nested messages
  repeated OrderItem items = 4;
  
  // Embedded/anonymous message
  ShippingAddress shipping = 5;
  
  message Customer {
    int32  id    = 1;
    string name  = 2;
    string email = 3;
  }
  
  message OrderItem {
    int32  product_id = 1;
    string name       = 2;
    int32  quantity   = 3;
    double price      = 4;
  }
  
  message ShippingAddress {
    string street   = 1;
    string city     = 2;
    string province = 3;
    string postcode = 4;
    string country  = 5;
  }
  
  // Money type (Google Common Types pattern)
  double total_amount = 6;
  string currency     = 7;
  
  // Timestamps
  int64 created_at = 8;
  int64 updated_at = 9;
}
```

```go
// ตัวอย่าง 6: Working with nested messages

package main

import (
	"fmt"
	"time"
	
	"google.golang.org/protobuf/proto"
)

func nestedMessageDemo() {
	// Creating nested proto messages
	// (In real code, these types come from protoc-generated code)
	
	order := createSampleOrder()
	
	// Access nested fields
	fmt.Printf("Order ID: %d\n", order.ID)
	fmt.Printf("Customer: %s (%s)\n", order.Customer.Name, order.Customer.Email)
	fmt.Printf("Items: %d\n", len(order.Items))
	fmt.Printf("Shipping to: %s, %s\n", order.Shipping.City, order.Shipping.Country)
	fmt.Printf("Total: %.2f %s\n", order.TotalAmount, order.Currency)
	
	// Serialize
	data, _ := proto.Marshal(convertToProto(order))
	fmt.Printf("Serialized size: %d bytes\n", len(data))
}

// Simulated order struct
type Order struct {
	ID       int32
	Status   string
	Customer struct {
		ID    int32
		Name  string
		Email string
	}
	Items []struct {
		ProductID int32
		Name      string
		Quantity  int32
		Price     float64
	}
	Shipping struct {
		Street   string
		City     string
		Province string
		Postcode string
		Country  string
	}
	TotalAmount float64
	Currency    string
	CreatedAt   time.Time
}

func createSampleOrder() Order {
	var o Order
	o.ID = 12345
	o.Status = "pending"
	o.Customer.ID = 1
	o.Customer.Name = "Alice"
	o.Customer.Email = "alice@example.com"
	o.Items = append(o.Items, struct {
		ProductID int32
		Name      string
		Quantity  int32
		Price     float64
	}{1, "MacBook Pro", 1, 79900})
	o.Shipping.Street = "123 Main St"
	o.Shipping.City = "Bangkok"
	o.Shipping.Province = "Bangkok"
	o.Shipping.Postcode = "10110"
	o.Shipping.Country = "Thailand"
	o.TotalAmount = 79900
	o.Currency = "THB"
	o.CreatedAt = time.Now()
	return o
}

func convertToProto(o Order) proto.Message {
	// Placeholder - in real code this would be the generated type
	return nil
}

func main() {
	nestedMessageDemo()
}
```

---

## 6. Enums

```protobuf
// ตัวอย่าง 7: Enum definitions

syntax = "proto3";

// Top-level enum
enum Status {
  // First value must be 0
  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE = 1;
  STATUS_INACTIVE = 2;
  STATUS_DELETED = 3;
}

message User {
  int32  id     = 1;
  string name   = 2;
  Status status = 3;
  
  // Nested enum
  Role role = 4;
  
  enum Role {
    ROLE_UNSPECIFIED = 0;
    ROLE_USER        = 1;
    ROLE_ADMIN       = 2;
    ROLE_SUPERADMIN  = 3;
  }
}

// Enum with allow_alias
enum Direction {
  option allow_alias = true;
  DIRECTION_UNSPECIFIED = 0;
  NORTH = 1;
  UP = 1; // alias for NORTH
  SOUTH = 2;
  EAST = 3;
  WEST = 4;
}
```

```go
// ตัวอย่าง 8: Working with enums in Go

package main

import "fmt"

// Simulated proto-generated enum
type UserStatus int32

const (
	UserStatus_UNSPECIFIED UserStatus = 0
	UserStatus_ACTIVE      UserStatus = 1
	UserStatus_INACTIVE    UserStatus = 2
	UserStatus_DELETED     UserStatus = 3
)

func (s UserStatus) String() string {
	switch s {
	case UserStatus_UNSPECIFIED:
		return "UNSPECIFIED"
	case UserStatus_ACTIVE:
		return "ACTIVE"
	case UserStatus_INACTIVE:
		return "INACTIVE"
	case UserStatus_DELETED:
		return "DELETED"
	default:
		return fmt.Sprintf("UserStatus(%d)", s)
	}
}

type UserRole int32

const (
	UserRole_UNSPECIFIED UserRole = 0
	UserRole_USER        UserRole = 1
	UserRole_ADMIN       UserRole = 2
	UserRole_SUPERADMIN  UserRole = 3
)

func enumDemo() {
	status := UserStatus_ACTIVE
	role := UserRole_ADMIN
	
	fmt.Printf("Status: %v (%d)\n", status, int(status))
	fmt.Printf("Role: %v (%d)\n", role, int(role))
	
	// Switch on enum
	switch status {
	case UserStatus_ACTIVE:
		fmt.Println("User is active")
	case UserStatus_INACTIVE:
		fmt.Println("User is inactive")
	case UserStatus_DELETED:
		fmt.Println("User is deleted")
	default:
		fmt.Println("Unknown status")
	}
	
	// Convert from string (not generated but useful pattern)
	statusMap := map[string]UserStatus{
		"ACTIVE":      UserStatus_ACTIVE,
		"INACTIVE":    UserStatus_INACTIVE,
		"DELETED":     UserStatus_DELETED,
	}
	
	if s, ok := statusMap["ACTIVE"]; ok {
		fmt.Printf("Parsed: %v\n", s)
	}
}

func main() {
	enumDemo()
}
```

---

## 7. Oneof

```protobuf
// ตัวอย่าง 9: Oneof - only one field can be set

message Notification {
  string user_id = 1;
  string title   = 2;
  string body    = 3;
  
  // Only one of these can be set at a time
  oneof delivery_method {
    EmailDelivery  email  = 4;
    PushDelivery   push   = 5;
    SMSDelivery    sms    = 6;
  }
  
  message EmailDelivery {
    string to      = 1;
    string subject = 2;
    bool   html    = 3;
  }
  
  message PushDelivery {
    string device_token = 1;
    string platform     = 2; // "ios", "android"
  }
  
  message SMSDelivery {
    string phone_number = 1;
    string sender_id    = 2;
  }
}

// Another example: API response
message ApiResponse {
  int32  status_code = 1;
  string message     = 2;
  
  oneof payload {
    User    user    = 3;
    Product product = 4;
    Order   order   = 5;
    bytes   raw     = 6; // fallback
  }
}
```

```go
// ตัวอย่าง 10: Working with oneof in Go

package main

import "fmt"

// Simulated oneof types
type Notification struct {
	UserID string
	Title  string
	Body   string
	
	// Oneof implemented as interface in Go
	DeliveryMethod isNotification_DeliveryMethod
}

type isNotification_DeliveryMethod interface {
	isNotification_DeliveryMethod()
}

type EmailDelivery struct {
	To      string
	Subject string
	HTML    bool
}

type PushDelivery struct {
	DeviceToken string
	Platform    string
}

type SMSDelivery struct {
	PhoneNumber string
	SenderID    string
}

// Implement interface
func (*EmailDelivery) isNotification_DeliveryMethod() {}
func (*PushDelivery) isNotification_DeliveryMethod()  {}
func (*SMSDelivery) isNotification_DeliveryMethod()   {}

// Notification_Email wraps EmailDelivery for oneof
type Notification_Email struct {
	Email *EmailDelivery
}

type Notification_Push struct {
	Push *PushDelivery
}

type Notification_SMS struct {
	SMS *SMSDelivery
}

func (n *Notification_Email) isNotification_DeliveryMethod() {}
func (n *Notification_Push) isNotification_DeliveryMethod()  {}
func (n *Notification_SMS) isNotification_DeliveryMethod()   {}

func sendNotification(n *Notification) error {
	fmt.Printf("Sending notification to user %s: %s\n", n.UserID, n.Title)
	
	switch method := n.DeliveryMethod.(type) {
	case *Notification_Email:
		fmt.Printf("  Via email to: %s (html=%v)\n", method.Email.To, method.Email.HTML)
	case *Notification_Push:
		fmt.Printf("  Via push to device: %s (%s)\n", method.Push.DeviceToken, method.Push.Platform)
	case *Notification_SMS:
		fmt.Printf("  Via SMS to: %s\n", method.SMS.PhoneNumber)
	default:
		fmt.Println("  No delivery method specified")
	}
	
	return nil
}

func oneofDemo() {
	notifications := []*Notification{
		{
			UserID: "user-1",
			Title:  "Welcome!",
			Body:   "Thanks for joining",
			DeliveryMethod: &Notification_Email{
				Email: &EmailDelivery{
					To:      "alice@example.com",
					Subject: "Welcome to our service",
					HTML:    true,
				},
			},
		},
		{
			UserID: "user-2",
			Title:  "New message",
			Body:   "You have a new message",
			DeliveryMethod: &Notification_Push{
				Push: &PushDelivery{
					DeviceToken: "device-token-abc123",
					Platform:    "ios",
				},
			},
		},
		{
			UserID: "user-3",
			Title:  "Verification code",
			Body:   "Your code is: 123456",
			DeliveryMethod: &Notification_SMS{
				SMS: &SMSDelivery{
					PhoneNumber: "+66812345678",
					SenderID:    "MyApp",
				},
			},
		},
	}
	
	for _, n := range notifications {
		sendNotification(n)
	}
}

func main() {
	oneofDemo()
}
```

---

## 8. Maps in Protobuf

```protobuf
// ตัวอย่าง 11: Map fields

message Configuration {
  // map<key_type, value_type> field_name = field_number;
  map<string, string> labels      = 1;
  map<string, int32>  settings    = 2;
  map<string, Feature> features   = 3;
  
  message Feature {
    bool   enabled    = 1;
    string description = 2;
  }
}

message UserPreferences {
  string user_id = 1;
  
  // Theme settings
  map<string, string> theme = 2;
  
  // Notification preferences
  map<string, bool> notifications = 3;
  
  // Language preferences per region
  map<string, string> regional_settings = 4;
}

// Note: Map keys can be: int32, int64, uint32, uint64, sint32, sint64,
// fixed32, fixed64, sfixed32, sfixed64, bool, string
// NOT float, double, bytes, or enum
```

```go
// ตัวอย่าง 12: Working with maps in Go

package main

import "fmt"

// Simulated proto types
type Configuration struct {
	Labels   map[string]string
	Settings map[string]int32
	Features map[string]*Feature
}

type Feature struct {
	Enabled     bool
	Description string
}

func mapDemo() {
	config := &Configuration{
		Labels: map[string]string{
			"env":     "production",
			"version": "1.0",
			"team":    "backend",
		},
		Settings: map[string]int32{
			"max_connections": 100,
			"timeout_ms":     5000,
			"max_retries":    3,
		},
		Features: map[string]*Feature{
			"dark_mode": {Enabled: true, Description: "Dark theme"},
			"beta":      {Enabled: false, Description: "Beta features"},
			"analytics": {Enabled: true, Description: "Usage tracking"},
		},
	}
	
	// Access map fields
	fmt.Printf("Environment: %s\n", config.Labels["env"])
	fmt.Printf("Max connections: %d\n", config.Settings["max_connections"])
	
	// Check feature enabled
	if feature, ok := config.Features["dark_mode"]; ok {
		fmt.Printf("Dark mode: enabled=%v (%s)\n", feature.Enabled, feature.Description)
	}
	
	// Iterate map
	fmt.Println("\nAll features:")
	for name, feature := range config.Features {
		fmt.Printf("  %s: enabled=%v\n", name, feature.Enabled)
	}
	
	// Modify map
	config.Labels["updated"] = "2025-01-01"
	config.Settings["max_connections"] = 200
	
	fmt.Printf("\nUpdated max connections: %d\n", config.Settings["max_connections"])
}

func main() {
	mapDemo()
}
```

---

## 9. Services Definition

```protobuf
// ตัวอย่าง 13: Complete service definitions

syntax = "proto3";

package ecommerce.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

option go_package = "github.com/yourorg/ecommerce/gen/ecommerce/v1";

// Product types
message Product {
  string   id          = 1;
  string   name        = 2;
  string   description = 3;
  double   price       = 4;
  int32    stock       = 5;
  repeated string categories = 6;
  
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

// Request/Response types
message CreateProductRequest {
  string   name        = 1;
  string   description = 2;
  double   price       = 3;
  int32    stock       = 4;
  repeated string categories = 5;
}

message GetProductRequest {
  string id = 1;
}

message UpdateProductRequest {
  string   id          = 1;
  string   name        = 2;
  double   price       = 3;
  int32    stock_delta = 4;
}

message DeleteProductRequest {
  string id = 1;
}

message ListProductsRequest {
  int32  page      = 1;
  int32  page_size = 2;
  string query     = 3;
  string category  = 4;
  
  enum SortBy {
    SORT_BY_UNSPECIFIED = 0;
    SORT_BY_NAME        = 1;
    SORT_BY_PRICE       = 2;
    SORT_BY_CREATED_AT  = 3;
  }
  
  SortBy sort_by   = 5;
  bool   ascending = 6;
}

message ListProductsResponse {
  repeated Product products = 1;
  int32            total    = 2;
  int32            page     = 3;
  int32            pages    = 4;
}

message ProductEvent {
  string product_id  = 1;
  string event_type  = 2; // "created", "updated", "deleted"
  Product product    = 3;
  google.protobuf.Timestamp timestamp = 4;
}

// Product Service
service ProductService {
  // CRUD operations (Unary)
  rpc CreateProduct(CreateProductRequest) returns (Product);
  rpc GetProduct(GetProductRequest) returns (Product);
  rpc UpdateProduct(UpdateProductRequest) returns (Product);
  rpc DeleteProduct(DeleteProductRequest) returns (google.protobuf.Empty);
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
  
  // Server streaming - watch product changes
  rpc WatchProducts(google.protobuf.Empty) returns (stream ProductEvent);
  
  // Client streaming - bulk import
  rpc BulkCreateProducts(stream CreateProductRequest) returns (ListProductsResponse);
  
  // Bidirectional - real-time inventory updates
  rpc SyncInventory(stream UpdateProductRequest) returns (stream Product);
}
```

---

## 10. Well-Known Types

```protobuf
// ตัวอย่าง 14: Using Well-Known Types

syntax = "proto3";

import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/any.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/empty.proto";

message ApiRequest {
  // Timestamp (instead of int64)
  google.protobuf.Timestamp created_at = 1;
  
  // Duration
  google.protobuf.Duration timeout = 2;
  
  // Nullable string (wrapper allows null)
  google.protobuf.StringValue middle_name = 3;
  google.protobuf.Int32Value optional_age = 4;
  google.protobuf.BoolValue verified = 5;
  
  // Any - can hold any proto message
  google.protobuf.Any payload = 6;
  
  // Struct - arbitrary JSON-like structure
  google.protobuf.Struct metadata = 7;
}

// Empty message (for RPCs with no input/output)
// service Example {
//   rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
// }
```

```go
// ตัวอย่าง 15: Working with Well-Known Types

package main

import (
	"fmt"
	"time"
	
	"google.golang.org/protobuf/types/known/timestamppb"
	"google.golang.org/protobuf/types/known/durationpb"
	"google.golang.org/protobuf/types/known/wrapperspb"
	"google.golang.org/protobuf/types/known/structpb"
	"google.golang.org/protobuf/types/known/anypb"
)

func wellKnownTypesDemo() {
	// Timestamp
	now := time.Now()
	ts := timestamppb.New(now)
	fmt.Printf("Timestamp: %v\n", ts.AsTime())
	
	// Duration
	d := durationpb.New(5 * time.Minute)
	fmt.Printf("Duration: %v\n", d.AsDuration())
	
	// Wrappers (nullable values)
	var name *wrapperspb.StringValue = nil // represents null
	_ = name
	
	nameVal := wrapperspb.String("Alice") // has value
	fmt.Printf("Name: %s\n", nameVal.GetValue())
	
	ageVal := wrapperspb.Int32(30)
	fmt.Printf("Age: %d\n", ageVal.GetValue())
	
	// Struct (JSON-like)
	data, _ := structpb.NewStruct(map[string]interface{}{
		"name":   "Alice",
		"age":    30,
		"active": true,
		"scores": []interface{}{85, 92, 78},
	})
	fmt.Printf("Struct keys: %v\n", getStructKeys(data))
	
	// Any
	_ = anypb.New
	fmt.Println("Any type can hold any proto message")
}

func getStructKeys(s *structpb.Struct) []string {
	keys := make([]string, 0, len(s.Fields))
	for k := range s.Fields {
		keys = append(keys, k)
	}
	return keys
}

func main() {
	wellKnownTypesDemo()
}
```

---

## 11. Proto Versioning and Compatibility

```protobuf
// ตัวอย่าง 16: Backward compatibility rules

// Version 1
message UserV1 {
  int32  id    = 1;
  string name  = 2;
  string email = 3;
}

// Version 2 - SAFE changes (backward compatible)
message UserV2 {
  int32  id       = 1;  // KEEP same field number
  string name     = 2;  // KEEP same field name and type
  string email    = 3;  // KEEP existing fields
  
  // SAFE: Add new optional fields
  string phone    = 4;
  string address  = 5;
  
  // SAFE: Add new message fields
  Profile profile = 6;
  
  message Profile {
    string bio    = 1;
    string avatar = 2;
  }
}

// BREAKING changes (avoid!):
// 1. Change field number
// 2. Change field type
// 3. Remove field (use reserved instead)
// 4. Rename fields that are used for JSON serialization

// When removing a field: use reserved
message UserV3 {
  int32  id      = 1;
  string name    = 2;
  string email   = 3;
  string phone   = 4;
  // Removed: address (was field 5)
  
  // Reserve to prevent reuse
  reserved 5;
  reserved "address";
  
  // Safe to add new fields
  string timezone = 6;
}
```

---

## 12. Code Generation

```bash
# ตัวอย่าง 17: protoc commands

# Basic generation
protoc \
  --proto_path=proto \
  --go_out=gen \
  --go_opt=paths=source_relative \
  --go-grpc_out=gen \
  --go-grpc_opt=paths=source_relative \
  proto/**/*.proto

# With Google APIs
protoc \
  --proto_path=proto \
  --proto_path=vendor/github.com/googleapis/googleapis \
  --go_out=gen \
  --go_opt=paths=source_relative \
  --go-grpc_out=gen \
  --go-grpc_opt=paths=source_relative \
  proto/myapp/*.proto

# Generate with buf (modern alternative to protoc)
# buf.gen.yaml:
# version: v1
# plugins:
#   - plugin: go
#     out: gen
#     opt: paths=source_relative
#   - plugin: go-grpc
#     out: gen
#     opt: paths=source_relative

# buf generate
```

```go
// ตัวอย่าง 18: Buf configuration (buf.yaml and buf.gen.yaml)

/*
# buf.yaml
version: v1
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT

# buf.gen.yaml
version: v1
plugins:
  - plugin: go
    out: gen/go
    opt:
      - paths=source_relative
  - plugin: go-grpc
    out: gen/go
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
  - plugin: grpc-gateway
    out: gen/go
    opt:
      - paths=source_relative
  - plugin: openapiv2
    out: gen/swagger
*/

package main

import "fmt"

func bufToolsDemo() {
	fmt.Println("Modern proto tooling with buf:")
	fmt.Println()
	fmt.Println("1. buf lint    - lint proto files")
	fmt.Println("2. buf build   - compile proto files")
	fmt.Println("3. buf generate - generate code")
	fmt.Println("4. buf breaking - check for breaking changes")
	fmt.Println("5. buf push    - push to Buf Schema Registry")
}

func main() {
	bufToolsDemo()
}
```

---

## 13. JSON Representation

```go
// ตัวอย่าง 19: Proto JSON marshaling

package main

import (
	"fmt"
	
	"google.golang.org/protobuf/encoding/protojson"
	"google.golang.org/protobuf/proto"
)

// Proto to JSON (using protojson - NOT encoding/json)
func protoJSONDemo() {
	// In real code, use generated types
	// p := &pb.User{Id: 1, Name: "Alice", Email: "alice@example.com"}
	
	// Marshal options
	marshaler := protojson.MarshalOptions{
		Indent:          "  ",   // pretty print
		EmitUnpopulated: false,  // don't include empty fields
		UseProtoNames:   false,  // use camelCase (JSON convention)
		// UseProtoNames: true  // use snake_case (proto convention)
	}
	_ = marshaler
	
	// Unmarshal options
	unmarshaler := protojson.UnmarshalOptions{
		DiscardUnknown: true, // ignore unknown fields
	}
	_ = unmarshaler
	
	fmt.Println("Proto JSON marshaling:")
	fmt.Println("  Use protojson package (not encoding/json)")
	fmt.Println("  Handles timestamp, duration, etc. correctly")
	fmt.Println("  Supports pretty printing")
	fmt.Println("  Field names are camelCase by default")
	
	_ = proto.Marshal
}

// ตัวอย่าง 20: Complete proto workflow example

// proto/user/user.proto:
const protoFileContent = `
syntax = "proto3";
package user.v1;
option go_package = "github.com/example/app/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

message User {
  int32  id    = 1;
  string name  = 2;
  string email = 3;
  
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;
  
  UserStatus status = 6;
  
  enum UserStatus {
    USER_STATUS_UNSPECIFIED = 0;
    USER_STATUS_ACTIVE      = 1;
    USER_STATUS_INACTIVE    = 2;
  }
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
}

message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name     = 1;
  string email    = 2;
  string password = 3;
}
`

func protoWorkflowDemo() {
	fmt.Println("=== Proto Workflow ===")
	fmt.Println()
	fmt.Println("1. Write .proto file")
	fmt.Println("2. Run: buf generate (or protoc)")
	fmt.Println("3. Generated files:")
	fmt.Println("   - gen/user/v1/user.pb.go   (message types)")
	fmt.Println("   - gen/user/v1/user_grpc.pb.go (service interface)")
	fmt.Println("4. Implement service interface")
	fmt.Println("5. Register with gRPC server")
	fmt.Println()
	fmt.Printf("Proto file:\n%s\n", protoFileContent[:200]+"...")
}

func main() {
	protoJSONDemo()
	protoWorkflowDemo()
}
```

---

## สรุป

ใน Part 43 เราได้เรียนรู้:

1. **Proto3 Syntax**: message types, field definitions, packages
2. **Field Types**: scalar types, field numbers, reserved fields
3. **Nested Messages**: embedded types สำหรับ complex data
4. **Enums**: type-safe constants
5. **Oneof**: exactly one field from a set
6. **Maps**: key-value structures
7. **Services**: RPC method definitions
8. **Well-Known Types**: Timestamp, Duration, Wrappers, Any, Struct
9. **Versioning**: backward compatibility rules
10. **Code Generation**: protoc and buf tools
11. **JSON Marshaling**: protojson package

---

## Resources

- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [Protocol Buffers Go Tutorial](https://protobuf.dev/getting-started/gotutorial/)
- [buf - Modern Proto Tooling](https://buf.build/)
- [Well-Known Types](https://protobuf.dev/reference/protobuf/google.protobuf/)
- [Google API Design Guide](https://cloud.google.com/apis/design)
