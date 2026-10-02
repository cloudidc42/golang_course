# Part 17: Strings และ String Manipulation ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `strings` package อย่างครบถ้วน
- แปลงข้อมูลด้วย `strconv` package
- สร้าง string อย่างมีประสิทธิภาพด้วย `strings.Builder`
- ใช้ Regular Expressions กับ `regexp` package
- เข้าใจ Unicode และ Runes ใน Go
- Format string ด้วย `fmt.Sprintf` และ format verbs

---

## 17.1 strings Package

### 17.1.1 การตรวจสอบ String

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    s := "Hello, สวัสดี World!"
    
    // Contains - ตรวจสอบว่ามี substring หรือไม่
    fmt.Println(strings.Contains(s, "สวัสดี"))     // true
    fmt.Println(strings.Contains(s, "goodbye"))   // false
    
    // ContainsAny - ตรวจสอบว่ามีตัวอักษรใดๆ หรือไม่
    fmt.Println(strings.ContainsAny(s, "aeiou")) // true (มี vowels)
    fmt.Println(strings.ContainsAny(s, "xyz"))   // true (มี x)
    
    // ContainsRune - ตรวจสอบ rune เฉพาะ
    fmt.Println(strings.ContainsRune(s, 'H')) // true
    fmt.Println(strings.ContainsRune(s, '!')) // true
    
    // HasPrefix - ขึ้นต้นด้วย
    fmt.Println(strings.HasPrefix(s, "Hello")) // true
    fmt.Println(strings.HasPrefix(s, "World")) // false
    
    // HasSuffix - ลงท้ายด้วย
    fmt.Println(strings.HasSuffix(s, "World!")) // true
    fmt.Println(strings.HasSuffix(s, "Hello"))  // false
    
    // Count - นับจำนวน substring
    text := "banana"
    fmt.Println(strings.Count(text, "a"))  // 3
    fmt.Println(strings.Count(text, "na")) // 2
    fmt.Println(strings.Count(text, ""))  // 7 (len+1)
    
    // Index - หาตำแหน่ง (return -1 ถ้าไม่พบ)
    fmt.Println(strings.Index(s, "สวัสดี"))  // ตำแหน่ง bytes
    fmt.Println(strings.Index(s, "nothere")) // -1
    
    // LastIndex - หาตำแหน่งจากหลัง
    path := "/home/user/documents/file.txt"
    fmt.Println(strings.LastIndex(path, "/")) // ตำแหน่ง / สุดท้าย
}
```

### 17.1.2 การแปลง String

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // ToUpper / ToLower
    s := "Hello, World!"
    fmt.Println(strings.ToUpper(s)) // HELLO, WORLD!
    fmt.Println(strings.ToLower(s)) // hello, world!
    
    // Title (deprecated ใน Go 1.18, ใช้ golang.org/x/text แทน)
    // strings.Title("hello world") → "Hello World"
    
    // Replace - แทนที่ (n=-1 แทนทั้งหมด)
    text := "foo bar foo baz foo"
    fmt.Println(strings.Replace(text, "foo", "qux", 1))   // qux bar foo baz foo
    fmt.Println(strings.Replace(text, "foo", "qux", 2))   // qux bar qux baz foo
    fmt.Println(strings.Replace(text, "foo", "qux", -1))  // qux bar qux baz qux
    fmt.Println(strings.ReplaceAll(text, "foo", "qux"))   // qux bar qux baz qux
    
    // Trim - ตัดช่องว่าง/ตัวอักษร
    padded := "   Hello, World!   "
    fmt.Printf("%q\n", strings.TrimSpace(padded))       // "Hello, World!"
    fmt.Printf("%q\n", strings.Trim(padded, " "))       // "Hello, World!"
    fmt.Printf("%q\n", strings.TrimLeft(padded, " "))   // "Hello, World!   "
    fmt.Printf("%q\n", strings.TrimRight(padded, " "))  // "   Hello, World!"
    
    // TrimPrefix / TrimSuffix
    url := "https://www.example.com"
    fmt.Println(strings.TrimPrefix(url, "https://"))  // www.example.com
    fmt.Println(strings.TrimSuffix(url, ".com"))      // https://www.example
    
    // TrimFunc
    result := strings.TrimFunc("  Hello  ", func(r rune) bool {
        return r == ' '
    })
    fmt.Println(result) // Hello
}
```

### 17.1.3 Split และ Join

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // Split
    csv := "apple,banana,cherry,date"
    parts := strings.Split(csv, ",")
    fmt.Println(parts)        // [apple banana cherry date]
    fmt.Println(len(parts))   // 4
    
    // Split พร้อม limit
    fmt.Println(strings.SplitN(csv, ",", 2))  // [apple banana,cherry,date]
    fmt.Println(strings.SplitN(csv, ",", 3))  // [apple banana cherry,date]
    
    // SplitAfter - เก็บ separator ด้วย
    fmt.Println(strings.SplitAfter(csv, ",")) // [apple, banana, cherry, date]
    
    // Fields - split ด้วย whitespace
    text := "Hello   World\t\tGo\n\nLang"
    words := strings.Fields(text)
    fmt.Println(words)     // [Hello World Go Lang]
    fmt.Println(len(words)) // 4
    
    // FieldsFunc - split ด้วย custom function
    text2 := "foo1bar2baz3qux"
    parts2 := strings.FieldsFunc(text2, func(r rune) bool {
        return r >= '0' && r <= '9'
    })
    fmt.Println(parts2) // [foo bar baz qux]
    
    // Join
    words2 := []string{"สวัสดี", "ชาว", "โลก"}
    fmt.Println(strings.Join(words2, " "))  // สวัสดี ชาว โลก
    fmt.Println(strings.Join(words2, "-"))  // สวัสดี-ชาว-โลก
    fmt.Println(strings.Join(words2, ""))   // สวัสดีชาวโลก
    
    // Repeat
    fmt.Println(strings.Repeat("abc", 3)) // abcabcabc
    fmt.Println(strings.Repeat("-", 20))  // --------------------
    
    // Map - แปลงแต่ละ rune
    rot13 := func(r rune) rune {
        switch {
        case r >= 'A' && r <= 'Z':
            return 'A' + (r-'A'+13)%26
        case r >= 'a' && r <= 'z':
            return 'a' + (r-'a'+13)%26
        }
        return r
    }
    
    encoded := strings.Map(rot13, "Hello, World!")
    fmt.Println(encoded) // Uryyb, Jbeyq!
    decoded := strings.Map(rot13, encoded)
    fmt.Println(decoded) // Hello, World!
}
```

### 17.1.4 String Comparison

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // EqualFold - case-insensitive comparison
    fmt.Println(strings.EqualFold("Go", "go"))         // true
    fmt.Println(strings.EqualFold("Hello", "HELLO"))   // true
    fmt.Println(strings.EqualFold("สวัสดี", "สวัสดี")) // true
    
    // Compare - เหมือน strcmp
    // -1 ถ้า a < b, 0 ถ้า a == b, 1 ถ้า a > b
    fmt.Println(strings.Compare("apple", "banana")) // -1
    fmt.Println(strings.Compare("banana", "apple")) // 1
    fmt.Println(strings.Compare("apple", "apple"))  // 0
    
    // IndexRune
    s := "Hello, 世界!"
    fmt.Println(strings.IndexRune(s, '世')) // ตำแหน่ง byte ของ '世'
    fmt.Println(strings.IndexRune(s, 'H')) // 0
    
    // IndexByte
    fmt.Println(strings.IndexByte(s, 'H')) // 0
    fmt.Println(strings.IndexByte(s, '!')) // ตำแหน่ง '!'
    
    // IndexAny - ตำแหน่งแรกที่พบตัวอักษรใดๆ
    fmt.Println(strings.IndexAny("Hello", "aeiou")) // 1 (e)
    
    // LastIndexAny
    fmt.Println(strings.LastIndexAny("Hello", "aeiou")) // 4 (o)
    
    // Cut - Go 1.18+
    before, after, found := strings.Cut("user:password@host", ":")
    fmt.Printf("before=%q, after=%q, found=%v\n", before, after, found)
    // before="user", after="password@host", found=true
    
    // CutPrefix / CutSuffix - Go 1.20+
    if after2, found2 := strings.CutPrefix("https://example.com", "https://"); found2 {
        fmt.Println("Domain:", after2) // example.com
    }
}
```

---

## 17.2 strconv Package

### 17.2.1 แปลง String เป็นตัวเลข

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Atoi - string to int (ง่ายที่สุด)
    n, err := strconv.Atoi("42")
    if err == nil {
        fmt.Printf("Atoi: %d (type: %T)\n", n, n)
    }
    
    // Atoi error handling
    _, err = strconv.Atoi("hello")
    fmt.Printf("Atoi error: %v\n", err)
    
    // ParseInt - ยืดหยุ่นกว่า
    i64, _ := strconv.ParseInt("100", 10, 64)   // base 10, 64-bit
    i32, _ := strconv.ParseInt("100", 10, 32)   // base 10, 32-bit
    hex, _ := strconv.ParseInt("FF", 16, 64)    // base 16
    bin, _ := strconv.ParseInt("1010", 2, 64)   // base 2
    oct, _ := strconv.ParseInt("77", 8, 64)     // base 8
    
    fmt.Printf("ParseInt decimal: %d\n", i64)
    fmt.Printf("ParseInt 32-bit:  %d\n", i32)
    fmt.Printf("ParseInt hex FF:  %d\n", hex)
    fmt.Printf("ParseInt bin 1010: %d\n", bin)
    fmt.Printf("ParseInt oct 77:  %d\n", oct)
    
    // ParseUint
    u64, _ := strconv.ParseUint("18446744073709551615", 10, 64) // MaxUint64
    fmt.Printf("ParseUint: %d\n", u64)
    
    // ParseFloat
    f64, _ := strconv.ParseFloat("3.14159", 64)
    f32, _ := strconv.ParseFloat("3.14", 32)
    fmt.Printf("ParseFloat64: %f\n", f64)
    fmt.Printf("ParseFloat32: %f\n", float32(f32))
    
    // ParseBool
    b1, _ := strconv.ParseBool("true")
    b2, _ := strconv.ParseBool("false")
    b3, _ := strconv.ParseBool("1")
    b4, _ := strconv.ParseBool("0")
    b5, _ := strconv.ParseBool("T")
    b6, _ := strconv.ParseBool("F")
    
    fmt.Printf("ParseBool: %v %v %v %v %v %v\n", b1, b2, b3, b4, b5, b6)
}
```

### 17.2.2 แปลงตัวเลขเป็น String

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Itoa - int to string
    s := strconv.Itoa(42)
    fmt.Printf("Itoa: %q (type: %T)\n", s, s)
    
    // FormatInt
    fmt.Println(strconv.FormatInt(255, 2))   // "11111111" (binary)
    fmt.Println(strconv.FormatInt(255, 8))   // "377" (octal)
    fmt.Println(strconv.FormatInt(255, 10))  // "255" (decimal)
    fmt.Println(strconv.FormatInt(255, 16))  // "ff" (hex)
    
    // FormatFloat
    // 'f' = decimal, 'e' = scientific, 'g' = shortest
    // precision = จำนวนทศนิยม (-1 = ใช้น้อยสุด)
    f := 3.14159265358979
    fmt.Println(strconv.FormatFloat(f, 'f', 2, 64))   // "3.14"
    fmt.Println(strconv.FormatFloat(f, 'f', 6, 64))   // "3.141593"
    fmt.Println(strconv.FormatFloat(f, 'e', 4, 64))   // "3.1416e+00"
    fmt.Println(strconv.FormatFloat(f, 'g', -1, 64))  // "3.14159265358979"
    
    // FormatBool
    fmt.Println(strconv.FormatBool(true))  // "true"
    fmt.Println(strconv.FormatBool(false)) // "false"
    
    // AppendInt - เพิ่มเข้า slice
    buf := []byte("number: ")
    buf = strconv.AppendInt(buf, 42, 10)
    fmt.Printf("AppendInt: %s\n", buf)
    
    // Quote / Unquote
    quoted := strconv.Quote("Hello, \"World\"!")
    fmt.Println("Quoted:", quoted)
    
    unquoted, _ := strconv.Unquote(quoted)
    fmt.Println("Unquoted:", unquoted)
    
    // QuoteToASCII - แปลง non-ASCII เป็น escape sequences
    ascii := strconv.QuoteToASCII("สวัสดี")
    fmt.Println("QuoteToASCII:", ascii)
    
    // CanBackquote
    fmt.Println(strconv.CanBackquote("simple string"))    // true
    fmt.Println(strconv.CanBackquote("has\ttab"))          // false
    fmt.Println(strconv.CanBackquote("has `backtick`"))    // false
}
```

### 17.2.3 Numeric Parsing Patterns

```go
package main

import (
    "fmt"
    "strconv"
)

func parseNumber(s string) (float64, error) {
    // พยายาม parse เป็น int ก่อน ถ้าไม่ได้ก็ float
    if i, err := strconv.ParseInt(s, 10, 64); err == nil {
        return float64(i), nil
    }
    return strconv.ParseFloat(s, 64)
}

func safeAtoi(s string, defaultVal int) int {
    if n, err := strconv.Atoi(s); err == nil {
        return n
    }
    return defaultVal
}

func main() {
    numbers := []string{"42", "3.14", "-100", "1e10", "abc", "0xFF"}
    
    fmt.Println("=== Parse Numbers ===")
    for _, s := range numbers {
        n, err := parseNumber(s)
        if err != nil {
            fmt.Printf("  %-10s -> error: %v\n", s, err)
        } else {
            fmt.Printf("  %-10s -> %.2f\n", s, n)
        }
    }
    
    fmt.Println("\n=== Safe Parse ===")
    inputs := []string{"10", "abc", "99", ""}
    for _, s := range inputs {
        n := safeAtoi(s, 0)
        fmt.Printf("  %q -> %d\n", s, n)
    }
    
    // อ่านจาก query string (ตัวอย่าง web)
    fmt.Println("\n=== Parse Query Parameters ===")
    params := map[string]string{
        "page":  "2",
        "limit": "20",
        "debug": "true",
        "score": "98.5",
    }
    
    page := safeAtoi(params["page"], 1)
    limit := safeAtoi(params["limit"], 10)
    debug, _ := strconv.ParseBool(params["debug"])
    score, _ := strconv.ParseFloat(params["score"], 64)
    
    fmt.Printf("page=%d, limit=%d, debug=%v, score=%.1f\n",
        page, limit, debug, score)
}
```

---

## 17.3 strings.Builder

### 17.3.1 ใช้ Builder สร้าง String

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // strings.Builder - efficient string concatenation
    var b strings.Builder
    
    // เขียนข้อมูล
    b.WriteString("Hello")
    b.WriteString(", ")
    b.WriteString("World")
    b.WriteByte('!')
    b.WriteRune('\n')
    
    // fmt.Fprintf ทำงานกับ Builder
    fmt.Fprintf(&b, "Pi = %.4f\n", 3.14159)
    fmt.Fprintf(&b, "Count = %d\n", 42)
    
    // ดูผลลัพธ์
    result := b.String()
    fmt.Print(result)
    
    fmt.Printf("Length: %d\n", b.Len())
    
    // Reset เพื่อใช้ใหม่
    b.Reset()
    fmt.Printf("After reset, length: %d\n", b.Len())
    
    // สร้าง HTML
    b.WriteString("<ul>\n")
    items := []string{"ข้าวผัด", "ต้มยำ", "ส้มตำ"}
    for _, item := range items {
        fmt.Fprintf(&b, "  <li>%s</li>\n", item)
    }
    b.WriteString("</ul>")
    
    fmt.Println(b.String())
}
```

### 17.3.2 เปรียบเทียบประสิทธิภาพ

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

func concatWithPlus(n int) string {
    result := ""
    for i := 0; i < n; i++ {
        result += fmt.Sprintf("item%d,", i)
    }
    return result
}

func concatWithBuilder(n int) string {
    var b strings.Builder
    for i := 0; i < n; i++ {
        fmt.Fprintf(&b, "item%d,", i)
    }
    return b.String()
}

func concatWithSlice(n int) string {
    parts := make([]string, n)
    for i := 0; i < n; i++ {
        parts[i] = fmt.Sprintf("item%d", i)
    }
    return strings.Join(parts, ",")
}

func benchmark(name string, f func(int) string, n int) {
    start := time.Now()
    result := f(n)
    elapsed := time.Since(start)
    fmt.Printf("%-20s: %v (len=%d)\n", name, elapsed, len(result))
}

func main() {
    n := 10000
    
    benchmark("+ operator", concatWithPlus, n)
    benchmark("strings.Builder", concatWithBuilder, n)
    benchmark("slice + Join", concatWithSlice, n)
    
    // ตัวอย่างการใช้งานจริง: สร้าง SQL query
    fmt.Println("\n=== สร้าง SQL ===")
    
    columns := []string{"id", "name", "email", "age"}
    table := "users"
    where := map[string]interface{}{
        "active": true,
        "age":    18,
    }
    
    var sql strings.Builder
    sql.WriteString("SELECT ")
    sql.WriteString(strings.Join(columns, ", "))
    sql.WriteString(" FROM ")
    sql.WriteString(table)
    
    if len(where) > 0 {
        sql.WriteString(" WHERE ")
        conditions := make([]string, 0, len(where))
        for col, val := range where {
            conditions = append(conditions, fmt.Sprintf("%s = %v", col, val))
        }
        sql.WriteString(strings.Join(conditions, " AND "))
    }
    
    fmt.Println(sql.String())
}
```

---

## 17.4 Regular Expressions (regexp Package)

### 17.4.1 พื้นฐาน regexp

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    // Compile - สร้าง regex pattern
    re, err := regexp.Compile(`\d+`)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    // MatchString - ตรวจสอบว่า match หรือไม่
    fmt.Println(re.MatchString("abc123"))  // true
    fmt.Println(re.MatchString("abc"))     // false
    
    // MustCompile - panic ถ้า pattern ผิด (ใช้สำหรับ static patterns)
    reEmail := regexp.MustCompile(`[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}`)
    
    emails := []string{
        "user@example.com",
        "invalid.email",
        "test@test.co.th",
        "not_email",
    }
    
    for _, e := range emails {
        fmt.Printf("%-25s -> %v\n", e, reEmail.MatchString(e))
    }
    
    // FindString - หา match แรก
    text := "Phone: 081-234-5678, also 02-987-6543"
    rePhone := regexp.MustCompile(`\d{2,3}-\d{3}-\d{4}`)
    
    fmt.Println("\nFirst match:", rePhone.FindString(text))
    
    // FindAllString - หาทุก match
    allMatches := rePhone.FindAllString(text, -1)
    fmt.Println("All matches:", allMatches)
    
    // FindStringIndex - หาตำแหน่ง
    idx := rePhone.FindStringIndex(text)
    fmt.Printf("Index: %v → %q\n", idx, text[idx[0]:idx[1]])
    
    // FindStringSubmatch - กลุ่ม capture
    reDate := regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
    match := reDate.FindStringSubmatch("วันที่: 2024-01-15 และ 2024-06-30")
    if len(match) > 0 {
        fmt.Printf("\nDate: %s (year=%s, month=%s, day=%s)\n",
            match[0], match[1], match[2], match[3])
    }
    
    // FindAllStringSubmatch
    allDates := reDate.FindAllStringSubmatch("2024-01-15 and 2024-06-30", -1)
    fmt.Println("All dates:")
    for _, d := range allDates {
        fmt.Printf("  Full=%s, Year=%s, Month=%s, Day=%s\n",
            d[0], d[1], d[2], d[3])
    }
}
```

### 17.4.2 Replace และ Split ด้วย regexp

```go
package main

import (
    "fmt"
    "regexp"
    "strings"
)

func main() {
    // ReplaceAllString
    re := regexp.MustCompile(`\s+`)
    text := "Hello   World\t\tGo   Lang"
    cleaned := re.ReplaceAllString(text, " ")
    fmt.Println("Cleaned:", cleaned)
    
    // ReplaceAllStringFunc - แทนที่ด้วย function
    reWord := regexp.MustCompile(`\b[a-z]+\b`)
    capitalized := reWord.ReplaceAllStringFunc(
        "hello world go lang",
        func(s string) string {
            return strings.ToUpper(s[:1]) + s[1:]
        },
    )
    fmt.Println("Capitalized:", capitalized)
    
    // ReplaceAllLiteralString - แทนที่แบบ literal
    reDollar := regexp.MustCompile(`\$\w+`)
    template := "Hello $name, your score is $score"
    replaced := reDollar.ReplaceAllLiteralString(template, "[REDACTED]")
    fmt.Println("Replaced:", replaced)
    
    // Split
    reCommaOrSemi := regexp.MustCompile(`[,;]\s*`)
    parts := reCommaOrSemi.Split("a, b; c,d; e", -1)
    fmt.Println("Split:", parts)
    
    // Named groups
    reIP := regexp.MustCompile(`(?P<host>\d+\.\d+\.\d+\.\d+):(?P<port>\d+)`)
    addr := "server at 192.168.1.100:8080"
    
    match := reIP.FindStringSubmatch(addr)
    if match != nil {
        groupNames := reIP.SubexpNames()
        result := make(map[string]string)
        for i, name := range groupNames {
            if i != 0 && name != "" {
                result[name] = match[i]
            }
        }
        fmt.Printf("\nHost: %s, Port: %s\n", result["host"], result["port"])
    }
}
```

### 17.4.3 Regex Patterns ที่ใช้บ่อย

```go
package main

import (
    "fmt"
    "regexp"
)

// Validator struct
type Validator struct {
    patterns map[string]*regexp.Regexp
}

func NewValidator() *Validator {
    return &Validator{
        patterns: map[string]*regexp.Regexp{
            "email":    regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`),
            "phone":    regexp.MustCompile(`^0[6-9]\d{8}$`), // Thai mobile
            "url":      regexp.MustCompile(`^https?://[^\s/$.?#].[^\s]*$`),
            "thai_id":  regexp.MustCompile(`^\d{13}$`),
            "postcode": regexp.MustCompile(`^\d{5}$`),
            "username": regexp.MustCompile(`^[a-zA-Z][a-zA-Z0-9_]{3,19}$`),
            "password": regexp.MustCompile(`^(?=.*[A-Z])(?=.*[a-z])(?=.*\d).{8,}$`),
            "ipv4":     regexp.MustCompile(`^(\d{1,3}\.){3}\d{1,3}$`),
            "date":     regexp.MustCompile(`^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$`),
            "hex":      regexp.MustCompile(`^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$`),
        },
    }
}

func (v *Validator) Validate(name, value string) bool {
    re, ok := v.patterns[name]
    if !ok {
        return false
    }
    return re.MatchString(value)
}

func main() {
    v := NewValidator()
    
    tests := []struct {
        name  string
        value string
    }{
        {"email", "user@example.com"},
        {"email", "invalid.email"},
        {"phone", "0812345678"},
        {"phone", "021234567"},
        {"url", "https://www.example.com"},
        {"url", "not a url"},
        {"postcode", "10110"},
        {"postcode", "abc"},
        {"username", "john_doe123"},
        {"username", "j"},          // too short
        {"date", "2024-01-15"},
        {"date", "2024-13-01"},     // invalid month
        {"hex", "#FF5733"},
        {"hex", "#GGG"},
        {"ipv4", "192.168.1.1"},
        {"ipv4", "999.999.999.999"},
    }
    
    fmt.Printf("%-12s %-30s %s\n", "Type", "Value", "Valid")
    fmt.Println(fmt.Sprintf("%s", "----------------------------------------"))
    
    for _, t := range tests {
        valid := v.Validate(t.name, t.value)
        icon := "✓"
        if !valid {
            icon = "✗"
        }
        fmt.Printf("%-12s %-30s %s\n", t.name, t.value, icon)
    }
}
```

---

## 17.5 Unicode และ Runes

### 17.5.1 Rune vs Byte

```go
package main

import (
    "fmt"
    "unicode"
    "unicode/utf8"
)

func main() {
    s := "Hello, สวัสดี! 🌍"
    
    // len() นับ bytes ไม่ใช่ characters
    fmt.Printf("String: %q\n", s)
    fmt.Printf("len (bytes): %d\n", len(s))
    fmt.Printf("len (runes): %d\n", utf8.RuneCountInString(s))
    
    // วน loop ด้วย range จะได้ rune (Unicode code point)
    fmt.Println("\n--- range loop (runes) ---")
    for i, r := range s {
        if i < 20 { // แสดงแค่ 20 ตัวแรก
            fmt.Printf("  index=%d, rune=%c (U+%04X)\n", i, r, r)
        }
    }
    
    // วน loop ด้วย index จะได้ bytes
    fmt.Println("\n--- byte loop ---")
    for i := 0; i < len(s) && i < 10; i++ {
        fmt.Printf("  s[%d] = 0x%02X\n", i, s[i])
    }
    
    // แปลง string เป็น []rune
    runes := []rune(s)
    fmt.Printf("\nrune count: %d\n", len(runes))
    fmt.Printf("runes[7]: %c\n", runes[7]) // ตัวที่ 8
    
    // แปลง rune เป็น string
    r := '🌍'
    fmt.Printf("rune %c uses %d bytes\n", r, utf8.RuneLen(r))
    
    // unicode package
    fmt.Println("\n--- unicode functions ---")
    chars := []rune{'A', 'a', '5', ' ', '!', 'ก', '中', '🎉'}
    for _, c := range chars {
        fmt.Printf("  %c: IsLetter=%v, IsDigit=%v, IsSpace=%v, IsUpper=%v\n",
            c,
            unicode.IsLetter(c),
            unicode.IsDigit(c),
            unicode.IsSpace(c),
            unicode.IsUpper(c),
        )
    }
    
    // utf8 package
    fmt.Println("\n--- utf8 functions ---")
    thai := "สวัสดี"
    fmt.Printf("RuneCount: %d\n", utf8.RuneCountInString(thai))
    fmt.Printf("Valid UTF-8: %v\n", utf8.ValidString(thai))
    
    // DecodeRuneInString
    r2, size := utf8.DecodeRuneInString(thai)
    fmt.Printf("First rune: %c (size: %d bytes)\n", r2, size)
}
```

### 17.5.2 String Manipulation กับ Unicode

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
    "unicode/utf8"
)

// Reverse string (Unicode-aware)
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// Count Thai characters
func countThai(s string) int {
    count := 0
    for _, r := range s {
        if r >= 0x0E00 && r <= 0x0E7F {
            count++
        }
    }
    return count
}

// Truncate string at rune boundary
func truncate(s string, maxRunes int) string {
    count := 0
    for i, _ := range s {
        if count == maxRunes {
            return s[:i] + "..."
        }
        count++
    }
    return s
}

// Title case for mixed Thai-English
func titleCase(s string) string {
    return strings.Map(func(r rune) rune {
        if unicode.IsLetter(r) {
            return unicode.ToTitle(r)
        }
        return r
    }, s)
}

func main() {
    // Test reverse
    s := "Hello สวัสดี 🌍"
    fmt.Printf("Original: %s\n", s)
    fmt.Printf("Reversed: %s\n", reverseString(s))
    
    // Count Thai
    mixed := "Hello สวัสดีครับ World"
    thaiCount := countThai(mixed)
    totalRunes := utf8.RuneCountInString(mixed)
    fmt.Printf("\nText: %s\n", mixed)
    fmt.Printf("Total runes: %d\n", totalRunes)
    fmt.Printf("Thai chars: %d\n", thaiCount)
    
    // Truncate
    long := "สวัสดีชาวโลกทุกท่าน Go is amazing!"
    fmt.Printf("\nTruncated: %s\n", truncate(long, 10))
    
    // Title case
    fmt.Printf("Title: %s\n", titleCase("hello world"))
    
    // byte slicing vs rune slicing
    thai := "สวัสดี"
    fmt.Printf("\nString: %s\n", thai)
    fmt.Printf("Bytes: %d\n", len(thai))
    fmt.Printf("Runes: %d\n", utf8.RuneCountInString(thai))
    
    // Wrong way (byte slice)
    // ผิด: fmt.Println(thai[:3]) → ไม่สมบูรณ์
    
    // Correct way (rune slice)
    runes := []rune(thai)
    fmt.Printf("First 3 chars: %s\n", string(runes[:3]))
}
```

---

## 17.6 fmt.Sprintf และ Format Verbs

### 17.6.1 Format Verbs ทั้งหมด

```go
package main

import "fmt"

type Point struct {
    X, Y int
}

func main() {
    // General
    fmt.Printf("%%v: %v\n", Point{1, 2})   // {1 2}
    fmt.Printf("%%+v: %+v\n", Point{1, 2}) // {X:1 Y:2}
    fmt.Printf("%%#v: %#v\n", Point{1, 2}) // main.Point{X:1, Y:2}
    fmt.Printf("%%T: %T\n", Point{1, 2})   // main.Point
    fmt.Printf("%%%%: %%\n")               // % (literal %)
    
    // Boolean
    fmt.Printf("%%t: %t, %t\n", true, false) // true, false
    
    // Integer
    n := 255
    fmt.Printf("%%b: %b\n", n)   // 11111111 (binary)
    fmt.Printf("%%c: %c\n", 65)  // A (character)
    fmt.Printf("%%d: %d\n", n)   // 255 (decimal)
    fmt.Printf("%%o: %o\n", n)   // 377 (octal)
    fmt.Printf("%%O: %O\n", n)   // 0o377 (octal with 0o prefix)
    fmt.Printf("%%q: %q\n", 65)  // 'A' (quoted character)
    fmt.Printf("%%x: %x\n", n)   // ff (hex lowercase)
    fmt.Printf("%%X: %X\n", n)   // FF (hex uppercase)
    fmt.Printf("%%U: %U\n", 'A') // U+0041 (Unicode)
    
    // Float
    f := 3.14159265358979
    fmt.Printf("%%e: %e\n", f)    // 3.141593e+00
    fmt.Printf("%%E: %E\n", f)    // 3.141593E+00
    fmt.Printf("%%f: %f\n", f)    // 3.141593
    fmt.Printf("%%.2f: %.2f\n", f) // 3.14
    fmt.Printf("%%g: %g\n", f)    // 3.14159265358979
    fmt.Printf("%%G: %G\n", f)    // 3.14159265358979
    
    // String/Bytes
    s := "Hello"
    fmt.Printf("%%s: %s\n", s)   // Hello
    fmt.Printf("%%q: %q\n", s)   // "Hello"
    fmt.Printf("%%x: %x\n", s)   // 48656c6c6f (hex)
    fmt.Printf("%%X: %X\n", s)   // 48656C6C6F
    
    // Pointer
    x := 42
    fmt.Printf("%%p: %p\n", &x)  // 0xc0000b4008 (address)
}
```

### 17.6.2 Width และ Precision

```go
package main

import "fmt"

func main() {
    // Width - จำนวนตัวอักษรขั้นต่ำ
    fmt.Printf("[%10d]\n", 42)    // [        42] (right-aligned)
    fmt.Printf("[%-10d]\n", 42)   // [42        ] (left-aligned)
    fmt.Printf("[%010d]\n", 42)   // [0000000042] (zero-padded)
    
    // Width กับ string
    fmt.Printf("[%10s]\n", "hi")  // [        hi]
    fmt.Printf("[%-10s]\n", "hi") // [hi        ]
    
    // Precision กับ float
    f := 3.14159
    fmt.Printf("[%10.2f]\n", f)  // [      3.14]
    fmt.Printf("[%-10.2f]\n", f) // [3.14      ]
    
    // Precision กับ string (max characters)
    fmt.Printf("[%.5s]\n", "Hello, World") // [Hello]
    
    // * สำหรับ dynamic width/precision
    width := 10
    prec := 3
    fmt.Printf("[%*.*f]\n", width, prec, f) // [     3.142]
    
    // ตาราง
    fmt.Printf("\n%-15s %10s %8s\n", "ชื่อสินค้า", "ราคา", "จำนวน")
    fmt.Println("-----------------------------------")
    
    products := []struct {
        name  string
        price float64
        qty   int
    }{
        {"iPhone 15", 35000.00, 50},
        {"Samsung S24", 28000.50, 75},
        {"iPad Air", 22000.00, 100},
    }
    
    for _, p := range products {
        fmt.Printf("%-15s %10.2f %8d\n", p.name, p.price, p.qty)
    }
}
```

### 17.6.3 Sprintf, Fprintf, Errorf

```go
package main

import (
    "fmt"
    "os"
)

func formatReport(title string, items []struct{ Name string; Value int }) string {
    var result string
    result += fmt.Sprintf("=== %s ===\n", title)
    result += fmt.Sprintf("%-20s %10s\n", "Item", "Value")
    result += fmt.Sprintf("%s\n", fmt.Sprintf("%-30s", "")[0:30])
    
    total := 0
    for _, item := range items {
        result += fmt.Sprintf("%-20s %10d\n", item.Name, item.Value)
        total += item.Value
    }
    
    result += fmt.Sprintf("%s\n", fmt.Sprintf("%-30s", "")[0:30])
    result += fmt.Sprintf("%-20s %10d\n", "Total", total)
    
    return result
}

func main() {
    // Sprintf - format เป็น string
    msg := fmt.Sprintf("Hello, %s! You have %d messages.", "Alice", 5)
    fmt.Println(msg)
    
    // Fprintf - format ไปยัง writer
    fmt.Fprintf(os.Stdout, "Today is %s\n", "Monday")
    
    // Errorf - สร้าง error ด้วย format
    id := 404
    err := fmt.Errorf("user with id %d not found", id)
    fmt.Println("Error:", err)
    
    // %w - wrap error
    originalErr := fmt.Errorf("database connection failed")
    wrappedErr := fmt.Errorf("getUserById(%d): %w", id, originalErr)
    fmt.Println("Wrapped error:", wrappedErr)
    
    // Stringer interface
    items := []struct {
        Name  string
        Value int
    }{
        {"Apple", 100},
        {"Banana", 50},
        {"Cherry", 200},
        {"Date", 75},
    }
    
    report := formatReport("Sales Report", items)
    fmt.Println(report)
    
    // Sprint, Sprintln
    s1 := fmt.Sprint("a", "b", "c")       // "abc"
    s2 := fmt.Sprintln("a", "b", "c")     // "a b c\n"
    s3 := fmt.Sprintf("%d+%d=%d", 1, 2, 3) // "1+2=3"
    
    fmt.Printf("Sprint: %q\n", s1)
    fmt.Printf("Sprintln: %q\n", s2)
    fmt.Printf("Sprintf: %q\n", s3)
}
```

---

## 17.7 Workshop: Text Processing Tool

```go
package main

import (
    "bufio"
    "fmt"
    "regexp"
    "sort"
    "strconv"
    "strings"
    "unicode"
)

type TextStats struct {
    Lines      int
    Words      int
    Chars      int
    Sentences  int
    Paragraphs int
    WordFreq   map[string]int
}

type TextProcessor struct {
    text    string
    stats   TextStats
    reWord  *regexp.Regexp
    reSent  *regexp.Regexp
}

func NewTextProcessor(text string) *TextProcessor {
    tp := &TextProcessor{
        text: text,
        stats: TextStats{
            WordFreq: make(map[string]int),
        },
        reWord: regexp.MustCompile(`\b\w+\b`),
        reSent: regexp.MustCompile(`[.!?]+`),
    }
    tp.analyze()
    return tp
}

func (tp *TextProcessor) analyze() {
    scanner := bufio.NewScanner(strings.NewReader(tp.text))
    
    for scanner.Scan() {
        line := scanner.Text()
        tp.stats.Lines++
        
        // นับ characters
        tp.stats.Chars += len([]rune(line))
        
        // นับ words
        words := tp.reWord.FindAllString(strings.ToLower(line), -1)
        tp.stats.Words += len(words)
        for _, w := range words {
            tp.stats.WordFreq[w]++
        }
        
        // นับ sentences
        tp.stats.Sentences += len(tp.reSent.FindAllString(line, -1))
    }
    
    // นับ paragraphs (บรรทัดว่างคั่น)
    paragraphs := strings.Split(tp.text, "\n\n")
    for _, p := range paragraphs {
        if strings.TrimSpace(p) != "" {
            tp.stats.Paragraphs++
        }
    }
}

func (tp *TextProcessor) TopWords(n int) []struct {
    Word  string
    Count int
} {
    type wordCount struct {
        Word  string
        Count int
    }
    
    var wcs []wordCount
    for word, count := range tp.stats.WordFreq {
        // ข้าม stopwords
        stopwords := map[string]bool{
            "the": true, "a": true, "an": true,
            "and": true, "or": true, "is": true, "are": true,
            "in": true, "on": true, "at": true, "to": true,
        }
        if !stopwords[word] && len(word) > 2 {
            wcs = append(wcs, wordCount{word, count})
        }
    }
    
    sort.Slice(wcs, func(i, j int) bool {
        return wcs[i].Count > wcs[j].Count
    })
    
    if n > len(wcs) {
        n = len(wcs)
    }
    
    result := make([]struct {
        Word  string
        Count int
    }, n)
    for i, wc := range wcs[:n] {
        result[i].Word = wc.Word
        result[i].Count = wc.Count
    }
    return result
}

func (tp *TextProcessor) Summary() string {
    var b strings.Builder
    
    fmt.Fprintf(&b, "=== Text Analysis ===\n")
    fmt.Fprintf(&b, "%-15s: %s\n", "Lines", strconv.Itoa(tp.stats.Lines))
    fmt.Fprintf(&b, "%-15s: %s\n", "Words", strconv.Itoa(tp.stats.Words))
    fmt.Fprintf(&b, "%-15s: %s\n", "Characters", strconv.Itoa(tp.stats.Chars))
    fmt.Fprintf(&b, "%-15s: %s\n", "Sentences", strconv.Itoa(tp.stats.Sentences))
    fmt.Fprintf(&b, "%-15s: %s\n", "Paragraphs", strconv.Itoa(tp.stats.Paragraphs))
    fmt.Fprintf(&b, "%-15s: %s\n", "Unique Words", strconv.Itoa(len(tp.stats.WordFreq)))
    
    avgWordsPerSent := 0.0
    if tp.stats.Sentences > 0 {
        avgWordsPerSent = float64(tp.stats.Words) / float64(tp.stats.Sentences)
    }
    fmt.Fprintf(&b, "%-15s: %.1f\n", "Avg Words/Sent", avgWordsPerSent)
    
    return b.String()
}

func (tp *TextProcessor) ExtractEmails() []string {
    reEmail := regexp.MustCompile(`[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}`)
    return reEmail.FindAllString(tp.text, -1)
}

func (tp *TextProcessor) ExtractURLs() []string {
    reURL := regexp.MustCompile(`https?://[^\s<>"{}|\\^\[\]` + "`" + `]+`)
    return reURL.FindAllString(tp.text, -1)
}

func (tp *TextProcessor) Sanitize() string {
    // ลบ special characters ที่ไม่ต้องการ
    reSpecial := regexp.MustCompile(`[^\p{L}\p{N}\s.,!?;:'"-]`)
    cleaned := reSpecial.ReplaceAllString(tp.text, "")
    
    // normalize whitespace
    reSpace := regexp.MustCompile(`\s+`)
    cleaned = reSpace.ReplaceAllString(cleaned, " ")
    
    return strings.TrimSpace(cleaned)
}

func (tp *TextProcessor) WordWrap(width int) string {
    words := strings.Fields(tp.text)
    var lines []string
    var current strings.Builder
    
    lineLen := 0
    for _, word := range words {
        wordRunes := len([]rune(word))
        
        if lineLen > 0 && lineLen+1+wordRunes > width {
            lines = append(lines, current.String())
            current.Reset()
            lineLen = 0
        }
        
        if lineLen > 0 {
            current.WriteByte(' ')
            lineLen++
        }
        current.WriteString(word)
        lineLen += wordRunes
    }
    
    if current.Len() > 0 {
        lines = append(lines, current.String())
    }
    
    return strings.Join(lines, "\n")
}

func (tp *TextProcessor) ToTitleCase() string {
    words := strings.Fields(tp.text)
    for i, word := range words {
        if len(word) == 0 {
            continue
        }
        runes := []rune(word)
        runes[0] = unicode.ToUpper(runes[0])
        words[i] = string(runes)
    }
    return strings.Join(words, " ")
}

func main() {
    text := `Go is an open source programming language that makes it easy to build simple, reliable, and efficient software.

Go was designed at Google in 2007 to improve programming productivity. The language is statically typed and compiled. Go has garbage collection, limited structural typing, memory safety features and CSP-style concurrent programming.

Contact us at info@golang.org or visit https://golang.org for more information. You can also reach us at support@google.com.

Go is awesome! We love Go. Go makes programming fun again.`
    
    tp := NewTextProcessor(text)
    
    // แสดงสถิติ
    fmt.Println(tp.Summary())
    
    // Top words
    fmt.Println("=== Top 5 Words ===")
    for _, wc := range tp.TopWords(5) {
        fmt.Printf("  %-15s: %d\n", wc.Word, wc.Count)
    }
    
    // Extract
    fmt.Println("\n=== Emails ===")
    for _, e := range tp.ExtractEmails() {
        fmt.Println(" ", e)
    }
    
    fmt.Println("\n=== URLs ===")
    for _, u := range tp.ExtractURLs() {
        fmt.Println(" ", u)
    }
    
    // Title case
    short := "hello world go programming"
    fmt.Printf("\nTitle case: %s\n", NewTextProcessor(short).ToTitleCase())
    
    // Word wrap
    fmt.Println("\n=== Word Wrap (40 chars) ===")
    paragraph := "Go is an open source programming language that makes it easy to build software."
    wrapped := NewTextProcessor(paragraph).WordWrap(40)
    fmt.Println(wrapped)
}
```

---

## สรุป

| Package | ฟังก์ชันสำคัญ |
|---------|-------------|
| `strings` | `Contains`, `HasPrefix`, `Split`, `Join`, `Replace`, `TrimSpace` |
| `strconv` | `Atoi`, `Itoa`, `ParseFloat`, `FormatInt` |
| `strings.Builder` | `WriteString`, `WriteByte`, `String`, `Reset` |
| `regexp` | `Compile`, `MatchString`, `FindAllString`, `ReplaceAllString` |
| `unicode` | `IsLetter`, `IsDigit`, `ToUpper`, `ToLower` |
| `unicode/utf8` | `RuneCountInString`, `DecodeRuneInString` |

## Resources

- [strings package](https://pkg.go.dev/strings)
- [strconv package](https://pkg.go.dev/strconv)
- [regexp package](https://pkg.go.dev/regexp)
- [regexp syntax](https://pkg.go.dev/regexp/syntax)
- [fmt package](https://pkg.go.dev/fmt)
- [Go Regex101 Tester](https://regex101.com/?flavor=golang)
