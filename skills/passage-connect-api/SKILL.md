---
name: passage-connect-api
description: >
  Build integrations with the Passage Connect API — the universal account linking platform.
  Use this skill when writing backend code that creates Passage links, handles Passage webhooks,
  or interacts with any Passage Connect endpoint (links, providers, claim, webhook verification).
  Trigger on: "Passage", "passage connect", "account linking", "link intent", "claim code",
  "passage webhook", "passage SDK integration", "passage provider", "getpassage",
  or any mention of the Passage Connect API, link-based account linking, or Passage integrations.
---

# Passage Connect API Integration

Passage Connect is a universal account linking platform — like Plaid, but for any online service. Developers create **link intents** via the API; end users open an app-clip/webview, log into the target service directly on their device (credentials never leave the device), and Passage runs automations to extract or write data on the user's behalf.

## Base URL

**Production:** `https://connect.getpassage.ai`
**Dev:** `https://connect.dev.getpassage.ai`

## Authentication

All `/v1/*` endpoints require a **Bearer API key**:

```
Authorization: Bearer psg_<key>
```

API keys are created in the Passage dashboard. Keys are scoped to an organization.

---

## Core Flow: Link-Based Account Linking

Inspired by Plaid's Link model — cheap D1 record first, session only on claim.

### Step 1: Create a Link Intent

```
POST /v1/links
Authorization: Bearer psg_xxx
Content-Type: application/json
```

**Single-operation body:**
```json
{
  "integrationId": "tmobile",
  "resource": "paymentMethod",
  "action": "read",
  "externalUserId": "user_123",
  "webhookUrl": "https://your-api.com/webhooks/passage",
  "debug": false,
  "expiresIn": 3600
}
```

**Multi-operation body:**
```json
{
  "integrationId": "tmobile",
  "operations": [
    { "resource": "paymentMethod", "action": "read" },
    { "resource": "mobileBillingStatement", "action": "read", "arguments": { "limit": 3 } }
  ],
  "externalUserId": "user_123",
  "webhookUrl": "https://your-api.com/webhooks/passage"
}
```

You must provide either `(resource + action)` OR `operations` — not both.

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `integrationId` | string | yes | Provider ID (e.g. `"tmobile"`, `"att"`, `"verizon"`) |
| `resource` | string | one of | Resource name (legacy single-op) |
| `action` | string | one of | Action name (legacy single-op) |
| `arguments` | object | no | Arguments for single-op |
| `operations` | array | one of | Array of `{ resource, action, arguments? }` |
| `externalUserId` | string | no | Your user ID for correlation |
| `webhookUrl` | string (URL) | no | ES256-signed webhook delivery URL |
| `debug` | boolean | no | Enable debug mode |
| `expiresIn` | integer | no | TTL in seconds (default: 4 hours) |

**Response (201):**
```json
{
  "linkId": "link_uuid",
  "claimCode": "clm_uuid",
  "integrationId": "tmobile",
  "operations": [{ "resource": "paymentMethod", "action": "read" }],
  "resource": "paymentMethod",
  "action": "read",
  "status": "pending",
  "appClipUrl": "https://clip.getpassage.ai?code=clm_uuid",
  "expiresAt": 1710000000000,
  "createdAt": 1710000000000
}
```

Present `appClipUrl` to the end user (QR code, deep link, etc.). No session is created until the user claims it.

### Step 2: User Claims the Link

When the user opens the app-clip, the SDK calls:

```
POST /link/claim
Content-Type: application/json

{ "code": "clm_uuid" }
```

This is a **public endpoint** (no API key) — called by the client SDK, not your backend. It creates a session and starts the automation.

**Response (200):**
```json
{
  "linkId": "link_uuid",
  "sessionId": "session_uuid",
  "clientToken": "ct_xxx",
  "websocketUrl": "wss://sessions.getpassage.ai/sessions/session_uuid/ws",
  "expiresAt": 1710000000000
}
```

### Step 3: Monitor via Webhook or Polling

**Webhook** (recommended): Passage sends an ES256-signed POST to your `webhookUrl` when the session completes or fails. See Webhooks below.

**Polling**: Check status with `GET /v1/links/:id`.

### Step 4: Read the Result

```
GET /v1/links/:id
Authorization: Bearer psg_xxx
```

**Response:**
```json
{
  "id": "link_uuid",
  "integrationId": "tmobile",
  "operations": [{ "resource": "paymentMethod", "action": "read" }],
  "resource": "paymentMethod",
  "action": "read",
  "externalUserId": "user_123",
  "status": "complete",
  "currentSessionId": "session_uuid",
  "result": { "paymentMethods": [{ "last4": "1234", "type": "credit_card", "isDefault": true }] },
  "error": null,
  "expiresAt": 1710000000000,
  "createdAt": 1710000000000,
  "updatedAt": 1710000000000
}
```

---

## All Endpoints

### Authenticated (API key required)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/links` | Create a link intent |
| `GET` | `/v1/links/:id` | Get link status and result |
| `GET` | `/v1/links` | List all links for your org |
| `GET` | `/v1/providers` | Discovery — list integrations with JSON Schema |

### Public (no auth)

| Method | Path | Description |
|---|---|---|
| `POST` | `/link/claim` | Claim a link (called by SDK) |
| `GET` | `/link/ws?code=clm_xxx` | WebSocket — real-time link status updates |
| `POST` | `/webhook_verification_key/get` | Fetch webhook signing public key |

---

## Link States

```
pending → active → complete
                 → failed    (re-claimable — new session each time)
       → expired
```

- **pending**: Created, waiting for user to claim
- **active**: User claimed, automation running
- **complete**: Automation succeeded, `result` populated
- **failed**: Automation failed, `error` populated. Can be re-claimed
- **expired**: TTL elapsed (default 4 hours)

---

## Available Integrations & Operations

Discover dynamically via `GET /v1/providers` (returns JSON Schema for arguments and results).

| Integration | Resource | Actions | Write Arguments |
|---|---|---|---|
| `tmobile` | `paymentMethod` | `read`, `write` | `{ cardNumber, expirationDate, cvv, nameOnCard, billingZip, billingAddress?, dryRun? }` |
| `tmobile` | `mobileBillingStatement` | `read` | `{ limit?: number }` |
| `att` | `paymentMethod` | `read`, `write` | same as tmobile |
| `att` | `mobileBillingStatement` | `read` | `{ limit?: number }` |
| `verizon` | `paymentMethod` | `read`, `write` | same as tmobile |
| `verizon` | `mobileBillingStatement` | `read` | `{ limit?: number }` |
| `ubereats` | `orderHistory` | `read` | `{ limit?: number }` |
| `apple-review` | `readHistory` | `read` | `{ limit?: number }` |

For full result schemas, read `references/types.md`.

---

## Webhooks

Passage sends webhooks as POST requests with ES256 JWT signatures.

**Headers sent with each webhook:**
- `X-Passage-Signature` — ES256-signed JWT
- `X-Passage-Timestamp` — Unix seconds

**JWT structure:**
- Header: `{ "alg": "ES256", "typ": "JWT", "kid": "<key_id>" }`
- Payload: `{ "iat": <unix_seconds>, "request_body_sha256": "<hex_hash_of_body>" }`

**Webhook body:**
```json
{
  "event": "link.complete",
  "linkId": "link_uuid",
  "status": "complete",
  "result": { "paymentMethods": [...] },
  "timestamp": 1710000000000
}
```

Or on failure:
```json
{
  "event": "link.failed",
  "linkId": "link_uuid",
  "status": "failed",
  "error": { "code": "AUTOMATION_ERROR", "message": "..." },
  "timestamp": 1710000000000
}
```

**Webhook events:** `link.complete`, `link.failed`

**Retry policy:** 3 attempts with exponential backoff (0ms, 1s, 5s). If all fail, the webhook is dropped but the link status is already persisted in D1.

### Verifying Webhooks

1. Parse the JWT header from `X-Passage-Signature` to get `kid`
2. Fetch the public key: `POST /webhook_verification_key/get` with `{ "key_id": "<kid>" }`
3. Verify the JWT signature using the returned PEM public key (ES256 / P-256)
4. Hash the raw request body with SHA-256, compare against `request_body_sha256` in the JWT payload
5. Check `iat` is recent (within your tolerance window)

```typescript
async function verifyPassageWebhook(req: Request): Promise<boolean> {
  const signature = req.headers.get('X-Passage-Signature')!;
  const rawBody = await req.text();

  // 1. Decode JWT header to get kid
  const [headerB64] = signature.split('.');
  const header = JSON.parse(atob(headerB64.replace(/-/g, '+').replace(/_/g, '/')));

  // 2. Fetch public key
  const keyRes = await fetch('https://connect.getpassage.ai/webhook_verification_key/get', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ key_id: header.kid }),
  });
  const { key: publicKeyPem } = await keyRes.json();

  // 3. Verify JWT signature with public key (use jose, jsonwebtoken, or Web Crypto)
  // 4. Compare request_body_sha256 with SHA-256 hex of rawBody
  // 5. Check iat recency

  return true; // after all checks pass
}
```

---

## Real-Time Link Status (WebSocket)

Client SDKs can subscribe to link status changes via WebSocket:

```
GET /link/ws?code=clm_uuid
Upgrade: websocket
```

The server pushes status updates as the link progresses through states. Useful for showing real-time progress in your UI.

---

## Error Handling

All endpoints return errors as:
```json
{
  "error": "Human-readable message",
  "issues": [{ "path": "field.name", "message": "validation message" }]
}
```

| Status | Meaning |
|---|---|
| `400` | Validation error, unknown operation, or invalid arguments |
| `401` | Missing or invalid API key |
| `404` | Link not found or invalid claim code |
| `409` | Link already being claimed, or already used (non-reusable) |
| `410` | Link expired |
| `502` | Upstream service error (session creation failed) |

---

## Code Patterns

### TypeScript — Create Link + Poll

```typescript
const PASSAGE_API_KEY = process.env.PASSAGE_API_KEY!;
const BASE_URL = 'https://connect.getpassage.ai';

async function createLink(integrationId: string, resource: string, action: string, userId: string) {
  const res = await fetch(`${BASE_URL}/v1/links`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${PASSAGE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      integrationId,
      resource,
      action,
      externalUserId: userId,
      webhookUrl: 'https://your-api.com/webhooks/passage',
    }),
  });
  if (!res.ok) throw new Error(`Failed to create link: ${res.status}`);
  return res.json();
}

async function getLinkResult(linkId: string) {
  const res = await fetch(`${BASE_URL}/v1/links/${linkId}`, {
    headers: { 'Authorization': `Bearer ${PASSAGE_API_KEY}` },
  });
  return res.json();
}
```

### Webhook Handler (Express)

```typescript
import express from 'express';

const app = express();
app.use('/webhooks/passage', express.raw({ type: 'application/json' }));

app.post('/webhooks/passage', async (req, res) => {
  const rawBody = req.body.toString();
  const signature = req.headers['x-passage-signature'] as string;

  // Verify signature (see Webhooks section above)

  const payload = JSON.parse(rawBody);
  switch (payload.event) {
    case 'link.complete':
      await handleSuccess(payload.linkId, payload.result);
      break;
    case 'link.failed':
      await handleFailure(payload.linkId, payload.error);
      break;
  }

  res.sendStatus(200);
});
```

### Webhook Handler (Hono / Cloudflare Workers)

```typescript
import { Hono } from 'hono';

const app = new Hono();

app.post('/webhooks/passage', async (c) => {
  const rawBody = await c.req.text();
  const signature = c.req.header('X-Passage-Signature')!;

  // Verify signature (see Webhooks section above)

  const payload = JSON.parse(rawBody);
  if (payload.event === 'link.complete') {
    await handleSuccess(payload.linkId, payload.result);
  }

  return c.text('ok');
});

export default app;
```

---

## Reference Files

For detailed TypeScript type definitions (result schemas, link types, webhook types), read `references/types.md`.
