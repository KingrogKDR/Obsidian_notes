## 1. Problem

Hetzner has no billing API and no FOCUS export. The only source of finalized cost data is a monthly PDF invoice, arriving weeks after the billing period it covers. A product built around near-real-time cost visibility can't wait a month for every number — so this connector runs two parallel systems that reconcile with each other: **live estimation** from Hetzner's Cloud API for the current month, superseded automatically by **real invoice data** once it arrives, however it arrives.

## 2. System context

One Atomity `Connection` represents one real Hetzner account. An account can contain several projects; Hetzner scopes API tokens per project with no cross-project API, so the connector holds one Vault-backed token per project under that single connection. This is the one structural decision everything else follows from — it's why credential resolution, resource tracking, and cost-line attribution are all project-aware even though the customer-facing concept is "one connection."

Four independent services participate:

- **Gateway** — the only internet-facing entry point. Fronts the Stalwart webhook (public, signature-authenticated, not session-authenticated) and the customer's own API-key-authenticated push endpoint. Also runs the background sweeper that reads invoice emails out of the mailbox (§3.1).
- **IAM** — owns connection identity: the account's `customer_number`, each project's token reference, the invoice-forwarding token, and bucket location metadata. Never touches secret values directly.
- **Vault** — owns secret values: API tokens, bucket credentials, and the system-level Stalwart credentials (webhook signing key and the JMAP service account). Resolves by an opaque credential ID IAM, gateway or cloud-connector supplies; has no concept of "Hetzner" at all.
- **cloud-connector** — does the actual work: ingests CSVs regardless of which channel delivered them, computes live estimates, writes cost lines, and retracts estimates once real data supersedes them.

A fifth participant, **Stalwart** (Atomity's own mail server at `stalwart.atomity.de`), is infrastructure rather than a service in this codebase: it receives the forwarded invoices and exposes them over JMAP.

## 3. The four data paths

### 3.1 Invoice email (Channel 1)

The customer forwards Hetzner's monthly invoice to a per-connection address of the form `import+<token>@in.atomity.de`. The token is opaque and was minted by IAM when the connection was created. Stalwart receives all such mail into one shared mailbox and fires a signed webhook to gateway for each arrival.

The webhook is only a wake-up. It carries Stalwart-internal ids and the base address without the token — nothing that identifies the customer or that JMAP accepts as a mail id. So gateway verifies the signature, acknowledges immediately, and wakes a background sweeper. The sweeper reads recent mail from the mailbox over JMAP (read-only), finds the routing token in the message's recipient and forwarding headers, resolves it through IAM to a tenant and connection, downloads the PDF, and forwards it to cloud-connector.

The mailbox is the durable queue. A short Redis claim per message keeps gateway replicas from processing the same email twice. A transient failure (cloud-connector or IAM unavailable, a JMAP error) releases the claim, and the next sweep — triggered by the next webhook or by a five-minute timer — retries it. A permanent outcome (unknown token, no PDF, content cloud-connector rejects) is marked done and not retried.

Cloud-connector then extracts a download link from the PDF — notably **not** from visible text (the link exists only as a clickable annotation on the real invoice, confirmed by inspection, not assumed) — and fetches the CSV it points to. From there it's an ordinary upload.

### 3.2 Object storage (Channel 2)

For customers who already script their own delivery into a Hetzner Object Storage bucket, cloud-connector pulls from it on a schedule. No gateway involvement — this is cloud-connector talking directly to Hetzner's S3-compatible storage using account-level credentials.

### 3.3 API push (Channel 3)

The same generic, cross-provider API-key upload path every provider uses. Nothing Hetzner-specific about it at all.

### 3.4 Live estimation

On each scheduled sync, cloud-connector lists every project token for the connection, queries each project's live resource inventory and Hetzner's pricing catalog, and computes a cost estimate per resource — hourly rates capped at the published monthly rate, matching how Hetzner itself bills.

**All three ingestion channels converge on one pipeline** inside cloud-connector regardless of origin. This is deliberate: it means retraction (§4) never needs to know which channel produced a CSV, only that one arrived.

## 4. Reconciling estimate and invoice

Every estimate cost line is tagged as an estimate. When a real CSV completes processing — from any channel — cloud-connector regenerates the same deterministic IDs the estimate would have used for that billing period and writes tombstones, then records the period as finalized. Live estimation checks that record before writing anything, so a period that's already been reconciled can never have an estimate resurrected on top of it, even by a mistaken or duplicate sync.

The billing period a retraction targets comes from the CSV's own data, not from when it happened to arrive — so a CSV uploaded late still retracts the correct month, not whichever month is currently open.

## 5. Key decisions and why

- **Atomity's own Stalwart server, not a third-party mail API.** Invoices, and the mail that carries them, stay on infrastructure Atomity runs: no new subprocessor, and no vendor data-residency question to answer for a pipeline that handles a customer's actual invoice. This replaced an earlier design built on Mailgun's EU region.
- **The webhook is a wake-up; the mailbox is the queue.** Stalwart's webhook lacks the routing token, and its delivery retries are bounded (it retries for a limited window — five minutes by default — then drops the event). A design that depended on the webhook arriving intact could lose an invoice silently. Reading the mailbox directly means a lost, late or failed webhook costs at most one sweep interval, and the webhook handler stays fast and independent of how long processing takes.
- **Standard JMAP, read-only, URLs from session discovery.** Gateway uses only RFC 8620/8621 behavior, never writes to the mailbox, and takes its API and download URLs from the JMAP session resource rather than assuming Stalwart's paths. The service account needs read access only.
- **The account, not a separate grouping entity, is the connection.** An earlier design introduced a dedicated "account group" entity to link several project-scoped connections together. Simplified once it became clear the connection itself could just be the account, with projects as child records underneath it — fewer moving parts, same capability.
- **Retraction keyed to CSV completion, not channel.** Avoids needing to special-case "we don't know when channel 3's upload will happen" — it was never a scheduling problem, only ever a reactive one.
- **PDF link extraction walks annotations, not text.** Verified against a real invoice before writing any code — assuming visible-text extraction would have shipped broken.
- **Per-project credential IDs, not per-connection.** Each project's token needs its own Vault-resolvable ID rather than sharing the connection's, since a connection's primary ID slot can only back one credential at a time (an existing convention this connector is the first to actually need two of).

## 6. Deliberately out of scope (for now)

- **Traffic overage billing** — blocked on metrics collection Hetzner doesn't expose the way this connector needs yet.
- **Cross-referencing the invoice CSV's per-project line items against live-estimate project attribution** — the data model carries what's needed (`project_name` on every cost line either way), but nothing reconciles them against each other yet.

## 7. Assumptions still to verify

- **Token visibility under real auto-forwarding.** Test mail sent directly to `import+<token>@` carried the token in `To:`. Customers using auto-forward rules may leave `To:` as their own address and carry the forward target only in envelope or forwarding headers. The sweeper scans `To`, `Cc` and the usual envelope and forwarding headers, but which of those Stalwart records, and whether Gmail and Outlook add them, is unconfirmed. If the token appears in none, address-based routing cannot identify the customer and Channel 1 needs a different mechanism.
- **End-to-end against the real JMAP server.** The JMAP client follows the RFCs and is tested against a conformant fake, not against Stalwart itself.
- **Token case.** Tokens are case-sensitive; an intermediary that lowercases the address would break the IAM lookup. The proposed fix is lowercase hex tokens, an IAM-side change.

# Considerations to be revisited later:

### Silent failure modes

1. **A rejected invoice is final.** If cloud-connector returns 4xx (for example, Hetzner changes the invoice layout and link extraction fails), the email is marked done and logged once. Alert on those. A manual re-drive means deleting the `stalwart-email:<id>` Redis key, and it only works inside the lookback window.
2. **A flooded shared mailbox.** Each sweep takes the newest 100 messages in the window. Spam or abuse sent to `import@` could push real invoices out of that window. Paginate, or filter, before this matters.
3. **No alerting on the sweeper.** JMAP down, auth failing, or repeated retries only appear in logs. A persistently failing email also logs an error every sweep for 48 hours. Add a metric for "emails seen but not done" and an attempt counter.
4. **Redis outage.** The claim store fails open, so multiple replicas can forward the same invoice. Cost lines dedupe, but duplicate `Import` rows appear in the import history.

### Security and lifecycle

5. **The inbound token is a bearer secret with no rotation.** Anyone with it can submit a PDF to that connection. The blast radius is limited (the PDF must contain a real Hetzner statement link), but there is no endpoint to regenerate a leaked token.
6. **Secrets are read once, at boot.** Rotating the Stalwart signing key or JMAP password means updating Vault and restarting gateway. Multiple environments sharing one Stalwart need separate webhooks and keys.
7. **Mailbox retention.** `import@` accumulates customer invoices indefinitely. Decide on a retention and cleanup policy, and whether that mailbox counts as a raw source for your 15-month raw-data requirement.
8. **Stalwart is a new critical dependency.** Plan its backup and availability. Sending servers retry for a while, but a long outage loses mail.

### Accuracy and data

9.  **Estimate accuracy.** Grandfathered prices (the API returns list price only) and traffic overage (not estimated at all) make estimates differ from the invoice. Pricing is fetched with the first project's token, so a revoked first token would stop estimation for the whole connection.
10. **Resource-ID uniqueness.** Multi-project deletion tracking assumes Hetzner resource IDs are unique across the whole account. That's still unverified.
11. **`pricing_key` history.** The earlier `UpsertHetznerResourceSeen` bug may have left it empty in production. Check the data, because deleted-resource tail costs were being silently skipped for those rows.
12. **Vault debt.** The `purpose` argument on `DecryptCredential` is still ignored. Consumers of the `credential.stored` event may be affected by the provider-string change (bare values instead of `CLOUD_PROVIDER_` prefixes).

### Housekeeping

13. **Reconciliation.** Nothing yet matches the invoice's per-project lines to the live-estimate project attribution. The data model supports it.