![Tapaya Docs MCP](banner-github.png)

![MCP](https://img.shields.io/badge/MCP-Streamable%20HTTP-blue) ![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2) ![Access](https://img.shields.io/badge/access-read--only-green) [![Docs](https://img.shields.io/badge/docs-tapaya.com-blue)](https://docs.tapaya.com) [![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)

Model Context Protocol server for the Tapaya documentation and REST API. Connects any MCP client (Claude Code, Cursor, VS Code, Claude Desktop) to search the docs and explore the API without leaving your editor.

## Install

Claude Code:

```bash
claude plugin marketplace add tapayadot/tapaya-mcp
claude plugin install tapaya-docs@tapaya
```

Or add the server directly without the plugin:

```bash
claude mcp add --transport http tapaya-docs https://docs.tapaya.com/mcp
```

Then check it is connected:

```bash
claude mcp list
```

The server is public and read-only. No API key or secret is required.

## Requirements

- An MCP client that supports remote Streamable HTTP servers.
- Network access to `https://docs.tapaya.com/mcp`.

There is no local runtime and no build step. The plugin only registers a remote server.

## Other clients

Cursor (`~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "tapaya-docs": {
      "url": "https://docs.tapaya.com/mcp"
    }
  }
}
```

VS Code (`.vscode/mcp.json`):

```json
{
  "servers": {
    "tapaya-docs": {
      "type": "http",
      "url": "https://docs.tapaya.com/mcp"
    }
  }
}
```

Claude Desktop (MCP config):

```json
{
  "mcpServers": {
    "tapaya-docs": {
      "url": "https://docs.tapaya.com/mcp"
    }
  }
}
```

Any other Streamable HTTP client: point it at `https://docs.tapaya.com/mcp`. You can list the available tools with a raw request:

```bash
curl -s https://docs.tapaya.com/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

One-click "Install in Cursor" and "Install in VS Code" links are on the docs page: [docs.tapaya.com/mcp-server](https://docs.tapaya.com/mcp-server).

## Tools

The server exposes five read-only tools.

| Tool | Purpose |
| --- | --- |
| `search_docs` | Full-text search across all documentation pages |
| `get_page` | Fetch the full Markdown of a page by its url |
| `list_pages` | List every page as title, url, and description |
| `search_api_endpoints` | Search the Tapaya REST API by keyword |
| `get_api_endpoint` | Full parameters and responses for one endpoint |

## What it connects to

A single remote MCP server at `https://docs.tapaya.com/mcp` over Streamable HTTP. Nothing runs locally; the plugin points your client at that endpoint.

A discovery manifest is also published at [`https://docs.tapaya.com/.well-known/mcp.json`](https://docs.tapaya.com/.well-known/mcp.json) for clients that support well-known MCP discovery.

## Notes and safety

- Read-only: the server never calls the live Tapaya API and never asks for keys or secrets.
- When combining multiple MCP servers, use the usual caution around prompt injection from tool output, and enable human confirmation for tools where your client supports it.

## Repository layout

- `.claude-plugin/marketplace.json` defines the `tapaya` marketplace.
- `plugins/tapaya-docs/` is the plugin: a `.mcp.json` declaring the remote Streamable HTTP server, plus its manifest and README.

## Documentation

Full documentation is available at [docs.tapaya.com](https://docs.tapaya.com). MCP setup lives at [docs.tapaya.com/mcp-server](https://docs.tapaya.com/mcp-server).

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
