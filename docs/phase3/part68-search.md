# Part 68: Full-Text Search

## เป้าหมายการเรียนรู้
- เข้าใจแนวคิด Full-text Search
- ใช้งาน Elasticsearch กับ Go
- ใช้งาน Meilisearch
- Full-text search ด้วย PostgreSQL
- Indexing strategies, Search ranking, Faceted search

---

## 1. Full-Text Search คืออะไร?

Full-text search คือการค้นหาเนื้อหาในเอกสารโดยพิจารณาความหมายและบริบท ไม่ใช่แค่การ match string ตรงๆ

```
LIKE '%golang%'     → ช้า, ไม่ flexible
Full-text search    → เร็ว, เข้าใจภาษา, ranking
```

### แนวคิดหลัก

```
Document: "Go is an open source programming language"
     ↓ Tokenize
Tokens: ["go", "open", "source", "programming", "language"]
     ↓ Normalize (lowercase, stemming, stop words)
Index Terms: ["go", "open", "sourc", "program", "languag"]
     ↓ Inverted Index
"go"       → [doc1, doc5, doc12]
"program"  → [doc1, doc3, doc8]
```

---

## 2. Elasticsearch กับ Go

### Setup

```go
// go.mod
// github.com/elastic/go-elasticsearch/v8 v8.x.x

// elasticsearch/client.go
package search

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "strings"
    
    "github.com/elastic/go-elasticsearch/v8"
    "github.com/elastic/go-elasticsearch/v8/esapi"
)

type ESClient struct {
    client *elasticsearch.Client
}

func NewESClient(addresses []string, username, password string) (*ESClient, error) {
    cfg := elasticsearch.Config{
        Addresses: addresses,
        Username:  username,
        Password:  password,
    }
    
    client, err := elasticsearch.NewClient(cfg)
    if err != nil {
        return nil, fmt.Errorf("failed to create ES client: %w", err)
    }
    
    // ทดสอบ connection
    res, err := client.Info()
    if err != nil {
        return nil, fmt.Errorf("failed to connect to ES: %w", err)
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return nil, fmt.Errorf("ES error: %s", res.Status())
    }
    
    log.Println("Connected to Elasticsearch")
    return &ESClient{client: client}, nil
}
```

### Index Management

```go
// elasticsearch/index.go
package search

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
)

// Mapping สำหรับ Product index
var productMapping = map[string]interface{}{
    "mappings": map[string]interface{}{
        "properties": map[string]interface{}{
            "id": map[string]interface{}{
                "type": "keyword",
            },
            "name": map[string]interface{}{
                "type":     "text",
                "analyzer": "thai_analyzer",
                "fields": map[string]interface{}{
                    "keyword": map[string]interface{}{
                        "type": "keyword",
                    },
                    "suggest": map[string]interface{}{
                        "type":     "completion",
                        "analyzer": "simple",
                    },
                },
            },
            "description": map[string]interface{}{
                "type":     "text",
                "analyzer": "thai_analyzer",
            },
            "category": map[string]interface{}{
                "type": "keyword",
            },
            "price": map[string]interface{}{
                "type": "float",
            },
            "tags": map[string]interface{}{
                "type": "keyword",
            },
            "rating": map[string]interface{}{
                "type": "float",
            },
            "stock": map[string]interface{}{
                "type": "integer",
            },
            "created_at": map[string]interface{}{
                "type":   "date",
                "format": "strict_date_optional_time",
            },
        },
    },
    "settings": map[string]interface{}{
        "number_of_shards":   3,
        "number_of_replicas": 1,
        "analysis": map[string]interface{}{
            "analyzer": map[string]interface{}{
                "thai_analyzer": map[string]interface{}{
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter":    []string{"lowercase", "thai_stop"},
                },
            },
            "filter": map[string]interface{}{
                "thai_stop": map[string]interface{}{
                    "type":      "stop",
                    "stopwords": "_thai_",
                },
            },
        },
    },
}

func (c *ESClient) CreateIndex(ctx context.Context, indexName string) error {
    body, err := json.Marshal(productMapping)
    if err != nil {
        return err
    }
    
    req := esapi.IndicesCreateRequest{
        Index: indexName,
        Body:  bytes.NewReader(body),
    }
    
    res, err := req.Do(ctx, c.client)
    if err != nil {
        return fmt.Errorf("failed to create index: %w", err)
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("ES error creating index: %s", res.String())
    }
    
    return nil
}

func (c *ESClient) DeleteIndex(ctx context.Context, indexName string) error {
    res, err := c.client.Indices.Delete([]string{indexName},
        c.client.Indices.Delete.WithContext(ctx),
    )
    if err != nil {
        return err
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("failed to delete index: %s", res.Status())
    }
    
    return nil
}
```

### Document Operations

```go
// elasticsearch/document.go
package search

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "strings"
    "time"
)

type Product struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Category    string    `json:"category"`
    Price       float64   `json:"price"`
    Tags        []string  `json:"tags"`
    Rating      float64   `json:"rating"`
    Stock       int       `json:"stock"`
    CreatedAt   time.Time `json:"created_at"`
}

// Index เอกสารเดียว
func (c *ESClient) IndexProduct(ctx context.Context, index string, product *Product) error {
    body, err := json.Marshal(product)
    if err != nil {
        return err
    }
    
    req := esapi.IndexRequest{
        Index:      index,
        DocumentID: product.ID,
        Body:       bytes.NewReader(body),
        Refresh:    "true", // อัปเดต index ทันที (ใช้ใน dev เท่านั้น)
    }
    
    res, err := req.Do(ctx, c.client)
    if err != nil {
        return fmt.Errorf("failed to index product: %w", err)
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("ES indexing error: %s", res.String())
    }
    
    return nil
}

// Bulk Indexing - เร็วกว่ามากสำหรับข้อมูลเยอะ
func (c *ESClient) BulkIndexProducts(ctx context.Context, index string, products []Product) error {
    var buf bytes.Buffer
    
    for _, p := range products {
        // Bulk action header
        meta := map[string]interface{}{
            "index": map[string]interface{}{
                "_index": index,
                "_id":    p.ID,
            },
        }
        metaLine, _ := json.Marshal(meta)
        buf.Write(metaLine)
        buf.WriteByte('\n')
        
        // Document body
        docLine, _ := json.Marshal(p)
        buf.Write(docLine)
        buf.WriteByte('\n')
    }
    
    res, err := c.client.Bulk(bytes.NewReader(buf.Bytes()),
        c.client.Bulk.WithContext(ctx),
        c.client.Bulk.WithIndex(index),
    )
    if err != nil {
        return fmt.Errorf("bulk index failed: %w", err)
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("bulk error: %s", res.String())
    }
    
    // ตรวจสอบ partial errors
    var bulkResponse struct {
        Errors bool `json:"errors"`
        Items  []map[string]map[string]interface{} `json:"items"`
    }
    
    json.NewDecoder(res.Body).Decode(&bulkResponse)
    
    if bulkResponse.Errors {
        var errs []string
        for _, item := range bulkResponse.Items {
            if action, ok := item["index"]; ok {
                if errInfo, ok := action["error"]; ok {
                    errs = append(errs, fmt.Sprintf("%v", errInfo))
                }
            }
        }
        return fmt.Errorf("bulk errors: %s", strings.Join(errs, "; "))
    }
    
    return nil
}

// Update document
func (c *ESClient) UpdateProduct(ctx context.Context, index, id string, updates map[string]interface{}) error {
    body, _ := json.Marshal(map[string]interface{}{
        "doc": updates,
    })
    
    res, err := c.client.Update(index, id,
        bytes.NewReader(body),
        c.client.Update.WithContext(ctx),
    )
    if err != nil {
        return err
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("update failed: %s", res.Status())
    }
    
    return nil
}
```

### Search Operations

```go
// elasticsearch/search.go
package search

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
)

type SearchParams struct {
    Query      string
    Categories []string
    MinPrice   float64
    MaxPrice   float64
    MinRating  float64
    Tags       []string
    SortBy     string // "relevance", "price_asc", "price_desc", "rating"
    Page       int
    PageSize   int
}

type SearchResult struct {
    Total    int64     `json:"total"`
    Products []Product `json:"products"`
    Facets   Facets    `json:"facets"`
}

type Facets struct {
    Categories []FacetBucket `json:"categories"`
    PriceRanges []FacetBucket `json:"price_ranges"`
    Tags       []FacetBucket `json:"tags"`
}

type FacetBucket struct {
    Key   string `json:"key"`
    Count int64  `json:"count"`
}

func (c *ESClient) SearchProducts(ctx context.Context, index string, params SearchParams) (*SearchResult, error) {
    // Build query
    query := buildSearchQuery(params)
    
    body, err := json.Marshal(query)
    if err != nil {
        return nil, err
    }
    
    from := (params.Page - 1) * params.PageSize
    
    res, err := c.client.Search(
        c.client.Search.WithContext(ctx),
        c.client.Search.WithIndex(index),
        c.client.Search.WithBody(bytes.NewReader(body)),
        c.client.Search.WithFrom(from),
        c.client.Search.WithSize(params.PageSize),
        c.client.Search.WithTrackTotalHits(true),
    )
    if err != nil {
        return nil, fmt.Errorf("search failed: %w", err)
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return nil, fmt.Errorf("ES search error: %s", res.String())
    }
    
    return parseSearchResponse(res.Body)
}

func buildSearchQuery(params SearchParams) map[string]interface{} {
    query := map[string]interface{}{
        "query": map[string]interface{}{},
    }
    
    var mustClauses []interface{}
    var filterClauses []interface{}
    
    // Full-text search
    if params.Query != "" {
        mustClauses = append(mustClauses, map[string]interface{}{
            "multi_match": map[string]interface{}{
                "query":  params.Query,
                "fields": []string{"name^3", "description", "tags"},
                "type":   "best_fields",
                "fuzziness": "AUTO",
            },
        })
    }
    
    // Category filter
    if len(params.Categories) > 0 {
        filterClauses = append(filterClauses, map[string]interface{}{
            "terms": map[string]interface{}{
                "category": params.Categories,
            },
        })
    }
    
    // Price range filter
    if params.MinPrice > 0 || params.MaxPrice > 0 {
        priceRange := map[string]interface{}{}
        if params.MinPrice > 0 {
            priceRange["gte"] = params.MinPrice
        }
        if params.MaxPrice > 0 {
            priceRange["lte"] = params.MaxPrice
        }
        filterClauses = append(filterClauses, map[string]interface{}{
            "range": map[string]interface{}{
                "price": priceRange,
            },
        })
    }
    
    // Rating filter
    if params.MinRating > 0 {
        filterClauses = append(filterClauses, map[string]interface{}{
            "range": map[string]interface{}{
                "rating": map[string]interface{}{
                    "gte": params.MinRating,
                },
            },
        })
    }
    
    // Tags filter
    if len(params.Tags) > 0 {
        filterClauses = append(filterClauses, map[string]interface{}{
            "terms": map[string]interface{}{
                "tags": params.Tags,
            },
        })
    }
    
    // Build bool query
    boolQuery := map[string]interface{}{}
    if len(mustClauses) > 0 {
        boolQuery["must"] = mustClauses
    } else {
        boolQuery["must"] = []interface{}{map[string]interface{}{"match_all": map[string]interface{}{}}}
    }
    if len(filterClauses) > 0 {
        boolQuery["filter"] = filterClauses
    }
    
    query["query"] = map[string]interface{}{
        "bool": boolQuery,
    }
    
    // Sorting
    switch params.SortBy {
    case "price_asc":
        query["sort"] = []interface{}{map[string]interface{}{"price": "asc"}}
    case "price_desc":
        query["sort"] = []interface{}{map[string]interface{}{"price": "desc"}}
    case "rating":
        query["sort"] = []interface{}{map[string]interface{}{"rating": "desc"}}
    default:
        // relevance (default ES behavior)
    }
    
    // Aggregations สำหรับ facets
    query["aggs"] = map[string]interface{}{
        "categories": map[string]interface{}{
            "terms": map[string]interface{}{
                "field": "category",
                "size":  20,
            },
        },
        "price_ranges": map[string]interface{}{
            "range": map[string]interface{}{
                "field": "price",
                "ranges": []interface{}{
                    map[string]interface{}{"key": "under_500", "to": 500},
                    map[string]interface{}{"key": "500_1000", "from": 500, "to": 1000},
                    map[string]interface{}{"key": "over_1000", "from": 1000},
                },
            },
        },
        "tags": map[string]interface{}{
            "terms": map[string]interface{}{
                "field": "tags",
                "size":  30,
            },
        },
    }
    
    // Highlight
    if params.Query != "" {
        query["highlight"] = map[string]interface{}{
            "fields": map[string]interface{}{
                "name":        map[string]interface{}{},
                "description": map[string]interface{}{"number_of_fragments": 3},
            },
        }
    }
    
    return query
}

// Autocomplete / Suggest
func (c *ESClient) Suggest(ctx context.Context, index, prefix string) ([]string, error) {
    query := map[string]interface{}{
        "suggest": map[string]interface{}{
            "product_suggest": map[string]interface{}{
                "prefix": prefix,
                "completion": map[string]interface{}{
                    "field": "name.suggest",
                    "size":  10,
                    "fuzzy": map[string]interface{}{
                        "fuzziness": 1,
                    },
                },
            },
        },
    }
    
    body, _ := json.Marshal(query)
    
    res, err := c.client.Search(
        c.client.Search.WithContext(ctx),
        c.client.Search.WithIndex(index),
        c.client.Search.WithBody(bytes.NewReader(body)),
        c.client.Search.WithSource("false"),
    )
    if err != nil {
        return nil, err
    }
    defer res.Body.Close()
    
    var result struct {
        Suggest map[string][]struct {
            Options []struct {
                Text string `json:"text"`
            } `json:"options"`
        } `json:"suggest"`
    }
    
    json.NewDecoder(res.Body).Decode(&result)
    
    var suggestions []string
    for _, opts := range result.Suggest["product_suggest"] {
        for _, opt := range opts.Options {
            suggestions = append(suggestions, opt.Text)
        }
    }
    
    return suggestions, nil
}
```

---

## 3. Meilisearch

```go
// meilisearch/client.go
package search

import (
    "context"
    "fmt"
    
    meilisearch "github.com/meilisearch/meilisearch-go"
)

type MeiliClient struct {
    client *meilisearch.Client
}

func NewMeiliClient(host, apiKey string) *MeiliClient {
    client := meilisearch.NewClient(meilisearch.ClientConfig{
        Host:   host,
        APIKey: apiKey,
    })
    
    return &MeiliClient{client: client}
}

func (m *MeiliClient) SetupIndex(indexName string) error {
    // สร้าง index
    task, err := m.client.CreateIndex(&meilisearch.IndexConfig{
        Uid:        indexName,
        PrimaryKey: "id",
    })
    if err != nil {
        return fmt.Errorf("failed to create index: %w", err)
    }
    
    // รอให้ task เสร็จ
    m.client.WaitForTask(task.TaskUID)
    
    index := m.client.Index(indexName)
    
    // กำหนด searchable attributes
    _, err = index.UpdateSearchableAttributes(&[]string{
        "name", "description", "category", "tags",
    })
    if err != nil {
        return err
    }
    
    // กำหนด filterable attributes สำหรับ faceting
    _, err = index.UpdateFilterableAttributes(&[]string{
        "category", "price", "tags", "rating",
    })
    if err != nil {
        return err
    }
    
    // กำหนด sortable attributes
    _, err = index.UpdateSortableAttributes(&[]string{
        "price", "rating", "created_at",
    })
    
    return err
}

func (m *MeiliClient) IndexProducts(indexName string, products []Product) error {
    index := m.client.Index(indexName)
    
    task, err := index.AddDocuments(products, "id")
    if err != nil {
        return fmt.Errorf("failed to add documents: %w", err)
    }
    
    // รอ indexing เสร็จ
    finalTask, err := m.client.WaitForTask(task.TaskUID)
    if err != nil {
        return err
    }
    
    if finalTask.Status == meilisearch.TaskStatusFailed {
        return fmt.Errorf("indexing failed: %s", finalTask.Error.Message)
    }
    
    return nil
}

type MeiliSearchParams struct {
    Query        string
    Filters      string   // Meilisearch filter syntax
    Sort         []string // ["price:asc", "rating:desc"]
    Facets       []string
    Page         int
    HitsPerPage  int
}

func (m *MeiliClient) Search(indexName string, params MeiliSearchParams) (*meilisearch.SearchResponse, error) {
    index := m.client.Index(indexName)
    
    searchReq := &meilisearch.SearchRequest{
        Query:               params.Query,
        Filter:              params.Filters,
        Sort:                params.Sort,
        Facets:              params.Facets,
        Page:                int64(params.Page),
        HitsPerPage:         int64(params.HitsPerPage),
        AttributesToHighlight: []string{"name", "description"},
        HighlightPreTag:     "<mark>",
        HighlightPostTag:    "</mark>",
        ShowMatchesPosition: true,
    }
    
    result, err := index.Search(params.Query, searchReq)
    if err != nil {
        return nil, fmt.Errorf("search failed: %w", err)
    }
    
    return result, nil
}

// ตัวอย่างการใช้งาน
func ExampleMeiliSearch() {
    client := NewMeiliClient("http://localhost:7700", "masterKey")
    
    // Setup
    client.SetupIndex("products")
    
    // Index data
    products := []Product{
        {ID: "1", Name: "MacBook Pro", Category: "laptop", Price: 59900, Rating: 4.8},
        {ID: "2", Name: "iPhone 15", Category: "phone", Price: 35900, Rating: 4.7},
    }
    client.IndexProducts("products", products)
    
    // Search
    result, _ := client.Search("products", MeiliSearchParams{
        Query:       "mac",
        Filters:     "price < 60000 AND category = 'laptop'",
        Sort:        []string{"rating:desc"},
        Facets:      []string{"category", "price"},
        HitsPerPage: 10,
        Page:        1,
    })
    
    fmt.Printf("Found %d results\n", result.EstimatedTotalHits)
}
```

---

## 4. PostgreSQL Full-Text Search

```go
// postgres_fts/search.go
package search

import (
    "context"
    "database/sql"
    "fmt"
    "strings"
)

// Setup
const setupSQL = `
-- เพิ่ม tsvector column
ALTER TABLE products ADD COLUMN IF NOT EXISTS search_vector tsvector;

-- สร้าง trigger สำหรับ auto-update search_vector
CREATE OR REPLACE FUNCTION update_product_search_vector() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(array_to_string(NEW.tags, ' '), '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_trigger
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_product_search_vector();

-- สร้าง GIN index สำหรับ full-text search
CREATE INDEX IF NOT EXISTS idx_products_search_vector ON products USING GIN(search_vector);

-- Update existing rows
UPDATE products SET search_vector = 
    setweight(to_tsvector('english', COALESCE(name, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(description, '')), 'B');
`

type PGFTSRepository struct {
    db *sql.DB
}

func (r *PGFTSRepository) Setup(ctx context.Context) error {
    _, err := r.db.ExecContext(ctx, setupSQL)
    return err
}

type FTSSearchParams struct {
    Query      string
    Category   string
    MinPrice   float64
    MaxPrice   float64
    Limit      int
    Offset     int
}

func (r *PGFTSRepository) Search(ctx context.Context, params FTSSearchParams) ([]Product, error) {
    var conditions []string
    var args []interface{}
    argIdx := 1
    
    // Full-text search condition
    if params.Query != "" {
        // plainto_tsquery: แปลง query ธรรมดาเป็น tsquery
        conditions = append(conditions, fmt.Sprintf(
            "search_vector @@ plainto_tsquery('english', $%d)", argIdx,
        ))
        args = append(args, params.Query)
        argIdx++
    }
    
    // Category filter
    if params.Category != "" {
        conditions = append(conditions, fmt.Sprintf("category = $%d", argIdx))
        args = append(args, params.Category)
        argIdx++
    }
    
    // Price range
    if params.MinPrice > 0 {
        conditions = append(conditions, fmt.Sprintf("price >= $%d", argIdx))
        args = append(args, params.MinPrice)
        argIdx++
    }
    if params.MaxPrice > 0 {
        conditions = append(conditions, fmt.Sprintf("price <= $%d", argIdx))
        args = append(args, params.MaxPrice)
        argIdx++
    }
    
    whereClause := ""
    if len(conditions) > 0 {
        whereClause = "WHERE " + strings.Join(conditions, " AND ")
    }
    
    // Order by relevance (ts_rank) เมื่อมี search query
    orderClause := "ORDER BY created_at DESC"
    if params.Query != "" {
        orderClause = fmt.Sprintf(
            "ORDER BY ts_rank(search_vector, plainto_tsquery('english', $1)) DESC, created_at DESC",
        )
    }
    
    query := fmt.Sprintf(`
        SELECT 
            id, 
            name, 
            description, 
            category, 
            price, 
            rating,
            ts_headline('english', description, plainto_tsquery('english', $1),
                'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15') AS excerpt
        FROM products
        %s
        %s
        LIMIT $%d OFFSET $%d
    `, whereClause, orderClause, argIdx, argIdx+1)
    
    args = append(args, params.Limit, params.Offset)
    
    rows, err := r.db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, fmt.Errorf("search query failed: %w", err)
    }
    defer rows.Close()
    
    var products []Product
    for rows.Next() {
        var p Product
        var excerpt string
        err := rows.Scan(
            &p.ID, &p.Name, &p.Description, 
            &p.Category, &p.Price, &p.Rating, &excerpt,
        )
        if err != nil {
            return nil, err
        }
        products = append(products, p)
    }
    
    return products, rows.Err()
}

// Fuzzy Search กับ trigram extension
const trigramSetup = `
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX IF NOT EXISTS idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
`

func (r *PGFTSRepository) FuzzySearch(ctx context.Context, name string, threshold float64) ([]Product, error) {
    query := `
        SELECT id, name, description, price, 
               similarity(name, $1) AS sim
        FROM products
        WHERE similarity(name, $1) > $2
        ORDER BY sim DESC
        LIMIT 20
    `
    
    rows, err := r.db.QueryContext(ctx, query, name, threshold)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var products []Product
    for rows.Next() {
        var p Product
        var sim float64
        rows.Scan(&p.ID, &p.Name, &p.Description, &p.Price, &sim)
        products = append(products, p)
    }
    
    return products, nil
}
```

---

## 5. Search Ranking

```go
// ranking/scorer.go
package ranking

import (
    "math"
    "strings"
    "time"
)

type SearchResult struct {
    Product   Product
    Score     float64
    Factors   map[string]float64
}

type RankingConfig struct {
    TextRelevanceWeight float64
    PopularityWeight    float64
    RecencyWeight       float64
    RatingWeight        float64
}

func DefaultConfig() RankingConfig {
    return RankingConfig{
        TextRelevanceWeight: 0.40,
        PopularityWeight:    0.25,
        RecencyWeight:       0.20,
        RatingWeight:        0.15,
    }
}

type Ranker struct {
    config RankingConfig
}

func (r *Ranker) Score(query string, product Product, esScore float64) SearchResult {
    factors := make(map[string]float64)
    
    // 1. Text relevance (จาก ES score, normalize 0-1)
    textScore := math.Min(esScore/10.0, 1.0)
    factors["text_relevance"] = textScore
    
    // 2. Popularity score (จาก view count, purchase count)
    popularityScore := math.Log1p(float64(product.ViewCount)) / math.Log1p(10000)
    factors["popularity"] = math.Min(popularityScore, 1.0)
    
    // 3. Recency score (ใหม่กว่าได้ score สูงกว่า)
    daysSinceCreation := time.Since(product.CreatedAt).Hours() / 24
    recencyScore := math.Exp(-daysSinceCreation / 365)
    factors["recency"] = recencyScore
    
    // 4. Rating score
    ratingScore := product.Rating / 5.0
    factors["rating"] = ratingScore
    
    // Weighted sum
    totalScore := 
        textScore     * r.config.TextRelevanceWeight +
        popularityScore * r.config.PopularityWeight +
        recencyScore  * r.config.RecencyWeight +
        ratingScore   * r.config.RatingWeight
    
    // Boost exact matches
    if strings.EqualFold(product.Name, query) {
        totalScore *= 1.5
        factors["exact_match_boost"] = 1.5
    }
    
    // Boost in-stock items
    if product.Stock > 0 {
        totalScore *= 1.1
        factors["in_stock_boost"] = 1.1
    }
    
    return SearchResult{
        Product: product,
        Score:   totalScore,
        Factors: factors,
    }
}
```

---

## 6. Faceted Search

```go
// facets/facets.go
package facets

import (
    "context"
    "database/sql"
    "fmt"
)

type FacetValue struct {
    Value string `json:"value"`
    Count int    `json:"count"`
    Selected bool `json:"selected"`
}

type Facet struct {
    Name   string       `json:"name"`
    Values []FacetValue `json:"values"`
}

type FacetedSearchParams struct {
    Query          string
    SelectedFacets map[string][]string // facet name -> selected values
    Page           int
    PageSize       int
}

type FacetedSearchResult struct {
    Products []Product
    Total    int
    Facets   []Facet
}

type FacetedSearchRepo struct {
    db *sql.DB
}

func (r *FacetedSearchRepo) Search(ctx context.Context, params FacetedSearchParams) (*FacetedSearchResult, error) {
    // Build base WHERE conditions
    whereConditions := []string{"deleted_at IS NULL"}
    var args []interface{}
    argIdx := 1
    
    if params.Query != "" {
        whereConditions = append(whereConditions, fmt.Sprintf(
            "search_vector @@ plainto_tsquery('english', $%d)", argIdx,
        ))
        args = append(args, params.Query)
        argIdx++
    }
    
    // Apply selected facets
    for facetName, selectedValues := range params.SelectedFacets {
        if len(selectedValues) == 0 {
            continue
        }
        
        placeholders := make([]string, len(selectedValues))
        for i, v := range selectedValues {
            placeholders[i] = fmt.Sprintf("$%d", argIdx)
            args = append(args, v)
            argIdx++
        }
        
        whereConditions = append(whereConditions, fmt.Sprintf(
            "%s IN (%s)", facetName, strings.Join(placeholders, ","),
        ))
    }
    
    whereClause := "WHERE " + strings.Join(whereConditions, " AND ")
    
    // Count total
    var total int
    countQuery := fmt.Sprintf("SELECT COUNT(*) FROM products %s", whereClause)
    r.db.QueryRowContext(ctx, countQuery, args...).Scan(&total)
    
    // Get products
    productQuery := fmt.Sprintf(`
        SELECT id, name, category, price, rating
        FROM products %s
        ORDER BY created_at DESC
        LIMIT $%d OFFSET $%d
    `, whereClause, argIdx, argIdx+1)
    
    productArgs := append(args, params.PageSize, (params.Page-1)*params.PageSize)
    rows, err := r.db.QueryContext(ctx, productQuery, productArgs...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var products []Product
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name, &p.Category, &p.Price, &p.Rating)
        products = append(products, p)
    }
    
    // Get facets (ต้องใช้ WHERE ที่ไม่รวม facet นั้น เพื่อแสดง count ที่ถูกต้อง)
    facets, err := r.getFacets(ctx, params)
    if err != nil {
        return nil, err
    }
    
    return &FacetedSearchResult{
        Products: products,
        Total:    total,
        Facets:   facets,
    }, nil
}

func (r *FacetedSearchRepo) getFacets(ctx context.Context, params FacetedSearchParams) ([]Facet, error) {
    var baseArgs []interface{}
    var baseConditions []string{"deleted_at IS NULL"}
    argIdx := 1
    
    if params.Query != "" {
        baseConditions = append(baseConditions, fmt.Sprintf(
            "search_vector @@ plainto_tsquery('english', $%d)", argIdx,
        ))
        baseArgs = append(baseArgs, params.Query)
        argIdx++
    }
    
    baseWhere := "WHERE " + strings.Join(baseConditions, " AND ")
    
    // Category facet
    catQuery := fmt.Sprintf(`
        SELECT category, COUNT(*) as count
        FROM products %s
        GROUP BY category
        ORDER BY count DESC
        LIMIT 20
    `, baseWhere)
    
    catRows, err := r.db.QueryContext(ctx, catQuery, baseArgs...)
    if err != nil {
        return nil, err
    }
    defer catRows.Close()
    
    var categoryFacet Facet
    categoryFacet.Name = "category"
    
    selectedCategories := params.SelectedFacets["category"]
    selectedMap := make(map[string]bool)
    for _, c := range selectedCategories {
        selectedMap[c] = true
    }
    
    for catRows.Next() {
        var v FacetValue
        catRows.Scan(&v.Value, &v.Count)
        v.Selected = selectedMap[v.Value]
        categoryFacet.Values = append(categoryFacet.Values, v)
    }
    
    return []Facet{categoryFacet}, nil
}
```

---

## Workshop: Product Search System

```go
// workshop/search_service.go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "strconv"
)

type SearchService struct {
    es     *ESClient
    pgRepo *PGFTSRepository
}

func NewSearchService(es *ESClient, pg *PGFTSRepository) *SearchService {
    return &SearchService{es: es, pgRepo: pg}
}

func (s *SearchService) SearchHandler(w http.ResponseWriter, r *http.Request) {
    q := r.URL.Query()
    
    params := SearchParams{
        Query:    q.Get("q"),
        SortBy:   q.Get("sort"),
        Page:     1,
        PageSize: 20,
    }
    
    if page, err := strconv.Atoi(q.Get("page")); err == nil {
        params.Page = page
    }
    if minPrice, err := strconv.ParseFloat(q.Get("min_price"), 64); err == nil {
        params.MinPrice = minPrice
    }
    if maxPrice, err := strconv.ParseFloat(q.Get("max_price"), 64); err == nil {
        params.MaxPrice = maxPrice
    }
    
    result, err := s.es.SearchProducts(r.Context(), "products", params)
    if err != nil {
        log.Printf("Search error: %v", err)
        http.Error(w, "Search failed", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}

func (s *SearchService) SuggestHandler(w http.ResponseWriter, r *http.Request) {
    prefix := r.URL.Query().Get("q")
    if prefix == "" {
        json.NewEncoder(w).Encode([]string{})
        return
    }
    
    suggestions, err := s.es.Suggest(r.Context(), "products", prefix)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(suggestions)
}

func main() {
    esClient, _ := NewESClient(
        []string{"http://localhost:9200"},
        "", "",
    )
    
    db, _ := sql.Open("postgres", "postgres://user:pass@localhost/db")
    pgRepo := &PGFTSRepository{db: db}
    pgRepo.Setup(context.Background())
    
    svc := NewSearchService(esClient, pgRepo)
    
    http.HandleFunc("/search", svc.SearchHandler)
    http.HandleFunc("/suggest", svc.SuggestHandler)
    
    log.Println("Search service running on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## สรุป

| Engine | ข้อดี | ข้อเสีย | Use Case |
|--------|-------|---------|---------|
| Elasticsearch | ทรงพลัง, scalable | Complex setup | Enterprise search |
| Meilisearch | ง่าย, เร็ว | Features น้อยกว่า | Product search |
| PostgreSQL FTS | ไม่ต้องติดตั้งเพิ่ม | Performance ต่ำกว่า | Simple search |

### Best Practices
1. ใช้ async indexing ไม่ block main request
2. Implement incremental indexing
3. Monitor index lag
4. Test search quality ด้วย metrics (NDCG, MRR)
5. A/B test ranking algorithms

---

*จบ Part 68: Full-Text Search*
