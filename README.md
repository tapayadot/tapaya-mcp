# Tapaya MCP

A Claude Code plugin marketplace for Tapaya's AI tooling.

This repository hosts the `tapaya` marketplace and its plugins. The current
plugin, `tapaya-docs`, connects any MCP client to the read-only Tapaya
documentation MCP server at `https://docs.tapaya.com/mcp`, giving agents five
tools to search the docs and explore the REST API.

## Install (Claude Code)

```bash
claude plugin marketplace add tapayadot/tapaya-mcp
claude plugin install tapaya-docs@tapaya
```

The server is public and read-only; no API key is required.

## Other clients

Cursor, VS Code, and Claude Desktop setup (including one-click install links)
is documented at https://docs.tapaya.com/mcp-server

## Contents

- `.claude-plugin/marketplace.json` defines the `tapaya` marketplace.
- `plugins/tapaya-docs/` is the plugin: a `.mcp.json` declaring the remote
  Streamable HTTP MCP server, plus its manifest and README.
