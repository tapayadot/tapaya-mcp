# Embedded Checkout demo: browser template

The browser half of the `demo-embedded-checkout` skill. The server is in
[server.md](server.md). Check the calls against the live docs first:

- https://docs.tapaya.com/online/integration/embedded_checkout#add-the-card-fields
- https://docs.tapaya.com/online/integration/embedded_checkout#submit-from-your-handler
- https://docs.tapaya.com/online/integration/embedded_checkout#show-the-outcome

## Why a bundler

The docs load the card fields with `loadTapaya` from the
`@tapayadot/checkout/client` package entry, which must be bundled for the
browser. The docs do not publish a plain `<script>` URL for Tapaya.js, so do
not hard-code one. The loader itself fetches Tapaya.js from
`https://js.tapaya.com`. The demo uses `esbuild` only to turn
`client/checkout.js` into `public/checkout.js`; any bundler works (Vite,
webpack, Next.js). If the user's app already has one, use it and skip esbuild.

The browser receives only the publishable key (`pk_test_…` in sandbox), the
API origin, and a per-payment client secret from `POST /api/payments`. The
secret merchant key never leaves the server.

## client/checkout.js

```js
import { loadTapaya } from '@tapayadot/checkout/client'

const form = document.querySelector('#checkout')
const button = form.querySelector('button')
const message = document.querySelector('#message')

// { publishableKey, apiOrigin } from your server. Always pass the server's apiOrigin,
// so the browser and the server stay in the same environment.
const response = await fetch('/api/config')
if (!response.ok) {
  message.textContent = 'Could not load the payment settings. Check the server log.'
  throw new Error(`GET /api/config failed (${response.status})`)
}
const config = await response.json()

const tapaya = await loadTapaya(config.publishableKey, { apiOrigin: config.apiOrigin })
const payment = tapaya.createPayment({
  prepare: '/api/payments', // same-origin bare POST with cookies, answered by server.mjs
  onChange: ({ canSubmit, outcome, error }) => {
    button.disabled = !canSubmit
    if (outcome === 'pending') message.textContent = 'Confirming the payment…'
    if (outcome === 'completed') location.assign('/checkout/confirmation')
    if (outcome === 'ended') message.textContent = 'This payment cannot be completed. Start a new checkout.'
    if (error) console.warn('Tapaya payment error', error.code, error.message)
  },
})
await payment.mount('#card-fields')

form.addEventListener('submit', async (event) => {
  event.preventDefault()
  if (!form.reportValidity() || !payment.snapshot.canSubmit) return
  message.textContent = ''
  await payment.submit().catch(() => {
    // The card fields show the reason. The shopper can retry when canSubmit allows it.
  })
})

// In a single-page app, call payment.destroy() when the checkout view unmounts.
```

## How the outcome maps to the page

From the "Show the outcome" section of the Embedded Checkout guide:

| `outcome` | What the demo does |
| --- | --- |
| `open` | Pay button enabled while `canSubmit` is true. A decline stays `open`; the next submit gets a new attempt. |
| `pending` | Shows "Confirming the payment…". The SDK keeps checking while the page is visible. |
| `completed` | Goes to `/checkout/confirmation`, where the server verifies the order. |
| `ended` | Tells the shopper to start a new checkout. |

## React instead of plain JS

If the user's page is React, use `usePayment()` from
`@tapayadot/checkout/react` instead. It takes the same options plus
`publishableKey` and `apiOrigin`, and returns `fieldRef`, `snapshot`, and
`submit()`. Fetch its current signature with `get_page`
`/online/integration/embedded_checkout` before writing it; the docs describe
it in one paragraph only.
