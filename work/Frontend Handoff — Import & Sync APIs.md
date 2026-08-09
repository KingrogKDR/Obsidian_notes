All endpoints: JWT (Bearer token). Base URL <stubbed, currently I am not aware>, paths under `/api/v1`. 

Two upload paths per provider tile:

- **Manual** → `POST /imports` (this doc, section A)
- **Automatic** → `POST /connections/{connectionID}/sync` (section B)

---

## A. Manual upload

### A1. Upload + preview — `POST /api/v1/imports`

`multipart/form-data`, **field order matters**: send `provider` BEFORE `file`.

| field      | type | notes                                               |
| ---------- | ---- | --------------------------------------------------- |
| `provider` | text | must come first. Exact enum string (see list below) |
| `file`     | file | exactly one. CSV / Parquet (supported formats only) |

Runs the preview synchronously and returns the quality report inline — no polling needed for the report.

**Provider values** (exact strings): `CLOUD_PROVIDER_AWS`, `CLOUD_PROVIDER_T_CLOUD_PUBLIC` (others as they’re added — confirm against the enum before hardcoding.)

**200 OK**

```json
{
  "data": {
    "import_id": "7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c",
    "status": "AWAITING_CONFIRMATION",
    "quality_report": {
      "import_id": "7f3a9c2e-...",
      "organization_id": "0abc...",
      "rejected_rows": 3,
      "duplicate_rows": 0,
      "unmapped_field_count": 2,
      "processing_ms": 412,
      "overall_status": "PASS",
      "reconciliation": [
        {
          "currency": "EUR",
          "source_billed": "10432.55",
          "normalized_billed": "10432.55",
          "rejected_cost": "0",
          "difference": "0",
          "residual_pct": "0",
          "status": "PASS"
        }
      ]
    }
  },
  "errors": null
}
```

- `overall_status` / per-currency `status`: `PASS` (within 1%), `WARN` (>1% gap — flag it, user can still confirm), `EMPTY` (no cost).
- **All money fields are strings** (decimals). Don’t parseFloat for math; render as-is or use a decimal lib.
- Rare: `quality_report: null` with `200` → report didn’t persist; fall back to A3.

**Errors**

|status|error_code|meaning|
|---|---|---|
|400|`INVALID_ARGUMENT`|missing/invalid provider, missing file, multiple files, `provider` sent after `file`|
|422|`PREVIEW_FAILED`|file couldn’t be parsed — tell user to check format/contents|
|502|`UPLOAD_FAILED`|staging problem (our side) — retry|

### A2a. Confirm — `POST /api/v1/imports/{importID}/confirm`

Commits the import (starts async processing). Then poll A4.

**202 Accepted**

```json
{ "data": { "import_id": "...", "status": "QUEUED" }, "errors": null }
```

|status|error_code|meaning|
|---|---|---|
|404|`NOT_FOUND`|import missing or not yours|
|409|`NOT_CONFIRMABLE`|not in AWAITING_CONFIRMATION. `message` = current status|

### A2b. Cancel — `POST /api/v1/imports/{importID}/cancel`

Deletes the staged file, marks CANCELLED.

**200 OK**

```json
{ "data": { "import_id": "...", "status": "CANCELLED" }, "errors": null }
```

|status|error_code|meaning|
|---|---|---|
|404|`NOT_FOUND`|import missing or not yours|
|409|`NOT_CANCELLABLE`|not in AWAITING_CONFIRMATION. `message` = current status|

### A3. Re-fetch report — `GET /api/v1/imports/{importID}/quality-report`

Same quality_report object as A1. Use on the `null` fallback or to show it again later (stays available after processing).

|status|error_code|meaning|
|---|---|---|
|404|`NOT_FOUND`|import missing or not yours|
|404|`NOT_READY`|report not available yet|

### A4. Poll status — `GET /api/v1/imports/{importID}`

**200 OK**

```json
{
  "data": {
    "import_id": "...",
    "connection_id": null,
    "provider": "CLOUD_PROVIDER_AWS",
    "status": "COMPLETED",
    "created_at": "2026-07-29T09:00:00Z",
    "updated_at": "2026-07-29T09:00:04Z"
  },
  "errors": null
}
```

`connection_id` is `null` for manual imports (no connection involved).

**Status values:** `PENDING` / `UPLOADING` / `PREVIEWING` (transient) · `AWAITING_CONFIRMATION` (show review screen) · `QUEUED` / `RUNNING` (spinner) · `COMPLETED` · `CANCELLED` · `FAILED`.

|status|error_code|meaning|
|---|---|---|
|404|`NOT_FOUND`|import missing or not yours|

---

## B. Automatic sync

These require NATS up on our side; if unavailable the routes are disabled (404). All JWT, all scoped to a `connectionID` (the connection the user set up for that provider).

### B1. Trigger sync — `POST /api/v1/connections/{connectionID}/sync`

Body is **optional** — send `{}` and we default the period (90 days on first sync, 60 on refresh).

Optional body:

```json
{
  "op_type": "CLOUD_OP_TYPE_FETCH_COST_DATA",
  "billing_period_start": "2026-05-01",
  "billing_period_end": "2026-07-30"
}
```

(dates `YYYY-MM-DD`; if you send one you must send both.)

**202 Accepted**

```json
{
  "data": {
    "sync_id": "018f...",
    "connection_id": "7f3a...",
    "status": "QUEUED",
    "op_type": "CLOUD_OP_TYPE_COST_FETCH",
    "billing_period_start": "2026-05-01",
    "billing_period_end": "2026-07-30",
    "accepted_rows": 0,
    "requested_at": "2026-07-30T10:00:00Z"
  },
  "errors": null
}
```

Then poll B3 until `HEALTHY` (done) or `FAILED`.

|status|error_code / body|meaning|
|---|---|---|
|404|connection not found|connection missing or not yours|
|409|connection is not active|connection status ≠ ACTIVE|
|409|(returns the in-flight sync object)|a sync is already running for this connection — body is the existing run|
|400|invalid period|bad dates|

> Note: sync errors use `{ "error": "message" }`, not the data/errors envelope.

### B2. Sync history — `GET /api/v1/connections/{connectionID}/sync-history`

**200 OK**

```json
{ "syncs": [ { /* same shape as B1 data */ }, ... ] }
```

Newest first, up to 50.

### B3. Single sync — `GET /api/v1/connections/{connectionID}/sync/{syncID}`

**200 OK** → one sync object (same shape as B1 `data`).

Sync statuses: `QUEUED` → `RUNNING` → `HEALTHY` (success) / `FAILED`.

|status|meaning|
|---|---|
|404|sync not found (or not yours / wrong connection)|

---

## Notes

- **Provider tag** on manual imports comes from the `provider` form field you send (MVP). Send the value matching the tile the user clicked.
- **connection_id is null** on manual imports and populated on sync — handle both.
- Two different error shapes: import endpoints use `{data, errors}`; sync endpoints use `{error}`. 
- Confirm the provider enum strings against the generated contract before hardcoding.