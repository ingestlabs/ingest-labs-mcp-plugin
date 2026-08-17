# Ingest Labs MCP

Remote MCP for your **production** Ingest Labs account:

```text
https://mcp.ingestlabs.com/v1/mcp
```

Use Streamable HTTP. The path must be `/v1/mcp` (not `/mcp` or `/v1`). Sign in with your Ingest Labs portal account when the client starts OAuth.

Works with any MCP-capable host that can reach a remote HTTPS MCP server (for example ChatGPT, Claude, Cursor, Perplexity, and others). This repository also packages optional client plugins/skills for hosts that install from GitHub.

Today the server exposes Insights analytics and Tag Manager tools; more capabilities will follow.

## Connect (any MCP client)

1. Add a remote MCP server with URL `https://mcp.ingestlabs.com/v1/mcp`.
2. Complete the browser login (same account as [console.ingestlabs.com](https://console.ingestlabs.com)).
3. Confirm with `ping`, then `list_vendors`.

Details and troubleshooting: [SETUP.md](SETUP.md).

### Claude Code (optional plugin package)

```bash
claude plugin marketplace add ingestlabs/ingest-labs-mcp-plugin
claude plugin install ingestlabs@ingestlabs
```

Restart Claude Code, enable the plugin if prompted, then complete portal OAuth.

### Cursor (optional)

Install this repo as a local Cursor plugin, or add the same MCP URL under Cursor MCP settings (see `.mcp.json` in this repo for the expected shape, including the public OAuth client id).

### Other hosts (ChatGPT, Claude.ai connectors, Perplexity, …)

Use that product’s “custom MCP / connector” UI and paste `https://mcp.ingestlabs.com/v1/mcp`. Follow its OAuth prompts to the Ingest Labs portal. No Claude Code install is required.

## What you can ask

- Attribution / MTA, channel revenue, journeys
- Meta and Google Ads (spend, ROAS — spend is not revenue)
- GA4 traffic and landing pages
- Ingest Labs sessionized events, TQS / invalid traffic
- Tag Manager fire aggregates and recorded events (needs a project)

Always pick an organization first (`list_vendors`). Tag / media-tag questions also need a project (`list_projects`).

## Privacy

- [Privacy Policy](https://ingestlabs.com/privacy-policy/)
- [Terms of Service](https://ingestlabs.com/terms-of-service/)
- Privacy contact: [privacy@ingestlabs.com](mailto:privacy@ingestlabs.com)

The MCP server reads Ingest Labs data for the signed-in user. It does not train models on your data.

## Support

[ingestlabs.com](https://www.ingestlabs.com)

## License

[MIT](LICENSE)
