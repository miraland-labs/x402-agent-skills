# x402-resources.json (Seller Resource Manifest)

Publishes **discoverable payable endpoints** to the pr402 facilitator index. Optional for accepting payments; required for **`GET /api/v1/facilitator/resources`** search.

## Typical location

Host at:

```
https://your-api.example.com/.well-known/x402-resources.json
```

Or any stable HTTPS URL referenced during enroll.

## Minimal shape

```json
{
  "schema": "x402/v1",
  "merchantWallet": "<seller_pubkey_base58>",
  "serviceUrl": "https://your-api.example.com",
  "resources": [
    {
      "resourceUrl": "https://your-api.example.com/api/premium",
      "description": "Premium JSON endpoint",
      "mimeType": "application/json",
      "scheme": "exact",
      "amount": "50000",
      "asset": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
    }
  ]
}
```

Adjust `asset` mint to match your rail (USDC mainnet mint shown; Devnet USDC differs — read from `GET /capabilities` or seller preview).

## Enroll commands

**x402-cli:**

```bash
node tools/x402-cli/dist/index.js enroll \
  --manifest ./x402-resources.json \
  --keypair ./seller-keypair.json \
  --facilitator https://ipay.sh/api/v1/facilitator
```

**Standalone script** (zero deps; source in [miraland-labs/x402](https://github.com/miraland-labs/x402/blob/main/tools/enroll.mjs)):

```bash
node tools/enroll.mjs \
  --manifest ./x402-resources.json \
  --keypair ./seller-keypair.json \
  --wallet <PUBKEY> \
  --facilitator https://ipay.sh/api/v1/facilitator
```

## Origin binding rules

- `merchantWallet` must be **Activated** on-chain
- Each `resourceUrl` host must match the registered merchant **`serviceUrl`** origin
- Enroll runs challenge → sign → register → optional live **probe** against the resource URL

## Flags

| Flag | Effect |
| --- | --- |
| `--no-listing` | Register resource metadata without public directory opt-in |
| `--no-probe` | Skip HTTP probe (offline / WIP endpoints) |
| `--dry-run` | Print actions without POST (enroll.mjs only) |

## Discovery for buyers

After enroll + listing opt-in, buyer agents search:

```
GET /api/v1/facilitator/resources?q=<tags>
```

Not `GET /providers` for resource search — see [miralandlabs/pr402 DISCOVERY.md](https://github.com/miralandlabs/pr402/blob/main/docs/DISCOVERY.md).

## Subscription routes

Recurring access patterns map to multiple resource URLs on the **`exact`** rail — see [SUBSCRIPTION_PATTERN.md](https://github.com/miraland-labs/x402/blob/main/SUBSCRIPTION_PATTERN.md) in the x402 hub repo.
