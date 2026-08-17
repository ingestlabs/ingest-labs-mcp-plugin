# Setup — Ingest Labs MCP

Use this when the plugin is installed but tools fail, OAuth is stuck, or Insights answers look empty.

## 1. Confirm the MCP URL

Production only:

```json
{
  "ingestlabs": {
    "type": "http",
    "url": "https://mcp.ingestlabs.com/v1/mcp"
  }
}
```

The path must be `/v1/mcp`. A URL that ends at `/v1` fails resource-identifier checks.

## 2. Sign in (OAuth)

When Claude or Cursor prompts for authentication:

1. Open the browser login (Ingest Labs portal).
2. Sign in with the same account you use on [console.ingestlabs.com](https://console.ingestlabs.com).
3. Return to the client and wait for the token exchange to finish.
4. Do not run `mcp_auth` repeatedly if login already failed — check portal access and retry once.

You need an Ingest Labs organization the account is allowed to see. Ask your admin if `list_vendors` returns nothing.

## 3. Ready check

1. Call **`ping`**. It must succeed.
2. Call **`list_vendors`**. If more than one org, ask which to use. Do not invent `vendor_id`.
3. For Tag Manager or `media_tags` Insights, call **`list_projects`** and confirm `project_id`.

## 4. Insights questions

Always, after scope is known:

1. `list_insights_contexts`
2. `get_insights_schema`
3. `execute_insights_query`

Do not invent dimension or metric ids. Prefer `date_preset` (`today` | `yesterday` | `last_7d` | `last_30d`) unless the user gave an absolute range.

## 5. If something fails

| Symptom | What to try |
| --- | --- |
| `needsAuth` / 401 | Complete portal OAuth; confirm the account can open the console |
| Protected resource does not match | Client URL must be exactly `https://mcp.ingestlabs.com/v1/mcp` |
| Empty vendors | Account has no org access — use a different user or ask an admin |
| Insights error on metric / aggregate | Re-run `get_insights_schema` and use returned ids only |
| Tag / media_tags errors | Pass `project_id` from `list_projects` |
