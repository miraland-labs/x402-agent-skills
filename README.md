# x402 Agent Skills (pr402 integrator)

Public [Agent Skills](https://agentskills.io/specification) for integrating with **pr402** on Solana — monetizing HTTP APIs as a seller or paying for resources as a buyer agent.

This repo contains the **pr402** router plus two specialist integrator skills. Protocol-dev skills stay in [miraland-labs/x402](https://github.com/miraland-labs/x402) under `.cursor/skills/`.

## Install

```bash
npx skills add miraland-labs/x402-agent-skills --skill pr402 -y
npx skills add miraland-labs/x402-agent-skills --skill pr402-seller -y
npx skills add miraland-labs/x402-agent-skills --skill pr402-buyer -y
```

List skills:

```bash
npx skills add miraland-labs/x402-agent-skills --list
```

After install, skills appear in your agent directory (e.g. `.cursor/skills/`). Listings on [skills.sh](https://www.skills.sh/) use `skills.sh.json` at repo root.

## Skills

| Skill | Audience | Starter repo |
| --- | --- | --- |
| `pr402` | Entry / router — pick seller vs buyer, shared URLs | — |
| `pr402-seller` | HTTP API providers monetizing via x402 + pr402 | [x402-seller-starter](https://github.com/miraland-labs/x402-seller-starter) |
| `pr402-buyer` | Paying agents, MCP hosts, autonomous buyers | [x402-buyer-starter](https://github.com/miraland-labs/x402-buyer-starter) |

## Production defaults

| | URL |
| --- | --- |
| Facilitator (Mainnet) | `https://ipay.sh` |
| Facilitator (Devnet) | `https://preview.ipay.sh` |
| pr402 source | [miralandlabs/pr402](https://github.com/miralandlabs/pr402) |
| Live runbook | [ipay.sh/agent-integration.md](https://ipay.sh/agent-integration.md) |

## Repo layout

```
skills.sh.json
skills/
  pr402/            SKILL.md (router — start here if role unclear)
  pr402-seller/     SKILL.md + references/
  pr402-buyer/      SKILL.md + references/
```

## Source of truth

**Public release:** this repo. Users install from here (`npx skills add miraland-labs/x402-agent-skills`).

**Development mirror:** [miraland-labs/x402](https://github.com/miraland-labs/x402) keeps a copy under `skills/` for hub docs and local Cursor symlinks. Edit in either place, then sync both before tagging a release here.

See [LICENSE](LICENSE).
