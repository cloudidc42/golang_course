# Part 93: Contributing to Go Open Source

## เป้าหมายของบทเรียน
- หา Go open source projects ที่เหมาะสม
- ขั้นตอนการ contribute
- Code review etiquette
- การเขียน good pull requests
- Building reputation ในชุมชน

---

## 1. หา Projects ที่เหมาะสม

```go
// project_finder.go - แนวทางหา projects
package main

import (
    "fmt"
)

/*
วิธีหา Go open source projects:

1. GitHub Topics:
   - https://github.com/topics/golang
   - https://github.com/topics/go
   
2. Awesome Go:
   - https://github.com/avelino/awesome-go
   
3. CNCF Landscape:
   - https://landscape.cncf.io/
   - Kubernetes, Prometheus, Jaeger ล้วนเป็น Go

4. Go Standard Library:
   - https://github.com/golang/go
   - Issues labeled "good first issue"
   
5. Personal interests:
   - Projects ที่คุณใช้อยู่แล้ว
   - Problems ที่คุณพบในงาน

Good First Issue Labels:
   - "good first issue"
   - "help wanted"
   - "beginner"
   - "easy"
*/

type Project struct {
    Name        string
    URL         string
    Category    string
    Difficulty  string
    Stars       string
    Description string
}

func main() {
    fmt.Println("=== Go Open Source Projects to Contribute ===\n")
    
    projects := []Project{
        {
            Name:        "Go (standard library)",
            URL:         "https://github.com/golang/go",
            Category:    "Language/Runtime",
            Difficulty:  "Hard",
            Stars:       "120k+",
            Description: "The Go programming language itself",
        },
        {
            Name:        "Kubernetes",
            URL:         "https://github.com/kubernetes/kubernetes",
            Category:    "Cloud Native",
            Difficulty:  "Hard",
            Stars:       "100k+",
            Description: "Container orchestration",
        },
        {
            Name:        "Prometheus",
            URL:         "https://github.com/prometheus/prometheus",
            Category:    "Monitoring",
            Difficulty:  "Medium",
            Stars:       "52k+",
            Description: "Monitoring and alerting toolkit",
        },
        {
            Name:        "Gin",
            URL:         "https://github.com/gin-gonic/gin",
            Category:    "Web Framework",
            Difficulty:  "Easy-Medium",
            Stars:       "76k+",
            Description: "HTTP web framework",
        },
        {
            Name:        "GORM",
            URL:         "https://github.com/go-gorm/gorm",
            Category:    "Database",
            Difficulty:  "Medium",
            Stars:       "35k+",
            Description: "ORM library for Go",
        },
        {
            Name:        "Cobra",
            URL:         "https://github.com/spf13/cobra",
            Category:    "CLI",
            Difficulty:  "Easy",
            Stars:       "36k+",
            Description: "CLI library for Go",
        },
        {
            Name:        "Viper",
            URL:         "https://github.com/spf13/viper",
            Category:    "Configuration",
            Difficulty:  "Easy",
            Stars:       "25k+",
            Description: "Configuration library",
        },
        {
            Name:        "golangci-lint",
            URL:         "https://github.com/golangci/golangci-lint",
            Category:    "Tools",
            Difficulty:  "Medium",
            Stars:       "15k+",
            Description: "Go linter aggregator",
        },
    }
    
    fmt.Printf("%-20s %-12s %-10s %s\n", "Project", "Category", "Difficulty", "Description")
    fmt.Println(string(make([]byte, 80)))
    
    for _, p := range projects {
        fmt.Printf("%-20s %-12s %-10s %s\n",
            p.Name[:min(len(p.Name), 19)],
            p.Category[:min(len(p.Category), 11)],
            p.Difficulty,
            p.Description)
    }
    
    fmt.Println("\n=== Finding Good First Issues ===")
    fmt.Println("\nSearch GitHub for:")
    fmt.Println("  is:open is:issue label:\"good first issue\" language:go")
    fmt.Println("  is:open is:issue label:\"help wanted\" language:go")
    fmt.Println("\nFilter by:")
    fmt.Println("  - Not assigned to anyone")
    fmt.Println("  - Recent activity (< 6 months)")
    fmt.Println("  - Has clear description")
    fmt.Println("  - Has test requirements")
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

---

## 2. ขั้นตอนการ Contribute

```go
// contribution_workflow.go - ขั้นตอน contribute
package main

import "fmt"

/*
Contribution Workflow:

1. Fork & Clone
   git clone https://github.com/YOUR_USERNAME/project.git
   cd project
   git remote add upstream https://github.com/original/project.git

2. Create Branch
   git checkout -b fix/issue-123-describe-the-fix
   git checkout -b feat/add-new-feature
   git checkout -b docs/update-readme

3. Develop
   - อ่าน CONTRIBUTING.md ก่อน
   - Follow existing code style
   - เขียน tests
   - Run existing tests: go test ./...
   - Run linter: golangci-lint run

4. Commit
   git add -p  # Add changes selectively
   git commit -m "fix: resolve nil pointer dereference in parser

   When the input is empty, the parser would panic with nil
   pointer dereference. This fix adds a nil check before
   accessing the token.
   
   Fixes #123"

5. Push & PR
   git push origin fix/issue-123
   # Create PR on GitHub

6. Address Review Comments
   git commit -m "address review: use errors.Is instead of == comparison"
   git push origin fix/issue-123

7. Squash if requested
   git rebase -i upstream/main
   git push -f origin fix/issue-123
*/

type ContributionStep struct {
    Number      int
    Title       string
    Commands    []string
    Tips        []string
}

func main() {
    fmt.Println("=== Contribution Workflow ===\n")
    
    steps := []ContributionStep{
        {
            Number: 1,
            Title:  "Setup",
            Commands: []string{
                "# Fork บน GitHub",
                "git clone https://github.com/YOU/project.git",
                "cd project",
                "git remote add upstream https://github.com/ORIGINAL/project.git",
            },
            Tips: []string{
                "อ่าน CONTRIBUTING.md, README.md, และ CODE_OF_CONDUCT.md",
                "ตรวจสอบว่า issue ยังเปิดอยู่และไม่มีคนทำ",
                "Comment ใน issue ว่าจะ work on มัน",
            },
        },
        {
            Number: 2,
            Title:  "Develop",
            Commands: []string{
                "git checkout -b fix/issue-123-short-description",
                "# เขียนโค้ดและ tests",
                "go test ./...",
                "go vet ./...",
                "golangci-lint run",
            },
            Tips: []string{
                "Keep changes focused - one thing per PR",
                "Match existing code style",
                "Add tests for new behavior",
                "Update documentation if needed",
            },
        },
        {
            Number: 3,
            Title:  "Commit",
            Commands: []string{
                "git add -p  # Review changes before staging",
                `git commit -m "fix: clear description of what changed"`,
            },
            Tips: []string{
                "Use conventional commits: fix:, feat:, docs:, test:, refactor:",
                "ใส่ issue number: 'Fixes #123'",
                "Explain WHY not just WHAT",
                "Keep commits atomic",
            },
        },
        {
            Number: 4,
            Title:  "Pull Request",
            Commands: []string{
                "git fetch upstream",
                "git rebase upstream/main",
                "git push origin fix/issue-123",
                "# Open PR on GitHub",
            },
            Tips: []string{
                "Fill PR template completely",
                "Link to related issues",
                "Add screenshots/examples if relevant",
                "Mark as draft if WIP",
            },
        },
        {
            Number: 5,
            Title:  "Review Process",
            Commands: []string{
                "# Address review comments",
                "git add -A",
                "git commit -m 'address review: use errors.Is'",
                "git push origin fix/issue-123",
            },
            Tips: []string{
                "ตอบ review comments อย่างสุภาพ",
                "อธิบายเหตุผลถ้าไม่เห็นด้วย",
                "Request re-review หลังแก้ไข",
                "Be patient - maintainers busy",
            },
        },
    }
    
    for _, step := range steps {
        fmt.Printf("Step %d: %s\n", step.Number, step.Title)
        
        fmt.Println("  Commands:")
        for _, cmd := range step.Commands {
            fmt.Printf("    %s\n", cmd)
        }
        
        fmt.Println("  Tips:")
        for _, tip := range step.Tips {
            fmt.Printf("    - %s\n", tip)
        }
        fmt.Println()
    }
}
```

---

## 3. การเขียน Good PR Description

```go
// pr_template.go - PR description template
package main

import "fmt"

func main() {
    fmt.Println("=== Pull Request Template ===\n")
    
    template := `## Summary
Brief description of what this PR does and why.

## Motivation
- Fixes #123: Describe the bug/issue
- OR Implements: Feature request description

## Changes Made
- List of specific changes
- Another change
- Updated tests

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing done (describe steps)
- [ ] Edge cases covered

## Breaking Changes
None / List breaking changes if any

## Screenshots (if applicable)
Before: ...
After: ...

## Notes for Reviewer
- Pay special attention to X
- Alternative approach considered: Y (chose this because Z)
`
    
    fmt.Println(template)
    
    fmt.Println("=== Commit Message Format ===\n")
    
    commitExamples := []struct {
        good    string
        bad     string
        reason  string
    }{
        {
            good:   "fix: handle nil pointer in JSON parser when input empty",
            bad:    "fix bug",
            reason: "Specific and descriptive",
        },
        {
            good:   "feat: add retry with exponential backoff to HTTP client",
            bad:    "add retry",
            reason: "Clear what was added and how",
        },
        {
            good:   "docs: update README with Docker deployment instructions",
            bad:    "update docs",
            reason: "Specific about what documentation changed",
        },
        {
            good:   "test: add table-driven tests for edge cases in tokenizer",
            bad:    "add tests",
            reason: "Describes what was tested",
        },
        {
            good:   "refactor: extract common validation logic to shared package",
            bad:    "refactor code",
            reason: "Explains what was refactored and why",
        },
    }
    
    fmt.Printf("%-55s %-25s %s\n", "GOOD", "BAD", "REASON")
    fmt.Println(string(make([]byte, 100)))
    
    for _, ex := range commitExamples {
        fmt.Printf("%-55s %-25s %s\n",
            ex.good[:min(len(ex.good), 54)],
            ex.bad,
            ex.reason)
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

## 4. Code Review Etiquette

```go
// code_review_etiquette.go - แนวทาง code review
package main

import "fmt"

func main() {
    fmt.Println("=== Code Review Etiquette ===\n")
    
    // As Author
    fmt.Println("As PR Author:")
    authorTips := []string{
        "ตรวจสอบ PR ตัวเองก่อนส่ง",
        "เขียน description ที่ชัดเจน",
        "Keep PR small และ focused",
        "ตอบ review comments ทุกข้อ",
        "อธิบายเหตุผลถ้าไม่เห็นด้วย (อย่างสุภาพ)",
        "Mark conversations resolved หลังแก้ไข",
        "Request re-review เมื่อแก้ครบ",
    }
    for _, tip := range authorTips {
        fmt.Printf("  - %s\n", tip)
    }
    
    // As Reviewer
    fmt.Println("\nAs Reviewer:")
    reviewerTips := []struct {
        do     string
        avoid  string
    }{
        {"Be specific: 'This could panic if x is nil'", "Vague: 'This is wrong'"},
        {"Ask questions: 'Why did you choose X over Y?'", "Assume bad intent"},
        {"Suggest improvements: 'Consider using errors.As'", "Just reject without alternative"},
        {"Acknowledge good work: 'Nice use of context here'", "Only point out negatives"},
        {"Prioritize comments: 'nit:' vs must-fix", "Treat all comments as equal"},
    }
    
    fmt.Printf("  %-45s %s\n", "DO", "AVOID")
    fmt.Println("  " + string(make([]byte, 75)))
    for _, tip := range reviewerTips {
        fmt.Printf("  %-45s %s\n", tip.do, tip.avoid)
    }
    
    // Common Review Prefixes
    fmt.Println("\nComment Prefixes (Conventional):")
    prefixes := []struct {
        prefix string
        meaning string
    }{
        {"nit:", "Minor style issue, optional to fix"},
        {"suggestion:", "Idea to consider, not required"},
        {"question:", "Asking for clarification"},
        {"concern:", "Something that should be addressed"},
        {"blocker:", "Must fix before merge"},
        {"praise:", "Positive feedback"},
    }
    
    for _, p := range prefixes {
        fmt.Printf("  %-15s %s\n", p.prefix, p.meaning)
    }
}
```

---

## 5. Building Community Presence

```go
// community_building.go - สร้าง presence ในชุมชน
package main

import "fmt"

func main() {
    fmt.Println("=== Building Go Community Presence ===\n")
    
    activities := []struct {
        activity string
        time     string
        impact   string
        example  string
    }{
        {
            activity: "Answer Stack Overflow questions",
            time:     "30 min/day",
            impact:   "High",
            example:  "tag: [go] on StackOverflow",
        },
        {
            activity: "Write blog posts",
            time:     "2-3 hrs/post",
            impact:   "High",
            example:  "dev.to, Medium, personal blog",
        },
        {
            activity: "Give conference talks",
            time:     "20+ hrs preparation",
            impact:   "Very High",
            example:  "GopherCon, local Go meetups",
        },
        {
            activity: "Contribute to documentation",
            time:     "1-2 hrs",
            impact:   "Medium",
            example:  "Fix typos, add examples",
        },
        {
            activity: "Review PRs (not your own)",
            time:     "1-2 hrs/week",
            impact:   "Medium-High",
            example:  "Join as a reviewer",
        },
        {
            activity: "Create tools/libraries",
            time:     "Ongoing",
            impact:   "Very High",
            example:  "Publish to GitHub",
        },
        {
            activity: "Attend Go meetups",
            time:     "Monthly",
            impact:   "Medium",
            example:  "golang.org/wiki/GoUserGroups",
        },
    }
    
    fmt.Printf("%-35s %-15s %-12s %s\n", "Activity", "Time Invest", "Impact", "Where")
    fmt.Println(string(make([]byte, 85)))
    
    for _, a := range activities {
        fmt.Printf("%-35s %-15s %-12s %s\n",
            a.activity[:min(len(a.activity), 34)],
            a.time,
            a.impact,
            a.example)
    }
    
    fmt.Println("\n=== Go Community Resources ===\n")
    resources := []string{
        "https://forum.golangbridge.org/       - Go community forum",
        "https://discord.gg/golang             - Go Discord",
        "https://gophers.slack.com/            - Gophers Slack",
        "https://twitter.com/golang            - Go Twitter",
        "https://reddit.com/r/golang/          - Go Reddit",
        "https://go.dev/blog/                  - Official Go Blog",
        "https://gophercon.com/                - GopherCon Conference",
    }
    
    for _, r := range resources {
        fmt.Printf("  %s\n", r)
    }
    
    fmt.Println("\n=== Your First Contribution Checklist ===\n")
    checklist := []string{
        "[ ] อ่าน CONTRIBUTING.md ของ project",
        "[ ] Setup development environment",
        "[ ] รัน existing tests ให้ผ่านก่อน",
        "[ ] เลือก 'good first issue'",
        "[ ] Comment ใน issue ว่าจะทำ",
        "[ ] สร้าง fork และ branch",
        "[ ] เขียนโค้ด + tests",
        "[ ] รัน tests และ linter",
        "[ ] เปิด PR พร้อม description ที่ดี",
        "[ ] ตอบ review comments อย่างรวดเร็ว",
        "[ ] ฉลองเมื่อ PR ถูก merge!",
    }
    
    for _, item := range checklist {
        fmt.Printf("  %s\n", item)
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

บทนี้ครอบคลุมการ contribute ใน Go open source:

1. **การหา projects** - GitHub topics, Awesome Go, CNCF
2. **Contribution workflow** - fork, develop, PR, review
3. **PR best practices** - description, commit messages
4. **Code review etiquette** - เป็นทั้ง author และ reviewer ที่ดี
5. **Community building** - สร้าง presence ระยะยาว

### Key Takeaways

- เริ่มจาก "good first issue" และ documentation fixes
- PR เล็กและ focused merge ได้เร็วกว่า
- Communication ดีเท่าๆ กับ code quality
- Consistency สำคัญกว่า quantity
- ชุมชน Go เป็นมิตรและยินดีต้อนรับ newcomers
