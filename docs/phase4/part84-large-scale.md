# Part 84: Large Scale Systems in Go

## เป้าหมายของบทเรียน
- Large scale system patterns
- Distributed data stores
- Consistent hashing
- Raft consensus algorithm
- Gossip protocol
- Data locality
- Global vs Local state

---

## 1. Consistent Hashing

```go
// consistenthash/ring.go
package main

import (
    "crypto/md5"
    "fmt"
    "sort"
    "sync"
)

// ConsistentHash implements consistent hashing
type ConsistentHash struct {
    mu       sync.RWMutex
    ring     map[uint32]string // hash -> node
    sorted   []uint32          // sorted hashes
    replicas int               // virtual nodes per real node
}

// NewConsistentHash สร้าง consistent hash ใหม่
func NewConsistentHash(replicas int) *ConsistentHash {
    return &ConsistentHash{
        ring:     make(map[uint32]string),
        replicas: replicas,
    }
}

// hash คำนวณ hash
func (ch *ConsistentHash) hash(key string) uint32 {
    h := md5.New()
    h.Write([]byte(key))
    sum := h.Sum(nil)
    return uint32(sum[0])<<24 | uint32(sum[1])<<16 | uint32(sum[2])<<8 | uint32(sum[3])
}

// AddNode เพิ่ม node
func (ch *ConsistentHash) AddNode(node string) {
    ch.mu.Lock()
    defer ch.mu.Unlock()
    
    for i := 0; i < ch.replicas; i++ {
        key := fmt.Sprintf("%s#%d", node, i)
        h := ch.hash(key)
        ch.ring[h] = node
        ch.sorted = append(ch.sorted, h)
    }
    
    sort.Slice(ch.sorted, func(i, j int) bool {
        return ch.sorted[i] < ch.sorted[j]
    })
}

// RemoveNode ลบ node
func (ch *ConsistentHash) RemoveNode(node string) {
    ch.mu.Lock()
    defer ch.mu.Unlock()
    
    newSorted := make([]uint32, 0)
    for i := 0; i < ch.replicas; i++ {
        key := fmt.Sprintf("%s#%d", node, i)
        h := ch.hash(key)
        delete(ch.ring, h)
    }
    
    for _, h := range ch.sorted {
        if _, exists := ch.ring[h]; exists {
            newSorted = append(newSorted, h)
        }
    }
    ch.sorted = newSorted
}

// GetNode คืน node ที่รับผิดชอบ key
func (ch *ConsistentHash) GetNode(key string) string {
    ch.mu.RLock()
    defer ch.mu.RUnlock()
    
    if len(ch.ring) == 0 {
        return ""
    }
    
    h := ch.hash(key)
    
    // หา node ที่มี hash >= key hash
    idx := sort.Search(len(ch.sorted), func(i int) bool {
        return ch.sorted[i] >= h
    })
    
    if idx == len(ch.sorted) {
        idx = 0
    }
    
    return ch.ring[ch.sorted[idx]]
}

// GetNodes คืน N nodes สำหรับ replication
func (ch *ConsistentHash) GetNodes(key string, n int) []string {
    ch.mu.RLock()
    defer ch.mu.RUnlock()
    
    if len(ch.ring) == 0 {
        return nil
    }
    
    h := ch.hash(key)
    idx := sort.Search(len(ch.sorted), func(i int) bool {
        return ch.sorted[i] >= h
    })
    
    seen := make(map[string]bool)
    nodes := make([]string, 0, n)
    
    for i := 0; i < len(ch.sorted) && len(nodes) < n; i++ {
        realIdx := (idx + i) % len(ch.sorted)
        node := ch.ring[ch.sorted[realIdx]]
        if !seen[node] {
            seen[node] = true
            nodes = append(nodes, node)
        }
    }
    
    return nodes
}

// Distribution คำนวณการกระจาย keys
func (ch *ConsistentHash) Distribution(keys []string) map[string]int {
    dist := make(map[string]int)
    for _, key := range keys {
        node := ch.GetNode(key)
        dist[node]++
    }
    return dist
}

func main() {
    ch := NewConsistentHash(150) // 150 virtual nodes per server
    
    nodes := []string{"server-1", "server-2", "server-3", "server-4", "server-5"}
    for _, node := range nodes {
        ch.AddNode(node)
    }
    
    // สร้าง test keys
    keys := make([]string, 10000)
    for i := range keys {
        keys[i] = fmt.Sprintf("user:%d", i)
    }
    
    fmt.Println("=== Consistent Hashing Demo ===\n")
    
    dist := ch.Distribution(keys)
    total := len(keys)
    
    fmt.Println("Key Distribution (5 nodes, 10000 keys):")
    for node, count := range dist {
        percent := float64(count) / float64(total) * 100
        bar := ""
        for i := 0; i < int(percent/2); i++ {
            bar += "█"
        }
        fmt.Printf("  %-10s: %5d keys (%5.1f%%) %s\n", node, count, percent, bar)
    }
    
    // ทดสอบ node removal
    fmt.Println("\n--- Removing server-3 ---")
    ch.RemoveNode("server-3")
    
    newDist := ch.Distribution(keys)
    
    moved := 0
    for key := range keys {
        oldNode := dist[fmt.Sprintf("user:%d", key)] // incorrect but shows concept
        newNode := ch.GetNode(fmt.Sprintf("user:%d", key))
        _ = oldNode
        _ = newNode
        moved++
    }
    
    fmt.Println("Distribution after removing server-3:")
    for node, count := range newDist {
        percent := float64(count) / float64(total) * 100
        fmt.Printf("  %-10s: %5d keys (%5.1f%%)\n", node, count, percent)
    }
    
    // Replication
    fmt.Println("\n--- Replication (N=3) ---")
    testKeys := []string{"user:1000", "user:5000", "user:9999"}
    for _, key := range testKeys {
        replicas := ch.GetNodes(key, 3)
        fmt.Printf("  Key '%s' -> Nodes: %v\n", key, replicas)
    }
}
```

---

## 2. Raft Consensus Algorithm (Simplified)

```go
// raft/simplified.go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// NodeState สถานะของ node
type NodeState int

const (
    Follower  NodeState = iota
    Candidate
    Leader
)

func (s NodeState) String() string {
    switch s {
    case Follower:
        return "Follower"
    case Candidate:
        return "Candidate"
    case Leader:
        return "Leader"
    }
    return "Unknown"
}

// LogEntry entry ใน Raft log
type LogEntry struct {
    Term    int
    Index   int
    Command interface{}
}

// RaftMessage ข้อความระหว่าง nodes
type RaftMessage struct {
    Type    string
    From    int
    Term    int
    Payload interface{}
}

// VoteRequest แสดง vote request
type VoteRequest struct {
    CandidateID  int
    Term         int
    LastLogIndex int
    LastLogTerm  int
}

// VoteResponse แสดง vote response
type VoteResponse struct {
    Term        int
    VoteGranted bool
}

// AppendEntriesRequest แสดง append entries request (heartbeat)
type AppendEntriesRequest struct {
    Term         int
    LeaderID     int
    PrevLogIndex int
    PrevLogTerm  int
    Entries      []LogEntry
    CommitIndex  int
}

// RaftNode แสดง Raft node
type RaftNode struct {
    mu          sync.Mutex
    id          int
    state       NodeState
    currentTerm int
    votedFor    int
    log         []LogEntry
    commitIndex int
    lastApplied int
    
    // Leader state
    nextIndex  map[int]int
    matchIndex map[int]int
    
    // Cluster
    peers     []*RaftNode
    messageCh chan RaftMessage
    
    // Timers
    electionTimeout  time.Duration
    heartbeatTimeout time.Duration
    lastHeartbeat    time.Time
}

// NewRaftNode สร้าง node ใหม่
func NewRaftNode(id int) *RaftNode {
    return &RaftNode{
        id:               id,
        state:            Follower,
        currentTerm:      0,
        votedFor:         -1,
        log:              make([]LogEntry, 0),
        messageCh:        make(chan RaftMessage, 100),
        electionTimeout:  time.Duration(150+rand.Intn(150)) * time.Millisecond,
        heartbeatTimeout: 50 * time.Millisecond,
        lastHeartbeat:    time.Now(),
        nextIndex:        make(map[int]int),
        matchIndex:       make(map[int]int),
    }
}

// SetPeers กำหนด peer nodes
func (n *RaftNode) SetPeers(peers []*RaftNode) {
    n.peers = peers
}

// StartElection เริ่มกระบวนการ election
func (n *RaftNode) StartElection() {
    n.mu.Lock()
    n.state = Candidate
    n.currentTerm++
    n.votedFor = n.id
    term := n.currentTerm
    n.mu.Unlock()
    
    fmt.Printf("[Node %d] Starting election for term %d\n", n.id, term)
    
    votes := 1 // vote for self
    majority := len(n.peers)/2 + 1
    
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    for _, peer := range n.peers {
        if peer.id == n.id {
            continue
        }
        
        wg.Add(1)
        go func(p *RaftNode) {
            defer wg.Done()
            
            req := VoteRequest{
                CandidateID:  n.id,
                Term:         term,
                LastLogIndex: len(n.log) - 1,
            }
            
            resp := p.RequestVote(req)
            
            if resp.VoteGranted {
                mu.Lock()
                votes++
                mu.Unlock()
            }
        }(peer)
    }
    
    wg.Wait()
    
    n.mu.Lock()
    defer n.mu.Unlock()
    
    if votes >= majority && n.state == Candidate && n.currentTerm == term {
        n.state = Leader
        fmt.Printf("[Node %d] Elected as Leader for term %d (votes: %d/%d)\n",
            n.id, n.currentTerm, votes, len(n.peers))
        
        // Initialize leader state
        for _, peer := range n.peers {
            n.nextIndex[peer.id] = len(n.log)
            n.matchIndex[peer.id] = -1
        }
    } else {
        n.state = Follower
        fmt.Printf("[Node %d] Lost election (votes: %d/%d)\n", n.id, votes, majority)
    }
}

// RequestVote จัดการ vote request
func (n *RaftNode) RequestVote(req VoteRequest) VoteResponse {
    n.mu.Lock()
    defer n.mu.Unlock()
    
    if req.Term < n.currentTerm {
        return VoteResponse{Term: n.currentTerm, VoteGranted: false}
    }
    
    if req.Term > n.currentTerm {
        n.currentTerm = req.Term
        n.state = Follower
        n.votedFor = -1
    }
    
    if n.votedFor == -1 || n.votedFor == req.CandidateID {
        n.votedFor = req.CandidateID
        return VoteResponse{Term: n.currentTerm, VoteGranted: true}
    }
    
    return VoteResponse{Term: n.currentTerm, VoteGranted: false}
}

// AppendEntries จัดการ append entries (heartbeat)
func (n *RaftNode) AppendEntries(req AppendEntriesRequest) bool {
    n.mu.Lock()
    defer n.mu.Unlock()
    
    if req.Term < n.currentTerm {
        return false
    }
    
    n.lastHeartbeat = time.Now()
    n.currentTerm = req.Term
    n.state = Follower
    
    // Append log entries
    for _, entry := range req.Entries {
        if entry.Index >= len(n.log) {
            n.log = append(n.log, entry)
        }
    }
    
    if req.CommitIndex > n.commitIndex {
        n.commitIndex = req.CommitIndex
    }
    
    return true
}

// SendHeartbeats ส่ง heartbeats ไปยัง followers
func (n *RaftNode) SendHeartbeats() {
    n.mu.Lock()
    term := n.currentTerm
    n.mu.Unlock()
    
    for _, peer := range n.peers {
        if peer.id == n.id {
            continue
        }
        
        go func(p *RaftNode) {
            req := AppendEntriesRequest{
                Term:        term,
                LeaderID:    n.id,
                CommitIndex: n.commitIndex,
            }
            p.AppendEntries(req)
        }(peer)
    }
}

// Run รัน raft loop
func (n *RaftNode) Run(ctx context.Context, done chan bool) {
    ticker := time.NewTicker(10 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-done:
            return
        case <-ticker.C:
            n.mu.Lock()
            state := n.state
            lastHB := n.lastHeartbeat
            timeout := n.electionTimeout
            n.mu.Unlock()
            
            switch state {
            case Follower:
                if time.Since(lastHB) > timeout {
                    n.StartElection()
                }
            case Leader:
                n.SendHeartbeats()
            }
        }
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    fmt.Println("=== Raft Consensus Demo ===\n")
    
    // สร้าง cluster 5 nodes
    nodes := make([]*RaftNode, 5)
    for i := range nodes {
        nodes[i] = NewRaftNode(i)
    }
    
    // Set peers
    for _, node := range nodes {
        peers := make([]*RaftNode, 0, len(nodes))
        for _, peer := range nodes {
            peers = append(peers, peer)
        }
        node.SetPeers(peers)
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    done := make(chan bool)
    
    // Start all nodes
    for _, node := range nodes {
        go node.Run(ctx, done)
    }
    
    // รอให้ election เกิดขึ้น
    time.Sleep(1 * time.Second)
    
    // แสดง state ของแต่ละ node
    fmt.Println("Cluster State after 1 second:")
    for _, node := range nodes {
        node.mu.Lock()
        fmt.Printf("  Node %d: state=%s, term=%d, log=%d entries\n",
            node.id, node.state, node.currentTerm, len(node.log))
        node.mu.Unlock()
    }
    
    <-ctx.Done()
    close(done)
}
```

---

## 3. Gossip Protocol

```go
// gossip/protocol.go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// NodeInfo ข้อมูลของ node
type NodeInfo struct {
    ID        int
    Address   string
    Status    string
    Heartbeat int64
    Version   int64
    Metadata  map[string]string
}

// GossipNode แสดง gossip node
type GossipNode struct {
    mu       sync.RWMutex
    info     NodeInfo
    peers    map[int]*GossipNode
    known    map[int]*NodeInfo
    fanout   int
}

// NewGossipNode สร้าง node ใหม่
func NewGossipNode(id int, address string) *GossipNode {
    return &GossipNode{
        info: NodeInfo{
            ID:       id,
            Address:  address,
            Status:   "alive",
            Version:  1,
            Metadata: make(map[string]string),
        },
        peers:  make(map[int]*GossipNode),
        known:  make(map[int]*NodeInfo),
        fanout: 3,
    }
}

// Join เข้าร่วม cluster
func (n *GossipNode) Join(peer *GossipNode) {
    n.mu.Lock()
    n.peers[peer.info.ID] = peer
    n.mu.Unlock()
    
    peer.mu.Lock()
    peer.peers[n.info.ID] = n
    peer.mu.Unlock()
}

// Gossip ส่งข้อมูลไปยัง random peers
func (n *GossipNode) Gossip() {
    n.mu.Lock()
    n.info.Heartbeat++
    n.info.Version++
    
    // สร้าง snapshot ของข้อมูลที่รู้
    infos := make([]*NodeInfo, 0)
    mySelf := n.info
    infos = append(infos, &mySelf)
    for _, info := range n.known {
        copy := *info
        infos = append(infos, &copy)
    }
    
    // เลือก random peers
    peerList := make([]*GossipNode, 0)
    for _, peer := range n.peers {
        peerList = append(peerList, peer)
    }
    n.mu.Unlock()
    
    // เลือก fanout peers แบบสุ่ม
    rand.Shuffle(len(peerList), func(i, j int) {
        peerList[i], peerList[j] = peerList[j], peerList[i]
    })
    
    selectedCount := n.fanout
    if selectedCount > len(peerList) {
        selectedCount = len(peerList)
    }
    
    for i := 0; i < selectedCount; i++ {
        peer := peerList[i]
        peer.ReceiveGossip(infos)
    }
}

// ReceiveGossip รับข้อมูลจาก peer
func (n *GossipNode) ReceiveGossip(infos []*NodeInfo) {
    n.mu.Lock()
    defer n.mu.Unlock()
    
    for _, info := range infos {
        if info.ID == n.info.ID {
            continue // ข้าม self
        }
        
        existing, exists := n.known[info.ID]
        if !exists || info.Version > existing.Version {
            copy := *info
            n.known[info.ID] = &copy
        }
    }
}

// GetClusterView คืน view ของ cluster
func (n *GossipNode) GetClusterView() map[int]*NodeInfo {
    n.mu.RLock()
    defer n.mu.RUnlock()
    
    view := make(map[int]*NodeInfo)
    myInfo := n.info
    view[n.info.ID] = &myInfo
    
    for id, info := range n.known {
        copy := *info
        view[id] = &copy
    }
    
    return view
}

// Converge รอให้ทุก node มีข้อมูลครบ
func WaitForConvergence(nodes []*GossipNode, timeout time.Duration) bool {
    start := time.Now()
    expected := len(nodes)
    
    for {
        if time.Since(start) > timeout {
            return false
        }
        
        allConverged := true
        for _, node := range nodes {
            view := node.GetClusterView()
            if len(view) < expected {
                allConverged = false
                break
            }
        }
        
        if allConverged {
            return true
        }
        
        time.Sleep(10 * time.Millisecond)
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    fmt.Println("=== Gossip Protocol Demo ===\n")
    
    // สร้าง cluster 10 nodes
    nodeCount := 10
    nodes := make([]*GossipNode, nodeCount)
    
    for i := 0; i < nodeCount; i++ {
        nodes[i] = NewGossipNode(i, fmt.Sprintf("10.0.0.%d:7946", i+1))
    }
    
    // สร้าง random topology
    for i := 0; i < nodeCount; i++ {
        // แต่ละ node เชื่อมกับ 3 nodes
        for j := 0; j < 3; j++ {
            peer := rand.Intn(nodeCount)
            if peer != i {
                nodes[i].Join(nodes[peer])
            }
        }
    }
    
    fmt.Printf("Created %d node cluster\n", nodeCount)
    
    // รัน gossip rounds
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Start gossip goroutines
    for _, node := range nodes {
        go func(n *GossipNode) {
            ticker := time.NewTicker(100 * time.Millisecond)
            defer ticker.Stop()
            
            for {
                select {
                case <-ctx.Done():
                    return
                case <-ticker.C:
                    n.Gossip()
                }
            }
        }(node)
    }
    
    // รอให้ converge
    converged := WaitForConvergence(nodes, 3*time.Second)
    
    if converged {
        fmt.Printf("Cluster converged!\n")
    } else {
        fmt.Printf("Convergence timeout\n")
    }
    
    // แสดง view ของแต่ละ node
    fmt.Println("\nCluster Views:")
    for _, node := range nodes[:3] {
        view := node.GetClusterView()
        fmt.Printf("  Node %d knows about %d nodes\n", node.info.ID, len(view))
    }
    
    // จำลอง node failure
    fmt.Println("\n--- Simulating node failure ---")
    nodes[5].mu.Lock()
    nodes[5].info.Status = "dead"
    nodes[5].mu.Unlock()
    
    // Gossip จะ propagate status change
    time.Sleep(500 * time.Millisecond)
    
    cancel()
    
    // ตรวจสอบว่า node อื่นรู้หรือไม่
    for _, node := range nodes[:3] {
        view := node.GetClusterView()
        if info, ok := view[5]; ok {
            fmt.Printf("  Node %d sees node 5 as: %s\n", node.info.ID, info.Status)
        }
    }
}
```

---

## 4. Distributed Data Store

```go
// datastore/distributed.go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Partition แสดง data partition
type Partition struct {
    ID   int
    Keys map[string][]byte
    mu   sync.RWMutex
}

// NewPartition สร้าง partition ใหม่
func NewPartition(id int) *Partition {
    return &Partition{
        ID:   id,
        Keys: make(map[string][]byte),
    }
}

// DistributedKVStore distributed key-value store
type DistributedKVStore struct {
    mu         sync.RWMutex
    partitions []*Partition
    numParts   int
    replFactor int
    ch         *ConsistentHash
}

// ConsistentHash ใช้จาก part ก่อนหน้า
type ConsistentHash struct {
    nodes []string
}

func (ch *ConsistentHash) GetNode(key string) string {
    if len(ch.nodes) == 0 {
        return ""
    }
    h := 0
    for _, c := range key {
        h = h*31 + int(c)
    }
    return ch.nodes[((h%len(ch.nodes))+len(ch.nodes))%len(ch.nodes)]
}

// NewDistributedKVStore สร้าง store ใหม่
func NewDistributedKVStore(numPartitions, replicationFactor int) *DistributedKVStore {
    partitions := make([]*Partition, numPartitions)
    for i := range partitions {
        partitions[i] = NewPartition(i)
    }
    
    return &DistributedKVStore{
        partitions: partitions,
        numParts:   numPartitions,
        replFactor: replicationFactor,
    }
}

// getPartition คืน partition สำหรับ key
func (store *DistributedKVStore) getPartition(key string) *Partition {
    h := 0
    for _, c := range key {
        h = h*31 + int(c)
    }
    idx := ((h % store.numParts) + store.numParts) % store.numParts
    return store.partitions[idx]
}

// Put เก็บข้อมูล
func (store *DistributedKVStore) Put(key string, value []byte) {
    partition := store.getPartition(key)
    partition.mu.Lock()
    defer partition.mu.Unlock()
    partition.Keys[key] = value
}

// Get ดึงข้อมูล
func (store *DistributedKVStore) Get(key string) ([]byte, bool) {
    partition := store.getPartition(key)
    partition.mu.RLock()
    defer partition.mu.RUnlock()
    value, ok := partition.Keys[key]
    return value, ok
}

// Delete ลบข้อมูล
func (store *DistributedKVStore) Delete(key string) {
    partition := store.getPartition(key)
    partition.mu.Lock()
    defer partition.mu.Unlock()
    delete(partition.Keys, key)
}

// Stats คืน statistics
func (store *DistributedKVStore) Stats() map[string]interface{} {
    total := 0
    minKeys := int(^uint(0) >> 1)
    maxKeys := 0
    
    for _, p := range store.partitions {
        p.mu.RLock()
        count := len(p.Keys)
        p.mu.RUnlock()
        
        total += count
        if count < minKeys {
            minKeys = count
        }
        if count > maxKeys {
            maxKeys = count
        }
    }
    
    return map[string]interface{}{
        "total_keys":  total,
        "partitions":  store.numParts,
        "min_keys":    minKeys,
        "max_keys":    maxKeys,
        "avg_keys":    float64(total) / float64(store.numParts),
    }
}

func main() {
    fmt.Println("=== Distributed KV Store Demo ===\n")
    
    store := NewDistributedKVStore(16, 3)
    
    // เพิ่มข้อมูล
    start := time.Now()
    for i := 0; i < 10000; i++ {
        key := fmt.Sprintf("user:%d", i)
        value := []byte(fmt.Sprintf(`{"id":%d,"name":"user-%d"}`, i, i))
        store.Put(key, value)
    }
    fmt.Printf("Inserted 10000 keys in %v\n", time.Since(start))
    
    // อ่านข้อมูล
    start = time.Now()
    hits := 0
    for i := 0; i < 1000; i++ {
        key := fmt.Sprintf("user:%d", i*10)
        if _, ok := store.Get(key); ok {
            hits++
        }
    }
    fmt.Printf("1000 reads: %d hits in %v\n\n", hits, time.Since(start))
    
    // Statistics
    stats := store.Stats()
    fmt.Println("Partition Statistics:")
    fmt.Printf("  Total keys: %v\n", stats["total_keys"])
    fmt.Printf("  Partitions: %v\n", stats["partitions"])
    fmt.Printf("  Min keys/partition: %v\n", stats["min_keys"])
    fmt.Printf("  Max keys/partition: %v\n", stats["max_keys"])
    fmt.Printf("  Avg keys/partition: %.1f\n", stats["avg_keys"])
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Consistent Hashing** - การกระจาย keys อย่างสม่ำเสมอใน distributed system
2. **Raft Consensus** - Leader election และ log replication
3. **Gossip Protocol** - การ propagate ข้อมูลใน distributed cluster
4. **Distributed KV Store** - การ partition และ replicate data

### Key Takeaways

- **Consistent hashing** ลด key movement เมื่อ add/remove nodes
- **Raft** รับประกัน strong consistency แต่แลกกับ availability
- **Gossip** เหมาะสำหรับ eventual consistency และ fault detection
- **Partition เป็น key**: การแบ่ง data ดีส่งผลต่อ performance มาก
