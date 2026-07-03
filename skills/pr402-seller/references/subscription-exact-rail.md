# Subscription APIs on the exact rail

Time-window access (hourly, daily, monthly) on top of pr402 **`exact`** — **not** pay-per-call and **not** `sla-escrow`.

**Critical rule:** HTTP **402 only on `POST /api/v1/subscribe`**. Data routes use **`Authorization: Bearer`** only — never gate every request with x402.

Full wire contract: [SUBSCRIPTION_PATTERN.md](https://github.com/miraland-labs/x402/blob/main/SUBSCRIPTION_PATTERN.md) (x402 hub).

## Canonical repos (from each project's `origin`)

| Project | GitHub | Account |
| --- | --- | --- |
| Subscription seller starter | [miraland-labs/x402-subscription-starter](https://github.com/miraland-labs/x402-subscription-starter) | org |
| Seller SDK (`@pr402/subscription-seller`) | [miraland-labs/x402-subscription-seller](https://github.com/miraland-labs/x402-subscription-seller) | org |
| Hosted JWT auth (Tier B) | [miralandlabs/subscription-auth](https://github.com/miralandlabs/subscription-auth) | individual |
| Wire contract doc | [miraland-labs/x402](https://github.com/miraland-labs/x402) → `SUBSCRIPTION_PATTERN.md` | org (hub) |

Do not swap `miralandlabs` ↔ `miraland-labs` — they are different GitHub owners.

## When to use this path

| User wants | Use |
| --- | --- |
| Pay once → access for N hours/days | This reference + [x402-subscription-starter](https://github.com/miraland-labs/x402-subscription-starter) |
| Pay on every API call | [runtime-sdk.md](runtime-sdk.md) + [x402-seller-starter](https://github.com/miraland-labs/x402-seller-starter) |
| Conditional delivery / oracle | Stop — `sla-escrow` + [oracles Seller Guide](https://github.com/miraland-labs/oracles/blob/main/docs/SELLER_GUIDE.md) |

## Wire contract (seller)

### Discovery

- `GET /api/v1/subscribe/info` — tiers, pricing, covered endpoints
- `GET /.well-known/x402-resources.json` — optional public catalog

### Purchase (only x402-gated route)

```
POST /api/v1/subscribe?tier=hourly|daily|monthly
```

| Request | Response |
| --- | --- |
| No valid `PAYMENT-SIGNATURE` | **HTTP 402** + `accepts[]` envelope |
| Valid `PAYMENT-SIGNATURE` | Seller verify/settle via pr402 → return JWT |

Success body (example):

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

### Data routes

```
GET /api/v1/premium-data
Authorization: Bearer <jwt>
```

Return standard errors: `MISSING_TOKEN`, `TOKEN_EXPIRED`, `TOKEN_REVOKED`, `TOKEN_INVALID` (401); `TOKEN_SCOPE_MISMATCH` (403).

### JWT claims (canonical)

```json
{
  "sub": "api.example.com",
  "payer": "<buyer-wallet-pubkey>",
  "tier": "monthly",
  "resources": ["*"],
  "jti": "<uuid>",
  "iat": 1718270000,
  "exp": 1718273600
}
```

Rate-limit by `payer` claim after JWT validation.

## Starter and SDK

Fork **[x402-subscription-starter](https://github.com/miraland-labs/x402-subscription-starter)** — not `x402-seller-starter`.

Seller SDK source + npm: **[x402-subscription-seller](https://github.com/miraland-labs/x402-subscription-seller)** → `npm install @pr402/subscription-seller`

Shared pr402 env (same contract as pay-per-call): `FACILITATOR_BASE_URL`, `MERCHANT_WALLET`, `X402_*` (see starter `.env.example` — e.g. `X402_ACCEPTS_EXTRA_JSON`).

## Tier A vs Tier B (JWT auth)

Payment is identical in both tiers. Only **JWT signing/validation** differs.

| | **Tier A — local** | **Tier B — hosted auth** |
| --- | --- | --- |
| **`SUBSCRIPTION_MODE`** | `local` (default) | `tier-b` |
| **When** | Single seller, fastest fork | RS256/JWKS, shared revocation |
| **Sign JWT** | Local `JWT_SECRET` (HS256) | [miralandlabs/subscription-auth](https://github.com/miralandlabs/subscription-auth) RS256 |
| **Verify JWT** | Your server | Your server via JWKS |
| **Revoke early** | Local DB (`StrictStore` recommended) | Poll revocation feed ~60s (fail-open default) |
| **Register auth** | No | **Once** at deploy |

Tier A minimum env: `JWT_SECRET`, `FACILITATOR_BASE_URL`, `MERCHANT_WALLET`, `X402_*`.

Tier B seller env (after one-time service registration on **miralandlabs/subscription-auth**):

```bash
SUBSCRIPTION_MODE=tier-b
SUBSCRIPTION_AUTH_BASE_URL=https://auth.ipay.sh          # devnet tests: https://preview.auth.ipay.sh
SUBSCRIPTION_AUTH_ISS=https://auth.ipay.sh
SUBSCRIPTION_AUTH_SERVICE_ID=api.myproduct.com
SUBSCRIPTION_AUTH_MERCHANT_SECRET_KEY=<base58_64_byte_secret>
REVOCATION_POLL_INTERVAL_SEC=60
```

After pr402 settle on `/subscribe`, seller wallet-signs challenge → `POST /v1/tokens/issue` on auth service → return RS256 JWT to buyer.

**Tier B setup docs:** [SUBSCRIPTION_AUTH_FOR_SELLERS.md](https://github.com/miralandlabs/subscription-auth/blob/main/docs/SUBSCRIPTION_AUTH_FOR_SELLERS.md)

**Do not** expose subscription-auth to buyers — they only talk to your seller.

## Seller checklist

1. [ ] `/api/v1/subscribe/info` lists tiers and prices
2. [ ] **402 + pr402 settle only on `/subscribe`** — not on data routes
3. [ ] Data routes: Bearer JWT only
4. [ ] Pick Tier A (`SUBSCRIPTION_MODE=local` + `JWT_SECRET`) or Tier B (`tier-b` + auth registration)
5. [ ] Return `persistenceHint` on subscribe success (starter convention)
6. [ ] Rate limits keyed on JWT `payer` claim

## E2E test pointers

| Stack | Command / path |
| --- | --- |
| Tier B auth-only (no USDC) | [miralandlabs/subscription-auth](https://github.com/miralandlabs/subscription-auth) → `scripts/e2e-tier-b-auth.mjs` |
| Full pr402 + seller + buyer | [tier-b-preview-e2e](https://github.com/miraland-labs/x402-subscription-starter/tree/main/examples/tier-b-preview-e2e) |

## Related

- **Buyer side:** `pr402-buyer` → [subscription-client.md](../../pr402-buyer/references/subscription-client.md)
- **Pay-per-call seller:** [runtime-sdk.md](runtime-sdk.md)
- **Manifest / discovery:** [x402-resources-manifest.md](x402-resources-manifest.md)
