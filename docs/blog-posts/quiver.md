# Quiver

A local-first vector database combining structured queries with high-speed similarity search—without the complexity of distributed systems.

## The Vector Search Problem

Picture this: You've built an amazing AI application that needs vector search. The usual choices? Heavyweight vector databases that demand a fleet of machines—or barebones libraries that can't handle metadata, filtering, or hybrid queries.

Why is there no SQLite for vector search?
Something lightweight. Something fast. Something that just works.

Every day, developers face this challenge:

- Heavyweight databases designed for clusters you don't need
- Basic libraries that can't handle structured data alongside vectors
- The choice between power and simplicity
- Infrastructure complexity that gets in the way of building

## Quiver: Vector Search Meets Simplicity

Quiver is what happens when vector search meets simplicity:

- Blazing-fast HNSW for high-performance similarity search
- DuckDB-style local-first indexing—no clusters, no complexity
- SQL-like filtering alongside vector search
- Low overhead—runs on your laptop, embedded in applications, or on-prem

```bash
# Get started in two commands
go get github.com/TFMV/quiver
quiver serve
```

## Why Quiver? The Speed You Need, The Simplicity You Want

### 🚀 Lightning-Fast Local Search

Quiver's HNSW implementation isn't just fast—it's optimized for single-node performance:

```go
// Find similar vectors in microseconds, right on your machine
results, err := index.Search(queryVector, 10)
```

### 🎯 Hybrid Search That Just Works

Combine vector similarity with structured filters, all processed locally:

```go
// Find similar science articles from last week
results, err := index.SearchWithFilter(
    queryVector,
    10,
    "category = 'science' AND date > '2024-03-10'"
)
```

### 📦 Smart Local Processing

Optimized for your machine's resources:

- Efficient memory management
- Local metadata caching
- Background processing that respects your CPU

### 🏹 Local Batch Processing with Arrow

Native support for Apache Arrow means efficient local data handling:

```go
// Batch process vectors locally with Arrow
err := index.AppendFromArrow(record)
```

## Show Me The Numbers

How fast is Quiver?

- 28,800 vector searches per second
- Hybrid search in <0.5ms
- Batch processing at 10M+ vectors/sec

Here are our benchmarks running on a standard 8-core laptop with 1M 128-dimensional vectors:

| Operation     | Throughput    | Latency | Memory/Op | Allocs/Op |
| ------------- | ------------- | ------- | --------- | --------- |
| Search        | 28.8K ops/sec | 41µs    | 1.5 KB    | 18        |
| Hybrid Search | 2.5K ops/sec  | 432µs   | 7.5 KB    | 278       |
| Add           | 5.5K ops/sec  | 3.2ms   | 1.4 KB    | 21        |
| Add Parallel  | 4.5K ops/sec  | 5.3ms   | 1.3 KB    | 20        |
| Arrow Append* | 100 ops/sec   | 2.7s    | 2.3 MB    | 32,332    |

### Batch Ingestion That Scales Without Clusters

- Append 100,000 vectors per operation with Arrow
- Optimized for local-first AI workloads, no distributed infrastructure required
- High throughput: 10M+ records/sec for structured metadata + vectors

What these numbers mean in practice:

- **Fast local queries**: 41µs latency on a single machine
- **Efficient memory use**: Most operations need just 1.5KB of memory
- **Smart resource use**: Parallel operations optimize your machine's capabilities
- **Local batch processing**: Efficient data loading with Arrow

## Where Quiver Shines

### 📚 Semantic Document Search

```go
// Index a document with rich metadata
err := index.Add(docID, embedding, map[string]interface{}{
    "title": "Understanding Vector Search",
    "author": "Jane Doe",
    "tags": []string{"AI", "databases"},
    "readingTime": 5
})
```

### 🎯 Smart Recommendations

```go
// Find similar products within price range
results, err := index.SearchWithFilter(
    userPreferences,
    5,
    "price BETWEEN 10 AND 50"
)
```

### 🖼️ Visual Search

```go
// Find similar images with specific attributes
results, err := index.SearchWithFilter(
    imageEmbedding,
    10,
    "resolution = 'HD' AND style = 'minimalist'"
)
```

### 🤖 RAG Applications

Perfect for Retrieval-Augmented Generation:

```go
// Find relevant context for LLM
context, err := index.SearchWithFilter(
    queryEmbedding,
    3,
    "confidence > 0.8"
)
```

## Getting Started: Two-Minute Setup

### 1. Installation

```bash
go get github.com/TFMV/quiver
```

### 2. Initialize Your Index

```go
index, err := quiver.New(quiver.Config{
    Dimension: 128,
    StoragePath: "vectors.db",
    MaxElements: 100000,
})
```

### 3. Start Searching

```go
// Add vectors with metadata
err = index.Add(1, vectorData, map[string]interface{}{
    "category": "science",
    "author": "Jane Doe",
})

// Search with type safety
results, err := index.Search(queryVector, 10)
```

## REST API: Language-Agnostic Integration

No Go? No problem. Quiver speaks HTTP:

```bash
curl -X POST http://localhost:8080/search/hybrid \
  -H "Content-Type: application/json" \
  -d '{
    "vector": [0.1, 0.2, ...],
    "k": 5,
    "filter": "category = 'science'"
  }'
```

## Why Choose Quiver?

🚀 **Zero setup**—install & search in minutes
⚡ **Blazing-fast**—sub-millisecond queries, even on a laptop
🛠️ **Developer-friendly**—Go SDK & REST API
💡 **No cloud lock-in**—runs anywhere, no clusters required

## What's Next?

Your AI. Your Data. Your Machine.

Fast, flexible vector search—no clusters, no overhead.
Whether you're running on a laptop, embedding in an app, or scaling in production, Quiver gives you vector search that just works.

Ready to simplify vector search? [Check out Quiver on GitHub](https://github.com/TFMV/quiver).
