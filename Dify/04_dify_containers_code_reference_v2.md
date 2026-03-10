# DIFY CONTAINERS — CODE REFERENCE
### Dify v1.13.0 | All Commands, Configs, and Scripts

---

## SECTION 1 — SETUP AND STARTUP

```bash
# Clone Dify
git clone https://github.com/langgenius/dify
cd dify/docker

# Create your environment file
cp .env.example .env

# Start all containers
docker compose up -d

# Check all 12 containers
docker compose ps
```

**Expected healthy state:**
```
NAME                         STATUS
docker-init_permissions-1    Exited (0)   ← correct — job done
docker-nginx-1               Up
docker-web-1                 Up
docker-api-1                 Up
docker-worker-1              Up
docker-worker_beat-1         Up
docker-plugin_daemon-1       Up
docker-db_postgres-1         Up (healthy)
docker-redis-1               Up
docker-weaviate-1            Up
docker-sandbox-1             Up
docker-ssrf_proxy-1          Up
```

```bash
# Fix: api crashes on first run (db still initializing)
docker compose restart api

# Fix: wait for db to be ready before restarting
docker compose logs db_postgres | grep "ready to accept connections"
docker compose restart api
```

---

## SECTION 2 — VIEWING LOGS

```bash
# Tail any container in real time
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f worker_beat
docker compose logs -f plugin_daemon
docker compose logs -f nginx
docker compose logs -f db_postgres
docker compose logs -f redis
docker compose logs -f weaviate
docker compose logs -f sandbox
docker compose logs -f ssrf_proxy
docker compose logs -f web

# View init container (already exited)
docker compose logs init_permissions

# Last 100 lines without following
docker compose logs --tail=100 api

# With timestamps
docker compose logs -f -t api

# Multiple containers simultaneously
docker compose logs -f api worker

# Watch all intelligence layer
docker compose logs -f api worker worker_beat plugin_daemon

# Watch all state layer
docker compose logs -f db_postgres redis weaviate
```

**Filtering for errors:**
```bash
docker compose logs api | grep -i "error\|exception\|traceback"
docker compose logs worker | grep -i "failed\|retry\|exception"
docker compose logs worker_beat | grep -i "error\|failed"
docker compose logs plugin_daemon | grep -i "error\|plugin\|failed"
docker compose logs ssrf_proxy | grep "DENIED"
docker compose logs sandbox | grep -i "permission\|denied\|timeout"
```

---

## SECTION 3 — CONTAINER INSPECTION

```bash
# See main process inside each container
docker compose exec api ps aux | head -5
docker compose exec worker ps aux | head -5
docker compose exec worker_beat ps aux | head -5
docker compose exec plugin_daemon ps aux | head -5
docker compose exec db_postgres ps aux | head -5
docker compose exec weaviate ps aux | head -5

# Live resource usage — all containers
docker stats

# One-time snapshot
docker stats --no-stream

# Environment variables in api
docker compose exec api env | sort

# Check which networks exist
docker network ls | grep docker
```

---

## SECTION 4 — DATABASE OPERATIONS

```bash
# Open psql shell
docker compose exec db_postgres psql -U postgres -d dify

# One-liner queries
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT COUNT(*) FROM apps;"
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT COUNT(*) FROM messages;"
```

**Useful queries inside psql:**

```sql
-- List all tables
\dt

-- Your agent apps
SELECT id, name, mode, created_at
FROM apps
ORDER BY created_at DESC LIMIT 10;

-- Recent conversations
SELECT id, app_id, created_at
FROM conversations
ORDER BY created_at DESC LIMIT 10;

-- Messages in a conversation
SELECT role, LEFT(content, 100) AS preview, token_count, created_at
FROM messages
WHERE conversation_id = 'your-conversation-id'
ORDER BY created_at;

-- Agent reasoning trace for a message
SELECT position, thought, tool,
       LEFT(tool_input::text, 100) AS input_preview,
       LEFT(observation, 100) AS obs_preview
FROM message_agent_thoughts
WHERE message_id = 'your-message-id'
ORDER BY position;

-- Workflow execution history
SELECT id, status, total_tokens, elapsed_time, created_at
FROM workflow_runs
ORDER BY created_at DESC LIMIT 10;

-- Per-node execution detail
SELECT node_type, status, elapsed_time,
       LEFT(inputs::text, 80) AS input_preview,
       LEFT(outputs::text, 80) AS output_preview
FROM workflow_node_executions
WHERE workflow_run_id = 'your-run-id'
ORDER BY created_at;

-- Knowledge Base documents and status
SELECT name, indexing_status, word_count, tokens, created_at
FROM dataset_documents
WHERE dataset_id = 'your-kb-id'
ORDER BY created_at;

-- Document chunks for a document
SELECT position, LEFT(content, 100) AS chunk_preview, tokens
FROM dataset_segments
WHERE document_id = 'your-document-id'
ORDER BY position;

-- Reset stuck indexing documents
UPDATE dataset_documents
SET indexing_status = 'waiting'
WHERE indexing_status = 'indexing';

-- Token usage by app
SELECT app_id,
       SUM(answer_tokens + prompt_tokens) AS total_tokens,
       COUNT(*) AS message_count
FROM messages
GROUP BY app_id
ORDER BY total_tokens DESC;

-- Installed plugins
SELECT name, version, status, created_at
FROM plugin_installations
ORDER BY created_at DESC;
```

**Backup and restore:**
```bash
# Full backup
docker compose exec db_postgres pg_dump -U postgres dify > dify_backup_$(date +%Y%m%d_%H%M%S).sql

# Compressed backup
docker compose exec db_postgres pg_dump -U postgres -Fc dify > dify_backup_$(date +%Y%m%d).dump

# Restore from SQL
docker compose exec -T db_postgres psql -U postgres dify < dify_backup_20240101.sql

# Restore from compressed
docker compose exec -T db_postgres pg_restore -U postgres -d dify dify_backup_20240101.dump

# Run migrations after upgrade
docker compose exec api flask db upgrade
```

---

## SECTION 5 — REDIS OPERATIONS

```bash
# Open redis-cli
docker compose exec redis redis-cli

# Health check
docker compose exec redis redis-cli ping

# Queue depths
docker compose exec redis redis-cli LLEN celery
docker compose exec redis redis-cli LLEN celery:dataset

# Memory usage
docker compose exec redis redis-cli INFO memory | grep used_memory_human

# All keys (use carefully in production)
docker compose exec redis redis-cli KEYS "*"

# Clear cache (does not affect job queues)
docker compose exec redis redis-cli FLUSHDB

# Check specific cached key
docker compose exec redis redis-cli GET "api_key:your-key-here"
```

**Enable Redis persistence — add to docker-compose.yaml:**
```yaml
redis:
  image: redis:6-alpine
  command: redis-server --appendonly yes --appendfsync everysec
  volumes:
    - redis_data:/data

volumes:
  redis_data:
```

---

## SECTION 6 — WEAVIATE OPERATIONS

```bash
# Health check
curl http://localhost:8080/v1/.well-known/ready

# List all collections (one per KB)
curl http://localhost:8080/v1/schema

# Check Weaviate memory
docker stats docker-weaviate-1 --no-stream
```

**GraphQL query to count chunks in a KB:**
```bash
curl -X POST http://localhost:8080/v1/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ Aggregate { Dataset_YOURKBID { meta { count } } } }"
  }'
```

---

## SECTION 7 — SANDBOX OPERATIONS

```bash
# Health check
curl http://localhost:8194/health

# Test Python code execution
curl -X POST http://localhost:8194/v1/sandbox/run \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: dify-sandbox" \
  -d '{
    "language": "python3",
    "code": "def main(loan: float, income: float) -> dict:\n    dti = round((loan * 0.085) / income, 2)\n    return {\"dti_ratio\": dti, \"high_risk\": dti > 0.5}",
    "inputs": {"loan": 2500000.0, "income": 800000.0},
    "timeout": 10
  }'
```

---

## SECTION 8 — PLUGIN DAEMON OPERATIONS

```bash
# Check plugin_daemon logs
docker compose logs -f plugin_daemon

# Check plugin_daemon health (port varies — check your .env)
curl http://localhost:5002/health

# List installed plugins via Dify API
curl http://localhost/v1/plugins \
  -H "Authorization: Bearer your-api-key"
```

---

## SECTION 9 — NGINX OPERATIONS

```bash
# Test routing to api
curl -s -o /dev/null -w "%{http_code}" http://localhost/v1/

# Test routing to web
curl -s -o /dev/null -w "%{http_code}" http://localhost/

# Reload nginx config without full restart
docker compose exec nginx nginx -s reload

# Check nginx error log
docker compose exec nginx cat /var/log/nginx/error.log

# Test SSRF proxy is blocking internal IPs
# (do this from inside the api container)
docker compose exec api curl -x http://ssrf_proxy:3128 http://192.168.1.1/
# Expected: connection denied
```

---

## SECTION 10 — SCALING

```bash
# Scale api for concurrent users
docker compose up -d --scale api=3

# Scale worker for heavy indexing
docker compose up -d --scale worker=4

# NEVER scale worker_beat — always exactly 1
# NEVER scale plugin_daemon without coordination

# Check scaled replicas
docker compose ps api
docker compose ps worker

# Scale back down
docker compose up -d --scale api=1
docker compose up -d --scale worker=1
```

---

## SECTION 11 — UPGRADE PROCEDURE

```bash
# Step 1: Pull new images
docker compose pull

# Step 2: Stop app containers (keep db, redis, weaviate running)
docker compose stop api worker worker_beat plugin_daemon web nginx

# Step 3: Run DB migrations BEFORE starting new api
docker compose run --rm api flask db upgrade

# Step 4: Start updated containers
docker compose up -d api worker worker_beat plugin_daemon web nginx

# Step 5: Verify
docker compose ps
docker compose logs -f api
```

---

## SECTION 12 — SWITCHING VECTOR STORES

**Switch to pgvector (recommended for air-gapped):**
```bash
# In .env
VECTOR_STORE=pgvector
# Remove or comment out weaviate from docker-compose.yaml

# Apply migrations to create pgvector tables
docker compose exec api flask db upgrade

# Re-queue all documents for re-indexing
docker compose exec db_postgres psql -U postgres -d dify -c \
  "UPDATE dataset_documents SET indexing_status='waiting';"

# Restart worker to process re-indexing
docker compose restart worker
```

**Switch to Qdrant:**
```bash
# In .env
VECTOR_STORE=qdrant
QDRANT_URL=http://qdrant:6333
```

Add to docker-compose.yaml:
```yaml
qdrant:
  image: qdrant/qdrant:latest
  volumes:
    - qdrant_data:/qdrant/storage
  restart: always
```

---

## SECTION 13 — AIR-GAPPED SETUP WITH OLLAMA

**Add Ollama to docker-compose.yaml:**
```yaml
ollama:
  image: ollama/ollama:latest
  ports:
    - "11434:11434"
  volumes:
    - ollama_models:/root/.ollama
  restart: always
```

```bash
# Pull models
docker compose exec ollama ollama pull llama3.2:8b
docker compose exec ollama ollama pull nomic-embed-text

# List available models
docker compose exec ollama ollama list
```

**In Dify console:**
Settings → Model Providers → Add → OpenAI-API-compatible
- Base URL: `http://ollama:11434/v1`
- API Key: `ollama`
- Model name: `llama3.2:8b`

**In .env for air-gapped:**
```bash
DISABLE_TELEMETRY=true
```

---

## SECTION 14 — PRODUCTION SECURITY HARDENING

```bash
# Generate SECRET_KEY
openssl rand -hex 32

# In .env — change all defaults
SECRET_KEY=your-generated-32-char-secret
DB_PASSWORD=your-strong-db-password
CODE_EXECUTION_API_KEY=your-sandbox-secret
```

**In docker-compose.yaml — remove direct port exposure for internal services:**
```yaml
# Comment out or remove these lines
# ports:
#   - "5432:5432"   # db_postgres
#   - "6379:6379"   # redis
#   - "8080:8080"   # weaviate
#   - "8194:8194"   # sandbox
```

**Generate self-signed SSL for development:**
```bash
mkdir -p docker/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout docker/nginx/ssl/key.pem \
  -out docker/nginx/ssl/cert.pem \
  -subj "/CN=dify.internal.yourdomain.com"
```

---

## SECTION 15 — BACKUP AND DISASTER RECOVERY

**Full backup script:**
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
DIR="/backups/dify/$DATE"
mkdir -p "$DIR"

echo "Backing up PostgreSQL..."
docker compose exec -T db_postgres pg_dump -U postgres dify > "$DIR/postgres.sql"

echo "Backing up uploaded files..."
docker cp docker-api-1:/app/storage "$DIR/storage" 2>/dev/null || true

echo "Backup complete: $DIR"
echo "Remember: also backup the weaviate_data Docker volume"
echo "Run: docker run --rm -v docker_weaviate_data:/data -v $DIR:/backup alpine tar czf /backup/weaviate.tar.gz /data"
```

**Backup Weaviate volume:**
```bash
docker run --rm \
  -v docker_weaviate_data:/data \
  -v /your/backup/path:/backup \
  alpine tar czf /backup/weaviate_$(date +%Y%m%d).tar.gz /data
```

**Check Weaviate/Postgres sync:**
```bash
# Count segments in Postgres with vector IDs
docker compose exec db_postgres psql -U postgres -d dify -t -c \
  "SELECT COUNT(*) FROM dataset_segments WHERE index_node_id IS NOT NULL;"

# Should approximately match total vectors in Weaviate
curl -s http://localhost:8080/v1/schema | python3 -c \
  "import sys,json; schema=json.load(sys.stdin); [print(c['class']) for c in schema.get('classes',[])]"
```

**Reset all stuck indexing jobs:**
```bash
docker compose exec db_postgres psql -U postgres -d dify -c \
  "UPDATE dataset_documents SET indexing_status='waiting' WHERE indexing_status='indexing';"
docker compose restart worker
```

---

*Code Reference v2.0 | Dify v1.13.0 | 12 Containers*
*All commands aligned to docker-compose container names from v1.13.0 output*
