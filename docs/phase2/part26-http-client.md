# Part 26: HTTP Client ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `net/http` package สำหรับทำ HTTP requests
- ส่ง GET, POST, PUT, DELETE requests
- ตั้งค่า Headers และ Query Parameters
- ส่ง Request Body แบบ JSON
- จัดการ Response อย่างถูกต้อง
- ใช้ Timeouts และ Context
- เขียน Retry Logic
- สร้าง Custom HTTP Client

---

## 26.1 พื้นฐาน HTTP Client

Go มี `net/http` package ที่ built-in มาพร้อมกับ standard library ซึ่งมีความสามารถครบครันสำหรับการทำ HTTP requests

### การ Import Package

```go
package main

import (
    "fmt"
    "net/http"
    "io"
    "log"
)
```

### Simple GET Request (ตัวอย่างที่ 1)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    // ทำ GET request แบบง่ายที่สุด
    resp, err := http.Get("https://httpbin.org/get")
    if err != nil {
        log.Fatal("Error making request:", err)
    }
    defer resp.Body.Close() // สำคัญมาก! ต้อง close body เสมอ

    // อ่าน response body
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal("Error reading body:", err)
    }

    fmt.Println("Status:", resp.Status)
    fmt.Println("Status Code:", resp.StatusCode)
    fmt.Println("Body:", string(body))
}
```

**หมายเหตุสำคัญ:** ต้องเรียก `resp.Body.Close()` เสมอ ไม่งั้นจะเกิด resource leak

---

## 26.2 GET Request พร้อม Query Parameters

### Query Parameters แบบ Manual (ตัวอย่างที่ 2)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    // ใส่ query parameters ใน URL โดยตรง
    url := "https://httpbin.org/get?name=john&age=30&city=bangkok"
    
    resp, err := http.Get(url)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### Query Parameters แบบ url.Values (ตัวอย่างที่ 3)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "net/url"
)

func main() {
    // สร้าง base URL
    baseURL := "https://httpbin.org/get"
    
    // สร้าง query parameters
    params := url.Values{}
    params.Add("name", "สมชาย")
    params.Add("age", "25")
    params.Add("city", "กรุงเทพ")
    params.Add("hobby", "coding")
    params.Add("hobby", "reading") // เพิ่มค่าซ้ำ key ได้
    
    // รวม URL กับ params
    fullURL := baseURL + "?" + params.Encode()
    fmt.Println("Request URL:", fullURL)
    
    resp, err := http.Get(fullURL)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### สร้าง URL แบบ url.Parse (ตัวอย่างที่ 4)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "net/url"
)

func main() {
    // parse URL แล้วเพิ่ม query params
    u, err := url.Parse("https://httpbin.org/get")
    if err != nil {
        log.Fatal(err)
    }
    
    // เข้าถึง query params ของ URL
    q := u.Query()
    q.Set("search", "golang tutorial")
    q.Set("page", "1")
    q.Set("limit", "10")
    
    // กำหนด query กลับเข้า URL
    u.RawQuery = q.Encode()
    
    fmt.Println("Final URL:", u.String())
    
    resp, err := http.Get(u.String())
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response:", string(body))
}
```

---

## 26.3 การตั้งค่า Headers

### GET Request พร้อม Custom Headers (ตัวอย่างที่ 5)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    // สร้าง request object เองเพื่อกำหนด headers
    req, err := http.NewRequest("GET", "https://httpbin.org/headers", nil)
    if err != nil {
        log.Fatal(err)
    }
    
    // ตั้งค่า Headers
    req.Header.Set("Authorization", "Bearer my-secret-token")
    req.Header.Set("Accept", "application/json")
    req.Header.Set("X-Custom-Header", "my-value")
    req.Header.Set("User-Agent", "MyGoApp/1.0")
    
    // เพิ่ม header (ถ้า key มีอยู่แล้ว จะเพิ่มค่าใหม่ ไม่ใช่แทนที่)
    req.Header.Add("X-Request-ID", "req-123456")
    
    // ใช้ default client ส่ง request
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response Headers sent:")
    fmt.Println(string(body))
}
```

---

## 26.4 POST Request

### POST พร้อม Form Data (ตัวอย่างที่ 6)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "net/url"
    "strings"
)

func main() {
    // ส่ง form data
    formData := url.Values{
        "username": {"john_doe"},
        "password": {"secret123"},
        "email":    {"john@example.com"},
    }
    
    resp, err := http.PostForm("https://httpbin.org/post", formData)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Status:", resp.Status)
    fmt.Println("Response:", string(body))
    
    _ = strings.NewReader("") // suppress unused import warning
}
```

### POST พร้อม JSON Body (ตัวอย่างที่ 7)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
)

type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

func main() {
    // สร้าง request body
    payload := CreateUserRequest{
        Name:  "สมชาย รักเรียน",
        Email: "somchai@example.com",
        Age:   28,
    }
    
    // แปลง struct เป็น JSON
    jsonData, err := json.Marshal(payload)
    if err != nil {
        log.Fatal("Error marshaling JSON:", err)
    }
    
    // สร้าง POST request
    req, err := http.NewRequest("POST", "https://httpbin.org/post", bytes.NewBuffer(jsonData))
    if err != nil {
        log.Fatal(err)
    }
    
    // ตั้งค่า Content-Type เป็น JSON
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Accept", "application/json")
    
    // ส่ง request
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Status:", resp.Status)
    fmt.Println("Response:", string(body))
}
```

---

## 26.5 PUT และ DELETE Requests

### PUT Request (ตัวอย่างที่ 8)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
)

type UpdateUserRequest struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func main() {
    payload := UpdateUserRequest{
        ID:    1,
        Name:  "สมชาย อัพเดทแล้ว",
        Email: "somchai.updated@example.com",
    }
    
    jsonData, err := json.Marshal(payload)
    if err != nil {
        log.Fatal(err)
    }
    
    req, err := http.NewRequest("PUT", "https://httpbin.org/put", bytes.NewBuffer(jsonData))
    if err != nil {
        log.Fatal(err)
    }
    req.Header.Set("Content-Type", "application/json")
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("PUT Response:", resp.Status)
    fmt.Println(string(body))
}
```

### PATCH Request (ตัวอย่างที่ 9)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    // Partial update - ส่งแค่ field ที่ต้องการ update
    patch := map[string]interface{}{
        "email": "new-email@example.com",
    }
    
    jsonData, _ := json.Marshal(patch)
    
    req, err := http.NewRequest("PATCH", "https://httpbin.org/patch", bytes.NewBuffer(jsonData))
    if err != nil {
        log.Fatal(err)
    }
    req.Header.Set("Content-Type", "application/json")
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    fmt.Println("PATCH Response:", resp.Status)
}
```

### DELETE Request (ตัวอย่างที่ 10)

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // DELETE request
    req, err := http.NewRequest("DELETE", "https://httpbin.org/delete", nil)
    if err != nil {
        log.Fatal(err)
    }
    req.Header.Set("Authorization", "Bearer token123")
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    fmt.Println("DELETE Response:", resp.Status)
    
    if resp.StatusCode == http.StatusOK || resp.StatusCode == http.StatusNoContent {
        fmt.Println("Deleted successfully!")
    }
}
```

---

## 26.6 Response Handling

### อ่าน Response Headers (ตัวอย่างที่ 11)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    resp, err := http.Get("https://httpbin.org/get")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // อ่าน Response Headers
    fmt.Println("=== Response Headers ===")
    for key, values := range resp.Header {
        for _, value := range values {
            fmt.Printf("%s: %s\n", key, value)
        }
    }
    
    // อ่าน header เฉพาะ key
    contentType := resp.Header.Get("Content-Type")
    fmt.Println("\nContent-Type:", contentType)
    
    // อ่าน body
    body, _ := io.ReadAll(resp.Body)
    fmt.Println("\nBody length:", len(body), "bytes")
}
```

### Parse JSON Response (ตัวอย่างที่ 12)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

// โครงสร้างสำหรับ parse response จาก httpbin.org
type HttpBinGetResponse struct {
    Args    map[string]string `json:"args"`
    Headers map[string]string `json:"headers"`
    Origin  string            `json:"origin"`
    URL     string            `json:"url"`
}

func main() {
    resp, err := http.Get("https://httpbin.org/get?foo=bar&hello=world")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // ตรวจสอบ status code ก่อน
    if resp.StatusCode != http.StatusOK {
        log.Fatalf("unexpected status: %s", resp.Status)
    }

    // Parse JSON response
    var result HttpBinGetResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        log.Fatal("Error decoding JSON:", err)
    }
    
    fmt.Println("Origin IP:", result.Origin)
    fmt.Println("Request URL:", result.URL)
    fmt.Println("Query Args:")
    for k, v := range result.Args {
        fmt.Printf("  %s = %s\n", k, v)
    }
}
```

### Handle HTTP Error Responses (ตัวอย่างที่ 13)

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
)

type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func handleResponse(resp *http.Response) ([]byte, error) {
    defer resp.Body.Close()
    
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("reading body: %w", err)
    }
    
    // จัดการ error responses
    if resp.StatusCode >= 400 {
        var apiErr APIError
        if err := json.Unmarshal(body, &apiErr); err == nil {
            return nil, fmt.Errorf("API error %d: %s", apiErr.Code, apiErr.Message)
        }
        return nil, fmt.Errorf("HTTP error %d: %s", resp.StatusCode, string(body))
    }
    
    return body, nil
}

func main() {
    // ทดสอบ 404
    resp, err := http.Get("https://httpbin.org/status/404")
    if err != nil {
        log.Fatal(err)
    }
    
    body, err := handleResponse(resp)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Println("Success:", string(body))
}
```

---

## 26.7 Timeouts และ Context

### กำหนด Timeout บน Client (ตัวอย่างที่ 14)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

func main() {
    // สร้าง custom client พร้อม timeout
    client := &http.Client{
        Timeout: 10 * time.Second, // timeout รวมทั้งหมด
    }
    
    resp, err := client.Get("https://httpbin.org/delay/2") // จะ delay 2 วินาที
    if err != nil {
        log.Fatal("Request failed:", err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response:", string(body[:100]))
}
```

### ใช้ Context สำหรับ Timeout (ตัวอย่างที่ 15)

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

func fetchWithContext(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}

func main() {
    // สร้าง context พร้อม timeout 5 วินาที
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel() // สำคัญ! ต้อง cancel เสมอ
    
    body, err := fetchWithContext(ctx, "https://httpbin.org/get")
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            log.Fatal("Request timed out!")
        }
        log.Fatal("Request failed:", err)
    }
    
    fmt.Println("Response length:", len(body))
    
    // Context Cancellation
    ctx2, cancel2 := context.WithCancel(context.Background())
    
    // Cancel หลัง 2 วินาที
    go func() {
        time.Sleep(2 * time.Second)
        cancel2()
        fmt.Println("Context cancelled!")
    }()
    
    body2, err := fetchWithContext(ctx2, "https://httpbin.org/delay/5")
    if err != nil {
        fmt.Println("Error (expected):", err)
    } else {
        fmt.Println("Body:", string(body2[:50]))
    }
}
```

---

## 26.8 Custom HTTP Client

### Custom Transport (ตัวอย่างที่ 16)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

func createOptimizedClient() *http.Client {
    // Custom transport สำหรับ performance tuning
    transport := &http.Transport{
        MaxIdleConns:        100,              // จำนวน idle connections สูงสุด
        MaxIdleConnsPerHost: 10,               // idle connections ต่อ host
        MaxConnsPerHost:     20,               // connections สูงสุดต่อ host
        IdleConnTimeout:     90 * time.Second, // timeout สำหรับ idle connections
        
        // TLS settings
        TLSHandshakeTimeout: 10 * time.Second,
        
        // Proxy
        // Proxy: http.ProxyFromEnvironment,
    }
    
    return &http.Client{
        Timeout:   30 * time.Second,
        Transport: transport,
    }
}

func main() {
    client := createOptimizedClient()
    
    // ทดสอบ client
    resp, err := client.Get("https://httpbin.org/get")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("Response (%d bytes): %s\n", len(body), body[:100])
}
```

### Logging Middleware Transport (ตัวอย่างที่ 17)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

// LoggingTransport เป็น custom RoundTripper ที่ log ทุก request/response
type LoggingTransport struct {
    Transport http.RoundTripper
}

func (t *LoggingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    start := time.Now()
    
    // Log request
    fmt.Printf("[HTTP] %s %s\n", req.Method, req.URL)
    
    // ส่ง request ไปยัง underlying transport
    resp, err := t.Transport.RoundTrip(req)
    
    duration := time.Since(start)
    
    if err != nil {
        fmt.Printf("[HTTP] Error after %v: %v\n", duration, err)
        return nil, err
    }
    
    // Log response
    fmt.Printf("[HTTP] %s %s -> %d (%v)\n", req.Method, req.URL, resp.StatusCode, duration)
    
    return resp, nil
}

func main() {
    client := &http.Client{
        Timeout: 10 * time.Second,
        Transport: &LoggingTransport{
            Transport: http.DefaultTransport,
        },
    }
    
    resp, err := client.Get("https://httpbin.org/get")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response:", string(body[:100]))
}
```

---

## 26.9 Retry Logic

### Simple Retry (ตัวอย่างที่ 18)

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

func fetchWithRetry(url string, maxRetries int) ([]byte, error) {
    var lastErr error
    
    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            // Exponential backoff: 1s, 2s, 4s, ...
            waitTime := time.Duration(1<<uint(attempt-1)) * time.Second
            fmt.Printf("Retry %d/%d, waiting %v...\n", attempt, maxRetries, waitTime)
            time.Sleep(waitTime)
        }
        
        resp, err := http.Get(url)
        if err != nil {
            lastErr = err
            fmt.Printf("Attempt %d failed: %v\n", attempt+1, err)
            continue
        }
        
        // ถ้าได้ 5xx ลอง retry
        if resp.StatusCode >= 500 {
            resp.Body.Close()
            lastErr = fmt.Errorf("server error: %d", resp.StatusCode)
            fmt.Printf("Attempt %d got %d, will retry\n", attempt+1, resp.StatusCode)
            continue
        }
        
        // สำเร็จ!
        defer resp.Body.Close()
        return io.ReadAll(resp.Body)
    }
    
    return nil, fmt.Errorf("failed after %d attempts: %w", maxRetries+1, lastErr)
}

func main() {
    // ทดสอบกับ URL ที่จะส่ง 503 บางครั้ง
    body, err := fetchWithRetry("https://httpbin.org/status/503", 3)
    if err != nil {
        fmt.Println("Final error:", err)
    } else {
        fmt.Println("Success:", string(body))
    }
}
```

### Retry พร้อม Jitter (ตัวอย่างที่ 19)

```go
package main

import (
    "context"
    "fmt"
    "io"
    "math/rand"
    "net/http"
    "time"
)

type RetryConfig struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    // Jitter เพิ่ม randomness เพื่อหลีกเลี่ยง thundering herd
    Jitter      bool
}

func fetchWithAdvancedRetry(ctx context.Context, client *http.Client, url string, config RetryConfig) ([]byte, error) {
    var lastErr error
    
    for attempt := 0; attempt < config.MaxAttempts; attempt++ {
        if attempt > 0 {
            delay := config.BaseDelay * time.Duration(1<<uint(attempt-1))
            if delay > config.MaxDelay {
                delay = config.MaxDelay
            }
            if config.Jitter {
                // เพิ่ม ±25% jitter
                jitter := time.Duration(rand.Int63n(int64(delay) / 2))
                delay = delay + jitter - delay/4
            }
            
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-time.After(delay):
            }
        }
        
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return nil, err
        }
        
        resp, err := client.Do(req)
        if err != nil {
            lastErr = err
            continue
        }
        
        // Retry สำหรับ 429 (Too Many Requests) และ 5xx
        if resp.StatusCode == http.StatusTooManyRequests || resp.StatusCode >= 500 {
            resp.Body.Close()
            lastErr = fmt.Errorf("status %d", resp.StatusCode)
            fmt.Printf("Attempt %d: got %d, retrying...\n", attempt+1, resp.StatusCode)
            continue
        }
        
        defer resp.Body.Close()
        return io.ReadAll(resp.Body)
    }
    
    return nil, fmt.Errorf("exceeded max attempts: %w", lastErr)
}

func main() {
    ctx := context.Background()
    client := &http.Client{Timeout: 10 * time.Second}
    
    config := RetryConfig{
        MaxAttempts: 3,
        BaseDelay:   500 * time.Millisecond,
        MaxDelay:    5 * time.Second,
        Jitter:      true,
    }
    
    body, err := fetchWithAdvancedRetry(ctx, client, "https://httpbin.org/get", config)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Success! Got %d bytes\n", len(body))
    }
}
```

---

## 26.10 HTTP Client Helper Functions

### Generic HTTP Client Helper (ตัวอย่างที่ 20)

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

// HTTPClient เป็น wrapper สำหรับ http.Client
type HTTPClient struct {
    client  *http.Client
    baseURL string
    headers map[string]string
}

func NewHTTPClient(baseURL string, timeout time.Duration) *HTTPClient {
    return &HTTPClient{
        client: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 10,
            },
        },
        baseURL: baseURL,
        headers: make(map[string]string),
    }
}

func (c *HTTPClient) SetHeader(key, value string) {
    c.headers[key] = value
}

func (c *HTTPClient) do(ctx context.Context, method, path string, body interface{}) (*http.Response, error) {
    var bodyReader io.Reader
    
    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, fmt.Errorf("marshaling body: %w", err)
        }
        bodyReader = bytes.NewBuffer(jsonData)
    }
    
    url := c.baseURL + path
    req, err := http.NewRequestWithContext(ctx, method, url, bodyReader)
    if err != nil {
        return nil, err
    }
    
    // ตั้งค่า default headers
    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }
    req.Header.Set("Accept", "application/json")
    
    // ตั้งค่า custom headers
    for k, v := range c.headers {
        req.Header.Set(k, v)
    }
    
    return c.client.Do(req)
}

func (c *HTTPClient) Get(ctx context.Context, path string, result interface{}) error {
    resp, err := c.do(ctx, "GET", path, nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(body))
    }
    
    return json.NewDecoder(resp.Body).Decode(result)
}

func (c *HTTPClient) Post(ctx context.Context, path string, payload, result interface{}) error {
    resp, err := c.do(ctx, "POST", path, payload)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(body))
    }
    
    if result != nil {
        return json.NewDecoder(resp.Body).Decode(result)
    }
    return nil
}

// Response struct สำหรับ httpbin.org
type HttpBinResponse struct {
    Origin string            `json:"origin"`
    URL    string            `json:"url"`
    Args   map[string]string `json:"args"`
    JSON   interface{}       `json:"json"`
}

func main() {
    ctx := context.Background()
    
    // สร้าง client
    client := NewHTTPClient("https://httpbin.org", 30*time.Second)
    client.SetHeader("Authorization", "Bearer demo-token")
    client.SetHeader("X-App-Version", "1.0.0")
    
    // GET request
    var getResult HttpBinResponse
    err := client.Get(ctx, "/get", &getResult)
    if err != nil {
        fmt.Println("GET Error:", err)
    } else {
        fmt.Println("GET Origin:", getResult.Origin)
    }
    
    // POST request
    payload := map[string]interface{}{
        "name": "test",
        "value": 42,
    }
    
    var postResult HttpBinResponse
    err = client.Post(ctx, "/post", payload, &postResult)
    if err != nil {
        fmt.Println("POST Error:", err)
    } else {
        fmt.Println("POST data sent:", postResult.JSON)
    }
}
```

---

## Workshop: REST API Client

สร้าง client สำหรับ JSONPlaceholder API (https://jsonplaceholder.typicode.com)

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "bytes"
    "time"
)

// Models
type Post struct {
    UserID int    `json:"userId"`
    ID     int    `json:"id"`
    Title  string `json:"title"`
    Body   string `json:"body"`
}

type Comment struct {
    PostID int    `json:"postId"`
    ID     int    `json:"id"`
    Name   string `json:"name"`
    Email  string `json:"email"`
    Body   string `json:"body"`
}

type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

// JSONPlaceholder API Client
type JSONPlaceholderClient struct {
    client  *http.Client
    baseURL string
}

func NewJSONPlaceholderClient() *JSONPlaceholderClient {
    return &JSONPlaceholderClient{
        client: &http.Client{
            Timeout: 10 * time.Second,
        },
        baseURL: "https://jsonplaceholder.typicode.com",
    }
}

// Generic request helper
func (c *JSONPlaceholderClient) request(ctx context.Context, method, path string, body interface{}, result interface{}) error {
    var bodyReader io.Reader
    if body != nil {
        data, err := json.Marshal(body)
        if err != nil {
            return fmt.Errorf("marshal error: %w", err)
        }
        bodyReader = bytes.NewBuffer(data)
    }
    
    req, err := http.NewRequestWithContext(ctx, method, c.baseURL+path, bodyReader)
    if err != nil {
        return err
    }
    
    if body != nil {
        req.Header.Set("Content-Type", "application/json; charset=UTF-8")
    }
    
    resp, err := c.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        data, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("API error %d: %s", resp.StatusCode, string(data))
    }
    
    if result != nil {
        return json.NewDecoder(resp.Body).Decode(result)
    }
    return nil
}

// Posts API
func (c *JSONPlaceholderClient) GetPosts(ctx context.Context) ([]Post, error) {
    var posts []Post
    return posts, c.request(ctx, "GET", "/posts", nil, &posts)
}

func (c *JSONPlaceholderClient) GetPost(ctx context.Context, id int) (*Post, error) {
    var post Post
    return &post, c.request(ctx, "GET", fmt.Sprintf("/posts/%d", id), nil, &post)
}

func (c *JSONPlaceholderClient) CreatePost(ctx context.Context, post Post) (*Post, error) {
    var created Post
    return &created, c.request(ctx, "POST", "/posts", post, &created)
}

func (c *JSONPlaceholderClient) UpdatePost(ctx context.Context, id int, post Post) (*Post, error) {
    var updated Post
    return &updated, c.request(ctx, "PUT", fmt.Sprintf("/posts/%d", id), post, &updated)
}

func (c *JSONPlaceholderClient) DeletePost(ctx context.Context, id int) error {
    return c.request(ctx, "DELETE", fmt.Sprintf("/posts/%d", id), nil, nil)
}

func (c *JSONPlaceholderClient) GetPostComments(ctx context.Context, postID int) ([]Comment, error) {
    var comments []Comment
    return comments, c.request(ctx, "GET", fmt.Sprintf("/posts/%d/comments", postID), nil, &comments)
}

// Users API
func (c *JSONPlaceholderClient) GetUsers(ctx context.Context) ([]User, error) {
    var users []User
    return users, c.request(ctx, "GET", "/users", nil, &users)
}

func main() {
    ctx := context.Background()
    client := NewJSONPlaceholderClient()
    
    // 1. ดึงรายการ posts ทั้งหมด
    posts, err := client.GetPosts(ctx)
    if err != nil {
        log.Fatal("GetPosts error:", err)
    }
    fmt.Printf("Total posts: %d\n", len(posts))
    fmt.Printf("First post: %s\n", posts[0].Title)
    
    // 2. ดึง post เฉพาะ ID
    post, err := client.GetPost(ctx, 1)
    if err != nil {
        log.Fatal("GetPost error:", err)
    }
    fmt.Printf("\nPost #1:\n  Title: %s\n  Body: %s\n", post.Title, post.Body[:50]+"...")
    
    // 3. สร้าง post ใหม่
    newPost, err := client.CreatePost(ctx, Post{
        UserID: 1,
        Title:  "ทดสอบการสร้าง Post ใหม่",
        Body:   "เนื้อหาของ post นี้เป็นการทดสอบ",
    })
    if err != nil {
        log.Fatal("CreatePost error:", err)
    }
    fmt.Printf("\nCreated post ID: %d\n", newPost.ID)
    
    // 4. อัพเดท post
    updated, err := client.UpdatePost(ctx, 1, Post{
        ID:     1,
        UserID: 1,
        Title:  "Updated Title",
        Body:   "Updated Body",
    })
    if err != nil {
        log.Fatal("UpdatePost error:", err)
    }
    fmt.Printf("\nUpdated post: %s\n", updated.Title)
    
    // 5. ดึง comments ของ post
    comments, err := client.GetPostComments(ctx, 1)
    if err != nil {
        log.Fatal("GetPostComments error:", err)
    }
    fmt.Printf("\nComments for post #1: %d comments\n", len(comments))
    
    // 6. ลบ post
    err = client.DeletePost(ctx, 1)
    if err != nil {
        log.Fatal("DeletePost error:", err)
    }
    fmt.Println("\nPost deleted successfully!")
    
    // 7. ดึงรายการ users
    users, err := client.GetUsers(ctx)
    if err != nil {
        log.Fatal("GetUsers error:", err)
    }
    fmt.Printf("\nUsers:\n")
    for _, u := range users[:3] {
        fmt.Printf("  - %s (%s)\n", u.Name, u.Email)
    }
}
```

---

## สรุป Part 26

สิ่งที่เรียนรู้ใน Part นี้:

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| GET Request | `http.Get()` หรือ `http.NewRequest()` + `client.Do()` |
| POST/PUT/DELETE | ใช้ `http.NewRequest()` กำหนด method เอง |
| Headers | `req.Header.Set()` / `req.Header.Add()` |
| Query Params | `url.Values` + `params.Encode()` |
| JSON Body | `json.Marshal()` + `bytes.NewBuffer()` |
| Response | ตรวจ StatusCode, อ่าน Body, `defer resp.Body.Close()` |
| Timeout | `client.Timeout` หรือ `context.WithTimeout()` |
| Retry | Exponential backoff + Jitter |
| Custom Client | Custom Transport สำหรับ logging/middleware |

### Resources
- [net/http documentation](https://pkg.go.dev/net/http)
- [context package](https://pkg.go.dev/context)
- [net/url package](https://pkg.go.dev/net/url)
- [httpbin.org](https://httpbin.org) - HTTP request & response service
