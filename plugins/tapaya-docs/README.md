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

## Skills

The plugin also adds eight skills. Each one has Claude check the current docs
through the MCP server before answering or writing code, cite the pages it
used, and say when the docs do not cover something. Code the skills help you
write reads keys from environment variables and keeps them on your server.

| Skill | What it does | Example prompt |
| --- | --- | --- |
| `answer-tapaya-questions` | Answers Tapaya questions from the docs with page links, and drafts a feature request you confirm before it is sent | "How do I refund a Hosted Checkout payment with Tapaya?" |
| `demo-tap-to-pay` | Builds an Android (Kotlin) Accept SDK demo, plus a tiny Node.js server that issues the SDK login token | "Build me a Tapaya Tap to Pay demo for my Pixel" |
| `demo-hosted-checkout` | Builds a Node.js server that creates a Hosted Checkout session, redirects to it, and verifies the payment | "Make a minimal Tapaya Hosted Checkout demo in Node" |
| `demo-embedded-checkout` | Builds a Node.js server and a page with Tapaya's embedded card fields | "Show me Tapaya Embedded Checkout card fields in a demo page" |
| `webhooks` | Configures a webhook destination and writes a receiver that verifies signatures, then tests it | "Add a verified Tapaya webhook receiver to my Express app" |
| `merchant-onboarding` | Picks an onboarding flow and walks through the Platform API calls to onboard and manage merchants | "Onboard merchants for Checkout from our own signup form" |
| `saved-cards` | Saves cards with consent and charges them later: subscriptions, trials, top-ups, holds, refunds | "Add monthly renewals with saved cards to my Tapaya checkout" |
| `go-live-check` | Reviews your integration against the QA checklists and reports pass, fail, or unknown per item | "Is my Tapaya integration ready for production?" |

The demos need a sandbox organization in the Tapaya Platform: a merchant
checkout key for the Checkout demos, or a Server Secret Token and a physical
Android device for Tap to Pay. Each skill says where to find them.

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

## Data and privacy

The plugin installs no executable code. It points your client at
`https://docs.tapaya.com/mcp` and adds eight skills, which are Markdown
instructions and templates. The docs and API tools send your search terms and
the page or endpoint you ask for to that server, and return public
documentation. `request_feature` sends the text of your feature request to the
Tapaya product team. The plugin itself uses no API key or account.

The skills send nothing. Code they help you build, such as the demo apps,
runs on your machine and calls the Tapaya sandbox API
(`https://api.sandbox.tapaya.com`) with your own sandbox key, read from a
local `.env` file that the demos keep out of version control.

- Privacy policy: https://tapaya.com/privacy
- Terms: https://tapaya.com/terms
- Support: https://docs.tapaya.com/support

## License

Apache-2.0. See the `LICENSE` file at the repository root.
