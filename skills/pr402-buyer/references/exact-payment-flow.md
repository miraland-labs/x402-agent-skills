# Exact payment flow (buyer)

Step-by-step protocol under `@pr402/client`, MCP, and hand-rolled implementations.

## 1. Discover facilitator capabilities

```bash
BASE="https://ipay.sh"
curl -sS "$BASE/api/v1/facilitator/capabilities" | jq .
curl -sS "$BASE/api/v1/facilitator/supported" | jq .
```

Devnet: set `BASE="https://preview.ipay.sh"`.

Confirm `features.unsignedExactPaymentTxBuild` (or equivalent) before relying on build endpoint.

## 2. Receive 402 from seller

Save:

- Full `paymentRequirements` / `PaymentRequired` object
- The chosen **`accepts[]`** entry (scheme, network, asset, amount, payTo, extra)

Header names (x402 v2):

| Header | Content |
| --- | --- |
| `PAYMENT-REQUIRED` | Base64-encoded PaymentRequired JSON |

Some sellers return JSON body only; both patterns exist in the wild.

## 3. Build unsigned transaction

```http
POST /api/v1/facilitator/build-exact-payment-tx
Content-Type: application/json

{
  "payer": "<buyer_pubkey>",
  "accepted": { ... one accepts[] line, scheme: "exact" ... },
  "resource": "<canonical resource URL>"
}
```

Optional: `skipSourceBalanceCheck`, `autoWrapSol` — see OpenAPI.

**Scheme rule:** if the 402 line says `v2:solana:exact`, normalize to **`exact`** in this request. pr402 accepts aliases on build but `verifyBodyTemplate` uses wire `exact`.

Response (camelCase JSON):

- `transaction` — base64 bincode unsigned `VersionedTransaction`
- `verifyBodyTemplate` — ready-made body for verify/settle
- `payerSignatureIndex` — which signer slot the payer must sign
- `notes[]` — signing order hints

## 4. Sign locally

1. Base64-decode `transaction` → bincode → `VersionedTransaction`
2. Sign at `payerSignatureIndex` with payer keypair
3. Re-encode signed tx to base64
4. Replace `verifyBodyTemplate.paymentPayload.payload.transaction` with signed value
5. Keep `paymentPayload.accepted` and `paymentRequirements` **identical** (same JSON object)

Do **not** add address lookup tables — pr402 rejects ALT-loaded txs today.

## 5. Call seller with proof (default — starter/SDK path)

Retry the original HTTP request with header:

```http
PAYMENT-SIGNATURE: <verifyBodyTemplate JSON string, or base64 of same>
```

This matches [x402-buyer-starter](https://github.com/miraland-labs/x402-buyer-starter), `@pr402/client` (`pr402-buy`, `X402AgentClient.fetchWithAutoPay`), MCP, and `createPay402Fetch` / `X402Client.buy`: the buyer sends proof; the **seller** runs facilitator verify/settle.

## 6. Seller verify/settle and response

On `PAYMENT-SIGNATURE`, the seller SDK (or gate) calls facilitator `/verify` and `/settle` with the proof body, then returns the protected resource.

On success, inspect:

```http
PAYMENT-RESPONSE: <base64 JSON>
```

Fields typically include `success`, `transaction` (signature), `network`, `payer`.

## Seller-side vs buyer-side settle

| Pattern | Who calls verify/settle | When to use |
| --- | --- | --- |
| **SDK / CLI / MCP / starter** (`pr402-buy`, MCP, `createPay402Fetch`, bash starter) | Buyer builds proof; **seller** verify/settle on `PAYMENT-SIGNATURE` | **Default** for HTTP 402 APIs |
| **Manual buyer verify/settle** | Buyer calls `/verify` + `/settle` before seller retry | Debugging, custom flows without seller gate |

Default to the SDK/starter path unless the user explicitly hand-rolls buyer-side settlement.

## Advanced: direct buyer verify/settle

Same JSON body for both calls — use when **not** relying on seller-side settlement (manual curl or facilitator debugging; not `pr402-buy` / MCP / starter default):

```bash
curl -sS -X POST "$BASE/api/v1/facilitator/verify" \
  -H "Content-Type: application/json" -d @verify-body.json

curl -sS -X POST "$BASE/api/v1/facilitator/settle" \
  -H "Content-Type: application/json" -d @verify-body.json
```

`/settle` is idempotent — safe to retry if the tx already landed on-chain.

Reuse `correlationId` / `X-Correlation-ID` from verify when auditing.

If the seller still requires a proof header after direct settle, retry the resource with `PAYMENT-SIGNATURE` as in step 5.

## Failure recovery

| Error | Action |
| --- | --- |
| Blockhash expired | Re-run build-exact-payment-tx, re-sign |
| "not supported … Approved assets" | Pick allowlisted `asset` or different seller rail |
| Recipient / payTo mismatch | Seller's `accepts[]` wrong — not a buyer wallet bug |
| HTTP 200 + `{ "error": "..." }` | Treat as failure (bash starter exits non-zero) |

## sla-escrow branch

Use `POST /build-sla-escrow-payment-tx` instead — requires `slaHash`, `oracleAuthority`, etc. Trust template `payTo` over stale 402 line. See oracles workspace protocol doc.

## Discover sellers without prior 402

```
GET /api/v1/facilitator/providers?limit=20
GET /api/v1/facilitator/resources?q=<tag>
```

Build `accepts[]` from directory metadata, then continue at step 3.
