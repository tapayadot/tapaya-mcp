---
name: demo-hosted-checkout
description: Build a minimal, runnable Tapaya Hosted Checkout demo that creates a real Checkout session against the Tapaya sandbox from a small Node.js server, redirects the customer to the Tapaya payment page, and verifies the result on the server. Use when the user asks for a Hosted Checkout demo, sample, quick start, or proof of concept, or wants to try redirect-based online payments with Tapaya.
---

# Tapaya Hosted Checkout demo

Build the smallest working demo of Tapaya Hosted Checkout: a Node.js server
creates a Checkout session in the sandbox, sends the browser to the Tapaya
payment page with HTTP 303, then retrieves the session to confirm payment.
Every call is real. Nothing is mocked.

The secret key stays on the server. The browser only ever sees the session
`url` that Tapaya returns.

Main docs pages:

- Hosted Checkout guide: https://docs.tapaya.com/online/integration/hosted_checkout
- Node.js SDK reference: https://docs.tapaya.com/online/integration/node_sdk
- API Integration (fields, statuses, environments): https://docs.tapaya.com/online/integration/api_integration
- Checkout settings (keys, return URLs): https://docs.tapaya.com/online/integration/checkout_settings
- Testing (test cards): https://docs.tapaya.com/online/getting-started/testing

## 1. Confirm what you need

Ask the user, briefly, before writing files:

1. **Stack.** The template is plain Node.js (22.13.0 or newer, ES modules)
   with `@tapayadot/checkout` and no framework. If they want another server
   language, use the HTTP examples on the Hosted Checkout page instead of the
   SDK (`POST /merchant/checkout-sessions`, `GET /merchant/checkout-sessions/{id}`,
   `Authorization: Bearer <key>`, required `Idempotency-Key`).
2. **A sandbox merchant that is Ready.** In the Tapaya Platform, open
   **Checkout** (https://platform.tapaya.com/checkout-settings). The merchant
   must show **Ready**: active, approved for online payments, gateway
   configured. Work in a sandbox organization so no real money moves.
   See https://docs.tapaya.com/online/integration/checkout_settings#onboard-a-merchant-for-checkout.
3. **A sandbox merchant checkout key.** Open the merchant's **Settings**,
   find **Merchant checkout keys**, enter a name, select **Create merchant
   key**, and copy it at once (it is shown only once). Sandbox keys start
   with `tapaya_test_`. See
   https://docs.tapaya.com/online/integration/checkout_settings#create-merchant-checkout-keys.
4. **Return URLs allowed.** Under the merchant's **Merchant API key redirects**,
   add `http://localhost:4242` to **Allowed websites** (HTTP is accepted for
   local hosts; the port must match). The demo passes its own `successUrl`
   and `cancelUrl`, which must match an allowed website.
   See https://docs.tapaya.com/online/integration/checkout_settings#configure-return-destinations.

Never ask the user to paste the key into the chat. They put it in `.env`
themselves. If a key starting with `tapaya_live_` shows up anywhere, stop and
tell them to use a sandbox key.

## 2. Check the docs before writing code

The plugin ships the `tapaya-docs` MCP server. Use it to confirm the current
details, because they change:

- `get_page` with `/online/integration/hosted_checkout` and
  `/online/integration/node_sdk`: SDK version, `new Tapaya()` options,
  `checkout.sessions.create()` and `get()` parameters and return fields,
  error classes.
- `get_page` with `/online/getting-started/testing`: current test cards.
- `get_page` with `/online/integration/api_integration`: `status` and
  `paymentStatus` values, the sandbox API URL.
- `search_docs` for anything else, such as `"return URL"` or `"idempotency"`.

Checkout session calls are documented in these guide pages, not in the
OpenAPI spec, so prefer `search_docs` and `get_page` over
`search_api_endpoints` here. If the docs differ from the template, follow the
docs and tell the user what changed.

## 3. Scaffold the files

Create a new folder (default `tapaya-hosted-checkout-demo/`) with the files in
[references/server.md](references/server.md):

- `package.json` with one dependency, `@tapayadot/checkout`, pinned to the
  version the Node.js SDK page documents (0.7.0 when this skill was written),
  and `"start": "node --env-file=.env server.mjs"`. The SDK does not load
  `.env` files itself.
- `.env.example` with the secret line commented out:

  ```bash
  TAPAYA_ENVIRONMENT=sandbox
  # TAPAYA_SECRET_KEY=tapaya_test_xxx  # paste your sandbox merchant checkout key
  PORT=4242
  ```

- `.gitignore` that ignores `.env` and `node_modules/`.
- `server.mjs`, which:
  1. Builds the client with `new Tapaya()`. It reads `TAPAYA_SECRET_KEY` and
     `TAPAYA_ENVIRONMENT`.
  2. On `POST /checkout`, stores a new order with a unique `merchantOrderId`
     and idempotency key **before** calling Tapaya, then calls
     `tapaya.checkout.sessions.create({ merchantOrderId, amount, currency, successUrl, cancelUrl }, { idempotencyKey })`.
     Amounts are integers in minor units (`2450` with `EUR` is EUR 24.50).
  3. Saves `session.id` with the order, then redirects to `session.url`
     unchanged with HTTP 303.
  4. On both the success and cancel URL, calls
     `tapaya.checkout.sessions.get(order.checkoutSessionId)` and treats the
     order as paid only when `id`, `merchantOrderId`, `amount`, and `currency`
     match and `paymentStatus === 'successful'`. A redirect alone proves
     nothing.

Keep it minimal. Do not add a framework, a database, or a bundler unless the
user asks.

## 4. Run and test

Tell the user:

```bash
cd tapaya-hosted-checkout-demo
npm install
cp .env.example .env   # then uncomment TAPAYA_SECRET_KEY and paste the sandbox key
npm start
```

Open http://localhost:4242, select **Pay with Tapaya**, and pay on the
Tapaya page (sandbox pages open on `checkout.sandbox.tapaya.com`). Use the test
cards from https://docs.tapaya.com/online/getting-started/testing#test-cards,
with any future expiry date and CVC `123`. At the time of writing:

| Card | Result |
| --- | --- |
| `4242 4242 4242 4242` | Payment succeeds |
| `4916 0184 7581 4056` | Declined; the session stays `open` and the customer can retry |
| `4012 0018 0000 0016` | 3D Secure challenge; complete it, then the server confirms success |

Re-check the table with `get_page` before quoting it. Each run uses a new
`merchantOrderId`, because an order can be paid only once.

## 5. Expected result and common errors

Expected: after paying with `4242 4242 4242 4242`, the browser returns to
`/checkout/return` and the page shows `paymentStatus: "successful"`,
`matchesOrder: true`, and **Paid**. The server log prints the session ID
(`cs_…`). Selecting cancel on the Tapaya page lands on `/checkout/cancel`,
which still checks the session and shows **Not paid**.

| Symptom | Likely cause and fix |
| --- | --- |
| Server exits at start with a missing key error | `TAPAYA_SECRET_KEY` is not set. Uncomment it in `.env` and start with `npm start` (which passes `--env-file`). |
| `TapayaError` at start about key and environment | A `tapaya_live_` key with `TAPAYA_ENVIRONMENT=sandbox`, or the reverse. Use a sandbox `tapaya_test_` key. |
| HTTP 401 (`TapayaAuthenticationError`) | Wrong, revoked, or production key. Create a new merchant checkout key in the sandbox organization. |
| HTTP 400/422 (`TapayaInvalidRequestError`) | Read `fieldErrors`. Common causes: return URL not under **Allowed websites** (origin and port must match exactly), currency not enabled for the merchant, merchant not Ready. |
| HTTP 409 (`TapayaConflictError`) | Same idempotency key with a different body (`API-0025`), or the order is already paid. Use a new order. |
| Page shows `unpaid` after paying | Tapaya may still be confirming. Refresh; the demo retrieves the session again. |
| `TapayaConnectionError` | Outcome unknown. Retry with the same idempotency key and body, never a new key. |

Include `requestId` from the error when the user contacts Tapaya support
(https://docs.tapaya.com/support/contacts).

## What the docs do not cover

Say so instead of guessing:

- The docs do not state that Tapaya adds the session ID to the success or
  cancel URL, so the demo tracks the order with its own cookie.
- The docs do not describe how to create or switch to a sandbox organization
  in the Platform. They only say sandbox keys start with `tapaya_test_` and
  sandbox calls go to `https://api.sandbox.tapaya.com`.
- Webhooks (`/webhooks/events#checkout-sessions`) and a recovery job for
  customers who never return are required in production but left out of the
  demo.
