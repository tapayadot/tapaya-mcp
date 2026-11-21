# Hosted Checkout demo: file templates

Templates for the `demo-hosted-checkout` skill. Before you copy them, check every
Tapaya call against the live docs with the `tapaya-docs` MCP tools. The docs
win if they differ from this file.

Sources:

- https://docs.tapaya.com/online/integration/hosted_checkout
- https://docs.tapaya.com/online/integration/node_sdk (`checkout.sessions.create()`, `get()`, errors)
- https://docs.tapaya.com/online/integration/api_integration (fields, statuses, environments)
- https://docs.tapaya.com/online/getting-started/testing (test cards)

The demo has no framework and one runtime dependency, `@tapayadot/checkout`.
It uses the built-in `node:http` server and Node's `--env-file` flag, because
the SDK does not load `.env` files itself.

## File tree

```text
tapaya-hosted-checkout-demo/
├── .env.example
├── .gitignore
├── package.json
└── server.mjs
```

## package.json

Node.js 22.13.0 or newer is required by the SDK. Pin the SDK version you
checked in the docs (the docs page states which version it documents).

```json
{
  "name": "tapaya-hosted-checkout-demo",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22.13.0" },
  "scripts": {
    "start": "node --env-file=.env server.mjs"
  },
  "dependencies": {
    "@tapayadot/checkout": "0.7.0"
  }
}
```

## .env.example

Copy to `.env`, then uncomment and fill in the secret line. Never commit `.env`.

```bash
# Sandbox only. The SDK reads both variables.
TAPAYA_ENVIRONMENT=sandbox
# TAPAYA_SECRET_KEY=tapaya_test_xxx  # paste your sandbox merchant checkout key (Platform > Checkout > merchant Settings > Create merchant key)

# Demo settings
PORT=4242
DEMO_CURRENCY=EUR
DEMO_AMOUNT=2450
```

## .gitignore

```text
node_modules/
.env
```

## server.mjs

```js
import { createServer } from 'node:http'
import { randomUUID } from 'node:crypto'
import { Tapaya, TapayaApiError, TapayaConnectionError, TapayaError } from '@tapayadot/checkout'

const PORT = Number(process.env.PORT ?? 4242)
const BASE_URL = `http://localhost:${PORT}` // must be an Allowed website for your key
const CURRENCY = (process.env.DEMO_CURRENCY ?? 'EUR').toUpperCase()
const AMOUNT = Number(process.env.DEMO_AMOUNT ?? 2450) // minor units: 2450 = 24.50

// Reads TAPAYA_SECRET_KEY and TAPAYA_ENVIRONMENT from the environment.
// Throws at startup if the key is missing.
const tapaya = new Tapaya()

// Demo only: orders live in memory and disappear on restart.
const orders = new Map()

const page = (title, body) => `<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>${title}</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>body{font-family:system-ui,sans-serif;max-width:32rem;margin:3rem auto;padding:0 1rem}
pre{background:#f4f4f5;padding:1rem;overflow:auto}button{font-size:1rem;padding:.6rem 1.2rem}</style>
</head><body>${body}</body></html>`

const escape = (value) => String(value).replace(/[&<>"']/g, (c) => `&#${c.charCodeAt(0)};`)
const money = (amount, currency) => `${currency} ${(amount / 100).toFixed(2)}`

function send(res, status, html, headers = {}) {
  res.writeHead(status, { 'Content-Type': 'text/html; charset=utf-8', 'Cache-Control': 'no-store', ...headers })
  res.end(html)
}

function orderFromCookie(req) {
  const match = /(?:^|;\s*)demo_order=([^;]+)/.exec(req.headers.cookie ?? '')
  return match ? orders.get(decodeURIComponent(match[1])) : undefined
}

function describeError(error) {
  if (error instanceof TapayaApiError) {
    return `HTTP ${error.status} ${error.code ?? ''}: ${error.message}\nrequestId: ${error.requestId ?? 'n/a'}\n` +
      JSON.stringify(error.fieldErrors, null, 2)
  }
  if (error instanceof TapayaConnectionError) return `Connection error, outcome unknown: ${error.message}`
  if (error instanceof TapayaError) return `${error.name}: ${error.message}`
  return String(error)
}

// GET / : a one-item "shop" with a Pay button.
function home(res) {
  send(res, 200, page('Tapaya Hosted Checkout demo', `
    <h1>Demo shop</h1>
    <p>One demo item: <strong>${money(AMOUNT, CURRENCY)}</strong></p>
    <form method="post" action="/checkout"><button type="submit">Pay with Tapaya</button></form>
    <p><small>Sandbox only. Pay with a test card such as 4242 4242 4242 4242, any future expiry, CVC 123.</small></p>`))
}

// POST /checkout : persist the order, create the session, redirect with 303.
async function startCheckout(res) {
  const id = `demo-${Date.now()}-${randomUUID().slice(0, 8)}`
  const order = { id, amount: AMOUNT, currency: CURRENCY, checkoutAttemptKey: `${id}:attempt-1`, checkoutSessionId: null }
  orders.set(id, order) // persist BEFORE calling Tapaya

  try {
    const session = await tapaya.checkout.sessions.create(
      {
        merchantOrderId: order.id,
        amount: order.amount,
        currency: order.currency,
        items: [{ reference: 'demo-item', name: 'Demo item', description: 'Tapaya sandbox demo', quantity: 1, unitAmount: order.amount }],
        successUrl: `${BASE_URL}/checkout/return`,
        cancelUrl: `${BASE_URL}/checkout/cancel`,
      },
      { idempotencyKey: order.checkoutAttemptKey },
    )
    order.checkoutSessionId = session.id // persist session.id before redirecting
    console.log('Created session', session.id, 'for', order.id)
    res.writeHead(303, {
      Location: session.url, // use unchanged
      'Set-Cookie': `demo_order=${encodeURIComponent(order.id)}; Path=/; HttpOnly; SameSite=Lax`,
    })
    res.end()
  } catch (error) {
    console.error(error)
    send(res, 502, page('Checkout failed', `<h1>Could not create the session</h1><pre>${escape(describeError(error))}</pre><p><a href="/">Back</a></p>`))
  }
}

// GET /checkout/return and /checkout/cancel : never trust the redirect, retrieve the session.
async function checkResult(req, res, landedOn) {
  const order = orderFromCookie(req)
  if (!order?.checkoutSessionId) return send(res, 404, page('No order', '<p>No demo order found. <a href="/">Start again</a>.</p>'))

  try {
    const current = await tapaya.checkout.sessions.get(order.checkoutSessionId)
    const matches =
      current.id === order.checkoutSessionId &&
      current.merchantOrderId === order.id &&
      current.amount === order.amount &&
      current.currency === order.currency
    const paid = matches && current.paymentStatus === 'successful'
    if (paid && !order.fulfilledAt) order.fulfilledAt = new Date().toISOString() // fulfill once

    send(res, 200, page('Payment result', `
      <h1>${paid ? 'Paid' : 'Not paid'}</h1>
      <p>You landed on the <code>${landedOn}</code> URL. The result below comes from the server, not the redirect.</p>
      <pre>${escape(JSON.stringify({
        orderId: order.id,
        sessionId: current.id,
        matchesOrder: matches,
        status: current.status,
        paymentStatus: current.paymentStatus,
        amount: money(current.amount, current.currency),
        completedAt: current.completedAt,
      }, null, 2))}</pre>
      ${paid ? '' : '<p>If the status is <code>unpaid</code>, refresh this page in a moment.</p>'}
      <p><a href="/">New order</a></p>`))
  } catch (error) {
    console.error(error)
    send(res, 502, page('Status check failed', `<h1>Could not retrieve the session</h1><pre>${escape(describeError(error))}</pre>`))
  }
}

createServer(async (req, res) => {
  const { pathname } = new URL(req.url, BASE_URL)
  if (req.method === 'GET' && pathname === '/') return home(res)
  if (req.method === 'POST' && pathname === '/checkout') return startCheckout(res)
  if (req.method === 'GET' && pathname === '/checkout/return') return checkResult(req, res, 'success')
  if (req.method === 'GET' && pathname === '/checkout/cancel') return checkResult(req, res, 'cancel')
  send(res, 404, page('Not found', '<p>Not found</p>'))
}).listen(PORT, '127.0.0.1', () => {
  console.log(`Hosted Checkout demo on ${BASE_URL} (environment: ${process.env.TAPAYA_ENVIRONMENT ?? 'from key'})`)
})
```

## Notes on the template

- `successUrl` and `cancelUrl` use `http://localhost:<PORT>`. The Checkout
  settings page says HTTP is accepted for local hosts, and the SDK accepts
  `http://localhost` outside `NODE_ENV=production`. Add
  `http://localhost:4242` (with the port) to **Allowed websites** under the
  merchant's **Merchant API key redirects**, or creation is rejected.
- The docs do not say that Tapaya appends the session ID to the return URL, so
  the demo finds the order through its own cookie.
- `items` is display only. It does not change `amount`.
- A real shop needs a database, an authenticated checkout context, webhooks
  (`/webhooks/events#checkout-sessions`), and a job that re-checks unresolved
  sessions. The demo skips all of these.
