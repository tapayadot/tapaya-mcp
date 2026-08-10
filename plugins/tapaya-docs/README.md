# Tapaya Docs MCP

Adds the Tapaya documentation MCP server to your AI editor. It is read-only and
public: no API key or secret is required. The server exposes five tools:
`search_docs`, `get_page`, `list_pages`, `search_api_endpoints`, and
`get_api_endpoint`, backed by the Tapaya docs and REST API.

## Install (Claude Code)

```bash
claude plugin marketplace add tapayadot/tapaya-mcp
claude plugin install tapaya-docs@tapaya
```

## What it connects to

The plugin registers a single remote MCP server at
`https://docs.tapaya.com/mcp` over Streamable HTTP.

Full setup for other clients (Cursor, VS Code, Claude Desktop) is documented at
https://docs.tapaya.com/mcp-server
