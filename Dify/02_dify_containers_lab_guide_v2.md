# DIFY CONTAINERS — HANDS-ON LAB GUIDE
### Dify v1.13.0 | Developer Lab | Step-by-Step Exercises

---

## PRE-REQUISITES

Before starting this lab, ensure you have:
- Docker Desktop installed and running
- Docker Compose v2+ installed
- Dify v1.13.0 cloned from GitHub
- At least 8GB RAM available for Docker
- Ports 80, 5432, 6379, 8080 free on your machine

**Estimated time:** 2 hours for all labs

**How to check your Docker memory allocation:**
Docker Desktop → Settings → Resources → Memory → set to at least 8GB

---

## LAB 1 — START DIFY AND VERIFY ALL 12 CONTAINERS

**Goal:** Start the full Dify stack and understand what each status means.

**Steps:**

1. Navigate into the Dify docker directory
2. Copy `.env.example` to `.env`
3. Start all containers in detached mode using `docker compose up -d`
4. Run `docker compose ps` and observe the output

**What to verify:**

- `init_permissions` shows **Exited (0)** — this is correct and expected
- All other 11 containers show **Up** or **Healthy**
- `db_postgres` specifically shows **Healthy** — it has a health check

**If api shows "Restarting":**
Wait 30 seconds. Postgres was still initializing when api tried to connect.
Restart just the api container and check again.

**Open your browser:**
Navigate to `http://localhost` — you should see the Dify setup screen.

**Checkpoint questions:**
- Which container served the page you saw in your browser?
- Which container handled your browser's HTTP request first?
- Why does `init_permissions` show Exited but that is not a problem?

---

## LAB 2 — UNDERSTAND THE INIT CONTAINER

**Goal:** See what init_permissions actually did before exiting.

**Steps:**

1. View the logs of the init_permissions container
2. You will see a single command — a `chown` or `chmod` operation on directory paths
3. Identify which directories it fixed permissions on
4. Check those directories exist inside the api container

**Why this matters:**
The api and worker containers run as non-root users. Docker volumes are created as root. Without this init step, the api cannot write uploaded files to storage — your document uploads would fail silently.

**What to observe:**
The init container ran one command, succeeded, and exited. That is its entire lifecycle. It will not restart. It does not need to.

**Checkpoint question:**
What would happen to document uploads if init_permissions had exited with code 1 (failure) instead of code 0 (success)?

---

## LAB 3 — EXPLORE THE INTELLIGENCE LAYER

**Goal:** Understand the difference between api, worker, worker_beat, and plugin_daemon by looking at their running processes.

**Steps:**

1. Check the running process inside the api container
2. Check the running process inside the worker container
3. Check the running process inside the worker_beat container
4. Check the running process inside the plugin_daemon container

**What you will observe:**

- api → runs a gunicorn web server (HTTP server process)
- worker → runs `celery worker` (queue consumer process)
- worker_beat → runs `celery beat` (scheduler process)
- plugin_daemon → runs its own server process (different image entirely)

**Key observation:**
api and worker show very similar process names — same Docker image, different start command.
worker_beat is also the same image but runs the Beat scheduler.
plugin_daemon is a completely different image — `langgenius/dify-plugin-daemon`.

**Checkpoint question:**
api and worker use the same Docker image but behave completely differently. What is the single thing that makes them different at runtime?

---

## LAB 4 — WATCH NGINX ROUTING IN ACTION

**Goal:** See nginx directing traffic to the correct backend container.

**Steps:**

1. Open nginx container logs and keep them streaming in a terminal
2. In a browser, navigate to `http://localhost` — observe the nginx log entry
3. In a second terminal, make a direct API call to `http://localhost/v1/` — observe the log entry
4. Compare the two log lines — notice the different upstream targets

**What to observe:**
- Browser request to `/` → nginx sends to web container (port 3000)
- API call to `/v1/` → nginx sends to api container (port 5001)
- Both arrive at port 80 on nginx — routing happens based on URL path

**Try to reach api directly:**
Attempt to connect to `http://localhost:5001` in your browser.
You will get a connection refused — that port is not exposed to your host machine.
Only port 80 (nginx) is the door.

**Checkpoint question:**
What security benefit does having only one exposed port provide in a banking deployment?

---

## LAB 5 — WATCH A DOCUMENT GET INDEXED

**Goal:** Observe the worker container processing a Knowledge Base document.

**Steps:**

1. Open the worker container logs and keep them streaming
2. In the Dify console, create a new Knowledge Base
3. Upload a small PDF or text file
4. Watch the worker logs carefully

**What to observe in worker logs:**
- A task is received from the Redis queue
- Text extraction begins
- Chunking occurs
- Embedding API calls are made (multiple batches)
- Vectors written to Weaviate
- Chunk text written to Postgres
- Task completes — document status updates to "Indexed"

**Also watch:**
Open the Dify console Knowledge Base view. Watch the document status change from "Queued" → "Indexing" → "Indexed".

**Now stop the worker:**
Stop the worker container. Upload another document. Observe — the document stays in "Queued" forever. The api accepted the upload but has no one to process it.
Restart the worker — it picks up the queued job and processes it.

**Checkpoint question:**
The api accepted the document upload and returned success immediately, even though the worker was stopped. What does this tell you about how api and worker communicate?

---

## LAB 6 — OBSERVE WORKER_BEAT SCHEDULING

**Goal:** Understand what worker_beat does and confirm it is running.

**Steps:**

1. View the worker_beat container logs
2. Look for lines showing scheduled task registrations at startup
3. Look for periodic task firing messages as time passes

**What to observe:**
worker_beat logs show the schedule it has loaded — tasks and their intervals.
Every time a scheduled task fires, worker_beat puts a job into the Redis queue.
The regular worker picks it up and executes it.

**Confirm the schedule is loaded:**
In the logs, look for lines containing "beat" and task names like cleanup, statistics, or health check.

**What happens if you run two worker_beat containers:**
This is a thought experiment — do not do this in production.
Two Beat schedulers would fire every scheduled task twice. Double cleanup runs, double stats, duplicated jobs. This is why Beat is always a single instance.

**Checkpoint question:**
worker can be safely scaled to 4 replicas. worker_beat must always be exactly 1. What property of worker makes it safe to scale, and what property of worker_beat makes it unsafe?

---

## LAB 7 — EXPLORE THE PLUGIN DAEMON

**Goal:** Understand the plugin_daemon container and its role.

**Steps:**

1. View the plugin_daemon container logs at startup
2. Navigate to the Dify console → Settings → Plugins (or Marketplace if available)
3. Observe the plugin management interface
4. Check which plugins (if any) are installed

**What to observe in logs:**
plugin_daemon logs show it initializing its plugin runtime, loading any installed plugins, and exposing its internal API for dify-api to call into.

**Install a plugin (if Marketplace is accessible):**
Browse the Dify plugin marketplace. Install any available tool plugin. Watch plugin_daemon logs as it downloads and initializes the new plugin. Verify the new tool appears in the agent tool list.

**Simulate plugin_daemon failure:**
Stop the plugin_daemon container. Try to use a plugin-based tool in an agent. Observe the error. Restart plugin_daemon. Confirm the plugin works again.

**Checkpoint question:**
Why is it safer to run plugins inside plugin_daemon rather than inside the api container itself?

---

## LAB 8 — VERIFY REDIS AS THE JOB QUEUE

**Goal:** See Redis acting as the bridge between api and worker.

**Steps:**

1. Stop the worker container (not the worker_beat, not the api)
2. Connect to the Redis container and open redis-cli
3. Check the queue length — it should be 0 (empty)
4. Upload a document to a Knowledge Base
5. Check the queue length again — it should be 1 (job waiting)
6. Upload two more documents
7. Check the queue — it should be 3
8. Restart the worker container
9. Watch the queue drain as worker processes each job
10. Check the queue — it should return to 0

**What this demonstrates:**
api writes jobs to Redis. worker reads from Redis. They never communicate directly. Redis is the decoupling layer between them.

**Checkpoint question:**
If Redis restarted while you had 3 jobs queued (worker was still stopped), what would happen to those 3 jobs? What is the implication for production deployments?

---

## LAB 9 — SIMULATE CONTAINER FAILURES

**Goal:** Understand what breaks when specific containers go down — and what keeps working.

**Do these one at a time. Restart between each test.**

**Test 1 — Stop web**
- Stop web container
- Try opening the Dify console in browser → fails
- Make an API call to your deployed agent app → succeeds
- Result: Developer tooling broken. Users unaffected.

**Test 2 — Stop worker**
- Stop worker container
- Upload a document → stays "Queued" forever
- Chat with existing agent using current KB → works (existing knowledge intact)
- Result: New knowledge cannot be added. Existing agents still work.

**Test 3 — Stop weaviate**
- Stop weaviate container
- Ask your agent a question that needs document retrieval → "I don't know" or hallucination
- Ask your agent a general question not needing docs → may still work
- Result: RAG broken. Agent loses document knowledge.

**Test 4 — Stop sandbox**
- Stop sandbox container
- Run a workflow that has a Code Node → fails at that node
- Run a workflow with no Code Nodes → works fine
- Result: Code Nodes fail. Other workflow nodes unaffected.

**Test 5 — Stop plugin_daemon**
- Stop plugin_daemon container
- Use an agent with a plugin-based tool → that tool fails
- Use an agent with only built-in tools → works
- Result: Plugin capabilities broken. Core functionality intact.

**After each test:** Restart the stopped container. Confirm recovery.

**Checkpoint question:**
Rank the five containers you tested by impact severity — which failure hurts users most? Which is most silent?

---

## LAB 10 — EXPLORE THE DATABASE

**Goal:** See the tables that drive agent memory and execution.

**Steps:**

1. Connect to the Postgres container using psql
2. Connect to the dify database
3. List all tables with `\dt`
4. Query the apps table — find your agent app
5. Chat with your agent a few times
6. Query the messages table — find your conversation turns
7. Query the message_agent_thoughts table — see the agent's reasoning

**What to look for in message_agent_thoughts:**
- thought column: what the agent was thinking at each step
- tool column: which tool it chose to call (if any)
- tool_input column: what arguments it passed to the tool
- observation column: what the tool returned

This is the complete reasoning trace of your agent stored as structured data.

**Also explore:**
- dataset_segments table — see the chunk text from your indexed documents
- workflow_node_executions — run a workflow and see per-node logs appear here

**Checkpoint question:**
You can see every agent thought in message_agent_thoughts. How would you use this table to debug an agent that is selecting the wrong tool?

---

## LAB 11 — VERIFY SSRF PROTECTION

**Goal:** Confirm the SSRF proxy is blocking internal network access.

**Steps:**

1. In the Dify console, create a Workflow app
2. Add an HTTP Node
3. Set the URL to a private IP address — use something like `http://192.168.1.1` or `http://10.0.0.1`
4. Run the workflow
5. Observe — the HTTP Node returns a connection error
6. Check the ssrf_proxy logs — you will see the request was DENIED
7. Now change the URL to `https://httpbin.org/get` (a public test API)
8. Run the workflow again
9. Observe — the HTTP Node returns a successful JSON response

**What this proves:**
The proxy allows legitimate external API calls.
The proxy blocks any attempt to reach internal network addresses.

**Checkpoint question:**
In your DBS environment, core banking and treasury systems likely run on private IP ranges. Without the SSRF proxy, what risk would exist if a developer built a workflow with an HTTP Node?

---

## LAB 12 — FULL TRACE: FOLLOW ONE AGENT RUN

**Goal:** Watch all containers collaborate during a single agent response.

**Steps:**

1. Open logs for api, redis, weaviate, and db simultaneously in four terminal windows
2. Ensure your agent has a Knowledge Base attached with at least one indexed document
3. Ask the agent a question that requires document retrieval
4. Watch all four terminal windows

**Sequence to trace:**
- api log: request received, auth check
- redis log: cache hit for app configuration
- weaviate log: vector similarity search executed
- db log: conversation history query, chunk text fetch
- api log: LLM called, tokens streaming
- db log: new message turn saved

**After the run:**
In the Dify console, open the conversation log and expand the trace view.
Everything you saw in the raw container logs is now presented as a structured timeline.

**Checkpoint question:**
Count how many containers were involved in this single agent response. What does this tell you about the importance of each container in a production system?

---

## CLEAN UP

After completing all labs:

To stop all containers while keeping your data:
`docker compose stop`

To remove containers but keep volumes (data safe):
`docker compose down`

To remove everything including data (full reset):
`docker compose down -v`

**Keep your environment if continuing to the next training session.**

---

*Lab Guide v2.0 | Dify v1.13.0 | 12 Containers | Developer Level*
