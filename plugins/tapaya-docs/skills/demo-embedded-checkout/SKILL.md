---
name: demo-embedded-checkout
description: Build a minimal, runnable Tapaya Embedded Checkout demo that mounts Tapaya's secure card fields in your own page, prepares the order's payment from a small Node.js server with the @tapayadot/checkout SDK against the real Tapaya sandbox, and verifies the payment on the server. Use when the user asks for an Embedded Checkout demo, sample, or proof of concept, or wants to try Tapaya card fields inside their own checkout page.
---

# Tapaya Embedded Checkout demo

Build the smallest working demo of Tapaya Embedded Checkout: a page that
mounts Tapaya's card fields, a Node.js server that answers the fields'
`prepare` request with the order's payment, and a confirmation page that
verifies the payment on the server. Every call goes to the Tapaya sandbox.
Nothing is mocked.

The secret merchant key stays on the server. The browser receives only the
publishable key, the API origin, and a per-payment client secret. Card data
stays inside Tapaya's frame.

Main docs pages:

- Embedded Checkout guide: https://docs.tapaya.com/online/integration/embedded_checkout
- Node.js SDK, Embedded Checkout section: https://docs.tapaya.com/online/integration/node_sdk#embedded-checkout
- Checkout settings (merchant readiness, keys): https://docs.tapaya.com/online/integration/checkout_settings
- Testing (test cards): https://docs.tapaya.com/online/getting-started/testing

## 1. Confirm what you need

Ask the user, briefly, before writing files:

1. **Stack.** The template is a plain Node.js server (22.13.0 or newer) and a
   plain JavaScript page bundled with esbuild. One package,
   `@tapayadot/checkout`, holds the server client (`@tapayadot/checkout`), the
   browser loader (`@tapayadot/checkout/client`), and the React hook
   (`@tapayadot/checkout/react`). If the user already has a framework and
   bundler, put the server code in their routes and the browser code in their
   page instead.
2. **A sandbox merchant that is Ready.** In the Tapaya Platform, open
   **Checkout** (https://platform.tapaya.com/checkout-settings) and check the
   merchant shows **Ready**. Work in a sandbox organization so no real money
   moves. See https://docs.tapaya.com/online/integration/checkout_settings#onboard-a-merchant-for-checkout.
3. **A sandbox merchant checkout key.** Open the merchant's **Settings**, find
   **Merchant checkout keys**, select **Create merchant key**, and copy it
   right away (shown once). Sandbox keys start with `tapaya_test_`. See
   https://docs.tapaya.com/online/integration/checkout_settings#create-merchant-checkout-keys.

Embedded card payments do not use return URLs, so no Allowed websites setup is
needed. Never ask the user to paste the key into the chat; they put it in
`.env`. If a `tapaya_live_` key appears, stop and ask for a sandbox key.

## 2. Check the docs before writing code

The plugin ships the `tapaya-docs` MCP server. Confirm the current details
with it before writing code:

- `get_page` `/online/integration/embedded_checkout`: package entries,
  `loadTapaya(publishableKey, { apiOrigin })`, `createPayment({ prepare, onChange })`,
  `mount()`, `submit()`, `snapshot.outcome`, the prepare endpoint rules, and
  the CSP origins.
- `get_page` `/online/integration/node_sdk`: SDK version,
  `checkout.configuration()`, `checkout.sessions.prepare()` and `verify()`
  parameters and return fields, `PaymentsApiError`,
  `TapayaOrderMismatchError`.
- `get_page` `/online/getting-started/testing`: current test cards.
- `search_docs` for anything else, such as `"client secret"` or `"wallets"`.

If the docs differ from the templates, follow the docs and tell the user what
changed.

## 3. Scaffold the files

Create a folder (default `tapaya-embedded-checkout-demo/`) with the files in
[references/server.md](references/server.md) and
[references/browser.md](references/browser.md):

- `package.json`: `@tapayadot/checkout` pinned to the version the Node.js SDK
  page documents (0.7.0 when this skill was written), `esbuild` as a dev
  dependency, `build` and `start` scripts. `start` runs
  `node --env-file=.env server.mjs`, because the SDK does not load `.env`.
- `.env.example` with the secret line commented out:

  ```bash
  TAPAYA_ENVIRONMENT=sandbox
  # TAPAYA_SECRET_KEY=tapaya_test_xxx  # paste your sandbox merchant checkout key
  PORT=4243
  ```

- `.gitignore` for `.env`, `node_modules/`, and the generated `public/`.
- `server.mjs` with these routes:
  - `GET /` creates an order in memory (unique `merchantOrderId`, amount in
    minor units, uppercase currency), sets an HttpOnly cookie, and renders the
    form with `<div id="card-fields">` and a disabled Pay button.
  - `GET /api/config` returns `{ publishableKey, apiOrigin }` from
    `tapaya.checkout.configuration()`. Both are safe for the browser.
  - `POST /api/payments` is the `prepare` endpoint. It finds the order from
    the cookie, then calls `tapaya.checkout.sessions.prepare(details)` while
    the order awaits payment, or `verify(details)` otherwise, and returns
    `{ clientSecret, status }` with `Cache-Control: no-store`. It sends
    `clientSecret: null` once the order no longer accepts payment.
  - `GET /checkout/confirmation` calls `verify()` and records the order as
    paid only when `paymentStatus === 'successful'`.
- `client/checkout.js`: `loadTapaya`, `createPayment({ prepare: '/api/payments', onChange })`,
  `payment.mount('#card-fields')`, and a submit handler that calls
  `payment.submit()`.

Do not log or store the client secret. Do not add wallets unless asked: Apple
Pay needs a registered shop domain, which a localhost demo cannot have.

## 4. Run and test

Tell the user:

```bash
cd tapaya-embedded-checkout-demo
npm install
cp .env.example .env   # then uncomment TAPAYA_SECRET_KEY and paste the sandbox key
npm start              # builds public/checkout.js, then starts the server
```

Open http://localhost:4243. The card fields load inside Tapaya's frame. Pay
with a test card from https://docs.tapaya.com/online/getting-started/testing#test-cards,
any future expiry date, and CVC `123`. At the time of writing:

| Card | Result |
| --- | --- |
| `4242 4242 4242 4242` | Payment succeeds |
| `4916 0184 7581 4056` | Declined; the fields stay open and the shopper can try another card |
| `4012 0018 0000 0016` | 3D Secure challenge; complete it, then the server confirms success |

Re-check the table with `get_page` before quoting it. Reload `/` after a paid
order to get a new one, because each order can be paid only once.

## 5. Expected result and common errors

Expected: the Pay button enables once the card is complete. After paying with
`4242 4242 4242 4242`, `onChange` reports `outcome: 'completed'`, the browser
opens `/checkout/confirmation`, and the server shows **Paid** with
`paymentStatus: "successful"`. On mount the server log prints a `prepare <order> -> <payment id> open …`
line. A reload during a payment shows its current state; it never charges
twice.

| Symptom | Likely cause and fix |
| --- | --- |
| Server exits at start with a missing key error | `TAPAYA_SECRET_KEY` not set. Uncomment it in `.env` and use `npm start`. |
| `/api/config` returns 502 with HTTP 401 | Wrong, revoked, or production key. Create a sandbox merchant checkout key. |
| `/checkout.js` returns 500 | The bundle is missing. Run `npm run build`. |
| Card fields never appear; `PaymentError` code `load` | Tapaya.js could not load from `https://js.tapaya.com`. Check the network, ad blockers, and any CSP (`script-src` and `frame-src https://js.tapaya.com`). |
| `TapayaValidationError` on prepare | Lowercase currency or a bad order ID. Embedded calls need uppercase ISO 4217 and an ID of 1 to 200 characters. |
| `TapayaOrderMismatchError` | The order's amount or currency changed after payment started. Freeze them, or start a new order. |
| `PaymentsApiError` with another status | Read `status`, `code`, and `fieldErrors` on the server. A merchant that is not Ready or a currency the merchant cannot take are common causes. |
| Outcome stays `pending` | The result is not known yet. The SDK keeps checking; the server's `verify()` is the source of truth. |

Browser error codes (`card_invalid`, `authentication`, `tokenization`, and
others) are listed under "Handle errors" on the Embedded Checkout page.
Include `requestId` from server errors when contacting Tapaya support
(https://docs.tapaya.com/support/contacts).

## What the docs do not cover

Say so instead of guessing:

- No plain `<script>` URL for Tapaya.js is documented, only the
  `@tapayadot/checkout/client` import. The demo bundles it with esbuild, which
  is a tooling choice, not a Tapaya requirement.
- The docs do not describe how to create or switch to a sandbox organization
  in the Platform. They say sandbox keys start with `tapaya_test_` and
  sandbox calls go to `https://api.sandbox.tapaya.com`.
- Wallet testing (Apple Pay domain registration, Google Pay) and the
  scheduled job that settles abandoned orders are documented but left out of
  this demo.
