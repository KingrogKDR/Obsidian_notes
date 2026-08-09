## Context

Credential resolution is split across two paths:

**Onboarding (validate):** Frontend → Gateway → IAM. IAM loads the system account from vault, builds the credential, calls the AWS mapper directly, persists identity fields, and returns export discovery info to the frontend. Cloud-connector is not involved.

**Ongoing operations (sync/fetch/recommendations):** Cloud-connector resolves credentials from `connectionID` via its credential builder (IAM for connection metadata + vault for system account), then calls STS AssumeRole for temporary scoped access.

Both paths read the same Atomity system AWS account from vault.
## Vault team

### Seed Atomity's system AWS account

This is a single identity: Atomity's AWS account used for STS AssumeRole into customer accounts. All four fields belong together and rotate as a unit.

|Property|Value|
|---|---|
|Tenant ID|`system` (Atomity-internal namespace)|
|Credential ID|`system:aws` (confirm final ID with cloud-connector)|
|Purpose|`sts_assume_role`|

**Required fields:**

| Key                 | Value                                    |
| ------------------- | ---------------------------------------- |
| `access_key_id`     | Atomity IAM user access key              |
| `secret_access_key` | Atomity IAM user secret key              |
| `region`            | Default STS region (e.g. `eu-central-1`) |
The three are consumed at runtime by both IAM (validate) and cloud-connector (sync).

atomity: (in iam application.yml)
  aws:
    account-id: "123456789012"   # Atomity's own AWS account

## Testing checklist

- [ ] Vault: `system:aws` credential seeded and retrievable
- [ ] IAM: `GetConnection` returns `role_arn` + `external_id` for an AWS connection
- [ ] IAM: `POST /v1/connections/{id}/validate` calls mapper, persists identity fields only, returns export info
- [ ] IAM: validate with wrong roleARN fails cleanly, nothing persisted
- [ ] IAM: revalidate (empty roleARN) uses stored `conn.RoleARN`
- [ ] Cloud-connector: sync with role_arn → logs show "resolving AWS via STS system account"
- [ ] Cloud-connector: sync without role_arn → logs show fallback path
- [ ] Cloud-connector: cost lines land in ClickHouse after STS-based sync
- [ ] Cloud-connector: `GetCallerIdentity` returns customer account ID, not Atomity's
- [ ] Frontend: validate response shows account ID, export name, bucket, object count
- [ ] End-to-end: create connection → validate → sync → data visible in dashboard