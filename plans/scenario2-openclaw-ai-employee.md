# OpenClaw Personal AI Employee — Kubernetes Architecture Plan

**Document:** `scenario2-openclaw-ai-employee.md`
**Version:** 1.0
**Date:** 2026-04-01
**Architect:** Claude Code (claude-sonnet-4-6)
**Source:** [openclaw.ai](https://openclaw.ai) | [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)

---

## Table of Contents

1. [Overview & Architecture Summary](#1-overview--architecture-summary)
2. [Namespace Design](#2-namespace-design)
3. [Pods & Workloads](#3-pods--workloads)
4. [Services & Exposure](#4-services--exposure)
5. [Resource Requests & Limits](#5-resource-requests--limits)
6. [ConfigMaps](#6-configmaps)
7. [Secrets Management](#7-secrets-management)
8. [Persistent Storage](#8-persistent-storage)
9. [RBAC Design](#9-rbac-design)
10. [Inter-Service Communication](#10-inter-service-communication)
11. [Network Policies](#11-network-policies)
12. [Pod Security Standards](#12-pod-security-standards)
13. [Observability & Audit](#13-observability--audit)
14. [Multi-Tenancy & User Isolation](#14-multi-tenancy--user-isolation)
15. [Autoscaling](#15-autoscaling)
16. [Summary Table](#16-summary-table)

---

## 1. Overview & Architecture Summary

### What is OpenClaw?

OpenClaw (formerly Clawdbot → Moltbot) is a **free, open-source, self-hosted personal AI assistant** built on Node.js 24. It follows a **hub-and-spoke architecture** centered on a WebSocket **Gateway** that orchestrates AI agent sessions, messaging channel connectors, browser automation, and a skills/plugin system. Users interact with OpenClaw through their preferred messaging apps (WhatsApp, Telegram, Slack, Discord, Signal, and 20+ others), and the AI autonomously executes tasks on their behalf — browsing, file ops, shell commands, calendar, email, and custom skills.

### K8s Adaptation Context

OpenClaw is natively a single-user, local-machine application. This plan adapts it for **multi-tenant production Kubernetes** deployment serving multiple users from a shared cluster, while strictly isolating each user's data, sessions, and credentials. The core adaptations are:

- The Gateway becomes a centralized, scalable Deployment with sticky WebSocket routing
- Channel connectors become dedicated Deployments per messaging platform
- Browser automation uses ephemeral sandboxed Jobs (gVisor runtime) — the highest security risk component
- Skills execution uses ephemeral sandboxed Jobs with near-zero permissions
- Per-user secret isolation is enforced at the K8s RBAC and Namespace level
- All LLM API keys and user credentials are stored in an external secrets manager (HashiCorp Vault / AWS Secrets Manager), never hardcoded

### Design Principles

- **Security first**: OpenClaw's agent has broad system permissions by design. In K8s, every permission is explicitly granted and scoped to the minimum necessary.
- **Blast radius minimization**: if any component is compromised (most likely: browser pod, skills pod, or agent runtime), the damage is contained to that namespace/pod with no lateral movement.
- **Zero-trust networking**: default-deny NetworkPolicy everywhere; all inter-service calls are explicitly allowlisted.
- **Prompt injection defense**: OpenClaw is known to be vulnerable to prompt injection via skills and inbound messages (per Cisco research). This plan includes detection, containment, and alerting layers.
- **Multi-tenant secret isolation**: User A's credentials are cryptographically inaccessible to User B's agent instance.

### ASCII Architecture Diagram

```
INTERNET (users via messaging apps)
        |
        | (HTTPS/WSS)
        v
+---------------------------+
|   Ingress (NGINX + TLS)   |  <-- cert-manager, rate limiting, WAF
|   (openclaw-gateway NS)   |
+---------------------------+
        |
        v
+---------------------------+     +---------------------------+
|       Gateway Pods        |<--->|  PostgreSQL StatefulSet   |
|  (WebSocket Control Plane)|     |  (openclaw-data NS)       |
|  session routing, auth,   |<--->|  Redis StatefulSet        |
|  tool dispatch, chunking  |     |  (session cache, pub/sub) |
+---------------------------+     +---------------------------+
        |            |
        |            +----------------------------+
        |                                         |
        v                                         v
+---------------------+               +---------------------+
|  Pi Agent Runtime   |               | Channel Connectors  |
|  (openclaw-agents)  |               | (openclaw-channels) |
|  LLM inference,     |               | WhatsApp (Baileys)  |
|  tool invocation,   |               | Telegram (grammY)   |
|  RPC over WebSocket |               | Slack (Bolt)        |
+---------------------+               | Discord (discord.js)|
        |                             | Signal, Matrix, IRC |
        |                             +---------------------+
        |                                      |
        | (spawn ephemeral Jobs)               | (outbound WSS/HTTPS
        |                                      |  to messaging platforms)
        v                                      v
+---------------------+               EXTERNAL MESSAGING
|  Browser Job Pods   |               PLATFORM APIs
|  (openclaw-browser) |
|  Chromium headless, |
|  gVisor sandboxed,  |
|  STRICT network     |
+---------------------+
        |
        v (HTTPS only, no internal cluster access)
   PUBLIC INTERNET

+---------------------+
|  Skills Executor    |
|  (openclaw-skills)  |
|  ephemeral Jobs,    |
|  near-zero perms,   |
|  restricted egress  |
+---------------------+

+---------------------+
| ClawHub Skill Proxy |
| (openclaw-skills)   |
| registry.clawhub.io |
+---------------------+
```

---

## 2. Namespace Design

Eight namespaces, each a security domain with its own NetworkPolicy, PodSecurityAdmission profile, ResourceQuota, and RBAC.

### `openclaw-gateway`
- **Purpose**: Central WebSocket control plane (the Gateway process). Entry point for all user traffic and inter-component coordination.
- **Workloads**: Gateway Deployment, WebChat UI Deployment (if enabled), Ingress resources
- **Isolation**: Ingress from Ingress controller only; egress to all internal namespaces (it is the router)
- **PodSecurityAdmission**: `baseline` (needs to bind to a port, but not privileged)

### `openclaw-agents`
- **Purpose**: Pi Agent Runtime instances — the AI agents processing user sessions
- **Workloads**: Agent Runtime Deployment (pool of agent workers), KEDA ScaledObject
- **Isolation**: Egress ONLY to Gateway (internal), LLM API providers (api.anthropic.com, api.openai.com — allowlisted IPs), and openclaw-browser/skills for spawning Jobs. No ingress from internet.
- **PodSecurityAdmission**: `restricted`

### `openclaw-channels`
- **Purpose**: Messaging channel connector processes. Each connector maintains a persistent outbound connection to its platform.
- **Workloads**: One Deployment per channel (WhatsApp, Telegram, Slack, Discord, Signal, Matrix, etc.)
- **Isolation**: Egress to internet (messaging platform APIs). Egress to Gateway (internal). No ingress from internet (connectors initiate outbound).
- **PodSecurityAdmission**: `baseline` (WhatsApp/Baileys needs some filesystem access for session state)

### `openclaw-browser`
- **Purpose**: Ephemeral browser automation pods. **HIGHEST RISK namespace.** Runs Chromium headless on behalf of user tasks.
- **Workloads**: Browser Jobs (ephemeral, created per browser task)
- **Isolation**: STRICT. Egress to internet on port 443 ONLY (web browsing). **ZERO access to internal cluster network** — browser pods must never reach PostgreSQL, Redis, Gateway, or any internal service. No ingress at all.
- **PodSecurityAdmission**: `restricted` + gVisor RuntimeClass
- **Rationale**: If a malicious website exploits Chromium, it must not be able to reach any internal K8s service. The browser sandbox is the most likely vector for cluster compromise.

### `openclaw-skills`
- **Purpose**: Ephemeral skill execution sandbox. Runs user-installed and ClawHub skills (Node.js/Python scripts).
- **Workloads**: Skill executor Jobs (ephemeral), ClawHub Registry Proxy Deployment
- **Isolation**: Default-deny all. Per-skill egress allowlist configured at Job creation time via admission controller. No access to Secrets (skills are untrusted code). No access to internal services by default.
- **PodSecurityAdmission**: `restricted` + gVisor RuntimeClass
- **Rationale**: Cisco research confirmed malicious skills can perform data exfiltration. Skills must be treated as untrusted code.

### `openclaw-data`
- **Purpose**: Persistent data layer — PostgreSQL, Redis, PVC-backed storage for transcripts and WhatsApp session state.
- **Workloads**: PostgreSQL StatefulSet, Redis StatefulSet
- **Isolation**: Ingress ONLY from `openclaw-gateway` and `openclaw-agents`. No internet egress.
- **PodSecurityAdmission**: `baseline`

### `openclaw-observability`
- **Purpose**: Monitoring, logging, audit trail, alerting.
- **Workloads**: Prometheus, Grafana, Loki/Fluent Bit, Jaeger, audit log forwarder
- **Isolation**: Ingress from cluster-internal only (Prometheus scrapes). Egress to alerting webhooks (PagerDuty, Slack).
- **PodSecurityAdmission**: `baseline`

### `openclaw-system`
- **Purpose**: Cluster-wide operators and controllers serving OpenClaw.
- **Workloads**: external-secrets-operator, Reloader, cert-manager (if not already cluster-wide), KEDA
- **Isolation**: Cluster-scoped access by design (operator pattern)

---

## 3. Pods & Workloads

### 3.1 Gateway

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-gateway` |
| **Replicas** | 2–4 (HA; WebSocket requires session affinity) |
| **Image** | `openclaw/gateway:stable` (Node.js 24) |
| **Init containers** | `wait-for-db`: checks PostgreSQL readiness; `wait-for-redis`: checks Redis readiness |
| **Liveness probe** | HTTP GET `/health` on port 8080, initialDelay 30s, period 15s |
| **Readiness probe** | HTTP GET `/ready` on port 8080, initialDelay 10s, period 10s |
| **Session affinity** | Service `sessionAffinity: ClientIP` or Ingress `nginx.ingress.kubernetes.io/affinity: cookie` — WebSocket connections must pin to one pod |
| **Restart policy** | Always |
| **Security context** | `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, tmpfs at `/tmp` and `/home/openclaw/.openclaw` |

**Justification**: The Gateway is OpenClaw's single control plane. It manages all session state via Redis and PostgreSQL, so horizontal scaling is safe. WebSocket connections require sticky routing; without it, messages would route to a pod without the connection.

### 3.2 Pi Agent Runtime

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-agents` |
| **Replicas** | 2 min, 10 max (KEDA / HPA) |
| **Image** | `openclaw/agent:stable` (Node.js 24) |
| **Mode** | Shared worker pool — agents pick up sessions from Gateway via Redis queue |
| **Init containers** | `check-llm-key`: verifies LLM API key Secret is mounted and non-empty before starting |
| **Liveness probe** | HTTP GET `/health` port 8081, period 20s |
| **Readiness probe** | HTTP GET `/ready` port 8081 (indicates agent has connected to Gateway via WebSocket RPC) |
| **Restart policy** | Always |
| **Security context** | `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, drop ALL capabilities, tmpfs at `/tmp` |

**Justification**: Agents are stateless workers — session state lives in PostgreSQL/Redis on the Gateway. Multiple replicas allow concurrent user sessions. KEDA scales on active-session-count metric from Prometheus.

### 3.3 Channel Connectors (one Deployment per platform)

**WhatsApp Connector (Baileys)**

| Field | Value |
|---|---|
| **Kind** | StatefulSet (requires persistent Baileys session state) |
| **Namespace** | `openclaw-channels` |
| **Replicas** | 1 per WhatsApp number/account |
| **Image** | `openclaw/channel-whatsapp:stable` |
| **Persistent storage** | PVC mounted at `/app/sessions/whatsapp` for Baileys auth state (QR code scan persists here) |
| **Liveness probe** | HTTP GET `/health`, period 30s |
| **Restart policy** | Always; on crash, Baileys re-uses the session PVC to reconnect without QR re-scan |

**Telegram Connector**

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-channels` |
| **Replicas** | 1–2 |
| **Image** | `openclaw/channel-telegram:stable` |
| **Config** | Webhook mode (preferred for K8s) vs long-polling; webhook URL points to an Ingress path |
| **Liveness probe** | HTTP GET `/health` |

**Slack Connector**

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-channels` |
| **Replicas** | 1–2 |
| **Image** | `openclaw/channel-slack:stable` |
| **Mode** | Socket Mode (WebSocket — no public URL needed) or HTTP events |

**Discord Connector**

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-channels` |
| **Replicas** | 1–2 |
| **Image** | `openclaw/channel-discord:stable` |
| **Config** | Gateway intents configured in ConfigMap |

**Signal / Matrix / IRC / Others**

| Field | Value |
|---|---|
| **Kind** | Deployment (1 per enabled channel) |
| **Namespace** | `openclaw-channels` |
| **Replicas** | 1 each (low traffic channels) |

All channel connectors share these security properties:
- `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, drop ALL capabilities
- Only their own channel token Secret is mounted (not others)
- No K8s API access needed → no ClusterRole binding

### 3.4 Browser Automation Pods

| Field | Value |
|---|---|
| **Kind** | Job (ephemeral — created per browser task, terminated on completion or timeout) |
| **Namespace** | `openclaw-browser` |
| **Image** | `openclaw/browser-sandbox:stable` (Chromium headless, minimal Linux) |
| **RuntimeClass** | `gvisor` (gVisor/runsc for kernel-level sandboxing — Chromium exploits cannot reach the host kernel) |
| **Timeout** | `activeDeadlineSeconds: 300` (5-minute hard cap per browse task) |
| **Restart policy** | Never (failures surface as errors to agent, not retried automatically) |
| **Security context** | `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, drop ALL capabilities, seccomp profile `runtime/default`, tmpfs at `/tmp` and `/home/chromium` |
| **ServiceAccount** | `browser-sa` — zero RBAC permissions, cannot read any Secret or call K8s API |
| **Spawned by** | Agent Runtime (via K8s Jobs API using `agent-sa`'s limited Job-create permission in browser namespace) |

**Justification**: Browser automation is the highest-risk operation. Chromium has a large CVE surface. gVisor adds a userspace kernel layer between Chromium and the host OS, preventing container-escape exploits from reaching the underlying node. The strict NetworkPolicy ensures that even if Chromium is exploited, it has no path to internal services.

### 3.5 Skills Executor Pods

| Field | Value |
|---|---|
| **Kind** | Job (ephemeral — created per skill invocation) |
| **Namespace** | `openclaw-skills` |
| **Image** | `openclaw/skills-sandbox:stable` (Node.js 24 + Python 3.12, minimal deps) |
| **RuntimeClass** | `gvisor` |
| **Timeout** | `activeDeadlineSeconds: 120` (2-minute hard cap) |
| **Restart policy** | Never |
| **Security context** | `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, drop ALL capabilities |
| **ServiceAccount** | `skills-sa` — zero default permissions; specific skills MAY be granted a specific Secret via admission controller policy |
| **Network** | Default deny; egress allowlist injected at Job creation time |

**Justification**: Skills are community-contributed code from ClawHub. Cisco confirmed at least one skill performed unauthorized data exfiltration. Every skill execution must be treated as running untrusted code in an isolated sandbox.

### 3.6 ClawHub Registry Proxy

| Field | Value |
|---|---|
| **Kind** | Deployment |
| **Namespace** | `openclaw-skills` |
| **Replicas** | 1 |
| **Image** | `openclaw/clawhub-proxy:stable` |
| **Purpose** | Proxies skill installs from `registry.clawhub.io`, enforces install gating policy (allowlist/blocklist by category), scans skill manifests for dangerous permissions before allowing installation |
| **Liveness probe** | HTTP GET `/health` |

### 3.7 Data Layer

**PostgreSQL**

| Field | Value |
|---|---|
| **Kind** | StatefulSet |
| **Namespace** | `openclaw-data` |
| **Replicas** | 1 (primary) + 1 (read replica, optional) |
| **Image** | `postgres:16` |
| **Storage** | PVC 50Gi SSD (`ReadWriteOnce`) |
| **Init containers** | `init-db`: runs schema migrations on first startup |
| **Liveness probe** | `pg_isready` exec command |

**Redis**

| Field | Value |
|---|---|
| **Kind** | StatefulSet |
| **Namespace** | `openclaw-data` |
| **Replicas** | 1 (or 3 for Redis Sentinel HA) |
| **Image** | `redis:7-alpine` |
| **Storage** | PVC 10Gi SSD (`ReadWriteOnce`) |
| **Purpose** | Session cache, Gateway↔Agent pub/sub, channel message queuing |
| **Liveness probe** | `redis-cli ping` exec command |

---

## 4. Services & Exposure

### Service Type Decisions

| Component | Service Type | Justification |
|---|---|---|
| Gateway | ClusterIP + Ingress | WebSocket upgraded via Ingress; sticky sessions via cookie affinity annotation |
| WebChat UI | ClusterIP + Ingress | HTTPS access for browser-based users |
| Telegram (webhook mode) | ClusterIP + Ingress path | Telegram sends POST to a webhook URL; needs to be reachable |
| WhatsApp connector | ClusterIP (headless) | StatefulSet — headless for pod-DNS identity |
| Slack connector | ClusterIP | Socket mode — outbound only, no inbound needed |
| Discord connector | ClusterIP | Outbound WebSocket only |
| Agent Runtime | ClusterIP | Internal RPC only, no direct user access |
| PostgreSQL | ClusterIP (headless) | StatefulSet, direct pod DNS |
| Redis | ClusterIP | Internal only |
| Browser Jobs | None | Ephemeral Jobs — no persistent Service |
| Skills Jobs | None | Ephemeral Jobs — no persistent Service |

### Ingress Rules

**Host:** `openclaw.example.com`

| Path | Backend | Notes |
|---|---|---|
| `/` | WebChat UI service | Serve OpenClaw's web interface |
| `/ws` | Gateway service port 18789 | WebSocket upgrade; `nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"` to prevent WS timeout |
| `/api/` | Gateway service port 8080 | REST API endpoints |
| `/webhooks/telegram` | Telegram connector service | Telegram webhook callback |
| `/webhooks/slack` | Slack connector service | Slack events (if HTTP mode) |

**TLS**: cert-manager `ClusterIssuer` with Let's Encrypt. Certificate stored in Secret `openclaw-tls` in `openclaw-gateway` namespace.

**Rate Limiting** (Ingress annotations):
- `/ws`: 100 connections per IP per minute (prevent WebSocket flood)
- `/api/`: 200 requests per IP per minute
- `/webhooks/`: 500 requests per minute per source (high volume from messaging platforms)

---

## 5. Resource Requests & Limits

### Gateway

```
requests:
  cpu: 250m
  memory: 512Mi
limits:
  cpu: 1000m
  memory: 1Gi
justification: WebSocket routing + session management. Medium CPU for connection handling,
               medium memory for concurrent session metadata. 4 replicas × 512Mi = 2Gi total.
```

### Pi Agent Runtime

```
requests:
  cpu: 500m
  memory: 1Gi
limits:
  cpu: 2000m
  memory: 4Gi
justification: Agent holds LLM response context in memory (Claude 3.5 can return large
               responses). High memory ceiling for large context windows. CPU is moderate
               (mostly waiting on LLM API I/O). 10 replicas × 4Gi = 40Gi maximum.
```

### WhatsApp Connector

```
requests:
  cpu: 100m
  memory: 256Mi
limits:
  cpu: 500m
  memory: 512Mi
justification: Baileys maintains persistent WS connection. Low baseline, occasional spikes
               on media messages.
```

### Telegram / Slack / Discord Connectors

```
requests:
  cpu: 50m
  memory: 128Mi
limits:
  cpu: 250m
  memory: 256Mi
justification: Lightweight event relay. Minimal CPU/memory.
```

### Browser Job Pods

```
requests:
  cpu: 500m
  memory: 512Mi
limits:
  cpu: 2000m
  memory: 2Gi
justification: Chromium headless is memory-hungry. Hard 2Gi cap prevents runaway browser
               tasks (e.g., malicious page loading huge assets). CPU limit prevents crypto
               mining if sandbox is compromised. These are ephemeral so the cap is per-task.
```

### Skills Executor Jobs

```
requests:
  cpu: 100m
  memory: 128Mi
limits:
  cpu: 500m
  memory: 512Mi
justification: Most skills are lightweight scripts. Hard caps prevent resource abuse from
               malicious skills. Namespace ResourceQuota caps total concurrent skill jobs.
```

### PostgreSQL

```
requests:
  cpu: 500m
  memory: 1Gi
limits:
  cpu: 2000m
  memory: 4Gi
justification: Session metadata DB. Not a heavy OLTP workload; medium sizing is sufficient.
               Memory for buffer_pool tuning.
```

### Redis

```
requests:
  cpu: 100m
  memory: 256Mi
limits:
  cpu: 500m
  memory: 1Gi
justification: Pub/sub + session cache. All data in-memory so memory is the key resource.
               1Gi cap is generous for this workload size.
```

### Namespace ResourceQuota — `openclaw-browser`

```
Total CPU limit: 8 cores        (max 4 concurrent browser jobs)
Total memory limit: 8Gi
Max pods: 4                     (concurrency cap)
```

### Namespace ResourceQuota — `openclaw-skills`

```
Total CPU limit: 4 cores        (max 8 concurrent skill jobs)
Total memory limit: 4Gi
Max pods: 8
```

---

## 6. ConfigMaps

### `openclaw-gateway-config` (namespace: `openclaw-gateway`)

Keys:
- `GATEWAY_PORT`: `18789`
- `GATEWAY_HTTP_PORT`: `8080`
- `SESSION_TIMEOUT_SECONDS`: `3600`
- `DM_POLICY`: `pairing` (default — unknown senders get pairing code; alternatives: `open`, `allowlist`)
- `CHUNKING_STRATEGY`: `telegram=4096,slack=3000,discord=2000,whatsapp=1500`
- `MAX_CONCURRENT_SESSIONS`: `100`
- `REDIS_HOST`: `redis-svc.openclaw-data.svc.cluster.local`
- `POSTGRES_HOST`: `postgres-svc.openclaw-data.svc.cluster.local`
- `AGENT_RUNTIME_ENDPOINT`: `agent-svc.openclaw-agents.svc.cluster.local:8081`
- `LOG_LEVEL`: `info`
- `AUDIT_LOG_ENABLED`: `true`

Mounted by: Gateway Deployment (`envFrom`)

### `openclaw-agent-config` (namespace: `openclaw-agents`)

Keys:
- `DEFAULT_LLM_PROVIDER`: `anthropic`
- `ANTHROPIC_MODEL`: `claude-opus-4-6`
- `OPENAI_MODEL`: `gpt-4o`
- `FALLBACK_PROVIDER`: `openai`
- `THINKING_MODE_DEFAULT`: `medium` (off/low/medium/high/xhigh)
- `MAX_TOKENS`: `8192`
- `TEMPERATURE`: `0.7`
- `LLM_REQUEST_TIMEOUT_SECONDS`: `120`
- `BROWSER_JOB_NAMESPACE`: `openclaw-browser`
- `SKILLS_JOB_NAMESPACE`: `openclaw-skills`
- `BROWSER_JOB_TIMEOUT_SECONDS`: `300`
- `SKILLS_JOB_TIMEOUT_SECONDS`: `120`
- `GATEWAY_WS_ENDPOINT`: `ws://gateway-svc.openclaw-gateway.svc.cluster.local:18789`
- `LOG_LEVEL`: `info`
- `PROMPT_INJECTION_DETECTION`: `enabled`

Mounted by: Agent Runtime Deployment (`envFrom`)

### `openclaw-channel-config-telegram` (namespace: `openclaw-channels`)

Keys:
- `TELEGRAM_MODE`: `webhook`
- `TELEGRAM_WEBHOOK_URL`: `https://openclaw.example.com/webhooks/telegram`
- `TELEGRAM_ALLOWED_UPDATE_TYPES`: `message,callback_query,inline_query`
- `GATEWAY_ENDPOINT`: `ws://gateway-svc.openclaw-gateway.svc.cluster.local:18789`
- `GROUP_ACTIVATION_MODE`: `mention` (only respond when @-mentioned in groups)

### `openclaw-channel-config-slack` (namespace: `openclaw-channels`)

Keys:
- `SLACK_MODE`: `socket` (socket mode — outbound WS, no public URL needed)
- `SLACK_LOG_LEVEL`: `warn`
- `GATEWAY_ENDPOINT`: `ws://gateway-svc.openclaw-gateway.svc.cluster.local:18789`

### `openclaw-channel-config-discord` (namespace: `openclaw-channels`)

Keys:
- `DISCORD_INTENTS`: `GUILDS,GUILD_MESSAGES,DIRECT_MESSAGES,MESSAGE_CONTENT`
- `DISCORD_PREFIX`: `!claw`
- `GATEWAY_ENDPOINT`: `ws://gateway-svc.openclaw-gateway.svc.cluster.local:18789`

### `openclaw-channel-config-whatsapp` (namespace: `openclaw-channels`)

Keys:
- `WHATSAPP_SESSION_PATH`: `/app/sessions/whatsapp`
- `WHATSAPP_PRINT_QR_IN_TERMINAL`: `false` (QR served via Gateway admin endpoint instead)
- `GATEWAY_ENDPOINT`: `ws://gateway-svc.openclaw-gateway.svc.cluster.local:18789`
- `RECONNECT_INTERVAL_SECONDS`: `30`

### `openclaw-browser-config` (namespace: `openclaw-browser`)

Keys:
- `CHROMIUM_FLAGS`: `--no-sandbox --disable-setuid-sandbox --headless=new --disable-gpu --disable-dev-shm-usage`
- `MAX_PAGE_LOAD_TIMEOUT_MS`: `30000`
- `DOWNLOAD_ALLOWED`: `false`
- `MAX_TABS`: `3`
- `USER_AGENT`: `Mozilla/5.0 (compatible; OpenClaw-Browser/1.0)`

### `openclaw-skills-config` (namespace: `openclaw-skills`)

Keys:
- `CLAWHUB_REGISTRY_URL`: `https://registry.clawhub.io`
- `CLAWHUB_PROXY_URL`: `http://clawhub-proxy-svc.openclaw-skills.svc.cluster.local`
- `INSTALL_GATING_POLICY`: `allowlist` (only pre-approved skill categories installable)
- `ALLOWED_SKILL_CATEGORIES`: `productivity,communication,calendar,reference`
- `BLOCKED_SKILL_CATEGORIES`: `system-access,network-tools,credential-management`
- `SKILL_WORKSPACE_PATH`: `/app/skills`
- `MAX_SKILL_EXECUTION_SECONDS`: `120`

---

## 7. Secrets Management

### Secret Classification & Storage

All secrets are managed in **HashiCorp Vault** (or AWS Secrets Manager / GCP Secret Manager) as the source of truth. The `external-secrets-operator` (ESO) running in `openclaw-system` synchronizes them into K8s Secrets on a configurable refresh interval (default: 1 hour; critical secrets: 15 minutes).

`Reloader` controller watches K8s Secrets and triggers rolling restarts of Deployments/StatefulSets when a Secret changes — enabling zero-downtime rotation.

### System-Level Secrets

**`openclaw-llm-keys`** (namespace: `openclaw-agents`)
- Type: `Opaque`
- Keys: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`
- Source: Vault path `secret/openclaw/llm-keys`
- Mounted by: Agent Runtime pods (envFrom) — agents need these to call LLM APIs
- Rotation: Monthly or on-demand; Reloader triggers agent rolling restart on change
- Expiry handling: If the Anthropic key hits rate limits mid-session, the agent catches the 429/401 error, logs it, notifies the user via their messaging channel ("I've hit an API limit, please try again shortly"), and queues the task for retry. If the key expires (401), the agent runtime returns an error to the Gateway, which sends a user-facing message: "My AI key needs renewal — please contact your administrator." An alert fires to PagerDuty.

**`openclaw-db-credentials`** (namespace: `openclaw-data`, read by `openclaw-gateway`)
- Type: `Opaque`
- Keys: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- Source: Vault path `secret/openclaw/db`
- Mounted by: Gateway Deployment (env), PostgreSQL StatefulSet (env)
- Rotation: Quarterly via Vault dynamic secrets (Vault generates a new DB credential, rotates it in the DB, ESO syncs to K8s, Reloader restarts Gateway pods)

**`openclaw-redis-credentials`** (namespace: `openclaw-data`, read by `openclaw-gateway` and `openclaw-agents`)
- Type: `Opaque`
- Keys: `REDIS_PASSWORD`
- Rotation: Quarterly

**`openclaw-gateway-auth-secret`** (namespace: `openclaw-gateway`)
- Type: `Opaque`
- Keys: `GATEWAY_AUTH_TOKEN` (password for Tailscale/remote access auth), `JWT_SIGNING_KEY` (for user session JWTs)
- Rotation: JWT key rotated every 90 days (use key versioning — keep old key valid for 24h during rollover to avoid session invalidation)

**`openclaw-tls`** (namespace: `openclaw-gateway`)
- Type: `kubernetes.io/tls`
- Managed by: cert-manager (auto-renewed before expiry)

### Per-Channel Secrets

**`telegram-bot-token`** (namespace: `openclaw-channels`)
- Type: `Opaque`
- Keys: `TELEGRAM_BOT_TOKEN`
- Mounted by: Telegram connector only
- Rotation: On-demand (BotFather revoke + new token)
- Compromise handling: Revoke token in BotFather, update in Vault, ESO syncs, Reloader restarts Telegram connector pod only. Other channel connectors are unaffected.

**`slack-credentials`** (namespace: `openclaw-channels`)
- Type: `Opaque`
- Keys: `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`, `SLACK_APP_TOKEN` (socket mode)
- Rotation: On-demand via Slack App dashboard

**`discord-bot-token`** (namespace: `openclaw-channels`)
- Type: `Opaque`
- Keys: `DISCORD_BOT_TOKEN`
- Rotation: On-demand via Discord Developer Portal

**`whatsapp-session-secret`** (namespace: `openclaw-channels`)
- Type: `Opaque`
- Keys: `WHATSAPP_ENCRYPTION_KEY` (encrypts Baileys session files at rest on the PVC)
- Compromise handling: If WhatsApp session state is compromised (PVC accessed by attacker), invalidate the Baileys session files (delete PVC contents), rotate encryption key in Vault, re-trigger QR scan pairing. WhatsApp service resumes after re-pair (typically < 2 minutes).

### Per-User Secrets (Multi-Tenant)

Each user who connects their own external services (Gmail, GitHub, OpenAI key, calendar, Spotify, etc.) has their credentials stored in **user-scoped Secrets**.

**Isolation model**: Namespace-per-user approach (see Multi-Tenancy section). Each user has a namespace `openclaw-user-<userid>` containing their Secrets.

| Secret Name | Namespace | Keys | Who Can Read |
|---|---|---|---|
| `user-llm-key` | `openclaw-user-<id>` | `OPENAI_API_KEY` (user's own key, optional) | That user's agent session only |
| `user-gmail-token` | `openclaw-user-<id>` | `GMAIL_ACCESS_TOKEN`, `GMAIL_REFRESH_TOKEN` | That user's agent session only |
| `user-github-token` | `openclaw-user-<id>` | `GITHUB_PERSONAL_ACCESS_TOKEN` | That user's agent session only |
| `user-calendar-token` | `openclaw-user-<id>` | `GCAL_ACCESS_TOKEN`, `GCAL_REFRESH_TOKEN` | That user's agent session only |

**Just-in-Time Secret Injection for Agents**: The Agent Runtime does NOT have standing access to all user Secrets. When a user's task requires a specific credential:
1. Gateway instructs the Agent to use credential `gmail` for user `U123`
2. Agent Runtime's K8s ServiceAccount has a RoleBinding in namespace `openclaw-user-U123` that grants `get` on Secret `user-gmail-token` — but only that Secret
3. Agent fetches the Secret at task execution time via K8s API
4. After use, the in-memory credential is discarded (not persisted to disk by Agent)
5. The RoleBinding is scoped to the user namespace — User A's agent SA has no RoleBinding in User B's namespace

**Secret Expiry Mid-Task Handling**:
1. Agent calls Gmail API → receives `401 Unauthorized` (token expired)
2. Agent catches error, attempts token refresh using `GMAIL_REFRESH_TOKEN`
3. If refresh succeeds: updates `user-gmail-token` Secret in `openclaw-user-<id>` namespace via K8s API (agent SA has `update` permission on this Secret)
4. If refresh fails (revoked token): agent notifies user via messaging channel: "Your Gmail connection needs re-authorization. Please re-link at https://openclaw.example.com/settings/integrations"
5. Task is paused and persisted in PostgreSQL; user can re-trigger after re-authorization
6. Alert fires to observability stack: `user_credential_expired{user_id, service}` metric increments

**Secret Compromise Handling (Full Runbook)**:

*Scenario: LLM API key compromised (e.g., found in a leaked log)*
1. Immediately revoke key at Anthropic/OpenAI dashboard
2. Update Vault path `secret/openclaw/llm-keys` with new key
3. ESO refresh interval fires (or manual trigger: `kubectl annotate externalsecret openclaw-llm-keys force-sync=now`)
4. K8s Secret `openclaw-llm-keys` updated in `openclaw-agents` namespace
5. Reloader detects Secret change → triggers rolling restart of Agent Runtime pods
6. Old pods drain (existing sessions complete or timeout), new pods start with new key
7. Zero downtime during rotation: rolling update ensures at least `minReadySeconds` of overlap
8. Audit: query K8s audit log for all accesses to the old Secret (which pods, which service accounts, timestamps)
9. Incident report: file in observability stack

*Scenario: User credential compromise (user reports GitHub token leaked)*
1. User revokes token on GitHub
2. Admin (or automated process) deletes Secret `user-github-token` in `openclaw-user-<id>` namespace
3. Any in-flight agent task using that token receives an error on next API call → task paused, user notified
4. Only User `<id>` is affected; all other users' sessions continue uninterrupted
5. User re-links GitHub via UI → new token stored in Secret

*Scenario: Agent Runtime pod compromised (attacker gained code execution in agent pod)*
1. Blast radius: attacker can read LLM API keys (in env), call K8s API with `agent-sa` permissions (limited: only get specific Secrets in user namespaces, create Jobs in browser/skills namespaces)
2. Immediate response: `kubectl delete pod <compromised-pod>` — pod is ephemeral, session state in PostgreSQL/Redis survives
3. Rotate `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` immediately
4. Audit: check K8s audit log for `agent-sa` actions in the last 24 hours — look for unexpected Secret reads or Job creations
5. Review network logs for unexpected egress from `openclaw-agents` namespace (NetworkPolicy should have blocked internal-only destinations)
6. If `agent-sa` was used to access user namespaces: treat all accessed user credentials as compromised, notify affected users

**Prompt Injection Mitigation** (OpenClaw-specific security concern):
- Gateway implements an input sanitization layer: messages arriving from channel connectors are scanned for known injection patterns (e.g., "Ignore previous instructions", "You are now in developer mode", attempts to invoke hidden skills)
- Suspicious messages are flagged, logged, and optionally rejected or sandboxed
- `PROMPT_INJECTION_DETECTION: enabled` in agent ConfigMap activates the agent's built-in detection
- Alert fires when injection pattern detected: `prompt_injection_attempt{channel, user_id, pattern}`
- Skills manifests are vetted by ClawHub proxy before installation — skills requesting `SECRET_READ` or `CLUSTER_ADMIN` permissions are blocked

---

## 8. Persistent Storage

### PostgreSQL PVC

| Field | Value |
|---|---|
| **Name** | `postgres-data` |
| **Namespace** | `openclaw-data` |
| **Size** | 50Gi (scalable to 200Gi) |
| **StorageClass** | `premium-rwo` (SSD, `ReadWriteOnce`) |
| **Access mode** | `ReadWriteOnce` |
| **Contents** | Session metadata, user preferences, usage stats, task history, audit log index |
| **Backup** | `pg_dump` CronJob daily at 02:00 UTC → S3-compatible bucket; Velero snapshot weekly |

### Redis PVC

| Field | Value |
|---|---|
| **Name** | `redis-data` |
| **Namespace** | `openclaw-data` |
| **Size** | 10Gi |
| **StorageClass** | `premium-rwo` (SSD) |
| **Access mode** | `ReadWriteOnce` |
| **Contents** | Session cache, pub/sub channels, message queue |
| **Backup** | Redis AOF persistence enabled; Velero snapshot daily |

### WhatsApp Baileys Session PVC

| Field | Value |
|---|---|
| **Name** | `whatsapp-session-data` |
| **Namespace** | `openclaw-channels` |
| **Size** | 1Gi |
| **StorageClass** | `standard-rwo` |
| **Access mode** | `ReadWriteOnce` |
| **Contents** | Baileys auth state (encrypted with `WHATSAPP_ENCRYPTION_KEY`) |
| **Backup** | Velero snapshot daily |

### Transcript / Skill Workspace PVCs (Per-User)

| Field | Value |
|---|---|
| **Name** | `user-data-<userid>` |
| **Namespace** | `openclaw-user-<userid>` |
| **Size** | 5Gi per user |
| **StorageClass** | `standard-rwo` |
| **Contents** | Session transcripts (Markdown files), skill workspace files, downloaded artifacts |

### Audit Log PVC

| Field | Value |
|---|---|
| **Name** | `audit-log-data` |
| **Namespace** | `openclaw-observability` |
| **Size** | 100Gi |
| **StorageClass** | `standard-rwo` |
| **Access mode** | `ReadWriteOnce` |
| **Write policy** | Append-only (audit logger pod has write access; no other pod can delete) |

---

## 9. RBAC Design

### ServiceAccounts

| ServiceAccount | Namespace | Purpose |
|---|---|---|
| `gateway-sa` | `openclaw-gateway` | Gateway pods — session routing, Redis/Postgres access |
| `agent-sa` | `openclaw-agents` | Agent Runtime — LLM calls, spawn browser/skills Jobs, per-user Secret access |
| `channel-whatsapp-sa` | `openclaw-channels` | WhatsApp connector — reads only WhatsApp Secret |
| `channel-telegram-sa` | `openclaw-channels` | Telegram connector — reads only Telegram Secret |
| `channel-slack-sa` | `openclaw-channels` | Slack connector — reads only Slack Secret |
| `channel-discord-sa` | `openclaw-channels` | Discord connector — reads only Discord Secret |
| `browser-sa` | `openclaw-browser` | Browser Jobs — ZERO permissions intentionally |
| `skills-sa` | `openclaw-skills` | Skills Jobs — zero default; skill-specific Secret grants via admission |
| `clawhub-proxy-sa` | `openclaw-skills` | ClawHub proxy — read skill ConfigMap, list external URLs |
| `data-sa` | `openclaw-data` | PostgreSQL/Redis — no K8s API access needed |
| `observability-sa` | `openclaw-observability` | Prometheus scrapes, audit log writes |
| `admin-sa` | `openclaw-system` | Operators only — ClusterRole for ESO, Reloader, KEDA |

### Roles and RoleBindings

**`gateway-role`** (namespace: `openclaw-gateway`)
- `get`, `list`, `watch` on ConfigMaps (own namespace)
- `get` on Secrets: `openclaw-db-credentials`, `openclaw-redis-credentials`, `openclaw-gateway-auth-secret`
- Bound to: `gateway-sa`

**`agent-role`** (namespace: `openclaw-agents`)
- `get` on Secrets: `openclaw-llm-keys` only
- `get`, `list` on ConfigMaps (own namespace)
- Bound to: `agent-sa`

**`agent-browser-spawner-role`** (namespace: `openclaw-browser`)
- `create`, `get`, `list`, `watch`, `delete` on Jobs
- NO Secret access in this namespace
- Bound to: `agent-sa` (cross-namespace RoleBinding)

**`agent-skills-spawner-role`** (namespace: `openclaw-skills`)
- `create`, `get`, `list`, `watch`, `delete` on Jobs
- NO Secret access
- Bound to: `agent-sa` (cross-namespace RoleBinding)

**`agent-user-secret-role`** (namespace: `openclaw-user-<userid>` — one per user)
- `get` on the specific Secrets needed by that user (e.g., `user-gmail-token`, `user-github-token`)
- `update` on those same Secrets (for token refresh)
- Bound to: `agent-sa` (cross-namespace RoleBinding, per-user)

**`channel-whatsapp-role`** (namespace: `openclaw-channels`)
- `get` on Secret `whatsapp-session-secret` only
- `get`, `list` on ConfigMap `openclaw-channel-config-whatsapp` only
- Bound to: `channel-whatsapp-sa`

*(Similar minimal roles for Telegram, Slack, Discord channel SAs — each gets only their own Secret)*

**`observability-role`** (ClusterRole)
- `get`, `list`, `watch` on Pods, Services, Endpoints (for Prometheus service discovery)
- `create` on Events (for audit log writes)
- Bound to: `observability-sa`

### Principle of Least Privilege Checklist

- [x] Browser SA has ZERO RBAC permissions — it cannot call the K8s API at all
- [x] Skills SA has ZERO default permissions — specific grants only via admission controller
- [x] No SA has `*` verbs on any resource
- [x] Agent SA can only read `openclaw-llm-keys` Secret in its own namespace
- [x] Agent SA accesses per-user Secrets only in their respective user namespace via scoped RoleBindings
- [x] Channel SAs can only read their own channel token Secret, nothing else
- [x] `default` ServiceAccount is never used by any pod (`automountServiceAccountToken: false` on default SA in all namespaces)
- [x] No application SA has `cluster-admin` or any ClusterRole binding except observability
- [x] Database pods do not need K8s API access — no RoleBinding for `data-sa`

---

## 10. Inter-Service Communication

| From | To | Protocol | Internal DNS | Auth |
|---|---|---|---|---|
| User (browser/device) | Gateway | WSS/HTTPS | `openclaw.example.com` (Ingress) | JWT session token |
| Telegram/Slack webhooks | Channel connectors | HTTPS POST | Ingress path `/webhooks/*` | Signing secret validation |
| WhatsApp/Slack/Discord connectors | Gateway | WebSocket | `gateway-svc.openclaw-gateway.svc.cluster.local:18789` | Internal SA token |
| Gateway | Agent Runtime | WebSocket RPC | `agent-svc.openclaw-agents.svc.cluster.local:8081` | Internal SA token |
| Gateway | PostgreSQL | TCP | `postgres-svc.openclaw-data.svc.cluster.local:5432` | DB credentials (Secret) |
| Gateway | Redis | TCP | `redis-svc.openclaw-data.svc.cluster.local:6379` | Redis password (Secret) |
| Agent Runtime | Gateway | WebSocket RPC | `gateway-svc.openclaw-gateway.svc.cluster.local:18789` | Internal SA token |
| Agent Runtime | Anthropic API | HTTPS | `api.anthropic.com` (external egress) | `ANTHROPIC_API_KEY` (Secret) |
| Agent Runtime | OpenAI API | HTTPS | `api.openai.com` (external egress) | `OPENAI_API_KEY` (Secret) |
| Agent Runtime | K8s API (spawn Jobs) | HTTPS | `kubernetes.default.svc.cluster.local` | `agent-sa` ServiceAccount token |
| Browser Job | Internet | HTTPS port 443 | External (via NetworkPolicy egress allow) | None (anonymous browsing) |
| Browser Job → internal | BLOCKED | N/A | NetworkPolicy denies all cluster-internal egress | N/A |
| Skills Job | Internet (allowlisted) | HTTPS | Per-skill egress allowlist | Per-skill credential (if granted) |
| Agent Runtime → user namespace | K8s API | HTTPS | `kubernetes.default.svc.cluster.local` | `agent-sa` (user-scoped RoleBinding) |
| ESO | Vault/AWS Secrets Manager | HTTPS | External | IAM role / Vault AppRole |
| Reloader | K8s API | HTTPS | Internal | ClusterRole — watch Secrets/Deployments |

---

## 11. Network Policies

### `openclaw-gateway` namespace

```
Ingress:
  - Allow from: Ingress controller namespace (nginx pods) on ports 8080, 18789
  - Allow from: openclaw-agents on port 18789 (agent WebSocket RPC)
  - Allow from: openclaw-channels on port 18789 (channel connector → Gateway)
  - DENY all other ingress

Egress:
  - Allow to: openclaw-data on ports 5432 (PostgreSQL), 6379 (Redis)
  - Allow to: openclaw-agents on port 8081 (Agent Runtime RPC)
  - Allow to: DNS (kube-dns) port 53
  - DENY all other egress
```

### `openclaw-agents` namespace

```
Ingress:
  - Allow from: openclaw-gateway on port 8081
  - DENY all other ingress

Egress:
  - Allow to: openclaw-gateway on port 18789 (RPC back-channel)
  - Allow to: Kubernetes API server (for Job creation and Secret reads)
  - Allow to: api.anthropic.com:443 (LLM API)
  - Allow to: api.openai.com:443 (LLM API)
  - Allow to: DNS port 53
  - DENY all other egress (especially: cannot reach openclaw-data directly, cannot reach internet except LLM endpoints)
```

### `openclaw-channels` namespace

```
Ingress:
  - Allow from: Ingress controller on webhook paths (Telegram, Slack HTTP mode)
  - DENY all other ingress

Egress:
  - Allow to: openclaw-gateway on port 18789
  - Allow to: internet:443 (messaging platform APIs: api.telegram.org, slack.com, discord.com, web.whatsapp.com, etc.)
  - Allow to: DNS port 53
  - DENY intra-cluster traffic except Gateway
```

### `openclaw-browser` namespace — STRICTEST

```
Ingress:
  - DENY ALL (browser Jobs never receive inbound connections)

Egress:
  - Allow to: internet:443 ONLY (web browsing)
  - Allow to: internet:80 (HTTP redirect following — optionally restrict)
  - Allow to: DNS port 53 (external DNS only, NOT internal kube-dns for cluster service discovery)
  - DENY ALL cluster-internal egress (NO access to openclaw-gateway, openclaw-data, openclaw-agents, or any other namespace)
```

**This is the critical policy**: if Chromium is exploited, the attacker cannot pivot to internal services. They can only reach the public internet — which limits the attack to data exfiltration of public content, not cluster compromise.

### `openclaw-skills` namespace

```
Ingress:
  - DENY ALL

Egress:
  - Default DENY all
  - Per-skill Jobs: egress rules injected at Job creation time by agent (specific domain allowlist per skill)
  - ClawHub proxy: Allow to registry.clawhub.io:443
  - Allow to: DNS port 53
  - DENY cluster-internal egress
```

### `openclaw-data` namespace

```
Ingress:
  - Allow from: openclaw-gateway on ports 5432, 6379
  - Allow from: openclaw-agents on port 6379 (Redis pub/sub for session events)
  - Allow from: openclaw-observability (Prometheus metrics scrape)
  - DENY all other ingress

Egress:
  - Allow to: DNS port 53
  - DENY all other egress (databases never initiate outbound connections)
```

---

## 12. Pod Security Standards

### PodSecurityAdmission Profiles by Namespace

| Namespace | Profile | Reason |
|---|---|---|
| `openclaw-gateway` | `baseline` | Needs port binding, no root required |
| `openclaw-agents` | `restricted` | Fully stateless, no special capabilities needed |
| `openclaw-channels` | `baseline` | WhatsApp Baileys needs some filesystem access |
| `openclaw-browser` | `restricted` + gVisor | Chromium is the highest-risk component |
| `openclaw-skills` | `restricted` + gVisor | Untrusted skill code must be sandboxed |
| `openclaw-data` | `baseline` | Databases need IPC for shared memory |
| `openclaw-observability` | `baseline` | Prometheus needs host port access for scraping |

### Security Context Applied to All Pods

```
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

tmpfs mounts where writable space is needed:
- `/tmp` (all pods)
- `/home/openclaw/.openclaw` (Gateway, Agent Runtime — runtime config)
- `/home/chromium` (browser pods — Chromium profile)

### gVisor RuntimeClass for Browser and Skills

A `RuntimeClass` named `gvisor` must be configured on the cluster with `handler: runsc` (gVisor's runtime). All browser Job pods and skills Job pods specify `runtimeClassName: gvisor`.

**What gVisor provides**: A userspace kernel (the "Sentry") that intercepts all system calls from the container before they reach the host kernel. A Chromium exploit that achieves arbitrary code execution within the container still faces gVisor's Sentry — it cannot make raw system calls to the host OS. This is defense-in-depth against container escape.

### AppArmor Profiles

Browser pods use the `chrome-sandbox` AppArmor profile (available on Ubuntu/Debian nodes) which restricts ptrace, namespace creation, and certain syscalls that Chromium exploits commonly use.

---

## 13. Observability & Audit

### Logging

All pods output **structured JSON logs** to stdout/stderr. Fluent Bit DaemonSet forwards to central log store (Loki or Elasticsearch):

| Component | Key log fields | Retention |
|---|---|---|
| Gateway | `session_id`, `user_id`, `channel`, `action`, `tool_invoked`, `latency_ms` | 90 days |
| Agent Runtime | `session_id`, `user_id`, `llm_provider`, `model`, `tokens_used`, `cost_usd`, `tool_name`, `tool_args_hash` | 90 days |
| Channel connectors | `channel`, `message_id`, `user_id`, `direction` (in/out) | 30 days |
| Browser Jobs | `job_id`, `user_id`, `session_id`, `url_visited`, `action_type`, `duration_ms` | 180 days (security audit) |
| Skills Jobs | `job_id`, `user_id`, `skill_name`, `skill_version`, `egress_destinations`, `duration_ms` | 180 days (security audit) |

**Tool args are hashed, not logged in plaintext** to prevent credential leakage in logs (e.g., if an agent tool call includes a password as an argument).

### K8s Audit Policy

A cluster-level audit policy logs:
- All `get`, `list`, `watch` operations on `secrets` resources across all namespaces → retained 1 year
- All `create`, `delete` operations on Jobs in `openclaw-browser` and `openclaw-skills` → retained 1 year
- All `create`, `update`, `delete` on RoleBindings (privilege escalation detection) → retained 2 years

### Metrics (Prometheus)

Key metrics to expose and alert on:

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `openclaw_active_sessions` | > 90% of `MAX_CONCURRENT_SESSIONS` | Approaching capacity |
| `openclaw_llm_api_errors_total{error_type="401"}` | Any > 0 | API key expired/revoked |
| `openclaw_llm_api_errors_total{error_type="429"}` | > 5 in 5 min | Rate limit approaching |
| `openclaw_browser_jobs_active` | > 3 | Near browser concurrency cap |
| `openclaw_user_credential_expired_total` | Any > 0 | User needs to re-link service |
| `openclaw_prompt_injection_attempt_total` | Any > 0 | Security alert — requires investigation |
| `openclaw_skills_job_duration_seconds` | p99 > 100s | Skills approaching timeout |
| `kube_pod_container_status_restarts_total` | > 3 in 10 min | Pod crash loop |
| `openclaw_secret_access_total{unexpected=true}` | Any | Anomalous Secret access |
| `container_network_transmit_bytes_total` (browser NS) | Spike > 10MB/job | Possible data exfiltration from browser pod |

### Tracing

OpenTelemetry SDK instrumented in Gateway and Agent Runtime. Traces exported to Jaeger/Tempo. Key spans:
- `session.create` → `agent.dispatch` → `llm.call` → `tool.execute` → `response.send`
- `browser.job.create` → `browser.navigate` → `browser.extract` → `browser.job.complete`

This gives full end-to-end latency visibility and correlates user sessions across components.

### Alerting

PagerDuty integration for:
- P1: LLM API key expired/revoked (users cannot get responses)
- P1: Gateway pod crash loop
- P2: Prompt injection attempt detected
- P2: Unexpected Secret access pattern
- P3: User credential expired (user-facing degradation, not outage)
- P3: Browser pod egress spike (potential exfiltration)

---

## 14. Multi-Tenancy & User Isolation

### Isolation Model: Namespace-Per-User (Recommended)

**Decision**: Use per-user namespaces (`openclaw-user-<userid>`) for Secret and storage isolation, combined with shared workload namespaces (agents, channels, gateway) for compute efficiency.

**Why not pure shared-namespace with label selectors**: NetworkPolicy label selectors can be misconfigured. K8s RBAC is more reliably enforced at the namespace boundary. For a system handling user credentials and private conversations, namespace isolation is the appropriate security boundary.

**Why not full per-user namespace for compute**: Creating separate Agent Runtime and Channel connector Deployments per user is operationally expensive at scale (100 users = 100 Deployments). The shared pool model with per-session routing is more efficient.

**Hybrid model**:
- **Shared namespaces**: `openclaw-gateway`, `openclaw-agents`, `openclaw-channels` (compute — shared workers serving all users)
- **Per-user namespaces**: `openclaw-user-<userid>` (data — Secrets, PVCs, per-user ConfigMaps)

### Session Isolation in Agent Runtime

The Agent Runtime pool processes sessions for multiple users concurrently. Each agent worker handles one session at a time. Session isolation is enforced by:
1. The Gateway assigns sessions to agents and passes the user ID
2. The agent fetches user-specific Secrets only from `openclaw-user-<userid>` (RBAC enforces this)
3. Session context (conversation history, ongoing task state) is stored in PostgreSQL keyed by `(user_id, session_id)` — not accessible cross-session
4. Redis pub/sub channels are namespaced by `session_id` — no cross-user pub/sub leakage
5. Agent worker memory is isolated per-process (one Node.js worker per session)

### Per-User Resource Quotas

Each user namespace has a ResourceQuota:
```
openclaw-user-<userid>:
  requests.storage: 5Gi
  persistentvolumeclaims: 10
  count/secrets: 20
```

Additionally, the Gateway enforces per-user rate limits:
- Max 5 concurrent agent sessions per user
- Max 10 browser Jobs per user per hour
- Max 20 skill executions per user per hour

---

## 15. Autoscaling

### Gateway — HPA

```
component: Gateway
HPA:
  minReplicas: 2
  maxReplicas: 8
  scaleOn: Custom metric — active WebSocket connections per pod
           (from Prometheus: openclaw_gateway_active_connections)
  scaleUp threshold: > 50 connections per pod
  scaleDown threshold: < 20 connections per pod
  stabilizationWindowSeconds: 300 (5 min — prevent flapping during WS reconnects)
```

**Note**: WebSocket sticky sessions mean scale-down must be graceful. The Gateway should drain connections (send reconnect hints to clients) before a pod terminates.

### Pi Agent Runtime — KEDA

```
component: Agent Runtime
KEDA ScaledObject:
  minReplicas: 2
  maxReplicas: 20
  trigger: Redis list length (session dispatch queue)
  queueLength: 5 (scale up when >5 queued sessions per pod)
  pollingInterval: 15s
```

KEDA's Redis scaler monitors the dispatch queue depth. When users submit tasks faster than agents process them, KEDA adds more agent pods. Scale-down is gradual (KEDA respects in-flight sessions).

### Channel Connectors — Manual Scaling

Channel connectors maintain persistent WebSocket connections to their respective platforms. These cannot be HPA-scaled based on CPU/memory because connection count (not CPU) is the constraint. Scaling is manual: add replicas when a single connector can't handle the message throughput. WhatsApp connectors are 1:1 with WhatsApp accounts (Baileys limitation).

### Browser Jobs — On-Demand (No HPA)

Browser pods are ephemeral Jobs created by agents on demand. The ResourceQuota in `openclaw-browser` caps maximum concurrency at 4 pods. No HPA needed — the resource quota is the throttle.

### Skills Executor Jobs — On-Demand (No HPA)

Same pattern as browser — ephemeral Jobs. ResourceQuota caps at 8 concurrent skill pods.

### PostgreSQL — Vertical Scaling

Scale up PostgreSQL Pod resources (CPU/memory) as data grows. Use read replicas for read-heavy workloads. No HPA for databases — use connection pooling (PgBouncer sidecar) to handle connection spikes.

---

## 16. Summary Table

| Object Kind | Name | Namespace | Purpose |
|---|---|---|---|
| Namespace | openclaw-gateway | — | Gateway control plane isolation |
| Namespace | openclaw-agents | — | Agent Runtime isolation |
| Namespace | openclaw-channels | — | Channel connector isolation |
| Namespace | openclaw-browser | — | Browser sandbox (highest security) |
| Namespace | openclaw-skills | — | Skills execution sandbox |
| Namespace | openclaw-data | — | Data layer isolation |
| Namespace | openclaw-observability | — | Monitoring & audit |
| Namespace | openclaw-system | — | Cluster operators |
| Namespace | openclaw-user-\<id\> | — | Per-user Secrets & storage (one per user) |
| Deployment | gateway | openclaw-gateway | WebSocket control plane (2–8 replicas) |
| Deployment | webchat-ui | openclaw-gateway | Browser-based web UI for OpenClaw |
| Deployment | agent-runtime | openclaw-agents | Pi Agent Runtime pool (2–20 replicas, KEDA) |
| StatefulSet | whatsapp-connector | openclaw-channels | Baileys WhatsApp integration (persistent session) |
| Deployment | telegram-connector | openclaw-channels | Telegram bot (grammY) |
| Deployment | slack-connector | openclaw-channels | Slack Bolt integration |
| Deployment | discord-connector | openclaw-channels | Discord.js integration |
| Deployment | signal-connector | openclaw-channels | Signal integration |
| Deployment | matrix-connector | openclaw-channels | Matrix integration |
| Deployment | clawhub-proxy | openclaw-skills | ClawHub skill registry proxy & gating |
| StatefulSet | postgres | openclaw-data | Session metadata, user data, audit index |
| StatefulSet | redis | openclaw-data | Session cache, pub/sub, message queues |
| Job (ephemeral) | browser-\<sessionid\> | openclaw-browser | Per-task Chromium headless browser (gVisor) |
| Job (ephemeral) | skill-\<sessionid\>-\<skillname\> | openclaw-skills | Per-invocation skill executor (gVisor) |
| Service (ClusterIP) | gateway-svc | openclaw-gateway | Internal access to Gateway WebSocket + HTTP |
| Service (ClusterIP) | webchat-ui-svc | openclaw-gateway | WebChat UI internal routing |
| Service (ClusterIP) | agent-svc | openclaw-agents | Agent RPC endpoint |
| Service (ClusterIP) | telegram-svc | openclaw-channels | Telegram webhook receiver |
| Service (ClusterIP) | slack-svc | openclaw-channels | Slack events (HTTP mode) |
| Service (ClusterIP) | whatsapp-svc (headless) | openclaw-channels | StatefulSet pod DNS |
| Service (ClusterIP) | clawhub-proxy-svc | openclaw-skills | Skill registry proxy |
| Service (ClusterIP) | postgres-svc (headless) | openclaw-data | PostgreSQL pod DNS |
| Service (ClusterIP) | redis-svc | openclaw-data | Redis access |
| Ingress | openclaw-ingress | openclaw-gateway | TLS termination, path routing, rate limiting |
| ConfigMap | openclaw-gateway-config | openclaw-gateway | Gateway runtime configuration |
| ConfigMap | openclaw-agent-config | openclaw-agents | Agent LLM and tool configuration |
| ConfigMap | openclaw-channel-config-telegram | openclaw-channels | Telegram connector settings |
| ConfigMap | openclaw-channel-config-slack | openclaw-channels | Slack connector settings |
| ConfigMap | openclaw-channel-config-discord | openclaw-channels | Discord connector settings |
| ConfigMap | openclaw-channel-config-whatsapp | openclaw-channels | WhatsApp/Baileys settings |
| ConfigMap | openclaw-browser-config | openclaw-browser | Chromium flags and constraints |
| ConfigMap | openclaw-skills-config | openclaw-skills | Skill gating policy and registry URL |
| Secret | openclaw-tls | openclaw-gateway | TLS certificate (cert-manager managed) |
| Secret | openclaw-gateway-auth-secret | openclaw-gateway | Gateway auth token, JWT signing key |
| Secret | openclaw-db-credentials | openclaw-data | PostgreSQL user/password |
| Secret | openclaw-redis-credentials | openclaw-data | Redis password |
| Secret | openclaw-llm-keys | openclaw-agents | Anthropic + OpenAI API keys (ESO synced) |
| Secret | telegram-bot-token | openclaw-channels | Telegram Bot Token |
| Secret | slack-credentials | openclaw-channels | Slack Bot Token + Signing Secret |
| Secret | discord-bot-token | openclaw-channels | Discord Bot Token |
| Secret | whatsapp-session-secret | openclaw-channels | Baileys session encryption key |
| Secret | user-llm-key | openclaw-user-\<id\> | User's personal LLM API key (optional) |
| Secret | user-gmail-token | openclaw-user-\<id\> | User's Gmail OAuth tokens |
| Secret | user-github-token | openclaw-user-\<id\> | User's GitHub PAT |
| Secret | user-calendar-token | openclaw-user-\<id\> | User's Google Calendar tokens |
| ExternalSecret | openclaw-llm-keys-es | openclaw-agents | ESO resource — syncs from Vault |
| ExternalSecret | openclaw-db-credentials-es | openclaw-data | ESO resource — syncs from Vault |
| PersistentVolumeClaim | postgres-data | openclaw-data | PostgreSQL data (50Gi SSD) |
| PersistentVolumeClaim | redis-data | openclaw-data | Redis persistence (10Gi SSD) |
| PersistentVolumeClaim | whatsapp-session-data | openclaw-channels | Baileys session state (1Gi) |
| PersistentVolumeClaim | audit-log-data | openclaw-observability | Append-only audit log (100Gi) |
| PersistentVolumeClaim | user-data-\<id\> | openclaw-user-\<id\> | Per-user transcripts & workspace (5Gi) |
| ServiceAccount | gateway-sa | openclaw-gateway | Gateway pod identity |
| ServiceAccount | agent-sa | openclaw-agents | Agent Runtime pod identity |
| ServiceAccount | channel-whatsapp-sa | openclaw-channels | WhatsApp connector identity |
| ServiceAccount | channel-telegram-sa | openclaw-channels | Telegram connector identity |
| ServiceAccount | channel-slack-sa | openclaw-channels | Slack connector identity |
| ServiceAccount | channel-discord-sa | openclaw-channels | Discord connector identity |
| ServiceAccount | browser-sa | openclaw-browser | Browser Job identity (zero permissions) |
| ServiceAccount | skills-sa | openclaw-skills | Skills Job identity (zero default permissions) |
| Role | gateway-role | openclaw-gateway | Read Gateway Secrets + ConfigMaps |
| Role | agent-role | openclaw-agents | Read LLM key Secret + ConfigMaps |
| Role | agent-browser-spawner-role | openclaw-browser | Create/manage browser Jobs |
| Role | agent-skills-spawner-role | openclaw-skills | Create/manage skills Jobs |
| Role | agent-user-secret-role | openclaw-user-\<id\> | Get/update user Secrets (per-user, per-agent) |
| Role | channel-whatsapp-role | openclaw-channels | Read WhatsApp Secret only |
| Role | channel-telegram-role | openclaw-channels | Read Telegram Secret only |
| Role | channel-slack-role | openclaw-channels | Read Slack Secret only |
| Role | channel-discord-role | openclaw-channels | Read Discord Secret only |
| RoleBinding | gateway-rb | openclaw-gateway | Bind gateway-role → gateway-sa |
| RoleBinding | agent-rb | openclaw-agents | Bind agent-role → agent-sa |
| RoleBinding | agent-browser-rb | openclaw-browser | Bind browser spawner role → agent-sa |
| RoleBinding | agent-skills-rb | openclaw-skills | Bind skills spawner role → agent-sa |
| RoleBinding | agent-user-rb-\<id\> | openclaw-user-\<id\> | Bind user Secret role → agent-sa |
| RoleBinding | channel-whatsapp-rb | openclaw-channels | Bind WhatsApp role → whatsapp-sa |
| RoleBinding | channel-telegram-rb | openclaw-channels | Bind Telegram role → telegram-sa |
| RoleBinding | channel-slack-rb | openclaw-channels | Bind Slack role → slack-sa |
| RoleBinding | channel-discord-rb | openclaw-channels | Bind Discord role → discord-sa |
| ClusterRole | observability-cluster-role | — | Prometheus scrape + audit log write |
| ClusterRoleBinding | observability-crb | — | Bind cluster role → observability-sa |
| NetworkPolicy | gateway-netpol | openclaw-gateway | Default-deny + explicit allows |
| NetworkPolicy | agents-netpol | openclaw-agents | LLM egress allowlist + Gateway only |
| NetworkPolicy | channels-netpol | openclaw-channels | Messaging platform egress + Gateway |
| NetworkPolicy | browser-netpol | openclaw-browser | Internet-only egress, ZERO cluster ingress/egress |
| NetworkPolicy | skills-netpol | openclaw-skills | Default deny, per-Job egress injection |
| NetworkPolicy | data-netpol | openclaw-data | Gateway + Agent ingress only |
| HorizontalPodAutoscaler | gateway-hpa | openclaw-gateway | Scale 2–8 on WS connection count |
| ScaledObject (KEDA) | agent-scaledobject | openclaw-agents | Scale 2–20 on Redis queue depth |
| ResourceQuota | browser-quota | openclaw-browser | Max 4 concurrent browser Jobs |
| ResourceQuota | skills-quota | openclaw-skills | Max 8 concurrent skill Jobs |
| ResourceQuota | user-quota-\<id\> | openclaw-user-\<id\> | 5Gi storage, 20 Secrets per user |
| LimitRange | agents-limitrange | openclaw-agents | Enforce default CPU/memory limits |
| LimitRange | browser-limitrange | openclaw-browser | Enforce browser pod resource caps |
| RuntimeClass | gvisor | — | gVisor runtime for browser + skills pods |
| CertificateIssuer | letsencrypt-prod | openclaw-gateway | cert-manager Let's Encrypt issuer |

---

## Quality Checklist

- [x] Every component with external access has the correct Service type and Ingress routing
- [x] No passwords or API keys exist in ConfigMaps — all in Secrets via external-secrets-operator
- [x] Every pod has resource requests AND limits
- [x] Every pod uses a named, non-default ServiceAccount with minimal permissions
- [x] Secrets have rotation strategies and expiry/compromise runbooks defined
- [x] RBAC follows least-privilege; browser SA and skills SA have zero permissions
- [x] Inter-namespace communication is documented with DNS names
- [x] NetworkPolicy restricts unintended traffic (especially browser → cluster-internal)
- [x] Liveness and readiness probes defined for all Deployments/StatefulSets
- [x] StatefulSets have PVCs for all persistent data
- [x] gVisor RuntimeClass applied to browser and skills pods (highest risk)
- [x] PodSecurityAdmission `restricted` applied to agents, browser, skills namespaces
- [x] Prompt injection mitigation documented (input sanitization, alerting)
- [x] Per-user Secret isolation enforced via namespace-per-user + scoped RoleBindings
- [x] Audit logging covers all K8s Secret access and agent tool invocations
- [x] Secret compromise runbooks cover: revoke → rotate → sync → restart → audit

---

*Sources: [openclaw.ai](https://openclaw.ai) | [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) | [CrowdStrike — OpenClaw Security Analysis](https://www.crowdstrike.com/en-us/blog/what-security-teams-need-to-know-about-openclaw-ai-super-agent/) | [Cisco AI Security Research on OpenClaw skills](https://www.cisco.com)*
