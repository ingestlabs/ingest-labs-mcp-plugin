---
name: ingestlabs
description: >-
  Answer IngestLabs Insights / analytics questions via the ingestlabs MCP
  tools against production (mcp.ingestlabs.com): list_vendors, list_projects,
  list_insights_contexts, get_insights_schema, execute_insights_query,
  create_mdp_ai_report_from_insights. Use for production dashboards, reports,
  attribution, KPIs, IDL/CDP/media_tags metrics, and saving Insights queries as
  MDP AI reports.
---

# IngestLabs Insights (MCP) — Prod

Host LLM maps natural language → Insights schema ids (dimensions/metrics). MCP only discovers schema and executes.
**Environment: production** (`https://mcp.ingestlabs.com`).

## Prerequisites

1. MCP server **ingestlabs** connected (Streamable HTTP).
2. Production API:

```json
"ingestlabs": {
  "type": "http",
  "url": "https://mcp.ingestlabs.com/v1/mcp"
}
```

3. Complete OAuth (portal login) when your MCP client prompts. Endpoint is `/v1/mcp` (not `/mcp`).

If the server is connected but tools fail, follow [SETUP.md](../../SETUP.md).

## Scope (must resolve before analytics or config tools)

Resolve `vendor_id` (and `project_id` when needed) **before** calling Insights or Tag Manager tools.

1. **Organization / vendor**
   - If the user did not name an organization (or only used a nickname with more than one match): call **`list_vendors`**, present name + `vendor_id`, and **ask which org** to use.
   - Auto-pick only when the user has **exactly one** vendor, or the same vendor was already confirmed earlier in this conversation.
   - Never invent `vendor_id`. Never reuse example placeholders or another customer’s id.
2. **Project** (Tag Manager tools, or product `media_tags`)
   - If project is missing or ambiguous: call **`list_projects`** with the resolved `vendor_id`, present choices, and ask.
   - Do not call `execute_insights_query` for `media_tags` without `project_id`.
3. Keep the chosen scope for the rest of the thread unless the user switches org/project.

## Product and context routing

```
product: idl | cdp | media_tags
```

| Product | Notes |
| --- | --- |
| `idl` / `cdp` | Vendor-scoped; `project_id` usually not required |
| `media_tags` | **`project_id` required** |

When picking a context after `list_insights_contexts`, prefer match on **keywords / description** over name alone:

| User topic | Product | Prefer context id / keywords |
| --- | --- | --- |
| Attribution, MTA, channel revenue, journeys, UTM credit | `idl` | `mcp_attribution_touchpoints_trino` (attribution, touchpoints, MTA) |
| Meta / Facebook ads, ROAS, ad sets | `idl` | `mcp_facebook_ads_trino` |
| Google Ads, Search, Shopping, PMax | `idl` | `mcp_google_ads_trino` |
| GA4 daily property metrics (users, sessions) | `idl` | `mcp_ga4_metrics_daily_trino` |
| GA4 source / medium / channel | `idl` | `mcp_ga4_traffic_source_trino` |
| GA4 landing pages | `idl` | `mcp_ga4_landing_pages_trino` |
| Site events, sessions, ATC, checkout (IL) | `idl` | `mcp_sessionized_events_trino` |
| SaaS / product event metrics | `idl` | `mcp_events_saas_metrics_trino` |
| Traffic quality, bots, invalid traffic (TQS) | `idl` | `mcp_ingest_id_v2_tqs_trino` |
| Tag fire aggregates | `media_tags` | `mcp_il_trino_tag_records` |
| Raw tag recorded events | `media_tags` | `mcp_il_tag_recorded_events_trino` |

**Ambiguity rules**

- Ads without platform → ask Meta vs Google (or list both contexts and let the user choose).
- “Sessions / traffic” → distinguish GA4 bridge contexts vs IL sessionized events vs tag fires.
- “Revenue” by channel/journey → attribution context; do not treat ads **spend** as revenue.
- Attribution rows fan out (N touchpoints per conversion); use weighted attribution metrics as schema documents, do not naive-sum raw touchpoint rows as orders.
- If two contexts still look equally valid → show both and ask; do not guess.

## Mandatory Insights workflow

Always after scope is resolved, in this order:

1. **`list_insights_contexts`** — pick `context_id` from name / description / keywords
2. **`get_insights_schema`** — map NL fields to dimension/metric **ids**
3. **`execute_insights_query`** — run with those ids

Do not invent dimension or metric ids. Do not skip schema when unsure.
You may reuse a `context_id` and field ids already loaded in this session when the topic is unchanged; re-list when the domain changes.

## Execute rules

- Prefer **`date_preset`**: `today` | `yesterday` | `last_7d` | `last_30d` (vendor timezone).
- Or both **`from`** and **`to`** as epoch seconds GMT — never mix with `date_preset`.
- If `is_date_filter_optional` is false, dates are required.
- Use schema **ids** in `dimensions`, `metrics[].id`, `filters[].field`, `sort[].field`.
- For a dimension with `dim_typ: dt`, include `interval_suffix`: `m` | `h` | `d` | `w`.
- Obey each dimension's `is_mandatory` mode (`FILTER`, `SELECT`, or `FILTER_AND_SELECT`).
- Filter only dimensions with `filterable: true` (metrics are not filterable in execute); operators are case-insensitive and `CONTAINS` aliases `CONTAINS_IN`.
- `metrics[].aggregate_fn` is case-insensitive and optional for defaulted or `COMPUTED` metrics; ignore aggregate_fn on bare `COMPUTED` metrics.
- Prefer a small `limit` for exploration (e.g. 10–25); max 100. Prefer metric-only queries when the user wants one number. If `truncated: true`, say so and offer a tighter filter or higher limit.
- Need at least one dimension or metric.

## Save as MDP AI Report

After a successful **`execute_insights_query`**, the user may ask to save the query as a durable MDP AI report. Call **`create_mdp_ai_report_from_insights`** only when:

1. Scope is resolved (`vendor_id`, and `project_id` when `product` is `media_tags`).
2. You have already run **`execute_insights_query`** in this conversation and shown results.
3. The user **explicitly confirms** they want to save (name, audience flag, optional description/timing).

### Save workflow

1. Confirm with the user: report **name**, whether it **can be used as an audience** (`can_be_audience`), optional **description**, optional **start_timing** / **end_timing** for default run windows.
2. Reuse the **same** `vendor_id`, `product`, `project_id` (if any), `context_id`, `dimensions`, `metrics`, `filters`, `sort`, and `interval_suffix` as the last successful execute — **not** `date_preset`, `from`/`to`, or `limit` (those are execution-only).
3. Call **`create_mdp_ai_report_from_insights`**.
4. Return `report_id`, `name`, and `creation_source: 'MCP'`. Tell the user they can open the report in the portal under MDP Reports.

Do not call save before execute. Do not invent field ids — reuse ids from the prior execute (via schema). Do not save without user confirmation.

### Tool parameters

| Field | Required | Notes |
| --- | --- | --- |
| `vendor_id` | Yes | Confirmed org id |
| `product` | Yes | `idl` \| `cdp` \| `media_tags` |
| `project_id` | When `media_tags` | Same as execute |
| `context_id` | Yes | Same as execute |
| `name` | Yes | Report display name |
| `can_be_audience` | Yes | Boolean — ask the user |
| `description` | No | Optional |
| `dimensions` | No* | Same ids as execute |
| `metrics` | No* | Same ids + optional `aggregate_fn` as execute |
| `filters` | No | Same as execute (baked into compiled SQL) |
| `sort` | No | Same as execute |
| `interval_suffix` | When date dim | `m` \| `h` \| `d` \| `w` — same as execute |
| `start_timing` | No | Optional relative window start (`qty`, `unit`, `reset`) |
| `end_timing` | No | Optional relative window end (`qty`, `unit`, `reset`) |

\*At least one dimension or metric (same rule as execute).

**Not accepted:** `from`, `to`, `date_preset`, `limit`. Saved reports use relative `start_timing` / `end_timing` (when set) for portal runs, not the exploration date range.

### Portal behavior (MCP-created reports)

Reports saved via MCP have `creation_source: 'MCP'`. In the portal MDP Reports UI they support **view, run, export, delivery, and metadata/timing configuration** only:

- **Read-only query definition** — dimensions, metrics, filters, and sort are shown but not editable in v1 (no Thinkr chat panel).
- **Metadata edits** — name, description, audience flag, and timing can be updated; query definition and SQL are not changed via the portal.
- To change the query later, create a **new** report from MCP (v2 portal editor / MCP update tool deferred).

## Answering

- Lead with the number/table the user asked for.
- Cite **org name + vendor_id**, product, `context_id`, date range (`from`/`to` or preset), and that data is from **production**.
- On tool errors: fix args (ids, dates, `project_id`, scope) and retry once; then report the error.

## Tag Manager tools

- Hierarchy: vendor → project → providers / tags / data elements.
- List/get before create/update. Mutate only on **explicit** user request.
- Confirm project when the user named only the org.

## Out of scope

- Raw SQL / Thinkr nl2sql
- Editing saved MCP report query definitions in the portal (read-only in v1)
- Widget or dashboard CRUD
- `internal` Insights contexts
- Site performance / Lighthouse contexts (no `site_performance` product on these tools yet)
- Staging or local MCP environments (this plugin is production only)

## Examples

See [examples.md](examples.md).
