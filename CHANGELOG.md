# Changelog

All notable changes to this plugin are documented in this file. Version numbers match `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json`.

## 1.0.1

ChatGPT / Codex plugin packaging (same prod MCP and Insights skill).

- `.codex-plugin/plugin.json` for ChatGPT and Codex (skills + `.mcp.json`)
- `.agents/plugins/marketplace.json` so `codex plugin marketplace add ingestlabs/ingest-labs-mcp-plugin` can resolve this repo
- README: ChatGPT / Codex install subsection

## 1.0.0

Initial public release of **Ingest Labs MCP**.

- Production remote MCP at `https://mcp.ingestlabs.com/v1/mcp` (Streamable HTTP + portal OAuth)
- Skills for Insights analytics and Tag Manager workflows
- Claude Code marketplace package (`ingestlabs@ingestlabs`)
- Cursor plugin metadata (`.cursor-plugin/plugin.json`)
- Client-agnostic README and SETUP for ChatGPT, Claude, Cursor, Perplexity, and other MCP hosts
