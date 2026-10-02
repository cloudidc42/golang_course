# Part 97: Tech Lead และ Team Leadership

## เป้าหมายของบทเรียน
- Tech Lead responsibilities
- Code review culture
- Go style guide
- Onboarding new developers
- Team processes and rituals

---

## 1. Tech Lead Responsibilities

```go
// tech_lead.go - Tech Lead framework
package main

import (
    "fmt"
    "strings"
    "time"
)

/*
Tech Lead ไม่ใช่แค่ Senior Developer
Tech Lead = Technical Leadership + People Skills

Responsibilities:
1. Technical Vision & Architecture
2. Code Quality & Standards
3. Developer Mentoring
4. Cross-team Communication
5. Technical Risk Management
6. Hiring Support
*/

// TechLead responsibilities tracker
type TechLead struct {
    Name        string
    Team        string
    WeeklyTasks []WeeklyTask
}

type WeeklyTask struct {
    Category string
    Hours    float64
    Tasks    []string
}

func NewTechLead(name, team string) *TechLead {
    return &TechLead{
        Name: name,
        Team: team,
        WeeklyTasks: []WeeklyTask{
            {
                Category: "Code Reviews",
                Hours:    8,
                Tasks: []string{
                    "Review PRs ทุกวัน",
                    "Give constructive feedback",
                    "Ensure standards compliance",
                    "Unblock team members",
                },
            },
            {
                Category: "Architecture & Design",
                Hours:    6,
                Tasks: []string{
                    "Design system components",
                    "Write/review ADRs",
                    "Evaluate new technologies",
                    "Technical planning",
                },
            },
            {
                Category: "Mentoring",
                Hours:    5,
                Tasks: []string{
                    "1:1 meetings",
                    "Pair programming",
                    "Knowledge sharing sessions",
                    "Career guidance",
                },
            },
            {
                Category: "Project/Process",
                Hours:    6,
                Tasks: []string{
                    "Sprint planning",
                    "Technical estimation",
                    "Stakeholder communication",
                    "Incident response",
                },
            },
            {
                Category: "Own Development",
                Hours:    15,
                Tasks: []string{
                    "Feature development",
                    "Proof of concepts",
                    "Bug fixes",
                    "Infrastructure improvements",
                },
            },
        },
    }
}

func (tl *TechLead) PrintSchedule() {
    fmt.Printf("=== Tech Lead Weekly Schedule: %s ===\n\n", tl.Name)
    
    totalHours := 0.0
    for _, task := range tl.WeeklyTasks {
        totalHours += task.Hours
    }
    
    for _, task := range tl.WeeklyTasks {
        percent := task.Hours / totalHours * 100
        bar := strings.Repeat("█", int(percent/5))
        fmt.Printf("%-25s %4.1fh (%3.0f%%) %s\n",
            task.Category, task.Hours, percent, bar)
        for _, t := range task.Tasks {
            fmt.Printf("  - %s\n", t)
        }
        fmt.Println()
    }
    
    fmt.Printf("Total: %.1f hours/week\n", totalHours)
}

// TechDebt tracking
type TechDebtTracker struct {
    items []TechDebtItem
}

type TechDebtItem struct {
    ID          string
    Description string
    AddedAt     time.Time
    AddedBy     string
    Severity    string
    Resolved    bool
    ResolvedAt  *time.Time
}

func (t *TechDebtTracker) Add(item TechDebtItem) {
    item.AddedAt = time.Now()
    t.items = append(t.items, item)
}

func (t *TechDebtTracker) Open() []TechDebtItem {
    var result []TechDebtItem
    for _, item := range t.items {
        if !item.Resolved {
            result = append(result, item)
        }
    }
    return result
}

func main() {
    tl := NewTechLead("สมชาย", "Platform Team")
    tl.PrintSchedule()
    
    fmt.Println("\n=== Tech Lead Anti-Patterns ===\n")
    
    antiPatterns := []struct {
        pattern string
        symptom string
        fix     string
    }{
        {
            pattern: "Bottleneck",
            symptom: "All decisions go through you",
            fix:     "Delegate, document standards, trust team",
        },
        {
            pattern: "Hero Developer",
            symptom: "Only you can fix critical bugs",
            fix:     "Pair programming, knowledge sharing, rotation",
        },
        {
            pattern: "Micro-management",
            symptom: "Reviewing every line, blocking progress",
            fix:     "Define clear standards, trust but verify",
        },
        {
            pattern: "Ivory Tower",
            symptom: "Architecture too complex for team",
            fix:     "Co-design with team, validate with prototypes",
        },
        {
            pattern: "Yes Person",
            symptom: "Accepting all feature requests, no pushback",
            fix:     "Learn to say no with data and alternatives",
        },
    }
    
    for _, ap := range antiPatterns {
        fmt.Printf("Anti-pattern: %s\n", ap.pattern)
        fmt.Printf("  Symptom: %s\n", ap.symptom)
        fmt.Printf("  Fix:     %s\n\n", ap.fix)
    }
}
```

---

## 2. Go Style Guide

```go
// style_guide.go - Go coding standards
package main

import (
    "errors"
    "fmt"
)

/*
Go Style Guide สำหรับ Team:
Based on: https://google.github.io/styleguide/go/

1. Naming Conventions
2. Error Handling
3. Package Organization
4. Comments/Documentation
5. Testing
*/

// === NAMING CONVENTIONS ===

// GOOD: Short, descriptive names
type User struct {
    ID    int
    Name  string
    Email string
}

// BAD: Hungarian notation or type in name
type UserStruct struct {
    UserId    int    // avoid "Id" -> use "ID"
    UserName  string // redundant "User" prefix
    UserEmail string
}

// GOOD: Interface names end with -er
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Stringer interface {
    String() string
}

// GOOD: Error variables start with Err
var (
    ErrNotFound    = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

// GOOD: Unexported error types for wrapping
type NotFoundError struct {
    Resource string
    ID       int
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with id %d not found", e.Resource, e.ID)
}

// === ERROR HANDLING PATTERNS ===

// GOOD: Return errors, don't log them in library code
func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("invalid user id %d: %w", id, ErrInvalidInput)
    }
    
    // Simulate DB lookup
    if id > 1000 {
        return nil, &NotFoundError{Resource: "user", ID: id}
    }
    
    return &User{ID: id, Name: fmt.Sprintf("User%d", id)}, nil
}

// GOOD: Wrap errors with context
func getUserProfile(id int) (*User, error) {
    user, err := findUser(id)
    if err != nil {
        return nil, fmt.Errorf("getUserProfile: %w", err)
    }
    return user, nil
}

// GOOD: Use errors.Is and errors.As
func handleUserRequest(id int) {
    user, err := getUserProfile(id)
    
    if err != nil {
        if errors.Is(err, ErrInvalidInput) {
            fmt.Printf("Bad request: %v\n", err)
            return
        }
        
        var notFound *NotFoundError
        if errors.As(err, &notFound) {
            fmt.Printf("Resource not found: %s/%d\n", notFound.Resource, notFound.ID)
            return
        }
        
        fmt.Printf("Internal error: %v\n", err)
        return
    }
    
    fmt.Printf("Got user: %s\n", user.Name)
}

// === PACKAGE ORGANIZATION ===
/*
Recommended package structure:

project/
├── cmd/           - executables
│   └── server/
│       └── main.go
├── internal/      - private packages
│   ├── user/      - domain package
│   │   ├── user.go      - types
│   │   ├── service.go   - business logic
│   │   ├── repository.go - data access
│   │   └── handler.go   - HTTP handlers
│   └── auth/
├── pkg/           - public packages
│   └── httputil/
├── api/           - API definitions
│   └── v1/
└── docs/          - documentation
*/

// === COMMENTS ===

// Package main demonstrates Go style guide.
// Every exported type, function, and method should have a comment.

// User represents a system user with authentication credentials.
// It implements the Stringer interface for display purposes.
type UserWithComment struct {
    // ID is the unique identifier (database primary key).
    ID int

    // Name is the user's display name.
    // It must be between 2 and 50 characters.
    Name string
}

// String implements fmt.Stringer.
func (u *UserWithComment) String() string {
    return fmt.Sprintf("User{ID:%d, Name:%s}", u.ID, u.Name)
}

// Validate checks that the user has valid field values.
// It returns an error if any validation fails.
func (u *UserWithComment) Validate() error {
    if u.ID <= 0 {
        return fmt.Errorf("ID must be positive, got %d", u.ID)
    }
    if len(u.Name) < 2 || len(u.Name) > 50 {
        return fmt.Errorf("Name must be 2-50 chars, got %d", len(u.Name))
    }
    return nil
}

func main() {
    fmt.Println("=== Go Style Guide Demo ===\n")
    
    // Error handling demo
    fmt.Println("Error Handling:")
    
    testCases := []int{-1, 0, 500, 2000}
    for _, id := range testCases {
        fmt.Printf("  handleUserRequest(%d):\n", id)
        fmt.Print("    ")
        handleUserRequest(id)
    }
    
    // Validation demo
    fmt.Println("\nValidation:")
    
    users := []*UserWithComment{
        {ID: 1, Name: "สมชาย"},
        {ID: 0, Name: "X"},        // invalid
        {ID: 2, Name: ""},         // invalid
    }
    
    for _, u := range users {
        if err := u.Validate(); err != nil {
            fmt.Printf("  Invalid: %v\n", err)
        } else {
            fmt.Printf("  Valid: %s\n", u)
        }
    }
}
```

---

## 3. Onboarding Template

```go
// onboarding.go - Developer onboarding guide
package main

import (
    "fmt"
    "strings"
    "time"
)

// OnboardingPlan แผน onboarding
type OnboardingPlan struct {
    Developer string
    StartDate time.Time
    Weeks     []OnboardingWeek
}

type OnboardingWeek struct {
    Week   int
    Theme  string
    Tasks  []OnboardingTask
}

type OnboardingTask struct {
    Category string
    Task     string
    Duration string
    Done     bool
    Owner    string
}

func NewOnboardingPlan(developer string) *OnboardingPlan {
    return &OnboardingPlan{
        Developer: developer,
        StartDate: time.Now(),
        Weeks: []OnboardingWeek{
            {
                Week:  1,
                Theme: "Setup & Orientation",
                Tasks: []OnboardingTask{
                    {Category: "Setup", Task: "Setup development environment", Duration: "2h", Owner: "Team"},
                    {Category: "Setup", Task: "Clone all required repositories", Duration: "30m", Owner: "Team"},
                    {Category: "Setup", Task: "Setup VPN and access credentials", Duration: "1h", Owner: "IT"},
                    {Category: "Reading", Task: "Read system architecture doc", Duration: "2h", Owner: "Self"},
                    {Category: "Reading", Task: "Read README and CONTRIBUTING.md", Duration: "1h", Owner: "Self"},
                    {Category: "Meeting", Task: "1:1 with Tech Lead", Duration: "1h", Owner: "TL"},
                    {Category: "Meeting", Task: "Team standup introduction", Duration: "15m", Owner: "Team"},
                    {Category: "Meeting", Task: "Meet with Product Manager", Duration: "30m", Owner: "PM"},
                },
            },
            {
                Week:  2,
                Theme: "Codebase Deep Dive",
                Tasks: []OnboardingTask{
                    {Category: "Code", Task: "Walk through core services", Duration: "4h", Owner: "TL"},
                    {Category: "Code", Task: "Review recent PRs", Duration: "2h", Owner: "Self"},
                    {Category: "Code", Task: "Fix first 'good first issue'", Duration: "1d", Owner: "Self"},
                    {Category: "Code", Task: "Submit first PR", Duration: "1d", Owner: "Self"},
                    {Category: "Meeting", Task: "Architecture walkthrough", Duration: "2h", Owner: "TL"},
                    {Category: "Docs", Task: "Update setup docs with findings", Duration: "1h", Owner: "Self"},
                },
            },
            {
                Week:  3,
                Theme: "First Contribution",
                Tasks: []OnboardingTask{
                    {Category: "Code", Task: "Take on small feature", Duration: "3d", Owner: "Self"},
                    {Category: "Process", Task: "Lead first sprint planning item", Duration: "1h", Owner: "Self"},
                    {Category: "Code", Task: "Write tests for existing code", Duration: "1d", Owner: "Self"},
                    {Category: "Meeting", Task: "Feedback session with TL", Duration: "1h", Owner: "TL"},
                },
            },
            {
                Week:  4,
                Theme: "Independent Contribution",
                Tasks: []OnboardingTask{
                    {Category: "Code", Task: "Own a complete user story", Duration: "1w", Owner: "Self"},
                    {Category: "Meeting", Task: "Present work to team", Duration: "30m", Owner: "Self"},
                    {Category: "Process", Task: "Shadow on-call", Duration: "1w", Owner: "Self+TL"},
                    {Category: "Review", Task: "30-day retrospective", Duration: "1h", Owner: "TL"},
                },
            },
        },
    }
}

func (p *OnboardingPlan) Print() {
    fmt.Printf("=== Onboarding Plan: %s ===\n", p.Developer)
    fmt.Printf("Start Date: %s\n\n", p.StartDate.Format("2006-01-02"))
    
    for _, week := range p.Weeks {
        fmt.Printf("Week %d: %s\n", week.Week, week.Theme)
        fmt.Println(strings.Repeat("-", 50))
        
        for _, task := range week.Tasks {
            status := "[ ]"
            if task.Done {
                status = "[x]"
            }
            fmt.Printf("  %s [%-8s] %-40s %s (Owner: %s)\n",
                status,
                task.Category,
                task.Task[:min(len(task.Task), 39)],
                task.Duration,
                task.Owner)
        }
        fmt.Println()
    }
}

func main() {
    plan := NewOnboardingPlan("สมหญิง ใจดี")
    plan.Print()
    
    fmt.Println("=== Code Review Guidelines ===\n")
    
    guidelines := []struct {
        aspect string
        rules  []string
    }{
        {
            aspect: "When to review",
            rules: []string{
                "Review within 24 hours of PR creation",
                "Prioritize PRs blocking others",
                "Review your own PR before requesting review",
            },
        },
        {
            aspect: "What to check",
            rules: []string{
                "Correctness: Does it do what it says?",
                "Tests: Are edge cases covered?",
                "Performance: Any obvious bottlenecks?",
                "Security: Input validation, auth checks?",
                "Style: Follows team conventions?",
                "Documentation: Is complex logic explained?",
            },
        },
        {
            aspect: "How to give feedback",
            rules: []string{
                "Be specific and actionable",
                "Explain WHY, not just WHAT",
                "Use 'nit:' for minor suggestions",
                "Acknowledge good work",
                "Ask questions, don't assume",
                "Suggest, don't demand (unless blocker)",
            },
        },
    }
    
    for _, g := range guidelines {
        fmt.Printf("%s:\n", g.aspect)
        for _, r := range g.rules {
            fmt.Printf("  - %s\n", r)
        }
        fmt.Println()
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

## สรุป

บทนี้ครอบคลุม Tech Lead และ Team Leadership:

1. **Tech Lead Role** - responsibilities, time allocation, anti-patterns
2. **Go Style Guide** - naming, error handling, package organization
3. **Code Review Culture** - กระบวนการและ etiquette
4. **Onboarding** - แผน 30 วันสำหรับนักพัฒนาใหม่

### Key Takeaways

- Tech Lead = Technical + Leadership, ไม่ใช่แค่ code
- Code review เป็น culture ไม่ใช่แค่ process
- Onboarding ที่ดีช่วยลด time-to-productivity
- Style guide ควร enforce ด้วย tools (golangci-lint)
- สร้าง psychological safety ในทีม เพื่อให้คนกล้าถามและทดลอง
