# Database, Ollama, and Infrastructure Reference

Comprehensive reference for PostgreSQL, Neo4j, Ollama, and related infrastructure configuration.

## Table of Contents

- [PostgreSQL with pgvector](#postgresql-with-pgvector)
- [Neo4j Graph Database](#neo4j-graph-database)
- [Ollama LLM Server](#ollama-llm-server)
- [Embedding Models Reference](#embedding-models-reference)
- [Performance Tuning](#performance-tuning)

---

## PostgreSQL with pgvector

### Overview

PostgreSQL serves two purposes:
1. **Vector Storage** - Storing memory embeddings via pgvector extension
2. **Authentication** - Token storage and validation

### Schema

#### memories Table (managed by mem0)

```sql
CREATE TABLE memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    memory TEXT NOT NULL,
    embedding VECTOR(1024),  -- Dimension matches your embedding model
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- HNSW index for fast similarity search (dims <= 2000)
CREATE INDEX ON memories USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

#### auth_tokens Table

```sql
CREATE TABLE auth_tokens (
    id SERIAL PRIMARY KEY,
    token_hash VARCHAR(64) NOT NULL UNIQUE,
    token_prefix VARCHAR(12) NOT NULL,
    user_id VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    metadata JSONB
);

CREATE INDEX idx_auth_tokens_hash ON auth_tokens(token_hash);
CREATE INDEX idx_auth_tokens_user ON auth_tokens(user_id);
```

### Common Queries

#### Check Vector Dimensions

```sql
SELECT vector_dims(embedding) as dims, count(*)
FROM memories
GROUP BY vector_dims(embedding);
```

#### Search Memories by Similarity

```sql
SELECT id, memory, 1 - (embedding <=> $1::vector) as similarity
FROM memories
WHERE user_id = $2
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

#### Check Index Status

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'memories';
```

#### Memory Statistics

```sql
SELECT
    user_id,
    count(*) as memory_count,
    avg(char_length(memory)) as avg_length,
    min(created_at) as first_memory,
    max(created_at) as last_memory
FROM memories
GROUP BY user_id;
```

### HNSW Index Configuration

The HNSW (Hierarchical Navigable Small World) index provides fast approximate nearest neighbor search.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `m` | 16 | Max connections per layer (higher = more accurate, more memory) |
| `ef_construction` | 64 | Size of dynamic candidate list during construction |

**Dimension Limits:**
- HNSW indexing is **enabled** for dimensions ≤ 2000
- HNSW indexing is **disabled** for dimensions > 2000 (falls back to sequential scan)

### Maintenance

```sql
-- Vacuum and analyze for performance
VACUUM ANALYZE memories;

-- Reindex if needed
REINDEX INDEX memories_embedding_idx;

-- Check table size
SELECT pg_size_pretty(pg_total_relation_size('memories'));
```

---

## Neo4j Graph Database

### Overview

Neo4j stores the knowledge graph for memory relationships, components, and decisions.

### Node Types

```cypher
// Memory Node
(:Memory {
    id: "uuid",
    text: "memory content",
    user_id: "project-id",
    topic: "optional-topic",
    created: datetime(),
    trust_score: 0.0
})

// Component Node
(:Component {
    name: "component-name",
    type: "Service|Database|API|Feature",
    created: datetime(),
    metadata: {}
})

// Decision Node
(:Decision {
    id: "decision_timestamp",
    text: "decision description",
    user_id: "project-id",
    created: datetime(),
    metadata: {}
})

// Argument Node (for decisions)
(:Argument {
    type: "PRO|CON",
    text: "argument text",
    order: 0
})
```

### Relationship Types

| Relationship | From | To | Purpose |
|--------------|------|-----|---------|
| `RELATES_TO` | Memory | Memory | General association |
| `DEPENDS_ON` | Memory/Component | Memory/Component | Dependency |
| `SUPERSEDES` | Memory | Memory | Replaces old knowledge |
| `RESPONDS_TO` | Memory | Memory | Conversation thread |
| `EXTENDS` | Memory | Memory | Adds detail |
| `CONFLICTS_WITH` | Memory | Memory | Contradictory info |
| `AFFECTS` | Memory | Component | Memory affects component |
| `BASED_ON` | Decision | Argument | Pro argument |
| `CONSIDERED` | Decision | Argument | Con argument |
| `CHOSEN_OVER` | Decision | Decision | Alternative rejected |

### Common Cypher Queries

#### Find Related Memories (2 hops)

```cypher
MATCH path = (start:Memory {id: $id})-[*1..2]-(related:Memory)
RETURN DISTINCT related.id, related.text, length(path) as distance
ORDER BY distance;
```

#### Get Memory Evolution

```cypher
MATCH (m:Memory {topic: $topic})
OPTIONAL MATCH (m)-[:SUPERSEDES]->(old:Memory)
RETURN m.id, m.text, m.created, old.id as superseded
ORDER BY m.created ASC;
```

#### Component Impact Analysis

```cypher
MATCH (c:Component {name: $name})
OPTIONAL MATCH (dependent:Component)-[:DEPENDS_ON*]->(c)
OPTIONAL MATCH (m:Memory)-[:AFFECTS]->(c)
RETURN c.name,
       collect(DISTINCT dependent.name) as dependents,
       collect(DISTINCT m.id) as affecting_memories;
```

#### Memory Intelligence Summary

```cypher
MATCH (m:Memory {user_id: $user_id})
OPTIONAL MATCH (m)-[r]-()
WITH m, count(r) as connections
RETURN count(m) as total,
       avg(connections) as avg_connections,
       sum(CASE WHEN connections = 0 THEN 1 ELSE 0 END) as isolated;
```

### Indexes

```cypher
// Create indexes for performance
CREATE INDEX memory_id FOR (m:Memory) ON (m.id);
CREATE INDEX memory_user FOR (m:Memory) ON (m.user_id);
CREATE INDEX component_name FOR (c:Component) ON (c.name);
CREATE INDEX decision_id FOR (d:Decision) ON (d.id);
```

### Maintenance

```cypher
// Database stats
CALL db.stats.retrieve('GRAPH COUNTS');

// List all relationships
CALL db.relationshipTypes();

// Clear all data (careful!)
MATCH (n) DETACH DELETE n;
```

---

## Ollama LLM Server

### Overview

Ollama provides local LLM inference for:
1. **Memory Extraction** - Extracting key facts from conversations
2. **Embeddings** - Converting text to vectors for semantic search

### Recommended Models

#### LLM Models (Memory Extraction)

| Model | Parameters | VRAM | Speed | Quality |
|-------|------------|------|-------|---------|
| `qwen3:8b` | 8B | ~6GB | Fast | Good |
| `qwen3:14b` | 14B | ~10GB | Medium | Better |
| `llama3.2:8b` | 8B | ~6GB | Fast | Good |
| `mistral:7b` | 7B | ~5GB | Fast | Good |

#### Embedding Models

| Model | Dimensions | VRAM | HNSW | Notes |
|-------|------------|------|------|-------|
| `qwen3-embedding:0.6b` | 1024 | ~1GB | Yes | **Recommended** |
| `qwen3-embedding:4b` | 2560 | ~3GB | No | Good quality |
| `qwen3-embedding:8b` | 4096 | ~6GB | No | Best quality |
| `nomic-embed-text` | 768 | ~1GB | Yes | Fast, good |
| `all-minilm` | 384 | <1GB | Yes | Fastest |

### Installation and Setup

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull recommended models
ollama pull qwen3:8b
ollama pull qwen3-embedding:0.6b

# Start server (if not running as service)
ollama serve
```

### Configuration

```bash
# .env configuration
OLLAMA_BASE_URL=http://localhost:11434  # or http://gpu-server:11434
OLLAMA_LLM_MODEL=qwen3:8b
OLLAMA_EMBEDDING_MODEL=qwen3-embedding:0.6b
OLLAMA_EMBEDDING_DIMS=1024
```

### Network Configuration

For remote Ollama servers:

```bash
# On Ollama server, set environment variable
OLLAMA_HOST=0.0.0.0

# Or in systemd service file
Environment="OLLAMA_HOST=0.0.0.0"
```

### Health Check

```bash
# Check Ollama is running
curl http://localhost:11434/api/tags

# Test embedding
curl http://localhost:11434/api/embeddings -d '{
  "model": "qwen3-embedding:0.6b",
  "prompt": "test"
}'

# Test generation
curl http://localhost:11434/api/generate -d '{
  "model": "qwen3:8b",
  "prompt": "Hello",
  "stream": false
}'
```

---

## Embedding Models Reference

### Matryoshka Representation Learning (MRL)

Some models (like `qwen3-embedding`) support MRL, which allows truncating embeddings to smaller dimensions while preserving quality.

**How it works:**
1. Model outputs full-dimension embedding (e.g., 4096)
2. Truncate to target dimensions (e.g., 1024)
3. Re-normalize for cosine similarity

**Configuration:**
```bash
# Use 4b model but truncate to 1024 dims
OLLAMA_EMBEDDING_MODEL=qwen3-embedding:4b
OLLAMA_EMBEDDING_DIMS=1024  # Truncated from 2560
```

### Dimension Selection Guide

| Use Case | Recommended Dims | Model |
|----------|------------------|-------|
| Fast search, limited memory | 384-768 | all-minilm, nomic |
| Balanced (recommended) | 1024 | qwen3-embedding:0.6b |
| High quality, more storage | 1536-2560 | qwen3-embedding:4b |
| Maximum quality | 4096 | qwen3-embedding:8b |

### OpenAI Embeddings

If using OpenAI instead of Ollama:

```bash
LLM_PROVIDER=openai
OPENAI_API_KEY=sk-proj-...
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_EMBEDDING_DIMS=1536
```

| Model | Dimensions | Cost/1M tokens |
|-------|------------|----------------|
| text-embedding-3-small | 1536 | $0.02 |
| text-embedding-3-large | 3072 | $0.13 |
| text-embedding-ada-002 | 1536 | $0.10 |

---

## Performance Tuning

### PostgreSQL

```yaml
# docker-compose.yml additions
postgres:
  environment:
    - POSTGRES_SHARED_BUFFERS=256MB
    - POSTGRES_WORK_MEM=16MB
    - POSTGRES_MAINTENANCE_WORK_MEM=64MB
  shm_size: "256mb"
```

### Neo4j

```yaml
neo4j:
  environment:
    - NEO4J_dbms_memory_heap_initial__size=512m
    - NEO4J_dbms_memory_heap_max__size=1g
    - NEO4J_dbms_memory_pagecache_size=512m
```

### Ollama

```bash
# Increase context length
OLLAMA_NUM_CTX=4096

# GPU layers (for partial offload)
OLLAMA_NUM_GPU=99  # All layers on GPU
```

### Connection Pooling

```python
# MCP server connection pool
db_pool = await asyncpg.create_pool(
    min_size=2,
    max_size=10,
    command_timeout=60
)

# HTTP client timeout
http_client = httpx.AsyncClient(timeout=180.0)
```

---

## Monitoring

### PostgreSQL Metrics

```sql
-- Active connections
SELECT count(*) FROM pg_stat_activity;

-- Slow queries
SELECT query, calls, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

### Neo4j Metrics

```cypher
// Query performance
CALL db.stats.retrieve('GRAPH COUNTS');

// Memory usage
CALL dbms.listPools() YIELD pool, heapMemoryUsed, heapMemoryUsedBytes;
```

### Docker Stats

```bash
# Resource usage
docker stats mem0-mcp-postgres mem0-mcp-neo4j mem0-mcp-server

# Disk usage
docker system df -v
```

---

## Related Documentation

- [CONFIGURATION.md](./CONFIGURATION.md) - All configuration options
- [PERFORMANCE.md](./PERFORMANCE.md) - Detailed performance tuning
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues
- [PORTABLE_SETUP.md](./PORTABLE_SETUP.md) - Migration guide
