# Part 95: Architecture Review and Technical Decision Making

## เป้าหมายของบทเรียน
- Architecture review process
- Architecture Decision Records (ADRs)
- Technical debt management
- Refactoring strategies
- Migration planning

---

## 1. Architecture Decision Records (ADRs)

```go
// adr_generator.go - สร้าง ADR templates
package main

import (
    "fmt"
    "os"
    "strings"
    "time"
)

// ADR structure
type ADR struct {
    Number     int
    Title      string
    Date       string
    Status     string // Proposed, Accepted, Deprecated, Superseded
    Context    string
    Decision   string
    Rationale  string
    Consequences []string
    Alternatives []Alternative
}

// Alternative ตัวเลือกที่พิจารณา
type Alternative struct {
    Name string
    Pros []string
    Cons []string
}

// Generate สร้าง ADR markdown
func (a *ADR) Generate() string {
    var sb strings.Builder
    
    sb.WriteString(fmt.Sprintf("# ADR-%03d: %s\n\n", a.Number, a.Title))
    sb.WriteString(fmt.Sprintf("**Date:** %s\n\n", a.Date))
    sb.WriteString(fmt.Sprintf("**Status:** %s\n\n", a.Status))
    
    sb.WriteString("## Context\n\n")
    sb.WriteString(a.Context + "\n\n")
    
    sb.WriteString("## Decision\n\n")
    sb.WriteString(a.Decision + "\n\n")
    
    sb.WriteString("## Rationale\n\n")
    sb.WriteString(a.Rationale + "\n\n")
    
    if len(a.Alternatives) > 0 {
        sb.WriteString("## Alternatives Considered\n\n")
        for _, alt := range a.Alternatives {
            sb.WriteString(fmt.Sprintf("### %s\n\n", alt.Name))
            
            if len(alt.Pros) > 0 {
                sb.WriteString("**Pros:**\n")
                for _, pro := range alt.Pros {
                    sb.WriteString(fmt.Sprintf("- %s\n", pro))
                }
                sb.WriteString("\n")
            }
            
            if len(alt.Cons) > 0 {
                sb.WriteString("**Cons:**\n")
                for _, con := range alt.Cons {
                    sb.WriteString(fmt.Sprintf("- %s\n", con))
                }
                sb.WriteString("\n")
            }
        }
    }
    
    if len(a.Consequences) > 0 {
        sb.WriteString("## Consequences\n\n")
        for _, c := range a.Consequences {
            sb.WriteString(fmt.Sprintf("- %s\n", c))
        }
        sb.WriteString("\n")
    }
    
    return sb.String()
}

func main() {
    fmt.Println("=== ADR Generator Demo ===\n")
    
    adr := &ADR{
        Number: 1,
        Title:  "Use PostgreSQL as Primary Database",
        Date:   time.Now().Format("2006-01-02"),
        Status: "Accepted",
        Context: `Our system needs a reliable, scalable relational database.
We are building an e-commerce platform that requires ACID transactions,
complex queries, and strong consistency.`,
        Decision: `We will use PostgreSQL as our primary database for all
transactional data. We will use version 15+ with connection pooling
via PgBouncer.`,
        Rationale: `PostgreSQL offers excellent ACID compliance, JSON support
for flexible schemas, strong community, and proven scalability.
The team has prior experience with PostgreSQL.`,
        Alternatives: []Alternative{
            {
                Name: "MySQL",
                Pros: []string{"Widely known", "Good performance"},
                Cons: []string{"Weaker JSON support", "Less feature-rich"},
            },
            {
                Name: "MongoDB",
                Pros: []string{"Flexible schema", "Easy scaling"},
                Cons: []string{"No ACID for multi-doc", "Eventual consistency"},
            },
            {
                Name: "CockroachDB",
                Pros: []string{"Distributed", "PostgreSQL compatible"},
                Cons: []string{"Higher operational complexity", "Cost"},
            },
        },
        Consequences: []string{
            "Team needs PostgreSQL expertise",
            "Need PgBouncer for connection pooling",
            "Schema migrations with golang-migrate",
            "Backup strategy with pg_dump/PITR",
        },
    }
    
    content := adr.Generate()
    fmt.Println(content)
    
    // ในโปรดักชัน จะเขียนไปยัง docs/adr/
    filename := fmt.Sprintf("adr-%03d-%s.md", adr.Number,
        strings.ReplaceAll(strings.ToLower(adr.Title), " ", "-"))
    
    err := os.WriteFile("/tmp/"+filename, []byte(content), 0644)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("ADR saved to: /tmp/%s\n", filename)
    }
}
```

---

## 2. Technical Debt Assessment

```go
// tech_debt.go - วิเคราะห์ technical debt
package main

import (
    "fmt"
    "sort"
    "strings"
)

// DebtCategory ประเภทของ technical debt
type DebtCategory string

const (
    DebtCode          DebtCategory = "Code Quality"
    DebtArchitecture  DebtCategory = "Architecture"
    DebtTesting       DebtCategory = "Testing"
    DebtDocumentation DebtCategory = "Documentation"
    DebtDependency    DebtCategory = "Dependencies"
    DebtSecurity      DebtCategory = "Security"
    DebtPerformance   DebtCategory = "Performance"
)

// DebtItem รายการ technical debt
type DebtItem struct {
    ID          string
    Category    DebtCategory
    Title       string
    Description string
    Impact      int    // 1-5
    Effort      int    // 1-5 (1=small, 5=large)
    Risk        int    // 1-5
    File        string
    Tags        []string
}

// Priority คำนวณ priority score
func (d *DebtItem) Priority() float64 {
    // Higher impact + lower effort = higher priority
    return float64(d.Impact*d.Risk) / float64(d.Effort)
}

// DebtRegister ทะเบียน technical debt
type DebtRegister struct {
    items []*DebtItem
}

// Add เพิ่ม debt item
func (r *DebtRegister) Add(item *DebtItem) {
    r.items = append(r.items, item)
}

// ByPriority sort โดย priority
func (r *DebtRegister) ByPriority() []*DebtItem {
    sorted := make([]*DebtItem, len(r.items))
    copy(sorted, r.items)
    
    sort.Slice(sorted, func(i, j int) bool {
        return sorted[i].Priority() > sorted[j].Priority()
    })
    
    return sorted
}

// ByCategory group โดย category
func (r *DebtRegister) ByCategory() map[DebtCategory][]*DebtItem {
    result := make(map[DebtCategory][]*DebtItem)
    
    for _, item := range r.items {
        result[item.Category] = append(result[item.Category], item)
    }
    
    return result
}

// TotalScore คำนวณ total debt score
func (r *DebtRegister) TotalScore() float64 {
    score := 0.0
    for _, item := range r.items {
        score += float64(item.Impact * item.Risk)
    }
    return score
}

// Print แสดงรายงาน
func (r *DebtRegister) Print() {
    fmt.Printf("=== Technical Debt Register ===\n\n")
    fmt.Printf("Total items: %d\n", len(r.items))
    fmt.Printf("Total score: %.1f\n\n", r.TotalScore())
    
    fmt.Printf("%-6s %-20s %-15s %-5s %-6s %-5s %-8s\n",
        "ID", "Title", "Category", "Imp", "Effort", "Risk", "Priority")
    fmt.Println(strings.Repeat("-", 75))
    
    for _, item := range r.ByPriority() {
        fmt.Printf("%-6s %-20s %-15s %-5d %-6d %-5d %.2f\n",
            item.ID,
            item.Title[:min(len(item.Title), 19)],
            string(item.Category)[:min(len(string(item.Category)), 14)],
            item.Impact,
            item.Effort,
            item.Risk,
            item.Priority(),
        )
    }
}

func main() {
    register := &DebtRegister{}
    
    register.Add(&DebtItem{
        ID:          "TD-001",
        Category:    DebtCode,
        Title:       "God class in UserService",
        Description: "UserService has 2000+ lines, handles auth, profile, permissions",
        Impact:      5,
        Effort:      4,
        Risk:        4,
        File:        "internal/user/service.go",
        Tags:        []string{"refactoring", "solid"},
    })
    
    register.Add(&DebtItem{
        ID:          "TD-002",
        Category:    DebtTesting,
        Title:       "Missing integration tests",
        Description: "Payment service has 0% integration test coverage",
        Impact:      5,
        Effort:      3,
        Risk:        5,
        File:        "internal/payment/",
        Tags:        []string{"testing", "coverage"},
    })
    
    register.Add(&DebtItem{
        ID:          "TD-003",
        Category:    DebtDependency,
        Title:       "Outdated dependencies",
        Description: "20+ packages 2+ years old, some with CVEs",
        Impact:      4,
        Effort:      2,
        Risk:        4,
        File:        "go.mod",
        Tags:        []string{"security", "dependencies"},
    })
    
    register.Add(&DebtItem{
        ID:          "TD-004",
        Category:    DebtArchitecture,
        Title:       "Circular package imports",
        Description: "pkg A imports pkg B which imports pkg A",
        Impact:      3,
        Effort:      3,
        Risk:        3,
        Tags:        []string{"architecture", "coupling"},
    })
    
    register.Add(&DebtItem{
        ID:          "TD-005",
        Category:    DebtSecurity,
        Title:       "SQL injection in search",
        Description: "Search endpoint uses string concatenation",
        Impact:      5,
        Effort:      1,
        Risk:        5,
        File:        "internal/search/handler.go:45",
        Tags:        []string{"security", "critical"},
    })
    
    register.Print()
    
    fmt.Println("\n=== Category Breakdown ===\n")
    for category, items := range register.ByCategory() {
        fmt.Printf("%s: %d items\n", category, len(items))
    }
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

---

## 3. Refactoring Strategy

```go
// refactoring.go - Refactoring techniques
package main

import (
    "fmt"
)

/*
Refactoring Principles:

1. Boy Scout Rule: Leave code better than you found it
2. Don't break existing behavior
3. Small, incremental changes
4. Test-driven refactoring
5. One thing at a time

Common Refactorings:
- Extract Function/Method
- Rename Variable/Function
- Move Function to Package
- Replace Magic Numbers with Constants
- Replace Conditional with Polymorphism
- Introduce Parameter Object
- Extract Interface
- Replace Error Code with Exception
*/

// BEFORE: Messy function (bad example)
func processOrderBad(customerID int, items []map[string]interface{}, 
    discount float64, isVIP bool, shippingAddr string) (float64, error) {
    
    total := 0.0
    for _, item := range items {
        price := item["price"].(float64)
        qty := item["quantity"].(int)
        total += price * float64(qty)
    }
    
    if discount > 0 {
        total = total * (1 - discount)
    }
    
    if isVIP && total > 1000 {
        total = total * 0.9 // extra 10% for VIP
    }
    
    if total < 0 {
        return 0, fmt.Errorf("invalid total")
    }
    
    // shipping calculation mixed in...
    shipping := 50.0
    if total > 500 {
        shipping = 0
    }
    
    return total + shipping, nil
}

// AFTER: Refactored (good example)

// OrderItem ข้อมูล item ใน order
type OrderItem struct {
    ProductID int
    Name      string
    Price     float64
    Quantity  int
}

// OrderRequest order request
type OrderRequest struct {
    CustomerID  int
    Items       []OrderItem
    ShippingAddr string
}

// Customer ข้อมูล customer
type Customer struct {
    ID    int
    Name  string
    IsVIP bool
}

// PricingService คำนวณราคา
type PricingService struct {
    BaseShipping    float64
    FreeShippingMin float64
    VIPDiscount     float64
    VIPMinOrder     float64
}

// NewPricingService สร้าง service
func NewPricingService() *PricingService {
    return &PricingService{
        BaseShipping:    50.0,
        FreeShippingMin: 500.0,
        VIPDiscount:     0.10,
        VIPMinOrder:     1000.0,
    }
}

// CalculateSubtotal คำนวณ subtotal
func (s *PricingService) CalculateSubtotal(items []OrderItem) float64 {
    total := 0.0
    for _, item := range items {
        total += item.Price * float64(item.Quantity)
    }
    return total
}

// ApplyDiscount ลดราคา
func (s *PricingService) ApplyDiscount(price, discount float64) float64 {
    if discount <= 0 || discount >= 1 {
        return price
    }
    return price * (1 - discount)
}

// ApplyVIPDiscount ลดราคา VIP
func (s *PricingService) ApplyVIPDiscount(price float64, customer Customer) float64 {
    if customer.IsVIP && price >= s.VIPMinOrder {
        return price * (1 - s.VIPDiscount)
    }
    return price
}

// CalculateShipping คำนวณค่าส่ง
func (s *PricingService) CalculateShipping(subtotal float64) float64 {
    if subtotal >= s.FreeShippingMin {
        return 0
    }
    return s.BaseShipping
}

// CalculateTotal คำนวณราคารวม
func (s *PricingService) CalculateTotal(
    req OrderRequest, customer Customer, discount float64,
) (float64, error) {
    
    subtotal := s.CalculateSubtotal(req.Items)
    subtotal = s.ApplyDiscount(subtotal, discount)
    subtotal = s.ApplyVIPDiscount(subtotal, customer)
    
    if subtotal < 0 {
        return 0, fmt.Errorf("invalid order total: %.2f", subtotal)
    }
    
    shipping := s.CalculateShipping(subtotal)
    return subtotal + shipping, nil
}

func main() {
    fmt.Println("=== Refactoring Demo ===\n")
    
    svc := NewPricingService()
    
    customer := Customer{ID: 1, Name: "สมชาย", IsVIP: true}
    
    req := OrderRequest{
        CustomerID:   customer.ID,
        ShippingAddr: "123 Main St",
        Items: []OrderItem{
            {ProductID: 1, Name: "Go Book", Price: 350.0, Quantity: 2},
            {ProductID: 2, Name: "Keyboard", Price: 500.0, Quantity: 1},
        },
    }
    
    total, err := svc.CalculateTotal(req, customer, 0.05)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Customer: %s (VIP=%v)\n", customer.Name, customer.IsVIP)
    fmt.Printf("Subtotal: %.2f\n", svc.CalculateSubtotal(req.Items))
    fmt.Printf("Total (with discounts): %.2f\n", total)
}
```

---

## สรุป

บทนี้ครอบคลุม Architecture Review:

1. **ADRs** - บันทึกการตัดสินใจทางสถาปัตยกรรม
2. **Technical Debt** - วิเคราะห์และจัดลำดับความสำคัญ
3. **Refactoring** - Extract functions, separate concerns
4. **Migration Planning** - วางแผน migration ที่ปลอดภัย

### Key Takeaways

- ADRs ช่วย preserve context ของการตัดสินใจ
- Technical debt ต้องวัดและจัดลำดับ ไม่ใช่แค่บ่น
- Refactoring = เปลี่ยน structure โดยไม่เปลี่ยน behavior
- Test ก่อน refactor เสมอ
- Migration ต้องมี rollback plan
