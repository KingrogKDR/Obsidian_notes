## Context

When a customer connects their Azure tenant to Atomity, the first step is granting admin consent to Atomity's multi-tenant Entra ID app registration. This is a browser-based flow: the customer's Global Administrator visits a Microsoft-hosted consent URL, reviews the requested access, and clicks Accept.

After accepting, Azure redirects the admin's browser to a `redirect_uri` we control, appending query parameters that confirm the consent succeeded and identify which tenant just consented. The callback page needs to receive this redirect, interpret the result, and route the admin back into the connection wizard at the right step.

The consent itself is already complete server-side by the time the redirect fires — the callback page doesn't need to call any API to "finalize" consent. Its job is purely UX: tell the admin it worked, and get them back into the wizard without re-entering information Azure already confirmed.

---

## Configuration

Define a single environment-level constant for the callback URL. Every reference below uses this constant — nothing is hardcoded.

```
AZURE_CONSENT_REDIRECT_URI=<e.g. https://{BASE_URL}/connections/azure/callback>
```

This same value must be registered as a Web redirect URI on the Atomity app registration in Azure Portal → App registrations → Authentication. Character-for-character match is required (scheme, host, path, trailing slash presence/absence).

---

## What Azure sends to the callback

**On success** (admin clicked Accept):

```
GET {AZURE_CONSENT_REDIRECT_URI}?admin_consent=True&tenant={customer_tenant_id}&state={state_value}
```

| Param           | Type   | Description                                                                                                                                                                  |
| --------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `admin_consent` | string | Always the literal string `"True"` on success.                                                                                                                               |
| `tenant`        | string | The Entra tenant ID (GUID) of the tenant that just consented. This is the customer's tenant, confirmed by Azure itself — more reliable than the customer typing it manually. |
| `state`         | string | Echoed back unchanged from whatever we sent in the consent URL. We send the `connection_id` here (see below). Absent if the consent URL didn't include `state`.              |

**On failure** (admin declined, or something went wrong):

```
GET {AZURE_CONSENT_REDIRECT_URI}?error={error_code}&error_description={url_encoded_message}&tenant={customer_tenant_id}&state={state_value}
```

|Param|Type|Description|
|---|---|---|
|`error`|string|Error code, e.g. `"access_denied"`.|
|`error_description`|string|URL-encoded human-readable message, e.g. `"The+admin+declined+the+request"`.|
|`tenant`|string|May or may not be present on failure.|
|`state`|string|Same as success — echoed back unchanged.|

Both are standard GET requests with query parameters. No POST body, no fragment, no special headers.

---

## What needs to be built

### 1. Callback page/route

Create a route that `AZURE_CONSENT_REDIRECT_URI` resolves to. This is a client-side page, not an API route — it receives a browser redirect, not a backend webhook.

**Success behavior** (`admin_consent=True` present, `error` absent):

- Show a brief success state: _"Consent granted — returning to connection setup..."_
- Extract `tenant` (the confirmed Entra tenant ID) and `state` (the connection_id) from query params.
- Redirect to the Azure connection wizard, resuming at **step 2** (RBAC role assignment), passing both values so the wizard can:
    - Pre-fill the tenant ID field (eliminating manual entry and typo risk).
    - Know which connection is being set up (from the connection_id in `state`).

**Failure behavior** (`error` present):

- Show the error: _"Consent was not granted"_ with the decoded `error_description` as detail text.
- Provide a **"Try again"** action that navigates back to step 1 of the Azure connection wizard. If `state` (connection_id) is available, link back to that specific connection's wizard; otherwise, link to the connections list.

**No query params at all** (direct navigation, bookmark, etc.):

- Redirect to the connections list page. This route has no standalone purpose outside the consent flow.

### 2. Consent URL construction update

The Azure onboarding wizard's step 1 currently builds a consent URL like:

```
https://login.microsoftonline.com/{customer_tenant_id}/adminconsent?client_id={ATOMITY_AZURE_CLIENT_ID}
```

Update it to include two additional parameters:

```
https://login.microsoftonline.com/{customer_tenant_id}/adminconsent
  ?client_id={ATOMITY_AZURE_CLIENT_ID}
  &redirect_uri={AZURE_CONSENT_REDIRECT_URI}
  &state={connection_id}
```

- `redirect_uri` — the same constant defined above. Must be URL-encoded if it contains special characters.
- `state` — the `connection_id` of the in-progress Azure connection (returned by `POST /api/v1/connections` when the connection was first created). Azure does not interpret, validate, or modify this value — it echoes it back verbatim on the callback.

### 3. Wizard step 1 — auto-fill tenant ID

When the wizard loads with a tenant ID already available (either from the callback redirect's query params or from the connection's existing `azure_tenant_id` if resuming a partially-completed setup), pre-fill the "Your Entra tenant ID" input field and mark it as confirmed/read-only. The value came directly from Azure's own redirect, so it's authoritative — the customer should not need to re-type or edit it.

The manual input should remain available as a fallback for cases where the customer navigates to the wizard directly rather than through the callback flow.

---

## Security: state parameter validation

The `state` parameter is not just a UX convenience — it prevents CSRF.

Without validation, an attacker could craft a URL like:

```
{AZURE_CONSENT_REDIRECT_URI}?admin_consent=True&tenant=<attacker-tenant-id>&state=<victim-connection-id>
```

and trick someone into visiting it, potentially linking the wrong tenant to their connection.

**Required behavior:** On the callback page, before redirecting back to the wizard, verify that the `state` value (connection_id) corresponds to a real, in-progress Azure connection that the current logged-in user owns. The simplest implementation: check that the connection_id exists in the user's connection list (`GET /api/v1/connections`) and is in `PENDING` status with provider `CLOUD_PROVIDER_AZURE`. If it doesn't match, show an error rather than proceeding.

This is standard OAuth2 practice, not Azure-specific.

---

## Testing checklist

- [ ] Admin clicks consent URL → sees Microsoft consent screen (not "unable to sign you in").
- [ ] Admin accepts → redirected to callback page → sees success message → auto-redirected to wizard step 2.
- [ ] Tenant ID is pre-filled in the wizard and matches what Azure returned.
- [ ] Connection ID survives the round-trip (wizard step 1 → Azure → callback → wizard step 2).
- [ ] Admin declines consent → redirected to callback page → sees error with description → "Try again" links back to step 1.
- [ ] Direct navigation to callback URL (no query params) → redirected to connections list.
- [ ] Forged callback URL with wrong `state` → rejected, not processed.
- [ ] Consent URL works from an incognito window (no cached Azure session).
- [ ] Consent URL works when the admin is in a different Azure tenant than Atomity's own.

---

## Not in scope for this task

- The callback page does **not** call any Atomity backend API to "complete" or "register" the consent. Consent is recorded entirely on Azure's side. The backend learns about it later when the customer hits `/validate` and Atomity's Service Principal successfully authenticates against their tenant.
- No backend changes are needed for this task. The `redirect_uri` registration is an Azure Portal config change, not a code change.
- The `ATOMITY_AZURE_CLIENT_ID` constant already exists (used by `ConnectionsController.java`'s `@Value("${atomity.azure.client-id}")` and returned in the `POST /api/v1/connections` response as `atomity_azure_client_id`). Reuse it — do not duplicate.