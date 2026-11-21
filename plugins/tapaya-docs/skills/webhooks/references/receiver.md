# Minimal verified webhook receiver

A Tapaya webhook receiver for Node.js 22 or newer with no dependencies. It
reads the raw body, verifies `X-Tapaya-Signature`, checks the organization,
stores each event once, and answers `204` before any business logic runs.

Check the details against the live docs with the `tapaya-docs` MCP tools first
(`get_page` `/webhooks/receiving` and `/webhooks`). The docs win if they
differ from this file.

Sources:

- https://docs.tapaya.com/webhooks/receiving#check-the-signature (the `verifyTapayaWebhook` function below is the docs version)
- https://docs.tapaya.com/webhooks/receiving#store-and-acknowledge
- https://docs.tapaya.com/webhooks#event-envelope
- https://docs.tapaya.com/webhooks#delivery-rules

## File tree

```text
tapaya-webhook-receiver/
├── .env.example
├── .gitignore
├── package.json
├── verify-tapaya-webhook.mjs
└── server.mjs
```

## .env.example

Copy to `.env`, then uncomment and fill in the secret line. Never commit `.env`.

```bash
# Signing secret shown once when you save or rotate the destination
# (Platform > API Keys > Webhooks, or the PUT /integrator/organization/webhook response).
# TAPAYA_WEBHOOK_SECRET=twhsec_xxx  # paste your sandbox signing secret
# Organization ID shown in the same Webhooks section. Not a secret.
TAPAYA_ORGANIZATION_ID=paste-your-organization-id
PORT=8080
```

The variable names match the docs example.

## .gitignore

```text
node_modules/
.env
events.jsonl
```

## package.json

```json
{
  "name": "tapaya-webhook-receiver",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "start": "node --env-file=.env server.mjs"
  }
}
```

## verify-tapaya-webhook.mjs

Copied from the docs. Do not change the HMAC input: the timestamp, a period,
then the raw body bytes, keyed with the full `twhsec_…` secret as UTF-8.

```js
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyTapayaWebhook(rawBody, signatureHeader, secret, now = Date.now()) {
  if (!Buffer.isBuffer(rawBody) || typeof signatureHeader !== "string") {
    throw new Error("Missing raw body or signature");
  }
  if (typeof secret !== "string" || !secret.startsWith("twhsec_")) {
    throw new Error("Missing signing secret");
  }

  const match = /^t=(\d+),v1=([0-9a-f]{64})$/.exec(signatureHeader);
  if (!match) throw new Error("Malformed signature");

  const timestamp = Number(match[1]);
  if (!Number.isSafeInteger(timestamp) || Math.abs(now / 1000 - timestamp) > 300) {
    throw new Error("Expired signature");
  }

  const expected = createHmac("sha256", Buffer.from(secret, "utf8"))
    .update(`${match[1]}.`, "utf8")
    .update(rawBody)
    .digest();
  const received = Buffer.from(match[2], "hex");

  if (!timingSafeEqual(expected, received)) throw new Error("Invalid signature");
  return JSON.parse(rawBody.toString("utf8"));
}
```

## server.mjs

The demo stores events in `events.jsonl` and keeps seen IDs in memory. A real
receiver uses a database table with a unique constraint on the event `id`.

```js
import { createServer } from "node:http";
import { appendFile, readFile } from "node:fs/promises";
import { verifyTapayaWebhook } from "./verify-tapaya-webhook.mjs";

const PORT = Number(process.env.PORT ?? 8080);
const SECRET = process.env.TAPAYA_WEBHOOK_SECRET;
const ORGANIZATION_ID = process.env.TAPAYA_ORGANIZATION_ID;
const STORE = new URL("./events.jsonl", import.meta.url);
const MAX_BODY = 1024 * 1024;

if (!SECRET || !ORGANIZATION_ID) {
  console.error("Set TAPAYA_WEBHOOK_SECRET and TAPAYA_ORGANIZATION_ID in .env");
  process.exit(1);
}

// Load IDs already stored, so restarts keep deduplicating.
const seen = new Set(
  (await readFile(STORE, "utf8").catch(() => ""))
    .split("\n").filter(Boolean).map((line) => JSON.parse(line).id),
);

async function saveEventOnce(event) {
  if (seen.has(event.id)) return false;
  await appendFile(STORE, JSON.stringify(event) + "\n"); // throws -> non-2xx -> Tapaya retries
  seen.add(event.id);
  return true;
}

function readRawBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    let size = 0;
    req.on("data", (chunk) => {
      size += chunk.length;
      if (size > MAX_BODY) { reject(new Error("Body too large")); req.destroy(); return; }
      chunks.push(chunk);
    });
    req.on("end", () => resolve(Buffer.concat(chunks)));
    req.on("error", reject);
  });
}

createServer(async (req, res) => {
  if (req.method !== "POST" || req.url !== "/webhooks/tapaya") {
    res.writeHead(404).end();
    return;
  }

  let event;
  try {
    const raw = await readRawBody(req); // the original bytes, never re-serialized JSON
    event = verifyTapayaWebhook(raw, req.headers["x-tapaya-signature"], SECRET);
  } catch (error) {
    console.warn("Rejected webhook:", error.message);
    res.writeHead(400).end();
    return;
  }

  if (event.organizationId !== ORGANIZATION_ID) {
    console.warn("Rejected webhook for another organization");
    res.writeHead(400).end();
    return;
  }

  try {
    const isNew = await saveEventOnce(event);
    console.log(`${isNew ? "Stored" : "Duplicate"} ${event.type} ${event.id}`);
    res.writeHead(204).end(); // 2xx within 10 seconds, also for duplicates
  } catch (error) {
    console.error("Could not store event", error);
    res.writeHead(500).end(); // Tapaya retries
    return;
  }

  // Process stored events afterwards, from your own queue. Order is not guaranteed,
  // so reconcile older snapshots against current state. Route by event.type and
  // ignore types you do not recognize.
}).listen(PORT, () => {
  console.log(`Webhook receiver on port ${PORT}, path /webhooks/tapaya`);
});
```

## Test the receiver locally with a self-signed request

This checks your verification code without Tapaya. Run it in a second
terminal; it signs a sample body with the secret from `.env` exactly as the
docs describe.

```bash
node --env-file=.env --input-type=module -e '
import { createHmac } from "node:crypto";
const body = JSON.stringify({ id: "local-test-1", type: "webhook.test", apiVersion: "1",
  organizationId: process.env.TAPAYA_ORGANIZATION_ID, merchantId: null, paymentType: null,
  createdAt: new Date().toISOString(), test: true, data: { message: "local" } });
const t = Math.floor(Date.now() / 1000);
const v1 = createHmac("sha256", Buffer.from(process.env.TAPAYA_WEBHOOK_SECRET, "utf8"))
  .update(`${t}.`, "utf8").update(body).digest("hex");
const res = await fetch(`http://localhost:${process.env.PORT ?? 8080}/webhooks/tapaya`, {
  method: "POST", headers: { "Content-Type": "application/json", "X-Tapaya-Signature": `t=${t},v1=${v1}` }, body });
console.log(res.status);
'
```

Expected: `204` the first time, `204` again with "Duplicate" in the server
log the second time. Change one byte of the body after signing and expect
`400`.

## Express variant

If the user's app uses Express, follow the docs example instead: mount
`express.raw({ type: "application/json" })` on the webhook route **before** any
global JSON parser, and pass `req.body` (a Buffer) to `verifyTapayaWebhook`.
