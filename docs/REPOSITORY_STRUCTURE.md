# Repository Structure and Replication Guide

This document provides a complete overview of the repository structure and instructions for replicating or extending the project.

## Table of Contents

- [Directory Structure](#directory-structure)
- [Component Overview](#component-overview)
- [File-by-File Reference](#file-by-file-reference)
- [Replication Guide](#replication-guide)
- [Extension Points](#extension-points)

---

## Directory Structure

```
mem0-server-mcp/
├── .env.example              # Environment configuration template
├── docker-compose.yml        # Docker orchestration for all services
├── LICENSE                   # MIT License
├── README.md                 # Project overview and quick start
├── requirements-cli.txt      # CLI tool dependencies
│
├── docs/                     # Documentation
│   ├── API.md               # REST API reference
│   ├── ARCHITECTURE.md      # System architecture
│   ├── AUTHENTICATION.md    # Auth system details
│   ├── CONFIGURATION.md     # Configuration options
│   ├── DATABASE_REFERENCE.md # Database schemas and queries
│   ├── ENHANCEMENT_PROPOSALS.md # Future improvements
│   ├── GAP_ANALYSIS.md      # Feature gap analysis
│   ├── MCP_TOOLS.md         # MCP tool documentation
│   ├── PERFORMANCE.md       # Performance tuning
│   ├── PORTABLE_SETUP.md    # Migration and portability
│   ├── QUICKSTART.md        # 5-minute setup guide
│   ├── REPOSITORY_STRUCTURE.md # This file
│   ├── SECURITY.md          # Security considerations
│   └── TROUBLESHOOTING.md   # Common issues
│
├── examples/                 # Usage examples
│   ├── claude_code_config.json    # Claude Code MCP config
│   └── memory_workflows.md        # Example workflows
│
├── mcp-server/              # MCP Protocol Server (Port 8080)
│   ├── Dockerfile           # Container build
│   ├── requirements.txt     # Python dependencies
│   ├── auth.py             # Token authentication
│   ├── config.py           # MCP server configuration
│   ├── main.py             # FastMCP server with 13 tools
│   └── text_chunker.py     # Smart text chunking
│
├── mem0-server/             # Mem0 REST API Server (Port 8000)
│   ├── Dockerfile           # Container build
│   ├── requirements.txt     # Python dependencies
│   ├── config.py           # Mem0 configuration
│   ├── main.py             # FastAPI server with 28 endpoints
│   ├── graph_intelligence.py # Neo4j graph operations
│   └── truncate_embedder.py  # MRL embedding support
│
├── migrations/              # Database migrations
│   └── 001_create_auth_tokens.sql  # Auth token table
│
├── scripts/                 # Utility scripts
│   ├── mcp-token.py        # Token generation CLI
│   ├── start.sh            # Quick start script
│   └── test.sh             # Test runner
│
└── tests/                   # Test suite
    ├── conftest.py         # Pytest configuration
    ├── test_api.py         # API tests
    ├── test_auth.py        # Authentication tests
    ├── test_chunking.py    # Text chunking tests
    └── test_mcp.py         # MCP protocol tests
```

---

## Component Overview

### 1. MCP Server (`mcp-server/`)

The Model Context Protocol server that Claude Code connects to.

| File | Purpose | Key Classes/Functions |
|------|---------|----------------------|
| `main.py` | FastMCP server implementation | `mcp`, 13 `@mcp.tool()` functions |
| `auth.py` | Token validation | `TokenAuthenticator` |
| `config.py` | Configuration loading | Environment variable handling |
| `text_chunker.py` | Large text handling | `chunk_text_semantic()` |

**MCP Tools Provided:**
1. `add_coding_preference` - Store memories
2. `get_all_coding_preferences` - Retrieve all
3. `search_coding_preferences` - Semantic search
4. `delete_memory` - Remove memory
5. `get_memory_history` - View changes
6. `link_memories` - Create relationships
7. `get_related_memories` - Graph traversal
8. `analyze_memory_intelligence` - Full analysis
9. `create_component` - Architecture mapping
10. `link_component_dependency` - Dependencies
11. `analyze_component_impact` - Impact analysis
12. `create_decision` - Decision tracking
13. `get_decision_rationale` - Decision retrieval

### 2. Mem0 Server (`mem0-server/`)

The REST API backend that handles memory operations.

| File | Purpose | Key Classes/Functions |
|------|---------|----------------------|
| `main.py` | FastAPI application | 28 REST endpoints |
| `config.py` | Multi-provider config | `get_mem0_config()` |
| `graph_intelligence.py` | Neo4j operations | `MemoryIntelligence` class |
| `truncate_embedder.py` | Embedding truncation | `TruncateEmbedder` class |

**API Endpoint Categories:**
- Memory CRUD: `/memories`, `/memories/{id}`
- Search: `/search`, `/search/enhanced`
- Graph Intelligence: `/graph/*` (15 endpoints)
- Admin: `/configure`, `/reset`, `/health`

### 3. Database Layer

**PostgreSQL with pgvector:**
- Vector storage for semantic search
- Authentication token storage
- HNSW indexing for fast similarity search

**Neo4j:**
- Knowledge graph storage
- Relationship tracking
- Graph traversal queries

---

## File-by-File Reference

### Root Configuration Files

#### `.env.example`
```bash
# Template for environment configuration
# Copy to .env and customize
```
Contains all configurable options with documentation.

#### `docker-compose.yml`
Defines 4 services:
- `postgres` - pgvector/pgvector:pg17
- `neo4j` - neo4j:5.26.4 with APOC
- `mem0` - Custom FastAPI server
- `mcp` - Custom MCP server

### MCP Server Files

#### `mcp-server/main.py`

Core server implementation using FastMCP:

```python
# Key components:
mcp = FastMCP("mem0-mcp")  # Server instance

@mcp.tool(description="...")
async def add_coding_preference(text: str) -> str:
    # Tool implementation

def create_starlette_app(mcp_server, debug=False):
    # Creates ASGI app with:
    # - /sse endpoint (legacy)
    # - /mcp endpoint (HTTP Stream)
    # - /messages endpoint (SSE messages)
    # - AuthMiddleware for token extraction
```

#### `mcp-server/auth.py`

Token authentication system:

```python
class TokenAuthenticator:
    async def validate_token(token, user_id, project_id, action):
        # Validates against PostgreSQL
        # Returns {"valid": bool, "error": str, ...}
```

#### `mcp-server/text_chunker.py`

Smart text chunking for large inputs:

```python
def chunk_text_semantic(text, max_chunk_size=1000, overlap_size=150):
    # Splits at paragraph/sentence boundaries
    # Returns list of chunk dictionaries
```

### Mem0 Server Files

#### `mem0-server/main.py`

FastAPI application with async architecture:

```python
# Key patterns:
MEMORY_INSTANCE = Memory.from_config(config.get_mem0_config())
GRAPH_INTELLIGENCE = MemoryIntelligence(...)

@app.post("/memories")
async def add_memory(memory_create: MemoryCreate):
    # 1. Store in PostgreSQL (immediate)
    # 2. Sync to Neo4j (background task)
```

#### `mem0-server/graph_intelligence.py`

Neo4j graph operations:

```python
class MemoryIntelligence:
    # Relationship methods
    def link_memories(id1, id2, rel_type)
    def get_related_memories(id, depth)

    # Temporal methods
    def get_memory_evolution(topic)
    def find_superseded_memories(user_id)

    # Component methods
    def create_component(name, type)
    def get_impact_analysis(component)

    # Decision methods
    def create_decision(text, pros, cons, alternatives)
    def get_decision_rationale(id)

    # Analysis methods
    def analyze_memory_intelligence(user_id)
```

#### `mem0-server/truncate_embedder.py`

Support for Matryoshka Representation Learning (MRL) embeddings:

```python
class TruncateEmbedder:
    def embed(text):
        # Get full embedding from model
        # Truncate to target dimensions
        # Normalize for cosine similarity
```

---

## Replication Guide

### Step 1: Clone and Understand

```bash
git clone https://github.com/lprior-repo/mem0-server-mcp.git
cd mem0-server-mcp

# Understand the structure
tree -L 2
cat README.md
```

### Step 2: Set Up Dependencies

```bash
# Install development tools
pip install -r requirements-cli.txt

# Pull required Docker images
docker pull pgvector/pgvector:pg17
docker pull neo4j:5.26.4
```

### Step 3: Configure Environment

```bash
cp .env.example .env

# Edit .env:
# - Set OLLAMA_BASE_URL to your Ollama server
# - Adjust embedding model and dimensions
# - Set secure passwords for production
```

### Step 4: Build and Run

```bash
# Build custom images
docker compose build

# Start all services
docker compose up -d

# Verify health
curl http://localhost:8080/
curl http://localhost:8000/health
```

### Step 5: Generate Auth Token

```bash
python scripts/mcp-token.py generate \
  --user-id "your-project" \
  --description "Development token"
```

### Step 6: Configure Claude Code

```json
{
  "mcpServers": {
    "mem0": {
      "type": "sse",
      "url": "http://localhost:8080/sse",
      "headers": {
        "X-MCP-Token": "your-generated-token",
        "X-MCP-UserID": "your-project"
      }
    }
  }
}
```

---

## Extension Points

### Adding New MCP Tools

1. **Add tool in `mcp-server/main.py`:**

```python
@mcp.tool(
    description="""Your tool description here."""
)
async def your_new_tool(param1: str, param2: int = 10) -> str:
    auth_result = await validate_auth()
    if not auth_result["valid"]:
        return f"Authentication failed: {auth_result['error']}"

    # Your implementation
    return "Success"
```

2. **Add corresponding API endpoint in `mem0-server/main.py` if needed**

### Adding New Graph Operations

1. **Add method to `MemoryIntelligence` class:**

```python
def your_new_operation(self, param):
    with self.driver.session() as session:
        result = session.run("""
            MATCH (m:Memory)
            // Your Cypher query
            RETURN m
        """, param=param)
        return [dict(r) for r in result]
```

2. **Add API endpoint:**

```python
@app.get("/graph/your-operation")
def your_operation_endpoint(param: str):
    return GRAPH_INTELLIGENCE.your_new_operation(param)
```

3. **Add MCP tool wrapper**

### Adding New LLM Providers

1. **Update `mem0-server/config.py`:**

```python
def get_mem0_config():
    if LLM_PROVIDER == 'your_provider':
        return {
            "llm": {
                "provider": "your_provider",
                "config": {
                    # Provider-specific config
                }
            },
            "embedder": {
                # Embedder config
            },
            # ... rest of config
        }
```

2. **Add environment variables to `.env.example`**

3. **Update documentation**

---

## Development Workflow

### Running Tests

```bash
# All tests
pytest tests/

# Specific test file
pytest tests/test_api.py -v

# With coverage
pytest --cov=mcp-server --cov=mem0-server tests/
```

### Local Development (without Docker)

```bash
# Terminal 1: Mem0 server
cd mem0-server
pip install -r requirements.txt
python main.py

# Terminal 2: MCP server
cd mcp-server
pip install -r requirements.txt
python main.py --skip-wait
```

### Debugging

```bash
# View logs
docker compose logs -f mem0
docker compose logs -f mcp

# Shell into container
docker exec -it mem0-mcp-server bash

# Database queries
docker exec -it mem0-mcp-postgres psql -U postgres
```

---

## Related Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - High-level system design
- [API.md](./API.md) - REST API reference
- [MCP_TOOLS.md](./MCP_TOOLS.md) - MCP tool documentation
- [CONFIGURATION.md](./CONFIGURATION.md) - All configuration options
