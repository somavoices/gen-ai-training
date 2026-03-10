# DIFY CONTAINERS — REFERENCE DOCUMENT
### Dify v1.13.0 | Complete Reference | Keep This Alongside Your Deployment

---

## QUICK REFERENCE: ALL 12 CONTAINERS

| Container | Image | Port | Layer | Always Running? |
|---|---|---|---|---|
| init_permissions | busybox:latest | — | Init | No — exits after setup |
| nginx | nginx:latest | 80 (exposed) | Entry | Yes |
| web | langgenius/dify-web:1.13.0 | 3000 (internal) | Presentation | Yes |
| api | langgenius/dify-api:1.13.0 | 5001 (internal) | Intelligence | Yes |
| worker | langgenius/dify-api:1.13.0 | — | Intelligence | Yes |
| worker_beat | langgenius/dify-api:1.13.0 | — | Intelligence | Yes — exactly ONE |
| plugin_daemon | langgenius/dify-plugin-daemon:0.5.3 | — | Intelligence | Yes |
| db_postgres | postgres:15-alpine | 5432 (internal) | State | Yes |
| redis | redis:6-alpine | 6379 (internal) | State | Yes |
| weaviate | semitechnologies/weaviate:1.27.0 | 8080 (internal) | State | Yes |
| sandbox | langgenius/dify-sandbox:0.2.12 | 8194 (internal) | Safety | Yes |
| ssrf_proxy | ubuntu/squid:latest | 3128 (internal) | Safety | Yes |

---

## CONTAINER DETAIL REFERENCE

---

### init_permissions

**Image:** `busybox:latest`
**Expected status after startup:** `Exited (0)`

**Purpose:**
Runs once at platform startup to set correct file system permissions on shared Docker volumes. Ensures api, worker, and plugin_daemon (which run as non-root users) can read and write to the storage directories Docker created as root.

**Why it exists:**
Docker volumes are initialized with root ownership. Application containers in Dify run as unprivileged users for security. Without the permission fix, any operation that writes to shared storage — document uploads, plugin installs, file processing — silently fails.

**Lifecycle:**
Starts before all other containers. Runs one command. Exits. Never restarts. This is the correct and expected behavior.

**Failure signal:**
`Exited (1)` instead of `Exited (0)` means the permission fix failed. Other containers will experience file permission errors on storage operations. Check logs for the specific directory and error.

**Agentic AI relevance:**
Indirectly critical. Without correct permissions, document uploads fail, plugin installs fail, and any agent capability depending on file storage breaks.

---

### nginx

**Image:** `nginx:latest`
**Port exposed:** 80 (and 443 with SSL configured)

**Purpose:**
Reverse proxy and single entry point for all external traffic. Routes requests to the correct backend container based on URL path.

**Why it exists:**
- Flask (api) and Next.js (web) are application servers, not production-grade traffic handlers
- Single entry point simplifies firewall rules, SSL certificate management, and access logging
- Buffers slow client connections so backend containers are not blocked
- Configures long timeouts specifically for agentic workload durations

**Routing rules:**
| URL Pattern | Routed to |
|---|---|
| `/v1/*` | api:5001 |
| `/api/*` | api:5001 |
| `/console/api/*` | api:5001 |
| `/explore/*` | web:3000 |
| `/*` (all other) | web:3000 |

**Critical agentic AI setting:**
`proxy_read_timeout 3600s` — allows connections to remain open for up to one hour. Required for complex multi-tool agent runs that can take several minutes.

**Security role:**
Only port 80 is exposed externally. All other container ports (5001, 3000, 5432, 6379, 8080, 8194, 3128) are internal-only.

**Failure impact:**
All external access stops immediately. Internal containers continue running but are unreachable from outside.

**Recovery:** `docker compose restart nginx`

---

### web

**Image:** `langgenius/dify-web:1.13.0`
**Internal port:** 3000

**Purpose:**
Next.js frontend application. The browser-based developer console for building and managing Dify applications.

**Why it exists:**
Building agentic pipelines requires a visual interface. The workflow canvas, prompt lab, knowledge base management, plugin marketplace, and trace viewer cannot be usefully operated from a CLI.

**What it contains in v1.13.0:**
- Workflow canvas (visual DAG builder)
- Agent app builder
- Prompt Lab
- Knowledge Base upload and management
- Plugin Marketplace and management
- Logs, trace viewer, annotation panel
- App settings and API key management

**Important characteristic:**
dify-web has zero direct database access and makes no LLM calls. Every action is an HTTP call to dify-api. It is a presentation layer only.

**Failure impact:**
Developer console inaccessible. All deployed agent apps and API endpoints continue serving users without interruption. This is a developer tooling outage, not a user-facing outage.

**Recovery:** `docker compose restart web`

---

### api

**Image:** `langgenius/dify-api:1.13.0`
**Internal port:** 5001

**Purpose:**
Central application server. Orchestrates all agentic AI execution. The most critical container in the system.

**Why it exists:**
Agentic AI requires a runtime that can, within a single request lifecycle: authenticate the caller, load workflow state, execute a multi-step reasoning loop, invoke tools, query a vector database, inject memory, call an LLM, stream output token by token, and log every step. This is an application server responsibility.

**What it handles:**
- All REST API endpoints (`/v1/chat-messages`, `/v1/workflows/run`, etc.)
- ReAct and Function Calling agent execution loops
- Workflow DAG execution — node by node, variable passing, branching
- Full RAG pipeline — embed query, search Weaviate, fetch Postgres chunks, assemble context
- Conversation memory injection before every LLM call
- Streaming responses via Server-Sent Events
- Authentication, rate limiting, usage logging
- Communication with plugin_daemon for plugin tool calls

**Design principle:**
Stateless. All state lives in Postgres and Redis. This enables horizontal scaling — multiple api replicas behind nginx.

**Failure impact:**
Complete platform outage. Everything stops. No agent runs, no API responses, no console functionality.

**Recovery:** `docker compose restart api`

**Scale for load:** `docker compose up -d --scale api=3`

---

### worker

**Image:** `langgenius/dify-api:1.13.0` (same as api)
**Start command:** `celery -A app.celery worker`

**Purpose:**
Celery worker process for asynchronous background job processing.

**Why it exists:**
Some operations are too slow or resource-intensive to run inside a synchronous HTTP request:
- Document indexing: extract → chunk → embed → store in Weaviate. Takes 5–8 minutes for large documents.
- Batch re-indexing of Knowledge Bases
- Async workflow execution (non-streaming)
- Email notifications

**How it works:**
dify-api enqueues job messages to Redis. Worker consumes from that queue. Processes the job. Writes results to Postgres and Weaviate. Updates status in Postgres.

**Same image as api — why:**
Worker uses the same Python service classes as api (DocumentProcessor, EmbeddingService, etc.). Sharing one image means one bug fix deploys to both simultaneously with no version mismatch.

**Agentic AI relevance:**
Every document an agent can retrieve was indexed by this container. Worker builds the agent's knowledge base. Without a healthy worker, agents cannot gain new knowledge.

**Failure impact:**
Document uploads appear to succeed but never index. Knowledge Bases remain empty or stale. Async workflow runs queue up indefinitely.

**Recovery:** `docker compose restart worker`

**Scale for heavy indexing:** `docker compose up -d --scale worker=4`

---

### worker_beat

**Image:** `langgenius/dify-api:1.13.0` (same as api and worker)
**Start command:** `celery -A app.celery beat`

**Purpose:**
Celery Beat scheduler. Fires periodic recurring background tasks on a time-based schedule.

**Why it exists:**
The regular worker handles on-demand jobs triggered by user actions. Some tasks need to run regularly on a schedule regardless of user activity:
- Log pruning and cleanup (prevents unbounded database growth)
- Usage statistics aggregation
- Model provider health checks
- Expired session cleanup
- Scheduled report generation

**Critical constraint — always exactly one instance:**
Two Beat schedulers running simultaneously fire every scheduled job twice. Double cleanup runs, double statistics, duplicate notifications. Beat must never be scaled beyond one replica.

**Relationship to worker:**
worker_beat does not execute tasks itself. It puts scheduled task messages into the Redis queue. The regular worker picks them up and executes them. Beat schedules; worker executes.

**Failure impact:**
Scheduled maintenance stops running. Immediate user impact: none. Over days and weeks: log tables grow unbounded, usage stats become stale, expired data accumulates.

**Recovery:** `docker compose restart worker_beat`

---

### plugin_daemon

**Image:** `langgenius/dify-plugin-daemon:0.5.3-local`

**Purpose:**
Dedicated runtime for Dify's plugin ecosystem. Introduced in Dify 1.0 as a first-class feature.

**Why it exists:**
Before Dify 1.0, extending the platform required modifying core code. This created maintenance problems and made custom extensions fragile. Plugins solve this by providing a structured extension mechanism with runtime isolation.

If plugin code ran inside the api container:
- A crashing plugin crashes the entire api
- A memory-hungry plugin starves all agent runs
- Plugin versions are coupled to core Dify versions

The plugin_daemon runs plugins in isolation from the main api process.

**What plugins can provide in Dify 1.x:**
- New LLM model providers (bring your own model endpoint)
- New tool integrations for agents (custom API tools, enterprise integrations)
- Custom document processors for Knowledge Base ingestion
- New workflow node types
- Custom embedding models

**Plugin lifecycle:**
1. Developer or admin installs a plugin via console or API
2. plugin_daemon downloads and validates the plugin package
3. plugin_daemon initializes the plugin in its runtime
4. dify-api calls into plugin_daemon when an agent needs a plugin capability
5. plugin_daemon returns the result to dify-api

**Agentic AI relevance:**
Plugin-based tools extend what agents can do without touching core Dify code. Your custom banking tool integration — SWIFT message parser, core banking query, CIBIL lookup — can be packaged as a Dify plugin and hosted here.

**Failure impact:**
Any agent tool that is implemented as a plugin fails. Built-in tools and non-plugin capabilities continue working.

**Recovery:** `docker compose restart plugin_daemon`

---

### db_postgres

**Image:** `postgres:15-alpine`
**Internal port:** 5432

**Purpose:**
PostgreSQL relational database. Primary persistent store for all Dify application data.

**Why it exists:**
Agentic AI accumulates state constantly. Every conversation turn, every agent reasoning step, every document chunk, every workflow execution must be stored durably, consistently, and queryably. These are relational database guarantees.

**Key tables for agentic AI:**

| Table | Stores | Used for |
|---|---|---|
| apps | App config, instructions, tool list, model settings | Loading agent behavior at runtime |
| conversations | Session metadata | Scoping message history to users |
| messages | Every conversation turn, token counts, latency | Conversation memory injection |
| message_agent_thoughts | Per-cycle ReAct trace (thought, tool, observation) | Agent reasoning trace, debugging |
| dataset_segments | Document chunk text, position, token count | RAG context retrieval |
| dataset_documents | Document metadata, indexing status | Tracking KB document state |
| workflow_runs | Per-execution summary, status, total tokens | Execution history |
| workflow_node_executions | Per-node input, output, timing | Trace panel data |
| tool_providers | Custom tool definitions, credentials | Tool registry |
| plugin_configs | Installed plugin configurations | Plugin_daemon bootstrap |

**What it does NOT store:** Vector embeddings — those live in Weaviate. The two are linked by chunk ID.

**Agentic AI relevance:**
Contains all types of agent memory: behavioral (app config), episodic (messages), knowledge text (dataset_segments), procedural (workflow definitions), and execution history.

**Failure impact:**
dify-api and dify-worker crash immediately on startup. Complete platform outage.

**Recovery:** `docker compose restart db_postgres` → then restart api and worker

**Backup:** `docker compose exec db_postgres pg_dump -U postgres dify > backup.sql`

---

### redis

**Image:** `redis:6-alpine`
**Internal port:** 6379

**Purpose:**
In-memory data store serving simultaneously as message broker, application cache, and session state manager.

**Three distinct roles:**

**Role 1 — Celery Message Broker**
dify-api writes async task messages to Redis lists. worker and worker_beat read from those lists. Without Redis, the async task system has no transport layer — worker goes idle, worker_beat cannot deliver scheduled tasks.

**Role 2 — Application Cache**
Stores frequently read data to avoid hitting Postgres on every request:
- API key validation results (TTL: 60s)
- App configurations (TTL: 5 min — edits take up to 5 min to propagate live)
- Model provider configurations
- Rate limit counters per API key

**Role 3 — Streaming Session State**
Tracks active streaming agent runs — task ID, status, token buffer. Enables stream reconnection if client disconnects mid-response.

**Critical limitation:**
Data lives in RAM only by default. Container restart = all queued jobs lost. For production deployments, enable AOF (Append-Only File) persistence.

**Agentic AI relevance:**
Working memory for active workflow runs. Speed layer for all agent API calls. Coordination layer between api and worker. Without Redis, the async pipeline collapses.

**Failure impact:**
Worker goes idle (no queue). Streaming breaks. Performance degrades significantly. Rate limiting stops.

**Recovery:** `docker compose restart redis` → then restart worker and api

---

### weaviate

**Image:** `semitechnologies/weaviate:1.27.0`
**Internal port:** 8080

**Purpose:**
Purpose-built vector database. Stores high-dimensional embedding vectors and performs approximate nearest-neighbor (ANN) semantic similarity search.

**Why it exists:**
Finding the most semantically similar text across millions of document chunks requires ANN search over vector space. PostgreSQL stores data efficiently but cannot perform ANN search at scale. Weaviate uses the HNSW (Hierarchical Navigable Small World) graph index — built specifically for this problem. Retrieval time: milliseconds regardless of corpus size.

**What it stores:**
Per Knowledge Base, one collection. Per document chunk, one record:
- Embedding vector (1536 dimensions for OpenAI Ada-002, varies by model)
- Chunk ID (links to text in Postgres)
- Knowledge Base ID (for filtering)

**What it does NOT store:** The chunk text itself. Only the vector and identifiers.

**Retrieval flow:**
Query → embed (external API) → send vector to Weaviate → HNSW search → return top-k chunk IDs → fetch text from Postgres by IDs → assemble context → inject into LLM

**Agentic AI relevance:**
Makes RAG fast and semantic. Without it, agents cannot search your organization's documents. The agent's semantic memory lives entirely in this container.

**Failure impact:**
All Knowledge Base queries return empty. RAG-dependent agents hallucinate or respond with "I don't have information on that."

**Recovery:** `docker compose restart weaviate`

**Volume warning:**
Never delete `weaviate_data` volume without also re-indexing all documents. Weaviate vectors and Postgres chunk records are linked by ID — they must stay synchronized.

**Replaceable with:** pgvector (in Postgres), Qdrant, Pinecone, Milvus, OpenSearch

---

### sandbox

**Image:** `langgenius/dify-sandbox:0.2.12`
**Internal port:** 8194

**Purpose:**
Hardened, isolated container for executing Code Nodes in Dify workflows.

**Why it exists:**
Code Nodes run Python or JavaScript. That code can be written by users or generated by LLMs. Running untrusted code inside dify-api would be a critical security vulnerability — a bad script could crash the API, access the database, read secrets from environment variables, or open network connections to exfiltrate data.

The sandbox is a blast radius limiter. Problems stay inside it.

**Security controls:**
- Seccomp syscall filtering — blocks: `socket()` (no networking), `fork()` (no subprocesses), `execve()` (no shell), `mount()` (no filesystem manipulation)
- Network isolation — no access to Postgres, Redis, Weaviate, or the internet
- CPU and memory resource limits — runaway code cannot starve other containers
- Execution timeout — infinite loops are forcibly killed
- Process isolation — sandbox crash does not affect api

**Interface:**
dify-api sends HTTP POST: code + input variables + timeout.
Sandbox executes in isolation.
Returns: output variables + stdout + stderr + elapsed time.
That is the complete contract.

**Agentic AI relevance:**
Code Nodes are the mechanism for deterministic computation in agentic workflows — exact financial calculations, data transformation, API response parsing, list filtering. LLMs are unreliable for exact math. Code Nodes are not. Sandbox makes this safe for production.

**Failure impact:**
Every Code Node in every workflow returns a connection error. Workflows with Code Nodes fail at that step.

**Recovery:** `docker compose restart sandbox`

---

### ssrf_proxy

**Image:** `ubuntu/squid:latest`
**Internal port:** 3128

**Purpose:**
Squid HTTP proxy that intercepts and filters all outbound HTTP requests from dify-api — specifically HTTP Nodes in workflows and custom tool API calls.

**Why it exists:**
SSRF (Server-Side Request Forgery) is an attack where an adversary causes a server to make HTTP requests to unintended targets. In Dify, HTTP Nodes and tool calls make outbound HTTP requests. Without controls, a workflow could be configured to reach:
- Internal network services (private IP ranges)
- Cloud instance metadata endpoints (AWS: 169.254.169.254 → yields IAM credentials)
- Internal databases or admin panels

The proxy enforces that outbound HTTP from dify-api only reaches legitimate public internet addresses.

**Blocked destinations:**
- 10.0.0.0/8 (private class A)
- 172.16.0.0/12 (private class B)
- 192.168.0.0/16 (private class C)
- 127.0.0.0/8 (loopback)
- 169.254.0.0/16 (link-local / cloud metadata)
- ::1 (IPv6 loopback)

**All outbound HTTP from dify-api routes through this proxy:**
HTTP Nodes, custom tool calls, and external API calls all go through ssrf_proxy.

**Agentic AI relevance:**
HTTP Nodes are how agents call external tools. The proxy enforces that tool-use capability cannot reach internal infrastructure. In banking environments where Dify shares a network with core systems, this is a mandatory control.

**Failure impact:**
Most HTTP Nodes and external tool calls fail. ssrf_proxy is on a separate Docker network (`ssrf_proxy_network`) — connection failure here is explicit.

**Recovery:** `docker compose restart ssrf_proxy`

---

## DEPENDENCY AND FAILURE MATRIX

| Container fails | Immediate impact | User-facing impact |
|---|---|---|
| init_permissions | File ops may fail (if it failed at startup) | Uploads broken |
| nginx | All external access blocked | Total lockout |
| web | Console inaccessible | Developers only |
| api | All execution stops | Complete outage |
| worker | Indexing frozen | New knowledge unavailable |
| worker_beat | Scheduled tasks stop | Gradual degradation |
| plugin_daemon | Plugin tools fail | Plugin-dependent agents broken |
| db_postgres | api/worker crash | Complete outage |
| redis | Worker idle, streaming breaks | Performance + async broken |
| weaviate | RAG returns empty | Agents lose document knowledge |
| sandbox | Code Nodes fail | Affected workflows broken |
| ssrf_proxy | HTTP Nodes fail | External tool calls broken |

**Recovery order after full outage:**
db_postgres → redis → weaviate → sandbox → ssrf_proxy → api → worker → worker_beat → plugin_daemon → web → nginx

---

## DOCKER VOLUMES REFERENCE

| Volume | Container | Contents | Backup priority |
|---|---|---|---|
| db_data | db_postgres | All Postgres tables — everything | CRITICAL — daily |
| weaviate_data | weaviate | All vector embeddings | HIGH — must sync with db |
| dify_storage | api, worker | Raw uploaded files | MEDIUM |
| plugin_storage | plugin_daemon | Installed plugin packages | MEDIUM |
| (RAM only) | redis | Queue, cache, sessions | LOW — ephemeral |

---

## NETWORKS REFERENCE

Dify v1.13.0 creates two Docker networks:

**docker_default**
Main internal network. All containers communicate here.
api → db, redis, weaviate, sandbox, plugin_daemon

**docker_ssrf_proxy_network**
Isolated network for outbound HTTP.
api → ssrf_proxy → internet
Separated to enforce that only ssrf_proxy has outbound internet access.

---

## ENVIRONMENT VARIABLES QUICK REFERENCE

| Variable | Purpose | Default — change in production |
|---|---|---|
| SECRET_KEY | Flask session encryption | Must generate random value |
| DB_PASSWORD | Postgres password | difyai123456 — CHANGE THIS |
| REDIS_HOST | Redis location | redis |
| VECTOR_STORE | Vector DB choice | weaviate |
| WEAVIATE_ENDPOINT | Weaviate URL | http://weaviate:8080 |
| CODE_EXECUTION_ENDPOINT | Sandbox URL | http://sandbox:8194 |
| CODE_EXECUTION_API_KEY | Sandbox shared secret | dify-sandbox — CHANGE THIS |
| SSRF_PROXY_HTTP_URL | Outbound proxy | http://ssrf_proxy:3128 |
| PLUGIN_DAEMON_URL | Plugin daemon URL | http://plugin_daemon:5002 |
| DISABLE_TELEMETRY | Opt out usage data | false — set true for air-gap |

---

## REPLACEABLE COMPONENTS

| Default | Replace with | Reason |
|---|---|---|
| weaviate | pgvector | Simpler — all data in Postgres, one fewer container |
| weaviate | Qdrant | Better performance at large scale |
| weaviate | Pinecone | Fully managed cloud, no infra |
| db_postgres | External Postgres (RDS) | Managed HA, backups, replication |
| redis | External Redis (ElastiCache) | Managed persistence, clustering |
| File storage | Amazon S3 | Scalable, redundant |
| External LLMs | Ollama | Air-gapped, local inference |

**Air-gapped banking recommendation:**
- Replace weaviate with pgvector — one backup, one security perimeter
- Add Ollama — all LLM inference on-premise
- Use managed Postgres and Redis — leverage existing DBA infrastructure
- Set DISABLE_TELEMETRY=true

---

## OPERATIONS QUICK REFERENCE

| Task | Command |
|---|---|
| Start all | `docker compose up -d` |
| Stop all (keep data) | `docker compose stop` |
| Check all containers | `docker compose ps` |
| View container logs | `docker compose logs -f [name]` |
| Restart one container | `docker compose restart [name]` |
| Run DB migrations | `docker compose exec api flask db upgrade` |
| Backup database | `docker compose exec db_postgres pg_dump -U postgres dify > backup.sql` |
| Check Redis queue depth | `docker compose exec redis redis-cli LLEN celery` |
| Reset stuck indexing | UPDATE dataset_documents SET indexing_status='waiting' in psql |
| Scale api | `docker compose up -d --scale api=3` |
| Scale worker | `docker compose up -d --scale worker=4` |

---

*Reference Document v2.0 | Dify v1.13.0 | 12 Containers*
*Keep this document alongside your Dify deployment.*
