Base path: `/api/v1/connections` Auth: JWT via gateway proxy (logged-in user managing their tenant)

---

## POST /api/v1/connections

Creates a connection. `connection_type` determines the activation flow and is independent of `provider` — every provider supports both types.

### Request

```json
{
  "name": "prod-aws-billing",
  "provider": "CLOUD_PROVIDER_AWS",
  "connection_type": "AUTOMATED",
  "region": "eu-west-1"
}
```

|Field|Required|Notes|
|---|---|---|
|`name`|yes||
|`provider`|yes|One of the `CloudProvider` enum names (`CLOUD_PROVIDER_AWS`, `CLOUD_PROVIDER_GCP`, `CLOUD_PROVIDER_NEBIUS`, `CLOUD_PROVIDER_AZURE`, or a push provider like `CLOUD_PROVIDER_OVH`)|
|`connection_type`|no|`AUTOMATED` or `API_KEY`. Defaults to `AUTOMATED` if omitted|
|`region`|no|Only meaningful when `connection_type` is `AUTOMATED`; ignored otherwise|

### Response — `connection_type: "AUTOMATED"`

Status starts `PENDING`. Response includes provider-specific onboarding hints (unchanged from before, just now gated behind `connection_type` rather than inferred from `provider`).

```json
{
  "connection_id": "b3f1...",
  "name": "prod-aws-billing",
  "provider": "CLOUD_PROVIDER_AWS",
  "connection_type": "AUTOMATED",
  "status": "PENDING",
  "region": "eu-west-1",
  "external_id": "b7e2...",
  "atomity_account_id": "123456789012"
}
```

GCP/Azure equivalents swap `external_id`/`atomity_account_id` for `atomity_sa_email` / `atomity_azure_client_id` respectively, same as before. Nebius has no extra hint fields.

### Response — `connection_type: "API_KEY"`

Status is `ACTIVE` immediately. No onboarding hints — next step is generating a key.

```json
{
  "connection_id": "c4a2...",
  "name": "ovh-billing-upload",
  "provider": "CLOUD_PROVIDER_OVH",
  "connection_type": "API_KEY",
  "status": "ACTIVE"
}
```

Note: an `AWS` connection created with `connection_type: "API_KEY"` looks identical in shape to the OVH example above — same `ACTIVE` status, no onboarding hints, `region` omitted since it wasn't supplied. `connection_type` is what drives the response shape, not `provider`.

### Error responses (400)

```json
{ "error": { "code": "MISSING_NAME", "message": "name is required" } }
{ "error": { "code": "INVALID_PROVIDER", "message": "provider must be one of the CloudProvider enum names, e.g. CLOUD_PROVIDER_AWS" } }
{ "error": { "code": "INVALID_CONNECTION_TYPE", "message": "connection_type must be AUTOMATED or API_KEY" } }
```

---

## GET /api/v1/connections

Lists all connections for the tenant. No request body.

### Response

```json
{
  "data": [
    {
      "connection_id": "b3f1...",
      "name": "prod-aws-billing",
      "provider": "CLOUD_PROVIDER_AWS",
      "connection_type": "AUTOMATED",
      "status": "ACTIVE",
      "region": "eu-west-1",
      "provider_account_id": "123456789012",
      "role_arn": "arn:aws:iam::123456789012:role/AtomityReadOnly",
      "use_organizations": false,
      "sync_enabled": true,
      "sync_frequency_hours": 24,
      "last_sync_triggered_at": "2026-09-14T08:00:00Z"
    },
    {
      "connection_id": "c4a2...",
      "name": "ovh-billing-upload",
      "provider": "CLOUD_PROVIDER_OVH",
      "connection_type": "API_KEY",
      "status": "ACTIVE",
      "sync_enabled": false
    }
  ]
}
```

Fields present per row depend on `connection_type` and `provider`, same conditional-inclusion pattern as before (`connectionToMap` only adds provider-specific keys when they're populated). `region` is included only if set.

---

## POST /api/v1/connections/{id}/validate

**AUTOMATED connections only.** This is the onboarding-completion step; unchanged in shape from before except it now rejects `API_KEY` connections explicitly instead of silently accepting whatever fields happen to be present.

### Request (AWS example — shape is provider-specific, same as before)

```json
{
  "role_arn": "arn:aws:iam::123456789012:role/AtomityReadOnly",
  "aws_org_id": "o-abc123",
  "use_organizations": "true"
}
```

### Response — success

```json
{
  "status": "ACTIVE",
  "provider_account_id": "123456789012",
  "export_name": "atomity-cur-export",
  "bucket": "atomity-billing-exports",
  "prefix": "cur/",
  "export_object_count": 42,
  "latest_export_file": "cur/2026/09/export-0001.parquet"
}
```

### Response — wrong connection type (400, new)

```json
{
  "error": {
    "code": "WRONG_CONNECTION_TYPE",
    "message": "API_KEY connections do not use the validate flow; create an API key via POST /{id}/api-keys instead"
  }
}
```

### Response — validation failed (400)

```json
{ "error": { "code": "VALIDATION_FAILED", "message": "unable to assume role: access denied" } }
```

---

## PATCH /api/v1/connections/{id}/sub-tenant

Unchanged. Works identically regardless of `connection_type`.

### Request

```json
{ "sub_tenant_id": "9f2e1a4c-...-...-...-..." }
```

Omit or send `""` to unassign.

### Response

```json
{ "connection_id": "b3f1...", "sub_tenant_id": "9f2e1a4c-...-...-...-..." }
```

---

## PATCH /api/v1/connections/{id}/sync-schedule

**AUTOMATED connections only.** `API_KEY` connections receive data via customer push, not a scheduled pull, so there's nothing to schedule — same guard pattern as `/validate`.

### Request

```json
{ "enabled": true, "frequency_hours": 24 }
```

### Response — success

Returns the full connection object (same shape as one row from `GET /connections`), now including `connection_type` and `region`.

### Response — wrong connection type (400, new)

```json
{
  "error": {
    "code": "WRONG_CONNECTION_TYPE",
    "message": "API_KEY connections do not support sync scheduling; data is received via customer-initiated push, not a scheduled pull"
  }
}
```

---

## DELETE /api/v1/connections/{id}

Unchanged. Revokes regardless of type.

### Response

```json
{ "connection_id": "b3f1...", "status": "REVOKED" }
```

---

## API Keys — `/api/v1/connections/{connectionId}/api-keys`

Unchanged by this work — these always operated per-connection regardless of provider, and now apply just as naturally to an `AUTOMATED` connection as an `API_KEY` one (though in practice you'd only generate a key for an `API_KEY` connection; nothing stops you from also issuing one against an `AUTOMATED` connection today).

### POST — create key

**Request**

```json
{ "name": "prod ingestion pipeline" }
```

**Response**

```json
{
  "api_key_id": "d5b3...",
  "connection_id": "c4a2...",
  "prefix": "atk_live_",
  "name": "prod ingestion pipeline",
  "raw_key": "atk_live_9f8e7d6c5b4a3928f1e0...",
  "message": "Store this key now — it will not be shown again"
}
```

### GET — list keys

**Response**

```json
{
  "data": [
    {
      "api_key_id": "d5b3...",
      "prefix": "atk_live_",
      "name": "prod ingestion pipeline",
      "status": "ACTIVE",
      "created_at": "2026-09-01T12:00:00Z",
      "last_used_at": "2026-09-15T09:12:03Z"
    }
  ]
}
```

### DELETE — revoke key

**Response**

```json
{ "api_key_id": "d5b3...", "status": "REVOKED" }
```

---

## Summary of what changed vs. before this work

|Endpoint|Change|
|---|---|
|`POST /connections`|New `connection_type` (defaults `AUTOMATED`) and `region` input fields; response shape now branches on `connection_type`, not `provider`|
|`GET /connections`|Each row now includes `connection_type` and (if set) `region`|
|`POST /connections/{id}/validate`|New 400 `WRONG_CONNECTION_TYPE` guard for `API_KEY` connections|
|`PATCH /connections/{id}/sync-schedule`|New 400 `WRONG_CONNECTION_TYPE` guard for `API_KEY` connections; response includes the two new fields|
|everything else|Unchanged|