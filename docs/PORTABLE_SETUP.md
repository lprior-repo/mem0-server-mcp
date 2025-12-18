# Portable Setup and Migration Guide

This guide covers how to deploy mem0-server-mcp in different environments, migrate between setups, and ensure data portability.

## Table of Contents

- [Deployment Options](#deployment-options)
- [Environment Migration](#environment-migration)
- [Data Backup and Restore](#data-backup-and-restore)
- [Multi-Machine Setup](#multi-machine-setup)
- [Cloud Deployment](#cloud-deployment)

---

## Deployment Options

### Option 1: Local Docker Compose (Recommended for Development)

The default setup runs everything locally via Docker Compose:

```bash
# Clone and start
git clone https://github.com/lprior-repo/mem0-server-mcp.git
cd mem0-server-mcp
cp .env.example .env

# Edit .env with your Ollama server address
docker compose up -d
```

**Pros:**
- Simple one-command deployment
- All services isolated in containers
- Easy to tear down and rebuild

**Cons:**
- Requires Docker on the host machine
- Resources shared with host system

### Option 2: External Database Services

For production or shared environments, use external database services:

```bash
# .env configuration for external services
POSTGRES_HOST=your-postgres-host.example.com
POSTGRES_PORT=5432
POSTGRES_USER=mem0user
POSTGRES_PASSWORD=secure-password-here
POSTGRES_DB=mem0

NEO4J_URI=bolt://your-neo4j-host.example.com:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=secure-neo4j-password
```

Then run only the application containers:

```bash
# docker-compose.override.yml
version: '3.8'
services:
  postgres:
    profiles: ["disabled"]
  neo4j:
    profiles: ["disabled"]
```

### Option 3: Kubernetes Deployment

For enterprise deployments, deploy to Kubernetes:

```yaml
# k8s/mem0-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mem0-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mem0-server
  template:
    metadata:
      labels:
        app: mem0-server
    spec:
      containers:
      - name: mem0
        image: mem0-server:latest
        ports:
        - containerPort: 8000
        env:
        - name: POSTGRES_HOST
          valueFrom:
            secretKeyRef:
              name: mem0-secrets
              key: postgres-host
        # ... additional env vars from secrets
```

---

## Environment Migration

### Migrating from Development to Production

1. **Export your authentication tokens:**

```bash
# Connect to PostgreSQL and export tokens
docker exec -it mem0-mcp-postgres psql -U postgres -d postgres -c \
  "COPY (SELECT * FROM auth_tokens) TO STDOUT WITH CSV HEADER" > tokens_backup.csv
```

2. **Export your memories:**

```bash
# Use the API to export all memories for a user
curl -X GET "http://localhost:8000/memories?user_id=your_project_id" \
  -H "Content-Type: application/json" > memories_backup.json
```

3. **Export Neo4j graph data:**

```bash
# Use Neo4j's export functionality
docker exec -it mem0-mcp-neo4j cypher-shell -u neo4j -p mem0graph \
  "CALL apoc.export.json.all('export.json', {})"
```

4. **Import to production:**

```bash
# Import tokens
psql -h production-host -U postgres -d postgres -c \
  "COPY auth_tokens FROM STDIN WITH CSV HEADER" < tokens_backup.csv

# Import memories via API
curl -X POST "http://production:8000/memories" \
  -H "Content-Type: application/json" \
  -d @memories_backup.json
```

### Migrating Between LLM Providers

When switching from Ollama to OpenAI (or vice versa):

1. **Backup existing memories** (embeddings will need regeneration)

2. **Update .env:**

```bash
# Switch from Ollama to OpenAI
LLM_PROVIDER=openai
OPENAI_API_KEY=sk-proj-...
OPENAI_LLM_MODEL=gpt-4o
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_EMBEDDING_DIMS=1536
```

3. **Reset the vector store** (required when changing embedding dimensions):

```bash
curl -X POST "http://localhost:8000/reset"
```

4. **Re-import memories** (they will be re-embedded with new model)

> **Warning:** Changing embedding models requires re-indexing all memories. The vector store cannot mix embeddings from different models.

---

## Data Backup and Restore

### Automated Backup Script

Create `scripts/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backups/mem0-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup PostgreSQL
docker exec mem0-mcp-postgres pg_dump -U postgres postgres > "$BACKUP_DIR/postgres.sql"

# Backup Neo4j
docker exec mem0-mcp-neo4j neo4j-admin database dump neo4j --to-path=/tmp
docker cp mem0-mcp-neo4j:/tmp/neo4j.dump "$BACKUP_DIR/"

# Backup configuration
cp .env "$BACKUP_DIR/"

echo "Backup completed: $BACKUP_DIR"
```

### Restore Script

Create `scripts/restore.sh`:

```bash
#!/bin/bash
BACKUP_DIR="$1"

if [ -z "$BACKUP_DIR" ]; then
  echo "Usage: ./restore.sh /path/to/backup"
  exit 1
fi

# Restore PostgreSQL
docker exec -i mem0-mcp-postgres psql -U postgres postgres < "$BACKUP_DIR/postgres.sql"

# Restore Neo4j
docker cp "$BACKUP_DIR/neo4j.dump" mem0-mcp-neo4j:/tmp/
docker exec mem0-mcp-neo4j neo4j-admin database load neo4j --from-path=/tmp/neo4j.dump --overwrite-destination

echo "Restore completed from: $BACKUP_DIR"
```

### Scheduled Backups with Cron

```bash
# Add to crontab (daily backup at 2 AM)
0 2 * * * /path/to/mem0-server-mcp/scripts/backup.sh >> /var/log/mem0-backup.log 2>&1
```

---

## Multi-Machine Setup

### Scenario: Ollama on a Separate GPU Server

```
┌─────────────────┐      ┌─────────────────┐
│  Workstation    │      │   GPU Server    │
│                 │      │                 │
│  Claude Code    │      │  Ollama         │
│  MCP Server     │◄────►│  - qwen3:8b     │
│  Mem0 Server    │      │  - embeddings   │
│  PostgreSQL     │      │                 │
│  Neo4j          │      │                 │
└─────────────────┘      └─────────────────┘
```

**Configuration:**

```bash
# .env on workstation
OLLAMA_BASE_URL=http://192.168.1.100:11434  # GPU server IP
```

### Scenario: Shared Memory Server for Team

```
┌─────────────────┐
│  Dev Machine 1  │──┐
└─────────────────┘  │
                     │    ┌─────────────────┐
┌─────────────────┐  │    │  Memory Server  │
│  Dev Machine 2  │──┼───►│                 │
└─────────────────┘  │    │  MCP + Mem0     │
                     │    │  PostgreSQL     │
┌─────────────────┐  │    │  Neo4j          │
│  Dev Machine 3  │──┘    └─────────────────┘
└─────────────────┘
```

**Server Configuration:**

```bash
# Bind to all interfaces
MCP_HOST=0.0.0.0
MCP_PORT=8080
```

**Client Configuration (each dev machine):**

```json
// Claude Code MCP settings
{
  "mcpServers": {
    "mem0": {
      "type": "sse",
      "url": "http://memory-server.internal:8080/sse",
      "headers": {
        "X-MCP-Token": "user-specific-token",
        "X-MCP-UserID": "developer-name"
      }
    }
  }
}
```

---

## Cloud Deployment

### AWS Deployment

**Recommended Services:**
- **ECS/Fargate** for containers (mem0-server, mcp-server)
- **RDS PostgreSQL** with pgvector extension
- **Neptune** or self-managed Neo4j on EC2
- **EC2 with GPU** for Ollama (or use Bedrock/OpenAI)

**Cost Estimate (us-east-1):**
- RDS db.t3.medium: ~$30/month
- ECS Fargate (2 tasks): ~$50/month
- EC2 g4dn.xlarge (Ollama): ~$150/month
- Total: ~$230/month

### GCP Deployment

**Recommended Services:**
- **Cloud Run** for containers
- **Cloud SQL PostgreSQL** with pgvector
- **Neo4j Aura** (managed)
- **Vertex AI** or self-hosted Ollama on Compute Engine

### Azure Deployment

**Recommended Services:**
- **Azure Container Apps** for containers
- **Azure Database for PostgreSQL** (Flexible Server with pgvector)
- **Neo4j on Azure Marketplace**
- **Azure OpenAI Service** or VM with GPU for Ollama

---

## Configuration Templates

### Production .env Template

```bash
# Production configuration template
# Copy this to .env and fill in your values

# LLM Provider
LLM_PROVIDER=ollama  # or openai, anthropic

# Ollama (if using)
OLLAMA_BASE_URL=http://your-ollama-server:11434
OLLAMA_LLM_MODEL=qwen3:8b
OLLAMA_EMBEDDING_MODEL=qwen3-embedding:0.6b
OLLAMA_EMBEDDING_DIMS=1024

# PostgreSQL (production)
POSTGRES_HOST=your-postgres-host
POSTGRES_PORT=5432
POSTGRES_USER=mem0
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD
POSTGRES_DB=mem0
POSTGRES_COLLECTION_NAME=memories

# Neo4j (production)
NEO4J_URI=bolt://your-neo4j-host:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=CHANGE_ME_STRONG_PASSWORD

# Server Configuration
MCP_PORT=8080
MEM0_PORT=8000
LOG_LEVEL=WARNING

# Security
PROJECT_ID_MODE=auto
```

---

## Troubleshooting Migration Issues

### Issue: Embedding Dimension Mismatch

**Symptoms:** Error about vector dimensions when querying

**Solution:**
```sql
-- Check current dimension
SELECT vector_dims(embedding) FROM memories LIMIT 1;

-- If mismatch, reset and re-index
TRUNCATE memories;
```

### Issue: Neo4j Connection Timeout During Migration

**Symptoms:** Slow or hanging graph operations

**Solution:**
```bash
# Increase Neo4j memory
NEO4J_dbms_memory_heap_initial__size=512m
NEO4J_dbms_memory_heap_max__size=1g
```

### Issue: Authentication Tokens Not Working After Migration

**Symptoms:** 401 Unauthorized errors

**Solution:**
```sql
-- Verify tokens were imported correctly
SELECT token_prefix, user_id, is_active FROM auth_tokens;

-- Reactivate if needed
UPDATE auth_tokens SET is_active = true WHERE user_id = 'your_user';
```

---

## Next Steps

- [CONFIGURATION.md](./CONFIGURATION.md) - Detailed configuration options
- [SECURITY.md](./SECURITY.md) - Security best practices
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues and solutions
