---
name: answer-tapaya-questions
description: Answer any question about Tapaya (In-Store Acceptance, Accept SDK, Tapaya Terminal, Cloud Accept API, Online Payments, Hosted and Embedded Checkout, the @tapayadot/checkout SDK, Platform API, merchants, payouts, webhooks, compliance) from the official docs through the tapaya-docs MCP tools, with cited page URLs, a clear note when the docs do not cover something, and an optional feature request sent only after the user confirms. Use whenever the user asks how Tapaya works, which endpoint, field, SDK method, or setting to use, or what an error or status means.
---

# Answer Tapaya questions from the docs

Answer from the official Tapaya docs, through the `tapaya-docs` MCP server
this plugin ships, never from memory. Every fact gets a docs URL. When the
docs are silent, say so.

The server exposes `search_docs`, `get_page`, `list_pages`,
`search_api_endpoints`, `get_api_endpoint`, and `request_feature`. Setup and
tool list: https://docs.tapaya.com/mcp-server

## 1. Find the page

1. `search_docs` with the user's key terms (product, feature, field, error
   code). Results are fragments with a `url`; pick the most relevant pages.
2. `get_page` with that exact `url` to read the full page. Read the whole
   section you quote, including callouts and tables; rules often sit there.
3. If search misses, `list_pages` and browse by title. The docs are split
   into `/in-store/…`, `/online/…`, `/platform-api/…`, `/merchants/…`,
   `/webhooks/…`, `/financial-guide/…`, `/compliance/…`, and `/support/…`.

## 2. Get the exact API shape

- `search_api_endpoints` with keywords, then `get_api_endpoint` with the
  exact method and path for parameters, request body, responses, and
  security. Use it for Platform API endpoints (merchants, onboarding,
  reporting, refunds, webhooks configuration, SDK login token).
- **Checkout is different.** Checkout session and payment calls
  (`POST /merchant/checkout-sessions`, `GET /merchant/checkout-sessions/{id}`,
  `/merchant/payments…`, `/merchant/saved-payment-methods…`) are documented
  in the guide pages, mainly `/online/integration/api_integration` and
  `/online/integration/node_sdk`. Read those with `search_docs` and
  `get_page` rather than relying on `search_api_endpoints`.
- Numeric enums (`countryId`, `currencyId`, `businessTypeId`,
  `documentType`, `paymentProcessorId`) are listed in the Swagger docs, not
  the guides. Get them from `get_api_endpoint`; never guess a number.

## 3. Answer

- Lead with the direct answer, then the details the user needs.
- Cite the page for each fact as a full link, for example
  `https://docs.tapaya.com/online/integration/hosted_checkout#verify-the-payment`.
  Use the `#anchor` from the page heading when it helps.
- Quote field names, values, and limits exactly as the docs write them.
- Mention the environment when it matters: sandbox
  `https://api.sandbox.tapaya.com`, production `https://api.tapaya.com`.
- Note authentication differences when you show requests: Checkout uses
  `Authorization: Bearer <key>`; other Platform API calls take the Server
  Secret Token as-is.
- In code examples, read keys from environment variables. Never put a real
  key in an example, and never ask the user to paste one into the chat.

## 4. When the docs do not cover it

Say so plainly: "The Tapaya docs do not document X." Name the pages you
checked. Do not fill the gap with a guess, another provider's behavior, or
an invented field. Then offer the next step:

- For bugs or integration help: developer support,
  https://docs.tapaya.com/support/contacts
- For security issues: https://docs.tapaya.com/support/contacts#security-issues
- For missing functionality: a feature request (below).

## 5. Feature requests

`request_feature` posts a message to the Tapaya product team. Use it only for
something the docs show is missing (an endpoint, SDK method, payment method,
or docs page), and only with the user's consent:

1. Draft the request: `title` (3 to 120 characters), `details` (what they
   are building, where they are stuck, what they do instead; 10 to 4000
   characters), `product` (`in-store`, `online`, `platform`, `docs`, or
   `other`), `agent` (`Claude Code`), and optional `page`.
2. Show the draft to the user and ask whether to send it. Include `email`
   only if they agree to be contacted.
3. Send it only after they confirm. Never include API keys, secrets, or
   customer data.

See https://docs.tapaya.com/support/feature_request.

## 6. Never

- Never call the live Tapaya API, or any Tapaya API, to answer a question.
  The MCP tools read documentation only (except `request_feature`).
- Never present remembered or inferred details as documented facts.
- Never reveal or repeat a secret the user shares by mistake; tell them to
  revoke and replace it.

## Related skills

For hands-on work, hand off to the plugin's other skills: `demo-tap-to-pay`,
`demo-hosted-checkout`, `demo-embedded-checkout`, `webhooks`,
`merchant-onboarding`, `saved-cards`, and `go-live-check`.
