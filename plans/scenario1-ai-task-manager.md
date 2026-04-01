# AI Native Task Manager — Kubernetes Architecture Plan

**Document:** `scenario1-ai-task-manager.md`
**Version:** 1.0
**Date:** 2026-04-01
**Architect:** Claude Code (claude-sonnet-4-6)

---

## Table of Contents

1. Overview & Architecture Summary
2. Namespace Design
3. Pods & Workloads
4. Services & Exposure
5. Resource Requests & Limits
6. ConfigMaps
7. Secrets Management
8. Persistent Storage
9. RBAC Design
10. Inter-Service Communication
11. Network Policies
12. Observability
13. Autoscaling (HPA)
14. Summary Table

---

## 1. Overview & Architecture Summary

### Application Purpose

The AI Native Task Manager is a multi-component cloud-native application where users manage tasks through a React/Next.js frontend. An AI-powered agent autonomously processes, prioritizes, and acts on tasks using LLM APIs. A notification service delivers real-time and asynchronous alerts. A core backend API acts as the authoritative data layer.

### Design Philosophy

- **Defense in depth**: every boundary has its own network policy, RBAC scope, and secret rotation
- **Least privilege everywhere**: every workload has a dedicated ServiceAccount with minimal permissions
- **Stateless compute, stateful data**: all application workloads are Deployments; only PostgreSQL and Redis use StatefulSets
- **Separation of concerns by namespace**: four application namespaces plus shared infrastructure namespaces
- **Zero-trust networking**: default-deny NetworkPolicy in every namespace; egress is explicitly controlled, especially for the Todo Agent which calls external LLM APIs

### ASCII Communication Flow Diagram

```
EXTERNAL USERS (HTTPS :443)
         |
         v
+------------------+
|  Ingress (NGINX) |  <-- TLS termination, rate limiting, WAF annotations
+------------------+
         |
   +-----------+
   |           |
   v           v
+------+   +------+
|  UI  |   |  UI  |  (2–4 replicas, namespace: ui)
+------+   +------+
   |    \       |
   |     \      |
   v      \     v
+-------+  \  +-------------+
|Backend|   +->| Todo Agent  |  (1–3 replicas, namespace: ai-agent)
|  API  |<-----|             |
+-------+      +-------------+
   |   ^              |
   |   |              | (WebSocket back to UI via
   |   |              |  Notification Service or direct)
   v   |              v
+----------+    +-----------------+
| Postgres |    | Notification    |  (2 replicas, namespace: notifications)
| Redis    |    | Service         |
+----------+    +-----------------+
(namespace:          |
 data)               v
              +------+------+
              | UI (WS push)|   SMTP Relay (external egress)
              +------+------+
```

### Detailed Data Flow Narratives

**Task CRUD flow:** Browser -> Ingress -> UI (SSR Next.js) -> Ingress internal path OR direct backend call -> Backend API -> PostgreSQL. Cache reads/writes on Redis.

**AI Agent flow:** Todo Agent polls or receives task events via Redis Pub/Sub -> calls Anthropic/OpenAI API (external HTTPS egress) -> writes results back to Backend API -> Backend API publishes event -> Notification Service delivers in-app push.

**Notification flow:** Backend API publishes to Redis channel -> Notification Service subscriber picks up -> WebSocket push to connected browsers OR SMTP to user email.

**Session flow:** UI authenticates user -> Backend API issues JWT -> JWT stored client-side -> all subsequent calls carry Bearer token -> Backend API validates JWT (secret from Kubernetes Secret) -> Redis caches session metadata for fast lookups.

---

## 2. Namespace Design

### Namespace Inventory

| Namespace | Purpose | Isolation Rationale |
|---|---|---|
| `ui` | Frontend UI pods | Isolated from backend; only needs to call backend-api and notification-svc |
| `backend` | Backend API pods | Central hub; needs DB/Redis access; most sensitive namespace |
| `ai-agent` | Todo Agent pods | Needs external LLM API egress; isolated to contain any prompt-injection blast radius |
| `notifications` | Notification Service pods | Needs SMTP egress, WebSocket to clients; isolated from DB |
| `data` | PostgreSQL, Redis | Strictly data-layer; no external ingress, receives only from backend and ai-agent |
| `ingress-system` | NGINX Ingress Controller | Dedicated so RBAC for ingress is not mixed with app workloads |
| `monitoring` | Prometheus, Grafana, Loki, Jaeger | Observability stack; read access to cluster metrics |
| `cert-manager` | cert-manager for TLS | Cluster-scoped certificate automation |

### Rationale Deep Dive

**Why `data` is separate from `backend`:** If the `backend` namespace is compromised, the attacker still must overcome NetworkPolicy to reach PostgreSQL. Database credentials live in the `data` namespace as Secrets, not copied into `backend`. The `backend` pods receive only a connection string secret.

**Why `ai-agent` is isolated:** LLM agents are a novel attack surface. Prompt injection could cause the agent to attempt data exfiltration. By isolating in its own namespace, NetworkPolicy can restrict its egress to: (a) backend API only within cluster, (b) specific external LLM API CIDR/FQDN. No direct database access is ever granted to this namespace.

**Why `notifications` cannot talk to `data`:** Notification Service never needs raw DB access. It consumes events from Redis Pub/Sub channels only. The backend namespace writes the notification payloads; the notification service merely delivers them. This prevents a compromised notification pod from reading arbitrary user data.

**Namespace Labels (for NetworkPolicy selectors):**
- `ui`: `app.kubernetes.io/part-of=task-manager`, `team=frontend`
- `backend`: `app.kubernetes.io/part-of=task-manager`, `team=backend`
- `ai-agent`: `app.kubernetes.io/part-of=task-manager`, `team=ai`
- `notifications`: `app.kubernetes.io/part-of=task-manager`, `team=notifications`
- `data`: `app.kubernetes.io/part-of=task-manager`, `team=data`

---

## 3. Pods & Workloads

### 3.1 UI Interface (Namespace: `ui`)

**Workload Kind:** Deployment

**Rationale:** Stateless SSR application. Horizontal scaling is trivial; no stable network identity needed.

**Replicas:** 2 minimum (HPA scales to 4). Two replicas ensures availability during rolling updates.

**Pod Spec Details:**

- **Container image:** `task-manager/ui:latest` (Next.js production build, non-root user `nextjs:1001`)
- **Container port:** 3000 (HTTP, Next.js server)
- **Init container:** None required. Environment variables are injected via ConfigMap and Secrets at startup.
- **Liveness probe:** HTTP GET `/api/health` on port 3000, initialDelaySeconds: 15, periodSeconds: 20, failureThreshold: 3. This endpoint returns 200 if the Next.js server is accepting requests.
- **Readiness probe:** HTTP GET `/api/ready` on port 3000, initialDelaySeconds: 10, periodSeconds: 10, failureThreshold: 2. The `/api/ready` endpoint checks that the backend API is reachable (one internal HTTP call) before declaring readiness. This prevents routing traffic to a UI pod whose upstream is down.
- **Restart policy:** Always (default for Deployments).
- **terminationGracePeriodSeconds:** 30. Allows in-flight SSR requests to complete.
- **Update strategy:** RollingUpdate, maxUnavailable: 1, maxSurge: 1.
- **Pod Disruption Budget:** minAvailable: 1. Ensures at least one UI pod is always up during node drain.
- **Affinity:** Prefer spreading pods across availability zones using `topologySpreadConstraints` on `topology.kubernetes.io/zone`.
- **Security context:** `runAsNonRoot: true`, `runAsUser: 1001`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`. A writable `emptyDir` is mounted at `/tmp` for Next.js temp files.

---

### 3.2 Backend API (Namespace: `backend`)

**Workload Kind:** Deployment

**Rationale:** Stateless API server (FastAPI or Node.js). Scales horizontally. State is entirely in PostgreSQL and Redis.

**Replicas:** 2 minimum (HPA scales to 6).

**Pod Spec Details:**

- **Container image:** `task-manager/backend:latest` (Python 3.12 FastAPI or Node.js 22, non-root)
- **Container port:** 8000 (HTTP/REST)
- **Init container — database migration:** A dedicated `migration` init container runs `alembic upgrade head` (Python) or equivalent migration tool before the main container starts. This init container:
  - Uses the same database credentials Secret mounted as environment variables.
  - Has a `command` that exits 0 on success, non-zero on failure (blocking pod startup).
  - Is idempotent; Alembic tracks applied migrations.
  - Only the first pod to start runs migrations due to Alembic's migration lock table; subsequent pods' init containers will detect no pending migrations and exit 0 quickly.
  - Resource limits: 128Mi memory, 100m CPU (migration is brief).
- **Init container — wait-for-postgres:** Before the migration init container, a `wait-for-db` init container uses `pg_isready` in a loop to confirm PostgreSQL is accepting connections. This prevents migration failures due to DB not yet available during initial cluster startup.
- **Liveness probe:** HTTP GET `/health/live` port 8000, initialDelaySeconds: 20, periodSeconds: 15, failureThreshold: 3.
- **Readiness probe:** HTTP GET `/health/ready` port 8000, initialDelaySeconds: 10, periodSeconds: 10. The ready endpoint checks DB connectivity and Redis reachability.
- **Restart policy:** Always.
- **terminationGracePeriodSeconds:** 60. FastAPI/uvicorn processes in-flight requests; 60s is generous for API calls.
- **Update strategy:** RollingUpdate, maxUnavailable: 0, maxSurge: 2. Zero downtime is critical for the API hub.
- **Pod Disruption Budget:** minAvailable: 1.
- **Affinity:** Anti-affinity rule: no two backend pods on the same node (`podAntiAffinity` preferredDuringSchedulingIgnoredDuringExecution).
- **Security context:** `runAsNonRoot: true`, `runAsUser: 1000`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`.

---

### 3.3 Todo Agent (Namespace: `ai-agent`)

**Workload Kind:** Deployment

**Rationale:** Stateless Python worker. Each replica independently polls the task queue and processes tasks. No stable identity needed.

**Replicas:** 1 minimum (HPA scales to 3 based on queue depth). Over-scaling LLM agents is expensive; queue-depth-based scaling prevents unnecessary API cost.

**Pod Spec Details:**

- **Container image:** `task-manager/todo-agent:latest` (Python 3.12, LangChain/custom agent framework)
- **Container port:** 8080 (HTTP for internal health checks and admin endpoint; agent is primarily event-driven)
- **Init container:** `wait-for-backend` init container that polls `http://backend-api.backend.svc.cluster.local:8000/health/live` until 200 is returned. This ensures the agent does not start processing tasks before the API is available.
- **Liveness probe:** HTTP GET `/health` port 8080, initialDelaySeconds: 30, periodSeconds: 30, failureThreshold: 3. Longer initial delay because agent initialization (loading model configs, connecting to LLM) takes time.
- **Readiness probe:** HTTP GET `/ready` port 8080, initialDelaySeconds: 20, periodSeconds: 15. The `/ready` endpoint verifies that the agent can reach the Backend API and that it has a valid LLM API key (by testing a cheap API call like listing models, or simply verifying the key is non-empty and well-formed).
- **Restart policy:** Always.
- **terminationGracePeriodSeconds:** 120. LLM calls can take 10-30 seconds. 120s gives running tasks time to complete before the pod is killed. The agent must handle SIGTERM gracefully by stopping new task pickup and waiting for in-flight LLM calls.
- **Update strategy:** RollingUpdate, maxUnavailable: 0, maxSurge: 1.
- **Pod Disruption Budget:** minAvailable: 1 (if replicas >= 2). During single-replica operation, PDB is relaxed to allow node maintenance.
- **Security context:** `runAsNonRoot: true`, `runAsUser: 1000`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, `capabilities: drop: ALL`.

**Special consideration — LLM API egress:** The agent needs outbound HTTPS to `api.anthropic.com` and/or `api.openai.com` on port 443. This is the ONLY workload with external egress beyond the cluster. NetworkPolicy must explicitly allow this. Details in section 11.

---

### 3.4 Notification Service (Namespace: `notifications`)

**Workload Kind:** Deployment

**Rationale:** Stateless Node.js process managing WebSocket connections and SMTP delivery. WebSocket state can be re-established by clients; no affinity to specific pods needed if a reverse proxy/load balancer with sticky sessions is used for WebSocket (handled at Ingress level).

**Replicas:** 2 minimum (HPA scales to 4).

**Note on WebSocket scaling:** WebSocket connections are long-lived and stateful at the connection level. To scale Notification Service horizontally while keeping WebSocket delivery reliable, all notification pods subscribe to the same Redis Pub/Sub channel. When a notification event arrives on Redis, every pod receives it, but only the pod holding the relevant client's WebSocket connection delivers it. This is the fan-out pattern. No additional coordination is needed.

**Pod Spec Details:**

- **Container image:** `task-manager/notification-svc:latest` (Node.js 22, socket.io or `ws` library)
- **Container ports:** 3001 (HTTP + WebSocket upgrade)
- **Init container:** `wait-for-redis` init container that pings Redis using `redis-cli ping` in a loop until `PONG` is received.
- **Liveness probe:** HTTP GET `/health` port 3001, initialDelaySeconds: 10, periodSeconds: 15.
- **Readiness probe:** HTTP GET `/ready` port 3001, initialDelaySeconds: 8, periodSeconds: 10. Checks Redis Pub/Sub subscription is active.
- **Restart policy:** Always.
- **terminationGracePeriodSeconds:** 30. On SIGTERM, the service stops accepting new WebSocket connections and flushes any pending SMTP jobs.
- **Update strategy:** RollingUpdate, maxUnavailable: 1, maxSurge: 1. Clients reconnect automatically (socket.io reconnection logic).
- **Pod Disruption Budget:** minAvailable: 1.
- **Security context:** `runAsNonRoot: true`, `runAsUser: 1001`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`.

---

### 3.5 PostgreSQL (Namespace: `data`)

**Workload Kind:** StatefulSet

**Rationale:** PostgreSQL requires stable network identity for replication, stable storage attachment, and ordered startup/shutdown. StatefulSet provides all three.

**Replicas:** 3 (1 primary + 2 read replicas using streaming replication, managed by `patroni` or `Bitnami PostgreSQL` Helm chart patterns).

**Pod Spec Details:**

- **Container image:** `postgres:16` or `bitnami/postgresql:16`
- **Container port:** 5432
- **Init container — permissions fix:** Ensures the data directory (`/var/lib/postgresql/data`) is owned by the `postgres` user (UID 999). This is necessary when the PVC is provisioned with root ownership.
- **Init container — patroni config:** If using Patroni for HA, an init container writes the `patroni.yaml` config from a ConfigMap into the writable config directory.
- **Liveness probe:** `exec` command running `pg_isready -U postgres`, initialDelaySeconds: 30, periodSeconds: 10, failureThreshold: 6.
- **Readiness probe:** `exec` running `pg_isready -U postgres -d taskmanager`, initialDelaySeconds: 20, periodSeconds: 5. Pods that are not yet ready (during replica catch-up) are removed from the Service endpoints.
- **Restart policy:** Always.
- **terminationGracePeriodSeconds:** 60. PostgreSQL needs time to complete a clean checkpoint.
- **Update strategy:** RollingUpdate (StatefulSet), with `partition` set to control upgrade rollout.
- **Pod Disruption Budget:** minAvailable: 2. Never lose quorum; Patroni requires majority for leader election.
- **volumeClaimTemplates:** Each pod gets its own PVC (see section 8).

---

### 3.6 Redis (Namespace: `data`)

**Workload Kind:** StatefulSet

**Rationale:** Redis in cluster mode or sentinel mode needs stable pod names for peer discovery. StatefulSet is appropriate.

**Mode:** Redis Sentinel with 1 master + 2 replicas + 3 sentinel processes (or use Redis Cluster for sharding at scale). For this application, Redis Sentinel is sufficient and simpler.

**Replicas:** 3 (1 master, 2 replicas; Sentinel runs as a sidecar or separate StatefulSet).

**Pod Spec Details:**

- **Container image:** `redis:7`
- **Sidecar container:** `redis-sentinel` process as a sidecar, or a separate 3-replica Sentinel StatefulSet.
- **Container port:** 6379 (Redis), 26379 (Sentinel)
- **Init container:** Sets `vm.overcommit_memory = 1` via a privileged init container on the node (or handled at node level via DaemonSet). Also disables transparent huge pages if not done at node level.
- **Liveness probe:** `exec` running `redis-cli ping`, initialDelaySeconds: 15, periodSeconds: 10.
- **Readiness probe:** `exec` running `redis-cli ping`, initialDelaySeconds: 10, periodSeconds: 5.
- **terminationGracePeriodSeconds:** 30.
- **Pod Disruption Budget:** minAvailable: 2. Sentinel requires quorum of 2/3 to promote a new master.
- **volumeClaimTemplates:** Each Redis pod gets its own PVC for AOF persistence (see section 8).

---

## 4. Services & Exposure

### 4.1 External Ingress (Namespace: `ingress-system`)

**Controller:** NGINX Ingress Controller deployed as a Deployment (3 replicas) with a `LoadBalancer` Service fronting it. The cloud load balancer terminates at the NGINX layer; NGINX handles TLS termination using cert-manager-provisioned certificates.

**TLS:** cert-manager with Let's Encrypt ACME (DNS-01 challenge for wildcard, or HTTP-01 for specific hostnames). Certificates auto-renewed 30 days before expiry. TLS 1.2 minimum, TLS 1.3 preferred. Cipher suites restricted to AEAD ciphers.

**Ingress rules:**

| Host | Path | Backend Service | Notes |
|---|---|---|---|
| `app.taskmanager.com` | `/` | `ui-svc.ui:80` | Next.js SSR pages |
| `app.taskmanager.com` | `/api/v1/` | `backend-api-svc.backend:8000` | REST API (proxied through Ingress for CORS handling) |
| `app.taskmanager.com` | `/ws` | `notification-svc.notifications:3001` | WebSocket upgrade; `nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"` |
| `app.taskmanager.com` | `/notifications` | `notification-svc.notifications:3001` | HTTP long-poll fallback |

**Rate limiting annotations (on Ingress objects):**
- `nginx.ingress.kubernetes.io/limit-rps: "50"` on the `/api/v1/` Ingress — 50 requests/second per IP
- `nginx.ingress.kubernetes.io/limit-connections: "10"` on `/ws` — max 10 concurrent WebSocket connections per IP
- No rate limit on `/` (static SSR); CDN handles static asset caching

**Additional NGINX annotations:**
- `nginx.ingress.kubernetes.io/proxy-body-size: "1m"` — prevents large payload attacks on API
- `nginx.ingress.kubernetes.io/enable-cors: "true"` with explicit `cors-allow-origin`
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"` — HTTP to HTTPS redirect
- `nginx.ingress.kubernetes.io/use-regex: "true"` — for path-based routing

---

### 4.2 UI Service (Namespace: `ui`)

**Type:** ClusterIP

**Port:** 80 -> 3000 (targetPort on pod)

**Rationale:** UI is only accessed through the Ingress. No direct external exposure. The ClusterIP is used by the Ingress backend reference.

---

### 4.3 Backend API Service (Namespace: `backend`)

**Type:** ClusterIP

**Port:** 8000

**Rationale:** Backend API is accessed by UI (via Ingress path routing), by Todo Agent (direct cluster DNS), and by Notification Service (direct cluster DNS). Never directly exposed externally. ClusterIP provides stable DNS: `backend-api-svc.backend.svc.cluster.local`.

**Note:** A separate headless service (`backend-api-headless`) is created for internal service mesh discovery if a mesh like Istio or Linkerd is added later.

---

### 4.4 Todo Agent Service (Namespace: `ai-agent`)

**Type:** ClusterIP

**Port:** 8080

**Rationale:** The agent exposes an HTTP admin/health endpoint. The UI connects to it for real-time task status updates. This connection is via the Ingress (path `/agent/status`) or potentially via the Backend API acting as a proxy. The agent does not need a LoadBalancer; it is an internal service.

**Alternative pattern:** The UI does NOT connect directly to the agent. Instead, the UI polls the Backend API for task status, and the agent pushes updates to the Backend API, which in turn triggers the Notification Service. This is architecturally cleaner and recommended. The direct UI-to-Agent connection shown in the original diagram is replaced by this event-driven pattern to maintain clear separation.

---

### 4.5 Notification Service (Namespace: `notifications`)

**Type:** ClusterIP

**Port:** 3001

**Rationale:** Accessed externally only through Ingress (for WebSocket upgrades from browsers). Internal pod-to-pod communication uses cluster DNS. The Ingress handles WebSocket upgrade (`proxy-read-timeout: 3600`, `proxy-send-timeout: 3600`).

**Session Affinity:** The ClusterIP Service has `sessionAffinity: ClientIP` with `sessionAffinityConfig.clientIP.timeoutSeconds: 10800` (3 hours). This ensures a browser that established a WebSocket connection to a specific notification pod continues routing to that pod for subsequent HTTP requests (socket.io reconnection). For true WebSocket affinity at the Ingress level, the annotation `nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"` is used.

---

### 4.6 PostgreSQL Service (Namespace: `data`)

**Type:** ClusterIP (headless for StatefulSet pod DNS)

**Two services are created:**

1. **`postgres-svc`** (ClusterIP, port 5432): Points to the primary (write) pod via label selector `role=primary`. Patroni or pg-operator manages the `role=primary` label on the elected primary pod. Backend API connects here for writes.

2. **`postgres-read-svc`** (ClusterIP, port 5432): Points to all pods via label selector `app=postgres`. Used for read-only queries. Backend API's read replicas connect here.

3. **`postgres-headless`** (Headless): Required by StatefulSet for stable DNS names (`postgres-0.postgres-headless.data.svc.cluster.local`). Used by Patroni for peer communication.

---

### 4.7 Redis Service (Namespace: `data`)

**Two services:**

1. **`redis-master-svc`** (ClusterIP, port 6379): Routes to the Sentinel-elected master. Label `role=master` is managed by a Sentinel-aware sidecar or the Bitnami Redis chart.

2. **`redis-headless`** (Headless): For Sentinel peer discovery and StatefulSet DNS.

**Note:** Application code (Backend API, Notification Service) connects via the master service. Sentinel handles automatic failover. Application clients must handle connection retry and brief unavailability (typically 30-60 seconds during Sentinel-based failover). The Backend API uses a Redis client library with Sentinel support (e.g., `ioredis` for Node.js, `redis-py` with Sentinel for Python) that automatically reconnects to the new master.

---

## 5. Resource Requests & Limits

Resource sizing is based on expected workload characteristics and the principle that `requests` equal `limits` for memory (to avoid OOM evictions and ensure Guaranteed QoS class) and `requests` are lower than `limits` for CPU (Burstable QoS, allowing CPU burst without throttling normal operations).

### Resource Table

| Workload | Container | CPU Request | CPU Limit | Memory Request | Memory Limit | QoS Class | Justification |
|---|---|---|---|---|---|---|---|
| UI (Next.js) | `ui` | 100m | 500m | 256Mi | 512Mi | Burstable | SSR is CPU-intensive during render bursts; 512Mi for Node.js heap |
| Backend API | `backend-api` | 250m | 1000m | 512Mi | 1Gi | Burstable | FastAPI handles concurrent requests; DB connection pool uses memory |
| Backend API | `migration` (init) | 100m | 200m | 128Mi | 256Mi | Burstable | Short-lived; minimal resources needed |
| Backend API | `wait-for-db` (init) | 10m | 50m | 16Mi | 32Mi | Burstable | Trivial `pg_isready` loop |
| Todo Agent | `agent` | 500m | 2000m | 512Mi | 2Gi | Burstable | Python LLM processing is CPU-intensive; 2Gi for model context/embeddings in memory |
| Todo Agent | `wait-for-backend` (init) | 10m | 50m | 16Mi | 32Mi | Burstable | Trivial HTTP poll |
| Notification Svc | `notification` | 100m | 500m | 256Mi | 512Mi | Burstable | Node.js event loop; WebSocket state per connection ~1KB; 512Mi handles thousands of connections |
| PostgreSQL | `postgres` | 500m | 2000m | 1Gi | 4Gi | Burstable | PostgreSQL shared_buffers should be ~25% of available memory; 4Gi limit allows this headroom |
| PostgreSQL | `permissions-fix` (init) | 10m | 50m | 16Mi | 32Mi | Burstable | Simple `chown` |
| Redis | `redis` | 100m | 500m | 256Mi | 1Gi | Burstable | In-memory store; memory limit determines max dataset size |
| Redis | `redis-sentinel` (sidecar) | 50m | 100m | 64Mi | 128Mi | Burstable | Sentinel is lightweight; minimal CPU/memory |
| Ingress Controller | `nginx` | 200m | 1000m | 256Mi | 512Mi | Burstable | Handles all external traffic; spike headroom via CPU burst |

### Resource Design Notes

**Todo Agent — 2Gi memory limit:** LLM agent frameworks often load prompt templates, tool definitions, and maintain conversation context in memory. Python's memory management is less efficient than Go or Java. 2Gi provides sufficient headroom for LangChain's overhead, but agents should be implemented to not hold large contexts in-process (use the DB as the context store instead).

**PostgreSQL — 4Gi memory limit:** PostgreSQL's `shared_buffers` will be set to 1Gi (via ConfigMap parameter), `effective_cache_size` to 3Gi. The limit ensures the OS page cache within the container can supplement buffer cache. This is for a medium-sized deployment; production sizing should be based on actual dataset size.

**LimitRange in each namespace:** Each namespace has a LimitRange object that sets default requests/limits for any pod that does not specify them explicitly. This prevents unbounded resource consumption from misconfigured deployments.

**ResourceQuota per namespace:** Each namespace has a ResourceQuota to cap total CPU and memory consumption:

| Namespace | CPU Quota (requests) | CPU Quota (limits) | Memory Quota (requests) | Memory Quota (limits) |
|---|---|---|---|---|
| `ui` | 2 cores | 4 cores | 1Gi | 4Gi |
| `backend` | 4 cores | 8 cores | 4Gi | 8Gi |
| `ai-agent` | 4 cores | 8 cores | 4Gi | 8Gi |
| `notifications` | 1 core | 2 cores | 1Gi | 2Gi |
| `data` | 4 cores | 8 cores | 8Gi | 16Gi |

---

## 6. ConfigMaps

ConfigMaps store non-sensitive configuration. Every ConfigMap is named, scoped to its namespace, and mounted by specific pods. Update strategies are documented because changing a ConfigMap while pods are running requires deliberate action.

### CM-01: `ui-config` (Namespace: `ui`)

**Keys:**
- `NEXT_PUBLIC_API_BASE_URL`: `https://app.taskmanager.com/api/v1` — public API URL for client-side fetches
- `NEXT_PUBLIC_WS_URL`: `wss://app.taskmanager.com/ws` — WebSocket URL for real-time notifications
- `NEXT_PUBLIC_APP_NAME`: `AI Task Manager`
- `NEXT_PUBLIC_ENVIRONMENT`: `production`
- `NEXT_PUBLIC_FEATURE_AI_AGENT`: `true` — feature flag for AI agent UI components

**Mounted by:** `ui` Deployment pods, as environment variables (envFrom)

**Update strategy:** Rolling restart required (`kubectl rollout restart deployment/ui -n ui`). Next.js bakes `NEXT_PUBLIC_*` variables into the JavaScript bundle at build time, so these are build-time constants in the image. However, runtime-injectable variables that are NOT `NEXT_PUBLIC_` can be updated without image rebuild. The `NEXT_PUBLIC_*` variables here are noted for documentation; they are best embedded in the image during CI/CD. Non-public runtime config (API URLs for server-side fetches) can be ConfigMap-injected and a rolling restart applied.

---

### CM-02: `backend-config` (Namespace: `backend`)

**Keys:**
- `DATABASE_HOST`: `postgres-svc.data.svc.cluster.local`
- `DATABASE_READ_HOST`: `postgres-read-svc.data.svc.cluster.local`
- `DATABASE_PORT`: `5432`
- `DATABASE_NAME`: `taskmanager`
- `REDIS_HOST`: `redis-master-svc.data.svc.cluster.local`
- `REDIS_PORT`: `6379`
- `REDIS_DB_SESSIONS`: `0`
- `REDIS_DB_QUEUE`: `1`
- `REDIS_PUBSUB_CHANNEL_NOTIFICATIONS`: `notifications:events`
- `REDIS_PUBSUB_CHANNEL_AGENT`: `agent:tasks`
- `APP_PORT`: `8000`
- `APP_WORKERS`: `4` — uvicorn worker count
- `APP_LOG_LEVEL`: `info`
- `CORS_ALLOWED_ORIGINS`: `https://app.taskmanager.com`
- `JWT_ALGORITHM`: `HS256`
- `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`: `60`
- `JWT_REFRESH_TOKEN_EXPIRE_DAYS`: `7`
- `MAX_DB_CONNECTIONS`: `20` — SQLAlchemy pool size
- `MIGRATION_ENABLED`: `true` — controls whether init container runs migrations (set to `false` for read replicas)

**Mounted by:** `backend-api` Deployment and its init containers, as environment variables (envFrom + specific env var overrides)

**Update strategy:** Rolling restart required for most keys. Database host changes require careful coordination — update the ConfigMap, then restart with `kubectl rollout restart`. Connection pool settings are read at startup, so restart is required.

---

### CM-03: `agent-config` (Namespace: `ai-agent`)

**Keys:**
- `BACKEND_API_URL`: `http://backend-api-svc.backend.svc.cluster.local:8000`
- `LLM_PROVIDER`: `anthropic` (or `openai`)
- `LLM_MODEL`: `claude-opus-4-5` (or `gpt-4o`)
- `LLM_MAX_TOKENS`: `4096`
- `LLM_TEMPERATURE`: `0.7`
- `AGENT_POLL_INTERVAL_SECONDS`: `30`
- `AGENT_MAX_CONCURRENT_TASKS`: `3`
- `AGENT_TASK_TIMEOUT_SECONDS`: `90`
- `REDIS_HOST`: `redis-master-svc.data.svc.cluster.local`
- `REDIS_PORT`: `6379`
- `REDIS_PUBSUB_CHANNEL_AGENT`: `agent:tasks`
- `LOG_LEVEL`: `info`
- `ENABLE_TRACING`: `true`
- `OTEL_EXPORTER_OTLP_ENDPOINT`: `http://jaeger-collector.monitoring.svc.cluster.local:4317`

**Mounted by:** `todo-agent` Deployment pods

**Update strategy:** Rolling restart required. LLM model changes (e.g., `LLM_MODEL`) take effect on restart. Poll interval and concurrency settings could theoretically be hot-reloaded if the agent implements config file watching, but the simpler approach is rolling restart.

---

### CM-04: `notification-config` (Namespace: `notifications`)

**Keys:**
- `BACKEND_API_URL`: `http://backend-api-svc.backend.svc.cluster.local:8000`
- `REDIS_HOST`: `redis-master-svc.data.svc.cluster.local`
- `REDIS_PORT`: `6379`
- `REDIS_PUBSUB_CHANNEL_NOTIFICATIONS`: `notifications:events`
- `SMTP_HOST`: `smtp.sendgrid.net` (or internal relay)
- `SMTP_PORT`: `587`
- `SMTP_USE_TLS`: `true`
- `EMAIL_FROM`: `noreply@taskmanager.com`
- `EMAIL_FROM_NAME`: `Task Manager`
- `WS_PORT`: `3001`
- `WS_CORS_ORIGINS`: `https://app.taskmanager.com`
- `MAX_SMTP_RETRY_ATTEMPTS`: `3`
- `SMTP_RETRY_BACKOFF_SECONDS`: `5`

**Mounted by:** `notification-svc` Deployment pods

**Update strategy:** Rolling restart required. SMTP host changes are applied on restart.

---

### CM-05: `postgres-config` (Namespace: `data`)

**Keys:**
- `POSTGRES_DB`: `taskmanager`
- `POSTGRES_USER`: `taskmanager_user` (non-superuser application account)
- `POSTGRESQL_CONF_shared_buffers`: `1GB`
- `POSTGRESQL_CONF_effective_cache_size`: `3GB`
- `POSTGRESQL_CONF_max_connections`: `100`
- `POSTGRESQL_CONF_work_mem`: `64MB`
- `POSTGRESQL_CONF_wal_level`: `replica` — required for streaming replication
- `POSTGRESQL_CONF_max_wal_senders`: `5`
- `POSTGRESQL_CONF_hot_standby`: `on`
- `POSTGRESQL_CONF_log_min_duration_statement`: `500` — log slow queries over 500ms
- `REPLICATION_MODE`: `master` or `slave` (per pod, managed by Patroni)

**Mounted by:** `postgres` StatefulSet pods

**Update strategy:** PostgreSQL configuration changes (except `max_connections` and a few others) require at minimum `pg_reload_conf()` or a pod restart. For StatefulSet, use rolling update with partition. Changes to `max_connections` or `shared_buffers` require full pod restart. Use Patroni's config reload mechanism to apply changes without full restart where possible.

---

### CM-06: `redis-config` (Namespace: `data`)

**Keys:**
- `redis.conf`: full Redis configuration file content (mounted as a file via `configMap.items`)
  - `maxmemory 800mb`
  - `maxmemory-policy allkeys-lru` — evict LRU keys when memory is full (appropriate for cache usage; NOT appropriate for the queue/Pub/Sub data — see note below)
  - `appendonly yes` — AOF persistence enabled
  - `appendfsync everysec`
  - `save 900 1` — RDB snapshot
  - `save 300 10`
  - `save 60 10000`
  - `requirepass ""` — password set via environment variable from Secret, not config file
  - `loglevel notice`
  - `tcp-keepalive 60`
- `sentinel.conf`: Sentinel configuration
  - `sentinel monitor mymaster redis-0.redis-headless.data.svc.cluster.local 6379 2`
  - `sentinel down-after-milliseconds mymaster 5000`
  - `sentinel failover-timeout mymaster 30000`
  - `sentinel parallel-syncs mymaster 1`

**Note on maxmemory-policy:** `allkeys-lru` is safe for session/cache data. However, the task queue data in Redis DB 1 must NEVER be evicted. The recommended approach is to use separate Redis instances (or Redis keyspaces with `volatile-lru` for sessions with TTL set, and keys without TTL for queue items). Alternatively, use a different Redis database index and monitor memory carefully with alerts.

**Mounted by:** `redis` StatefulSet pods (file mount at `/etc/redis/redis.conf`)

**Update strategy:** Redis config reload: `redis-cli CONFIG REWRITE` + `CONFIG SET` for compatible parameters, or pod restart for others. Sentinel config changes require sentinel pod restart.

---

### CM-07: `ingress-nginx-config` (Namespace: `ingress-system`)

**Keys:**
- `proxy-body-size`: `1m`
- `proxy-read-timeout`: `3600`
- `proxy-send-timeout`: `3600`
- `keep-alive`: `75`
- `keep-alive-requests`: `100`
- `upstream-keepalive-connections`: `32`
- `log-format-upstream`: JSON format for structured logging to Loki
- `enable-opentracing`: `true`
- `opentracing-operation-name`: `HTTP $request_method $uri`

**Mounted by:** NGINX Ingress Controller (via controller's `--configmap` flag)

**Update strategy:** NGINX Ingress Controller watches this ConfigMap and applies changes via a reload of NGINX configuration (hot reload, no downtime).

---

### CM-08: `otel-config` (Namespace: `monitoring`)

**Keys:**
- `OTEL_SERVICE_NAME_UI`: `task-manager-ui`
- `OTEL_SERVICE_NAME_BACKEND`: `task-manager-backend`
- `OTEL_SERVICE_NAME_AGENT`: `task-manager-agent`
- `OTEL_SERVICE_NAME_NOTIFICATIONS`: `task-manager-notifications`
- `OTEL_EXPORTER_OTLP_ENDPOINT`: `http://otel-collector.monitoring.svc.cluster.local:4317`
- `OTEL_TRACES_SAMPLER`: `parentbased_traceidratio`
- `OTEL_TRACES_SAMPLER_ARG`: `0.1` — 10% sampling in production

**Mounted by:** All application Deployments that enable tracing

**Update strategy:** Rolling restart of each Deployment after update.

---

## 7. Secrets Management

### Secret Inventory

Every secret is documented with: name, namespace, keys, which workloads consume it, storage backend, rotation strategy, and breach response.

---

### SEC-01: `postgres-credentials` (Namespace: `data`)

**Keys:**
- `POSTGRES_PASSWORD`: superuser password (for admin tasks only)
- `POSTGRES_REPLICATION_PASSWORD`: used by replicas for streaming replication
- `POSTGRES_TASKMANAGER_PASSWORD`: application user password (less privileged)

**Consumed by:** `postgres` StatefulSet (all keys); `backend-api` init container (migration, needs `POSTGRES_TASKMANAGER_PASSWORD`)

**Note:** The Backend API deployment should NOT receive the superuser password. Only the migration init container needs elevated privileges, and only during schema migration.

**Storage approach:** Kubernetes Secret (opaque), base64-encoded. In production, these are sourced from an external secrets manager (HashiCorp Vault or AWS Secrets Manager) via the External Secrets Operator (ESO). ESO creates and refreshes the Kubernetes Secret from the external store on a defined `refreshInterval`.

**Rotation strategy:**
1. Generate a new password in Vault.
2. ESO detects the change and updates the Kubernetes Secret.
3. Use PostgreSQL's `ALTER USER taskmanager_user PASSWORD 'newpassword'` — this can be done online without downtime.
4. Rolling restart of `backend-api` Deployment picks up new password from updated Secret.
5. Old password is revoked in Vault after all pods have restarted (verified via rollout status).

**Breach response:** Immediately revoke the compromised password in PostgreSQL (`ALTER USER ... PASSWORD 'DISABLED'` or `REVOKE CONNECT`). Generate new password in Vault. Force-redeploy all consuming pods. Audit PostgreSQL logs for unauthorized queries. Rotate the Vault unseal keys if Vault itself is suspected.

---

### SEC-02: `backend-secrets` (Namespace: `backend`)

**Keys:**
- `DATABASE_URL`: full PostgreSQL connection string including password (`postgresql://taskmanager_user:PASSWORD@postgres-svc.data.svc.cluster.local:5432/taskmanager`)
- `REDIS_PASSWORD`: Redis auth password
- `JWT_SECRET_KEY`: HS256 signing key (256-bit random string)
- `JWT_REFRESH_SECRET_KEY`: separate key for refresh tokens

**Consumed by:** `backend-api` Deployment pods (all keys); `backend-api` migration init container (`DATABASE_URL` only)

**Storage approach:** External Secrets Operator sourcing from Vault. The `DATABASE_URL` is constructed from individual Vault secrets using ESO's template feature.

**Rotation strategy:**
- `REDIS_PASSWORD`: Update in Vault -> ESO updates Secret -> rolling restart. Redis requires `AUTH` with the new password; update Redis config simultaneously.
- `JWT_SECRET_KEY`: This is the most sensitive rotation. Rotating invalidates all existing user sessions. Strategy: (a) introduce a new key while keeping the old key valid for a grace period (implement multi-key JWT validation in the backend), (b) update Secret with both old and new keys as a list, (c) rolling restart, (d) after all pods have restarted and users have re-authenticated, remove old key and restart again. Total downtime: zero, but some users may need to re-login.
- **JWT expiry handling:** `JWT_ACCESS_TOKEN_EXPIRE_MINUTES: 60` (from CM-02). Short-lived access tokens limit the blast radius of token theft. Refresh tokens are rotated on each use.

**Breach response for `JWT_SECRET_KEY`:** Immediately rotate the key (accept some user session invalidation as a security necessity). Audit all API access logs for the period of suspected compromise. Consider invalidating all refresh tokens in the database (Redis or PostgreSQL-backed token blacklist).

---

### SEC-03: `agent-secrets` (Namespace: `ai-agent`)

**Keys:**
- `LLM_API_KEY`: Anthropic or OpenAI API key
- `BACKEND_API_KEY`: service-to-service API key for agent -> backend calls (avoids needing user-level JWT)
- `REDIS_PASSWORD`: same Redis password as backend (or a separate read-only Redis user)

**Consumed by:** `todo-agent` Deployment pods

**Storage approach:** External Secrets Operator from Vault. The LLM API key is stored in Vault with strict access policies: only the `ai-agent` service account's Vault role can read it.

**Rotation strategy for LLM API key:**
1. Generate a new API key in the LLM provider's dashboard.
2. Update Vault with the new key.
3. ESO updates the `agent-secrets` Kubernetes Secret.
4. Rolling restart of `todo-agent` Deployment.
5. Revoke the old API key in the LLM provider's dashboard after confirming new key is in use.
6. Monitor for LLM API errors post-rotation.

**Breach response for `LLM_API_KEY`:** Immediately revoke the key in Anthropic/OpenAI dashboard (this takes effect in seconds, cutting off any attacker). Generate new key. Update Vault. The agent will fail health checks and restart attempts will fail until ESO updates the Secret with the new key. The `wait-for-backend` init container prevents pods from starting in a broken state, but the readiness probe (which validates the LLM key) will prevent traffic to pods with bad keys. In practice, LLM API key revocation is the primary mitigation; Kubernetes restart handles the rest.

---

### SEC-04: `notification-secrets` (Namespace: `notifications`)

**Keys:**
- `SMTP_USERNAME`: SMTP relay username (e.g., SendGrid API key username)
- `SMTP_PASSWORD`: SMTP relay password or API key
- `REDIS_PASSWORD`: Redis auth password
- `PUSH_NOTIFICATION_VAPID_PRIVATE_KEY`: for web push notifications (VAPID)
- `PUSH_NOTIFICATION_VAPID_PUBLIC_KEY`: public half (could be a ConfigMap, but keeping with secrets for consistency)

**Consumed by:** `notification-svc` Deployment pods

**Storage approach:** External Secrets Operator from Vault.

**Rotation strategy for SMTP credentials:** Update in email provider dashboard -> Update in Vault -> ESO syncs -> rolling restart. SMTP sessions are short-lived, so rotation is straightforward.

**Breach response:** Revoke SMTP credentials immediately in provider dashboard. Audit sent-email logs for unauthorized sends. Rotate VAPID keys (this requires updating both server and client-side push subscriptions — push subscribers will need to re-subscribe, which happens automatically on next browser visit if the frontend handles the subscription renewal).

---

### SEC-05: `redis-credentials` (Namespace: `data`)

**Keys:**
- `REDIS_PASSWORD`: Redis AUTH password

**Consumed by:** `redis` StatefulSet pods (sets `requirepass`); also consumed by `backend-secrets`, `agent-secrets`, `notification-secrets` (which each have their own copy of the password value, sourced from the same Vault path)

**Storage approach:** Vault. ESO creates namespace-scoped copies in each consuming namespace.

**Rotation strategy:** Redis supports `CONFIG SET requirepass NEWPASSWORD` live without restart. The rotation sequence: (a) update Vault, (b) ESO updates all namespace secrets, (c) rolling restart of all consuming services, (d) live-update Redis with `CONFIG SET` to activate the new password simultaneously with the service restarts. Because ESO updates may not be perfectly simultaneous, there is a brief window where some pods have the old password and Redis still accepts it. This is mitigated by implementing a retry in the Redis client with exponential backoff.

---

### SEC-06: `tls-certificates` (Namespace: `ingress-system`)

**Keys:**
- `tls.crt`: TLS certificate chain (PEM)
- `tls.key`: Private key (PEM)

**Consumed by:** NGINX Ingress Controller (mounted via Ingress TLS spec)

**Storage approach:** cert-manager manages this Secret automatically. The Certificate resource references a ClusterIssuer (Let's Encrypt). cert-manager renews 30 days before expiry and updates the Secret in place. The Ingress Controller hot-reloads the new certificate within seconds.

**Breach response:** Revoke the certificate via Let's Encrypt's ACME revocation endpoint. cert-manager can be triggered to reissue immediately by deleting the Secret (cert-manager will recreate it). Update the Certificate resource if the key material itself was compromised (cert-manager will generate a new key pair).

---

### Secret Storage Architecture: External Secrets Operator

All secrets in production follow this flow:

```
HashiCorp Vault (or AWS Secrets Manager)
          |
          | (Vault Agent or ESO polling every 1h)
          v
External Secrets Operator (running in monitoring namespace)
          |
          | Creates/updates Kubernetes Secrets
          v
Kubernetes Secrets in each namespace
          |
          | Mounted as env vars or volume files
          v
Pod containers
```

**ExternalSecret refresh interval:** 1 hour for most secrets. For `tls-certificates`, cert-manager handles this independently. For `LLM_API_KEY`, consider shorter interval (15 minutes) given the sensitivity.

**Secret access audit:** Vault audit logging is enabled and forwarded to the Loki/monitoring stack. Every secret read is logged with the ServiceAccount identity that read it.

---

## 8. Persistent Storage

### 8.1 PostgreSQL Storage

**Provisioner:** Cloud-provider-specific (e.g., `kubernetes.io/gce-pd`, `ebs.csi.aws.com`, `disk.csi.azure.com`). In on-premises environments, use `rook-ceph` or `local-path-provisioner`.

**StorageClass name:** `postgres-ssd` — a custom StorageClass with:
- `provisioner`: cloud CSI driver
- `parameters.type`: `pd-ssd` (GCP) or `gp3` (AWS) — SSD-backed for IOPS
- `reclaimPolicy`: `Retain` — data is NOT deleted when PVC is deleted. This prevents accidental data loss.
- `volumeBindingMode`: `WaitForFirstConsumer` — delay provisioning until pod is scheduled, ensuring the PV is in the same availability zone as the pod.
- `allowVolumeExpansion`: `true` — allows PVC resize without downtime (CSI-supported)

**VolumeClaimTemplate (per PostgreSQL pod):**
- `storageClassName`: `postgres-ssd`
- `accessModes`: `ReadWriteOnce`
- `storage`: `50Gi` initial; expandable

**Volume mounts:**
- `/var/lib/postgresql/data` — PostgreSQL data directory

**Backup strategy:** PostgreSQL PVCs are backed up using Velero (with CSI snapshot support) on a daily schedule with 30-day retention. Additionally, `pg_basebackup` or WAL archiving to object storage (S3/GCS) provides point-in-time recovery. WAL archiving is configured via `postgresql.conf` (`archive_command`) referencing a cloud storage bucket.

---

### 8.2 Redis Storage

**StorageClass name:** `redis-ssd` — similar to `postgres-ssd` but can use slightly lower-performance SSD since Redis persistence is AOF/RDB on top of in-memory:
- `reclaimPolicy`: `Retain`
- `volumeBindingMode`: `WaitForFirstConsumer`
- `allowVolumeExpansion`: `true`

**VolumeClaimTemplate (per Redis pod):**
- `storageClassName`: `redis-ssd`
- `accessModes`: `ReadWriteOnce`
- `storage`: `10Gi` initial (sized for Redis AOF + RDB snapshots)

**Volume mounts:**
- `/data` — Redis data directory (AOF and RDB files)

**Note on Redis persistence:** Both AOF and RDB are enabled (see CM-06). AOF provides durability (up to 1-second data loss with `appendfsync everysec`). RDB provides fast restart. The 10Gi PVC accommodates AOF growth for reasonable dataset sizes. AOF rewrite is triggered automatically by Redis when the AOF file doubles in size.

**Backup strategy:** Velero CSI snapshots daily. Additionally, Redis can be configured to stream AOF to object storage using a sidecar or cron job.

---

## 9. RBAC Design

### Principle: One ServiceAccount Per Workload

Every Deployment/StatefulSet has a dedicated ServiceAccount. No workload uses the `default` ServiceAccount. The `default` ServiceAccount in every namespace has its `automountServiceAccountToken` set to `false`.

### ServiceAccount Inventory

| ServiceAccount | Namespace | Used By |
|---|---|---|
| `ui-sa` | `ui` | UI Deployment |
| `backend-api-sa` | `backend` | Backend API Deployment |
| `todo-agent-sa` | `ai-agent` | Todo Agent Deployment |
| `notification-svc-sa` | `notifications` | Notification Service Deployment |
| `postgres-sa` | `data` | PostgreSQL StatefulSet |
| `redis-sa` | `data` | Redis StatefulSet |
| `ingress-nginx-sa` | `ingress-system` | Ingress Controller |
| `cert-manager-sa` | `cert-manager` | cert-manager (managed by Helm chart) |
| `external-secrets-sa` | `monitoring` | External Secrets Operator |
| `prometheus-sa` | `monitoring` | Prometheus |

---

### RBAC Per Workload

**`ui-sa` (Namespace: `ui`)**
- Role: None (no Kubernetes API access needed)
- `automountServiceAccountToken: false`
- Rationale: The UI pod serves HTTP responses. It does not query the Kubernetes API. Token auto-mounting is explicitly disabled to prevent token theft via container breakout.

---

**`backend-api-sa` (Namespace: `backend`)**
- Role: None for normal operation
- Exception: A Role allowing `get` on Secrets in the `backend` namespace is needed if the backend reads Secrets directly via the Kubernetes API (NOT recommended; use environment variables injected by Kubelet instead). Prefer injecting secrets as env vars and NOT giving the pod Kubernetes API access to secrets.
- `automountServiceAccountToken: false`
- Rationale: Backend API reads its configuration from environment variables. No Kubernetes API access needed.

---

**`todo-agent-sa` (Namespace: `ai-agent`)**
- Role: None
- `automountServiceAccountToken: false`
- Rationale: Agent communicates with Backend API via HTTP. No Kubernetes API access needed. Strict isolation is especially important here given the LLM attack surface.

---

**`notification-svc-sa` (Namespace: `notifications`)**
- Role: None
- `automountServiceAccountToken: false`

---

**`postgres-sa` (Namespace: `data`)**
- Role: `get`, `list` on ConfigMaps in `data` namespace (for Patroni to read its configuration)
- RoleBinding: `postgres-config-reader` Role bound to `postgres-sa` in `data` namespace
- `automountServiceAccountToken: true` (required by Patroni for leader election via Kubernetes API — Patroni uses Kubernetes Endpoints or ConfigMaps as the DCS)

**Patroni-specific ClusterRole:** Patroni requires the ability to create/update/delete Endpoints and ConfigMaps (for leader election and config storage). This is namespaced to the `data` namespace:
- Resources: `configmaps`, `endpoints`
- Verbs: `create`, `get`, `list`, `watch`, `update`, `patch`, `delete`
- This is a Role (not ClusterRole) because it is scoped to `data` namespace.

---

**`redis-sa` (Namespace: `data`)**
- Role: `get` on ConfigMaps in `data` namespace (for reading `redis-config`)
- `automountServiceAccountToken: false`

---

**`ingress-nginx-sa` (Namespace: `ingress-system`)**
- ClusterRole: Required because NGINX Ingress Controller watches Ingress, Services, Endpoints, and Secrets cluster-wide (across namespaces).
- ClusterRole permissions:
  - `get`, `list`, `watch` on: `ingresses`, `services`, `endpoints`, `nodes`, `secrets`, `configmaps`, `ingressclasses`
  - `update` on `ingresses/status`
  - `create`, `patch` on `events`
- ClusterRoleBinding: bound to `ingress-nginx-sa`
- Rationale: Ingress controllers legitimately need cluster-wide read access to service/endpoint resources. The ClusterRole is read-only except for status updates and event creation, which are safe.

---

**`external-secrets-sa` (Namespace: `monitoring`)**
- ClusterRole permissions:
  - `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` on `secrets` across all namespaces — required to create/update synced Secrets
  - `get`, `list`, `watch` on `externalsecrets`, `clustersecretstores`, `secretstores`
- Rationale: ESO must create Secrets in multiple namespaces. This genuinely requires ClusterRole. The risk is mitigated by: (a) ESO itself is a trusted operator, (b) Vault RBAC controls which secrets ESO can actually access, (c) ESO's ServiceAccount is audited.

---

**`prometheus-sa` (Namespace: `monitoring`)**
- ClusterRole permissions:
  - `get`, `list`, `watch` on `pods`, `services`, `endpoints`, `nodes`, `nodes/proxy`, `nodes/metrics`
  - `get` on `/metrics` endpoint (non-resource URL)
- ClusterRoleBinding: bound to `prometheus-sa`
- Rationale: Prometheus scrapes metrics from all namespaces. Read-only cluster-wide access is the standard Prometheus RBAC pattern.

---

### Least-Privilege Checklist

- All application ServiceAccounts have `automountServiceAccountToken: false`
- No application pod runs as root (`runAsNonRoot: true`)
- No application pod has `allowPrivilegeEscalation: true`
- No application pod has host networking, host PID, or host IPC
- All containers drop `ALL` capabilities; only Patroni's PostgreSQL pods add `CHOWN` and `SETUID` capabilities (required by postgres binary)
- RBAC is namespace-scoped except for Ingress Controller and Prometheus (genuinely cluster-scoped needs)
- No wildcard (`*`) verbs or resources in any Role or ClusterRole
- Regular RBAC audit using `kubectl auth can-i --list --as system:serviceaccount:NAMESPACE:SA`

---

## 10. Inter-Service Communication

### Communication Matrix

| From | To | Protocol | DNS Name | Auth Method | Notes |
|---|---|---|---|---|---|
| Browser | Ingress | HTTPS (TLS 1.3) | `app.taskmanager.com` | None (public) | TLS terminated at Ingress |
| Ingress | UI | HTTP | `ui-svc.ui.svc.cluster.local:80` | None (internal trust) | In-cluster, NetworkPolicy protected |
| Ingress | Backend API | HTTP | `backend-api-svc.backend.svc.cluster.local:8000` | None (Ingress proxy) | JWT validation in Backend API |
| Ingress | Notification Svc | HTTP + WS | `notification-svc.notifications.svc.cluster.local:3001` | None (Ingress proxy) | WS upgrade handled by NGINX |
| UI (SSR) | Backend API | HTTP | `backend-api-svc.backend.svc.cluster.local:8000` | Bearer JWT (from user session) | SSR server-side calls |
| Backend API | PostgreSQL | TCP (libpq) | `postgres-svc.data.svc.cluster.local:5432` | Password (TLS-encrypted connection) | SSL mode: `require` |
| Backend API | Redis | TCP (RESP) | `redis-master-svc.data.svc.cluster.local:6379` | Password (`AUTH` command) | Redis AUTH + optional TLS |
| Backend API | Notification Svc | Redis Pub/Sub | (via Redis) | Redis AUTH | Backend publishes events; Notification Service subscribes |
| Todo Agent | Backend API | HTTP/REST | `backend-api-svc.backend.svc.cluster.local:8000` | Service-to-service API key (header `X-Agent-API-Key`) | Agent reads/writes tasks |
| Todo Agent | Anthropic API | HTTPS | `api.anthropic.com:443` | Bearer API key | External egress; only workload with external API access |
| Todo Agent | Redis | TCP (RESP) | `redis-master-svc.data.svc.cluster.local:6379` | Password | Reads task queue channel |
| Notification Svc | Redis | TCP (RESP) | `redis-master-svc.data.svc.cluster.local:6379` | Password | Subscribes to notification channel |
| Notification Svc | SMTP Relay | TCP (STARTTLS) | `smtp.sendgrid.net:587` | SMTP AUTH (username/password) | External egress |
| Prometheus | All pods | HTTP | Pod IPs via ServiceMonitor | None (cluster-internal scrape) | Metrics endpoints on `/metrics` |

### Service-to-Service Auth Deep Dive

**UI -> Backend API:** The browser holds a JWT (access token, 60-minute TTL) obtained during login. The Next.js SSR layer forwards this token to the Backend API via `Authorization: Bearer <token>`. The Backend API validates the JWT signature using `JWT_SECRET_KEY`. No mutual TLS is configured at the application layer (this could be added with a service mesh).

**Todo Agent -> Backend API:** The agent uses a static service-to-service API key stored in `agent-secrets` as `BACKEND_API_KEY`. The Backend API has a middleware that identifies requests from the agent via the `X-Agent-API-Key` header and grants agent-level permissions (broader read/write on tasks, no user-scoped filtering). This key has a separate rotation lifecycle from user JWTs.

**Backend API -> PostgreSQL:** TLS-encrypted connections with `sslmode=require`. The PostgreSQL pods are configured with self-signed certificates (generated at pod startup) or certificates from cert-manager. The Backend API trusts the PostgreSQL CA certificate (mounted from a Secret).

**Notification Service -> Backend API:** Not a direct HTTP call; the Notification Service is an event consumer. It subscribes to Redis Pub/Sub channels that the Backend API publishes to. No direct HTTP communication needed from Notification Service to Backend API, reducing the attack surface.

---

## 11. Network Policies

### Design Principle

Every namespace has a default-deny-all ingress AND egress policy applied first. Subsequent policies are additive allow rules. This means any communication not explicitly allowed is blocked.

---

### Namespace: `ui`

**Default deny all ingress + egress.**

**Allowed ingress:**
- From `ingress-system` namespace (label `app.kubernetes.io/name=ingress-nginx`) on port 3000 — allows Ingress Controller to reach UI pods

**Allowed egress:**
- To `backend` namespace, `backend-api-svc` pods, port 8000 — SSR server-side API calls
- To `notifications` namespace, `notification-svc` pods, port 3001 — if UI makes direct WebSocket connection (though this should go via Ingress; this rule is for internal pod-to-pod if direct path is used)
- To `kube-dns` (namespace `kube-system`, port 53 UDP/TCP) — DNS resolution
- To Kubernetes API server (port 443) — NOT allowed (UI pods have no API server access)

---

### Namespace: `backend`

**Default deny all ingress + egress.**

**Allowed ingress:**
- From `ingress-system` namespace on port 8000 — external API requests proxied by Ingress
- From `ui` namespace on port 8000 — direct SSR calls
- From `ai-agent` namespace on port 8000 — agent task CRUD calls
- From `monitoring` namespace (Prometheus scrape) on port 8000 — metrics

**Allowed egress:**
- To `data` namespace, PostgreSQL pods, port 5432 — database connections
- To `data` namespace, Redis pods, port 6379 — cache and Pub/Sub
- To `kube-dns` port 53 — DNS
- To `notifications` namespace on port 3001 — if Backend API calls Notification Service directly (but preferred via Redis Pub/Sub, so this may not be needed)

---

### Namespace: `ai-agent`

**Default deny all ingress + egress.**

**Allowed ingress:**
- From `monitoring` namespace on port 8080 — Prometheus scrape
- From `backend` namespace on port 8080 — if backend pushes tasks to agent via HTTP (otherwise no ingress needed)

**Allowed egress:**
- To `backend` namespace on port 8000 — agent reads/writes tasks
- To `data` namespace (Redis) on port 6379 — task queue
- To `kube-dns` port 53 — DNS
- **External egress (LLM APIs):** This is the critical special case. Two options:
  - **Option A (IP-based):** Allow egress to specific IP ranges for Anthropic/OpenAI. This is fragile as their IPs change. Use with a regularly-updated list.
  - **Option B (FQDN-based, with DNS-aware NetworkPolicy via Cilium):** If the cluster runs Cilium CNI, use `CiliumNetworkPolicy` with `toFQDNs` to explicitly allow `api.anthropic.com` and `api.openai.com` on port 443.
  - **Option C (Egress Gateway):** Route all ai-agent external egress through a dedicated Egress Gateway pod/node with static IP. The NetworkPolicy allows egress from ai-agent namespace to the Egress Gateway only. The Egress Gateway has its own egress policy to external APIs. This provides a single auditable exit point.
  - **Recommended:** Option C for production. Provides logging, rate limiting, and a stable egress IP that can be whitelisted on the LLM provider side.

---

### Namespace: `notifications`

**Default deny all ingress + egress.**

**Allowed ingress:**
- From `ingress-system` on port 3001 — WebSocket connections from browsers via Ingress
- From `monitoring` on port 3001 — Prometheus scrape

**Allowed egress:**
- To `data` namespace (Redis) on port 6379 — Pub/Sub subscription
- To external SMTP relay (port 587) — this requires either IP-based egress policy or Egress Gateway (same considerations as LLM API)
- To `kube-dns` port 53 — DNS

---

### Namespace: `data`

**Default deny all ingress + egress.**

**Allowed ingress to PostgreSQL:**
- From `backend` namespace on port 5432
- From `monitoring` namespace (postgres-exporter) on port 9187 — metrics exporter

**Allowed ingress to Redis:**
- From `backend` namespace on port 6379
- From `ai-agent` namespace on port 6379
- From `notifications` namespace on port 6379
- From `monitoring` namespace on port 9121 — Redis exporter

**Allowed egress from PostgreSQL:**
- To other PostgreSQL pods in `data` namespace on port 5432 — streaming replication between StatefulSet pods
- To `kube-dns` port 53

**Allowed egress from Redis:**
- To other Redis pods in `data` namespace on port 6379 — replication
- To other Redis pods on port 26379 — Sentinel communication
- To `kube-dns` port 53

---

### Namespace: `monitoring`

**Allowed ingress:**
- Grafana: from `ingress-system` on port 3000 — admin dashboard access
- Prometheus: no ingress (push-only via Alertmanager)
- Loki: no external ingress

**Allowed egress:**
- To all namespaces on all ports — Prometheus must scrape all pods
- To all namespaces on pod ports — Loki agent (Promtail) pushes logs from all pods
- External egress for Alertmanager webhooks (PagerDuty, Slack)

---

## 12. Observability

### 12.1 Logging

**Log aggregation stack:** Loki + Promtail (or Grafana Alloy as a replacement).

**Promtail DaemonSet:** Runs on every node, collects container logs from `/var/log/containers/`. Applies labels:
- `namespace`, `pod`, `container`, `app` (from pod labels)
- All applications output structured JSON logs; Promtail parses JSON and promotes log-level and trace-id as indexed labels.

**Log levels:**
- `backend-api`: INFO in production, DEBUG conditionally via `LOG_LEVEL` ConfigMap key (rolling restart required)
- `todo-agent`: INFO in production; LLM call/response details logged at DEBUG (never log raw LLM prompt/response in production to avoid PII leakage)
- `notification-svc`: INFO
- `postgres`: slow query log (>500ms, configured in CM-05), connection log
- `redis`: `notice` level (CM-06)

**Log retention:** Loki is configured with 30-day retention for production logs, 7-day for debug/trace logs.

**Sensitive data in logs:** All applications MUST NOT log JWT tokens, API keys, passwords, or PII (email addresses, task content). A log-scrubbing middleware is recommended in the Backend API.

---

### 12.2 Metrics

**Metrics stack:** Prometheus + Grafana.

**Prometheus ServiceMonitors (one per application):**
- `ui-monitor`: scrapes `/metrics` on port 3000 (Next.js custom metrics via `prom-client`)
- `backend-api-monitor`: scrapes `/metrics` on port 8000 (FastAPI instrumentation via `prometheus-fastapi-instrumentator`)
- `agent-monitor`: scrapes `/metrics` on port 8080 (custom metrics: tasks processed, LLM latency, queue depth)
- `notification-monitor`: scrapes `/metrics` on port 3001
- `postgres-monitor`: scrapes `postgres-exporter` sidecar on port 9187 (pg_stat_activity, pg_stat_replication, etc.)
- `redis-monitor`: scrapes `redis-exporter` sidecar on port 9121

**Custom metrics for HPA:**
- `agent_task_queue_depth`: number of pending tasks in Redis queue — exported by Todo Agent, used by HPA
- `backend_api_active_requests`: in-flight requests — used by HPA
- `ws_active_connections`: WebSocket connections per notification pod — used for capacity planning

**Key Grafana dashboards:**
- Task Manager Overview (request rates, error rates, latency p50/p95/p99)
- AI Agent Performance (LLM call latency, token usage, task throughput)
- Database Health (connections, replication lag, slow queries)
- Infrastructure (pod CPU/memory, PVC usage)

**PrometheusRules (alerting):**
- `BackendAPIHighErrorRate`: error rate > 5% for 5 minutes
- `PostgreSQLReplicationLag`: replica lag > 30 seconds
- `RedisMemoryHigh`: used memory > 80% of `maxmemory`
- `AgentLLMHighLatency`: p95 LLM call latency > 30 seconds
- `PodCrashLooping`: pod restart count > 5 in 10 minutes
- `PVCUsageHigh`: PVC usage > 85% of capacity
- `CertificateExpiringSoon`: cert-manager certificate expires in < 14 days

---

### 12.3 Distributed Tracing

**Tracing stack:** OpenTelemetry Collector + Jaeger (or Tempo for Loki-integrated setup).

**Trace propagation:** W3C Trace Context headers (`traceparent`, `tracestate`) propagated across all service calls. The Ingress Controller injects the initial trace context. All application services use OTEL SDKs to continue the trace.

**Instrumentation:**
- `backend-api`: auto-instrumented via `opentelemetry-fastapi-instrumentator`; custom spans around DB queries and Redis calls
- `todo-agent`: custom spans for each LLM API call, including `llm.model`, `llm.token_count` attributes
- `notification-svc`: spans for WebSocket push events and SMTP send operations

**Sampling:** 10% in production (CM-08 `OTEL_TRACES_SAMPLER_ARG: 0.1`). 100% sampling in staging. Error traces are always sampled (using `parentbased_traceidratio` sampler, which respects parent's sampling decision, plus a custom error-sampling rule).

**Jaeger storage:** Elasticsearch or Cassandra backend for Jaeger spans. Configured with 7-day retention.

---

### 12.4 Alerting

**Alertmanager** routes alerts to:
- PagerDuty (P1/P2 alerts: service down, data loss risk, security events)
- Slack `#task-manager-alerts` (P3/P4 alerts: high latency, elevated error rate)
- Email to on-call team

**Inhibition rules:** If `PodCrashLooping` is firing for `backend-api`, inhibit `BackendAPIHighErrorRate` (it's the same incident). If cluster-wide node failure is detected, inhibit pod-level alerts.

---

## 13. Autoscaling (HPA)

### HPA Design Philosophy

All HPAs use `stabilizationWindowSeconds` to prevent flapping. Scale-up is aggressive (short window); scale-down is conservative (long window) to avoid prematurely terminating connections.

---

### HPA-01: UI Deployment

**Namespace:** `ui`

**Min replicas:** 2 | **Max replicas:** 4

**Scale triggers:**
- CPU utilization > 70% (averaged across all UI pods) — primary trigger; Next.js SSR is CPU-bound during heavy page rendering
- Memory utilization > 80% — secondary trigger

**Scale-up stabilization window:** 30 seconds — scale up quickly under load spike
**Scale-down stabilization window:** 300 seconds (5 minutes) — avoid premature scale-down after a traffic burst

**Behavior:**
- Scale-up: add 1 pod per 30 seconds (to avoid thundering herd on the backend)
- Scale-down: remove 1 pod per 5 minutes

---

### HPA-02: Backend API Deployment

**Namespace:** `backend`

**Min replicas:** 2 | **Max replicas:** 6

**Scale triggers:**
- CPU utilization > 65% — FastAPI under concurrent request load
- Custom metric `backend_api_active_requests > 50` (per pod) — this is a more accurate signal than CPU for an I/O-bound API that spends most time waiting on DB
- HTTP request queue depth (if using KEDA instead of native HPA) — KEDA can scale based on NGINX Ingress active connections metric

**Scale-up stabilization window:** 15 seconds
**Scale-down stabilization window:** 600 seconds (10 minutes) — API has long-lived connections (DB pool); conservative scale-down avoids connection churn

---

### HPA-03: Todo Agent Deployment

**Namespace:** `ai-agent`

**Min replicas:** 1 | **Max replicas:** 3

**Scale triggers:**
- Custom metric `agent_task_queue_depth > 5` (tasks waiting in Redis queue) — the most meaningful signal; more tasks in queue = more agent replicas needed
- CPU utilization > 80% — LLM processing is CPU-bound

**Scale-up stabilization window:** 60 seconds — LLM API calls are expensive; don't over-provision
**Scale-down stabilization window:** 180 seconds — allow in-flight 90-second tasks to complete before scaling down

**Note on KEDA:** The custom metric `agent_task_queue_depth` requires either a custom metrics adapter or KEDA. KEDA's `RedisListLengthScaler` or `RedisPubSubScaler` can directly integrate with the Redis queue to trigger scaling without a custom metrics pipeline. Recommend KEDA for the ai-agent HPA.

---

### HPA-04: Notification Service Deployment

**Namespace:** `notifications`

**Min replicas:** 2 | **Max replicas:** 4

**Scale triggers:**
- Custom metric `ws_active_connections > 500` per pod — WebSocket connections are the primary resource consumer; each connection holds memory and an open file descriptor
- Memory utilization > 70% — secondary trigger

**Scale-up stabilization window:** 30 seconds
**Scale-down stabilization window:** 600 seconds — WebSocket connections are long-lived; do not scale down while connections are active (graceful drain must complete)

**Note on scale-down:** The Notification Service must implement a graceful connection drain on SIGTERM: (a) stop accepting new WebSocket connections, (b) notify existing clients to reconnect (via a `reconnect` event), (c) wait for clients to disconnect (up to `terminationGracePeriodSeconds: 30`). Socket.io handles this with `io.close()`.

---

### No HPA for StatefulSets (PostgreSQL, Redis)

PostgreSQL and Redis are NOT autoscaled via HPA. Their replica counts are fixed:
- PostgreSQL: 3 replicas (1 primary + 2 read replicas). Adding replicas requires Patroni coordination and is a manual operational decision.
- Redis Sentinel: 3 replicas. Sentinel quorum requires odd numbers; adding replicas is a manual operation.

Vertical scaling (adjusting resource requests/limits) is done manually with careful coordination for stateful workloads.

---

## 14. Summary Table — All Kubernetes Objects

| Kind | Name | Namespace | Purpose |
|---|---|---|---|
| Namespace | `ui` | cluster | UI workload isolation |
| Namespace | `backend` | cluster | Backend API isolation |
| Namespace | `ai-agent` | cluster | AI Agent isolation |
| Namespace | `notifications` | cluster | Notification Service isolation |
| Namespace | `data` | cluster | Database/cache isolation |
| Namespace | `ingress-system` | cluster | Ingress controller isolation |
| Namespace | `monitoring` | cluster | Observability stack |
| Namespace | `cert-manager` | cluster | TLS certificate automation |
| Deployment | `ui` | `ui` | Next.js SSR frontend |
| Deployment | `backend-api` | `backend` | FastAPI REST API |
| Deployment | `todo-agent` | `ai-agent` | LLM-backed AI task agent |
| Deployment | `notification-svc` | `notifications` | WebSocket + email notifications |
| Deployment | `ingress-nginx` | `ingress-system` | NGINX Ingress Controller |
| Deployment | `external-secrets-operator` | `monitoring` | ESO for Vault secret sync |
| Deployment | `otel-collector` | `monitoring` | OpenTelemetry collector |
| Deployment | `prometheus` | `monitoring` | Metrics scraping and alerting |
| Deployment | `grafana` | `monitoring` | Metrics/log visualization |
| Deployment | `loki` | `monitoring` | Log aggregation |
| Deployment | `jaeger` | `monitoring` | Distributed tracing |
| Deployment | `alertmanager` | `monitoring` | Alert routing |
| StatefulSet | `postgres` | `data` | PostgreSQL primary + replicas |
| StatefulSet | `redis` | `data` | Redis master + replicas |
| StatefulSet | `redis-sentinel` | `data` | Redis Sentinel quorum |
| DaemonSet | `promtail` | `monitoring` | Log collection from all nodes |
| Service | `ui-svc` | `ui` | ClusterIP for UI pods |
| Service | `backend-api-svc` | `backend` | ClusterIP for Backend API |
| Service | `backend-api-headless` | `backend` | Headless for future service mesh |
| Service | `todo-agent-svc` | `ai-agent` | ClusterIP for Agent health/admin |
| Service | `notification-svc` | `notifications` | ClusterIP + WS affinity |
| Service | `postgres-svc` | `data` | ClusterIP to primary PostgreSQL |
| Service | `postgres-read-svc` | `data` | ClusterIP to all PostgreSQL pods |
| Service | `postgres-headless` | `data` | Headless for StatefulSet DNS |
| Service | `redis-master-svc` | `data` | ClusterIP to Redis master |
| Service | `redis-headless` | `data` | Headless for StatefulSet + Sentinel DNS |
| Service | `ingress-nginx-svc` | `ingress-system` | LoadBalancer for external traffic |
| Ingress | `ui-ingress` | `ui` | Route `/` to UI pods |
| Ingress | `api-ingress` | `backend` | Route `/api/v1/` to Backend API |
| Ingress | `ws-ingress` | `notifications` | Route `/ws` with WebSocket upgrade |
| Ingress | `grafana-ingress` | `monitoring` | Admin dashboard access |
| ConfigMap | `ui-config` | `ui` | UI runtime configuration |
| ConfigMap | `backend-config` | `backend` | Backend API configuration |
| ConfigMap | `agent-config` | `ai-agent` | Agent configuration + LLM settings |
| ConfigMap | `notification-config` | `notifications` | Notification Service config |
| ConfigMap | `postgres-config` | `data` | PostgreSQL tuning parameters |
| ConfigMap | `redis-config` | `data` | Redis + Sentinel configuration |
| ConfigMap | `ingress-nginx-config` | `ingress-system` | NGINX global config |
| ConfigMap | `otel-config` | `monitoring` | OpenTelemetry configuration |
| Secret | `postgres-credentials` | `data` | PostgreSQL passwords |
| Secret | `backend-secrets` | `backend` | DB URL, Redis password, JWT keys |
| Secret | `agent-secrets` | `ai-agent` | LLM API key, service API key |
| Secret | `notification-secrets` | `notifications` | SMTP credentials, VAPID keys |
| Secret | `redis-credentials` | `data` | Redis AUTH password |
| Secret | `tls-certificates` | `ingress-system` | TLS cert/key for app.taskmanager.com |
| ServiceAccount | `ui-sa` | `ui` | Identity for UI pods |
| ServiceAccount | `backend-api-sa` | `backend` | Identity for Backend API pods |
| ServiceAccount | `todo-agent-sa` | `ai-agent` | Identity for Agent pods |
| ServiceAccount | `notification-svc-sa` | `notifications` | Identity for Notification pods |
| ServiceAccount | `postgres-sa` | `data` | Identity for PostgreSQL pods (Patroni) |
| ServiceAccount | `redis-sa` | `data` | Identity for Redis pods |
| ServiceAccount | `ingress-nginx-sa` | `ingress-system` | Identity for Ingress Controller |
| ServiceAccount | `prometheus-sa` | `monitoring` | Identity for Prometheus |
| ServiceAccount | `external-secrets-sa` | `monitoring` | Identity for ESO |
| Role | `postgres-config-reader` | `data` | Allow postgres-sa to read ConfigMaps |
| Role | `patroni-leader-election` | `data` | Allow Patroni ConfigMap/Endpoint CRUD |
| Role | `redis-config-reader` | `data` | Allow redis-sa to read ConfigMaps |
| RoleBinding | `postgres-config-reader-binding` | `data` | Bind postgres-sa to config-reader role |
| RoleBinding | `patroni-leader-election-binding` | `data` | Bind postgres-sa to Patroni role |
| RoleBinding | `redis-config-reader-binding` | `data` | Bind redis-sa to config-reader role |
| ClusterRole | `ingress-nginx-clusterrole` | cluster | Cluster-wide Ingress/Service read access |
| ClusterRole | `prometheus-clusterrole` | cluster | Cluster-wide pod/node metrics read |
| ClusterRole | `external-secrets-clusterrole` | cluster | Cluster-wide Secret create/update |
| ClusterRoleBinding | `ingress-nginx-clusterrolebinding` | cluster | Bind ingress-nginx-sa |
| ClusterRoleBinding | `prometheus-clusterrolebinding` | cluster | Bind prometheus-sa |
| ClusterRoleBinding | `external-secrets-clusterrolebinding` | cluster | Bind external-secrets-sa |
| HorizontalPodAutoscaler | `ui-hpa` | `ui` | Scale UI 2-4 replicas on CPU |
| HorizontalPodAutoscaler | `backend-api-hpa` | `backend` | Scale Backend 2-6 replicas on CPU + custom |
| HorizontalPodAutoscaler | `todo-agent-hpa` | `ai-agent` | Scale Agent 1-3 replicas on queue depth |
| HorizontalPodAutoscaler | `notification-svc-hpa` | `notifications` | Scale Notifications 2-4 on WS connections |
| PodDisruptionBudget | `ui-pdb` | `ui` | minAvailable: 1 |
| PodDisruptionBudget | `backend-api-pdb` | `backend` | minAvailable: 1 |
| PodDisruptionBudget | `todo-agent-pdb` | `ai-agent` | minAvailable: 1 |
| PodDisruptionBudget | `notification-svc-pdb` | `notifications` | minAvailable: 1 |
| PodDisruptionBudget | `postgres-pdb` | `data` | minAvailable: 2 |
| PodDisruptionBudget | `redis-pdb` | `data` | minAvailable: 2 |
| NetworkPolicy | `ui-default-deny` | `ui` | Deny all ingress + egress |
| NetworkPolicy | `ui-allow-ingress` | `ui` | Allow Ingress Controller -> UI:3000 |
| NetworkPolicy | `ui-allow-egress-backend` | `ui` | Allow UI -> Backend:8000 |
| NetworkPolicy | `ui-allow-dns` | `ui` | Allow DNS egress :53 |
| NetworkPolicy | `backend-default-deny` | `backend` | Deny all ingress + egress |
| NetworkPolicy | `backend-allow-ingress-traffic` | `backend` | Allow Ingress + UI + Agent -> Backend:8000 |
| NetworkPolicy | `backend-allow-egress-data` | `backend` | Allow Backend -> Postgres:5432 + Redis:6379 |
| NetworkPolicy | `backend-allow-dns` | `backend` | Allow DNS |
| NetworkPolicy | `agent-default-deny` | `ai-agent` | Deny all ingress + egress |
| NetworkPolicy | `agent-allow-egress-backend` | `ai-agent` | Allow Agent -> Backend:8000 |
| NetworkPolicy | `agent-allow-egress-redis` | `ai-agent` | Allow Agent -> Redis:6379 |
| NetworkPolicy | `agent-allow-egress-llm` | `ai-agent` | Allow Agent -> Egress Gateway (LLM APIs) |
| NetworkPolicy | `agent-allow-dns` | `ai-agent` | Allow DNS |
| NetworkPolicy | `notifications-default-deny` | `notifications` | Deny all ingress + egress |
| NetworkPolicy | `notifications-allow-ingress` | `notifications` | Allow Ingress Controller -> Notification:3001 |
| NetworkPolicy | `notifications-allow-egress-redis` | `notifications` | Allow Notification -> Redis:6379 |
| NetworkPolicy | `notifications-allow-egress-smtp` | `notifications` | Allow Notification -> Egress GW (SMTP) |
| NetworkPolicy | `notifications-allow-dns` | `notifications` | Allow DNS |
| NetworkPolicy | `data-default-deny` | `data` | Deny all ingress + egress |
| NetworkPolicy | `data-allow-postgres-ingress` | `data` | Allow Backend + monitoring -> Postgres:5432 |
| NetworkPolicy | `data-allow-redis-ingress` | `data` | Allow Backend + Agent + Notif + monitoring -> Redis:6379 |
| NetworkPolicy | `data-allow-replication` | `data` | Allow intra-namespace DB replication |
| NetworkPolicy | `data-allow-dns` | `data` | Allow DNS |
| PersistentVolumeClaim | `postgres-data-postgres-0` | `data` | PostgreSQL pod-0 data (50Gi SSD) |
| PersistentVolumeClaim | `postgres-data-postgres-1` | `data` | PostgreSQL pod-1 data (50Gi SSD) |
| PersistentVolumeClaim | `postgres-data-postgres-2` | `data` | PostgreSQL pod-2 data (50Gi SSD) |
| PersistentVolumeClaim | `redis-data-redis-0` | `data` | Redis pod-0 data (10Gi SSD) |
| PersistentVolumeClaim | `redis-data-redis-1` | `data` | Redis pod-1 data (10Gi SSD) |
| PersistentVolumeClaim | `redis-data-redis-2` | `data` | Redis pod-2 data (10Gi SSD) |
| StorageClass | `postgres-ssd` | cluster | SSD block storage for PostgreSQL (Retain) |
| StorageClass | `redis-ssd` | cluster | SSD block storage for Redis (Retain) |
| LimitRange | `ui-limits` | `ui` | Default resource limits for namespace |
| LimitRange | `backend-limits` | `backend` | Default resource limits for namespace |
| LimitRange | `agent-limits` | `ai-agent` | Default resource limits for namespace |
| LimitRange | `notifications-limits` | `notifications` | Default resource limits for namespace |
| LimitRange | `data-limits` | `data` | Default resource limits for namespace |
| ResourceQuota | `ui-quota` | `ui` | Namespace-level CPU/memory cap |
| ResourceQuota | `backend-quota` | `backend` | Namespace-level CPU/memory cap |
| ResourceQuota | `agent-quota` | `ai-agent` | Namespace-level CPU/memory cap |
| ResourceQuota | `notifications-quota` | `notifications` | Namespace-level CPU/memory cap |
| ResourceQuota | `data-quota` | `data` | Namespace-level CPU/memory cap |
| ServiceMonitor | `ui-monitor` | `monitoring` | Prometheus scrape config for UI |
| ServiceMonitor | `backend-api-monitor` | `monitoring` | Prometheus scrape config for Backend |
| ServiceMonitor | `agent-monitor` | `monitoring` | Prometheus scrape config for Agent |
| ServiceMonitor | `notification-monitor` | `monitoring` | Prometheus scrape config for Notifications |
| ServiceMonitor | `postgres-monitor` | `monitoring` | Prometheus scrape via pg-exporter sidecar |
| ServiceMonitor | `redis-monitor` | `monitoring` | Prometheus scrape via redis-exporter sidecar |
| PrometheusRule | `task-manager-alerts` | `monitoring` | All alerting rules |
| Certificate | `taskmanager-tls` | `ingress-system` | cert-manager TLS cert resource |
| ClusterIssuer | `letsencrypt-prod` | cluster | Let's Encrypt ACME issuer |
| ExternalSecret | `backend-secrets-sync` | `backend` | ESO sync from Vault |
| ExternalSecret | `agent-secrets-sync` | `ai-agent` | ESO sync from Vault |
| ExternalSecret | `notification-secrets-sync` | `notifications` | ESO sync from Vault |
| ExternalSecret | `postgres-credentials-sync` | `data` | ESO sync from Vault |
| ExternalSecret | `redis-credentials-sync` | `data` | ESO sync from Vault |

---

## Appendix: Key Architectural Decisions & Rationale

### Decision 1: Redis Pub/Sub for Backend-to-Notification Communication

Rather than having the Backend API make direct HTTP calls to the Notification Service, events are published to a Redis Pub/Sub channel. This decouples the services: the Backend API does not need to know the Notification Service's address, and the Notification Service can be scaled, replaced, or temporarily unavailable without blocking the Backend API. The trade-off is that Redis becomes a critical dependency for notification delivery; if Redis is down, notifications are lost (not queued). For production, use Redis Streams (persistent, consumer-group-based) instead of Pub/Sub for notification events where guaranteed delivery is needed.

### Decision 2: No Direct AI Agent to UI Connection

The original architecture diagram shows a direct connection from the Todo Agent to the UI. This is replaced with an event-driven model: Agent -> Backend API (task status update) -> Redis Pub/Sub -> Notification Service -> WebSocket -> Browser. This eliminates the need for the Agent to know about the UI's address, simplifies the Agent's networking requirements, and centralizes all backend state changes through the Backend API (single source of truth). The trade-off is slightly higher latency for real-time agent updates (one extra Redis hop), which is acceptable given the already multi-second latency of LLM operations.

### Decision 3: PostgreSQL HA with Patroni vs. External Operator

Patroni running as a sidecar within the StatefulSet provides HA without an external operator. The alternative (CloudNativePG or CrunchyData PGO) is more robust for production but adds operator complexity. For this plan, Patroni is assumed. The plan explicitly documents that the `postgres-sa` ServiceAccount needs Kubernetes API access for Patroni's leader election. If switching to CloudNativePG, the RBAC requirements change significantly (the operator manages them).

### Decision 4: KEDA for AI Agent Scaling

Native HPA cannot directly observe Redis queue depth without a custom metrics adapter. KEDA (Kubernetes Event-Driven Autoscaler) provides a clean `RedisScaler` that eliminates this complexity. KEDA itself is lightweight (one Deployment) and complements native HPA for the other services. KEDA should be deployed in its own namespace (`keda`) with appropriate RBAC to create HPA objects on behalf of ScaledObject resources.

---

*End of document — `scenario1-ai-task-manager.md`*

### Critical Files for Implementation

Since this is a blueprint document (no existing codebase was referenced), the critical files for materializing this plan as Kubernetes manifests would be:

- `/home/maiz/work/mine/pana/attempt/k8s/namespaces/namespaces.yaml` — all namespace definitions with labels
- `/home/maiz/work/mine/pana/attempt/k8s/data/postgres-statefulset.yaml` — most complex workload: StatefulSet, init containers, volumeClaimTemplates, Patroni RBAC
- `/home/maiz/work/mine/pana/attempt/k8s/ai-agent/deployment.yaml` — the most security-sensitive workload with external egress and LLM secret handling
- `/home/maiz/work/mine/pana/attempt/k8s/network-policies/` — all NetworkPolicy manifests, especially the ai-agent egress rules with FQDN/Egress Gateway patterns
- `/home/maiz/work/mine/pana/attempt/k8s/secrets/external-secrets/` — all ExternalSecret resources linking Vault paths to each namespace's Kubernetes Secrets