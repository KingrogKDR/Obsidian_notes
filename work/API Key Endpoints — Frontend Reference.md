Flow:

![[api_key_flow.excalidraw.png | How the api key is generated, validated and stored]]

All management endpoints (create, list, revoke) require a JWT `Authorization: Bearer` header — the logged-in user in the UI.

The upload endpoint uses `X-Api-Key` — the customer's backend script, not the UI's concern. But you can test it using curl (currently only single file upload works).

Base URL: GATEWAY_BASE_URL

---

## 1. Create API Key

**POST** `/api/v1/connections/{connectionId}/api-keys`

**Auth:** JWT Bearer (logged-in user)

**Request body** (optional):

```json
{
  "name": "prod ingestion pipeline"
}
```

**Response** `200 OK`:

```json
{
  "data": {
    "api_key_id": "a1b2c3d4-5678-9012-abcd-ef1234567890",
    "connection_id": "7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c",
    "prefix": "atk_XXXXXXXX",
    "name": "prod ingestion pipeline",
    "raw_key": "atk_dGhpcyBpcyBhIHRlc3Qga2V5IGZvciBkZW1v",
    "message": "Store this key now — it will not be shown again"
  },
  "errors": null
}
```

> **Important:** `raw_key` is returned exactly once. The UI must display it prominently with a copy-to-clipboard button and a warning. After this response, only the `prefix` is retrievable.

**Error responses:**

- `404` — connection not found or belongs to another tenant
- `400` — invalid request

**curl:**

```bash
curl -X POST \
  https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c/api-keys \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"name": "prod ingestion pipeline"}'
```

---

## 2. List API Keys

**GET** `/api/v1/connections/{connectionId}/api-keys`

**Auth:** JWT Bearer (logged-in user)

**Response** `200 OK`:

```json
{
  "data": [
    {
      "api_key_id": "a1b2c3d4-5678-9012-abcd-ef1234567890",
      "prefix": "atk_XXXXXXXX",
      "name": "prod ingestion pipeline",
      "status": "ACTIVE",
      "created_at": "2026-07-28T14:30:00Z",
      "last_used_at": "2026-08-09T08:15:22Z"
    },
    {
      "api_key_id": "b2c3d4e5-6789-0123-bcde-f12345678901",
      "prefix": "atk_YYYYYYYY",
      "name": "staging test",
      "status": "REVOKED",
      "created_at": "2026-07-25T10:00:00Z",
      "last_used_at": ""
    }
  ],
  "errors": null
}
```

> **Note:** Never returns hashes or raw keys — only prefix, name, status, and timestamps. `last_used_at` is empty string if the key has never been used.

**curl:**

```bash
curl -X GET \
  https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c/api-keys \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

---

## 3. Revoke API Key

**DELETE** `/api/v1/connections/{connectionId}/api-keys/{keyId}`

**Auth:** JWT Bearer (logged-in user)

**Response** `200 OK`:

```json
{
  "data": {
    "api_key_id": "a1b2c3d4-5678-9012-abcd-ef1234567890",
    "status": "REVOKED"
  },
  "errors": null
}
```

> Soft-delete — the row stays for audit. Any subsequent upload attempt using this key returns `401`.

**Error responses:**

- `404` — key not found, connection not found, or key doesn't belong to this connection

**curl:**

```bash
curl -X DELETE \
  https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c/api-keys/a1b2c3d4-5678-9012-abcd-ef1234567890 \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

---

## 4. Upload Billing File (Customer's Backend)

**POST** `/api/v1/connections/{connectionId}/imports`

**Auth:** `X-Api-Key` header (not JWT — this is the customer's script, not the UI)

**Request:** `multipart/form-data` with a single `file` field. No `provider` field needed — it's resolved from the connection.

**Response** `202 Accepted`:

```json
{
  "data": {
    "import_id": "f8e7d6c5-b4a3-9281-7654-321098fedcba",
    "connection_id": "7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c",
    "status": "QUEUED"
  },
  "errors": null
}
```

> The file is staged and queued for async processing. The frontend (or script) can poll `GET /api/v1/imports/{importID}` to track status.

**Error responses:**

- `401` — missing, invalid, or revoked API key
- `404` — connection_id in URL doesn't match the key's connection
- `400` — no file, multiple files, or missing multipart
- `422` — unsupported file type (not Parquet or CSV)
- `502` — staging failure

**curl:**

```bash
curl -X POST \
  https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c/imports \
  -H "X-Api-Key: atk_dGhpcyBpcyBhIHRlc3Qga2V5IGZvciBkZW1v" \
  -F "file=@billing-2026-07.parquet"
```

**CSV example:**

```bash
curl -X POST \
  https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-4b1d-4e8a-9f2c-1a8b3d5e7f9c/imports \
  -H "X-Api-Key: atk_dGhpcyBpcyBhIHRlc3Qga2V5IGZvciBkZW1v" \
  -F "file=@billing-2026-07.csv;type=text/csv"
```

---

## Endpoint Summary

| Endpoint                                    | Method | Auth      | Purpose                    |
| ------------------------------------------- | ------ | --------- | -------------------------- |
| `/api/v1/connections/{id}/api-keys`         | POST   | JWT       | Create a new API key       |
| `/api/v1/connections/{id}/api-keys`         | GET    | JWT       | List keys for a connection |
| `/api/v1/connections/{id}/api-keys/{keyId}` | DELETE | JWT       | Revoke a key               |
| `/api/v1/connections/{id}/imports`          | POST   | X-Api-Key | Upload a billing file      |

```shell 
# Upload endpoint
POST https://{GATEWAY_URL}/api/v1/connections/7f3a9c2e-.../imports

# Example:
curl -X POST \
  https://{GATEWAY_URL}/v1/connections/7f3a9c2e-.../imports \
  -H "X-Api-Key: YOUR_KEY_HERE" \
  -F "file=@billing-2026-07.parquet"
```