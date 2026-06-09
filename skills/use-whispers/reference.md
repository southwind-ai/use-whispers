# Southwind Whispers — API Reference

All paths are prefixed with `/api`. Authenticate every request with `X-API-Key: ak_...`.

**Base URL:** `https://app.southwind.ai/api`

> Canonical source of truth: `https://app.southwind.ai/api/docs` (live Redoc).
> This file is a curated reference for common developer use cases.

---

## Authentication

| Header | Value | Notes |
|--------|-------|-------|
| `X-API-Key` | `ak_<token>` | Required for all API calls |

No `Authorization` or `X-Organization-Id` headers needed — the org is inferred from the key.

### API Key management

| Method | Path | Permission needed |
|--------|------|-------------------|
| GET | `/v1/organization/api-keys` | `manage-api-keys` |
| POST | `/v1/organization/api-keys` | `manage-api-keys` |
| DELETE | `/v1/organization/api-keys/{key_id}` | `manage-api-keys` |

`POST` response includes `key_value` (`ak_...`) — shown once, store securely.

---

## Organization & Settings

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/me` | Current authenticated user |
| GET | `/v1/me/organizations` | All orgs the user belongs to |
| GET | `/v1/organization` | Current org details + active plan |
| GET | `/v1/organization/settings` | Language, currency, logo URL, glossary |
| PATCH | `/v1/organization/settings` | Update settings |
| POST | `/v1/organization/settings/logo` | Upload org logo (multipart) |
| GET | `/v1/organization/users` | List org members |
| POST | `/v1/organization/users` | Invite a user |
| DELETE | `/v1/organization/users` | Remove a user |

**Glossary**: an org-level dictionary of domain terms injected into agent prompts. Set via
`PATCH /v1/organization/settings` → `glossary` field. Use this to make reports
use your product's terminology.

---

## Agents

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/agents/` | List available agent types for this org |

Returns agent objects with `agent_id`, `display_name`, `description`.

---

## Data Origins

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/origins/` | List all origins (optional `?origin_id=<uuid>`) |
| DELETE | `/v1/origins/` | Bulk delete origins by IDs |
| GET | `/v1/origins/{origin_id}/sources` | List sources inside one origin |
| POST | `/v1/origins/{origin_id}/sources` | Add endpoint to an API origin + sync |
| POST | `/v1/origins/file/` | Ingest files → file or document origins |
| POST | `/v1/origins/data-api/` | Register external REST API origin |

### Ingest files — `POST /v1/origins/file/`

```json
{
  "files": [
    { "name": "sales_q1.csv", "url": "https://your-storage/sales_q1.csv" }
  ]
}
```

- File type is **auto-detected by extension**: `.csv`/`.xlsx` → tabular source; `.pdf`/`.txt`/`.md` → document source
- Returns: `{ "created_data_origins": [...], "unprocessed_files": [...] }`
- Each origin in `created_data_origins` has nested `data_sources[]`

### Register external API — `POST /v1/origins/data-api/`

```json
{
  "display_name": "Acme API",
  "base_url": "https://api.acme.com",
  "api_key": "secret-value",
  "api_key_header": "Authorization",
  "endpoints": [
    {
      "display_name": "Monthly Revenue",
      "path": "/v2/revenue/monthly",
      "format": "json",
      "query_params": { "year": "2025" }
    }
  ]
}
```

Returns: `{ "success": true, "data_sources": [...] }` — one source per endpoint.

### S3 multipart upload (large files)

For files too large to host at a URL, use the pre-signed S3 helpers:

| Method | Path |
|--------|------|
| GET | `/v1/origins/s3/params` |
| POST | `/v1/origins/s3/multipart` |
| GET | `/v1/origins/s3/multipart/{upload_id}/batch` |
| GET | `/v1/origins/s3/multipart/{upload_id}/{part_number}` |
| GET | `/v1/origins/s3/multipart/{upload_id}` |
| POST | `/v1/origins/s3/multipart/{upload_id}/complete` |
| DELETE | `/v1/origins/s3/multipart/{upload_id}` |

After upload, pass the resulting S3 keys to `POST /v1/origins/file/` as `s3_files` (see OpenAPI spec for `S3FileInput` shape).

---

## Data Sources

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/sources/` | List sources (`?source_id=` or `?origin_id=` filters) |
| DELETE | `/v1/sources/` | Bulk delete sources |
| GET | `/v1/sources/{source_id}/preview` | Paginated row preview |
| POST | `/v1/sources/{source_id}/sync` | Refresh data from upstream API |
| PATCH | `/v1/sources/{source_id}` | Update API source config + resync |
| PUT | `/v1/sources/{source_id}/tags` | Replace all tags on a source |

**Note**: Only API-backed sources can be synced. File sources are static.

### Response shape note (important for clients)

`GET /v1/sources/` may return one of these shapes depending on backend version:

- `{ "data_sources": [...] }` (common)
- `{ "items": [...] }`
- `[...]`

Robust clients should normalize all shapes before rendering.

---

## Reports

### Create a report — `POST /v1/reports/`

```json
{
  "agent_id": "custom_report",
  "data_sources_ids": ["<source-uuid>"],
  "params": {
    "language": "english",
    "currency": "USD",
    "prompt": "Analyze sales trends and highlight anomalies",
    "data_provenance": false
  },
  "improve_prompt": false
}
```

**Agent-specific params:**

| Agent | Extra params |
|-------|-------------|
| `custom_report` | `prompt` (required), `language`, `currency`, `data_provenance` |
| `rover_report` | `target_sections` (int), `data_provenance`, `seeded_question` (optional) |
| `blank_report` | None — completes immediately |

Returns: `{ "id": "<task_id>", "status": "queued", ... }`

### Report lifecycle

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/reports/` | List reports (`?user_filter=true` for own only) |
| GET | `/v1/reports/{task_id}` | Get status + full result blocks |
| DELETE | `/v1/reports/{task_id}` | Delete one report |
| DELETE | `/v1/reports/` | Bulk delete |
| PATCH | `/v1/reports/{task_id}/title` | Rename report |
| PUT | `/v1/reports/{task_id}` | Save manual edits to blocks |
| POST | `/v1/reports/{task_id}/redo` | Re-run with same config |

`GET /v1/reports/` is a listing endpoint and often returns summary objects (for example under `reports[]`), not full `result` blocks.
To render report content, always fetch `GET /v1/reports/{task_id}` after selecting a list item.

### Status polling

`GET /v1/reports/{task_id}` returns:

```json
{
  "id": "...",
  "type": "custom_report",
  "status": "completed",
  "last_update": 1775649600,
  "data_source_ids": ["..."],
  "title": "Q1 Sales Report",
  "params": { "language": "english", "currency": "USD", "prompt": "..." },
  "result": [ ... ],
  "decision_tree": [ ... ]
}
```

- `result` — BlockNote document blocks (only when `status: completed`)
- `decision_tree` — Rover investigation path (only on rover reports, top-level field)
- On failure: `params.error` contains the error message; `result` is absent
- Statuses: `queued` | `running` | `completed` | `failed`

### SSE event stream — `GET /v1/reports/{task_id}/events`

Real-time events (text/event-stream):
- `event: status` — `data: { "status": "...", "progress": 0-100 }` on each poll
- `event: thought` — `data: { "message": "...", ... }` agent reasoning step
- `event: done` — stream ended; task is `completed` or `failed` (fetch report to get result)

Note: the **chat stream** (`/chat`) uses different events: `data:` chunks, `event: error`, `event: chatComplete`.

### AI section — `POST /v1/reports/{task_id}/ai-section`

```json
{ "prompt": "Add a section on customer churn by cohort" }
```

Returns: `{ "blocks": [...] }` — merge into report and `PUT` to save.

### Prompt improvement — `POST /v1/reports/improve-prompt`

```json
{ "prompt": "sales analysis", "data_sources_ids": ["..."] }
```

Returns an enriched, context-aware prompt suggestion. Rate limit: ~20/min.

### Templates and dataset info

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/reports/templates` | List saved prompt templates |
| POST | `/v1/reports/templates` | Create a template |
| DELETE | `/v1/reports/templates/{template_id}` | Delete a template |
| GET | `/v1/reports/dataset-info` | List saved dataset context snippets |
| POST | `/v1/reports/dataset-info` | Create a dataset info |
| DELETE | `/v1/reports/dataset-info/{dataset_info_id}` | Delete a dataset info |

---

## Chat

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/reports/{task_id}/chat` | Send message → SSE stream response |
| GET | `/v1/reports/{task_id}/chat/history` | Full chat history |
| DELETE | `/v1/reports/{task_id}/chat/history` | Clear chat |

---

## Sharing & Export

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/reports/{task_id}/share` | Get share state |
| PUT | `/v1/reports/{task_id}/share` | Enable/disable public sharing |
| GET | `/v1/public/reports/{token}` | View shared report (no auth) |
| GET | `/v1/reports/{task_id}/export` | Download PDF, PPTX, or DOCX |

Export params: `?format=pdf` | `?format=pptx` | `?format=docx`

---

## Provenance

When `data_provenance: true` is set at report creation, agents record the exact
data rows that backed each claim.

```
GET /v1/reports/{task_id}/provenance/{provenance_id}
```

Provenance IDs are referenced inside result blocks.

---

## Billing & Usage

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/billing/plans` | Available subscription plans |
| GET | `/v1/billing/status` | Current org billing status |
| GET | `/v1/billing/usage` | Current plan limit consumption |
| POST | `/v1/billing/checkout` | Initiate Stripe checkout |
| GET | `/v1/billing/portal` | Stripe customer portal URL |

Check usage before resource-intensive operations; a 402 means a limit is reached.

---

## Health

```
GET /health
→ 200 OK
```

---

## Result Block Types

`result` is a **[BlockNote](https://www.blocknotejs.org/) document** — an array of block
objects in standard BlockNote JSON format.

Each block:
```json
{
  "id": "<uuid>",
  "type": "<block-type>",
  "props": { ... },
  "content": [ { "type": "text", "text": "...", "styles": {} } ],
  "children": []
}
```

### Block type reference

| `type` | Standard BlockNote? | Notable props |
|--------|--------------------|--------------------|
| `paragraph` | yes | `content[]` inline text |
| `heading` | yes | `props.level` (1–3) |
| `bulletListItem` | yes | nestable via `children[]` |
| `numberedListItem` | yes | nestable via `children[]` |
| `table` | yes | standard BlockNote table format |
| `image` | yes | `props.url`, `props.caption` |
| `columnList` / `column` | xl-multi-column extension | `children[]` of column blocks |
| `chart` | **custom** | `props.config` — ECharts option as JSON **string** |
| `alert` | **custom** | `content[]` inline text |
| `statistic` | **custom** | varies |

Provenance references appear as inline markers within `paragraph` blocks pointing to a `provenance_id`.

### Rendering options

**Option A — BlockNote renderer**: pass `result` as `initialContent` to a `BlockNoteEditor`.
Register the custom block specs (`chart`, `alert`, `statistic`) to handle those block types.

**Option B — Manual walk**: iterate blocks and render each type yourself.

### Rendering chart blocks

Chart config is at `block.props.config`. Library: **Apache ECharts**.
Depending on report/version, `props.config` may be:

- a JSON string (parse first), or
- an already-materialized object (use directly)

```js
import * as echarts from "echarts";

function renderChart(block, containerEl) {
  const raw = block.props.config;
  const option = typeof raw === "string" ? JSON.parse(raw) : raw;
  if (!option || typeof option !== "object") return;

  const chart = echarts.init(containerEl, null, { renderer: "svg" });
  chart.setOption({ ...option, backgroundColor: "transparent" });

  const observer = new ResizeObserver(() => chart.resize());
  observer.observe(containerEl);

  return () => { observer.disconnect(); chart.dispose(); };
}
```

Rules:
- Config is at `block.props.config`, **not** `block.config`
- Support both string and object config values
- SVG renderer (`renderer: "svg"`)
- `backgroundColor: "transparent"`
- Attach `ResizeObserver`
- Recommended sizes: pie/gauge/funnel → 600×400px; all others → 800×400px

---

## Permissions Reference

API keys inherit the permissions of their creating user:

| Permission | Grants access to |
|------------|-----------------|
| `generate-report` | Create and manage reports |
| `manage-files` | Upload and delete data origins/sources |
| `manage-glossary` | Edit org glossary |
| `manage-org-settings` | Org settings (language, currency, logo) |
| `manage-users` | Invite / remove team members |
| `manage-billing` | Billing and plan management |
| `manage-api-keys` | Create / revoke API keys |
