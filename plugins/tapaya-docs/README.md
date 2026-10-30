# Tapaya Docs MCP

Adds the Tapaya documentation MCP server to your AI editor. It is public: no
API key or secret is required. The server exposes five read-only tools,
`search_docs`, `get_page`, `list_pages`, `search_api_endpoints`, and
`get_api_endpoint`, backed by the Tapaya docs and REST API, plus
`request_feature`, which sends a feature request to the Tapaya product team.
Enable confirmation for `request_feature` in your client so each request is
reviewed before it is sent.

The docs cover In-Store Acceptance (Accept SDK, Tapaya Terminal) and Online
Payments (Tapaya Checkout: Hosted and Embedded Checkout, and the
`@tapayadot/checkout` Node.js SDK).

## Install (Claude Code)

```bash
claude plugin marketplace add tapayadot/tapaya-mcp
claude plugin install tapaya-docs@tapaya
```

## What it connects to

The plugin registers a single remote MCP server at
`https://docs.tapaya.com/mcp` over Streamable HTTP. A discovery manifest is
published at `https://docs.tapaya.com/.well-known/mcp.json`.

Full setup for other clients (Cursor, VS Code, Claude Desktop) is documented at
https://docs.tapaya.com/mcp-server
