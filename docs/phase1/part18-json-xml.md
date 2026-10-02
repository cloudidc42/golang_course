# Part 18: JSON, XML และ Data Serialization ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- Marshal/Unmarshal JSON ด้วย `encoding/json`
- ใช้ JSON struct tags
- สร้าง Custom JSON marshaling/unmarshaling
- ใช้ JSON Streaming กับ Encoder/Decoder
- ทำงานกับ XML ด้วย `encoding/xml`
- ใช้งาน YAML ด้วย `gopkg.in/yaml.v3`
- จัดการ unknown JSON fields
- Validate JSON

---

## 18.1 encoding/json พื้นฐาน

### 18.1.1 Marshal - แปลง Go struct เป็น JSON

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    Name    string `json:"name"`
    Age     int    `json:"age"`
    Email   string `json:"email"`
    IsAdmin bool   `json:"is_admin"`
}

func main() {
    p := Person{
        Name:    "สมชาย ใจดี",
        Age:     30,
        Email:   "somchai@example.com",
        IsAdmin: false,
    }
    
    // Marshal - แปลงเป็น JSON (compact)
    data, err := json.Marshal(p)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    fmt.Printf("JSON: %s\n", data)
    
    // MarshalIndent - แปลงแบบ pretty print
    pretty, err := json.MarshalIndent(p, "", "  ")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    fmt.Printf("Pretty JSON:\n%s\n", pretty)
    
    // Marshal slice
    people := []Person{
        {Name: "Alice", Age: 25, Email: "alice@example.com"},
        {Name: "Bob", Age: 30, Email: "bob@example.com"},
    }
    
    peopleJSON, _ := json.MarshalIndent(people, "", "  ")
    fmt.Printf("\nSlice JSON:\n%s\n", peopleJSON)
    
    // Marshal map
    data2, _ := json.MarshalIndent(map[string]interface{}{
        "key1": "value1",
        "key2": 42,
        "key3": true,
        "key4": nil,
    }, "", "  ")
    fmt.Printf("\nMap JSON:\n%s\n", data2)
}
```

### 18.1.2 Unmarshal - แปลง JSON เป็น Go struct

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Address struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    Country string `json:"country"`
    Zip     string `json:"zip"`
}

type User struct {
    ID      int      `json:"id"`
    Name    string   `json:"name"`
    Email   string   `json:"email"`
    Age     int      `json:"age"`
    Address Address  `json:"address"`
    Tags    []string `json:"tags"`
}

func main() {
    jsonStr := `{
        "id": 1,
        "name": "Alice Johnson",
        "email": "alice@example.com",
        "age": 28,
        "address": {
            "street": "123 Main St",
            "city": "Bangkok",
            "country": "Thailand",
            "zip": "10110"
        },
        "tags": ["developer", "golang", "cloud"]
    }`
    
    var user User
    err := json.Unmarshal([]byte(jsonStr), &user)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("User: %+v\n", user)
    fmt.Printf("Name: %s\n", user.Name)
    fmt.Printf("City: %s\n", user.Address.City)
    fmt.Printf("Tags: %v\n", user.Tags)
    
    // Unmarshal array
    jsonArr := `[
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
        {"id": 3, "name": "Charlie"}
    ]`
    
    var users []User
    json.Unmarshal([]byte(jsonArr), &users)
    
    fmt.Printf("\nUsers (%d):\n", len(users))
    for _, u := range users {
        fmt.Printf("  ID=%d, Name=%s\n", u.ID, u.Name)
    }
    
    // Unmarshal เป็น map
    var rawMap map[string]interface{}
    json.Unmarshal([]byte(jsonStr), &rawMap)
    
    fmt.Printf("\nRaw map keys: ")
    for k := range rawMap {
        fmt.Printf("%s ", k)
    }
    fmt.Println()
}
```

### 18.1.3 JSON Struct Tags

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    // omitempty - ไม่แสดงถ้าค่าเป็น zero value
    Description string  `json:"description,omitempty"`
    Price       float64 `json:"price"`
    // string - แปลงเป็น string แทน number
    Stock       int     `json:"stock,string"`
    // - ไม่รวมใน JSON
    internalID  string  `json:"-"`
    // ชื่อต่างจาก field
    CreatedAt   string  `json:"created_at"`
}

type Config struct {
    // ใช้ได้หลาย options
    Host     string `json:"host"`
    Port     int    `json:"port,omitempty"`
    Debug    bool   `json:"debug,omitempty"`
    Password string `json:"-"` // ไม่ serialize password
    // ไม่มี tag → ใช้ชื่อ field ตรงๆ
    APIKey   string
}

func main() {
    // Product ที่มีค่าครบ
    p1 := Product{
        ID:          1,
        Name:        "iPhone 15",
        Description: "Latest iPhone",
        Price:       35000.00,
        Stock:       50,
        CreatedAt:   "2024-01-01",
    }
    
    data1, _ := json.MarshalIndent(p1, "", "  ")
    fmt.Printf("Product with description:\n%s\n", data1)
    
    // Product ที่ description เป็น empty string
    p2 := Product{
        ID:        2,
        Name:      "Accessory",
        Price:     500.00,
        Stock:     100,
        CreatedAt: "2024-01-01",
        // Description ไม่กำหนด → omitted เพราะ omitempty
    }
    
    data2, _ := json.MarshalIndent(p2, "", "  ")
    fmt.Printf("\nProduct without description:\n%s\n", data2)
    
    // Config
    cfg := Config{
        Host:     "localhost",
        Port:     0,    // zero value → omitted
        Debug:    false, // zero value → omitted
        Password: "secret",
        APIKey:   "abc123",
    }
    
    cfgData, _ := json.MarshalIndent(cfg, "", "  ")
    fmt.Printf("\nConfig:\n%s\n", cfgData)
    
    // Unmarshal กับ string tag
    jsonStr := `{"id":1,"name":"test","price":100.0,"stock":"25","created_at":"2024-01-01"}`
    var p3 Product
    json.Unmarshal([]byte(jsonStr), &p3)
    fmt.Printf("\nUnmarshaled Stock: %d\n", p3.Stock)
}
```

---

## 18.2 Custom JSON Marshaling/Unmarshaling

### 18.2.1 Implement json.Marshaler

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
    "time"
)

// Custom time format
type CustomTime struct {
    time.Time
}

func (ct CustomTime) MarshalJSON() ([]byte, error) {
    return json.Marshal(ct.Time.Format("2006-01-02"))
}

func (ct *CustomTime) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    t, err := time.Parse("2006-01-02", s)
    if err != nil {
        return err
    }
    ct.Time = t
    return nil
}

// Custom type - Celsius
type Celsius float64

func (c Celsius) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`{"value": %.1f, "unit": "celsius"}`, c)), nil
}

func (c *Celsius) UnmarshalJSON(data []byte) error {
    var m struct {
        Value float64 `json:"value"`
        Unit  string  `json:"unit"`
    }
    if err := json.Unmarshal(data, &m); err != nil {
        // ลอง parse เป็น float ตรงๆ
        var f float64
        if err2 := json.Unmarshal(data, &f); err2 != nil {
            return err
        }
        *c = Celsius(f)
        return nil
    }
    
    switch strings.ToLower(m.Unit) {
    case "celsius":
        *c = Celsius(m.Value)
    case "fahrenheit":
        *c = Celsius((m.Value - 32) * 5 / 9)
    default:
        return fmt.Errorf("unknown unit: %s", m.Unit)
    }
    return nil
}

type WeatherReport struct {
    Date        CustomTime `json:"date"`
    Temperature Celsius    `json:"temperature"`
    Location    string     `json:"location"`
}

func main() {
    report := WeatherReport{
        Date:        CustomTime{time.Date(2024, 6, 15, 0, 0, 0, 0, time.UTC)},
        Temperature: Celsius(35.5),
        Location:    "Bangkok",
    }
    
    data, err := json.MarshalIndent(report, "", "  ")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    fmt.Printf("Marshaled:\n%s\n", data)
    
    // Unmarshal
    jsonStr := `{
        "date": "2024-07-20",
        "temperature": {"value": 38.0, "unit": "celsius"},
        "location": "Chiang Mai"
    }`
    
    var report2 WeatherReport
    err = json.Unmarshal([]byte(jsonStr), &report2)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("\nUnmarshaled:\n")
    fmt.Printf("Date: %s\n", report2.Date.Format("January 2, 2006"))
    fmt.Printf("Temperature: %.1f°C\n", report2.Temperature)
    fmt.Printf("Location: %s\n", report2.Location)
}
```

### 18.2.2 Interface กับ JSON

```go
package main

import (
    "encoding/json"
    "fmt"
)

// Polymorphic JSON - เก็บ type ลงใน JSON
type Shape interface {
    Area() float64
    Type() string
}

type Circle struct {
    Radius float64 `json:"radius"`
}

func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }
func (c Circle) Type() string  { return "circle" }

type Rectangle struct {
    Width  float64 `json:"width"`
    Height float64 `json:"height"`
}

func (r Rectangle) Area() float64 { return r.Width * r.Height }
func (r Rectangle) Type() string  { return "rectangle" }

type ShapeWrapper struct {
    ShapeType string          `json:"type"`
    Data      json.RawMessage `json:"data"`
}

func wrapShape(s Shape) ([]byte, error) {
    data, err := json.Marshal(s)
    if err != nil {
        return nil, err
    }
    return json.Marshal(ShapeWrapper{
        ShapeType: s.Type(),
        Data:      json.RawMessage(data),
    })
}

func unwrapShape(data []byte) (Shape, error) {
    var wrapper ShapeWrapper
    if err := json.Unmarshal(data, &wrapper); err != nil {
        return nil, err
    }
    
    switch wrapper.ShapeType {
    case "circle":
        var c Circle
        if err := json.Unmarshal(wrapper.Data, &c); err != nil {
            return nil, err
        }
        return c, nil
    case "rectangle":
        var r Rectangle
        if err := json.Unmarshal(wrapper.Data, &r); err != nil {
            return nil, err
        }
        return r, nil
    default:
        return nil, fmt.Errorf("unknown shape type: %s", wrapper.ShapeType)
    }
}

func main() {
    shapes := []Shape{
        Circle{Radius: 5.0},
        Rectangle{Width: 4.0, Height: 6.0},
        Circle{Radius: 3.0},
    }
    
    fmt.Println("=== Serialize Shapes ===")
    var serialized [][]byte
    for _, s := range shapes {
        data, err := wrapShape(s)
        if err != nil {
            fmt.Printf("error: %v\n", err)
            continue
        }
        serialized = append(serialized, data)
        fmt.Printf("%s\n", data)
    }
    
    fmt.Println("\n=== Deserialize Shapes ===")
    for _, data := range serialized {
        s, err := unwrapShape(data)
        if err != nil {
            fmt.Printf("error: %v\n", err)
            continue
        }
        fmt.Printf("Type: %-12s Area: %.2f\n", s.Type(), s.Area())
    }
}
```

---

## 18.3 JSON Streaming

### 18.3.1 json.Encoder และ json.Decoder

```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "os"
    "strings"
)

type LogEntry struct {
    Level   string            `json:"level"`
    Message string            `json:"message"`
    Fields  map[string]string `json:"fields,omitempty"`
}

func main() {
    // Encoder - เขียน JSON ทีละ object
    f, _ := os.CreateTemp("", "logs-*.jsonl")
    defer os.Remove(f.Name())
    defer f.Close()
    
    encoder := json.NewEncoder(f)
    
    logs := []LogEntry{
        {Level: "INFO", Message: "Server started", Fields: map[string]string{"port": "8080"}},
        {Level: "DEBUG", Message: "Processing request", Fields: map[string]string{"path": "/api/users"}},
        {Level: "ERROR", Message: "Database error", Fields: map[string]string{"err": "connection refused"}},
        {Level: "INFO", Message: "Request completed", Fields: map[string]string{"status": "200", "latency": "10ms"}},
    }
    
    for _, log := range logs {
        if err := encoder.Encode(log); err != nil {
            fmt.Printf("encode error: %v\n", err)
        }
    }
    
    fmt.Println("=== เขียน JSONL สำเร็จ ===")
    
    // อ่านกลับมา
    f.Seek(0, 0)
    decoder := json.NewDecoder(f)
    
    fmt.Println("\n=== อ่าน JSONL ===")
    for decoder.More() {
        var entry LogEntry
        if err := decoder.Decode(&entry); err != nil {
            fmt.Printf("decode error: %v\n", err)
            continue
        }
        fmt.Printf("[%s] %s\n", entry.Level, entry.Message)
        for k, v := range entry.Fields {
            fmt.Printf("  %s=%s\n", k, v)
        }
    }
    
    // Decoder กับ stream
    fmt.Println("\n=== Stream Decoder ===")
    stream := `{"name":"Alice","age":30}
{"name":"Bob","age":25}
{"name":"Charlie","age":35}`
    
    decoder2 := json.NewDecoder(strings.NewReader(stream))
    for decoder2.More() {
        var m map[string]interface{}
        decoder2.Decode(&m)
        fmt.Printf("  name=%s, age=%.0f\n", m["name"], m["age"])
    }
    
    // UseNumber - ป้องกัน float64 precision loss
    fmt.Println("\n=== UseNumber ===")
    jsonWithBigInt := `{"id": 9007199254740993, "amount": 1234567890.123456789}`
    
    decoder3 := json.NewDecoder(strings.NewReader(jsonWithBigInt))
    decoder3.UseNumber()
    
    var data map[string]json.Number
    decoder3.Decode(&data)
    
    id := data["id"]
    fmt.Printf("ID as Number: %s\n", id)
    idInt, _ := id.Int64()
    fmt.Printf("ID as Int64: %d\n", idInt)
    
    amount := data["amount"]
    amountFloat, _ := amount.Float64()
    fmt.Printf("Amount: %.6f\n", amountFloat)
    
    // Disallow unknown fields
    fmt.Println("\n=== DisallowUnknownFields ===")
    
    type StrictUser struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    
    decoder4 := json.NewDecoder(strings.NewReader(`{"name":"Alice","email":"alice@test.com","unknown_field":"oops"}`))
    decoder4.DisallowUnknownFields()
    
    var u StrictUser
    if err := decoder4.Decode(&u); err != nil {
        fmt.Printf("Error (expected): %v\n", err)
    } else {
        fmt.Printf("User: %+v\n", u)
    }
    
    _ = bufio.NewReader(nil) // suppress import
}
```

### 18.3.2 json.RawMessage

```go
package main

import (
    "encoding/json"
    "fmt"
)

// เก็บ JSON ดิบ เพื่อ delay parsing
type APIResponse struct {
    Status  string          `json:"status"`
    Code    int             `json:"code"`
    Data    json.RawMessage `json:"data"`
    Message string          `json:"message"`
}

type UserData struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

type ProductData struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func handleResponse(resp APIResponse, dataType string) {
    fmt.Printf("Status: %s, Code: %d\n", resp.Status, resp.Code)
    
    switch dataType {
    case "user":
        var user UserData
        if err := json.Unmarshal(resp.Data, &user); err != nil {
            fmt.Printf("error: %v\n", err)
            return
        }
        fmt.Printf("User: ID=%d, Name=%s\n", user.ID, user.Name)
        
    case "product":
        var product ProductData
        if err := json.Unmarshal(resp.Data, &product); err != nil {
            fmt.Printf("error: %v\n", err)
            return
        }
        fmt.Printf("Product: ID=%d, Name=%s, Price=%.2f\n",
            product.ID, product.Name, product.Price)
    }
}

func main() {
    // JSON ที่ data field เป็น user
    userJSON := `{
        "status": "success",
        "code": 200,
        "data": {"id": 1, "name": "Alice"},
        "message": "User found"
    }`
    
    var userResp APIResponse
    json.Unmarshal([]byte(userJSON), &userResp)
    handleResponse(userResp, "user")
    
    // JSON ที่ data field เป็น product
    productJSON := `{
        "status": "success",
        "code": 200,
        "data": {"id": 101, "name": "iPhone 15", "price": 35000.0},
        "message": "Product found"
    }`
    
    var productResp APIResponse
    json.Unmarshal([]byte(productJSON), &productResp)
    handleResponse(productResp, "product")
    
    // ใช้ RawMessage เพื่อสร้าง JSON
    fmt.Println("\n=== Build JSON with RawMessage ===")
    
    rawName := json.RawMessage(`"Alice Johnson"`)
    rawData := json.RawMessage(`{"key": "value", "nested": true}`)
    
    combined := struct {
        Name string          `json:"name"`
        Meta json.RawMessage `json:"meta"`
    }{
        Name: string(rawName[1 : len(rawName)-1]), // remove quotes
        Meta: rawData,
    }
    
    out, _ := json.MarshalIndent(combined, "", "  ")
    fmt.Println(string(out))
}
```

---

## 18.4 Handling Unknown Fields

### 18.4.1 Map สำหรับ Unknown Fields

```go
package main

import (
    "encoding/json"
    "fmt"
)

// Struct ที่รับ known fields + เก็บที่เหลือไว้ใน map
type FlexibleConfig struct {
    Name    string `json:"name"`
    Version string `json:"version"`
    // เก็บ fields อื่นๆ
    Extra map[string]json.RawMessage `json:"-"`
}

func (fc *FlexibleConfig) UnmarshalJSON(data []byte) error {
    // อ่านทุก field เป็น raw
    var raw map[string]json.RawMessage
    if err := json.Unmarshal(data, &raw); err != nil {
        return err
    }
    
    // Parse known fields
    if v, ok := raw["name"]; ok {
        json.Unmarshal(v, &fc.Name)
        delete(raw, "name")
    }
    if v, ok := raw["version"]; ok {
        json.Unmarshal(v, &fc.Version)
        delete(raw, "version")
    }
    
    // เก็บที่เหลือ
    fc.Extra = raw
    return nil
}

func main() {
    jsonStr := `{
        "name": "myapp",
        "version": "1.0.0",
        "debug": true,
        "workers": 4,
        "database": {
            "host": "localhost",
            "port": 5432
        },
        "features": ["featureA", "featureB"]
    }`
    
    var cfg FlexibleConfig
    if err := json.Unmarshal([]byte(jsonStr), &cfg); err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("Name: %s\n", cfg.Name)
    fmt.Printf("Version: %s\n", cfg.Version)
    fmt.Println("\nExtra fields:")
    for k, v := range cfg.Extra {
        fmt.Printf("  %s = %s\n", k, v)
    }
    
    // อ่าน extra field เฉพาะ
    if debug, ok := cfg.Extra["debug"]; ok {
        var d bool
        json.Unmarshal(debug, &d)
        fmt.Printf("\nDebug: %v\n", d)
    }
    
    if db, ok := cfg.Extra["database"]; ok {
        var dbConfig map[string]interface{}
        json.Unmarshal(db, &dbConfig)
        fmt.Printf("Database host: %v\n", dbConfig["host"])
    }
}
```

---

## 18.5 encoding/xml

### 18.5.1 XML Marshal/Unmarshal

```go
package main

import (
    "encoding/xml"
    "fmt"
)

type Book struct {
    XMLName  xml.Name `xml:"book"`
    ID       int      `xml:"id,attr"`
    Title    string   `xml:"title"`
    Author   string   `xml:"author"`
    Year     int      `xml:"year"`
    Price    float64  `xml:"price"`
    Tags     []string `xml:"tags>tag"`
    InStock  bool     `xml:"instock"`
}

type Library struct {
    XMLName xml.Name `xml:"library"`
    Name    string   `xml:"name,attr"`
    Books   []Book   `xml:"book"`
}

func main() {
    library := Library{
        Name: "City Library",
        Books: []Book{
            {
                ID:      1,
                Title:   "The Go Programming Language",
                Author:  "Alan Donovan",
                Year:    2015,
                Price:   45.99,
                Tags:    []string{"programming", "golang"},
                InStock: true,
            },
            {
                ID:      2,
                Title:   "Go in Action",
                Author:  "William Kennedy",
                Year:    2015,
                Price:   39.99,
                Tags:    []string{"programming", "golang", "concurrency"},
                InStock: false,
            },
        },
    }
    
    // Marshal
    output, err := xml.MarshalIndent(library, "", "  ")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    // เพิ่ม XML declaration
    fmt.Printf("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n%s\n", output)
    
    // Unmarshal
    xmlStr := `<?xml version="1.0" encoding="UTF-8"?>
<library name="Tech Library">
    <book id="1">
        <title>Learning Go</title>
        <author>Jon Bodner</author>
        <year>2021</year>
        <price>49.99</price>
        <tags><tag>golang</tag><tag>programming</tag></tags>
        <instock>true</instock>
    </book>
</library>`
    
    var lib2 Library
    err = xml.Unmarshal([]byte(xmlStr), &lib2)
    if err != nil {
        fmt.Printf("unmarshal error: %v\n", err)
        return
    }
    
    fmt.Printf("\nLibrary: %s\n", lib2.Name)
    for _, b := range lib2.Books {
        fmt.Printf("  Book: %s by %s (%.2f)\n", b.Title, b.Author, b.Price)
        fmt.Printf("    Tags: %v\n", b.Tags)
    }
}
```

### 18.5.2 XML Streaming

```go
package main

import (
    "encoding/xml"
    "fmt"
    "strings"
)

type Item struct {
    XMLName xml.Name `xml:"item"`
    ID      int      `xml:"id,attr"`
    Name    string   `xml:"name"`
    Value   float64  `xml:"value"`
}

func main() {
    xmlData := `<items>
        <item id="1"><name>Apple</name><value>1.5</value></item>
        <item id="2"><name>Banana</name><value>0.75</value></item>
        <item id="3"><name>Cherry</name><value>3.0</value></item>
        <item id="4"><name>Date</name><value>5.5</value></item>
    </items>`
    
    decoder := xml.NewDecoder(strings.NewReader(xmlData))
    
    fmt.Println("=== XML Streaming ===")
    for {
        tok, err := decoder.Token()
        if err != nil {
            break
        }
        
        switch t := tok.(type) {
        case xml.StartElement:
            if t.Name.Local == "item" {
                var item Item
                if err := decoder.DecodeElement(&item, &t); err != nil {
                    fmt.Printf("decode error: %v\n", err)
                    continue
                }
                fmt.Printf("Item %d: %s = %.2f\n", item.ID, item.Name, item.Value)
            }
        }
    }
    
    // XML Encoder
    fmt.Println("\n=== XML Encoder ===")
    var sb strings.Builder
    encoder := xml.NewEncoder(&sb)
    encoder.Indent("", "  ")
    
    items := []Item{
        {ID: 10, Name: "Elderberry", Value: 4.25},
        {ID: 11, Name: "Fig", Value: 2.00},
    }
    
    encoder.EncodeToken(xml.StartElement{Name: xml.Name{Local: "items"}})
    for _, item := range items {
        encoder.Encode(item)
    }
    encoder.EncodeToken(xml.EndElement{Name: xml.Name{Local: "items"}})
    encoder.Flush()
    
    fmt.Println(sb.String())
}
```

---

## 18.6 YAML (gopkg.in/yaml.v3)

### 18.6.1 YAML พื้นฐาน

```go
package main

import (
    "fmt"
    // ต้อง go get gopkg.in/yaml.v3
    // "gopkg.in/yaml.v3"
    "encoding/json" // ใช้ JSON แทนสาธิต
)

// โครงสร้างเหมือนกัน แต่ใช้ yaml tags
type AppConfig struct {
    App struct {
        Name    string `yaml:"name" json:"name"`
        Version string `yaml:"version" json:"version"`
        Debug   bool   `yaml:"debug" json:"debug"`
    } `yaml:"app" json:"app"`
    
    Server struct {
        Host string `yaml:"host" json:"host"`
        Port int    `yaml:"port" json:"port"`
        TLS  bool   `yaml:"tls" json:"tls"`
    } `yaml:"server" json:"server"`
    
    Database struct {
        Driver   string `yaml:"driver" json:"driver"`
        Host     string `yaml:"host" json:"host"`
        Port     int    `yaml:"port" json:"port"`
        Name     string `yaml:"name" json:"name"`
        User     string `yaml:"user" json:"user"`
        Password string `yaml:"password" json:"password"`
        MaxConns int    `yaml:"max_conns" json:"max_conns"`
    } `yaml:"database" json:"database"`
    
    Features []string `yaml:"features" json:"features"`
    
    Logging struct {
        Level  string `yaml:"level" json:"level"`
        Format string `yaml:"format" json:"format"`
    } `yaml:"logging" json:"logging"`
}

func main() {
    // YAML config string (ตัวอย่าง - ในโปรเจกต์จริงต้อง import yaml.v3)
    yamlExample := `
# Application Config
app:
  name: my-service
  version: "2.0.0"
  debug: false

server:
  host: 0.0.0.0
  port: 8080
  tls: true

database:
  driver: postgres
  host: db.example.com
  port: 5432
  name: mydb
  user: dbuser
  password: "secret123"
  max_conns: 10

features:
  - feature_a
  - feature_b
  - experimental_mode

logging:
  level: info
  format: json
`
    fmt.Println("YAML Config Example:")
    fmt.Println(yamlExample)
    
    // สาธิตการใช้ JSON แทน (โครงสร้างเดียวกัน)
    jsonConfig := `{
        "app": {"name": "my-service", "version": "2.0.0", "debug": false},
        "server": {"host": "0.0.0.0", "port": 8080, "tls": true},
        "database": {"driver": "postgres", "host": "db.example.com", "port": 5432, "name": "mydb", "user": "dbuser", "password": "secret123", "max_conns": 10},
        "features": ["feature_a", "feature_b"],
        "logging": {"level": "info", "format": "json"}
    }`
    
    var cfg AppConfig
    if err := json.Unmarshal([]byte(jsonConfig), &cfg); err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("App: %s v%s (debug: %v)\n", cfg.App.Name, cfg.App.Version, cfg.App.Debug)
    fmt.Printf("Server: %s:%d (TLS: %v)\n", cfg.Server.Host, cfg.Server.Port, cfg.Server.TLS)
    fmt.Printf("Database: %s@%s:%d/%s (max: %d)\n",
        cfg.Database.User, cfg.Database.Host, cfg.Database.Port,
        cfg.Database.Name, cfg.Database.MaxConns)
    fmt.Printf("Features: %v\n", cfg.Features)
    fmt.Printf("Logging: level=%s, format=%s\n", cfg.Logging.Level, cfg.Logging.Format)
}
```

---

## 18.7 JSON Validation

### 18.7.1 Validate JSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

func isValidJSON(s string) bool {
    var js json.RawMessage
    return json.Unmarshal([]byte(s), &js) == nil
}

type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error: field '%s' - %s", e.Field, e.Message)
}

type UserInput struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
    Role  string `json:"role"`
}

func validateUser(u UserInput) []ValidationError {
    var errors []ValidationError
    
    if strings.TrimSpace(u.Name) == "" {
        errors = append(errors, ValidationError{Field: "name", Message: "ต้องไม่ว่างเปล่า"})
    }
    if len(u.Name) < 2 {
        errors = append(errors, ValidationError{Field: "name", Message: "ต้องมีอย่างน้อย 2 ตัวอักษร"})
    }
    
    if !strings.Contains(u.Email, "@") {
        errors = append(errors, ValidationError{Field: "email", Message: "รูปแบบไม่ถูกต้อง"})
    }
    
    if u.Age < 18 || u.Age > 120 {
        errors = append(errors, ValidationError{Field: "age", Message: "ต้องอยู่ระหว่าง 18-120"})
    }
    
    validRoles := map[string]bool{"admin": true, "user": true, "moderator": true}
    if !validRoles[u.Role] {
        errors = append(errors, ValidationError{Field: "role", Message: fmt.Sprintf("ต้องเป็น admin, user, หรือ moderator (ได้รับ: %s)", u.Role)})
    }
    
    return errors
}

func processUserJSON(jsonStr string) error {
    // ตรวจสอบ JSON ก่อน
    if !isValidJSON(jsonStr) {
        return fmt.Errorf("JSON ไม่ถูกต้อง")
    }
    
    // Parse
    var user UserInput
    if err := json.Unmarshal([]byte(jsonStr), &user); err != nil {
        return fmt.Errorf("parse error: %w", err)
    }
    
    // Validate
    if errors := validateUser(user); len(errors) > 0 {
        msgs := make([]string, len(errors))
        for i, e := range errors {
            msgs[i] = e.Error()
        }
        return fmt.Errorf("validation failed:\n  %s", strings.Join(msgs, "\n  "))
    }
    
    fmt.Printf("User valid: %+v\n", user)
    return nil
}

func main() {
    tests := []struct {
        name string
        json string
    }{
        {
            "Valid user",
            `{"name": "Alice", "email": "alice@test.com", "age": 25, "role": "user"}`,
        },
        {
            "Invalid JSON",
            `{"name": "Bob", "email":}`,
        },
        {
            "Invalid fields",
            `{"name": "A", "email": "notanemail", "age": 15, "role": "superadmin"}`,
        },
        {
            "Empty name",
            `{"name": "", "email": "test@test.com", "age": 30, "role": "user"}`,
        },
    }
    
    for _, t := range tests {
        fmt.Printf("\n=== %s ===\n", t.name)
        if err := processUserJSON(t.json); err != nil {
            fmt.Printf("Error: %v\n", err)
        }
    }
}
```

---

## 18.8 Workshop: Config File Parser

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "path/filepath"
    "strings"
)

type Environment string

const (
    Development Environment = "development"
    Staging     Environment = "staging"
    Production  Environment = "production"
)

type DatabaseConfig struct {
    Driver          string `json:"driver"`
    Host            string `json:"host"`
    Port            int    `json:"port"`
    Name            string `json:"name"`
    User            string `json:"user"`
    Password        string `json:"password"`
    MaxOpenConns    int    `json:"max_open_conns"`
    MaxIdleConns    int    `json:"max_idle_conns"`
    ConnMaxLifetime int    `json:"conn_max_lifetime"` // seconds
    SSLMode         string `json:"ssl_mode"`
}

func (db DatabaseConfig) DSN() string {
    return fmt.Sprintf("%s://%s:%s@%s:%d/%s?sslmode=%s",
        db.Driver, db.User, db.Password,
        db.Host, db.Port, db.Name, db.SSLMode)
}

type RedisConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    Password string `json:"password"`
    DB       int    `json:"db"`
}

type ServerConfig struct {
    Host         string   `json:"host"`
    Port         int      `json:"port"`
    ReadTimeout  int      `json:"read_timeout"`
    WriteTimeout int      `json:"write_timeout"`
    AllowOrigins []string `json:"allow_origins"`
}

type LogConfig struct {
    Level  string `json:"level"`
    Format string `json:"format"`
    Output string `json:"output"`
    File   string `json:"file,omitempty"`
}

type AppConfig struct {
    Environment Environment    `json:"environment"`
    AppName     string         `json:"app_name"`
    Version     string         `json:"version"`
    Server      ServerConfig   `json:"server"`
    Database    DatabaseConfig `json:"database"`
    Redis       RedisConfig    `json:"redis"`
    Log         LogConfig      `json:"log"`
    Features    map[string]bool `json:"features"`
}

type ConfigLoader struct {
    searchPaths []string
    overrides   map[string]string
}

func NewConfigLoader(searchPaths ...string) *ConfigLoader {
    return &ConfigLoader{
        searchPaths: searchPaths,
        overrides:   make(map[string]string),
    }
}

func (cl *ConfigLoader) SetOverride(key, value string) {
    cl.overrides[key] = value
}

func (cl *ConfigLoader) Load(env Environment) (*AppConfig, error) {
    // หาไฟล์ config
    var configFile string
    
    candidates := []string{
        fmt.Sprintf("config.%s.json", env),
        "config.json",
    }
    
    for _, searchPath := range cl.searchPaths {
        for _, candidate := range candidates {
            path := filepath.Join(searchPath, candidate)
            if _, err := os.Stat(path); err == nil {
                configFile = path
                break
            }
        }
        if configFile != "" {
            break
        }
    }
    
    if configFile == "" {
        // ใช้ default config
        return cl.defaultConfig(env), nil
    }
    
    // อ่านไฟล์
    data, err := os.ReadFile(configFile)
    if err != nil {
        return nil, fmt.Errorf("อ่าน config ไม่ได้: %w", err)
    }
    
    var cfg AppConfig
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config ไม่ได้: %w", err)
    }
    
    // ใช้ overrides
    cl.applyOverrides(&cfg)
    
    return &cfg, nil
}

func (cl *ConfigLoader) defaultConfig(env Environment) *AppConfig {
    cfg := &AppConfig{
        Environment: env,
        AppName:     "myapp",
        Version:     "1.0.0",
        Server: ServerConfig{
            Host:         "0.0.0.0",
            Port:         8080,
            ReadTimeout:  30,
            WriteTimeout: 30,
            AllowOrigins: []string{"*"},
        },
        Database: DatabaseConfig{
            Driver:          "postgres",
            Host:            "localhost",
            Port:            5432,
            Name:            "mydb",
            User:            "postgres",
            Password:        "postgres",
            MaxOpenConns:    10,
            MaxIdleConns:    5,
            ConnMaxLifetime: 300,
            SSLMode:         "disable",
        },
        Redis: RedisConfig{
            Host: "localhost",
            Port: 6379,
            DB:   0,
        },
        Log: LogConfig{
            Level:  "debug",
            Format: "text",
            Output: "stdout",
        },
        Features: map[string]bool{
            "feature_a": true,
            "feature_b": false,
        },
    }
    
    // ปรับตาม environment
    if env == Production {
        cfg.Server.Port = 80
        cfg.Log.Level = "error"
        cfg.Log.Format = "json"
        cfg.Database.SSLMode = "require"
        cfg.Database.MaxOpenConns = 50
    } else if env == Staging {
        cfg.Log.Level = "warn"
    }
    
    return cfg
}

func (cl *ConfigLoader) applyOverrides(cfg *AppConfig) {
    for key, value := range cl.overrides {
        switch key {
        case "SERVER_PORT":
            fmt.Sscanf(value, "%d", &cfg.Server.Port)
        case "DB_HOST":
            cfg.Database.Host = value
        case "DB_PASSWORD":
            cfg.Database.Password = value
        case "LOG_LEVEL":
            cfg.Log.Level = value
        }
    }
}

func (cfg *AppConfig) Validate() []string {
    var errors []string
    
    if strings.TrimSpace(cfg.AppName) == "" {
        errors = append(errors, "app_name ต้องไม่ว่างเปล่า")
    }
    
    if cfg.Server.Port <= 0 || cfg.Server.Port > 65535 {
        errors = append(errors, fmt.Sprintf("server.port ไม่ถูกต้อง: %d", cfg.Server.Port))
    }
    
    if cfg.Database.Host == "" {
        errors = append(errors, "database.host ต้องไม่ว่างเปล่า")
    }
    
    if cfg.Database.Port <= 0 {
        errors = append(errors, "database.port ต้องมากกว่า 0")
    }
    
    validLevels := map[string]bool{"debug": true, "info": true, "warn": true, "error": true}
    if !validLevels[cfg.Log.Level] {
        errors = append(errors, fmt.Sprintf("log.level ไม่ถูกต้อง: %s", cfg.Log.Level))
    }
    
    return errors
}

func (cfg *AppConfig) Print() {
    fmt.Printf("=== Config: %s v%s [%s] ===\n", cfg.AppName, cfg.Version, cfg.Environment)
    fmt.Printf("Server: %s:%d\n", cfg.Server.Host, cfg.Server.Port)
    fmt.Printf("Database: %s\n", cfg.Database.DSN())
    fmt.Printf("Redis: %s:%d\n", cfg.Redis.Host, cfg.Redis.Port)
    fmt.Printf("Logging: level=%s, format=%s\n", cfg.Log.Level, cfg.Log.Format)
    fmt.Printf("Features:\n")
    for k, v := range cfg.Features {
        status := "disabled"
        if v {
            status = "enabled"
        }
        fmt.Printf("  %s: %s\n", k, status)
    }
}

func main() {
    // สร้างไฟล์ config จำลอง
    configJSON := `{
        "environment": "development",
        "app_name": "go-course-app",
        "version": "2.1.0",
        "server": {
            "host": "localhost",
            "port": 3000,
            "read_timeout": 30,
            "write_timeout": 60,
            "allow_origins": ["http://localhost:3000", "https://app.example.com"]
        },
        "database": {
            "driver": "postgres",
            "host": "db.local",
            "port": 5432,
            "name": "course_db",
            "user": "courseuser",
            "password": "coursepass",
            "max_open_conns": 20,
            "max_idle_conns": 5,
            "conn_max_lifetime": 600,
            "ssl_mode": "disable"
        },
        "redis": {
            "host": "redis.local",
            "port": 6379,
            "db": 0
        },
        "log": {
            "level": "debug",
            "format": "text",
            "output": "stdout"
        },
        "features": {
            "feature_a": true,
            "feature_b": true,
            "experimental": false
        }
    }`
    
    os.WriteFile("config.development.json", []byte(configJSON), 0644)
    defer os.Remove("config.development.json")
    
    // โหลด config
    loader := NewConfigLoader(".", "/etc/myapp")
    
    // กำหนด override จาก env vars
    loader.SetOverride("LOG_LEVEL", "info")
    
    cfg, err := loader.Load(Development)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    // Validate
    if errors := cfg.Validate(); len(errors) > 0 {
        fmt.Println("Config validation errors:")
        for _, e := range errors {
            fmt.Printf("  - %s\n", e)
        }
        return
    }
    
    cfg.Print()
    
    // Export กลับเป็น JSON
    fmt.Println("\n=== Export Config (masked) ===")
    export := *cfg
    export.Database.Password = "***"
    export.Redis.Password = "***"
    
    exportJSON, _ := json.MarshalIndent(export, "", "  ")
    fmt.Println(string(exportJSON[:400]) + "...")
    
    // Test default config (production)
    fmt.Println("\n=== Default Production Config ===")
    prodCfg, _ := loader.Load(Production)
    prodCfg.Print()
}
```

---

## สรุป

| Package | การใช้งาน |
|---------|----------|
| `encoding/json` | JSON สำหรับ API, config, data exchange |
| `encoding/xml` | XML สำหรับ SOAP, RSS, configuration |
| `gopkg.in/yaml.v3` | YAML สำหรับ config files (Kubernetes, Docker) |

### JSON Struct Tags สำคัญ

| Tag | ความหมาย |
|-----|----------|
| `json:"name"` | ใช้ชื่อนี้ใน JSON |
| `json:"name,omitempty"` | ข้ามถ้าเป็น zero value |
| `json:"-"` | ไม่รวมใน JSON เลย |
| `json:"name,string"` | แปลงตัวเลขเป็น string |

## Resources

- [encoding/json package](https://pkg.go.dev/encoding/json)
- [encoding/xml package](https://pkg.go.dev/encoding/xml)
- [json.org](https://www.json.org/json-en.html)
- [gopkg.in/yaml.v3](https://pkg.go.dev/gopkg.in/yaml.v3)
- [JSON Schema Validation](https://json-schema.org/)
