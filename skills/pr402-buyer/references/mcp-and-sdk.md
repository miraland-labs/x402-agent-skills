# MCP, SDK, and CLI paths

## MCP — Cursor / Claude Desktop (fastest for agents)

**Package:** `@pr402/mcp-server` (stdio over `@pr402/client`)

```bash
npx -y @pr402/mcp-server
```

**`.cursor/mcp.json` example** (from [x402-buyer-starter](https://github.com/miraland-labs/x402-buyer-starter/blob/main/examples/mcp/cursor-mcp.json)):

```json
{
  "mcpServers": {
    "pr402": {
      "command": "npx",
      "args": ["-y", "@pr402/mcp-server"],
      "env": {
        "PR402_FACILITATOR_URL": "https://ipay.sh",
        "PR402_PAYER_KEYPAIR_JSON": "/absolute/path/to/buyer-keypair.json"
      }
    }
  }
}
```

**Buyer tools** (catalog: `GET /agent-tools.json` on facilitator):

- `pr402_get_capabilities`
- `pr402_build_exact_payment`
- `pr402_pay_http_resource` — end-to-end paid HTTP fetch

**Seller tools** (same server — useful when agent helps onboard):

- `pr402_seller_preview`, `pr402_seller_rail_info`, `pr402_seller_provision_tx`, `pr402_enrich_payment_required`

Devnet testing: set `PR402_FACILITATOR_URL` to `https://preview.ipay.sh` and use a funded Devnet keypair.

Source: [miralandlabs/pr402/sdk/mcp](https://github.com/miralandlabs/pr402/tree/main/sdk/mcp)

## TypeScript SDK

**Published:** `@pr402/buyer-typescript` (starter) · `@pr402/client` (facilitator client + CLI)

```bash
npm install @pr402/buyer-typescript
# or
npm install @pr402/client
```

**High-level:**

```typescript
const fortune = await client.buy("https://seller.example/api/v1/resource", { key: "value" });
```

**fetch wrapper:**

```typescript
import { createPay402Fetch } from "@pr402/buyer-typescript";
const payFetch = createPay402Fetch(fetch, { payer, defaultFacilitatorBaseUrl });
const res = await payFetch(url, init);
```

Starter demos: clone [miraland-labs/x402-buyer-starter](https://github.com/miraland-labs/x402-buyer-starter), then `cd typescript && npm start`.

## Global CLI

```bash
npm i -g @pr402/client
pr402-buy --resource <url> --payer /path/keypair.json --mint <mint>
```

Rust equivalent: `cargo install pr402-client`

## Python

Starter: `x402-buyer-starter/python` — `pip install -r requirements.txt && python index.py`

LangChain: `pip install langchain-pr402` → `X402GetTool` / `X402PostTool`

## Bash (minimal)

`x402-buyer-starter/bash` — curl + Node `sign.js` for Ed25519.

Conventions:

- `set -e`
- Top-level JSON `"error"` → non-zero exit
- `node --no-deprecation sign.js` to suppress punycode noise
- `PR402_FACILITATOR_URL` overrides default (`https://ipay.sh` in production examples)

## Environment variables

| Variable | Purpose |
| --- | --- |
| `PR402_FACILITATOR_URL` | Facilitator origin (no trailing path required in most SDKs) |
| `PR402_PAYER_KEYPAIR_JSON` | Absolute path to payer keypair (MCP) |
| `SOLANA_RPC_URL` | Optional override (prefer facilitator health RPC) |

## Forge MCP (marketplace — separate from pr402 HTTP pay)

Digital goods on [http402.trade](https://http402.trade):

```bash
npx -y @http402/forge-mcp
```

Example config: `x402-buyer-starter/examples/mcp/forge-cursor-mcp.json`

Tools: `forge_list`, `forge_preview`, `forge_purchase`

See [forge-marketplace.md](forge-marketplace.md).
