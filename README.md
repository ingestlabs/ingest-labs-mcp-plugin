# Ingest Labs MCP Plugin

Claude (and Cursor) plugin for **Ingest Labs Insights / IQC** against production MCP:

`https://mcp.ingestlabs.com/v1/mcp`

Sign in with your Ingest Labs portal account (OAuth). The plugin registers the remote MCP server and a skill that maps natural-language analytics questions to Insights tools.

## Install (Claude Code)

```bash
claude plugin marketplace add ingestlabs/ingest-labs-mcp-plugin
claude plugin install ingestlabs@ingestlabs
```

Restart Claude Code and enable the plugin if prompted. Complete the portal login when OAuth starts. The MCP path is `/v1/mcp` (not `/mcp`).

See [SETUP.md](SETUP.md) for the connect → login → `ping` → Insights walkthrough.

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
