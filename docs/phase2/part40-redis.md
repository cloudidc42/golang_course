# Part 40: Redis ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- เชื่อมต่อและใช้งาน Redis กับ go-redis client
- ใช้ basic operations: GET, SET, DEL, EXPIRE
- ใช้ Redis data structures: Lists, Sets, Sorted Sets, Hashes
- ใช้ Pub/Sub และ Redis Streams
- สร้าง Distributed Locks
- ใช้ Redis เป็น cache layer
- เชื่อมต่อ Redis Cluster
- สร้าง Session Store (Workshop)

---

## 1. ติดตั้งและเชื่อมต่อ Redis

```bash
# ติดตั้ง go-redis
go get github.com/go-redis/redis/v8

# หรือ v9
go get github.com/redis/go-redis/v9
```

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

// ตัวอย่าง 1: การเชื่อมต่อ Redis
func connectRedis() *redis.Client {
	client := redis.NewClient(&redis.Options{
		Addr:         "localhost:6379",
		Password:     "", // no password
		DB:           0,  // default DB
		PoolSize:     10, // connection pool size
		MinIdleConns: 3,  // minimum idle connections
		MaxRetries:   3,  // retry on failure
		DialTimeout:  5 * time.Second,
		ReadTimeout:  3 * time.Second,
		WriteTimeout: 3 * time.Second,
	})
	
	ctx := context.Background()
	
	// Test connection
	pong, err := client.Ping(ctx).Result()
	if err != nil {
		panic(fmt.Sprintf("Cannot connect to Redis: %v", err))
	}
	fmt.Printf("Redis connected: %s\n", pong)
	
	return client
}

// ตัวอย่าง 2: Connection with TLS
func connectRedisSecure() *redis.Client {
	return redis.NewClient(&redis.Options{
		Addr:      "your-redis-host:6380",
		Password:  "your-password",
		DB:        0,
		// TLSConfig: &tls.Config{...},
	})
}

func main() {
	client := connectRedis()
	defer client.Close()
	
	fmt.Printf("Redis info: DB=%d\n", client.Options().DB)
}
```

---

## 2. Basic Operations

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 3: String operations
func stringOperations(client *redis.Client) {
	fmt.Println("=== String Operations ===")
	
	// SET
	err := client.Set(ctx, "name", "Alice", 0).Err()
	if err != nil {
		panic(err)
	}
	
	// SET with expiration
	client.Set(ctx, "session:abc", "user:123", 30*time.Minute)
	
	// GET
	val, err := client.Get(ctx, "name").Result()
	if err != nil {
		panic(err)
	}
	fmt.Printf("GET name: %s\n", val)
	
	// GETSET - get old value and set new
	oldVal, err := client.GetSet(ctx, "name", "Bob").Result()
	fmt.Printf("GETSET name: old=%s\n", oldVal)
	
	// MSET - set multiple
	client.MSet(ctx, "key1", "val1", "key2", "val2", "key3", "val3")
	
	// MGET - get multiple
	vals, err := client.MGet(ctx, "key1", "key2", "key3", "nonexistent").Result()
	if err == nil {
		fmt.Printf("MGET: %v\n", vals)
	}
	
	// INCR / DECR
	client.Set(ctx, "counter", 10, 0)
	client.Incr(ctx, "counter")
	client.IncrBy(ctx, "counter", 5)
	count, _ := client.Get(ctx, "counter").Int()
	fmt.Printf("Counter after incr: %d\n", count)
	
	// SETNX - set if not exists
	set, _ := client.SetNX(ctx, "unique_key", "value", time.Hour).Result()
	fmt.Printf("SETNX: %v\n", set)
	
	// APPEND
	client.Set(ctx, "log", "2025-01-01", 0)
	client.Append(ctx, "log", " entry1")
	logVal, _ := client.Get(ctx, "log").Result()
	fmt.Printf("APPEND result: %s\n", logVal)
	
	// STRLEN
	length, _ := client.StrLen(ctx, "log").Result()
	fmt.Printf("STRLEN: %d\n", length)
	
	// TTL operations
	client.Set(ctx, "temp", "value", 10*time.Second)
	ttl, _ := client.TTL(ctx, "temp").Result()
	fmt.Printf("TTL: %v\n", ttl)
	
	// EXPIRE - update TTL
	client.Expire(ctx, "temp", 30*time.Second)
	
	// PERSIST - remove TTL
	client.Persist(ctx, "temp")
	
	// DEL
	client.Del(ctx, "name", "temp", "counter")
}

// ตัวอย่าง 4: Key scanning
func keyOperations(client *redis.Client) {
	fmt.Println("\n=== Key Operations ===")
	
	// Set some keys
	for i := 0; i < 10; i++ {
		client.Set(ctx, fmt.Sprintf("user:%d", i), fmt.Sprintf("data%d", i), time.Hour)
	}
	
	// KEYS pattern (avoid in production for large datasets)
	keys, _ := client.Keys(ctx, "user:*").Result()
	fmt.Printf("Keys matching 'user:*': %v\n", keys)
	
	// SCAN - safe iteration for large datasets
	var cursor uint64
	var allKeys []string
	for {
		var keys []string
		var err error
		keys, cursor, err = client.Scan(ctx, cursor, "user:*", 5).Result()
		if err != nil {
			break
		}
		allKeys = append(allKeys, keys...)
		if cursor == 0 {
			break
		}
	}
	fmt.Printf("SCAN found %d keys\n", len(allKeys))
	
	// EXISTS
	exists, _ := client.Exists(ctx, "user:1", "nonexistent").Result()
	fmt.Printf("EXISTS: %d keys found\n", exists)
	
	// TYPE
	t, _ := client.Type(ctx, "user:1").Result()
	fmt.Printf("TYPE: %s\n", t)
	
	// RENAME
	client.Rename(ctx, "user:0", "user:renamed")
	
	// RENAMENX
	renamed, _ := client.RenameNX(ctx, "user:renamed", "user:1").Result()
	fmt.Printf("RENAMENX: %v\n", renamed)
	
	// Cleanup
	client.FlushDB(ctx)
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	stringOperations(client)
	keyOperations(client)
}
```

---

## 3. Lists

```go
package main

import (
	"context"
	"fmt"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 5: List operations
func listOperations(client *redis.Client) {
	fmt.Println("=== List Operations ===")
	
	key := "mylist"
	client.Del(ctx, key)
	
	// LPUSH / RPUSH
	client.RPush(ctx, key, "apple", "banana", "cherry")
	client.LPush(ctx, key, "avocado") // adds to front
	
	// LLEN - list length
	length, _ := client.LLen(ctx, key).Result()
	fmt.Printf("List length: %d\n", length)
	
	// LRANGE - get elements
	vals, _ := client.LRange(ctx, key, 0, -1).Result()
	fmt.Printf("Full list: %v\n", vals)
	
	// LPOP / RPOP
	left, _ := client.LPop(ctx, key).Result()
	fmt.Printf("LPOP: %s\n", left)
	
	right, _ := client.RPop(ctx, key).Result()
	fmt.Printf("RPOP: %s\n", right)
	
	// LINDEX - get by index
	item, _ := client.LIndex(ctx, key, 0).Result()
	fmt.Printf("LINDEX 0: %s\n", item)
	
	// LSET - set value at index
	client.LSet(ctx, key, 0, "mango")
	
	// LINSERT - insert before/after
	client.LInsertBefore(ctx, key, "banana", "kiwi")
	
	// LREM - remove elements
	client.LRem(ctx, key, 1, "kiwi")
	
	// LTRIM - trim list
	client.LTrim(ctx, key, 0, 5)
	
	vals, _ = client.LRange(ctx, key, 0, -1).Result()
	fmt.Printf("After operations: %v\n", vals)
	
	// Queue operations (FIFO)
	fmt.Println("\n--- Queue (FIFO) ---")
	queue := "job_queue"
	client.Del(ctx, queue)
	client.RPush(ctx, queue, "job1", "job2", "job3")
	
	for {
		job, err := client.LPop(ctx, queue).Result()
		if err != nil {
			break
		}
		fmt.Printf("Processing: %s\n", job)
	}
	
	// Stack operations (LIFO)
	fmt.Println("\n--- Stack (LIFO) ---")
	stack := "task_stack"
	client.Del(ctx, stack)
	client.LPush(ctx, stack, "task1", "task2", "task3")
	
	for {
		task, err := client.LPop(ctx, stack).Result()
		if err != nil {
			break
		}
		fmt.Printf("Executing: %s\n", task)
	}
}

// ตัวอย่าง 6: Blocking list operations (for message queues)
func blockingListDemo(client *redis.Client) {
	fmt.Println("\n=== Blocking List Operations ===")
	
	go func() {
		// Producer
		time.Sleep(500 * time.Millisecond)
		client.RPush(ctx, "work_queue", "work_item_1")
		client.RPush(ctx, "work_queue", "work_item_2")
	}()
	
	// Consumer with BLPOP (blocks until item available)
	for i := 0; i < 2; i++ {
		result, err := client.BLPop(ctx, 2*time.Second, "work_queue").Result()
		if err != nil {
			fmt.Printf("Timeout or error: %v\n", err)
			break
		}
		fmt.Printf("BLPOP received from %s: %s\n", result[0], result[1])
	}
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	listOperations(client)
	blockingListDemo(client)
}
```

---

## 4. Sets, Sorted Sets, Hashes

```go
package main

import (
	"context"
	"fmt"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 7: Set operations
func setOperations(client *redis.Client) {
	fmt.Println("=== Set Operations ===")
	
	// SADD - add members
	client.Del(ctx, "set1", "set2")
	client.SAdd(ctx, "set1", "a", "b", "c", "d")
	client.SAdd(ctx, "set2", "c", "d", "e", "f")
	
	// SMEMBERS - all members
	members, _ := client.SMembers(ctx, "set1").Result()
	fmt.Printf("set1: %v\n", members)
	
	// SISMEMBER - check membership
	isMember, _ := client.SIsMember(ctx, "set1", "a").Result()
	fmt.Printf("'a' in set1: %v\n", isMember)
	
	// SCARD - count
	count, _ := client.SCard(ctx, "set1").Result()
	fmt.Printf("set1 size: %d\n", count)
	
	// SUNION - union
	union, _ := client.SUnion(ctx, "set1", "set2").Result()
	fmt.Printf("union: %v\n", union)
	
	// SINTER - intersection
	inter, _ := client.SInter(ctx, "set1", "set2").Result()
	fmt.Printf("intersection: %v\n", inter)
	
	// SDIFF - difference
	diff, _ := client.SDiff(ctx, "set1", "set2").Result()
	fmt.Printf("set1 - set2: %v\n", diff)
	
	// SRANDMEMBER - random member
	randMember, _ := client.SRandMember(ctx, "set1").Result()
	fmt.Printf("random from set1: %s\n", randMember)
	
	// SPOP - remove and return random
	popped, _ := client.SPop(ctx, "set1").Result()
	fmt.Printf("SPOP from set1: %s\n", popped)
	
	// SMOVE - move member between sets
	client.SMove(ctx, "set1", "set2", "b")
}

// ตัวอย่าง 8: Sorted Set operations
func sortedSetOperations(client *redis.Client) {
	fmt.Println("\n=== Sorted Set Operations ===")
	
	leaderboard := "game:leaderboard"
	client.Del(ctx, leaderboard)
	
	// ZADD - add with score
	client.ZAdd(ctx, leaderboard, 
		&redis.Z{Score: 100, Member: "Alice"},
		&redis.Z{Score: 250, Member: "Bob"},
		&redis.Z{Score: 175, Member: "Charlie"},
		&redis.Z{Score: 320, Member: "Diana"},
		&redis.Z{Score: 90, Member: "Eve"},
	)
	
	// ZRANGE - range by rank (low to high)
	rankings, _ := client.ZRangeWithScores(ctx, leaderboard, 0, -1).Result()
	fmt.Println("Leaderboard (low to high):")
	for i, z := range rankings {
		fmt.Printf("  %d. %s: %.0f\n", i+1, z.Member, z.Score)
	}
	
	// ZREVRANGE - range by rank (high to low)
	top3, _ := client.ZRevRangeWithScores(ctx, leaderboard, 0, 2).Result()
	fmt.Println("Top 3 players:")
	for i, z := range top3 {
		fmt.Printf("  %d. %s: %.0f\n", i+1, z.Member, z.Score)
	}
	
	// ZSCORE - get score
	score, _ := client.ZScore(ctx, leaderboard, "Bob").Result()
	fmt.Printf("Bob's score: %.0f\n", score)
	
	// ZRANK - get rank (0-indexed, low to high)
	rank, _ := client.ZRank(ctx, leaderboard, "Bob").Result()
	fmt.Printf("Bob's rank (low-to-high): %d\n", rank)
	
	// ZREVRANK - rank from high to low
	revRank, _ := client.ZRevRank(ctx, leaderboard, "Bob").Result()
	fmt.Printf("Bob's rank (high-to-low): %d\n", revRank)
	
	// ZINCRBY - increment score
	client.ZIncrBy(ctx, leaderboard, 50, "Alice")
	newScore, _ := client.ZScore(ctx, leaderboard, "Alice").Result()
	fmt.Printf("Alice's new score: %.0f\n", newScore)
	
	// ZRANGEBYSCORE - range by score
	players, _ := client.ZRangeByScoreWithScores(ctx, leaderboard, &redis.ZRangeBy{
		Min: "100",
		Max: "300",
	}).Result()
	fmt.Println("Players scoring 100-300:")
	for _, z := range players {
		fmt.Printf("  %s: %.0f\n", z.Member, z.Score)
	}
	
	// ZCOUNT - count in score range
	count, _ := client.ZCount(ctx, leaderboard, "100", "+inf").Result()
	fmt.Printf("Players scoring >= 100: %d\n", count)
	
	// ZCARD - total count
	total, _ := client.ZCard(ctx, leaderboard).Result()
	fmt.Printf("Total players: %d\n", total)
	
	// ZREM
	client.ZRem(ctx, leaderboard, "Eve")
}

// ตัวอย่าง 9: Hash operations
func hashOperations(client *redis.Client) {
	fmt.Println("\n=== Hash Operations ===")
	
	userKey := "user:1001"
	client.Del(ctx, userKey)
	
	// HSET - set fields
	client.HSet(ctx, userKey, 
		"name", "Alice",
		"email", "alice@example.com",
		"age", 30,
		"city", "Bangkok",
	)
	
	// HGET - get single field
	name, _ := client.HGet(ctx, userKey, "name").Result()
	fmt.Printf("HGET name: %s\n", name)
	
	// HMGET - get multiple fields
	vals, _ := client.HMGet(ctx, userKey, "name", "email", "age").Result()
	fmt.Printf("HMGET: %v\n", vals)
	
	// HGETALL - get all fields
	allFields, _ := client.HGetAll(ctx, userKey).Result()
	fmt.Printf("HGETALL: %v\n", allFields)
	
	// HKEYS / HVALS
	keys, _ := client.HKeys(ctx, userKey).Result()
	fmt.Printf("HKEYS: %v\n", keys)
	
	vals2, _ := client.HVals(ctx, userKey).Result()
	fmt.Printf("HVALS: %v\n", vals2)
	
	// HLEN
	length, _ := client.HLen(ctx, userKey).Result()
	fmt.Printf("HLEN: %d\n", length)
	
	// HEXISTS
	exists, _ := client.HExists(ctx, userKey, "email").Result()
	fmt.Printf("HEXISTS email: %v\n", exists)
	
	// HDEL
	client.HDel(ctx, userKey, "city")
	
	// HINCRBY - increment numeric field
	client.HIncrBy(ctx, userKey, "age", 1)
	age, _ := client.HGet(ctx, userKey, "age").Result()
	fmt.Printf("Age after increment: %s\n", age)
	
	// HSET multiple at once
	client.HSet(ctx, "product:1", map[string]interface{}{
		"name":     "Widget",
		"price":    9.99,
		"stock":    100,
		"category": "electronics",
	})
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	setOperations(client)
	sortedSetOperations(client)
	hashOperations(client)
}
```

---

## 5. Pub/Sub

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 10: Pub/Sub basic
func pubSubDemo(client *redis.Client) {
	fmt.Println("=== Pub/Sub Demo ===")
	
	// Subscribe to channels
	pubsub := client.Subscribe(ctx, "news", "sports")
	defer pubsub.Close()
	
	// Receive messages in goroutine
	go func() {
		for msg := range pubsub.Channel() {
			fmt.Printf("Received on [%s]: %s\n", msg.Channel, msg.Payload)
		}
	}()
	
	// Publish messages
	time.Sleep(100 * time.Millisecond)
	client.Publish(ctx, "news", "Breaking: Go 2.0 released!")
	client.Publish(ctx, "sports", "Thailand wins gold medal!")
	client.Publish(ctx, "news", "Weather: Sunny today")
	
	time.Sleep(200 * time.Millisecond)
}

// ตัวอย่าง 11: Pattern subscribe
func patternSubDemo(client *redis.Client) {
	fmt.Println("\n=== Pattern Subscribe ===")
	
	// Subscribe to all channels matching "user:*"
	pubsub := client.PSubscribe(ctx, "user:*", "order:*")
	defer pubsub.Close()
	
	go func() {
		for msg := range pubsub.Channel() {
			switch m := msg.(type) {
			case *redis.Message:
				fmt.Printf("Message on [%s]: %s\n", m.Channel, m.Payload)
			case *redis.Subscription:
				fmt.Printf("Subscription: %s %s\n", m.Kind, m.Channel)
			}
		}
	}()
	
	time.Sleep(100 * time.Millisecond)
	client.Publish(ctx, "user:login", `{"user_id": 123, "ip": "192.168.1.1"}`)
	client.Publish(ctx, "user:logout", `{"user_id": 456}`)
	client.Publish(ctx, "order:created", `{"order_id": 789, "amount": 99.99}`)
	
	time.Sleep(200 * time.Millisecond)
}

// ตัวอย่าง 12: Chat room using Pub/Sub
type ChatRoom struct {
	client  *redis.Client
	channel string
}

func NewChatRoom(client *redis.Client, roomName string) *ChatRoom {
	return &ChatRoom{
		client:  client,
		channel: fmt.Sprintf("chat:%s", roomName),
	}
}

func (cr *ChatRoom) Join(username string) (<-chan string, func()) {
	pubsub := cr.client.Subscribe(ctx, cr.channel)
	messages := make(chan string, 100)
	
	go func() {
		for msg := range pubsub.Channel() {
			messages <- msg.Payload
		}
		close(messages)
	}()
	
	// Announce join
	cr.client.Publish(ctx, cr.channel, fmt.Sprintf("%s joined the room", username))
	
	leave := func() {
		cr.client.Publish(ctx, cr.channel, fmt.Sprintf("%s left the room", username))
		pubsub.Close()
	}
	
	return messages, leave
}

func (cr *ChatRoom) Send(username, message string) {
	cr.client.Publish(ctx, cr.channel, fmt.Sprintf("[%s] %s", username, message))
}

func chatRoomDemo(client *redis.Client) {
	fmt.Println("\n=== Chat Room Demo ===")
	
	room := NewChatRoom(client, "general")
	
	// Alice joins
	aliceMessages, aliceLeave := room.Join("Alice")
	defer aliceLeave()
	
	// Bob joins
	bobMessages, bobLeave := room.Join("Bob")
	defer bobLeave()
	
	go func() {
		for msg := range aliceMessages {
			fmt.Printf("Alice sees: %s\n", msg)
		}
	}()
	
	go func() {
		for msg := range bobMessages {
			fmt.Printf("Bob sees: %s\n", msg)
		}
	}()
	
	time.Sleep(100 * time.Millisecond)
	room.Send("Alice", "Hello everyone!")
	room.Send("Bob", "Hi Alice!")
	room.Send("Alice", "How are you?")
	
	time.Sleep(200 * time.Millisecond)
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	pubSubDemo(client)
	patternSubDemo(client)
	chatRoomDemo(client)
}
```

---

## 6. Redis Streams

```go
package main

import (
	"context"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 13: Redis Streams - XADD / XREAD
func redisStreamsDemo(client *redis.Client) {
	fmt.Println("=== Redis Streams Demo ===")
	
	streamKey := "events"
	client.Del(ctx, streamKey)
	
	// XADD - add messages to stream
	for i := 1; i <= 5; i++ {
		id, _ := client.XAdd(ctx, &redis.XAddArgs{
			Stream: streamKey,
			Values: map[string]interface{}{
				"event_type": "click",
				"user_id":    i * 100,
				"page":       fmt.Sprintf("/page%d", i),
				"timestamp":  time.Now().Unix(),
			},
		}).Result()
		fmt.Printf("Added message: %s\n", id)
	}
	
	// XLEN - stream length
	length, _ := client.XLen(ctx, streamKey).Result()
	fmt.Printf("Stream length: %d\n", length)
	
	// XRANGE - read messages
	messages, _ := client.XRange(ctx, streamKey, "-", "+").Result()
	fmt.Println("\nAll messages:")
	for _, msg := range messages {
		fmt.Printf("  ID: %s, Values: %v\n", msg.ID, msg.Values)
	}
	
	// XREAD - read from position
	results, _ := client.XRead(ctx, &redis.XReadArgs{
		Streams: []string{streamKey, "0"},
		Count:   3,
		Block:   0,
	}).Result()
	
	fmt.Printf("\nXREAD (first 3):")
	for _, stream := range results {
		for _, msg := range stream.Messages {
			fmt.Printf("  %s: %v\n", msg.ID, msg.Values)
		}
	}
}

// ตัวอย่าง 14: Consumer Groups
func consumerGroupsDemo(client *redis.Client) {
	fmt.Println("\n=== Consumer Groups Demo ===")
	
	streamKey := "orders"
	groupName := "order-processors"
	client.Del(ctx, streamKey)
	
	// Add some messages
	for i := 1; i <= 6; i++ {
		client.XAdd(ctx, &redis.XAddArgs{
			Stream: streamKey,
			Values: map[string]interface{}{
				"order_id": i * 1000,
				"amount":   float64(i) * 99.99,
				"status":   "pending",
			},
		})
	}
	
	// Create consumer group
	client.XGroupCreate(ctx, streamKey, groupName, "0")
	
	// Consumer 1 reads
	msgs1, _ := client.XReadGroup(ctx, &redis.XReadGroupArgs{
		Group:    groupName,
		Consumer: "worker-1",
		Streams:  []string{streamKey, ">"},
		Count:    3,
	}).Result()
	
	fmt.Println("Worker-1 got:")
	for _, stream := range msgs1 {
		for _, msg := range stream.Messages {
			fmt.Printf("  %s: %v\n", msg.ID, msg.Values)
			// Acknowledge processing
			client.XAck(ctx, streamKey, groupName, msg.ID)
		}
	}
	
	// Consumer 2 reads remaining
	msgs2, _ := client.XReadGroup(ctx, &redis.XReadGroupArgs{
		Group:    groupName,
		Consumer: "worker-2",
		Streams:  []string{streamKey, ">"},
		Count:    10,
	}).Result()
	
	fmt.Println("Worker-2 got:")
	for _, stream := range msgs2 {
		for _, msg := range stream.Messages {
			fmt.Printf("  %s: %v\n", msg.ID, msg.Values)
			client.XAck(ctx, streamKey, groupName, msg.ID)
		}
	}
	
	// XPENDING - check pending messages
	pending, _ := client.XPending(ctx, streamKey, groupName).Result()
	fmt.Printf("\nPending messages: %d\n", pending.Count)
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	redisStreamsDemo(client)
	consumerGroupsDemo(client)
}
```

---

## 7. Distributed Locks

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 15: Simple distributed lock
type RedisLock struct {
	client  *redis.Client
	key     string
	value   string
	ttl     time.Duration
}

func NewRedisLock(client *redis.Client, resource string, ttl time.Duration) *RedisLock {
	return &RedisLock{
		client: client,
		key:    fmt.Sprintf("lock:%s", resource),
		value:  fmt.Sprintf("%d", rand.Int63()),
		ttl:    ttl,
	}
}

func (l *RedisLock) TryAcquire(ctx context.Context) (bool, error) {
	// SET key value NX EX ttl
	result, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
	if err != nil {
		return false, fmt.Errorf("lock acquire failed: %w", err)
	}
	return result, nil
}

func (l *RedisLock) Acquire(ctx context.Context, timeout time.Duration) error {
	deadline := time.Now().Add(timeout)
	
	for time.Now().Before(deadline) {
		acquired, err := l.TryAcquire(ctx)
		if err != nil {
			return err
		}
		if acquired {
			return nil
		}
		
		// Wait before retry with jitter
		waitTime := 50 + time.Duration(rand.Intn(50))*time.Millisecond
		time.Sleep(waitTime)
	}
	
	return fmt.Errorf("lock acquisition timed out after %v", timeout)
}

// Release using Lua script to ensure atomicity
const releaseScript = `
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
`

func (l *RedisLock) Release(ctx context.Context) error {
	result, err := l.client.Eval(ctx, releaseScript, []string{l.key}, l.value).Int()
	if err != nil {
		return fmt.Errorf("lock release failed: %w", err)
	}
	if result == 0 {
		return fmt.Errorf("lock was already expired or released by another process")
	}
	return nil
}

func (l *RedisLock) Extend(ctx context.Context) error {
	const extendScript = `
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("pexpire", KEYS[1], ARGV[2])
else
    return 0
end
`
	result, err := l.client.Eval(ctx, extendScript,
		[]string{l.key}, l.value, l.ttl.Milliseconds()).Int()
	if err != nil {
		return err
	}
	if result == 0 {
		return fmt.Errorf("failed to extend lock: lock expired or not owned")
	}
	return nil
}

// ตัวอย่าง 16: Using distributed lock
func distributedLockDemo(client *redis.Client) {
	fmt.Println("=== Distributed Lock Demo ===")
	
	var wg sync.WaitGroup
	results := make(chan string, 10)
	
	// Simulate multiple processes competing for lock
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(processID int) {
			defer wg.Done()
			
			lock := NewRedisLock(client, "shared-resource", 5*time.Second)
			
			err := lock.Acquire(ctx, 3*time.Second)
			if err != nil {
				results <- fmt.Sprintf("Process %d: FAILED to acquire lock: %v", processID, err)
				return
			}
			
			results <- fmt.Sprintf("Process %d: ACQUIRED lock", processID)
			
			// Simulate work
			time.Sleep(time.Duration(100+rand.Intn(200)) * time.Millisecond)
			
			// Release lock
			if err := lock.Release(ctx); err != nil {
				results <- fmt.Sprintf("Process %d: error releasing: %v", processID, err)
			} else {
				results <- fmt.Sprintf("Process %d: RELEASED lock", processID)
			}
		}(i)
	}
	
	go func() {
		wg.Wait()
		close(results)
	}()
	
	for r := range results {
		fmt.Printf("  %s\n", r)
	}
}

// ตัวอย่าง 17: Redlock algorithm (multi-node)
// In production, use a library like redsync
func redlockConcept() {
	fmt.Println("\n=== Redlock Concept ===")
	fmt.Println("Redlock algorithm uses N independent Redis nodes (N >= 3)")
	fmt.Println("Lock is acquired when majority (N/2 + 1) nodes confirm")
	fmt.Println()
	fmt.Println("Steps:")
	fmt.Println("1. Get current timestamp")
	fmt.Println("2. Try to acquire lock on all N nodes with TTL")
	fmt.Println("3. Calculate elapsed time")
	fmt.Println("4. Lock is valid if: majority acquired AND elapsed < TTL")
	fmt.Println("5. If not valid, release all acquired locks")
	fmt.Println()
	fmt.Println("In production, use: github.com/go-redsync/redsync")
}

func main() {
	rand.Seed(time.Now().UnixNano())
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	distributedLockDemo(client)
	redlockConcept()
}
```

---

## 8. Redis as Cache Layer

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"
	
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// ตัวอย่าง 18: Redis cache helper

type RedisCache struct {
	client *redis.Client
	prefix string
}

func NewRedisCache(client *redis.Client, prefix string) *RedisCache {
	return &RedisCache{client: client, prefix: prefix}
}

func (rc *RedisCache) key(k string) string {
	return rc.prefix + ":" + k
}

func (rc *RedisCache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
	data, err := json.Marshal(value)
	if err != nil {
		return fmt.Errorf("marshal error: %w", err)
	}
	
	return rc.client.Set(ctx, rc.key(key), data, ttl).Err()
}

func (rc *RedisCache) Get(ctx context.Context, key string, dest interface{}) error {
	data, err := rc.client.Get(ctx, rc.key(key)).Bytes()
	if err != nil {
		if err == redis.Nil {
			return fmt.Errorf("cache miss: %s", key)
		}
		return fmt.Errorf("redis get error: %w", err)
	}
	
	return json.Unmarshal(data, dest)
}

func (rc *RedisCache) Delete(ctx context.Context, keys ...string) error {
	fullKeys := make([]string, len(keys))
	for i, k := range keys {
		fullKeys[i] = rc.key(k)
	}
	return rc.client.Del(ctx, fullKeys...).Err()
}

func (rc *RedisCache) GetOrSet(ctx context.Context, key string, dest interface{}, 
	loader func() (interface{}, error), ttl time.Duration) error {
	
	err := rc.Get(ctx, key, dest)
	if err == nil {
		return nil // cache hit
	}
	
	// Cache miss, load from source
	value, err := loader()
	if err != nil {
		return fmt.Errorf("loader error: %w", err)
	}
	
	// Store in cache
	if err := rc.Set(ctx, key, value, ttl); err != nil {
		fmt.Printf("Warning: failed to cache %s: %v\n", key, err)
	}
	
	// Copy to dest
	data, _ := json.Marshal(value)
	return json.Unmarshal(data, dest)
}

// ตัวอย่าง 19: Using Redis cache
type Product struct {
	ID       int     `json:"id"`
	Name     string  `json:"name"`
	Price    float64 `json:"price"`
	Category string  `json:"category"`
}

func redisCacheDemo(client *redis.Client) {
	fmt.Println("=== Redis Cache Demo ===")
	
	cache := NewRedisCache(client, "myapp")
	
	// Cache a product
	product := Product{
		ID:       1,
		Name:     "Widget Pro",
		Price:    29.99,
		Category: "electronics",
	}
	
	if err := cache.Set(ctx, "product:1", product, 5*time.Minute); err != nil {
		fmt.Printf("Error caching: %v\n", err)
		return
	}
	fmt.Println("Product cached")
	
	// Retrieve from cache
	var cached Product
	if err := cache.Get(ctx, "product:1", &cached); err != nil {
		fmt.Printf("Cache miss: %v\n", err)
	} else {
		fmt.Printf("From cache: %+v\n", cached)
	}
	
	// GetOrSet pattern
	var product2 Product
	err := cache.GetOrSet(ctx, "product:2", &product2, func() (interface{}, error) {
		fmt.Println("Loading product 2 from DB...")
		time.Sleep(50 * time.Millisecond) // simulate DB query
		return Product{ID: 2, Name: "Gadget Plus", Price: 49.99, Category: "electronics"}, nil
	}, 5*time.Minute)
	
	if err != nil {
		fmt.Printf("Error: %v\n", err)
	} else {
		fmt.Printf("Got product 2: %+v\n", product2)
	}
	
	// Second call - should be from cache
	var product2Again Product
	cache.GetOrSet(ctx, "product:2", &product2Again, func() (interface{}, error) {
		fmt.Println("This should NOT be printed (cache hit)")
		return nil, nil
	}, 5*time.Minute)
	fmt.Printf("Got product 2 again: %+v\n", product2Again)
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	redisCacheDemo(client)
}
```

---

## Workshop: Session Store

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"net/http"
	"time"
	
	"github.com/gin-gonic/gin"
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

// Workshop: Session Store using Redis

type Session struct {
	ID        string            `json:"id"`
	UserID    int               `json:"user_id"`
	UserName  string            `json:"user_name"`
	Role      string            `json:"role"`
	Data      map[string]string `json:"data"`
	CreatedAt time.Time         `json:"created_at"`
	ExpiresAt time.Time         `json:"expires_at"`
}

type SessionStore struct {
	client  *redis.Client
	prefix  string
	ttl     time.Duration
}

func NewSessionStore(client *redis.Client, ttl time.Duration) *SessionStore {
	return &SessionStore{
		client: client,
		prefix: "session:",
		ttl:    ttl,
	}
}

func generateSessionID() (string, error) {
	bytes := make([]byte, 32)
	if _, err := rand.Read(bytes); err != nil {
		return "", err
	}
	return hex.EncodeToString(bytes), nil
}

func (ss *SessionStore) Create(ctx context.Context, userID int, userName, role string) (*Session, error) {
	sessionID, err := generateSessionID()
	if err != nil {
		return nil, fmt.Errorf("failed to generate session ID: %w", err)
	}
	
	now := time.Now()
	session := &Session{
		ID:        sessionID,
		UserID:    userID,
		UserName:  userName,
		Role:      role,
		Data:      make(map[string]string),
		CreatedAt: now,
		ExpiresAt: now.Add(ss.ttl),
	}
	
	if err := ss.Save(ctx, session); err != nil {
		return nil, err
	}
	
	return session, nil
}

func (ss *SessionStore) Save(ctx context.Context, session *Session) error {
	data, err := json.Marshal(session)
	if err != nil {
		return fmt.Errorf("failed to marshal session: %w", err)
	}
	
	key := ss.prefix + session.ID
	return ss.client.Set(ctx, key, data, ss.ttl).Err()
}

func (ss *SessionStore) Get(ctx context.Context, sessionID string) (*Session, error) {
	key := ss.prefix + sessionID
	
	data, err := ss.client.Get(ctx, key).Bytes()
	if err != nil {
		if err == redis.Nil {
			return nil, fmt.Errorf("session not found or expired")
		}
		return nil, fmt.Errorf("redis error: %w", err)
	}
	
	var session Session
	if err := json.Unmarshal(data, &session); err != nil {
		return nil, fmt.Errorf("failed to unmarshal session: %w", err)
	}
	
	return &session, nil
}

func (ss *SessionStore) Delete(ctx context.Context, sessionID string) error {
	key := ss.prefix + sessionID
	return ss.client.Del(ctx, key).Err()
}

func (ss *SessionStore) Refresh(ctx context.Context, sessionID string) error {
	session, err := ss.Get(ctx, sessionID)
	if err != nil {
		return err
	}
	
	session.ExpiresAt = time.Now().Add(ss.ttl)
	return ss.Save(ctx, session)
}

func (ss *SessionStore) SetData(ctx context.Context, sessionID, key, value string) error {
	session, err := ss.Get(ctx, sessionID)
	if err != nil {
		return err
	}
	
	session.Data[key] = value
	return ss.Save(ctx, session)
}

func (ss *SessionStore) Count(ctx context.Context) (int64, error) {
	keys, err := ss.client.Keys(ctx, ss.prefix+"*").Result()
	if err != nil {
		return 0, err
	}
	return int64(len(keys)), nil
}

// Gin middleware for session management
func SessionMiddleware(store *SessionStore, cookieName string) gin.HandlerFunc {
	return func(c *gin.Context) {
		// Get session ID from cookie
		sessionID, err := c.Cookie(cookieName)
		if err == nil && sessionID != "" {
			// Load session
			if session, err := store.Get(ctx, sessionID); err == nil {
				c.Set("session", session)
				c.Set("user_id", session.UserID)
				c.Set("user_role", session.Role)
				
				// Refresh session TTL
				go store.Refresh(ctx, sessionID)
			}
		}
		
		c.Next()
	}
}

// Helper to require authentication
func RequireAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		_, exists := c.Get("session")
		if !exists {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "authentication required"})
			c.Abort()
			return
		}
		c.Next()
	}
}

func setupSessionServer(store *SessionStore) *gin.Engine {
	r := gin.New()
	r.Use(gin.Logger())
	r.Use(SessionMiddleware(store, "session_id"))
	
	// Login endpoint
	r.POST("/login", func(c *gin.Context) {
		var req struct {
			Username string `json:"username"`
			Password string `json:"password"`
		}
		
		if err := c.BindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
			return
		}
		
		// Validate credentials (simplified)
		if req.Username != "admin" || req.Password != "password" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
			return
		}
		
		// Create session
		session, err := store.Create(ctx, 1, req.Username, "admin")
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create session"})
			return
		}
		
		// Set cookie
		c.SetCookie("session_id", session.ID, int(30*time.Minute.Seconds()),
			"/", "", false, true)
		
		c.JSON(http.StatusOK, gin.H{
			"message":    "logged in successfully",
			"session_id": session.ID,
		})
	})
	
	// Protected endpoint
	r.GET("/profile", RequireAuth(), func(c *gin.Context) {
		session, _ := c.Get("session")
		s := session.(*Session)
		
		c.JSON(http.StatusOK, gin.H{
			"user_id":  s.UserID,
			"username": s.UserName,
			"role":     s.Role,
		})
	})
	
	// Logout
	r.POST("/logout", RequireAuth(), func(c *gin.Context) {
		session, _ := c.Get("session")
		s := session.(*Session)
		
		store.Delete(ctx, s.ID)
		c.SetCookie("session_id", "", -1, "/", "", false, true)
		
		c.JSON(http.StatusOK, gin.H{"message": "logged out"})
	})
	
	return r
}

func sessionWorkshopDemo() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer client.Close()
	
	store := NewSessionStore(client, 30*time.Minute)
	
	fmt.Println("=== Session Store Workshop ===")
	
	// Create session
	session, err := store.Create(ctx, 42, "alice", "user")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Created session: %s\n", session.ID[:16]+"...")
	
	// Get session
	retrieved, err := store.Get(ctx, session.ID)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Retrieved: user=%s, role=%s\n", retrieved.UserName, retrieved.Role)
	
	// Set session data
	store.SetData(ctx, session.ID, "preferred_language", "th")
	store.SetData(ctx, session.ID, "theme", "dark")
	
	// Get updated session
	updated, _ := store.Get(ctx, session.ID)
	fmt.Printf("Session data: %v\n", updated.Data)
	
	// Count sessions
	count, _ := store.Count(ctx)
	fmt.Printf("Active sessions: %d\n", count)
	
	// Delete session
	store.Delete(ctx, session.ID)
	
	count, _ = store.Count(ctx)
	fmt.Printf("After logout: %d sessions\n", count)
}

func main() {
	sessionWorkshopDemo()
}
```

---

## สรุป

ใน Part 40 เราได้เรียนรู้:

1. **การเชื่อมต่อ Redis**: ใช้ go-redis client พร้อม connection pool
2. **String Operations**: GET, SET, INCR, MGET, TTL management
3. **Lists**: Queue และ Stack patterns ด้วย LPUSH/RPUSH/LPOP/RPOP
4. **Sets**: Union, Intersection, Difference operations
5. **Sorted Sets**: Leaderboards, ranking systems
6. **Hashes**: Structured data storage
7. **Pub/Sub**: Real-time messaging between processes
8. **Redis Streams**: Ordered log of messages with consumer groups
9. **Distributed Locks**: Prevent race conditions across servers
10. **Redis as Cache**: JSON serialization/deserialization layer
11. **Workshop**: Complete Session Store implementation

---

## Resources

- [go-redis Documentation](https://redis.uptrace.dev/)
- [Redis Commands Reference](https://redis.io/commands/)
- [Redis Data Structures](https://redis.io/docs/data-types/)
- [Redis Streams](https://redis.io/docs/data-types/streams/)
- [Distributed Locks with Redis](https://redis.io/docs/manual/patterns/distributed-locks/)
