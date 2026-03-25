# CIPP Integration Setup Guide

[CIPP (CyberDrain Improved Partner Portal)](https://cipp.app) is an open-source
multitenant Microsoft 365 management platform for MSPs. This integration syncs
your CIPP-managed customer tenants to Bifrost organizations and exposes CIPP's
API surface to Bifrost workflows and agents.

---

## What you get

- **Tenant sync**: CIPP customer tenants → Bifrost org mappings (`entity_id = defaultDomainName`)
- **Org-mapping picker**: live tenant dropdown in the Bifrost integration UI
- **350+ API endpoints** available to workflows via `modules/cipp.CIPPClient`:
  users, licenses, alerts, Defender, domain health, Conditional Access,
  Intune/Autopilot, Exchange, and more

---

## Prerequisites

- CIPP deployed and managing your customer tenants
- Application Administrator (or Global Administrator) role in your Azure AD tenant
- Azure CLI (`az`) available, or access to the Azure Portal
- Bifrost running with git sync enabled

---

## Step 1 — Create a CIPP API Client

1. In the CIPP UI, go to **Settings → Backend → API Clients**
2. Click **Add API Client**, give it a name (e.g. `Bifrost`)
3. CIPP will display a **Client ID** and **Client Secret** — copy them immediately,
   the secret is shown only once
4. Note the **tenant ID** shown in your token URL and the **API scope**
   (typically `api://{client-id}/.default`)

Store these in your secrets manager. Example using `pass`:

```bash
pass insert cipp/client-id       # paste the Client ID
pass insert cipp/api-secret      # paste the Client Secret
pass insert cipp/azuread-tenant-id  # your Azure AD tenant ID
pass insert cipp/base-url        # https://{your-cipp-instance}.azurewebsites.net
pass insert cipp/token-url       # https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
pass insert cipp/api-scope       # api://{client-id}/.default
```

---

## Step 2 — Fix the Application ID URI (critical)

> **This step is required.** CIPP creates the Azure AD app registration for the
> API client but does not set the Application ID URI. Without it, every token
> request will fail with `AADSTS500011: resource principal not found`, even after
> granting admin consent.

### Via Azure CLI (recommended)

```bash
CLIENT_ID=$(pass show cipp/client-id)   # or paste the client ID directly

az login   # sign in as Application Administrator or Global Administrator
az ad app update \
  --id "$CLIENT_ID" \
  --identifier-uris "api://$CLIENT_ID"

# Verify
az ad app show --id "$CLIENT_ID" --query "identifierUris" -o json
# Expected: ["api://{client-id}"]
```

### Via Azure Portal

1. Go to **Azure AD → App Registrations** → find the app by its client ID
2. Click **Expose an API**
3. Next to "Application ID URI", click **Add** (or **Set**)
4. Accept the default `api://{client-id}` and click **Save**

---

## Step 3 — Grant admin consent

The app registration needs a service principal in your tenant before Azure AD
will issue tokens for it.

```bash
# Grant admin consent via Azure Portal:
# Azure AD → App Registrations → {your app} → API permissions
# → Grant admin consent for {your tenant}

# Or verify the service principal exists via CLI:
az ad sp show --id $(pass show cipp/client-id) --query "displayName" -o tsv
```

If the service principal doesn't exist yet:

```bash
az ad sp create --id $(pass show cipp/client-id)
```

---

## Step 4 — Verify authentication

```bash
curl -s -X POST "$(pass show cipp/token-url)" \
  -d "grant_type=client_credentials" \
  -d "client_id=$(pass show cipp/client-id)" \
  -d "client_secret=$(pass show cipp/api-secret)" \
  -d "scope=$(pass show cipp/api-scope)" \
  | python3 -m json.tool

# Should return: { "access_token": "eyJ...", "token_type": "Bearer", ... }
```

Then confirm API access:

```bash
TOKEN=$(curl -s -X POST "$(pass show cipp/token-url)" \
  -d "grant_type=client_credentials" \
  -d "client_id=$(pass show cipp/client-id)" \
  -d "client_secret=$(pass show cipp/api-secret)" \
  -d "scope=$(pass show cipp/api-scope)" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s "$(pass show cipp/base-url)/api/ListTenants" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{len(d)} tenants')"
```

---

## Step 5 — Register the integration in Bifrost

```bash
# POST /api/integrations
curl -s -X POST https://your-bifrost/api/integrations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CIPP",
    "entity_id": "defaultDomainName",
    "entity_id_name": "Tenant",
    "config_schema": [
      {"key": "base_url",      "type": "string", "required": true, "position": 0,
       "description": "CIPP API base URL (e.g. https://cippidlq5.azurewebsites.net)"},
      {"key": "tenant_id",     "type": "string", "required": true, "position": 1,
       "description": "Azure AD tenant ID"},
      {"key": "client_id",     "type": "string", "required": true, "position": 2,
       "description": "API client application ID"},
      {"key": "client_secret", "type": "secret", "required": true, "position": 3,
       "description": "API client secret"},
      {"key": "api_scope",     "type": "string", "required": true, "position": 4,
       "description": "CIPP API OAuth scope (api://{client-id}/.default)"}
    ]
  }'

# PUT /api/integrations/{id}/config with credentials from pass
```

Then push the workflows and register them:

```bash
bifrost push modules/cipp.py features/cipp/
bifrost workflow register features/cipp/workflows/sync_tenants.py
bifrost workflow register features/cipp/workflows/data_providers.py
```

Set the data provider on the integration:

```bash
# PUT /api/integrations/{cipp-id}
# body: { "list_entities_data_provider_id": "{list-cipp-tenants-workflow-id}" }
```

---

## Step 6 — Run the sync

```bash
bifrost run features/cipp/workflows/sync_tenants.py --workflow sync_cipp_tenants
```

Expected output:

```json
{
  "total": 124,
  "mapped": 120,
  "already_mapped": 0,
  "created_orgs": 4,
  "errors": []
}
```

---

## Usage in workflows

```python
from modules.cipp import get_client

async def my_workflow(org):
    client = await get_client()
    try:
        # get_client() reads base_url, tenant_id, client_id, etc. from
        # the "CIPP" Bifrost integration config
        tenants = await client.list_tenants()

        # scope to a specific org's tenant via IntegrationMapping
        cipp = await integrations.get("CIPP", scope=org.id)
        tenant_domain = cipp.entity_id  # e.g. "contoso.onmicrosoft.com"

        users    = await client.list_users(tenant_domain)
        licenses = await client.list_licenses(tenant_domain)
        alerts   = await client.list_alerts(tenant_domain)
    finally:
        await client.close()
```

The generic `call()` method covers any endpoint not wrapped by a named method:

```python
result = await client.call("ListMailboxes", tenantFilter="contoso.onmicrosoft.com")
result = await client.call("ExecSetOoO", method="POST",
                           tenantFilter="contoso.onmicrosoft.com",
                           userid="user@contoso.com",
                           AutoReplyState="Enabled",
                           InternalMessage="Out of office")
```

---

## Troubleshooting

### `AADSTS500011: resource principal not found`

The Application ID URI is not set on the app registration. Run:

```bash
az ad app update --id {client-id} --identifier-uris "api://{client-id}"
```

This is the most common failure and is not caused by wrong credentials or
missing admin consent — it is purely a missing field on the app registration
that CIPP does not set automatically.

### `AADSTS70011: invalid scope`

The `api_scope` value doesn't match `identifierUris`. Verify:

```bash
az ad app show --id {client-id} --query "identifierUris" -o json
```

The scope in `pass show cipp/api-scope` must match exactly.

### `401 Unauthorized` from CIPP API (token acquired successfully)

The CIPP Function App's authentication may be checking for specific app roles.
Check CIPP Settings → Backend → API Clients and confirm the client is listed
as active. If CIPP shows the client but calls still 401, the Function App's
Easy Auth may need the service principal to be explicitly assigned.

### Service principal already exists error from `az ad sp create`

The service principal was already created (possibly by a prior admin consent).
This is not an error — proceed to verify the `identifierUris` instead.
