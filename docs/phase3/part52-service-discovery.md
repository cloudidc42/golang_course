# Part 52: Service Discovery

## เป้าหมายการเรียนรู้
- เข้าใจ Service Discovery concepts
- ใช้งาน Consul กับ Go
- ใช้งาน etcd service registry
- เปรียบเทียบ Client-side vs Server-side discovery
- ทำ Health checks
- Load balancing ด้วย service discovery
- DNS-based service discovery

---

## 1. Service Discovery Concepts

ใน Microservices แต่ละ service ต้องรู้ที่อยู่ของ service อื่น แต่ใน cloud environment ที่ service ถูก scale up/down หรือ restart บ่อย IP address เปลี่ยนตลอด Service Discovery แก้ปัญหานี้

### ประเภทของ Service Discovery

**1. Client-side Discovery**: Client ดึง service registry มาเองแล้วเลือก instance

```
Client --> Service Registry --> [IP list] --> Client เลือกเอง --> Service Instance
```

**2. Server-side Discovery**: Client ส่ง request ไปที่ Load Balancer แล้ว LB query registry

```
Client --> Load Balancer --> Service Registry --> Service Instance
```

```go
// discovery/types.go
package discovery

import (
	"context"
	"time"
)

// ServiceInstance แทน instance ของ service หนึ่ง
type ServiceInstance struct {
	ID       string            `json:"id"`
	Name     string            `json:"name"`
	Host     string            `json:"host"`
	Port     int               `json:"port"`
	Tags     []string          `json:"tags"`
	Meta     map[string]string `json:"meta"`
	Healthy  bool              `json:"healthy"`
	LastSeen time.Time         `json:"last_seen"`
}

func (s *ServiceInstance) Address() string {
	return fmt.Sprintf("%s:%d", s.Host, s.Port)
}

// Registry interface - สามารถ implement ด้วย Consul, etcd, หรือ in-memory
type Registry interface {
	Register(ctx context.Context, instance ServiceInstance) error
	Deregister(ctx context.Context, instanceID string) error
	Discover(ctx context.Context, serviceName string) ([]ServiceInstance, error)
	HealthCheck(ctx context.Context, instanceID string) error
}
```

```go
// discovery/inmemory.go
package discovery

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// InMemoryRegistry สำหรับ testing
type InMemoryRegistry struct {
	mu        sync.RWMutex
	instances map[string]ServiceInstance
}

func NewInMemoryRegistry() *InMemoryRegistry {
	return &InMemoryRegistry{
		instances: make(map[string]ServiceInstance),
	}
}

func (r *InMemoryRegistry) Register(ctx context.Context, instance ServiceInstance) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	instance.LastSeen = time.Now()
	instance.Healthy = true
	r.instances[instance.ID] = instance
	return nil
}

func (r *InMemoryRegistry) Deregister(ctx context.Context, instanceID string) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	delete(r.instances, instanceID)
	return nil
}

func (r *InMemoryRegistry) Discover(ctx context.Context, serviceName string) ([]ServiceInstance, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	var result []ServiceInstance
	for _, inst := range r.instances {
		if inst.Name == serviceName && inst.Healthy {
			result = append(result, inst)
		}
	}

	if len(result) == 0 {
		return nil, fmt.Errorf("no healthy instances found for service: %s", serviceName)
	}
	return result, nil
}

func (r *InMemoryRegistry) HealthCheck(ctx context.Context, instanceID string) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	inst, ok := r.instances[instanceID]
	if !ok {
		return fmt.Errorf("instance not found: %s", instanceID)
	}
	inst.LastSeen = time.Now()
	r.instances[instanceID] = inst
	return nil
}
```

---

## 2. Consul Integration

Consul เป็น popular service discovery tool ที่มี health checking, KV store, และ DNS

```go
// discovery/consul/registry.go
package consul

import (
	"context"
	"fmt"
	"log"
	"time"

	consulapi "github.com/hashicorp/consul/api"
)

type ConsulRegistry struct {
	client *consulapi.Client
}

func NewConsulRegistry(addr string) (*ConsulRegistry, error) {
	config := consulapi.DefaultConfig()
	config.Address = addr

	client, err := consulapi.NewClient(config)
	if err != nil {
		return nil, fmt.Errorf("creating consul client: %w", err)
	}

	return &ConsulRegistry{client: client}, nil
}

type RegistrationConfig struct {
	ID      string
	Name    string
	Host    string
	Port    int
	Tags    []string
	Meta    map[string]string
	HealthPath string
	HealthInterval string
	HealthTimeout  string
}

func (r *ConsulRegistry) Register(ctx context.Context, cfg RegistrationConfig) error {
	healthURL := fmt.Sprintf("http://%s:%d%s", cfg.Host, cfg.Port, cfg.HealthPath)
	if cfg.HealthPath == "" {
		healthURL = fmt.Sprintf("http://%s:%d/health", cfg.Host, cfg.Port)
	}

	interval := cfg.HealthInterval
	if interval == "" {
		interval = "10s"
	}
	timeout := cfg.HealthTimeout
	if timeout == "" {
		timeout = "5s"
	}

	registration := &consulapi.AgentServiceRegistration{
		ID:      cfg.ID,
		Name:    cfg.Name,
		Address: cfg.Host,
		Port:    cfg.Port,
		Tags:    cfg.Tags,
		Meta:    cfg.Meta,
		Check: &consulapi.AgentServiceCheck{
			HTTP:                           healthURL,
			Interval:                       interval,
			Timeout:                        timeout,
			DeregisterCriticalServiceAfter: "30s",
		},
	}

	if err := r.client.Agent().ServiceRegister(registration); err != nil {
		return fmt.Errorf("registering service: %w", err)
	}

	log.Printf("Registered service %s (%s) with Consul", cfg.Name, cfg.ID)
	return nil
}

func (r *ConsulRegistry) Deregister(ctx context.Context, instanceID string) error {
	if err := r.client.Agent().ServiceDeregister(instanceID); err != nil {
		return fmt.Errorf("deregistering service: %w", err)
	}
	log.Printf("Deregistered service %s from Consul", instanceID)
	return nil
}

func (r *ConsulRegistry) Discover(ctx context.Context, serviceName string) ([]ServiceInstance, error) {
	// ดึงเฉพาะ healthy instances
	entries, _, err := r.client.Health().Service(serviceName, "", true, nil)
	if err != nil {
		return nil, fmt.Errorf("discovering service %s: %w", serviceName, err)
	}

	var instances []ServiceInstance
	for _, entry := range entries {
		instances = append(instances, ServiceInstance{
			ID:      entry.Service.ID,
			Name:    entry.Service.Service,
			Host:    entry.Service.Address,
			Port:    entry.Service.Port,
			Tags:    entry.Service.Tags,
			Meta:    entry.Service.Meta,
			Healthy: true,
		})
	}

	if len(instances) == 0 {
		return nil, fmt.Errorf("no healthy instances for service: %s", serviceName)
	}
	return instances, nil
}

// WatchService ติดตามการเปลี่ยนแปลงของ service
func (r *ConsulRegistry) WatchService(ctx context.Context, serviceName string, onChange func([]ServiceInstance)) error {
	var lastIndex uint64

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		queryOpts := &consulapi.QueryOptions{
			WaitIndex: lastIndex,
			WaitTime:  30 * time.Second,
		}

		entries, meta, err := r.client.Health().Service(serviceName, "", true, queryOpts)
		if err != nil {
			log.Printf("Error watching service %s: %v", serviceName, err)
			time.Sleep(5 * time.Second)
			continue
		}

		if meta.LastIndex > lastIndex {
			lastIndex = meta.LastIndex
			var instances []ServiceInstance
			for _, entry := range entries {
				instances = append(instances, ServiceInstance{
					ID:      entry.Service.ID,
					Name:    entry.Service.Service,
					Host:    entry.Service.Address,
					Port:    entry.Service.Port,
					Healthy: true,
				})
			}
			onChange(instances)
		}
	}
}
```

```go
// discovery/consul/kv.go - Consul KV Store
package consul

import (
	"context"
	"fmt"

	consulapi "github.com/hashicorp/consul/api"
)

type KVStore struct {
	client *consulapi.Client
}

func NewKVStore(client *consulapi.Client) *KVStore {
	return &KVStore{client: client}
}

func (k *KVStore) Put(ctx context.Context, key, value string) error {
	_, err := k.client.KV().Put(&consulapi.KVPair{
		Key:   key,
		Value: []byte(value),
	}, nil)
	return err
}

func (k *KVStore) Get(ctx context.Context, key string) (string, error) {
	pair, _, err := k.client.KV().Get(key, nil)
	if err != nil {
		return "", err
	}
	if pair == nil {
		return "", fmt.Errorf("key not found: %s", key)
	}
	return string(pair.Value), nil
}

func (k *KVStore) Delete(ctx context.Context, key string) error {
	_, err := k.client.KV().Delete(key, nil)
	return err
}

func (k *KVStore) List(ctx context.Context, prefix string) (map[string]string, error) {
	pairs, _, err := k.client.KV().List(prefix, nil)
	if err != nil {
		return nil, err
	}

	result := make(map[string]string)
	for _, p := range pairs {
		result[p.Key] = string(p.Value)
	}
	return result, nil
}
```

---

## 3. etcd Service Registry

```go
// discovery/etcd/registry.go
package etcd

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

const (
	servicePrefix = "/services/"
	ttl           = 10 // seconds
)

type EtcdRegistry struct {
	client *clientv3.Client
}

func NewEtcdRegistry(endpoints []string) (*EtcdRegistry, error) {
	client, err := clientv3.New(clientv3.Config{
		Endpoints:   endpoints,
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		return nil, fmt.Errorf("creating etcd client: %w", err)
	}
	return &EtcdRegistry{client: client}, nil
}

func (r *EtcdRegistry) Register(ctx context.Context, instance ServiceInstance) error {
	// สร้าง lease สำหรับ TTL
	leaseResp, err := r.client.Grant(ctx, ttl)
	if err != nil {
		return fmt.Errorf("creating lease: %w", err)
	}

	key := fmt.Sprintf("%s%s/%s", servicePrefix, instance.Name, instance.ID)
	
	data, err := json.Marshal(instance)
	if err != nil {
		return fmt.Errorf("marshaling instance: %w", err)
	}

	// Register พร้อม lease
	_, err = r.client.Put(ctx, key, string(data), clientv3.WithLease(leaseResp.ID))
	if err != nil {
		return fmt.Errorf("registering instance: %w", err)
	}

	// Keep alive - ต่ออายุ lease อัตโนมัติ
	keepAliveCh, err := r.client.KeepAlive(ctx, leaseResp.ID)
	if err != nil {
		return fmt.Errorf("starting keepalive: %w", err)
	}

	// Monitor keepalive ใน goroutine
	go func() {
		for {
			select {
			case <-ctx.Done():
				log.Printf("Service %s context cancelled, stopping keepalive", instance.ID)
				return
			case resp, ok := <-keepAliveCh:
				if !ok {
					log.Printf("Keepalive channel closed for %s", instance.ID)
					return
				}
				if resp == nil {
					log.Printf("Keepalive failed for %s", instance.ID)
					return
				}
			}
		}
	}()

	log.Printf("Registered %s with etcd, lease TTL: %ds", instance.ID, ttl)
	return nil
}

func (r *EtcdRegistry) Deregister(ctx context.Context, name, id string) error {
	key := fmt.Sprintf("%s%s/%s", servicePrefix, name, id)
	_, err := r.client.Delete(ctx, key)
	return err
}

func (r *EtcdRegistry) Discover(ctx context.Context, serviceName string) ([]ServiceInstance, error) {
	prefix := fmt.Sprintf("%s%s/", servicePrefix, serviceName)
	
	resp, err := r.client.Get(ctx, prefix, clientv3.WithPrefix())
	if err != nil {
		return nil, fmt.Errorf("discovering service: %w", err)
	}

	var instances []ServiceInstance
	for _, kv := range resp.Kvs {
		var inst ServiceInstance
		if err := json.Unmarshal(kv.Value, &inst); err != nil {
			log.Printf("Error unmarshaling instance: %v", err)
			continue
		}
		instances = append(instances, inst)
	}

	if len(instances) == 0 {
		return nil, fmt.Errorf("no instances found for: %s", serviceName)
	}
	return instances, nil
}

// WatchService ติดตามการเปลี่ยนแปลงด้วย etcd watch
func (r *EtcdRegistry) WatchService(ctx context.Context, serviceName string, onChange func([]ServiceInstance)) {
	prefix := fmt.Sprintf("%s%s/", servicePrefix, serviceName)
	
	watchCh := r.client.Watch(ctx, prefix, clientv3.WithPrefix())
	
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			case resp := <-watchCh:
				if resp.Err() != nil {
					log.Printf("Watch error: %v", resp.Err())
					continue
				}
				// Rediscover service instances
				instances, err := r.Discover(ctx, serviceName)
				if err != nil {
					log.Printf("Error rediscovering: %v", err)
					continue
				}
				onChange(instances)
			}
		}
	}()
}

func (r *EtcdRegistry) Close() error {
	return r.client.Close()
}
```

---

## 4. Client-side Load Balancing

```go
// loadbalancer/balancer.go
package loadbalancer

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// LoadBalancer interface
type LoadBalancer interface {
	Pick(instances []ServiceInstance) (*ServiceInstance, error)
}

// ServiceInstance
type ServiceInstance struct {
	ID   string
	Host string
	Port int
	Weight int
}

func (s *ServiceInstance) Address() string {
	return fmt.Sprintf("%s:%d", s.Host, s.Port)
}

// Round Robin
type RoundRobinBalancer struct {
	counter uint64
}

func (b *RoundRobinBalancer) Pick(instances []ServiceInstance) (*ServiceInstance, error) {
	if len(instances) == 0 {
		return nil, fmt.Errorf("no instances available")
	}
	idx := atomic.AddUint64(&b.counter, 1) % uint64(len(instances))
	return &instances[idx], nil
}

// Random
type RandomBalancer struct{}

func (b *RandomBalancer) Pick(instances []ServiceInstance) (*ServiceInstance, error) {
	if len(instances) == 0 {
		return nil, fmt.Errorf("no instances available")
	}
	idx := rand.Intn(len(instances))
	return &instances[idx], nil
}

// Weighted Round Robin
type WeightedRoundRobinBalancer struct {
	mu      sync.Mutex
	current int
	weights []int
}

func (b *WeightedRoundRobinBalancer) Pick(instances []ServiceInstance) (*ServiceInstance, error) {
	if len(instances) == 0 {
		return nil, fmt.Errorf("no instances available")
	}

	b.mu.Lock()
	defer b.mu.Unlock()

	// สร้าง weighted list
	var weighted []int
	for i, inst := range instances {
		w := inst.Weight
		if w <= 0 {
			w = 1
		}
		for j := 0; j < w; j++ {
			weighted = append(weighted, i)
		}
	}

	idx := b.current % len(weighted)
	b.current++
	return &instances[weighted[idx]], nil
}

// Least Connections (ต้องการ connection tracking)
type LeastConnectionsBalancer struct {
	mu          sync.RWMutex
	connections map[string]int
}

func NewLeastConnectionsBalancer() *LeastConnectionsBalancer {
	return &LeastConnectionsBalancer{
		connections: make(map[string]int),
	}
}

func (b *LeastConnectionsBalancer) Pick(instances []ServiceInstance) (*ServiceInstance, error) {
	if len(instances) == 0 {
		return nil, fmt.Errorf("no instances available")
	}

	b.mu.RLock()
	defer b.mu.RUnlock()

	var best *ServiceInstance
	minConns := -1

	for i := range instances {
		conns := b.connections[instances[i].ID]
		if minConns == -1 || conns < minConns {
			minConns = conns
			best = &instances[i]
		}
	}
	return best, nil
}

func (b *LeastConnectionsBalancer) Increment(instanceID string) {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.connections[instanceID]++
}

func (b *LeastConnectionsBalancer) Decrement(instanceID string) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.connections[instanceID] > 0 {
		b.connections[instanceID]--
	}
}
```

---

## 5. Service Discovery Client

```go
// discovery/client.go
package discovery

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
)

// DiscoveryClient เป็น high-level client สำหรับ service discovery
type DiscoveryClient struct {
	registry  Registry
	balancer  LoadBalancer
	cache     map[string][]ServiceInstance
	cacheMu   sync.RWMutex
	cacheTTL  time.Duration
	cacheTime map[string]time.Time
}

type LoadBalancer interface {
	Pick(instances []ServiceInstance) (*ServiceInstance, error)
}

func NewDiscoveryClient(registry Registry, balancer LoadBalancer) *DiscoveryClient {
	return &DiscoveryClient{
		registry:  registry,
		balancer:  balancer,
		cache:     make(map[string][]ServiceInstance),
		cacheTTL:  30 * time.Second,
		cacheTime: make(map[string]time.Time),
	}
}

func (c *DiscoveryClient) GetInstance(ctx context.Context, serviceName string) (*ServiceInstance, error) {
	instances, err := c.getInstances(ctx, serviceName)
	if err != nil {
		return nil, err
	}
	return c.balancer.Pick(instances)
}

func (c *DiscoveryClient) getInstances(ctx context.Context, serviceName string) ([]ServiceInstance, error) {
	// Check cache
	c.cacheMu.RLock()
	if instances, ok := c.cache[serviceName]; ok {
		if time.Since(c.cacheTime[serviceName]) < c.cacheTTL {
			c.cacheMu.RUnlock()
			return instances, nil
		}
	}
	c.cacheMu.RUnlock()

	// Fetch from registry
	instances, err := c.registry.Discover(ctx, serviceName)
	if err != nil {
		// Return stale cache if available
		c.cacheMu.RLock()
		if stale, ok := c.cache[serviceName]; ok && len(stale) > 0 {
			c.cacheMu.RUnlock()
			log.Printf("Using stale cache for %s: %v", serviceName, err)
			return stale, nil
		}
		c.cacheMu.RUnlock()
		return nil, err
	}

	// Update cache
	c.cacheMu.Lock()
	c.cache[serviceName] = instances
	c.cacheTime[serviceName] = time.Now()
	c.cacheMu.Unlock()

	return instances, nil
}

// HTTPClient ที่ integrate กับ service discovery
type ServiceHTTPClient struct {
	discovery  *DiscoveryClient
	httpClient *http.Client
}

func NewServiceHTTPClient(discovery *DiscoveryClient) *ServiceHTTPClient {
	return &ServiceHTTPClient{
		discovery: discovery,
		httpClient: &http.Client{
			Timeout: 10 * time.Second,
		},
	}
}

func (c *ServiceHTTPClient) Get(ctx context.Context, serviceName, path string) (*http.Response, error) {
	instance, err := c.discovery.GetInstance(ctx, serviceName)
	if err != nil {
		return nil, fmt.Errorf("getting instance for %s: %w", serviceName, err)
	}

	url := fmt.Sprintf("http://%s%s", instance.Address(), path)
	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, err
	}

	return c.httpClient.Do(req)
}
```

---

## 6. Health Checks

```go
// health/checker.go
package health

import (
	"context"
	"database/sql"
	"fmt"
	"net/http"
	"time"
)

// HealthChecker interface
type HealthChecker interface {
	Check(ctx context.Context) error
	Name() string
}

// DatabaseHealthChecker
type DatabaseHealthChecker struct {
	db *sql.DB
}

func NewDatabaseChecker(db *sql.DB) *DatabaseHealthChecker {
	return &DatabaseHealthChecker{db: db}
}

func (c *DatabaseHealthChecker) Name() string { return "database" }

func (c *DatabaseHealthChecker) Check(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	return c.db.PingContext(ctx)
}

// HTTPHealthChecker ตรวจสอบ external service
type HTTPHealthChecker struct {
	name   string
	url    string
	client *http.Client
}

func NewHTTPChecker(name, url string) *HTTPHealthChecker {
	return &HTTPHealthChecker{
		name: name,
		url:  url,
		client: &http.Client{
			Timeout: 5 * time.Second,
		},
	}
}

func (c *HTTPHealthChecker) Name() string { return c.name }

func (c *HTTPHealthChecker) Check(ctx context.Context) error {
	req, err := http.NewRequestWithContext(ctx, "GET", c.url, nil)
	if err != nil {
		return fmt.Errorf("creating request: %w", err)
	}

	resp, err := c.client.Do(req)
	if err != nil {
		return fmt.Errorf("executing request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 500 {
		return fmt.Errorf("unhealthy status: %d", resp.StatusCode)
	}
	return nil
}

// CompositeHealthChecker รวม checker หลายตัว
type CompositeHealthChecker struct {
	checkers []HealthChecker
}

func NewCompositeChecker(checkers ...HealthChecker) *CompositeHealthChecker {
	return &CompositeHealthChecker{checkers: checkers}
}

func (c *CompositeHealthChecker) CheckAll(ctx context.Context) map[string]error {
	results := make(map[string]error)
	
	resultCh := make(chan struct {
		name string
		err  error
	}, len(c.checkers))

	for _, checker := range c.checkers {
		go func(ch HealthChecker) {
			err := ch.Check(ctx)
			resultCh <- struct {
				name string
				err  error
			}{ch.Name(), err}
		}(checker)
	}

	for range c.checkers {
		result := <-resultCh
		results[result.name] = result.err
	}
	return results
}
```

---

## 7. DNS-based Service Discovery

```go
// discovery/dns.go
package discovery

import (
	"context"
	"fmt"
	"log"
	"net"
	"time"
)

// DNSResolver ใช้ DNS สำหรับ service discovery
type DNSResolver struct {
	resolver *net.Resolver
	domain   string
}

func NewDNSResolver(domain string) *DNSResolver {
	return &DNSResolver{
		resolver: &net.Resolver{
			PreferGo: true,
			Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
				d := net.Dialer{
					Timeout: time.Millisecond * 5000,
				}
				return d.DialContext(ctx, "udp", "8.8.8.8:53")
			},
		},
		domain: domain,
	}
}

// LookupSRV ค้นหา service ผ่าน SRV record
func (r *DNSResolver) LookupSRV(ctx context.Context, service, proto string) ([]*ServiceInstance, error) {
	name := fmt.Sprintf("_%s._%s.%s", service, proto, r.domain)
	
	_, addrs, err := r.resolver.LookupSRV(ctx, service, proto, r.domain)
	if err != nil {
		return nil, fmt.Errorf("SRV lookup for %s: %w", name, err)
	}

	var instances []*ServiceInstance
	for _, addr := range addrs {
		instances = append(instances, &ServiceInstance{
			Name: service,
			Host: addr.Target,
			Port: int(addr.Port),
		})
	}
	return instances, nil
}

// LookupA ค้นหา IP address ผ่าน A record
func (r *DNSResolver) LookupA(ctx context.Context, hostname string) ([]string, error) {
	ips, err := r.resolver.LookupHost(ctx, hostname)
	if err != nil {
		return nil, fmt.Errorf("A lookup for %s: %w", hostname, err)
	}
	return ips, nil
}

// ServiceDiscovery ผ่าน Kubernetes DNS
// ใน Kubernetes: <service>.<namespace>.svc.cluster.local
type KubernetesDNS struct {
	namespace string
}

func NewKubernetesDNS(namespace string) *KubernetesDNS {
	return &KubernetesDNS{namespace: namespace}
}

func (k *KubernetesDNS) ServiceAddress(serviceName string) string {
	return fmt.Sprintf("%s.%s.svc.cluster.local", serviceName, k.namespace)
}

func (k *KubernetesDNS) Resolve(ctx context.Context, serviceName string) ([]string, error) {
	hostname := k.ServiceAddress(serviceName)
	ips, err := net.DefaultResolver.LookupHost(ctx, hostname)
	if err != nil {
		return nil, fmt.Errorf("resolving %s: %w", hostname, err)
	}
	log.Printf("Resolved %s to %v", hostname, ips)
	return ips, nil
}
```

---

## 8. Self-Registration Pattern

```go
// registration/self_register.go
package registration

import (
	"context"
	"fmt"
	"log"
	"net"
	"os"
	"time"
)

// SelfRegistrar handles automatic service registration
type SelfRegistrar struct {
	registry   ServiceRegistry
	instanceID string
	config     RegistrationConfig
	stopCh     chan struct{}
}

type ServiceRegistry interface {
	Register(ctx context.Context, cfg RegistrationConfig) error
	Deregister(ctx context.Context, id string) error
}

type RegistrationConfig struct {
	ID             string
	Name           string
	Host           string
	Port           int
	HealthEndpoint string
	Tags           []string
	Meta           map[string]string
}

func NewSelfRegistrar(registry ServiceRegistry, config RegistrationConfig) (*SelfRegistrar, error) {
	// Auto-detect host if not set
	if config.Host == "" {
		host, err := getLocalIP()
		if err != nil {
			return nil, fmt.Errorf("detecting local IP: %w", err)
		}
		config.Host = host
	}

	// Auto-generate ID if not set
	if config.ID == "" {
		hostname, _ := os.Hostname()
		config.ID = fmt.Sprintf("%s-%s-%d", config.Name, hostname, config.Port)
	}

	return &SelfRegistrar{
		registry:   registry,
		instanceID: config.ID,
		config:     config,
		stopCh:     make(chan struct{}),
	}, nil
}

func (r *SelfRegistrar) Start(ctx context.Context) error {
	if err := r.registry.Register(ctx, r.config); err != nil {
		return fmt.Errorf("initial registration: %w", err)
	}

	log.Printf("Service %s registered with ID %s at %s:%d",
		r.config.Name, r.config.ID, r.config.Host, r.config.Port)
	return nil
}

func (r *SelfRegistrar) Stop(ctx context.Context) error {
	close(r.stopCh)
	if err := r.registry.Deregister(ctx, r.instanceID); err != nil {
		return fmt.Errorf("deregistering: %w", err)
	}
	log.Printf("Service %s deregistered", r.instanceID)
	return nil
}

func getLocalIP() (string, error) {
	addrs, err := net.InterfaceAddrs()
	if err != nil {
		return "", err
	}

	for _, addr := range addrs {
		if ipNet, ok := addr.(*net.IPNet); ok && !ipNet.IP.IsLoopback() {
			if ipNet.IP.To4() != nil {
				return ipNet.IP.String(), nil
			}
		}
	}
	return "127.0.0.1", nil
}
```

---

## 9. Service Discovery Middleware

```go
// middleware/discovery.go
package middleware

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"time"
)

// ServiceProxy proxy requests ไปยัง discovered service instances
type ServiceProxy struct {
	discovery InstanceDiscoverer
	balancer  LoadBalancer
}

type InstanceDiscoverer interface {
	Discover(ctx context.Context, serviceName string) ([]ServiceInstance, error)
}

type LoadBalancer interface {
	Pick([]ServiceInstance) (*ServiceInstance, error)
}

type ServiceInstance struct {
	ID   string
	Host string
	Port int
}

func (s *ServiceInstance) URL() string {
	return fmt.Sprintf("http://%s:%d", s.Host, s.Port)
}

func NewServiceProxy(discovery InstanceDiscoverer, balancer LoadBalancer) *ServiceProxy {
	return &ServiceProxy{
		discovery: discovery,
		balancer:  balancer,
	}
}

func (p *ServiceProxy) ProxyTo(serviceName string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		instances, err := p.discovery.Discover(ctx, serviceName)
		if err != nil {
			log.Printf("Error discovering %s: %v", serviceName, err)
			http.Error(w, "Service unavailable", http.StatusServiceUnavailable)
			return
		}

		instance, err := p.balancer.Pick(instances)
		if err != nil {
			http.Error(w, "No instances available", http.StatusServiceUnavailable)
			return
		}

		targetURL, err := url.Parse(instance.URL())
		if err != nil {
			http.Error(w, "Invalid service URL", http.StatusInternalServerError)
			return
		}

		proxy := httputil.NewSingleHostReverseProxy(targetURL)
		proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
			log.Printf("Proxy error to %s: %v", instance.URL(), err)
			http.Error(w, "Bad gateway", http.StatusBadGateway)
		}

		// Add headers for tracing
		r.Header.Set("X-Forwarded-Instance", instance.ID)
		r.Header.Set("X-Proxy-Time", time.Now().Format(time.RFC3339))

		proxy.ServeHTTP(w, r)
	}
}
```

---

## Workshop: Service Discovery Demo

```go
// workshop/discovery_demo.go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"sync"
	"time"
)

// Simple in-memory registry demo
type Registry struct {
	mu        sync.RWMutex
	instances map[string][]Instance
}

type Instance struct {
	ID      string
	Host    string
	Port    int
	Healthy bool
}

func NewRegistry() *Registry {
	return &Registry{instances: make(map[string][]Instance)}
}

func (r *Registry) Register(service string, inst Instance) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.instances[service] = append(r.instances[service], inst)
	log.Printf("Registered: %s -> %s:%d", service, inst.Host, inst.Port)
}

func (r *Registry) Deregister(service, id string) {
	r.mu.Lock()
	defer r.mu.Unlock()
	instances := r.instances[service]
	for i, inst := range instances {
		if inst.ID == id {
			r.instances[service] = append(instances[:i], instances[i+1:]...)
			log.Printf("Deregistered: %s/%s", service, id)
			return
		}
	}
}

func (r *Registry) Discover(service string) []Instance {
	r.mu.RLock()
	defer r.mu.RUnlock()
	var healthy []Instance
	for _, inst := range r.instances[service] {
		if inst.Healthy {
			healthy = append(healthy, inst)
		}
	}
	return healthy
}

// Round-robin balancer
type RoundRobin struct {
	counter uint64
	mu      sync.Mutex
}

func (rb *RoundRobin) Pick(instances []Instance) *Instance {
	if len(instances) == 0 {
		return nil
	}
	rb.mu.Lock()
	idx := rb.counter % uint64(len(instances))
	rb.counter++
	rb.mu.Unlock()
	return &instances[idx]
}

func main() {
	registry := NewRegistry()
	balancer := &RoundRobin{}

	// Register 3 instances ของ user-service
	for i := 1; i <= 3; i++ {
		registry.Register("user-service", Instance{
			ID:      fmt.Sprintf("user-%d", i),
			Host:    "localhost",
			Port:    8080 + i,
			Healthy: true,
		})
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Simulate 10 requests
	for i := 0; i < 10; i++ {
		select {
		case <-ctx.Done():
			return
		default:
		}

		instances := registry.Discover("user-service")
		instance := balancer.Pick(instances)
		if instance == nil {
			log.Println("No instances available!")
			continue
		}
		log.Printf("Request %d -> %s:%d", i+1, instance.Host, instance.Port)
		time.Sleep(50 * time.Millisecond)
	}

	// Simulate instance failure
	registry.mu.Lock()
	for i := range registry.instances["user-service"] {
		if registry.instances["user-service"][i].ID == "user-2" {
			registry.instances["user-service"][i].Healthy = false
			log.Println("Instance user-2 marked unhealthy!")
		}
	}
	registry.mu.Unlock()

	// Continue requests after failure
	for i := 0; i < 5; i++ {
		instances := registry.Discover("user-service")
		instance := balancer.Pick(instances)
		if instance != nil {
			log.Printf("Post-failure request %d -> %s:%d", i+1, instance.Host, instance.Port)
		}
		time.Sleep(50 * time.Millisecond)
	}

	// Dynamic scaling
	log.Println("\n--- Scaling up ---")
	for i := 4; i <= 6; i++ {
		registry.Register("user-service", Instance{
			ID:      fmt.Sprintf("user-%d", i),
			Host:    "localhost",
			Port:    8080 + i,
			Healthy: true,
		})
		time.Sleep(100 * time.Millisecond)
	}

	instances := registry.Discover("user-service")
	log.Printf("Available instances after scale-up: %d", len(instances))

	_ = rand.Intn // suppress unused warning
}
```

---

## สรุป

| Pattern | เมื่อไหร่ใช้ | ข้อดี | ข้อเสีย |
|---------|------------|-------|--------|
| In-memory Registry | Development/Testing | ง่าย, เร็ว | ไม่ persistent |
| Consul | Production | Full-featured, health checks | ต้อง run Consul cluster |
| etcd | Production (Kubernetes) | ถูก integrate กับ k8s | Complex |
| DNS | Simple scenarios | ไม่ต้อง external tool | Limited features |
| Client-side | ต้องการ fine-grained control | Flexible | Client ต้อง implement LB |
| Server-side | Simple client | Client เรียบง่าย | Need LB infrastructure |

---

**ต่อไป**: Part 53 - API Gateway
