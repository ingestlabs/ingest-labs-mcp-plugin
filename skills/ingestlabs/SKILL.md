---
name: ingestlabs
description: >-
  Answer IngestLabs Insights / analytics questions via the ingestlabs MCP
  tools against production (mcp.ingestlabs.com): list_vendors, list_projects,
  list_insights_contexts, get_insights_schema, execute_insights_query,
  create_mdp_ai_report_from_insights, list_mdp_ai_reports, get_mdp_ai_report,
  execute_mdp_ai_report, update_mdp_ai_report_from_insights. Use for production
  dashboards, reports, attribution, KPIs, IDL/CDP/media_tags metrics, and MDP AI
  report lifecycle.
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

When picking a context after `list_insights_contexts`, prefer match on **keywords / description** over name alone.
Use the **`context_id` returned by the tool** (do not invent ids). Common MCP catalog ids:

| User topic | Product | Prefer context id / keywords |
| --- | --- | --- |
| Attribution, MTA, channel revenue, UTM credit (touchpoint grain) | `idl` | `mcp_attribution_touchpoints` |
| Conversion / order list with first+last touch UTMs (txn grain) | `idl` | `mcp_mdp_attributed_transactions` |
| One-shot touchpoint + Shopify/ads/session mega-merge (legacy) | `idl` | `mcp_mdp_attribution_journey` — prefer ATP/mat_txn + enrichments instead when possible |
| Meta / Facebook ads, ROAS, ad sets | `idl` | `mcp_facebook_ads` |
| Google Ads, Search, Shopping, PMax | `idl` | `mcp_google_ads` |
| TikTok ads | `idl` | `mcp_tiktok_ads` |
| Shopify orders / customers | `idl` | `mcp_shopify_orders` / `mcp_shopify_customers` |
| GA4 daily property metrics (users, sessions) | `idl` | `mcp_ga4_metrics_daily` |
| GA4 source / medium / channel | `idl` | `mcp_ga4_traffic_source` |
| GA4 landing pages | `idl` | `mcp_ga4_landing_pages` |
| Site events, sessions, ATC, checkout (IL) | `idl` | `mcp_sessionized_events` |
| Item / SKU events | `idl` | `mcp_events_il_items` |
| SaaS / product event metrics | `idl` | `mcp_events_saas_metrics` |
| Traffic quality, bots, invalid traffic (TQS) | `idl` | `mcp_ingest_id_v2_tqs` |
| Tag fire aggregates | `media_tags` | `mcp_il_trino_tag_records` |
| Raw tag recorded events | `media_tags` | `mcp_il_tag_recorded_events` |

**Ambiguity rules**

- Ads without platform → ask Meta vs Google vs TikTok (or list contexts and let the user choose).
- “Sessions / traffic” → distinguish GA4 bridge contexts vs IL sessionized events vs tag fires.
- “Revenue” by channel/journey → attribution context; do not treat ads **spend** as revenue.
- Attribution rows fan out (N touchpoints per conversion); use weighted attribution metrics as schema documents, do not naive-sum raw touchpoint rows as orders.
- Transaction list with FT/LT UTMs → `mcp_mdp_attributed_transactions`, not touchpoint grain.
- Prefer **primary grain + curated enrichments** over `mcp_mdp_attribution_journey` when the question spans domains (see Cross-context joins). Journey still works for deep ads hierarchy in one shot until enrichment-intra walks are universal.
- If two contexts still look equally valid → show both and ask; do not guess.

## Mandatory Insights workflow

Always after scope is resolved, in this order:

1. **`list_insights_contexts`** — pick primary `context_id` (defines **row grain**) from name / description / keywords
2. **`get_insights_schema`** — map NL fields to dimension/metric **ids**; read **`joinable_contexts`**
3. If the question needs columns from another domain listed in `joinable_contexts`: call **`get_insights_schema` again** with `include_context_ids` (max 3) to merge those dims/metrics
4. **`execute_insights_query`** — primary `context_id` + optional **`enrichment_context_ids`** (same ids as include) + field ids from the merged schema

Do not invent dimension or metric ids. Do not invent join keys or enrichment contexts. Do not skip schema when unsure.
You may reuse a `context_id` and field ids already loaded in this session when the topic is unchanged; re-list when the domain changes.

## Cross-context joins (curated)

Some primaries allow **left-join enrichments** from an allowlisted registry (not free joins).

| Concept | Meaning |
| --- | --- |
| Primary `context_id` | Owns row grain and date window |
| `joinable_contexts` | From `get_insights_schema` — which contexts may be enriched, with `join_summary` / `edge_id` |
| `include_context_ids` | Schema-only: merge enrichment dims/metrics into the schema response (max 3) |
| `enrichment_context_ids` | Execute/report: actually left-join those contexts (must appear in `joinable_contexts`) |

**Rules**

- Only enrich contexts returned in `joinable_contexts` for that primary. Never invent edges.
- Date filters apply to the **primary** only; enrichments are left joins (nulls when unmatched).
- Typical compositions: `mcp_mdp_attributed_transactions` or `mcp_attribution_touchpoints` + `mcp_shopify_orders` (+ customers via SO→SC), sessionized, item events, or ads **campaign** contexts.
- Deeper ads hierarchy (adset/ad) after campaign enrichment may still require the ads context alone or journey until the planner walks enrichment intra-relationships.
- When saving an MDP AI report after an enriched execute, pass the same **`enrichment_context_ids`** (and field ids) as the execute — not dates/limit.

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
3. The user **explicitly confirms** they want to save (report **name**; optional description/timing).

### Save workflow

1. Confirm with the user: report **name**, optional **description**, optional **start_timing** / **end_timing** for default run windows. Do **not** ask about `can_be_audience` — default it to `false` (omit the field, or pass `false`). Only set `true` if the user explicitly asks for audience eligibility.
2. Reuse the **same** `vendor_id`, `product`, `project_id` (if any), `context_id`, `enrichment_context_ids` (if any), `dimensions`, `metrics`, `filters`, `sort`, and `interval_suffix` as the last successful execute — **not** `date_preset`, `from`/`to`, or `limit` (those are execution-only).
3. Call **`create_mdp_ai_report_from_insights`**.
4. In the chat response, show only **Report ID** (`report_id`) and **Report URL** (`portal_url` as a clickable link). Do **not** display `creation_source` (or raw field names like `report_id` / `portal_url`). Optionally mention the report name in the opening line.

Do not call save before execute. Do not invent field ids — reuse ids from the prior execute (via schema). Do not save without user confirmation.

### Tool parameters

| Field | Required | Notes |
| --- | --- | --- |
| `vendor_id` | Yes | Confirmed org id |
| `product` | Yes | `idl` \| `cdp` \| `media_tags` |
| `project_id` | When `media_tags` | Same as execute |
| `context_id` | Yes | Same as execute |
| `name` | Yes | Report display name |
| `can_be_audience` | No | Default `false`. Do **not** ask — only set `true` if the user explicitly requests audience use |
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

## Manage MDP AI Reports

After reports exist, use these tools (same vendor scope rules as Insights):

| Tool | When |
| --- | --- |
| `list_mdp_ai_reports` | Discover reports. **Required** `scope`: `CUSTOM` or `PREBUILT`. For AI-saved reports prefer `scope=CUSTOM` + `creation_source=MCP`. Optional `search`, `offset`, `limit` (max 100). |
| `get_mdp_ai_report` | Inspect metadata, `idl_query_request` (with context name/description), timings, column map. SQL omitted unless `include_sql: true` (debugging only). |
| `execute_mdp_ai_report` | Run saved SQL for a date range. Same date rules as Insights: `date_preset` **or** both `from`+`to`. Default `limit` 25, max 100. Does not change the stored definition. |
| `update_mdp_ai_report_from_insights` | Recompile an **MCP** report’s query from a new Insights shape. Same confirm + prior successful `execute_insights_query` rules as create. Only `creation_source=MCP` reports. |

### Update workflow

1. User confirms they want to change an existing MCP report (and which `report_id`).
2. Run **`execute_insights_query`** with the new fields and show results.
3. Call **`update_mdp_ai_report_from_insights`** with the same field ids (no dates/limit) plus `report_id`.
4. In the chat response, show only **Report ID** and **Report URL** (same rules as create). Do **not** display `creation_source`.

Do not update AI_CHAT / Thinkr reports via this tool. Do not pass client SQL.

### Portal behavior (MCP-created reports)

Reports saved via MCP have `creation_source: 'MCP'`. In the portal MDP Reports UI they support **view, run, export, delivery, and metadata/timing configuration**:

- **Read-only query definition** in the portal — use **`update_mdp_ai_report_from_insights`** to change fields via MCP.
- **Metadata edits** — name, description, audience flag, and timing can also be updated in the portal without changing SQL.

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
- Editing AI_CHAT report queries via MCP (Thinkr/portal owns those)
- Widget or dashboard CRUD
- `internal` Insights contexts
- Site performance / Lighthouse contexts (no `site_performance` product on these tools yet)
- Staging or local MCP environments (this plugin is production only)

## Examples

See [examples.md](examples.md).
