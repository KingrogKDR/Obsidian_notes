The key mental model: none of them hands you a finished FOCUS 1.2 file the way AWS/Nebius do, so for all three we're doing some degree of **assemble-and-enrich** yourself. They differ in how much raw material they give us and how we authenticate.

## Hetzner Cloud

**Auth:** Single API token per project — `Authorization: Bearer <token>` against `api.hetzner.cloud/v1`. Static secret, so same rotation/expiry needs to be applied on it.

**Cost ingestion:** There's no cost export and no FOCUS. What we get:

- The monthly PDF invoice plus a separate "individual consumption statement" (the itemized breakdown) — not machine-friendly for a pipeline.
- The API's server/resource list, which returns each resource with its hourly rate and monthly cap.

So the realistic path is: 

Poll the resource endpoints (`/servers`, `/primary_ips`, `/volumes`, `/load_balancers`, `/floating_ips`), read the embedded prices, and **compute cost ourselves** from "resource × hours running × rate, capped at monthly." We are effectively rebuilding Hetzner's billing engine, including their quirk that hourly usage rounds up and the monthly cap is a `min(cap, hourly_rate×hours)`.

**Utilization ingestion:** Hetzner has a native per-server metrics endpoint: `GET /v1/servers/{id}/metrics` with `type` (cpu / disk / network), a `start`/`end` window, and a `step`. Load balancers have an equivalent. It returns proper time-series, so no third-party exporter needed.

**Enrichment burden:** Highest on cost, lowest on metrics.

**Problem:** The API key that will be handed to us gives us all sorts of permission to that project. So, the customer has to trust us for not misusing the API token and form guardrails around it. So, this violates the principle of least-privilege for us. Also, high level of enrichment requirement.

#OVHcloud

**Authentication:** Application Key + Application Secret + Consumer Key, where the Consumer Key is scoped to specific API routes and HTTP methods when you generate it. That scoping is genuinely useful for us: we can mint a consumer key that's read-only and limited to just the billing/usage routes, which is closer to least-privilege than Hetzner's all-or-nothing token.

**Cost ingestion:** Two options:

- The **FinOps FOCUS tool** in OVH Labs: connect the account, it collects and formats the last 5 invoicing cycles across all Public Cloud projects as FOCUS-compliant CSV we can export. Caveat: it's **FOCUS 1.0** and "as much as possible" conformant — so expect a wider schema gap than Nebius's 1.2, and it's a pull/export, not an auto-drop to a bucket.
- The **raw billing API** (`/me/bill`, `/me/consumption`, and per-project `/cloud/project/{id}/usage/current`), which is updated hourly.

**Utilization ingestion:** OVH exposes monitoring for Public Cloud instances through its observability/metrics offering (Warp10-lineage, Prometheus/OpenTSDB-compatible query interfaces). We'd pull time-series per instance via that endpoint with the same API credentials. But we need to verify the current endpoint naming when we build it because OVH has been migrating its observability products, so there can be breaking changes and experimental features.

**Enrichment burden:** Low-to-medium on cost (FOCUS 1.0 already exists; we mainly upgrade the mapping toward 1.2 and fill gaps), medium on metrics.

**Problem:** Migrating API can cause breaking changes and therefore would require constant monitoring of their API to match ours

## Open Telekom Cloud (T-Systems)

**Auth:** It's OpenStack under the hood (Huawei Cloud lineage), so two options: a **Keystone IAM token** (username / password / domain, token valid 24h) for most calls, or **AK/SK request signing** (access key + secret key, AWS-SigV4-like) which is what the metrics tooling typically uses. Both are the customer's own tenant credentials handed to us. So, these static credentials would need rotation/expiry as well.

**Cost ingestion:** No FOCUS. Two building blocks:

- The **public Pricing API** (no key required — this is what OpenCost uses for OTC) gives us unit prices per flavor/service.
- Resource inventory via the standard OpenStack APIs (Nova for ECS instances, Cinder for EVS volumes, etc.) tells us what's running.

As with Hetzner, we combine inventory × running-hours × published price to synthesize cost records. OTC billing itself is monthly-invoice-based, so there's no line-item cost export to lean on. If the customer's OTC contract includes the BSS/billing query APIs from the Huawei lineage, those can give itemized consumption — worth checking per-customer, as availability varies by contract tier.

**Utilization ingestion:** **Cloud Eye (CES)** is the metrics service. We query it per service-namespace (`SYS.ECS`, `SYS.RDS`, `SYS.ELB`, …) via the CES API (`/V1.0/{project_id}/metric-data` and batch-query variants). There are mature open-source Prometheus exporters (e.g. iits-consulting/otc-prometheus-exporter, tiagoReichert's Cloud Eye exporter) that already wrap this namespace-by-namespace — worth using as a reference implementation or directly, rather than writing the CES polling from scratch.

**Enrichment burden:** High on cost (synthesize from pricing + inventory, like Hetzner), low-to-medium on metrics (CES is capable, exporters exist).

**Problem**: High level of manual enrichment required.

## Cross-cutting: the enrichment layer we'll need to own

Because only OVH gives us anything FOCUS-shaped, our ingestion architecture for these three should assume a **normalization/enrichment stage** that others like AWS/Azure/Nebius didn't need:

| Concern              | Hetzner                            | OVHcloud                             | OTC (T-Systems)                   |
| -------------------- | ---------------------------------- | ------------------------------------ | --------------------------------- |
| FOCUS out of the box | None                               | 1.0 (partial)                        | None                              |
| Cost source          | Compute from resource list + rates | Billing API / FOCUS tool             | Pricing API + OpenStack inventory |
| Metrics source       | Native `/metrics` endpoint         | Observability/metrics platform       | Cloud Eye (CES)                   |
| Auth                 | Bearer token (static)              | App key/secret + scoped consumer key | Keystone token or AK/SK           |
| Your biggest task    | Rebuild cost from usage            | Upgrade FOCUS 1.0→1.2 mapping        | Synthesize cost; wrap CES         |

Concretely, the enrichment work is: (1) map each provider's cost fields onto our canonical FOCUS 1.2 schema — treating missing FOCUS columns as nullable (2) for Hetzner and OTC, run a **rating step** that joins live resource inventory to published unit prices and accumulates running-hours into cost — this is net-new pipeline logic these two need and the bucket-drop providers don't; (3) join utilization by resource ID onto the cost records so a VM's CPU/GPU series lines up with its spend row, which is where "enrich" pays off (cost-per-utilization, idle-resource detection) since none of these three deliver cost and metrics pre-joined.

Two flags worth carrying forward:

- **Currency/FX:** Hetzner and OTC bill in EUR, OVH varies by region/entity. If we normalize to a reporting currency we'll need an FX step — FOCUS has `BillingCurrency`/pricing-currency columns for exactly this, so populate them rather than pre-converting.
- **The rating approach is an approximation.** Synthesizing cost from inventory × published price won't perfectly match the customer's actual invoice (reserved/committed discounts, sub-€0.01 waivers on OVH, Hetzner's round-up rules). Set expectations that Hetzner/OTC "cost" is a modeled estimate reconciled monthly against the real invoice, whereas OVH (from its billing API) and AWS/Nebius are authoritative. That reconciliation gap is worth surfacing in the product rather than hiding.
