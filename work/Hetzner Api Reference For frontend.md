Base envelope, every endpoint: success → `{"data": {...}}`, failure → `{"errors": [{"error_code": "...", "message": "..."}]}`. Never both, never null keys present — branch on which key exists.

Call order: 1 fires at step 0. 2 fires per "add project" click at step 4 (fire-and-forget, any order). 3 and 4 fire together at the final "Connect" step, not before — 3 needs data collected across steps 0 and 2, held in frontend state until submission.

## 1. Create the connection — step 0, on Next

`POST /api/v1/connections`

```json
{ "name": "Production Hetzner", "provider": "CLOUD_PROVIDER_HETZNER", "connection_type": "CONNECTION_TYPE_AUTOMATED" }
```

`.data` — _(fields marked ⚠ are inferred from the entity, not confirmed against the controller)_

```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "name": "Production Hetzner",
  "provider": "CLOUD_PROVIDER_HETZNER",
  "connection_type": "CONNECTION_TYPE_AUTOMATED",
  "status": "PENDING",
  "hetzner_inbound_token": "aB3xQ9zK2mN8pR5t",
  "hetzner_inbound_email": "import+aB3xQ9zK2mN8pR5t@in.atomity.de"
}
```

⚠ = `id`, `tenant_id`, `name`, `provider`, `connection_type`, `status`. `hetzner_inbound_token`/`hetzner_inbound_email` are confirmed. Use `hetzner_inbound_email` verbatim for step 1's display — don't reconstruct it client-side.

## 2. Add a project token — step 4, per "Add" click

`POST /api/v1/connections/{id}/hetzner-project-tokens`

```json
{ "project_name": "backend-staging", "api_token": "..." }
```

`.data`

```json
{ "id": "uuid", "project_name": "backend-staging" }
```

## 3. Validate — step 5, on Connect

`POST /api/v1/connections/{id}/validate`

```json
{
  "hetzner_customer_number": "K0279164926",
  "hetzner_project_name": "backend-prod",
  "hetzner_api_token": "...",
  "hetzner_bucket_endpoint": "https://fsn1.your-objectstorage.com",
  "hetzner_bucket_name": "...",
  "hetzner_bucket_prefix": "...",
  "hetzner_bucket_access_key_id": "...",
  "hetzner_bucket_secret_access_key": "..."
}
```

`hetzner_customer_number` / `hetzner_project_name` / `hetzner_api_token` required. The five `hetzner_bucket_*` fields: all-or-nothing, omit the group entirely if step 2 was skipped. `hetzner_customer_number` must match `^K\d{10}$` — validate client-side before submit, same regex the backend enforces. Response body: ⚠ not confirmed.

## 4. Create an API key — step 5, only if step 3's toggle was checked

`POST /api/v1/connections/{id}/api-keys`

```json
{ "name": "prod ingestion pipeline" }
```

`.data`

```json
{
  "api_key_id": "uuid",
  "connection_id": "uuid",
  "prefix": "...",
  "name": "prod ingestion pipeline",
  "raw_key": "...",
  "message": "Store this key now — it will not be shown again"
}
```

`raw_key` is shown exactly once — this response is the only place it ever appears. Display it with a copy button and a persistent warning; it cannot be retrieved again.

## Not part of the wizard but useful

- `GET /api/v1/connections/{id}/hetzner-project-tokens` → `.data`: `[{id, project_name}]` 
- `DELETE /api/v1/connections/{id}/hetzner-project-tokens/{tokenId}` → `.data`: `{id}` 
- `GET /api/v1/connections/{id}/api-keys` → `.data`: `[{api_key_id, prefix, name, status, created_at, last_used_at}]` (no `raw_key` returned again after creation) 
- `DELETE /api/v1/connections/{id}/api-keys/{keyId}` → `.data`: `{api_key_id, status: "REVOKED"}`