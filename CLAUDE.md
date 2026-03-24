# Bifrost Community Workspace

## HaloPSA API Patterns

- **POST endpoints expect a list payload**: `create_quotation([{...}])` not `create_quotation({...})`. Sending a plain dict returns 400. This applies broadly — tickets, clients, quotations, etc.
- **Create and update use the same POST endpoint**: include `id` in the payload to update an existing record.
- **API returns DotDict** (not plain dict) — normalize with `ticket if isinstance(ticket, dict) else dict(ticket)` before using `.get()`.
- **SQL queries**: `POST /Report` with `[{"sql": query, "_testonly": true, "_loadreportonly": true}]` — rows at `result.report.rows`, all values returned as strings.
- **No ORDER BY in SQL** — HaloPSA SQL Views don't support it. Sort client-side in Python.
- **String comparisons**: use `<>` not `!=` in HaloPSA SQL.
- SDK client handles retries with exponential backoff (429/5xx) — don't add retry logic in helpers.

## Bifrost SDK Patterns

- **Max table query limit is 1000** — requests with limit >1000 return 422.
- **Table operators**: `in`, `gte`, `lte`, `gt`, `lt`, `ilike`, `like`, `is_null`.
- **`order_by` combined with `in`** on some tables can cause 422 — sort client-side when using bulk `in` queries.
- **Forms**: Dynamic dropdowns use `data_provider_id` (flat UUID string), NOT nested `data_provider: {id: ...}` object.
- **Forms**: `type: select` for dropdowns — `multiselect` is NOT a valid form field type.
- **AI SDK**: `ai.complete()` does not support a `temperature` parameter.
- **Always use `--params` with `bifrost run`** to avoid opening a browser: `bifrost run file.py --workflow name --params '{}'`.

## Key File Locations

- HaloPSA extensions: `modules/extensions/halopsa.py`
- Auto-generated HaloPSA SDK: `modules/halopsa.py` (3.9MB)
- Microsoft CSP workflows: `features/microsoft_csp/workflows/`
- AutoElevate agent tools: `features/autoelevate/workflows/tools.py`
- HaloPSA Report Agent workflows: `features/halopsa_reporting/`
