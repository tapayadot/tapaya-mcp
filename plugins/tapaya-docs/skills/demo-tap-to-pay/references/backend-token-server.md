# Tap to Pay demo: sandbox setup and token server

The backend half of the `demo-tap-to-pay` skill. Check every call against the
live docs with the `tapaya-docs` MCP tools first (`get_page`,
`get_api_endpoint` with `POST /merchant/auth/login`). The docs win if they
differ from this file.

Sources:

- https://docs.tapaya.com/in-store/integration/backend_integration
  (`POST /merchant/auth/register`, `POST /merchant/auth/login`)
- https://docs.tapaya.com/platform-api#authentication (Server Secret Token, header format)
- https://docs.tapaya.com/platform-api/merchant_onboarding#invite-a-merchant-to-complete-hosted-onboarding
- https://docs.tapaya.com/in-store/getting-started/quick_start_guide#onboard-the-merchant (sandbox auto-approval)

Platform API calls send the Server Secret Token **as is** in `Authorization`,
with no `Bearer` prefix. (Checkout is different and uses `Bearer`.)

## File tree

```text
tapaya-tap-to-pay-demo/
└── backend/
    ├── .env.example
    ├── .gitignore
    ├── package.json
    └── server.mjs
```

## .env.example

Copy to `.env`, then uncomment and fill in the secret line. Never commit `.env`.

```bash
# Sandbox Platform API. Do not point this at https://api.tapaya.com for the demo.
TAPAYA_API_URL=https://api.sandbox.tapaya.com
# TAPAYA_SERVER_SECRET_TOKEN=xxx_paste_your_sandbox_secret_key  # Platform > Settings > APIs (API Keys page) > Create key
# TAPAYA_MERCHANT_TOKEN=demo-merchant-1  # the merchantToken you registered below
PORT=3000
```

The variable names are this demo's choice; the docs do not prescribe them.

## .gitignore

```text
node_modules/
.env
```

## One-time sandbox setup (terminal)

Run these from a trusted terminal after filling in `.env`. They read the
values from the environment, so the secret never appears in the command.

```bash
cd backend
set -a; . ./.env; set +a
```

1. Register a merchant shell. Pick a stable `merchantToken` and a real inbox
   you can read. A `409` means it is already registered, which is fine.

   ```bash
   curl -sS -X POST "$TAPAYA_API_URL/merchant/auth/register" \
     -H 'Content-Type: application/json' \
     -H "Authorization: $TAPAYA_SERVER_SECRET_TOKEN" \
     -d "{\"merchantToken\":\"$TAPAYA_MERCHANT_TOKEN\",\"merchantName\":\"Tap to Pay demo\",\"email\":\"you@example.com\"}"
   ```

2. Onboard it. Either open the merchant in the Tapaya Platform and complete
   onboarding there (**Payments > Merchants**), or send a hosted invite bound
   to the same `merchantToken` and open the returned `registrationUrl`:

   ```bash
   curl -sS -X POST "$TAPAYA_API_URL/integrator/merchant/send-registration-invite" \
     -H 'Content-Type: application/json' \
     -H "Authorization: $TAPAYA_SERVER_SECRET_TOKEN" \
     -d "{\"email\":\"you@example.com\",\"merchantToken\":\"$TAPAYA_MERCHANT_TOKEN\"}"
   ```

   In sandbox, merchants are approved automatically right after onboarding.
   Request in-store payments during onboarding.

## package.json

No runtime dependencies. Node's built-in `fetch` calls the API, and
`--env-file` loads `.env`.

```json
{
  "name": "tapaya-tap-to-pay-demo-backend",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "start": "node --env-file=.env server.mjs"
  }
}
```

## server.mjs

```js
import { createServer } from 'node:http'

const PORT = Number(process.env.PORT ?? 3000)
const API_URL = process.env.TAPAYA_API_URL ?? 'https://api.sandbox.tapaya.com'
const SECRET = process.env.TAPAYA_SERVER_SECRET_TOKEN
const MERCHANT_TOKEN = process.env.TAPAYA_MERCHANT_TOKEN

if (!SECRET || !MERCHANT_TOKEN) {
  console.error('Set TAPAYA_SERVER_SECRET_TOKEN and TAPAYA_MERCHANT_TOKEN in backend/.env')
  process.exit(1)
}
if (new URL(API_URL).hostname !== 'api.sandbox.tapaya.com') {
  console.error(`Refusing to run the demo against ${API_URL}. Use https://api.sandbox.tapaya.com`)
  process.exit(1)
}

// POST /sdk-token : asks Tapaya for a short-lived SDK login token for the demo merchant.
// A real backend first authenticates its own user and picks THEIR merchantToken.
async function sdkToken(res) {
  try {
    const response = await fetch(new URL('/merchant/auth/login', API_URL), {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: SECRET }, // no "Bearer"
      body: JSON.stringify({ merchantToken: MERCHANT_TOKEN, allowOnboarding: true }),
      signal: AbortSignal.timeout(15000),
    })
    const body = await response.json().catch(() => ({}))
    if (!response.ok) {
      console.error('login failed', response.status, body.errorCode, body.message)
      res.writeHead(502, { 'Content-Type': 'application/json' })
      return res.end(JSON.stringify({ error: `Tapaya login failed (${response.status} ${body.errorCode ?? ''})` }))
    }
    console.log('Issued SDK token for', MERCHANT_TOKEN) // never log the token itself
    res.writeHead(200, { 'Content-Type': 'application/json', 'Cache-Control': 'no-store' })
    res.end(JSON.stringify({ token: body.token }))
  } catch (error) {
    console.error(error)
    res.writeHead(502, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify({ error: 'Could not reach Tapaya' }))
  }
}

createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/sdk-token') return sdkToken(res)
  res.writeHead(404).end()
}).listen(PORT, '127.0.0.1', () => {
  console.log(`Token server on http://127.0.0.1:${PORT} -> ${API_URL}`)
  console.log(`Phone access over USB: adb reverse tcp:${PORT} tcp:${PORT}`)
})
```

## Notes on the template

- The server listens on `127.0.0.1` only. The phone reaches it through
  `adb reverse` over USB, so nobody else on the network can mint tokens for
  the demo merchant.
- The token response is `{ "token": "..." }`. Pass it to
  `Accept.auth.authenticate()` and discard it. You need a fresh token each time
  the SDK signs in.
- `allowOnboarding` defaults to `true` and must be `true` on the first call so
  the merchant can enroll a payment method.
- Login errors return `{ message, errorCode, errors }` with status 400, 401,
  or 404 (`get_api_endpoint` `POST /merchant/auth/login`).
