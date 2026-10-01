# Atomity Kubernetes Cost Optimization — Roadmap (High Level)

**Status:** Draft v2
**Last updated:** October 2026
**Scope of this document:** what gets built, in what order, and where it
lives. Details are decided inside each iteration.

---

## 1. Priorities and principles

Fixed priority order:

1. EKS optimization (CAST AI is the reference for features and workflows)
2. GKE and AKS (reuse the EKS architecture)
3. Self-managed Kubernetes
4. Anomaly detection across supported environments

Principles carried over from the Phase 1 MVP:

- **Recommend, never execute.** No resizing, no deletion, no changes to
  cluster configuration.
- **Every recommendation is deterministic and explainable.** Rule ID,
  rule version, evidence, savings calculation, confidence, risk.
- **Tenant isolation everywhere.** `organization_id` on every table,
  every query, every agent token.
- **Each iteration ends in a demo.** Nothing is built that cannot be shown.
- **Provider-neutral core, thin provider adapters.** EKS is built
  concretely first; GKE, AKS and self-managed prove the seams.

---

## 2. What already exists

| Existing piece | What the Kubernetes work reuses |
|---|---|
| Cloud connectors + task runner | Billing already lands in ClickHouse `cost_line` for AWS, GCP, Azure, Nebius and Hetzner/OVH (confirm which, see §7). Node prices come from here. |
| ClickHouse client repo | Shared access layer for both new and existing repos. Gets new query methods for Kubernetes tables. |
| Insights repo | Rule engine, Postgres recommendation store, lifecycle (plan to audit, dismiss, reopen, assign, mute), history, compare options, overlap ranking, anomaly and forecast domain. |
| Pricing catalogs (inside insights) | Instance type to vCPU, memory and hourly price. Lives under `internal/`, so it cannot be imported by another repo. |

---

## 3. Architecture

### 3.1 Repositories and responsibilities

| Repo | Owns |
|---|---|
| **`atomity-k8s`** (new) | In-cluster agent, ingestion gateway, Kubernetes ClickHouse tables, usage statistics, cost allocation, provider adapters. Go. |
| **insights** (existing) | Kubernetes rule pack and a Kubernetes reader. Persistence, lifecycle, history, overlap ranking and anomaly detection stay where they are. |
| **cloud connector** (existing) | Unchanged. Remains the only place that talks to cloud accounts. |
| **ClickHouse client** (existing) | New read and write methods for the Kubernetes tables. |

Rule of thumb: if it is about the **cloud account**, it lives in the
connector repo. If it is about the **cluster**, it lives in `atomity-k8s`.
If it is a **judgment about cost** (a recommendation or anomaly), it lives
in insights.

### 3.2 Data flow

```
 Customer cluster                          Atomity
┌──────────────┐  HTTPS (push)  ┌───────────────────┐
│ atomity-agent│───────────────►│ ingestion gateway │
└──────────────┘                └─────────┬─────────┘
                                          ▼
 ┌──────────────────┐   ┌──────────────────────────────────┐
 │ cloud connectors │──►│            ClickHouse            │
 │ (billing, exist.)│   │ cost_line  │  k8s_* tables (new) │
 └──────────────────┘   └───────┬────────────────┬─────────┘
                                │                │
                 allocation job │                │ K8s reader
                 (task runner)  ▼                ▼
                         k8s cost tables    insights rule engine
                                                 │
                                                 ▼
                                      Postgres recommendations
                                      (lifecycle, UI, overlap)
```

The agent only pushes outbound. Customers never expose their cluster
API to Atomity and Atomity never holds cluster credentials.

### 3.3 The two cores

**Provider-neutral core** (`atomity-k8s/core`, plus the K8s rule pack in
insights). Identical for every Kubernetes distribution. Never imports a
cloud SDK or reads a cloud-specific label.

- Agent and its wire protocol
- Ingestion gateway
- Usage statistics (percentiles, observation depth, confidence)
- Cost allocation engine
- Kubernetes ClickHouse schemas
- K8s rule pack and reader (in insights)
- Node-pricing interface

**Provider-specific adapters** (`atomity-k8s/adapters/<provider>`). Each
implements the same small set of responsibilities:

| Responsibility | EKS example | GKE example |
|---|---|---|
| Cluster identity | Link cluster to an existing AWS connection and account | Link to GCP project |
| Node to billing mapping | `spec.providerID` to EC2 instance ID to billed cost lines | Different, see §4 Iteration 7 |
| Node pricing | Billed cost first, list price as fallback | Same pattern, GCP specifics |
| Autoscaler profile | Karpenter, Cluster Autoscaler, EKS Auto Mode | Standard vs Autopilot |
| Control-plane cost | Flat hourly fee | Standard fee, Autopilot per-pod |

Decision rule: "does this change if I switch from EKS to GKE?" Yes means
adapter, no means core. When unsure, put it in the core and let the
second provider prove whether it should move.

### 3.4 Technology

- **Language:** Go everywhere (the Python prototype is only the spec).
- **Agent packaging:** Helm chart, distroless image, separate `go.mod`
  so it can never link a cloud SDK.
- **Storage:** ClickHouse for all Kubernetes data. Postgres only through
  insights.
- **Jobs:** the existing task runner.
- **Local development:** `kind` cluster plus a ClickHouse container.
- **CI guardrails:** import lint so `core/` cannot import `adapters/` or
  any cloud SDK.

---

## 4. Iterations

### Iteration 0 — Foundations and contracts

**Goal:** agree the contracts before writing features.

- Create the `atomity-k8s` repo, CI, import lint, Helm skeleton.
- Define the Kubernetes resource ID convention. It must include the
  cluster ID, because insights identifies a recommendation by
  `(organization_id, rule_id, resource_id)`.
- Draft the ClickHouse table schemas and the agent-to-gateway protocol
  (versioned from day one, since agents stay deployed for months).
- Port the Python allocation prototype and its 9 tests to Go; the tests
  are the specification.

**Demo:** CI green; Go allocation passes the ported tests with identical
numbers.

### Iteration 1 — Agent and ingestion (any cluster)

**Goal:** get cluster data into ClickHouse from any Kubernetes cluster.

- **`atomity-k8s`:** agent (reads kubelet and Kubernetes API, read-only
  RBAC, push-only), ingestion gateway (token maps to organization and
  cluster, batched idempotent writes), cluster registration.
- Agent reports: pods, requests and limits, usage, nodes (instance type,
  `providerID`, capacity), autoscaler and HPA/VPA presence.

**Demo:** install the Helm chart on `kind`, see pods and nodes in
ClickHouse within minutes; remove it with no side effects; prove the
service account cannot write anything.

### Iteration 2 — Usage statistics

**Goal:** turn samples into trustworthy percentiles.

- **`atomity-k8s`:** pre-aggregated 5-minute buckets from the agent,
  p50/p90/p95/p99 per container, observation-depth confidence
  (insufficient, low, moderate, high).
- Prototype percentile computation with ClickHouse quantile aggregate
  states first; build a custom histogram engine only if that falls short.

**Demo:** percentiles per container with a confidence label that grows
as history accumulates.

### Iteration 3 — Cost allocation

**Goal:** turn usage into money over time.

- **`atomity-k8s`:** hourly allocation job on the task runner. Effective
  footprint `max(request, usage)`, separate CPU and memory cost pools,
  idle and overhead as first-class buckets, pod and node lifecycle
  handled by time window.
- Output tables: node cost, pod cost, namespace and workload rollups.
- Node pricing behind an interface; a manual default rate until
  Iteration 4.

**Demo:** namespace cost over 24 hours with workload, overhead and idle
trending; a pod that scaled down stops accruing cost at the right time.

### Iteration 4 — EKS adapter

**Goal:** real dollars on a real EKS cluster.

- **`atomity-k8s/adapters/eks`:** cluster linked to an existing AWS
  connection; node to billed-cost mapping through `providerID`; list
  price fallback for recent hours; control-plane fee; autoscaler profile
  (Karpenter, Cluster Autoscaler, Auto Mode surcharge); on-demand vs
  spot awareness.
- Onboarding flow: connect, install the Helm chart, verify data arriving.

**Demo:** connect a real EKS cluster, see real cost per namespace and
workload, with the control-plane fee as a separate line.

### Iteration 5 — Kubernetes recommendations (EKS)

**Goal:** the first actionable recommendations, in the existing list.

- **insights:** a Kubernetes reader and a rule pack as ordinary rules.
  Each rule holds its own reader, so the shared `CostReader` interface
  and existing fakes stay untouched.
- Initial rules: over-provisioned CPU, over-provisioned memory, unset
  requests, idle workload, node-pool idle capacity.
- Recommendations carry evidence, savings calculation, confidence from
  observation depth, and autoscaler, HPA and VPA notes.
- Each rule returns the full current set per run, because the store
  resolves anything not re-detected.

**Demo:** deliberately over-provisioned workloads produce recommendations
in the existing recommendations UI, with dismissal and reopen working.

Iterations 3 and 5 can overlap, developing rules against `kind` with
default pricing while the EKS adapter is built.

### Iteration 6 — Reconciliation and node-to-pod linkage

**Goal:** make the numbers auditable and connect pod-level and node-level
recommendations.

- **`atomity-k8s`:** compare allocated cost against billed node cost from
  `cost_line`; produce a reconciliation report; optionally use AWS split
  cost allocation data as a second reference.
- **insights:** link pod recommendations to the existing AWS node-level
  recommendations through `OverlapGroup`.

**Demo:** reconciliation report for a billing period with categorized
differences; a node recommendation and its pod recommendations shown
together without double-counted savings.

### Iteration 7 — GKE adapter

**Goal:** prove the core works on a second provider.

- **`atomity-k8s/adapters/gke`:** Standard (same model as EKS) and
  Autopilot (pod-priced, no node layer, allocation bypasses nodes).
- GCP billing rows do not carry machine type names (per comments in the
  insights code), so the node to billing mapping needs its own approach.

**Demo:** EKS and GKE clusters in one organization, one dashboard, the
same rules firing on both.

### Iteration 8 — AKS adapter

**Goal:** complete the three major managed platforms.

- **`atomity-k8s/adapters/aks`:** node auto-provisioning detection, tier
  handling, Azure billing mapping.
- Check whether other connected providers offer managed Kubernetes
  (Nebius, OVH) and add adapters if so.

**Demo:** EKS, GKE and AKS side by side in a single organization.

### Iteration 9 — Self-managed Kubernetes

**Goal:** clusters with no cloud bill.

- **`atomity-k8s/adapters/selfmanaged`:** manual node pricing (per node
  or per group), no reconciliation, no native recommendations.
- Push-only agent means no cluster credentials are needed.
- Hetzner servers can be priced through the existing connector if the
  node mapping works.

**Demo:** a `kubeadm` or `kind` cluster treated as self-managed, with
manually entered prices, in the same dashboard and rules.

### Iteration 10 — Kubernetes anomaly detection

**Goal:** detect unusual cost, usage and infrastructure behavior across
all supported environments.

- **insights:** extend the existing anomaly pipeline with Kubernetes
  dimensions (cluster, node pool, namespace, workload).
- Signals: cost deviation, usage deviation, unexpected node count change,
  restart and OOM spikes (agent reports restart events).

**Demo:** scale a deployment 10x, see an anomaly with the specific
workload and namespace as contributors, plus in-app and email alerts.

### Dependency summary

```
0 → 1 → 2 → 3 ──► 4 → 5 → 6 ──► 7, 8, 9 (parallel) ──► 10
                  └──(5 can start against kind after 3)
```

---

## 5. Cross-cutting decisions

| Decision | Current lean | Resolve by |
|---|---|---|
| Where K8s rules live | In insights as ordinary rules. The `internal/` packages cannot be imported from another repo, so keeping rules in `atomity-k8s` would first require extracting a shared module. | Iteration 0 |
| K8s resource ID format | Includes cluster ID, unique within the organization | Iteration 0 |
| Raw K8s telemetry table | Own pre-aggregated tables, not `resource_metric` (narrow per-sample schema, no dimension columns) | Iteration 0 |
| Node utilization into `resource_metric` | Avoid until K8s-aware. The generic underutilised-compute rule would call a node idle on actual usage while the scheduler sees it as full. | Iteration 5 |
| Recommendation categories | Reuse the existing ten, or add Kubernetes categories (requires domain and UI changes) | Iteration 5 |
| Overlap semantics | Today winner-takes-all (losers marked DUPLICATE). Decide whether pod recommendations join the node's group or form their own. | Iteration 6 |
| Query methods location | In the shared ClickHouse client so both repos can use them | Iteration 1 |
| Percentile method | ClickHouse quantile states first, custom histogram only if needed | Iteration 2 |
| Node price source | Billed cost via `providerID` first. List-price fallback needs either a copy of the catalog, a shared module, or a self-hosted price API. | Iteration 4 |
| True-up policy | Raw allocation with explicit reconciliation, or scale to match the bill. Product decision. | Iteration 6 |

---

## 6. Out of scope

- Any automated execution (resizing, deletion, purchases, config changes)
- Terraform or Pulumi integration for onboarding (Helm only for now)
- Prometheus and CloudWatch backfill (optional later accelerator for the
  cold-start window)
- Application-performance optimization
- Compliance and sovereignty scoring

---

## 7. Open items to verify

1. Does `resource_metric` receive data today? Comments in the insights
   code say it does not; the integration status says metrics are in.
2. Hetzner or OVH? The insights code has an OVH catalog and no Hetzner.
3. Does `spec.providerID` reliably match billing instance IDs on each
   provider? Test on a real cluster before relying on it.
4. Does the pricing catalog cover the instance types Karpenter selects?
5. Which service serves the cost dashboards that will show Kubernetes
   cost views?
6. Does the `go` directive in the insights `go.mod` say 1.22 or later?
   `OverlapGroup: &m.ResourceID` takes the address of a loop variable
   field, which is only safe from 1.22.
