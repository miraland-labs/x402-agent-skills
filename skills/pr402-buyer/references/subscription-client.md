# Subscription buyer flow

Buy **time-window access** on the **`exact`** rail — pay once per window, then use a JWT on data routes.

Buyers talk **only to the seller**. They never call [miralandlabs/subscription-auth](https://github.com/miralandlabs/subscription-auth) (seller-only Tier B infrastructure).

## Canonical repos (from each project's `origin`)

| Project | GitHub | Account |
| --- | --- | --- |
| Buyer SDK (`x402-subscription-client`) | [miraland-labs/x402-subscription-client](https://github.com/miraland-labs/x402-subscription-client) | org |
| Reference seller | [miraland-labs/x402-subscription-starter](https://github.com/miraland-labs/x402-subscription-starter) | org |
| Wire contract doc | [miraland-labs/x402](https://github.com/miraland-labs/x402) → `SUBSCRIPTION_PATTERN.md` | org (hub) |

Do not swap `miralandlabs` ↔ `miraland-labs` — they are different GitHub owners.

Prefer **`x402-subscription-client`** over hand-rolling. Payment proof logic lives in `src/pr402-exact-flow.ts`.

## Install

```bash
npm install x402-subscription-client
```

Or clone [miraland-labs/x402-subscription-client](https://github.com/miraland-labs/x402-subscription-client) for source/examples.

## Flow overview

```
1. GET  /api/v1/subscribe/info     → pick tier
2. POST /api/v1/subscribe?tier=…   → 402 → PAYMENT-SIGNATURE (exact rail)
3. Store token from success JSON
4. GET  /api/v1/premium-data       → Authorization: Bearer <token>
5. On TOKEN_EXPIRED / TOKEN_REVOKED → re-subscribe (SDK auto-renews — see below)
```

Step 2 payment half matches the **`exact` golden path** in the main skill — target `/subscribe`, not a per-call data route.

## Step 1 — Discover tiers

```http
GET /api/v1/subscribe/info
```

Read tier names (`hourly`, `daily`, `monthly`), prices, and covered endpoints.

## Step 2 — Subscribe (exact payment)

```http
POST /api/v1/subscribe?tier=monthly
```

Without payment: **HTTP 402** + `accepts[]`. Follow [exact-payment-flow.md](exact-payment-flow.md) steps 3–5 against this endpoint:

1. `POST /api/v1/facilitator/build-exact-payment-tx`
2. Sign `VersionedTransaction`
3. Retry `POST /subscribe` with **`PAYMENT-SIGNATURE`**

**pr402 1.2 note:** merge capabilities rail `extra` into both `paymentPayload.accepted.extra` and `paymentRequirements.extra` before sending proof (`buildExactPaymentProofJsonString` in the subscription client does this).

Do **not** call facilitator `/verify` or `/settle` directly — the seller handles settlement on `/subscribe`.

## Step 3 — Store JWT

Success response:

```json
{
  "success": true,
  "token": "<jwt>",
  "tier": "monthly",
  "expiresAt": "2026-07-23T12:00:00.000Z",
  "durationSeconds": 2592000,
  "usage": "Authorization: Bearer <token> on protected data routes",
  "persistenceHint": "..."
}
```

Persist `token` and `expiresAt`. Honor any `persistenceHint` from the seller.

## Step 4 — Data routes (Bearer)

```http
GET /api/v1/premium-data
Authorization: Bearer <jwt>
```

No `PAYMENT-SIGNATURE` on data routes.

## Step 5 — Expiry and renewal

| Error | Hand-rolled client | `X402SubscriptionClient` |
| --- | --- | --- |
| `401 TOKEN_EXPIRED` | Re-run subscribe (new exact payment) | **Auto-renews** via `subscribe()` on data calls |
| `401 TOKEN_REVOKED` | Re-subscribe if access needed | **Auto-renews** (new payment) |
| `401 TOKEN_INVALID` | Discard token; re-subscribe | Surfaces `SubscriptionApiError` |
| `403 TOKEN_SCOPE_MISMATCH` | Wrong endpoint scope | Surfaces `SubscriptionApiError` |

## x402-subscription-client quick start

```typescript
import { Keypair } from '@solana/web3.js';
import { X402SubscriptionClient } from 'x402-subscription-client';

const client = new X402SubscriptionClient({
  payerKeypair: buyerKeypair,
  endpointBaseUrl: 'http://127.0.0.1:3000',
  defaultFacilitatorUrl: 'https://preview.ipay.sh',
});

await client.subscribe('hourly');
const echo = await client.echo({ hello: 'world' });
```

Persist across restarts: `saveSubscriptionToFile()` / `loadSubscriptionFromFile()`.

Use this instead of `@pr402/client` / `createPay402Fetch` when the API is subscription-shaped (single `/subscribe` gate + Bearer data routes).

For **pay-per-call** APIs (402 on every resource), stay on the main buyer golden path — no JWT phase.

## TypeScript / MCP shortcut

If the user already uses `@pr402/buyer-typescript` or MCP for generic 402 APIs, subscription APIs still need the JWT lifecycle above. Point them at **miraland-labs/x402-subscription-client** or implement steps 1–5 manually.

## Related

- Wire contract: [SUBSCRIPTION_PATTERN.md](https://github.com/miraland-labs/x402/blob/main/SUBSCRIPTION_PATTERN.md)
- Seller setup: `pr402-seller` → [subscription-exact-rail.md](../../pr402-seller/references/subscription-exact-rail.md)
- Exact payment steps: [exact-payment-flow.md](exact-payment-flow.md)
- Tier B E2E: [x402-subscription-starter/examples/tier-b-preview-e2e](https://github.com/miraland-labs/x402-subscription-starter/tree/main/examples/tier-b-preview-e2e)
