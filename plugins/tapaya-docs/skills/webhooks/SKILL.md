---
name: webhooks
description: Set up, receive, verify, and test Tapaya webhooks - configure the organization's destination and event types, verify the X-Tapaya-Signature HMAC over the raw body, store events once and acknowledge with 2xx, and send a test webhook. Use when the user asks about Tapaya webhooks, webhook signatures or signing secrets, event types such as payment.online.succeeded or checkout.session.completed, or a webhook receiver that fails or gets no events.
---

# Tapaya webhooks

Help the user configure a Tapaya webhook destination, write a receiver that
verifies every request, and test it. Ground every detail in the docs.

Sources:

- Overview, envelope, delivery rules: https://docs.tapaya.com/webhooks
- Configuration (Platform and API): https://docs.tapaya.com/webhooks/configuration
- Event types: https://docs.tapaya.com/webhooks/events
- Verify and test: https://docs.tapaya.com/webhooks/receiving

## 1. Check the docs first

Webhook rules change, and the event catalog grows. Before answering or writing
code, read the relevant page with the `tapaya-docs` MCP tools:

- `get_page` `/webhooks/receiving` for the signature format and the
  reference `verifyTapayaWebhook` function.
- `get_page` `/webhooks/events` for event names and `data` shapes.
- `get_page` `/webhooks/configuration` for the destination API and URL rules.
- `search_api_endpoints` `"webhook"` and `get_api_endpoint` for the exact
  `PUT`/`GET`/`DELETE /integrator/organization/webhook` schema.

Cite the page URL for each rule you state. If the docs do not answer a
question, say so.

## 2. Key facts (verify before relying on them)

- **One destination per organization**, one active signing secret
  (`twhsec_…`). It is shown once, on create or rotation. It only verifies
  webhooks; the Server Secret Token authorizes configuration calls.
- **Configure** in the Platform under **API Keys > Webhooks**
  (https://platform.tapaya.com/api-keys/): URL, events or **Subscribe to all
  event types**, **Send webhooks** on, **Save**, copy the secret. Or call
  `PUT /integrator/organization/webhook` with `url`, `eventTypes`,
  `subscribeAllEventTypes`, `enabled`, `rotateSecret`, sending the Server
  Secret Token as-is in `Authorization`. A PUT replaces the whole
  configuration.
- **URL rules:** public HTTPS on port 443, no localhost or private IPs, no
  credentials, fragment, redirects, or custom auth headers.
- **Signature:** header `X-Tapaya-Signature: t=<unix seconds>,v1=<hex>`.
  `v1` is HMAC-SHA256 over `<t>.` followed by the **raw body bytes**, keyed
  with the full secret as UTF-8 (prefix included). Compare in constant time.
  Reject timestamps more than 300 seconds from your clock.
- **Envelope:** `id`, `type`, `apiVersion` (`"1"`), `organizationId`,
  `merchantId`, `paymentType`, `createdAt`, `data`. Check `organizationId`.
- **Delivery:** any `2xx` within 10 seconds counts. Up to 12 attempts over
  about 20 hours. At least once, order not guaranteed, 2 concurrent requests
  per organization. Past deliveries cannot be replayed; catch up through the
  payment and merchant APIs.
- **Event names:** payment and refund events use a category:
  `payment.online.*`, `payment.instore.card.*`, `payment.instore.cash.*`.
  Unqualified names like `payment.succeeded` are rejected when saving. Most
  integrations start with `payment.online.succeeded`, `payment.online.failed`,
  and `payment.online.refund.succeeded`; Checkout adds
  `checkout.session.completed` and `checkout.session.expired`; onboarding
  platforms add `merchant.onboarding.approved` and
  `merchant.onboarding.action_required`.
- **Status words differ:** event names say `succeeded`/`canceled`; status
  values say `success`/`cancelled`. A `checkout.session.completed` event is
  not proof of captured funds; check `paymentStatus`.

## 3. Build the receiver

Ask which stack the user has. For Node.js, scaffold the template in
[references/receiver.md](references/receiver.md): a dependency-free
`node:http` server, the docs' `verifyTapayaWebhook` function, an
organization check, store-once by event `id`, and `204` before processing.
For Express, mount `express.raw({ type: "application/json" })` on the route
before any JSON parser, as the docs show.

For other languages, port the same steps: raw bytes, parse the header with
`^t=(\d+),v1=([0-9a-f]{64})$`, 300-second window, HMAC-SHA256 with the
secret as the UTF-8 key over `t + "." + body`, constant-time compare, then
`JSON.parse`. Say the port is yours; the docs show Node.js only.

Secrets go in `.env`, never in code or chat. In `.env.example`, keep the
secret commented out:

```bash
# TAPAYA_WEBHOOK_SECRET=twhsec_xxx  # paste the signing secret shown once in Platform > API Keys > Webhooks
TAPAYA_ORGANIZATION_ID=paste-your-organization-id
```

## 4. Test it

1. **Locally, without Tapaya:** sign a sample body with the same secret and
   POST it to the receiver (the reference has a one-off Node command). Expect
   `204`, then `204` with "Duplicate", then `400` after changing a byte.
2. **From Tapaya:** expose the receiver through a public HTTPS tunnel (the
   docs allow this; follow the tunnel tool's own install docs), save the
   tunnel URL as the destination, and use the Platform's test action. It
   sends one signed `webhook.test` with no retries, even when webhooks are
   disabled. Manual tests are limited to 10 starts per 60 seconds per
   organization.
3. **Real events:** create sandbox activity (for example with the
   `demo-hosted-checkout` skill) and watch for `payment.online.*` and
   `checkout.session.*`. `webhook.test` has a different `data` shape, so
   route by `type`.

## 5. Common problems

| Symptom | Check |
| --- | --- |
| Signature mismatch | Raw bytes, not re-serialized JSON. Full `twhsec_` secret, not the Server Secret Token. A body parser running before the route. |
| Expired signature | Receiver clock out of sync. Use the header `t`, not `createdAt`. |
| URL rejected on save (`400`) | Not public HTTPS on 443, or has redirects, credentials, or a fragment. Unsupported event type name. |
| No events arrive | Wrong organization, **Send webhooks** off, event type not selected. Past changes are not backfilled. |
| Duplicates or older snapshots | Expected. Deduplicate by `id` and reconcile against current state. |
| Test shows `timeout` | The receiver did work before answering. Store, answer `2xx`, process later. |
| During secret rotation | Try the new secret first, then the old one, until the queue drains. |

## Gaps in the docs

- Verification code is shown for Node.js only.
- No sandbox-specific webhook setup is described beyond using the sandbox API
  base URL `https://api.sandbox.tapaya.com` for configuration calls.
