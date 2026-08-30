Flow:

![[gcs_bucket.excalidraw.png | Ways for ingestion of cost data from GCP]]

Finalized flow anchored to FOCUS export specificially:

```
Customer: Enable FOCUS export (Console)
    ↓
Google: Creates immutable dataset  →  gcp_billing_immutable_<ID>_eu
        Creates FOCUS table        →  gcp_billing_export_focus_<ID>
        (no storage charges, 2-year TTL)
    ↓
Customer: Creates GCS bucket + scheduled EXPORT DATA query (Atomity provides templates)
    ↓
Daily:  BigQuery FOCUS table  →  EXPORT DATA  →  Parquet in gs://bucket/prefix/@run_date/*.parquet
    ↓
Atomity: GCSExportSource reads Parquet  →  same pipeline as AWS S3

```

File structure:

```
auth.go        — build a GCS client
gcs_source.go  — list + download Parquet from GCS (implements exporter.ExportSource)
mapper.go      — wire GCS source to the parser (same as AWS wires S3 source to its parser)
types.go       — constants, field mappings
onboarding.go  — customer setup templates
```

Required credentials:

- service_account_json 
```json
{
	"type": "service_account",
	"client": "ATOMITY GCP SA email",
}
```
- billing_project_id - customer's project identifier. Used for display purposes and logging.
- gcs_bucket
- gcs_prefix

## Important Context

- **`type` must be `"service_account"`** — GCP JSON keys can also have `type: "authorized_user"` (personal OAuth) or `type: "external_account"` (workload identity federation). Only `service_account` is valid for a system SA key from vault. If someone accidentally pastes their personal credential, this catches it with a clear message instead of a mysterious auth failure downstream.

## Atomity service account setup

**Step 1: Create the Atomity GCP project**

Go to console.google.com → click the project dropdown at the top → **New Project**

- Project name: `atomity-prod` (or `atomity-connector`)
- Organization: your org if you have one, otherwise "No organization"
- Click **Create**

**Step 2: Create the service account**

Stay in the `atomity-prod` project → hamburger menu → **IAM & Admin** → **Service Accounts** → **Create Service Account**

- Service account name: `connector`
- Service account ID: auto-fills to `connector` (this becomes the email: `connector@atomity-prod.iam.gserviceaccount.com`)
- Description: "Atomity cloud connector — reads customer billing exports from GCS"
- Click **Create and Continue**
- Skip the "Grant this service account access to project" step (click **Continue**) — we don't need any roles on Atomity's own project
- Skip "Grant users access" — click **Done**

**Step 3: Generate the JSON key**

In the Service Accounts list, click on `connector@atomity-prod.iam.gserviceaccount.com` → **Keys** tab → **Add Key** → **Create new key**

- Key type: **JSON**
- Click **Create**

The browser downloads a file like `atomity-prod-abc123def456.json`. This is the `service_account_json`. It looks like:

```json
{
  "type": "service_account",
  "project_id": "atomity-prod",
  "private_key_id": "abc123...",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIE...\n-----END PRIVATE KEY-----\n",
  "client_email": "connector@atomity-prod.iam.gserviceaccount.com",
  "client_id": "117234567890",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/connector%40atomity-prod.iam.gserviceaccount.com"
}
```

GCP generated all of this. The `"type": "service_account"` is set by GCP — that's the field our `auth.go` validates.

**Step 4: Store in vault, delete the file**

Store the entire JSON blob as the system credential in vault and delete the downloaded file. You can never download this key again — if lost, you revoke it and create a new one from the Keys tab.

**Step 5: Note the SA email**

`connector@atomity-prod.iam.gserviceaccount.com` — this is what the Atomity connection wizard displays to customers so they can grant `storage.objectViewer` on their bucket.

That's the entire Atomity-side setup. Done once, works for every customer. No roles needed on Atomity's own project — all permissions come from customers granting access to their own buckets.

Following is how the conformance gaps can be solved or mapped between GCP and AWS:

|Empty field|Can enrich?|Source|
|---|---|---|
|`InvoiceIssuerName`|✅ static|Always "Google Cloud"|
|`SubAccountType`|✅ static|Always "Project"|
|`ServiceCategory`|✅ derive|`ServiceName` / `x_ServiceId`|
|`ChargeFrequency`|✅ derive|`PricingCategory` + `ChargeCategory`|
|`CommitmentDiscount*`|✅ extract|`x_Credits` where Type is CUD/SUD|
|`ResourceType`|✅ derive|`ServiceName` patterns|
|`BillingAccountName`|❌|Not in any GCP field|
|`InvoiceID`|❌|Not in any GCP field|
|`ServiceSubcategory`|❌|Too unreliable to derive|
|`SkuMeter`|❌|Not in any GCP field|

**Before enrichment — grouping by ServiceCategory:**

```
AWS:      Compute: $12,340  |  Storage: $3,210  |  Database: $5,670
GCP:      (empty)   ← useless, breaks the dashboard
```

**After enrichment:**

```
AWS:      Compute: $12,340  |  Storage: $3,210  |  Database: $5,670
GCP:      Compute: $5,200   |  Storage: $2,100  |  Database: $1,600
Combined: Compute: $17,540  |  Storage: $5,310  |  Database: $7,270  ← cross-provider works
```

Six fields enriched, four left empty (genuinely unavailable):

| Field                 | Enrichment                    | Source                                                                |
| --------------------- | ----------------------------- | --------------------------------------------------------------------- |
| `InvoiceIssuerName`   | `"Google Cloud"`              | Static — GCP has one issuer                                           |
| `SubAccountType`      | `"Project"`                   | Static — GCP sub-accounts are always projects                         |
| `ServiceCategory`     | `enrichServiceCategory()`     | Maps `ServiceName`/`x_ServiceId` → Compute, Storage, Database, etc.   |
| `ChargeFrequency`     | `enrichChargeFrequency()`     | Committed → Recurring, Tax → One-Time, else → Usage-Based             |
| `ResourceType`        | `enrichResourceType()`        | Compute Engine → Virtual Machine, Cloud SQL → Database Instance, etc. |
| `CommitmentDiscount*` | `extractCommitmentDiscount()` | Finds CUD/SUD in `x_Credits[]`, extracts ID/Name/Type/Status          |

