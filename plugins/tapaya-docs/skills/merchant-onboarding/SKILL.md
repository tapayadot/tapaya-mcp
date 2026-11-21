---
name: merchant-onboarding
description: Onboard and manage Tapaya merchants through the Platform API and the Tapaya Platform - pick an onboarding flow (Platform UI, own UI, email invite, campaign link), create merchant shells, send hosted invites, validate and submit KYB, request in-store and online capabilities, track approval, and manage invites, members, terminals, and commission profiles. Use when the user asks how to onboard, register, invite, or approve a Tapaya merchant, about KYB or capability values, or about merchant management endpoints.
---

# Tapaya merchant onboarding

Help the user onboard merchants and manage them through the Tapaya Platform
API, grounded in the docs. A merchant onboards once; one onboarding covers
In-Store Acceptance, Online Payments (Checkout), or both.

Sources:

- Onboarding guide (flows, documents, verification): https://docs.tapaya.com/merchants/onboarding
- Platform API (environments, authentication): https://docs.tapaya.com/platform-api
- Merchant Onboarding API: https://docs.tapaya.com/platform-api/merchant_onboarding
- Merchant Management API: https://docs.tapaya.com/platform-api/merchant_management
- Accept SDK merchant shell and login token: https://docs.tapaya.com/in-store/integration/backend_integration

## 1. Check the docs first

Onboarding fields and rules change often (for example, proof of address for
every stakeholder is enforced from 2 December 2026). Before answering or
writing code:

- `get_page` `/platform-api/merchant_onboarding` for the flow you need.
- `get_page` `/platform-api/merchant_management` for invites, members,
  terminals, and commission profiles.
- `search_api_endpoints` and `get_api_endpoint` for the exact request and
  response schema, for example `"send registration invite"`,
  `"onboarding validate"`, `"onboarding status"`.
- Numeric enums (`countryId`, `businessTypeId`, `documentType`,
  `paymentProcessorId`) are listed in the Swagger docs
  (https://api.tapaya.com/openapi/index.html), not in the guides. Look them up
  with `get_api_endpoint`; never guess a number.

Cite the page URL for each rule you state. If the docs do not answer a
question, say so.

## 2. Authentication and environments

- Every call uses the **Server Secret Token** (labelled **Secret key** on the
  API Keys page, https://platform.tapaya.com/api-keys/), sent **as-is** in
  `Authorization`, without `Bearer`. Server-side only; tokens cannot be
  scoped, so treat each as full organization access.
- Sandbox: `https://api.sandbox.tapaya.com`. Production:
  `https://api.tapaya.com`. Do not test in production. In sandbox, merchants
  are approved automatically right after onboarding.
- Rate limit: 100 requests per minute per IP address (`429` beyond that).

Any code you write reads the token from an environment variable. In
`.env.example`, keep it commented out:

```bash
TAPAYA_API_URL=https://api.sandbox.tapaya.com
# TAPAYA_SERVER_SECRET_TOKEN=xxx_paste_your_sandbox_secret_key  # Platform > Settings > APIs > Create key
```

## 3. Choose the flow

Ask who enters the merchant's data, then pick (all four are available in the
Platform UI too):

| Flow | When | Entry point |
| --- | --- | --- |
| Platform UI | Your operators onboard on the merchant's behalf | Platform: **Payments > Merchants > Add a merchant manually > Start onboarding** |
| Email invite | A known merchant self-onboards on a Tapaya-hosted form | `POST /integrator/merchant/send-registration-invite` (returns `registrationUrl`) |
| Campaign link | Many or unknown merchants self-register | `POST /integrator/merchant/reusable-registration-invites` |
| Your own UI | You collect KYB in your product | `POST /integrator/merchant/onboarding/validate/*`, then `POST /integrator/merchant/onboarding` |

**If your system owns the merchant ID** (for example for the Accept SDK),
create the shell first with `POST /merchant/auth/register` or
`POST /integrator/merchant`, then send the invite with the **same**
`merchantToken`. Otherwise completing the invite creates a second merchant.

Do not confuse `POST /integrator/merchant/{merchantId}/invite`: it invites a
platform user to an existing merchant and does not start KYB.

## 4. Your own UI: rules that trip people up

- `POST /integrator/merchant/onboarding` creates the merchant **and** submits
  onboarding in one call. Invalid data returns `422` with field errors and no
  merchant is created.
- `capability` is required: `1` both, `2` terminals (in-store), `3` Checkout
  (online). Send the matching `terminal` and/or `online` objects. For
  Checkout, `online.url` is required and its hostname must contain `www`;
  `statementDescriptor` is 5 to 22 characters; `topCountries` holds one to
  four entries totalling 100.
- The body is strict: unknown fields are rejected. Amounts are in minor units.
- Before submitting, get the merchant's consent to the Tapaya Integrators
  Terms (https://tapaya.com/terms) and Privacy Policy
  (https://tapaya.com/privacy), list Tapaya as a data processor in your own
  privacy policy, send the merchant's own IP as `signerIp` when updating the
  processor account, and collect proof of address for every stakeholder.
- Upload documents with `POST /integrator/merchant/{merchantId}/document`;
  set `submitMerchant=true` on the final upload.

## 5. Track approval

- `GET /integrator/merchant/{merchantId}/onboarding/status`: read `pipeline`
  (`stage`, `currentStep`, `steps`, `channels.inStore`, `channels.online`) and
  `currentReview.missingRequirements`. Per processor, `chargesEnabled` means
  it can take payments and `payoutsEnabled` means it can receive money.
- Subscribe to webhooks `merchant.onboarding.approved`,
  `merchant.onboarding.action_required`, and `merchant.status_changed` (see
  the `webhooks` skill and https://docs.tapaya.com/webhooks/events#merchant-onboarding).
- In-store apps can call `Accept.merchant.onboardingStatus()`.
- For Checkout, the merchant must also show **Ready** in **Checkout**
  settings (https://docs.tapaya.com/online/integration/checkout_settings).
  Creating a merchant or a key alone does not enable payments, and in-store
  approval alone does not enable Checkout.

## 6. Manage merchants

From the Merchant Management page: read a saved onboarding draft
(`GET /integrator/merchant/{merchantId}/onboarding`), save channel selections
(`POST …/channel`), list members and terminals, list and expire email invites
and reusable links, pause or resume links, resend a stakeholder invite (not
within 10 minutes), manage commission profiles and the default profile, and
read a merchant's pricing. Fetch the exact paths with `get_page` before
writing calls.

Common errors there: `400` invalid data or deleting a default or assigned
profile; `401` bad token; `404` not in your organization; `409` stale profile
revision, conflicting onboarding state, used or revoked invite, or a resend
within 10 minutes.

## 7. Answer format

Name the flow, list the calls in order with their docs URLs, show requests
with the token read from the environment (never a literal key), and end with
how to confirm approval. Mark anything the docs leave open as a gap.

## Gaps in the docs

- The invite flow says to poll `GET /integrator/merchant/{merchantId}/status`
  (merchant and payment-method status), while the approval section documents
  `GET /integrator/merchant/{merchantId}/onboarding/status` (processor
  pipeline). Both exist in the API spec; the guides do not say when to prefer
  one over the other. Read both schemas with `get_api_endpoint`.
- Enum values live only in the Swagger docs.
- The docs do not describe creating or switching to a sandbox organization in
  the Platform.
