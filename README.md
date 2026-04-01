# Deliverables

## Skill: `/k8s-plan`

**Location:** `.claude/skills/k8s-plan/SKILL.md` (312 lines)

A comprehensive Kubernetes planning skill that:

- Asks grouped clarifying questions when context is incomplete (workload type, persistence, exposure, scale, sensitive data, auth model)
- Produces plans covering all 13 required areas: Deployments/StatefulSets, Services, resource limits, ConfigMaps, Secrets, Namespaces, RBAC, inter-service communication, Network Policies, Autoscaling, Observability, PVCs, and Summary Tables
- Enforces decision rules (e.g., ConfigMaps for config — never credentials; external secret managers + ESO for all API keys)

---

## Plan 1: AI Native Task Manager

**File:** [`plans/scenario1-ai-task-manager.md`](plans/scenario1-ai-task-manager.md) (83KB, 1403 lines)

Covers the 4-component architecture (UI ↔ Backend API ↔ Todo Agent ↔ Notification Service) with full K8s blueprint including dedicated namespaces, PostgreSQL/Redis StatefulSets, RBAC per-component, and inter-service WebSocket/HTTP communication patterns.

---

## Plan 2: OpenClaw Personal AI Employee

**File:** [`plans/scenario2-openclaw-ai-employee.md`](plans/scenario2-openclaw-ai-employee.md) (65KB, 1304 lines)

Based on actual OpenClaw architecture (Node.js Gateway + Pi Agent Runtime + Channel Connectors hub-and-spoke model). Key security highlights:

- **gVisor RuntimeClass** on browser and skills pods — Chromium exploits cannot reach the host kernel
- **Zero-permission browser SA** — browser pods cannot call the K8s API or reach internal services
- **Namespace-per-user** for Secret isolation — User A's Gmail/GitHub tokens are cryptographically inaccessible to User B's agent
- **Just-in-time Secret injection** — agents only get the specific credential needed for the current step
- **Full compromise runbooks** — for LLM key leak, channel token steal, WhatsApp session compromise, and agent pod compromise
- **Prompt injection mitigation** — Gateway-level scanning + alerting (addressing the known Cisco-reported vulnerability)
