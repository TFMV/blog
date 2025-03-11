# Building a Hybrid Vector Search Database with Arrow and DuckDB

---

## How we combined HNSW for fast vector search with SQL-based metadata filtering for a next-gen AI database

Vector databases are no longer optional. As embeddings-based applications explode across industries, developers need speed, flexibility, and efficiency when storing, searching, and retrieving high-dimensional vectors. But here's the problem:

Most vector databases force you to choose between raw performance and SQL-powered filtering.

That's why we built Quiver — a Go-powered, hybrid vector database that delivers the best of both worlds:

- ✅ HNSW for fast, high-recall vector search
- ✅ DuckDB for structured metadata filtering
- ✅ Apache Arrow for efficient, zero-copy data movement

With Quiver, you can run complex queries that mix vector similarity with structured constraints — without killing performance.

This article isn't just a high-level introduction. We're diving deep into the internals — how I integrated Apache Arrow, optimized DuckDB for metadata management, and built a high-speed HNSW index — to create a vector search engine that doesn't compromise.

Let's get into it.

## Vector Databases Have a Problem

Vector databases are critical infrastructure for AI applications — powering everything from semantic search and recommendation systems to image similarity and anomaly detection. But despite their importance, most solutions come with serious trade-offs.

### Where Existing Vector Databases Fall Short

- 🔥 **Performance vs. Flexibility Trade-offs** → Most vector databases are built for either fast similarity search or rich metadata filtering — not both. If you want speed, you lose SQL-style filtering. If you want SQL filtering, you lose performance.
- 🔥 **Heavy Resource Consumption** → Many solutions demand huge memory and CPU overhead to maintain vector indices, making them expensive at scale.
- 🔥 **Operational Complexity** → Most vector databases require careful tuning, background maintenance processes, and periodic reindexing to stay performant.
- 🔥 **Integration Challenges** → Existing solutions often sit outside an organization's primary data stack, requiring custom pipelines and workarounds to sync with relational databases and analytical engines.

### How Existing Solutions Approach the Problem

Current vector search solutions take different approaches, each with trade-offs:

- **Weaviate** → Uses HNSW for vector search with a GraphQL-based query engine. Weaviate allows metadata filtering but stores metadata in RocksDB, which comes with additional operational overhead.
- **Pinecone** → A managed service optimized for fast retrieval, but you lose control over the underlying infrastructure and can't easily run it locally.
- **FAISS** → A highly optimized C++ library for pure vector search — but lacks a built-in metadata store, requiring developers to pair it with an external database.
- **pgvector** → Brings vector search inside PostgreSQL, enabling SQL-based filtering, but scales poorly on large datasets due to PostgreSQL's row-based storage.

## Why We Built Quiver

We wanted a vector database that didn't force these trade-offs — one that keeps up with the best ANN search engines while supporting rich, efficient metadata filtering.

- ✅ HNSW for high-speed vector search
- ✅ DuckDB for SQL-powered metadata filtering
- ✅ Apache Arrow for zero-copy, high-performance data movement

Quiver avoids the overhead of external databases, scales efficiently, and fits seamlessly into modern AI and analytics stacks. Let's break down how it works.

## Under the Hood

Quiver is built on three key components, each designed to maximize speed, flexibility, and efficiency:

1️⃣ **Vector Index** → An HNSW (Hierarchical Navigable Small World) graph for fast approximate nearest neighbor (ANN) search.
2️⃣ **Metadata Store** → A DuckDB-backed SQL engine for structured metadata filtering.
3️⃣ **Arrow Appender** → A zero-copy data pipeline powered by Apache Arrow, keeping everything memory-efficient and fast.

These pieces work together to eliminate the trade-offs most vector databases force you to make — allowing queries that combine vector similarity with structured constraints without wrecking performance.

### Blazing-Fast Search: The Vector Index

At the core of Quiver's vector search engine is HNSW (Hierarchical Navigable Small World) — a graph-based ANN algorithm built for speed and scalability.

HNSW is what makes Quiver's search both fast and accurate, offering logarithmic search complexity while keeping recall high. It works by constructing a multi-layered graph where similar vectors cluster together, drastically reducing the number of comparisons needed for a query.

Our Go-based implementation of HNSW is optimized for:

- ✅ Memory efficiency → Minimizing index size without sacrificing recall.
- ✅ High-speed inserts → Batch processing for fast vector ingestion.
- ✅ Precision tuning → Fine-grained control over search parameters for balancing speed vs. accuracy.

```go
type Index struct {
    config          Config
    hnsw            *hnsw.Graph[uint64]
    metadata        map[uint64]map[string]interface{}
    vectors         map[uint64][]float32
    duckdb          *DuckDB
    dbConn          *DuckDBConn
    // Additional fields omitted for brevity
}
```

The HNSW graph is parameterized by several key configuration options:

```go
type Config struct {
    Dimension       int
    StoragePath     string
    Distance        DistanceMetric
    MaxElements     uint64
    HNSWM           int // HNSW hyperparameter M
    HNSWEfConstruct int // HNSW hyperparameter efConstruction
    HNSWEfSearch    int // HNSW hyperparameter ef used during queries
    BatchSize       int // Number of vectors to batch before insertion
    // Additional fields omitted for brevity
}
```

The `HNSWM` parameter controls the maximum number of connections per node in the graph, while `HNSWEfConstruct` and `HNSWEfSearch` control the search breadth during index construction and query time, respectively. These parameters allow for fine-tuning the trade-off between search speed and accuracy.

### SQL Meets Vectors: The Metadata Store

What sets Quiver apart isn't just fast vector search — it's fast vector search with real SQL filtering. That's where DuckDB comes in.

Unlike traditional vector databases that rely on key-value stores or embedded document storage, Quiver uses DuckDB, an in-process analytical database built for speed. This gives us:

- ✅ Full SQL Support → Complex metadata filtering, joins, and aggregations.
- ✅ Columnar Storage → Optimized for analytical workloads, not just key-value lookups.
- ✅ Blazing-Fast Queries → DuckDB is vectorized, meaning it processes queries in parallel with SIMD acceleration.
- ✅ Minimal Overhead → No need for a heavyweight external database — DuckDB runs entirely in-memory when needed.

Our DuckDB integration is designed to be lightweight but powerful, ensuring that metadata queries never become the bottleneck. This is what makes Quiver more than just a vector store — it's a vector-native database that actually understands your data.

### Bridging the Gap with Apache Arrow

HNSW handles fast vector search. ✅
DuckDB gives us powerful SQL filtering. ✅

But there's a missing piece: efficient data movement between them.

Even with a blazing-fast vector index and a high-performance metadata store, Quiver needed a way to move data seamlessly between components — without serialization overhead, unnecessary memory copies, or slow conversion steps.

That's where Apache Arrow comes in.

#### Arrow as the Data Backbone

Quiver doesn't just use DuckDB — it integrates with it at the memory level. Instead of relying on traditional row-based data movement (which would force slow conversions and copies), we use Arrow Database Connectivity (ADBC) to communicate directly with DuckDB in Arrow-native format.

```go
type DuckDB struct {
    mu     sync.Mutex
    db     adbc.Database
    driver adbc.Driver
    opts   DuckDBOptions
    conns  []*DuckDBConn
}

type DuckDBConn struct {
    parent *DuckDB
    conn   adbc.Connection
}
```

With ADBC, metadata queries return results as Arrow RecordBatches, which can be processed without ever leaving columnar memory. This means that vector search results and metadata lookups stay fast, efficient, and zero-copy.

#### Why Apache Arrow?

Most vector databases waste CPU cycles shuffling bytes around — loading, converting, and copying data between different formats. Arrow eliminates this problem by keeping everything in a single, efficient memory format.

Here's why that matters:

- ✅ Zero-Copy Data Transfer → No serialization/deserialization overhead when moving data between Quiver's components.
- ✅ Columnar Storage Efficiency → DuckDB and Arrow both use columnar formats, making queries and analytics dramatically faster.
- ✅ Interoperability → Because Arrow is a standard, Quiver can integrate seamlessly with Python, PostgreSQL, Spark, and other modern data tools.
- ✅ SIMD-Optimized Execution → Arrow enables vectorized processing, meaning modern CPUs can scan and compute over large datasets much faster.

#### How Quiver Uses Arrow

We leverage Arrow in two key areas:

1️⃣ **Ingesting Data** → Bulk-loading vectors and metadata without format conversion.
2️⃣ **Query Results** → Returning search results in an Arrow-native format, making downstream processing in analytics engines or ML pipelines much faster.

By eliminating data movement bottlenecks, Arrow makes Quiver more than just a vector search engine — it's a high-performance data pipeline that fits seamlessly into modern AI and analytics stacks.

## Hybrid Search: Vector Similarity Meets SQL Filtering

Raw vector search is powerful — but it's not enough.

Real-world use cases demand more than just similarity matching. You don't just want to find the most similar product images — you want to filter by price, category, stock status, or user preferences.

That's what makes Quiver different.

By combining HNSW for vector search with DuckDB for structured filtering, Quiver enables hybrid search — queries that mix semantic similarity with SQL-based constraints in a way that's both fast and flexible.

For example:

🔍 Find the 10 most similar product images, but only include products that are in stock and priced under $50.

This kind of query is impossible in most vector databases without pre-filtering or complex workarounds. Quiver makes it seamless.

### Two Approaches to Hybrid Search

Quiver supports two ways to combine vector search with metadata filtering, depending on the use case:

1️⃣ **Pre-Filtering (SQL First, Vector Search Later)**

When metadata constraints are highly selective, Quiver filters first — using DuckDB to narrow down the dataset before running vector search on a smaller set of candidates.

```go
func (idx *Index) SearchWithFilter(query []float32, k int, filter string) ([]SearchResult, error) {
    // Execute SQL filter query
    filteredIDs, err := idx.executeFilterQuery(filter)
    if err != nil {
        return nil, err
    }
    
    // Perform vector search on filtered subset
    results, err := idx.searchFiltered(query, k, filteredIDs)
    if err != nil {
        return nil, err
    }
    
    return results, nil
}
```

✅ Best for: Narrowing down a large dataset when the metadata filter eliminates most records.

2️⃣ **Post-Filtering (Vector Search First, SQL Later)**

When vector similarity is the dominant factor, Quiver runs the ANN search first, then filters the results using DuckDB.

```go
func (idx *Index) SearchWithPostFilter(query []float32, k int, filter string) ([]SearchResult, error) {
    // Perform vector search with a larger k to account for filtering
    results, err := idx.Search(query, k*2, 0, 0)
    if err != nil {
        return nil, err
    }
    
    // Extract IDs for filtering
    ids := make([]uint64, len(results))
    for i, r := range results {
        ids[i] = r.ID
    }
    
    // Apply filter to results
    filteredResults, err := idx.filterResults(ids, filter, k)
    if err != nil {
        return nil, err
    }
    
    return filteredResults, nil
}
```

✅ Best for: Cases where the vector search results are already highly relevant, and metadata filtering is secondary.

### How Quiver Optimizes Hybrid Search

Quiver doesn't just run both steps in sequence — it chooses the best approach dynamically based on the query.

- If the metadata filter is highly selective → Pre-filtering is faster.
- If the metadata filter is loose → Post-filtering avoids unnecessary constraints.

By intelligently switching between these approaches, Quiver avoids the performance pitfalls that make hybrid search slow in other databases.

The result? A system that feels as fast as raw vector search but acts as flexible as a SQL database.

## Making Quiver Fast at Scale

Building a vector database is one thing — making it fast and scalable is another.

Quiver isn't just optimized for raw search speed. We've engineered it for high-throughput inserts, low-latency queries, and efficient storage — without unnecessary overhead.

Here's how we make it fast.

### 1️⃣ Batch Processing for High-Speed Ingestion

Vector addition needs to be fast and scalable, especially when ingesting large datasets. Instead of inserting vectors one by one, Quiver batches inserts to improve throughput.

Each new vector is added to a buffer, which is flushed in bulk once it reaches the configured batch size:

```go
func (idx *Index) Add(id uint64, vector []float32, meta map[string]interface{}) error {
    // Add to batch buffer
    idx.batchLock.Lock()
    idx.batchBuffer = append(idx.batchBuffer, vectorMeta{
        id:     id,
        vector: vector,
        meta:   meta,
    })
    idx.batchLock.Unlock()
    
    // Flush batch if it reaches the configured size
    if len(idx.batchBuffer) >= idx.config.BatchSize {
        return idx.flushBatch()
    }
    
    return nil
}
```

A background goroutine automatically flushes the batch buffer, ensuring optimal insert performance without blocking writes.

✅ Why it matters: Batching minimizes disk I/O and computational overhead, making bulk imports orders of magnitude faster than inserting vectors individually.

### 2️⃣ Caching for Low-Latency Metadata Lookups

Metadata filtering is a key feature of Quiver, but repeated database queries can slow things down.

To avoid unnecessary lookups, Quiver caches metadata in memory:

```go
func (idx *Index) getMetadata(id uint64) map[string]interface{} {
    // Check cache first
    if meta, ok := idx.cache.Load(id); ok {
        return meta.(map[string]interface{})
    }
    
    // Fetch from database if not in cache
    idx.lock.RLock()
    meta, exists := idx.metadata[id]
    idx.lock.RUnlock()
    
    if exists {
        // Store in cache for future use
        idx.cache.Store(id, meta)
        return meta
    }
    
    return nil
}
```

✅ Why it matters: Instead of hitting DuckDB for every query, Quiver retrieves frequently accessed metadata from memory, significantly reducing query latency.

### 3️⃣ Efficient Persistence and Backup

Speed is important, but so is data durability.

Quiver supports efficient persistence and backup mechanisms to ensure that vector indices and metadata aren't lost.

```go
func (idx *Index) persistToStorage() error {
    // Implementation details for persisting the index to disk
}

func (idx *Index) Backup(path string, incremental bool, compress bool) error {
    // Implementation details for creating backups
}
```

- 🔹 Incremental backups → Store only what's changed, minimizing storage costs.
- 🔹 Compression options → Reduce disk space usage without compromising speed.

✅ Why it matters: Quiver provides persistence without sacrificing performance, ensuring that large-scale vector indices remain both durable and efficient.

## How Fast is Quiver?

Speed isn't just a feature — it's the foundation of a great vector database.

We built Quiver to handle high-throughput inserts, low-latency queries, and efficient hybrid search without the performance bottlenecks that plague traditional solutions. Here's what that looks like in practice (benchmarked on an M2 Pro CPU):

- ✅ Fast Vector Search → 16.9K queries per second with 59µs latency.
- ✅ Hybrid Search (Vectors + SQL Filtering) → 4.8K queries per second with 208µs latency.
- ✅ Negative Search (Exclusion-based ranking) → 7.9K queries per second, dynamically reranking results in 126µs.
- ✅ Batch Inserts (1,000 vectors at a time) → 6.6 batches per second, processing 19MB per batch.

For real-time applications, sub-millisecond query times mean Quiver can scale effortlessly across millions of vectors. Whether you're handling embeddings for AI-powered search, large-scale recommendations, or anomaly detection, Quiver delivers speed without compromise.

## Use Cases

Quiver's hybrid search capabilities allow you to go beyond simple vector similarity by incorporating metadata filtering, ranking, and exclusion-based logic in a single query.

Let's explore two powerful use cases:

🔹 **Simple Example: Find Relevant Documents**

A basic use case for Quiver is semantic search with filtering — for example, retrieving the most relevant news articles published in the past week.

```go
// Find documents similar to the query that were published in the last 7 days
results, err := index.SearchWithFilter(
    queryVector,
    10,
    "published_date > DATE_SUB(NOW(), INTERVAL 7 DAY)"
)
```

This ensures only recent documents are returned while still ranking results based on vector similarity.

🔹 **Advanced Example: Negative Search (Excluding Irrelevant Matches)**

In many applications, it's not enough to just find similar items — you also need to exclude certain results.

For example, a recommendation system might want to:

- Find items similar to what a user likes ✅
- Exclude items they've already purchased ❌
- Avoid items that match known negative preferences ❌

Instead of filtering metadata after the fact, Quiver reranks results in the vector space, actively pushing down unwanted matches using a negative search query.

```go
// Find movies similar to what the user likes but exclude those they disliked
results, err := index.SearchWithNegatives(
    likedMovieVector,    // Positive query
    dislikedMovieVectors, // Negative queries
    10, 1, 10,          // k, page, pageSize
)
```

Under the hood, Quiver:

- ✅ Performs a high-recall vector search to find candidates
- ✅ Computes similarity to both positive and negative examples
- ✅ Reranks results, pushing down items similar to the negative examples

This approach improves recommendation accuracy, ensuring the user sees highly relevant, non-repetitive results.

### Why Negative Search Matters

Most vector databases can't handle exclusion-based ranking efficiently — forcing developers to run multiple queries and manually filter results.

Quiver's built-in negative search allows:

- ✅ Personalized recommendations (exclude previously seen or irrelevant items)
- ✅ Bias reduction in search results (downweight unwanted clusters)
- ✅ More diverse results by actively discouraging over-represented embeddings

## The Future of Hybrid Search is Here

Quiver isn't just another vector database. It's a fundamental rethink of what's possible when you combine fast similarity search, real SQL filtering, and zero-copy data movement into a single, high-performance system.

Instead of forcing trade-offs between speed and flexibility, Quiver embraces both, enabling hybrid search that feels natural and runs at scale.

We built Quiver because we saw an opportunity to do things differently — to eliminate bottlenecks, remove complexity, and provide a vector-native database that integrates seamlessly into modern AI and analytics stacks.

And this is just the beginning.

## Get Involved

Quiver is open source. That means you can:

- ✅ Try it today → Get started with a few lines of Go.
- ✅ Contribute → Help shape the future of hybrid vector search.
- ✅ Join the discussion → We're building this in the open, and we want your feedback.
