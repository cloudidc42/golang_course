# Part 99: Complete Fintech System

## เป้าหมายของบทเรียน
- Core banking system
- Account management
- Transaction processing
- KYC/AML compliance
- Fraud detection
- Real-time notifications

---

## 1. Core Banking Architecture

```go
// fintech/core/types.go
package main

import (
    "fmt"
    "sync"
    "time"
    "math"
    "strings"
)

// AccountType ประเภทบัญชี
type AccountType string

const (
    Savings  AccountType = "savings"
    Checking AccountType = "checking"
    Business AccountType = "business"
)

// TransactionType ประเภทธุรกรรม
type TransactionType string

const (
    Deposit    TransactionType = "deposit"
    Withdrawal TransactionType = "withdrawal"
    Transfer   TransactionType = "transfer"
    Payment    TransactionType = "payment"
)

// TransactionStatus สถานะธุรกรรม
type TransactionStatus string

const (
    TxPending   TransactionStatus = "pending"
    TxCompleted TransactionStatus = "completed"
    TxFailed    TransactionStatus = "failed"
    TxReversed  TransactionStatus = "reversed"
)

// Currency สกุลเงิน
type Currency string

const (
    THB Currency = "THB"
    USD Currency = "USD"
    EUR Currency = "EUR"
)

// Money จำนวนเงิน (ใช้ int เพื่อหลีกเลี่ยง floating point)
// เก็บเป็น satang (0.01 THB)
type Money struct {
    Amount   int64    // in smallest unit (satang)
    Currency Currency
}

func NewMoney(baht float64, currency Currency) Money {
    return Money{
        Amount:   int64(math.Round(baht * 100)),
        Currency: currency,
    }
}

func (m Money) Baht() float64 {
    return float64(m.Amount) / 100
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, fmt.Errorf("currency mismatch: %s vs %s", m.Currency, other.Currency)
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

func (m Money) Sub(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, fmt.Errorf("currency mismatch: %s vs %s", m.Currency, other.Currency)
    }
    return Money{Amount: m.Amount - other.Amount, Currency: m.Currency}, nil
}

func (m Money) IsNegative() bool {
    return m.Amount < 0
}

func (m Money) String() string {
    return fmt.Sprintf("%.2f %s", m.Baht(), m.Currency)
}

// Account บัญชีธนาคาร
type Account struct {
    ID          string
    CustomerID  string
    Type        AccountType
    Balance     Money
    IBAN        string
    Status      string // active, frozen, closed
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// Transaction ธุรกรรม
type Transaction struct {
    ID          string
    FromAccount string
    ToAccount   string
    Type        TransactionType
    Amount      Money
    Fee         Money
    Status      TransactionStatus
    Description string
    Reference   string
    Metadata    map[string]string
    CreatedAt   time.Time
    CompletedAt *time.Time
}

// Ledger Entry สำหรับ double-entry bookkeeping
type LedgerEntry struct {
    ID            string
    TransactionID string
    AccountID     string
    Debit         Money
    Credit        Money
    Balance       Money
    Description   string
    CreatedAt     time.Time
}
```

---

## 2. Account Service

```go
// account_service.go
// (continued in same package for demo)

// AccountService จัดการบัญชี
type AccountService struct {
    mu       sync.RWMutex
    accounts map[string]*Account
}

func NewAccountService() *AccountService {
    return &AccountService{
        accounts: make(map[string]*Account),
    }
}

func (s *AccountService) CreateAccount(customerID string, accountType AccountType) (*Account, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    account := &Account{
        ID:         generateAccountID(),
        CustomerID: customerID,
        Type:       accountType,
        Balance:    NewMoney(0, THB),
        IBAN:       generateIBAN(),
        Status:     "active",
        CreatedAt:  time.Now(),
        UpdatedAt:  time.Now(),
    }
    
    s.accounts[account.ID] = account
    return account, nil
}

func (s *AccountService) GetAccount(id string) (*Account, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    acc, ok := s.accounts[id]
    if !ok {
        return nil, fmt.Errorf("account not found: %s", id)
    }
    return acc, nil
}

func (s *AccountService) GetBalance(id string) (Money, error) {
    acc, err := s.GetAccount(id)
    if err != nil {
        return Money{}, err
    }
    return acc.Balance, nil
}

func (s *AccountService) UpdateBalance(id string, amount Money) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    acc, ok := s.accounts[id]
    if !ok {
        return fmt.Errorf("account not found: %s", id)
    }
    
    if acc.Status != "active" {
        return fmt.Errorf("account %s is %s", id, acc.Status)
    }
    
    newBalance, err := acc.Balance.Add(amount)
    if err != nil {
        return err
    }
    
    if newBalance.IsNegative() && acc.Type != Checking {
        return fmt.Errorf("insufficient funds in account %s", id)
    }
    
    acc.Balance = newBalance
    acc.UpdatedAt = time.Now()
    return nil
}

func generateAccountID() string {
    return fmt.Sprintf("ACC%d", time.Now().UnixNano())
}

func generateIBAN() string {
    return fmt.Sprintf("TH%d", time.Now().UnixNano()%10000000000)
}
```

---

## 3. Transaction Processing

```go
// transaction_service.go

// TransactionService ประมวลผลธุรกรรม
type TransactionService struct {
    mu           sync.Mutex
    transactions map[string]*Transaction
    ledger       []*LedgerEntry
    accountSvc   *AccountService
}

func NewTransactionService(accountSvc *AccountService) *TransactionService {
    return &TransactionService{
        transactions: make(map[string]*Transaction),
        accountSvc:   accountSvc,
    }
}

// Deposit ฝากเงิน
func (s *TransactionService) Deposit(accountID string, amount Money, description string) (*Transaction, error) {
    if amount.Amount <= 0 {
        return nil, fmt.Errorf("deposit amount must be positive")
    }
    
    tx := &Transaction{
        ID:          generateTxID(),
        ToAccount:   accountID,
        Type:        Deposit,
        Amount:      amount,
        Fee:         NewMoney(0, amount.Currency),
        Status:      TxPending,
        Description: description,
        Reference:   generateReference(),
        CreatedAt:   time.Now(),
    }
    
    // Update account balance
    if err := s.accountSvc.UpdateBalance(accountID, amount); err != nil {
        tx.Status = TxFailed
        s.saveTransaction(tx)
        return tx, err
    }
    
    now := time.Now()
    tx.Status = TxCompleted
    tx.CompletedAt = &now
    
    s.saveTransaction(tx)
    s.addLedgerEntry(tx, accountID, NewMoney(0, amount.Currency), amount)
    
    return tx, nil
}

// Withdraw ถอนเงิน
func (s *TransactionService) Withdraw(accountID string, amount Money, description string) (*Transaction, error) {
    if amount.Amount <= 0 {
        return nil, fmt.Errorf("withdrawal amount must be positive")
    }
    
    // Check balance
    balance, err := s.accountSvc.GetBalance(accountID)
    if err != nil {
        return nil, err
    }
    
    if balance.Amount < amount.Amount {
        return nil, fmt.Errorf("insufficient funds: have %.2f, need %.2f",
            balance.Baht(), amount.Baht())
    }
    
    // Calculate fee (0.1% for withdrawal)
    feeAmount := int64(float64(amount.Amount) * 0.001)
    fee := Money{Amount: feeAmount, Currency: amount.Currency}
    
    tx := &Transaction{
        ID:          generateTxID(),
        FromAccount: accountID,
        Type:        Withdrawal,
        Amount:      amount,
        Fee:         fee,
        Status:      TxPending,
        Description: description,
        Reference:   generateReference(),
        CreatedAt:   time.Now(),
    }
    
    // Deduct amount + fee
    totalDeduction := Money{Amount: -(amount.Amount + fee.Amount), Currency: amount.Currency}
    
    if err := s.accountSvc.UpdateBalance(accountID, totalDeduction); err != nil {
        tx.Status = TxFailed
        s.saveTransaction(tx)
        return tx, err
    }
    
    now := time.Now()
    tx.Status = TxCompleted
    tx.CompletedAt = &now
    
    s.saveTransaction(tx)
    return tx, nil
}

// Transfer โอนเงิน
func (s *TransactionService) Transfer(fromID, toID string, amount Money, description string) (*Transaction, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if amount.Amount <= 0 {
        return nil, fmt.Errorf("transfer amount must be positive")
    }
    
    if fromID == toID {
        return nil, fmt.Errorf("cannot transfer to same account")
    }
    
    // Check source balance
    fromBalance, err := s.accountSvc.GetBalance(fromID)
    if err != nil {
        return nil, fmt.Errorf("source account: %w", err)
    }
    
    // Calculate fee
    feeAmount := int64(float64(amount.Amount) * 0.002) // 0.2% transfer fee
    fee := Money{Amount: feeAmount, Currency: amount.Currency}
    
    if fromBalance.Amount < amount.Amount+fee.Amount {
        return nil, fmt.Errorf("insufficient funds including fee")
    }
    
    tx := &Transaction{
        ID:          generateTxID(),
        FromAccount: fromID,
        ToAccount:   toID,
        Type:        Transfer,
        Amount:      amount,
        Fee:         fee,
        Status:      TxPending,
        Description: description,
        Reference:   generateReference(),
        CreatedAt:   time.Now(),
    }
    
    // Deduct from source
    deduction := Money{Amount: -(amount.Amount + fee.Amount), Currency: amount.Currency}
    if err := s.accountSvc.UpdateBalance(fromID, deduction); err != nil {
        tx.Status = TxFailed
        s.saveTransaction(tx)
        return tx, err
    }
    
    // Credit to destination
    if err := s.accountSvc.UpdateBalance(toID, amount); err != nil {
        // Rollback source
        s.accountSvc.UpdateBalance(fromID, amount)
        tx.Status = TxFailed
        s.saveTransaction(tx)
        return tx, err
    }
    
    now := time.Now()
    tx.Status = TxCompleted
    tx.CompletedAt = &now
    
    s.saveTransaction(tx)
    return tx, nil
}

func (s *TransactionService) saveTransaction(tx *Transaction) {
    s.transactions[tx.ID] = tx
}

func (s *TransactionService) addLedgerEntry(tx *Transaction, accountID string, debit, credit Money) {
    // Get current balance
    balance, _ := s.accountSvc.GetBalance(accountID)
    
    entry := &LedgerEntry{
        ID:            fmt.Sprintf("LED%d", time.Now().UnixNano()),
        TransactionID: tx.ID,
        AccountID:     accountID,
        Debit:         debit,
        Credit:        credit,
        Balance:       balance,
        Description:   tx.Description,
        CreatedAt:     time.Now(),
    }
    s.ledger = append(s.ledger, entry)
}

func generateTxID() string {
    return fmt.Sprintf("TXN%d", time.Now().UnixNano())
}

func generateReference() string {
    return fmt.Sprintf("REF%d", time.Now().UnixNano()%1000000)
}
```

---

## 4. KYC/AML Compliance

```go
// kyc_aml.go - Know Your Customer / Anti-Money Laundering

// KYCStatus สถานะ KYC
type KYCStatus string

const (
    KYCPending   KYCStatus = "pending"
    KYCApproved  KYCStatus = "approved"
    KYCRejected  KYCStatus = "rejected"
    KYCReview    KYCStatus = "under_review"
)

// KYCProfile ข้อมูล KYC
type KYCProfile struct {
    CustomerID    string
    NationalID    string
    FirstName     string
    LastName      string
    DateOfBirth   string
    Address       string
    DocumentType  string
    DocumentURL   string
    Status        KYCStatus
    RiskLevel     string // low, medium, high
    SubmittedAt   time.Time
    ReviewedAt    *time.Time
    ReviewerID    string
    Notes         string
}

// AMLRule กฎ AML
type AMLRule struct {
    ID          string
    Name        string
    Description string
    Threshold   float64
    Action      string // flag, block, report
}

// AMLChecker ตรวจสอบ AML
type AMLChecker struct {
    rules []AMLRule
    flags []AMLFlag
    mu    sync.Mutex
}

// AMLFlag การแจ้งเตือน AML
type AMLFlag struct {
    ID            string
    TransactionID string
    CustomerID    string
    RuleID        string
    RuleName      string
    Reason        string
    Amount        float64
    FlaggedAt     time.Time
    Status        string
}

func NewAMLChecker() *AMLChecker {
    return &AMLChecker{
        rules: []AMLRule{
            {
                ID:          "RULE001",
                Name:        "Large Cash Transaction",
                Description: "Single transaction over 450,000 THB",
                Threshold:   450000,
                Action:      "report",
            },
            {
                ID:          "RULE002",
                Name:        "Structuring Pattern",
                Description: "Multiple transactions just below 450,000 THB threshold",
                Threshold:   400000,
                Action:      "flag",
            },
            {
                ID:          "RULE003",
                Name:        "Rapid Large Transfers",
                Description: "Transfer over 100,000 THB to new account within 24h",
                Threshold:   100000,
                Action:      "flag",
            },
        },
    }
}

// CheckTransaction ตรวจสอบ transaction
func (a *AMLChecker) CheckTransaction(tx *Transaction) []AMLFlag {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    var flags []AMLFlag
    amount := tx.Amount.Baht()
    
    for _, rule := range a.rules {
        if amount >= rule.Threshold {
            flag := AMLFlag{
                ID:            fmt.Sprintf("AML%d", time.Now().UnixNano()),
                TransactionID: tx.ID,
                RuleID:        rule.ID,
                RuleName:      rule.Name,
                Reason:        fmt.Sprintf("%s: %.2f THB exceeds %.2f threshold", rule.Name, amount, rule.Threshold),
                Amount:        amount,
                FlaggedAt:     time.Now(),
                Status:        "open",
            }
            flags = append(flags, flag)
            a.flags = append(a.flags, flag)
        }
    }
    
    return flags
}

func (a *AMLChecker) GetFlags() []AMLFlag {
    a.mu.Lock()
    defer a.mu.Unlock()
    return append([]AMLFlag{}, a.flags...)
}
```

---

## 5. Fraud Detection

```go
// fraud_detection.go

// FraudScore คะแนนความเสี่ยง fraud
type FraudScore struct {
    Score      float64   // 0-100
    Risk       string    // low, medium, high, critical
    Factors    []string
    Timestamp  time.Time
}

// FraudDetector ตรวจจับ fraud
type FraudDetector struct {
    txHistory  map[string][]*Transaction  // customerID -> transactions
    mu         sync.RWMutex
}

func NewFraudDetector() *FraudDetector {
    return &FraudDetector{
        txHistory: make(map[string][]*Transaction),
    }
}

// RecordTransaction บันทึก transaction สำหรับ pattern analysis
func (f *FraudDetector) RecordTransaction(customerID string, tx *Transaction) {
    f.mu.Lock()
    defer f.mu.Unlock()
    
    f.txHistory[customerID] = append(f.txHistory[customerID], tx)
    
    // Keep only last 100 transactions per customer
    if len(f.txHistory[customerID]) > 100 {
        f.txHistory[customerID] = f.txHistory[customerID][1:]
    }
}

// AnalyzeTransaction วิเคราะห์ความเสี่ยง
func (f *FraudDetector) AnalyzeTransaction(customerID string, tx *Transaction) FraudScore {
    f.mu.RLock()
    defer f.mu.RUnlock()
    
    score := 0.0
    var factors []string
    
    amount := tx.Amount.Baht()
    history := f.txHistory[customerID]
    
    // Factor 1: Large transaction
    if amount > 50000 {
        score += 20
        factors = append(factors, fmt.Sprintf("Large amount: %.2f THB", amount))
    }
    
    // Factor 2: Unusual time (2am - 5am)
    hour := tx.CreatedAt.Hour()
    if hour >= 2 && hour <= 5 {
        score += 15
        factors = append(factors, fmt.Sprintf("Unusual time: %02d:xx", hour))
    }
    
    // Factor 3: Velocity check (many transactions in short time)
    recentCount := 0
    cutoff := time.Now().Add(-1 * time.Hour)
    for _, htx := range history {
        if htx.CreatedAt.After(cutoff) {
            recentCount++
        }
    }
    if recentCount > 10 {
        score += 25
        factors = append(factors, fmt.Sprintf("High velocity: %d txns in 1h", recentCount))
    }
    
    // Factor 4: New recipient (first time transfer to this account)
    if tx.Type == Transfer {
        isFirstTransfer := true
        for _, htx := range history {
            if htx.ToAccount == tx.ToAccount {
                isFirstTransfer = false
                break
            }
        }
        if isFirstTransfer {
            score += 10
            factors = append(factors, "First transfer to this recipient")
        }
    }
    
    // Determine risk level
    risk := "low"
    switch {
    case score >= 75:
        risk = "critical"
    case score >= 50:
        risk = "high"
    case score >= 25:
        risk = "medium"
    }
    
    return FraudScore{
        Score:     score,
        Risk:      risk,
        Factors:   factors,
        Timestamp: time.Now(),
    }
}
```

---

## 6. Main Demo

```go
func main() {
    fmt.Println("=== Fintech System Demo ===\n")
    
    // Initialize services
    accountSvc := NewAccountService()
    txSvc := NewTransactionService(accountSvc)
    aml := NewAMLChecker()
    fraud := NewFraudDetector()
    
    // Create accounts
    fmt.Println("1. Creating accounts...")
    acc1, _ := accountSvc.CreateAccount("CUST001", Savings)
    acc2, _ := accountSvc.CreateAccount("CUST002", Savings)
    acc3, _ := accountSvc.CreateAccount("CUST001", Checking)
    
    fmt.Printf("   Account 1 (Savings): %s\n", acc1.ID)
    fmt.Printf("   Account 2 (Savings): %s\n", acc2.ID)
    fmt.Printf("   Account 3 (Checking): %s\n", acc3.ID)
    
    // Deposit
    fmt.Println("\n2. Making deposits...")
    tx1, err := txSvc.Deposit(acc1.ID, NewMoney(100000, THB), "Initial deposit")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("   Deposited: %s (ref: %s)\n", tx1.Amount, tx1.Reference)
    
    tx2, _ := txSvc.Deposit(acc2.ID, NewMoney(50000, THB), "Initial deposit")
    fmt.Printf("   Deposited: %s (ref: %s)\n", tx2.Amount, tx2.Reference)
    
    // Check balances
    fmt.Println("\n3. Checking balances...")
    bal1, _ := accountSvc.GetBalance(acc1.ID)
    bal2, _ := accountSvc.GetBalance(acc2.ID)
    fmt.Printf("   Account 1: %s\n", bal1)
    fmt.Printf("   Account 2: %s\n", bal2)
    
    // Transfer
    fmt.Println("\n4. Transfer between accounts...")
    txTransfer, err := txSvc.Transfer(acc1.ID, acc2.ID, NewMoney(25000, THB), "Payment for services")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("   Transfer: %s (fee: %s)\n", txTransfer.Amount, txTransfer.Fee)
    fmt.Printf("   Status: %s\n", txTransfer.Status)
    
    // Updated balances
    fmt.Println("\n5. Updated balances...")
    bal1After, _ := accountSvc.GetBalance(acc1.ID)
    bal2After, _ := accountSvc.GetBalance(acc2.ID)
    fmt.Printf("   Account 1: %s\n", bal1After)
    fmt.Printf("   Account 2: %s\n", bal2After)
    
    // AML Check
    fmt.Println("\n6. AML check on large transaction...")
    largeTx, _ := txSvc.Deposit(acc3.ID, NewMoney(500000, THB), "Business income")
    flags := aml.CheckTransaction(largeTx)
    
    if len(flags) > 0 {
        fmt.Printf("   AML FLAGS RAISED: %d\n", len(flags))
        for _, flag := range flags {
            fmt.Printf("   - [%s] %s\n", flag.RuleName, flag.Reason)
        }
    }
    
    // Fraud detection
    fmt.Println("\n7. Fraud analysis...")
    fraud.RecordTransaction("CUST001", txTransfer)
    
    fraudScore := fraud.AnalyzeTransaction("CUST001", txTransfer)
    fmt.Printf("   Fraud score: %.1f/100 (risk: %s)\n", fraudScore.Score, fraudScore.Risk)
    if len(fraudScore.Factors) > 0 {
        fmt.Printf("   Risk factors:\n")
        for _, f := range fraudScore.Factors {
            fmt.Printf("     - %s\n", f)
        }
    }
    
    // Summary
    fmt.Println("\n=== Transaction Summary ===")
    allTxns := []*Transaction{tx1, tx2, txTransfer, largeTx}
    for _, tx := range allTxns {
        symbol := ""
        if tx.Type == Deposit {
            symbol = "+"
        } else if tx.Type == Withdrawal || tx.Type == Transfer {
            symbol = "-"
        }
        fmt.Printf("  [%s] %s%s (%s)\n", tx.Type, symbol, tx.Amount, tx.Status)
    }
    
    fmt.Println("\n=== Production Checklist ===")
    checklist := []string{
        "[ ] HSM สำหรับ encryption keys",
        "[ ] PCI DSS compliance",
        "[ ] Bank of Thailand regulations",
        "[ ] Real-time fraud ML model",
        "[ ] 99.99% uptime SLA",
        "[ ] Disaster recovery < 15 minutes RTO",
        "[ ] End-to-end encryption",
        "[ ] Audit logging (tamper-proof)",
    }
    for _, item := range checklist {
        fmt.Printf("  %s\n", item)
    }
    
    _ = strings.ToUpper
}
```

---

## สรุป

บทนี้สร้าง Complete Fintech System:

1. **Core Banking** - accounts, transactions, double-entry ledger
2. **Money type** - integer-based ป้องกัน floating point errors
3. **Transaction Processing** - deposit, withdrawal, transfer with fees
4. **KYC/AML** - compliance ตามกฎหมาย
5. **Fraud Detection** - real-time risk scoring

### Key Takeaways

- ใช้ integer สำหรับ money (หลีกเลี่ยง floating point)
- Double-entry bookkeeping รับประกัน consistency
- ACID transactions สำคัญมากสำหรับ banking
- AML/KYC เป็น legal requirement ไม่ใช่ optional
- Fraud detection ต้องทำงาน real-time
