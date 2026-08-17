# IngestLabs Insights — examples (Prod)

Replace `VENDOR_ID` / `PROJECT_ID` with real ids from MCP or the user. Data is production.

## Example 0 — Org missing from the prompt (always do this first)

**User:** “What was revenue by channel last week?”

**Bad:** Call `list_insights_contexts` with a guessed `vendor_id`, or invent an id.

**Good:**

1. `list_vendors` `{}`
2. If **multiple** vendors: list name + `vendor_id` and **ask** which organization. Stop until the user picks (or has already confirmed one in this thread).
3. If **exactly one** vendor: use it and say which org you are using.
4. Only then: `list_insights_contexts` → `get_insights_schema` → `execute_insights_query` with that `vendor_id`.

**Similar for media tags / Tag Manager without project:** after `vendor_id`, call `list_projects` and ask when project is missing or ambiguous.

## Example 1 — IDL revenue last 7 days by channel

**User:** “What was revenue by channel last week for vendor VENDOR_ID?”

1. Scope already given → skip `list_vendors` (or use it only to resolve a display name).
2. `list_insights_contexts` `{ "vendor_id": "VENDOR_ID", "product": "idl" }`  
   → choose context whose keywords/description match attribution / revenue / channel (prefer `mcp_attribution_touchpoints_trino`).
3. `get_insights_schema` `{ "vendor_id": "VENDOR_ID", "product": "idl", "context_id": "<chosen>" }`  
   → note dimension id for channel, metric id for revenue (+ default `aggregate_fn`).
4. `execute_insights_query`:

```json
{
  "vendor_id": "VENDOR_ID",
  "product": "idl",
  "context_id": "<chosen>",
  "dimensions": ["<channel_dim_id>"],
  "metrics": [{ "id": "<revenue_metric_id>" }],
  "date_preset": "last_7d",
  "sort": [{ "field": "<revenue_metric_id>", "dir": "desc" }],
  "limit": 25
}
```

**Answer:** Rank channels by revenue; cite org + `context_id` + date range; mention `truncated` if true. Note **production**.

## Example 2 — Media Tags event counts for a project

**User:** “How many recorded tag events yesterday on project PROJECT_ID?” (vendor already known as VENDOR_ID)

1. `list_insights_contexts` `{ "vendor_id": "VENDOR_ID", "product": "media_tags" }`
2. `get_insights_schema` for the events context; confirm mandatory project filter.
3. `execute_insights_query`:

```json
{
  "vendor_id": "VENDOR_ID",
  "product": "media_tags",
  "project_id": "PROJECT_ID",
  "context_id": "<events_context_id>",
  "metrics": [{ "id": "<event_count_metric_id>" }],
  "date_preset": "yesterday",
  "limit": 25
}
```

Omit `project_id` → tool error; do not call execute without it. If project was not named, `list_projects` first and ask.

## Example 3 — Absolute GMT range

**User:** “Show top 10 campaigns from 2026-07-01 through 2026-07-15 UTC.” (vendor already known)

Convert bounds to epoch seconds GMT yourself, then:

```json
{
  "vendor_id": "VENDOR_ID",
  "product": "idl",
  "context_id": "<campaign_context_id>",
  "dimensions": ["<campaign_dim_id>"],
  "metrics": [{ "id": "<metric_id>" }],
  "from": 1782864000,
  "to": 1784159999,
  "sort": [{ "field": "<metric_id>", "dir": "desc" }],
  "limit": 10
}
```

Do **not** also set `date_preset`. For ads without platform named, ask Meta vs Google before choosing `context_id`.

## Example 4 — Wrong then fix

**Bad:** invent metric id `revenue` without schema.  
**Good:** `get_insights_schema` → use returned metric `id` (e.g. `total_revenue`).

**Bad:** `date_preset: "last_7d"` and `from`/`to` together.  
**Good:** one or the other.

**Bad:** run Insights tools before org is known.  
**Good:** `list_vendors` → confirm org → Insights workflow.
