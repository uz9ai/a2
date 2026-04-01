---
name: k8s-plan
description: Generate a comprehensive Kubernetes architecture plan for any application. Covers Deployments/StatefulSets, Services, ConfigMaps, Secrets, Namespaces, RBAC, resource limits, and inter-service communication. Use when designing K8s blueprints, planning microservice deployments, or architecting cloud-native applications.
argument-hint: [project-name or description]
disable-model-invocation: false
---

# Kubernetes Architecture Planning Skill

You are an expert Kubernetes architect. When this skill is invoked, produce a **detailed Kubernetes architecture plan** — a blueprint only, no YAML code unless explicitly requested.

## Clarifying Questions Protocol

Before producing the plan, assess whether the following are already clear from context. If **any** critical information is missing, ask the user grouped clarifying questions (ask all at once, not one at a time):

### Critical (always resolve if unknown):
1. **Workload type** — stateless services? Stateful databases? Background workers? AI agents?
2. **Data persistence** — which components need persistent storage (databases, file uploads, model weights)?
3. **External exposure** — which services need to be reachable from the internet vs internal-only?
4. **Environment** — development, staging, production, or multi-env? Single cluster or multi-cluster?
5. **Scale expectations** — expected concurrent users / RPS? Any burst traffic patterns?
6. **Sensitive data** — what API keys, passwords, tokens, or credentials does each component need?

### Important (ask if unclear):
7. **Auth model** — does the app use OAuth, JWT, API keys, mTLS between services?
8. **AI/LLM usage** — which components call LLM APIs (OpenAI, Anthropic, etc.)? Key rotation needs?
9. **Notification/messaging** — email, push, webhooks, message queues (Kafka/RabbitMQ)?
10. **Multi-tenancy** — does the app serve multiple tenants that need isolation?
11. **Observability** — logging stack, metrics, tracing requirements?

If context is rich enough (e.g., the user has described all components), proceed directly to the plan.

---

## Plan Structure to Produce

Produce the plan as a Markdown document with the following sections:

---

### 1. Overview & Architecture Summary
- High-level description of components and their relationships
- Communication flow diagram (ASCII)
- Namespace layout rationale

---

### 2. Namespace Design
For each namespace, specify:
- Name and purpose
- What workloads live there
- Network isolation policy (can it talk to other namespaces?)

**Decision rules:**
- Separate namespaces for: frontend, backend, data-layer, agents/AI, observability, security/secrets
- Use `default` only for non-sensitive dev/test throwaway work
- Production workloads MUST be in named namespaces

---

### 3. Pods & Workloads

For each component, define:

| Field | Decision |
|---|---|
| **Workload type** | Deployment (stateless), StatefulSet (stateful/DB), DaemonSet (node-level), Job/CronJob (batch) |
| **Replicas** | Minimum and maximum, justification |
| **Container image** | Logical name (e.g., `task-manager/backend:latest`) |
| **Init containers** | DB migrations, secret pre-checks, dependency readiness |
| **Liveness probe** | HTTP/TCP/exec endpoint + timing |
| **Readiness probe** | HTTP/TCP/exec endpoint + timing |
| **Restart policy** | Always / OnFailure / Never |

**Decision rules:**
- REST APIs and UIs → Deployment
- Databases (PostgreSQL, Redis, MongoDB) → StatefulSet with PVC
- AI agents doing long-running work → Deployment with resource limits tuned for LLM calls
- One-time setup (schema migration) → Job

---

### 4. Services & Exposure

For each service, specify type and justification:

| Service Type | When to Use |
|---|---|
| **ClusterIP** | Internal-only: backend APIs, databases, agents, notification service |
| **NodePort** | Dev/testing only — avoid in production |
| **LoadBalancer** | Production external entry point when not using Ingress |
| **Headless** | StatefulSets needing direct pod DNS (databases, Kafka) |
| **ExternalName** | Proxy to an external DNS/SaaS service |

Then specify **Ingress rules** for any externally exposed services:
- Ingress class (nginx, traefik, cloud ALB)
- Host/path routing rules
- TLS termination (cert-manager, ACM)
- Rate limiting annotations

---

### 5. Resource Requests & Limits

For each workload, specify:

```
component: <name>
  requests:
    cpu: <value>      # guaranteed allocation
    memory: <value>   # guaranteed allocation
  limits:
    cpu: <value>      # hard cap (throttle if exceeded)
    memory: <value>   # hard cap (OOM kill if exceeded)
  justification: <why these values>
```

**Decision rules:**
- LLM API callers: high memory (waiting on HTTP), low CPU; set generous timeouts
- Database pods: high memory for buffer pool; CPU varies
- UI/frontend: low resources
- Agents processing in-memory: scale memory with context window size
- Always set both requests AND limits to avoid noisy-neighbor issues
- Use `LimitRange` in namespace to enforce defaults

---

### 6. ConfigMaps

For each ConfigMap, specify:
- Name and namespace
- What configuration it holds (keys, not values)
- Which pods mount it (volume mount vs envFrom)
- Update strategy (rolling restart needed? use `Reloader` annotation)

**Decision rules — what goes in ConfigMap:**
- Application feature flags
- Non-sensitive environment variables (LOG_LEVEL, PORT, REGION)
- Database hostnames/ports (not credentials)
- LLM model names, temperature, max_tokens settings
- Service discovery URLs (internal ClusterIP names)
- Retry/timeout/backoff configuration

**What does NOT go in ConfigMap:** passwords, API keys, tokens, certs → use Secrets

---

### 7. Secrets Management

#### Secret Classification

| Secret Type | Storage Approach |
|---|---|
| LLM API keys (OpenAI, Anthropic) | K8s Secret + external secret manager |
| Database credentials | K8s Secret, rotated via controller |
| JWT signing keys | K8s Secret with rotation policy |
| TLS certificates | cert-manager managed Secret |
| OAuth client secrets | K8s Secret + sealed secrets |
| Service-to-service tokens | SPIFFE/SPIRE or mTLS, short-lived |

#### For each Secret, specify:
- Name, namespace, type (`Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`)
- Keys stored (not values — never put actual secrets in the plan)
- Which pods consume it (env var injection vs volume mount)
- Rotation strategy and expiry handling

#### Secret Expiry & Compromise Handling

**Rotation strategy:**
1. External secret managers (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) with `external-secrets-operator` for automatic sync
2. Set `SECRET_EXPIRY_DAYS` annotation for tracking
3. Use `Reloader` controller to trigger rolling restarts on secret updates without downtime

**On compromise/expiry:**
1. Immediately revoke the secret at the source (LLM provider dashboard, DB)
2. Update the secret in the external manager
3. `external-secrets-operator` syncs to K8s within its refresh interval (configurable, e.g., 1h)
4. `Reloader` detects the K8s Secret change and triggers a rolling restart
5. Old pods terminate after new pods pass readiness checks (zero downtime)
6. Audit: check which pods consumed the compromised secret via RBAC audit logs

**For AI agent access:**
- Agents should never have cluster-admin or broad Secret read access
- Use dedicated ServiceAccounts with minimal RBAC — only the specific Secrets they need
- Consider IRSA (IAM Roles for Service Accounts) on AWS or Workload Identity on GCP to avoid static credentials entirely

---

### 8. Persistent Storage (if applicable)

For stateful components:
- PersistentVolumeClaim name and size
- StorageClass (fast SSD for DB, standard for logs)
- Access mode (ReadWriteOnce for DB, ReadWriteMany for shared)
- Backup strategy (Velero, cloud snapshots)

---

### 9. Namespace & RBAC Design

#### Namespace Layout

Define a namespace-per-concern pattern. Example for a 3-tier app:

```
<app>-frontend     → UI, static assets
<app>-backend      → APIs, business logic
<app>-agents       → AI agents, background workers
<app>-data         → databases, caches, queues
<app>-notifications → notification services
<app>-observability → Prometheus, Grafana, Jaeger
```

#### RBAC Design

For each **ServiceAccount**, specify:

| ServiceAccount | Namespace | Permissions (Roles) | Justification |
|---|---|---|---|
| `ui-sa` | frontend | read ConfigMaps | config only |
| `backend-sa` | backend | read Secrets (own), read/write data namespace Services | needs DB creds + calls data layer |
| `agent-sa` | agents | read Secrets (LLM keys only), write to backend API | minimal blast radius |
| `notification-sa` | notifications | read Secrets (SMTP/push keys), list Pods in own NS | send notifications |
| `ci-deploy-sa` | all | create/update Deployments, ConfigMaps | CI/CD pipeline only |

**ClusterRoles vs Roles:**
- Prefer `Role` (namespace-scoped) over `ClusterRole` for application workloads
- Only use `ClusterRole` for: cluster-wide monitoring, ingress controllers, cert-manager
- Never bind `cluster-admin` to application service accounts

**Principle of Least Privilege checklist:**
- [ ] Each SA can only read Secrets it owns
- [ ] No SA has `*` verbs on any resource
- [ ] Agents cannot list all Secrets in the cluster
- [ ] Database pods have no RBAC permissions at all (they don't need K8s API access)

---

### 10. Inter-Service Communication

For each service-to-service call, document:

| From | To | Protocol | Discovery | Auth |
|---|---|---|---|---|
| UI | Backend API | HTTPS | `backend-svc.backend.svc.cluster.local` | JWT bearer |
| UI | Todo Agent | WebSocket/HTTP | `agent-svc.agents.svc.cluster.local` | JWT bearer |
| Backend API | Database | TCP | `postgres-svc.data.svc.cluster.local:5432` | DB credentials (Secret) |
| Backend API | Redis | TCP | `redis-svc.data.svc.cluster.local:6379` | Redis password (Secret) |
| Agent | Backend API | HTTPS | `backend-svc.backend.svc.cluster.local` | Internal SA token |
| Notification | Backend API | HTTPS | `backend-svc.backend.svc.cluster.local` | Internal SA token |

**Network Policies:**
- Define `NetworkPolicy` for each namespace to whitelist ingress/egress
- Default-deny all ingress within a namespace, then explicitly allow
- Agents namespace: allow egress to internet (LLM API calls) but restrict ingress to backend-only

---

### 11. Observability & Health

- **Logging**: structured JSON logs, forwarded via Fluent Bit DaemonSet → central log store
- **Metrics**: Prometheus scrape annotations on each pod (`prometheus.io/scrape: "true"`)
- **Tracing**: OpenTelemetry sidecar or SDK instrumentation, export to Jaeger/Tempo
- **Alerts**: PodCrashLooping, HPA max replicas reached, Secret expiry approaching

---

### 12. Autoscaling

For each Deployment, specify HPA policy:

```
component: <name>
  HPA:
    minReplicas: N
    maxReplicas: M
    scaleOn: CPU at 70% | memory at 80% | custom metric (queue depth)
```

---

### 13. Summary Table

Produce a final summary table listing every K8s object in the plan:

| Object Kind | Name | Namespace | Purpose |
|---|---|---|---|
| Namespace | backend | — | Backend API isolation |
| Deployment | backend-api | backend | REST API server |
| Service (ClusterIP) | backend-svc | backend | Internal access to backend |
| ConfigMap | backend-config | backend | App config, model settings |
| Secret | backend-secrets | backend | DB password, JWT key |
| ServiceAccount | backend-sa | backend | Pod identity |
| Role | backend-role | backend | Read own Secrets |
| RoleBinding | backend-rb | backend | Bind role to SA |
| ... | | | |

---

## Quality Checklist

Before finalizing the plan, verify:
- [ ] Every component that needs external access has the right Service type
- [ ] No passwords or API keys exist in ConfigMaps
- [ ] Every pod has resource requests AND limits
- [ ] Every pod uses a non-root ServiceAccount (not `default`)
- [ ] Secrets have rotation strategies defined
- [ ] RBAC follows least-privilege
- [ ] Inter-namespace communication is documented
- [ ] Network policies restrict unintended traffic
- [ ] Liveness and readiness probes are defined for all Deployments
- [ ] StatefulSets have PVCs for persistent data
