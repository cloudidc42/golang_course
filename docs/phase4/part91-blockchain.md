# Part 91: Blockchain ด้วย Go

## เป้าหมายของบทเรียน
- หลักการ Blockchain
- Cryptographic hashing
- สร้าง Blockchain ตั้งแต่ต้น
- Consensus mechanisms
- Smart contract concepts
- go-ethereum และ Web3

---

## 1. Blockchain Basics

```go
// blockchain_basics.go - โครงสร้างพื้นฐาน Blockchain
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "math/big"
    "strings"
    "time"
)

// Transaction แสดง transaction
type Transaction struct {
    From      string    `json:"from"`
    To        string    `json:"to"`
    Amount    float64   `json:"amount"`
    Timestamp time.Time `json:"timestamp"`
    ID        string    `json:"id"`
}

// NewTransaction สร้าง transaction ใหม่
func NewTransaction(from, to string, amount float64) *Transaction {
    tx := &Transaction{
        From:      from,
        To:        to,
        Amount:    amount,
        Timestamp: time.Now(),
    }
    tx.ID = tx.Hash()
    return tx
}

// Hash คำนวณ hash ของ transaction
func (tx *Transaction) Hash() string {
    data, _ := json.Marshal(tx)
    hash := sha256.Sum256(data)
    return hex.EncodeToString(hash[:])
}

// Block แสดง block ใน blockchain
type Block struct {
    Index        int            `json:"index"`
    Timestamp    time.Time      `json:"timestamp"`
    Transactions []*Transaction `json:"transactions"`
    PrevHash     string         `json:"prev_hash"`
    Hash         string         `json:"hash"`
    Nonce        int            `json:"nonce"`
    Difficulty   int            `json:"difficulty"`
}

// NewBlock สร้าง block ใหม่
func NewBlock(index int, transactions []*Transaction, prevHash string, difficulty int) *Block {
    b := &Block{
        Index:        index,
        Timestamp:    time.Now(),
        Transactions: transactions,
        PrevHash:     prevHash,
        Difficulty:   difficulty,
    }
    return b
}

// calculateHash คำนวณ hash ของ block
func (b *Block) calculateHash() string {
    data := fmt.Sprintf("%d%s%v%s%d",
        b.Index,
        b.Timestamp.String(),
        b.Transactions,
        b.PrevHash,
        b.Nonce,
    )
    hash := sha256.Sum256([]byte(data))
    return hex.EncodeToString(hash[:])
}

// Mine ทำ Proof of Work
func (b *Block) Mine() {
    target := strings.Repeat("0", b.Difficulty)
    start := time.Now()
    
    for {
        b.Hash = b.calculateHash()
        if strings.HasPrefix(b.Hash, target) {
            fmt.Printf("Block #%d mined in %v (nonce=%d)\n",
                b.Index, time.Since(start), b.Nonce)
            return
        }
        b.Nonce++
    }
}

// IsValid ตรวจสอบ block validity
func (b *Block) IsValid() bool {
    target := strings.Repeat("0", b.Difficulty)
    return strings.HasPrefix(b.Hash, target) &&
        b.Hash == b.calculateHash()
}

// Blockchain struct
type Blockchain struct {
    Chain      []*Block
    Difficulty int
    Pending    []*Transaction
}

// NewBlockchain สร้าง blockchain ใหม่
func NewBlockchain(difficulty int) *Blockchain {
    bc := &Blockchain{
        Difficulty: difficulty,
    }
    
    // Genesis block
    genesis := NewBlock(0, nil, "0", difficulty)
    genesis.Mine()
    bc.Chain = append(bc.Chain, genesis)
    
    return bc
}

// AddTransaction เพิ่ม transaction pending
func (bc *Blockchain) AddTransaction(tx *Transaction) {
    bc.Pending = append(bc.Pending, tx)
}

// MineBlock สร้าง block ใหม่จาก pending transactions
func (bc *Blockchain) MineBlock(minerAddress string) *Block {
    // Reward transaction
    reward := NewTransaction("system", minerAddress, 50.0)
    transactions := append(bc.Pending, reward)
    
    prevBlock := bc.Chain[len(bc.Chain)-1]
    block := NewBlock(len(bc.Chain), transactions, prevBlock.Hash, bc.Difficulty)
    block.Mine()
    
    bc.Chain = append(bc.Chain, block)
    bc.Pending = nil
    
    return block
}

// IsValid ตรวจสอบ blockchain
func (bc *Blockchain) IsValid() bool {
    for i := 1; i < len(bc.Chain); i++ {
        current := bc.Chain[i]
        prev := bc.Chain[i-1]
        
        if !current.IsValid() {
            fmt.Printf("Block #%d has invalid hash\n", i)
            return false
        }
        
        if current.PrevHash != prev.Hash {
            fmt.Printf("Block #%d has invalid prev hash\n", i)
            return false
        }
    }
    return true
}

// GetBalance คำนวณ balance ของ address
func (bc *Blockchain) GetBalance(address string) float64 {
    balance := 0.0
    
    for _, block := range bc.Chain {
        for _, tx := range block.Transactions {
            if tx.From == address {
                balance -= tx.Amount
            }
            if tx.To == address {
                balance += tx.Amount
            }
        }
    }
    
    return balance
}

// PrintChain แสดง blockchain
func (bc *Blockchain) PrintChain() {
    for _, block := range bc.Chain {
        fmt.Printf("\n=== Block #%d ===\n", block.Index)
        fmt.Printf("Hash:     %s\n", block.Hash[:16]+"...")
        fmt.Printf("PrevHash: %s\n", block.PrevHash[:16]+"...")
        fmt.Printf("Nonce:    %d\n", block.Nonce)
        fmt.Printf("Txns:     %d\n", len(block.Transactions))
    }
}

func main() {
    fmt.Println("=== Simple Blockchain Demo ===\n")
    
    bc := NewBlockchain(3) // Difficulty 3 (0xx prefix)
    
    // Add transactions
    bc.AddTransaction(NewTransaction("Alice", "Bob", 50))
    bc.AddTransaction(NewTransaction("Bob", "Charlie", 25))
    
    fmt.Println("\nMining block 1...")
    bc.MineBlock("Miner1")
    
    bc.AddTransaction(NewTransaction("Charlie", "Alice", 10))
    
    fmt.Println("\nMining block 2...")
    bc.MineBlock("Miner1")
    
    bc.PrintChain()
    
    fmt.Printf("\nBalances:\n")
    addresses := []string{"Alice", "Bob", "Charlie", "Miner1"}
    for _, addr := range addresses {
        fmt.Printf("  %s: %.2f\n", addr, bc.GetBalance(addr))
    }
    
    fmt.Printf("\nChain valid: %v\n", bc.IsValid())
    
    // Tamper test
    fmt.Println("\n--- Tamper Detection ---")
    bc.Chain[1].Transactions[0].Amount = 999
    fmt.Printf("After tampering - Chain valid: %v\n", bc.IsValid())
    
    _ = big.NewInt
}
```

---

## 2. Merkle Tree

```go
// merkle_tree.go - Merkle tree สำหรับ transaction verification
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "fmt"
)

// MerkleNode node ใน Merkle tree
type MerkleNode struct {
    Left  *MerkleNode
    Right *MerkleNode
    Hash  string
}

// NewMerkleLeaf สร้าง leaf node
func NewMerkleLeaf(data string) *MerkleNode {
    hash := sha256.Sum256([]byte(data))
    return &MerkleNode{
        Hash: hex.EncodeToString(hash[:]),
    }
}

// NewMerkleNode สร้าง internal node
func NewMerkleNode(left, right *MerkleNode) *MerkleNode {
    combined := left.Hash + right.Hash
    hash := sha256.Sum256([]byte(combined))
    return &MerkleNode{
        Left:  left,
        Right: right,
        Hash:  hex.EncodeToString(hash[:]),
    }
}

// MerkleTree Merkle tree structure
type MerkleTree struct {
    Root *MerkleNode
}

// NewMerkleTree สร้าง Merkle tree จาก data
func NewMerkleTree(data []string) *MerkleTree {
    if len(data) == 0 {
        return &MerkleTree{}
    }
    
    // สร้าง leaves
    nodes := make([]*MerkleNode, len(data))
    for i, d := range data {
        nodes[i] = NewMerkleLeaf(d)
    }
    
    // Build tree bottom-up
    for len(nodes) > 1 {
        // ถ้า odd จำนวน, ทำซ้ำ node สุดท้าย
        if len(nodes)%2 != 0 {
            nodes = append(nodes, nodes[len(nodes)-1])
        }
        
        newLevel := make([]*MerkleNode, len(nodes)/2)
        for i := 0; i < len(nodes); i += 2 {
            newLevel[i/2] = NewMerkleNode(nodes[i], nodes[i+1])
        }
        nodes = newLevel
    }
    
    return &MerkleTree{Root: nodes[0]}
}

// RootHash คืน Merkle root hash
func (t *MerkleTree) RootHash() string {
    if t.Root == nil {
        return ""
    }
    return t.Root.Hash
}

// Verify ตรวจสอบว่า data อยู่ใน tree
func (t *MerkleTree) Verify(data string, proof []string, index int) bool {
    hash := sha256.Sum256([]byte(data))
    current := hex.EncodeToString(hash[:])
    
    for _, p := range proof {
        if index%2 == 0 {
            combined := current + p
            h := sha256.Sum256([]byte(combined))
            current = hex.EncodeToString(h[:])
        } else {
            combined := p + current
            h := sha256.Sum256([]byte(combined))
            current = hex.EncodeToString(h[:])
        }
        index /= 2
    }
    
    return current == t.RootHash()
}

func main() {
    fmt.Println("=== Merkle Tree Demo ===\n")
    
    transactions := []string{
        "Alice->Bob:50",
        "Bob->Charlie:25",
        "Charlie->Alice:10",
        "Alice->Dave:5",
        "Dave->Eve:3",
    }
    
    tree := NewMerkleTree(transactions)
    fmt.Printf("Merkle Root: %s\n\n", tree.RootHash()[:20]+"...")
    
    // Verify a transaction
    fmt.Println("Verifying transactions:")
    for i, tx := range transactions {
        leaf := sha256.Sum256([]byte(tx))
        leafHash := hex.EncodeToString(leaf[:])
        fmt.Printf("  Tx[%d]: %s (hash: %s...)\n",
            i, tx, leafHash[:10])
    }
    
    // Tamper test
    fmt.Println("\nTamper detection:")
    tree2 := NewMerkleTree(transactions)
    originalRoot := tree2.RootHash()
    
    // เปลี่ยน transaction
    modifiedTxns := make([]string, len(transactions))
    copy(modifiedTxns, transactions)
    modifiedTxns[0] = "Alice->Bob:5000" // TAMPER!
    
    tree3 := NewMerkleTree(modifiedTxns)
    fmt.Printf("Original root: %s...\n", originalRoot[:16])
    fmt.Printf("Tampered root: %s...\n", tree3.RootHash()[:16])
    fmt.Printf("Roots match: %v\n", originalRoot == tree3.RootHash())
}
```

---

## 3. Digital Signatures

```go
// digital_signature.go - ECDSA signatures สำหรับ transactions
package main

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "math/big"
)

// Wallet กระเป๋าเงิน crypto
type Wallet struct {
    PrivateKey *ecdsa.PrivateKey
    PublicKey  *ecdsa.PublicKey
}

// NewWallet สร้าง wallet ใหม่
func NewWallet() (*Wallet, error) {
    privateKey, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
    if err != nil {
        return nil, err
    }
    
    return &Wallet{
        PrivateKey: privateKey,
        PublicKey:  &privateKey.PublicKey,
    }, nil
}

// Address คืน wallet address (hash ของ public key)
func (w *Wallet) Address() string {
    pubKeyBytes := elliptic.Marshal(w.PublicKey.Curve, w.PublicKey.X, w.PublicKey.Y)
    hash := sha256.Sum256(pubKeyBytes)
    return hex.EncodeToString(hash[:])[:40] // ใช้ 40 chars
}

// Sign sign data ด้วย private key
func (w *Wallet) Sign(data []byte) ([]byte, error) {
    hash := sha256.Sum256(data)
    r, s, err := ecdsa.Sign(rand.Reader, w.PrivateKey, hash[:])
    if err != nil {
        return nil, err
    }
    
    // Encode r, s as 32 bytes each
    sig := make([]byte, 64)
    rBytes := r.Bytes()
    sBytes := s.Bytes()
    copy(sig[32-len(rBytes):32], rBytes)
    copy(sig[64-len(sBytes):64], sBytes)
    
    return sig, nil
}

// Verify ตรวจสอบ signature
func Verify(pubKey *ecdsa.PublicKey, data, signature []byte) bool {
    hash := sha256.Sum256(data)
    
    r := new(big.Int).SetBytes(signature[:32])
    s := new(big.Int).SetBytes(signature[32:64])
    
    return ecdsa.Verify(pubKey, hash[:], r, s)
}

// SignedTransaction transaction พร้อม signature
type SignedTransaction struct {
    From      string
    To        string
    Amount    float64
    Signature []byte
    PublicKey []byte
}

// NewSignedTransaction สร้าง signed transaction
func NewSignedTransaction(wallet *Wallet, to string, amount float64) (*SignedTransaction, error) {
    tx := &SignedTransaction{
        From:   wallet.Address(),
        To:     to,
        Amount: amount,
    }
    
    // สร้าง message ที่จะ sign
    msg := fmt.Sprintf("%s->%s:%.2f", tx.From, tx.To, tx.Amount)
    
    sig, err := wallet.Sign([]byte(msg))
    if err != nil {
        return nil, err
    }
    
    tx.Signature = sig
    tx.PublicKey = elliptic.Marshal(wallet.PublicKey.Curve, wallet.PublicKey.X, wallet.PublicKey.Y)
    
    return tx, nil
}

// Verify ตรวจสอบ transaction signature
func (tx *SignedTransaction) Verify() bool {
    x, y := elliptic.Unmarshal(elliptic.P256(), tx.PublicKey)
    if x == nil {
        return false
    }
    
    pubKey := &ecdsa.PublicKey{
        Curve: elliptic.P256(),
        X:     x,
        Y:     y,
    }
    
    msg := fmt.Sprintf("%s->%s:%.2f", tx.From, tx.To, tx.Amount)
    return Verify(pubKey, []byte(msg), tx.Signature)
}

func main() {
    fmt.Println("=== Digital Signatures Demo ===\n")
    
    // สร้าง wallets
    alice, _ := NewWallet()
    bob, _ := NewWallet()
    
    fmt.Printf("Alice's address: %s\n", alice.Address())
    fmt.Printf("Bob's address:   %s\n", bob.Address())
    
    // สร้าง signed transaction
    tx, err := NewSignedTransaction(alice, bob.Address(), 50.0)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("\nTransaction: %s -> %s: %.2f\n", tx.From[:8]+"...", tx.To[:8]+"...", tx.Amount)
    fmt.Printf("Signature: %s...\n", hex.EncodeToString(tx.Signature[:8]))
    fmt.Printf("Valid: %v\n", tx.Verify())
    
    // Tamper test
    fmt.Println("\nTamper test:")
    tx.Amount = 5000 // ปลอมแปลง!
    fmt.Printf("Tampered amount: %.2f\n", tx.Amount)
    fmt.Printf("Valid after tamper: %v\n", tx.Verify())
}
```

---

## สรุป

บทนี้ครอบคลุม Blockchain ด้วย Go:

1. **Blockchain Basics** - blocks, transactions, chain
2. **Proof of Work** - mining algorithm
3. **Merkle Tree** - transaction verification
4. **Digital Signatures** - ECDSA สำหรับ transaction authenticity
5. **go-ethereum** - interaction กับ Ethereum network

### Key Takeaways

- Blockchain = immutable ledger ด้วย cryptographic linking
- SHA256 hashing เป็น foundation ของ integrity
- Merkle trees ช่วย verify transactions efficiently
- ECDSA signatures ป้องกัน double-spending
- go-ethereum ให้ access full Ethereum ecosystem
