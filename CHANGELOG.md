# Changelog

All notable changes to this plugin are documented in this file. Version numbers match `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json`.

## 1.0.3

Bundle a Codex-native MCP server so install does not require a manual `config.toml` entry.

- Add `.codex-plugin/mcp.json` with Streamable HTTP `url` + `auth: oauth` (Codex ignores Claude's `type`/`oauth.clientId` shape in root `.mcp.json`)
- Point `.codex-plugin/plugin.json` `mcpServers` at that file
- Root `.mcp.json` unchanged for Claude Code / Cursor

## 1.0.2

Ship the Ingest Labs logo as a bundled Codex/ChatGPT asset.

- `assets/logo.png` (400×400 PNG)
- `.codex-plugin/plugin.json` `interface.logo` and `composerIcon` point at `./assets/logo.png`

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
