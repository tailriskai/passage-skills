# Passage Connect API — Type Reference

## Link Types

```typescript
// POST /v1/links request body
interface CreateLinkRequest {
  integrationId: string;
  // Single-op (legacy) — provide resource+action OR operations, not both
  resource?: string;
  action?: string;
  arguments?: Record<string, unknown>;
  // Multi-op
  operations?: OperationItem[];
  // Optional
  externalUserId?: string;
  webhookUrl?: string;     // URL, receives ES256-signed webhooks
  debug?: boolean;
  expiresIn?: number;      // TTL in seconds (default: 14400 = 4 hours)
}

interface OperationItem {
  resource: string;
  action: string;
  arguments?: Record<string, unknown>;
}

// POST /v1/links response (201)
interface CreateLinkResponse {
  linkId: string;
  claimCode: string;       // clm_uuid — used by SDK to claim
  integrationId: string;
  operations: { resource: string; action: string }[];
  resource: string;        // first operation (legacy)
  action: string;          // first operation (legacy)
  status: "pending";
  appClipUrl: string;      // present to user (QR code, deep link)
  expiresAt: number;       // epoch ms
  createdAt: number;       // epoch ms
}

// GET /v1/links/:id response
interface GetLinkResponse {
  id: string;
  integrationId: string;
  operations?: OperationItem[];
  resource: string;
  action: string;
  externalUserId: string | null;
  status: "pending" | "active" | "complete" | "failed" | "expired";
  currentSessionId: string | null;
  result?: unknown;        // populated on complete
  error?: unknown;         // populated on failed
  expiresAt: number;
  createdAt: number;
  updatedAt: number;
}

// GET /v1/links response
interface ListLinksResponse {
  links: Array<{
    id: string;
    integrationId: string;
    operations?: OperationItem[];
    resource: string;
    action: string;
    externalUserId: string | null;
    status: string;
    currentSessionId: string | null;
    expiresAt: number;
    createdAt: number;
    updatedAt: number;
  }>;
}

// POST /link/claim request (public, no auth)
interface ClaimLinkRequest {
  code: string;  // clm_uuid
}

// POST /link/claim response
interface ClaimLinkResponse {
  linkId: string;
  sessionId: string;
  clientToken: string;
  websocketUrl: string;    // wss://sessions.getpassage.ai/sessions/:id/ws
  expiresAt: number;
}
```

## Webhook Types

```typescript
type WebhookEvent = "link.complete" | "link.failed";

interface WebhookPayload {
  event: WebhookEvent;
  linkId: string;
  status: "complete" | "failed";
  result?: unknown;        // present on link.complete
  error?: WebhookError;    // present on link.failed
  timestamp: number;       // epoch ms
}

interface WebhookError {
  code: string;    // e.g. "AUTOMATION_ERROR", "CLIENT_TIMEOUT", "CONNECTION_FAILED",
                   //      "UNKNOWN_PROVIDER", "RESULT_VALIDATION_ERROR", "INTERNAL_ERROR"
  message: string;
}

// POST /webhook_verification_key/get
interface WebhookKeyRequest {
  key_id: string;
}

interface WebhookKeyResponse {
  key_id: string;
  key: string;  // PEM-encoded SPKI public key (EC P-256)
}

// Webhook signature structure (ES256 JWT)
// Header: { alg: "ES256", typ: "JWT", kid: string }
// Payload: { iat: number, request_body_sha256: string }
// Delivered as: X-Passage-Signature header
// Timestamp as:  X-Passage-Timestamp header (Unix seconds)
```

## Provider / Integration Types

```typescript
// GET /v1/providers response
interface ProvidersResponse {
  providers: Array<{
    id: string;       // "tmobile", "att", "verizon", "ubereats", "apple-review"
    name: string;     // "T-Mobile", "AT&T", "Verizon", "Uber Eats", "Apple Review"
    resources: Array<{
      name: string;   // "paymentMethod", "mobileBillingStatement", "orderHistory", "readHistory"
      operations: Record<string, {
        description: string;
        arguments: JSONSchema | null;   // null = no arguments needed
        result: JSONSchema | null;      // null = no schema defined
      }>;
    }>;
  }>;
}
```

## Result Schemas by Resource

### paymentMethod — read

```typescript
interface PaymentMethodReadResult {
  paymentMethods: Array<{
    order: number;
    vendorId: string;
    last4: string;
    type: "credit_card" | "debit_card" | "bank_account" | "other";
    isDefault: boolean;
    brand?: string;
    expirationDate?: string;
    nickname?: string;
    billingZip?: string;
    billingAddress?: Record<string, unknown>;
    raw: unknown;   // provider-specific raw data
  }>;
}
```

### paymentMethod — write

```typescript
// Required arguments
interface PaymentMethodWriteArgs {
  cardNumber: string;          // full card number
  expirationDate: string;     // MM/YY format
  cvv: string;                // CVV/security code
  nameOnCard: string;         // name as it appears on card
  billingZip: string;         // billing ZIP code
  billingAddress?: {           // optional full address
    line1: string;
    line2?: string;
    city: string;
    state: string;
    country?: string;
  };
  dryRun?: boolean;           // if true, validate without submitting
}

// Result is a discriminated union
type PaymentMethodWriteResult =
  | { dryRun: true; card: { last4: string; nickname?: string } }
  | { success: true; card: { last4: string } };
```

### mobileBillingStatement — read

```typescript
interface BillingReadArgs {
  limit?: number;   // max statements to return (positive integer)
}

interface BillingReadResult {
  billing: Array<{
    statementId?: string;
    periodStart: string;   // required
    periodEnd?: string;
    dueDate?: string;
    amountDue: number;     // required, finite
    currency: string;      // required, e.g. "USD"
  }>;
}
```

### orderHistory — read (Uber Eats)

```typescript
interface OrderHistoryArgs {
  limit?: number;   // positive integer
}

interface OrderHistoryResult {
  orderCount: number;
  orders: unknown;   // provider-specific structure
}
```

### readHistory — read (Apple Review)

```typescript
interface ReadHistoryArgs {
  limit?: number;   // positive integer
}

interface ReadHistoryResult {
  history: Array<{
    id: string;
    title: string;
    source: string;
    duration: number;
    readAt: string;
  }>;
}
```

## Error Response Shape

```typescript
// Standard error (all endpoints)
interface ErrorResponse {
  error: string;
}

// Validation error (400)
interface ValidationErrorResponse {
  error: "Validation failed";
  issues: Array<{
    path: string;      // dot-separated field path
    message: string;   // human-readable validation message
  }>;
}
```
