## Production-Grade Technical Plan

Document purpose: Define every aspect of the Hetzner Cloud connector from authentication through cost computation, FOCUS normalisation, metrics ingestion, recommendations, and operational hardening.

---

## 1. Provider Profile

Hetzner Cloud is a German cloud provider (Hetzner Online GmbH) offering compute, block storage, networking, load balancing, DNS, and object storage across six locations: Falkenstein (fsn1), Nuremberg (nbg1), Helsinki (hel1), Ashburn (ash), Hillsboro (hil), and Singapore (sin). All billing is in EUR, monthly in arrears, with hourly granularity capped at a monthly maximum per resource.

Three facts shape this connector's design:

1. There is no billing export and no cost API. Hetzner's Cloud API is a resource-management API — it can list resources and their embedded pricing, but it does not expose invoices, consumption history, or cost line items programmatically. The monthly invoice PDF and its companion "individual consumption statement" (downloadable as CSV from the web console) have no API endpoint.
    
2. The API token model is binary: read-only or read-write, scoped to a single project, with no fine-grained IAM, no temporary credentials, no IP allowlisting, and no token expiry. A read-only token can only issue GET requests.
    
3. The API rate limit is 3,600 requests per hour per token, tracked via `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` response headers.
    

---

## 2. Authentication

### 2.1 Token type

Atomity must require a **read-only** API token. This restricts the token to GET requests only, which prevents Atomity from creating, deleting, modifying, or stopping any customer resource.

### 2.2 Project-scoped tokens

Each Hetzner Cloud API token is bound to exactly one project. A customer with multiple projects must create one read-only token per project and register each as a separate Atomity connection. Atomity must surface this clearly in the onboarding wizard: "Hetzner Cloud tokens are scoped to a single project. To monitor multiple projects, add a connection for each."

### 2.3 Token generation guidance

The onboarding wizard must provide step-by-step instructions:

1. Open console.hetzner.cloud and select the target project.
2. Navigate to Security → API Tokens.
3. Click "Generate API Token."
4. Enter a description (e.g., "Atomity cost visibility — read only").
5. Select **Read** permission (not Read & Write).
6. Copy the token immediately — Hetzner displays it only once.

### 2.4 Credential storage

The token must be stored in Atomity's dedicated secret-management system (as per SEC-002). It must never appear in application logs, frontend local storage, or API responses.

### 2.5 Token validation

On connection creation and on every scheduled sync, Atomity must validate the token by calling `GET /v1/servers?per_page=1`. A successful response confirms the token is valid and has read access. A `401 Unauthorized` response means the token is invalid or revoked. A `403 Forbidden` means the token exists but lacks sufficient permissions (this should not occur for a read-only token calling a GET endpoint, but should be handled).

### 2.6 Least-privilege gap and mitigations

**Problem:** A read-only Hetzner token grants read access to every resource type in the project — servers, volumes, networks, firewalls, SSH keys, certificates, images, and more. There is no way to create a token that can only read servers and pricing but not SSH keys or firewall rules.

**Mitigations:**

- Atomity must only call the endpoints it needs (enumerated in Section 5). It must not call endpoints for SSH keys, certificates, firewalls, images, ISOs, or networks.
- The onboarding wizard must display a transparency notice: "The Hetzner Cloud API token grants read access to all resources in this project. Atomity will only read resource inventory, pricing, labels, and metrics. It will not read SSH keys, firewall rules, certificates, or network configurations."
- The audit log must record every distinct Hetzner API endpoint called, so customers can verify Atomity's access pattern.
- Atomity should document a recommendation that customers create a dedicated Hetzner project for billing visibility if they want to limit exposure, although this is rarely practical.

---

## 3. Cost Ingestion Strategy

### 3.1 The core problem

Hetzner has no billing API, no cost export, and no FOCUS support. The only machine-accessible cost data is the pricing embedded in resource and type endpoints. This means Atomity must **compute cost from resource inventory and pricing rules**, effectively reconstructing Hetzner's billing engine.

### 3.2 Dual-mode ingestion

**Mode A: Computed cost (primary, automated)**

Poll resource inventory endpoints at a scheduled interval. For each resource, calculate cost using Hetzner's published pricing rules. This is the default and automatic mode.

**Mode B: Invoice CSV upload (reconciliation, manual)**

The customer may upload the individual consumption statement CSV (downloadable from Hetzner's web console under Invoices → Overview → Download as CSV). This serves as a reconciliation source and provides historically accurate billing data. Manual upload remains mandatory as a fallback because no programmatic access to invoices exists.

### 3.3 Hetzner billing rules (must be faithfully implemented)

**Rule 1 — Hourly billing with monthly cap:** `cost = min(monthly_cap, hourly_rate × hours_used)`

**Rule 2 — Partial hours round up:** A resource created for 5 minutes is billed for 1 full hour.

**Rule 3 — Billing runs from creation to deletion:** A server that is powered off but not deleted continues to accrue charges. Volumes, primary IPs, and floating IPs similarly bill until deleted.

**Rule 4 — Location-based pricing:** Prices vary by location (fsn1, nbg1, hel1, ash, hil, sin). The pricing endpoint and server_type endpoint both return per-location prices.

**Rule 5 — Traffic overage:** Each server plan includes a monthly outbound traffic allowance (varying by location: 20 TB for EU, lower for US/SG). Overage is billed at €1/TB for EU-Central locations; higher for Singapore (approximately €7.40/TB). Inbound traffic is free.

**Rule 6 — Backups:** Server backups are priced as a percentage surcharge on the server's base price (currently 20%).

**Rule 7 — Snapshots:** Billed per GB/month of snapshot storage consumed.

**Rule 8 — Currency:** All Hetzner pricing is in EUR. Hetzner does not publish USD prices; any USD amounts are derived conversions.

**Rule 9 — VAT:** The API returns both net and gross prices. Atomity must store both but use **net** prices for cross-provider cost analytics (consistent with how AWS reports pre-tax costs).

### 3.4 Computed-cost calculation engine

For each sync cycle, the engine must:

1. Fetch the current global pricing from `GET /v1/pricing` (cached for up to 24 hours, refreshed on any detected change).
2. Fetch server types from `GET /v1/server_types` for per-location hourly and monthly rates, included traffic, and traffic overage pricing.
3. Fetch load balancer types from `GET /v1/load_balancer_types` for per-location pricing and included traffic.
4. Enumerate all active resources (servers, volumes, primary IPs, floating IPs, load balancers).
5. For each resource, compute accrued cost since the last sync (or since resource creation if first seen).
6. Apply the monthly cap rule: at month boundary, clamp cumulative hourly cost to the monthly cap.
7. Persist the computed cost lines.

**Resource lifecycle tracking:**

Atomity must maintain a local resource state table that records:

- Resource ID, type, name, labels.
- First seen timestamp (from the resource's `created` field).
- Last seen timestamp (from the most recent sync).
- Deletion detected timestamp (resource absent from API response).
- Server type / volume size / IP type at each observation (to detect resizes).

When a resource disappears from the API response, it has been deleted. Atomity must record the deletion time as the last-seen timestamp and compute final charges through that point.

### 3.5 Billable resource types

|Resource|API Endpoint|Pricing source|Billing model|
|---|---|---|---|
|Cloud Server|`GET /v1/servers`|`server_type.prices[]`|Hourly, monthly cap|
|Volume|`GET /v1/volumes`|`GET /v1/pricing` → `volume`|Per GB/month, hourly|
|Primary IP|`GET /v1/primary_ips`|`GET /v1/pricing` → `primary_ips`|Hourly, monthly cap|
|Floating IP|`GET /v1/floating_ips`|`GET /v1/pricing` → `floating_ips`|Monthly|
|Load Balancer|`GET /v1/load_balancers`|`load_balancer_type.prices[]`|Hourly, monthly cap|
|Server Backup|Derived from server|% surcharge on server base price|Included in server cost|
|Snapshot|`GET /v1/images?type=snapshot`|`GET /v1/pricing` → `image`|Per GB/month|
|Traffic Overage|Derived from server metrics|`server_type.prices[].price_per_tb_traffic`|Per TB over included|

### 3.6 Invoice CSV upload workflow

When a customer uploads the Hetzner consumption CSV:

1. Parse and validate the file.
2. Show the user: detected billing period, currency (EUR), row count, recognised fields, unmapped fields, invalid rows, and duplicate-file status.
3. The user confirms or cancels.
4. On confirmation, ingest as authoritative billing records that supersede any computed cost for the same billing period.
5. Run reconciliation (Section 9).

---

## 4. API Endpoints Used

Atomity's Hetzner connector must call only the following endpoints. No other endpoints are permitted.

### 4.1 Resource inventory

|Endpoint|Purpose|Pagination|
|---|---|---|
|`GET /v1/servers`|Server inventory, server_type, status, labels, created, location|Yes, `per_page` + `page`|
|`GET /v1/volumes`|Volume inventory, size, server attachment, labels, location|Yes|
|`GET /v1/primary_ips`|Primary IP inventory, type (ipv4/ipv6), assignee, labels, location|Yes|
|`GET /v1/floating_ips`|Floating IP inventory, type, server assignment, labels, location|Yes|
|`GET /v1/load_balancers`|Load balancer inventory, type, targets, labels, location|Yes|
|`GET /v1/images?type=snapshot`|Snapshot inventory for billing (size in GB)|Yes|

### 4.2 Pricing reference

|Endpoint|Purpose|Cache TTL|
|---|---|---|
|`GET /v1/pricing`|Global pricing for all resource types (volumes, IPs, images, traffic)|24 hours|
|`GET /v1/server_types`|Per-location hourly and monthly prices, included traffic, traffic overage|24 hours|
|`GET /v1/load_balancer_types`|Per-location hourly and monthly prices, included traffic, traffic overage|24 hours|

### 4.3 Metrics

|Endpoint|Purpose|Rate consideration|
|---|---|---|
|`GET /v1/servers/{id}/metrics`|CPU, disk, and network time-series for a specific server|1 call per metric type per server|
|`GET /v1/load_balancers/{id}/metrics`|Load balancer metrics|1 call per load balancer|

### 4.4 Metadata

|Endpoint|Purpose|
|---|---|
|`GET /v1/locations`|Location names, cities, country codes|

### 4.5 Rate-limit budget

With 3,600 requests/hour and typical sync needs:

**Inventory + pricing (per sync):** ~10 requests for a small project (< 25 servers), up to ~50 for a larger project (100+ servers with pagination). This is negligible.

**Metrics (per sync):** 3 calls per server (cpu + disk + network) + 1 per load balancer. A project with 50 servers and 5 LBs = 155 metric calls per sync.

**Budget:** At hourly syncs, a 50-server project uses ~205 of 3,600 requests/hour (~6%). At 15-minute inventory syncs with hourly metric syncs, budget remains well within limits.

**Safeguard:** The connector must read `RateLimit-Remaining` from every response. If remaining falls below 360 (10% of budget), the connector must pause and schedule retry after the `RateLimit-Reset` timestamp. This must be logged as a degraded-connection event.

---

## 5. FOCUS Normalisation

### 5.1 FOCUS field mapping

Hetzner has no native FOCUS support. Every FOCUS field must be mapped from API response fields.

|FOCUS Field|Hetzner Source|Notes|
|---|---|---|
|BillingAccountId|Hetzner project ID (derived from token scope)|One token = one project|
|BillingAccountName|Connection name in Atomity|Hetzner API does not expose project name|
|BillingCurrency|`EUR` (always)|Hardcoded|
|BillingPeriodStart|First day of calendar month|Hetzner bills monthly|
|BillingPeriodEnd|Last day of calendar month||
|ChargePeriodStart|Resource `created` or period start (whichever is later)||
|ChargePeriodEnd|Deletion time or period end (whichever is earlier)||
|ServiceName|Mapped from resource type (see 5.2)||
|ServiceCategory|Mapped from resource type (see 5.2)||
|RegionId|`server.location.name` (e.g., `fsn1`)||
|RegionName|`server.location.description` (e.g., `Falkenstein 1 DC14`)||
|AvailabilityZone|Not applicable|Hetzner does not expose AZs|
|ResourceId|`server.id`, `volume.id`, etc.|Numeric IDs, cast to string|
|ResourceName|`server.name`, `volume.name`, etc.||
|ResourceType|`server.server_type.name` (e.g., `cx22`), `volume`, `primary_ip`, etc.||
|ChargeCategory|`Usage` for all compute/storage; `Tax` if gross-net delta is surfaced||
|ChargeFrequency|`Usage-Based`||
|ConsumedQuantity|Hours used (for compute), GB-months (for storage)|Atomity-computed|
|ConsumedUnit|`Hours`, `GB-Month`, `TB`||
|ListUnitPrice|`price_hourly.net` from server_type/pricing||
|ListCost|Atomity-computed: quantity × unit price||
|BilledCost|ListCost (no negotiated discounts in Hetzner Cloud)||
|EffectiveCost|BilledCost (no amortisation constructs)||
|Tags|`server.labels` (key-value pairs)|Labels in Hetzner = Tags in FOCUS|
|CommitmentDiscountId|Not applicable|Hetzner has no RIs or Savings Plans|
|SubAccountId|Not applicable|Single project per token|

### 5.2 Service category mapping

|Hetzner Resource|FOCUS ServiceName|FOCUS ServiceCategory|
|---|---|---|
|Cloud Server|Hetzner Cloud Server|Compute|
|Volume|Hetzner Block Storage|Storage|
|Primary IP (IPv4)|Hetzner Primary IPv4|Networking|
|Primary IP (IPv6)|Hetzner Primary IPv6|Networking|
|Floating IP|Hetzner Floating IP|Networking|
|Load Balancer|Hetzner Load Balancer|Networking|
|Snapshot|Hetzner Snapshot|Storage|
|Backup|Hetzner Server Backup|Storage|
|Traffic Overage|Hetzner Traffic Overage|Networking|

### 5.3 Provider-native attributes

All fields from the Hetzner API response that are not mapped to FOCUS must be preserved in `provider_native_attributes` JSON. This includes at minimum:

- `server.server_type.architecture` (x86 or arm64)
- `server.server_type.cpu_type` (shared or dedicated)
- `server.server_type.cores`
- `server.server_type.memory` (GB)
- `server.server_type.disk` (GB)
- `server.status` (running, off, initializing, etc.)
- `server.public_net` details
- `volume.format` (ext4, xfs)
- `volume.linux_device`
- `load_balancer.algorithm.type`
- `load_balancer.targets` (count and types)
- Full `labels` object
- `server.placement_group`

### 5.4 Frontend terminology mapping

|Primary frontend label (FOCUS)|Secondary provider label (Hetzner)|
|---|---|
|Sub-account|Hetzner: Project|
|Service|Hetzner: Product/Resource Type|
|Resource ID|Hetzner: Resource ID|
|Region|Hetzner: Location|
|Effective Cost|Hetzner: Computed net cost|
|Tags|Hetzner: Labels|

---

## 6. Metrics Ingestion

### 6.1 Available metrics

Hetzner's native per-server metrics endpoint (`GET /v1/servers/{id}/metrics`) provides hypervisor-level time-series data:

|Metric type|Series returned|Unit|
|---|---|---|
|`cpu`|CPU utilisation percentage|% (0–100 per vCPU)|
|`disk`|`disk.0.iops.read`, `disk.0.iops.write`, `disk.0.bandwidth.read`, `disk.0.bandwidth.write`|IOPS, bytes/s|
|`network`|`network.0.bandwidth.in`, `network.0.bandwidth.out`, `network.0.pps.in`, `network.0.pps.out`|bytes/s, packets/s|

Load balancers have an equivalent metrics endpoint: `GET /v1/load_balancers/{id}/metrics`.

### 6.2 Limitations

- **No memory utilisation.** Hetzner metrics are measured outside the guest at the hypervisor level. Memory usage is not exposed.
- **No disk utilisation (capacity).** Only IOPS and throughput are available, not percentage of disk space used.
- These are the same limitations as the Hetzner Cloud Console graphs.

### 6.3 Collection strategy

- **Frequency:** Collect daily aggregated metrics. The API accepts `start`, `end` (RFC 3339), and `step` parameters. Use a 24-hour window with a `step` of 3600 (hourly data points) for daily collection.
- **Retention in Hetzner:** The API retains metrics for approximately 30 days.
- **Atomity storage:** Store in the `ResourceMetric` entity with daily granularity (daily max, daily average, daily p95 where calculable from the hourly points).
- **Rate-limit awareness:** Metrics collection is the most request-intensive operation. For a project with N servers, each daily collection requires 3N + M calls (3 metric types × N servers + M load balancers). Schedule metrics collection during off-peak hours and space calls to stay within budget.

### 6.4 Traffic metering for overage computation

Network metrics provide the raw bytes in/out. To compute traffic overage:

1. Sum `network.0.bandwidth.out` over the billing month for each server (outbound only; inbound is free).
2. Compare against the server plan's included traffic allowance (from `server_type.included_traffic`, returned in bytes).
3. If total outbound exceeds the allowance, the overage is billed at the per-TB rate from `server_type.prices[].price_per_tb_traffic`.

Note: The traffic allowance is pooled across all servers in a project (as confirmed by Hetzner documentation). The connector must sum outbound across all servers before comparing against the sum of all included traffic allowances.

---

## 7. Resource Inventory

### 7.1 Supported resource types

|Resource|Hetzner API|Fields stored|
|---|---|---|
|Cloud Server|`GET /v1/servers`|id, name, status, server_type (name, cores, memory, disk, architecture, cpu_type), location, labels, created, public_net, volumes, load_balancers|
|Volume|`GET /v1/volumes`|id, name, size (GB), server (attached to), location, labels, created, format, status|
|Primary IP|`GET /v1/primary_ips`|id, name, ip, type (ipv4/ipv6), assignee_id, assignee_type, location, labels, created|
|Floating IP|`GET /v1/floating_ips`|id, name, ip, type, server (assigned to), location, labels, created|
|Load Balancer|`GET /v1/load_balancers`|id, name, load_balancer_type, location, labels, created, algorithm, targets, services, ingoing/outgoing traffic|

### 7.2 Sync cadence

- **Resource inventory:** Every 15 minutes (detects creations and deletions promptly).
- **Pricing reference:** Every 24 hours (Hetzner price changes are infrequent and announced in advance).
- **Metrics:** Every 24 hours (daily aggregation).
- **Cost computation:** Every hour (applies latest inventory state against pricing).

---

## 8. Recommendations

### 8.1 Applicable recommendation categories

Given the metrics and inventory data available, the Hetzner connector can support these recommendation categories:

|#|Category|Detection method|Confidence|
|---|---|---|---|
|1|Underutilised compute|CPU utilisation consistently below threshold (default: 10% average over 14 days)|Medium — no memory data|
|2|Oversized compute|CPU consistently low + server_type has more cores/RAM than workload needs|Medium|
|3|Stopped server with continuing charges|`server.status == "off"` for > 24 hours|High|
|4|Unattached volume|`volume.server == null` for > 7 days|High|
|5|Unassigned floating IP|`floating_ip.server == null` for > 7 days|High|
|6|Unassigned primary IP|`primary_ip.assignee_id == null` for > 7 days|High|
|7|Missing labels|Resource has no labels (equivalent to "missing mandatory tags")|High|
|8|Significant cost increase|Service-level or resource-level cost increase > 30% month-over-month|High|
|9|Right-sizing opportunity|Server could use a smaller server_type based on CPU/network metrics|Medium|
|10|ARM migration opportunity|x86 server with low CPU could move to cheaper CAX (ARM) type|Low — requires workload compatibility assessment|

### 8.2 Recommendations NOT possible

Due to Hetzner's platform characteristics, these categories from the Atomity model do not apply:

- **Commitment-purchase opportunity:** Hetzner has no Reserved Instances or Savings Plans.
- **Commitment underutilisation:** Same reason.
- **Low-utilisation RDS:** Hetzner has no managed database service.
- **Provider-native recommendations:** Hetzner has no cost optimisation API.

### 8.3 Savings calculation

For right-sizing: `savings = (current_monthly_cost - proposed_monthly_cost)`. Both values come from `server_types` pricing for the current and proposed types in the same location.

For stopped servers: `savings = server_monthly_cost` (deleting the server or snapshotting and deleting).

For unattached resources: `savings = resource_monthly_cost`.

All savings figures must include the caveat that Hetzner has no partial-month proration for mid-month changes — the monthly cap applies from creation to deletion within each billing month.

---

## 9. Reconciliation

### 9.1 Computed vs actual reconciliation

When the customer uploads a Hetzner consumption CSV, Atomity must run reconciliation:

1. Match each CSV line item to an Atomity-computed cost line by resource ID and billing period.
2. Calculate the absolute difference and percentage difference.
3. Flag any pair with > 1% variance.
4. Document known reconciliation gap categories:
    - **Rounding differences:** Hetzner rounds hours up; Atomity's observation interval may introduce sub-hour rounding differences.
    - **Deleted resources:** If a resource was created and deleted between two inventory syncs (within 15 minutes), Atomity may miss it entirely. The CSV will show the charge; Atomity's computed cost will not.
    - **Pricing changes:** If Hetzner changes pricing mid-month (rare, announced), the cached pricing may not reflect the exact switchover.
    - **Traffic overage:** Traffic pooling rules are complex; computed overage may differ from Hetzner's calculation.
    - **Snapshots:** Snapshot sizes may change between observations if a user deletes and recreates snapshots.
    - **VAT/discounts:** The CSV may include VAT or volume discounts that the API pricing does not reflect.

### 9.2 Reconciliation target

For computed cost (Mode A), the target is:

**Absolute normalised Billed Cost difference ≤ 5% of the source total** per billing period.

This is relaxed from the 1% target for AWS/T Cloud Public because Atomity is reconstructing billing from inventory rather than ingesting actual billing data. If the customer provides the consumption CSV (Mode B), the standard ≤ 1% target applies against that CSV as the authoritative source.

---

## 10. Connection Wizard

### 10.1 Steps

**Step 1 — Provider selection:** User selects "Hetzner Cloud" from the Connections page.

**Step 2 — Project identification:** User enters a connection name (descriptive, e.g., "Production EU") and optionally a Hetzner project identifier (for their own reference; Atomity cannot read the project name from the API).

**Step 3 — Token entry:**

- Display the step-by-step token generation instructions (Section 2.3).
- Emphasise: select "Read" permission, not "Read & Write."
- Input field for the API token.
- Show a transparency notice about what Atomity will and will not access.

**Step 4 — Validation:**

- Call `GET /v1/servers?per_page=1` to verify token validity.
- Call `GET /v1/pricing` to verify pricing data accessibility.
- Call `GET /v1/locations` to confirm reachability.
- Display: token status (valid/invalid), permission level (if detectable from 403 errors on write attempts — though we never attempt writes), resource count detected.

**Step 5 — Confirmation:**

- Show a summary: project name (user-provided), location of resources detected, number of servers/volumes/IPs/LBs found, estimated monthly cost based on current inventory.
- User confirms to activate the connection.

### 10.2 Connection states

The standard Atomity connection states apply: DRAFT → VALIDATING → CONNECTED → SYNCING → HEALTHY → DEGRADED → FAILED → DISCONNECTED.

**DEGRADED** triggers: rate-limit exhaustion (remaining < 10%), partial sync failure (some endpoints failing but others succeeding), stale data (> 2 hours since last successful full sync).

**FAILED** triggers: token revoked (401), all endpoints returning errors, 3 consecutive full-sync failures.

---

## 11. Data Model Extensions

### 11.1 Hetzner-specific entities

**HetznerResourceSnapshot** (stored per sync, used for cost computation):

|Field|Type|Purpose|
|---|---|---|
|id|UUID|Atomity-generated|
|organization_id|UUID|Tenant isolation|
|connection_id|UUID|Originating connection|
|hetzner_resource_id|Integer|Hetzner's numeric resource ID|
|resource_type|Enum|SERVER, VOLUME, PRIMARY_IP, FLOATING_IP, LOAD_BALANCER, SNAPSHOT|
|resource_name|String||
|server_type_name|String|e.g., cx22, cpx31, ccx13 (null for non-servers)|
|location|String|e.g., fsn1, hel1|
|status|String|running, off, etc.|
|size_gb|Integer|Volume size / snapshot size (null for non-storage)|
|labels|JSONB|Key-value pairs|
|hourly_price_net|Decimal|From pricing API at sync time|
|monthly_cap_net|Decimal|From pricing API at sync time|
|created_at_provider|Timestamp|Hetzner `created` field|
|first_seen|Timestamp|First Atomity observation|
|last_seen|Timestamp|Most recent Atomity observation|
|deleted_detected|Timestamp|When resource disappeared from API|
|snapshot_timestamp|Timestamp|When this observation was taken|
|raw_api_response|JSONB|Full API response for this resource|

**HetznerPricingSnapshot** (stored when pricing changes detected):

|Field|Type|Purpose|
|---|---|---|
|id|UUID||
|fetched_at|Timestamp||
|pricing_data|JSONB|Full `/v1/pricing` response|
|server_types_data|JSONB|Full `/v1/server_types` response|
|lb_types_data|JSONB|Full `/v1/load_balancer_types` response|
|checksum|String|SHA-256 of concatenated responses|

### 11.2 Cost line identity for Hetzner

Since there is no source file or source row, the cost_line_id generation is adapted:

```
cost_line_id = SHA-256(
  organization_id +
  "HETZNER" +
  connection_id +
  hetzner_resource_id +
  resource_type +
  charge_period_start (day) +
  charge_period_end (day)
)
```

This produces one cost line per resource per day, making ingestion idempotent — reprocessing the same day's data overwrites rather than duplicates.

For CSV-uploaded cost lines, the standard source_file_id + source_row_number identity applies.

---

## 12. Job Design

### 12.1 Job types (Hetzner-specific)

|Job|Trigger|Frequency|
|---|---|---|
|HETZNER_TOKEN_VALIDATION|Connection creation, manual retry|On demand|
|HETZNER_PRICING_REFRESH|Scheduled|Every 24 hours|
|HETZNER_RESOURCE_INVENTORY_SYNC|Scheduled|Every 15 minutes|
|HETZNER_METRICS_COLLECTION|Scheduled|Every 24 hours|
|HETZNER_COST_COMPUTATION|After RESOURCE_INVENTORY_SYNC|Every hour|
|HETZNER_TRAFFIC_COMPUTATION|After METRICS_COLLECTION|Every 24 hours|
|HETZNER_INVOICE_CSV_PARSE|Manual upload trigger|On demand|
|HETZNER_RECONCILIATION|After INVOICE_CSV_PARSE|On demand|

### 12.2 Retry and resilience

All jobs must follow the standard Atomity operational requirements:

- Automatic retries with exponential backoff.
- Dead-letter state after 3 consecutive failures.
- Resume from last successful checkpoint.
- Idempotent processing (re-running any job must not duplicate data).
- Rate-limit-aware: check `RateLimit-Remaining` before every API call; pause and reschedule if exhausted.

### 12.3 Rate-limit-aware request pipeline

Implement a Hetzner-specific HTTP client wrapper that:

1. Reads `RateLimit-Remaining` and `RateLimit-Reset` from every response.
2. Maintains a local token bucket mirroring Hetzner's allowance.
3. If `RateLimit-Remaining` < 100, switches to a slower request cadence (1 request per 2 seconds).
4. If `RateLimit-Remaining` == 0, pauses all requests until `RateLimit-Reset` + 5 seconds buffer.
5. Logs all rate-limit events and surfaces them in the connection status.

---

## 13. Forecasting and Anomaly Detection

### 13.1 Forecasting

Hetzner cost is more predictable than hyperscaler cost because pricing is fixed (no spot/on-demand variance, no commitment discounts, no complex tiering). Forecasting operates on the normalised cost lines produced by the cost computation engine, using Atomity's standard cross-provider forecasting models (seasonal naïve, weighted moving average, Holt-Winters).

Special considerations:

- Hetzner price changes are announced weeks in advance (e.g., the April 2026 price adjustment). If a price change is known, the forecast should incorporate it.
- Traffic overage is the most variable Hetzner cost component and should be forecast separately from compute/storage.

### 13.2 Anomaly detection

Standard Atomity anomaly detection works on normalised cost data. Hetzner-specific anomaly patterns:

- **Sudden server count increase:** Many new servers created in a short window (detectable from resource inventory).
- **Traffic spike:** Outbound traffic significantly above historical pattern (detectable from network metrics).
- **Snapshot proliferation:** Large number of snapshots created, accumulating storage cost.
- **Orphaned resource accumulation:** Growing count of unattached volumes or unassigned IPs.

---

## 14. Security Considerations

### 14.1 Data residency

All Hetzner connector data must be stored and processed in the EU (SEC-005), which aligns with Hetzner's own EU-primary infrastructure.

### 14.2 What Atomity must never do

- Never call any write endpoint (POST, PUT, DELETE) — the read-only token prevents this at the API level, but the connector code must also never construct such requests.
- Never store the API token in application logs, error messages, or frontend state.
- Never expose Hetzner resource details (server IPs, network configurations) outside the cost/inventory context.
- Never call endpoints for SSH keys (`/v1/ssh_keys`), certificates (`/v1/certificates`), firewalls (`/v1/firewalls`), images (except snapshots), ISOs, or networks — these contain security-sensitive information that Atomity does not need.

### 14.3 Token rotation recommendation

Since Hetzner tokens do not expire, Atomity should surface a recommendation to rotate the token periodically (e.g., every 90 days). The connection settings page should provide a "Rotate Token" action that accepts a new token, validates it, and replaces the stored secret without interrupting data continuity.

---

## 15. Limitations and Customer Communication

The following limitations must be clearly documented in the product and communicated during onboarding:

1. **Computed cost, not actual billing:** Hetzner does not provide a billing API. Cost figures are computed by Atomity based on resource inventory and published pricing. They may differ from the actual invoice by up to 5%. Uploading the consumption CSV improves accuracy.
    
2. **No memory utilisation metrics:** Hetzner's API provides CPU, disk I/O, and network metrics only. Memory utilisation requires an in-guest agent (not part of Atomity's scope). Right-sizing recommendations are therefore based on CPU alone, which may be insufficient for memory-bound workloads.
    
3. **Short-lived resources may be missed:** Resources created and deleted between inventory syncs (< 15 minutes) may not be captured. The consumption CSV captures these.
    
4. **No commitment discount analysis:** Hetzner has no Reserved Instances, Savings Plans, or equivalent. The commitment-related FOCUS fields will be empty.
    
5. **Project-level granularity only:** Each token covers one project. There is no account-level overview across multiple projects unless multiple connections are configured.
    
6. **Token permission is broader than necessary:** The read-only token grants access to all resource types in the project. Atomity only reads inventory, pricing, and metrics, but cannot technically restrict the token's scope further.
    

---

## 16. Acceptance Criteria

### AC-HET-001: Connection setup

An Admin can:

1. Follow the Hetzner Cloud connection wizard.
2. Enter a read-only API token.
3. See validation results (token valid, resources detected).
4. Activate the connection.
5. See the connection in HEALTHY state within 30 minutes.

### AC-HET-002: Resource discovery

After initial sync:

1. All servers, volumes, primary IPs, floating IPs, and load balancers in the project are visible.
2. Each resource shows its Hetzner labels, location, type, and status.
3. Resource count matches what the customer sees in the Hetzner Console.

### AC-HET-003: Cost computation

For a billing period with known resources:

1. Computed cost for continuously-running servers matches monthly cap price (net).
2. Computed cost for a server deleted mid-month reflects hourly billing up to deletion.
3. Volume, IP, and LB costs are computed and visible.
4. Total computed cost is within 5% of the actual Hetzner invoice (when CSV uploaded for reconciliation).

### AC-HET-004: FOCUS normalisation

Every Hetzner cost line has:

1. Valid FOCUS fields (ServiceName, ServiceCategory, RegionId, ResourceId, ResourceType, BilledCost, EffectiveCost, Tags).
2. Provider-native attributes preserved in JSON.
3. `provider = "HETZNER"` and `focus_spec_version` set.
4. `cost_line_id` is deterministic and idempotent.

### AC-HET-005: Multi-provider visibility

Hetzner cost data appears alongside AWS and T Cloud Public data in:

1. Organisation overview KPI cards.
2. Cost Explorer (groupable by provider, service, region, resource, tags).
3. Spend trend charts.
4. Top movers analysis.

### AC-HET-006: Metrics and recommendations

1. CPU utilisation graphs are visible for Hetzner servers.
2. At least three recommendation categories produce valid results from real or test data.
3. Each recommendation includes evidence, savings calculation, confidence, and risk.

### AC-HET-007: Invoice CSV upload

An Admin can:

1. Upload a Hetzner consumption CSV.
2. See file validation results (period, rows, fields).
3. Confirm import.
4. See reconciliation report comparing computed vs actual cost.

### AC-HET-008: Rate-limit resilience

1. Under normal operation, the connector never exhausts the rate limit.
2. If rate limit is approached (< 10% remaining), the connector degrades gracefully and resumes automatically after reset.
3. Rate-limit events are visible in the connection sync history.

### AC-HET-009: Tenant isolation

Standard Atomity tenant isolation tests pass for Hetzner data (no cross-organisation leakage via REST APIs, export endpoints, or analytical queries).

### AC-HET-010: Idempotency

Rerunning the cost computation job for the same period does not create duplicate cost lines or change totals.

---

## 17. Build Order

1. **Hetzner HTTP client** — rate-limit-aware wrapper, read-only enforcement, token validation.
2. **Connection wizard** — token input, validation, project setup.
3. **Resource inventory sync** — all resource endpoints, pagination, state tracking.
4. **Pricing engine** — fetch and cache pricing, compute cost per resource per day.
5. **FOCUS normalisation** — map all fields, generate cost lines, idempotent upsert.
6. **Cost Explorer integration** — Hetzner data visible alongside other providers.
7. **Metrics collection** — CPU, disk, network time-series ingestion.
8. **Traffic overage computation** — monthly outbound aggregation vs included allowance.
9. **Recommendations** — rule engine for underutilised, unattached, stopped, missing labels.
10. **Invoice CSV upload** — parser, validation UI, import, reconciliation.
11. **Anomaly detection integration** — Hetzner cost data feeds standard detection pipeline.
12. **Hardening** — retry, recovery, rate-limit edge cases, performance testing, tenant isolation testing.