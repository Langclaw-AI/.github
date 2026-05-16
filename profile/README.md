# Langclaw

**Autonomous AI research platform for verifiable trend intelligence on 0G.**

Langclaw is a wallet-authenticated agent platform: chat, memory, API keys, prepaid usage billing, scheduled automation, and an OpenAI-compatible proxy to **0G Compute Router**. Its research engine **SignalGraph** turns one topic into live X, GitHub, Docs, and HackQuest signals through coordinated agents, stores evidence on **0G Storage**, and anchors brief hashes on **0G Chain**.

> **Langclaw** = product (dashboard + API + billing). **SignalGraph** = multi-agent research engine in the backend. **Two on-chain contracts:** `LangclawUsageVault` (deposits) and `SignalGraphRegistry` (research proof).

## Repositories

Organization: [**Langclaw-AI**](https://github.com/Langclaw-AI)

| Repository | Stack | Port | README |
| ---------- | ----- | ---- | ------ |
| [frontend](https://github.com/Langclaw-AI/frontend) | Next.js 16, React 19, RainbowKit, AI SDK | 3000 | [README](https://github.com/Langclaw-AI/frontend/blob/master/README.md) |
| [backend](https://github.com/Langclaw-AI/backend) | Node HTTP API, TypeScript, Supabase | 3001 | [README](https://github.com/Langclaw-AI/backend/blob/main/README.md) |
| [contracts](https://github.com/Langclaw-AI/contracts) | Foundry, `LangclawUsageVault` | — | [README](https://github.com/Langclaw-AI/contracts/blob/master/README.md) |

## Quick start

Clone the three repos (sibling folders work well locally):

```bash
git clone https://github.com/Langclaw-AI/backend.git
git clone https://github.com/Langclaw-AI/frontend.git
git clone https://github.com/Langclaw-AI/contracts.git
```

```bash
# 1. Backend
cd backend && cp .env.example .env
# Set SUPABASE_*, LANGCLAW_API_KEY_PEPPER, OG_COMPUTE_API_KEY at minimum
npm install && npm run dev

# 2. Frontend (new terminal)
cd frontend && cp .env.example .env.local
pnpm install && pnpm dev
```

```bash
curl http://localhost:3001/health
# {"ok":true,"service":"signalgraph-backend"}
```

Open [http://localhost:3000](http://localhost:3000). Connect a wallet and sign the Langclaw login message.

## Architecture

```text
User → Frontend (:3000) → Backend API (:3001)
                              ├─ Supabase (sessions, memory, keys, usage ledger)
                              ├─ OpenClaw CLI (Planner, Trend, Evidence, Verifier, Conclusion)
                              ├─ Providers (X/Brave, GitHub, Tavily, HackQuest)
                              ├─ 0G Compute Router (/v1/*)
                              ├─ 0G Storage (evidence bundles)
                              ├─ SignalGraphRegistry (brief hash)
                              └─ LangclawUsageVault (prepaid 0G deposits)
```

### SignalGraph workflow

Entry: `runSignalGraphWorkflow(topic)` → `POST /api/discover` or `/api/discover/stream` (NDJSON).

| Step | Runtime | Responsibility |
| ---- | ------- | -------------- |
| Planner | OpenClaw | Search plan for X, GitHub, Docs, HackQuest |
| Discovery | TypeScript | Live source fetch (API keys server-side) |
| Source normalizer | TypeScript | Source cards and excerpts |
| Trend scorer | OpenClaw | Rank trends |
| Evidence packager | OpenClaw | Claim map, bundle summary |
| Verifier | OpenClaw | Unsupported claims, brief hash input |
| Final conclusion | 0G Compute → OpenClaw → fallback | User-facing answer |
| 0G Storage / Chain | TypeScript | Upload bundle; `registerBrief` on registry |

OpenClaw skills: [backend/openclaw/skills](https://github.com/Langclaw-AI/backend/tree/main/openclaw/skills). Install: [OpenClaw docs](https://docs.openclaw.ai/install).

### Product features

| Feature | UI | API |
| ------- | -- | --- |
| Chat | `/chat` | `POST /api/chat/stream`, `/api/chat/sessions` |
| Research | Chat (agent mode) | `POST /api/discover`, `/api/discover/stream` |
| Memory | `/memory` | `POST /api/memory` |
| API keys | `/key` | `POST /api/api-keys` |
| Usage / billing | `/usage` | `POST /api/usage/*` |
| Automation | `/task`, `/settings` | `POST /api/automation/*` |
| 0G models | — | `GET/POST /v1/*` (alias `/api/0g/*`) |

## Authentication

**Wallet session** — sign:

```text
Login to Langclaw
Address: 0x...
Time: 2026-05-16T12:00:00.000Z
```

Required for API key management, deposits, withdrawals.

**API key** — `Authorization: Bearer lck_live_...` for runtime APIs. Cannot manage keys or withdraw.

Full spec: [API_REFERENCE.md](https://github.com/Langclaw-AI/backend/blob/main/docs/API_REFERENCE.md).

## Smart contracts (0G mainnet, chain ID `16661`)

| Contract | Repository | Purpose |
| -------- | ---------- | ------- |
| **LangclawUsageVault** | [contracts](https://github.com/Langclaw-AI/contracts) | Native 0G deposits; backend-authorized withdrawals |
| **SignalGraphRegistry** | [backend/contracts](https://github.com/Langclaw-AI/backend/blob/main/contracts/SignalGraphRegistry.sol) | `registerBrief(hash, storageUri)` for research proof |

**Billing flow:** user deposits 0G to vault → `POST /api/usage/deposit/verify` credits off-chain neuron balance → usage charges deduct ledger. Router (`OG_COMPUTE_API_KEY`) is funded separately on [pc.0g.ai](https://pc.0g.ai).

**Withdrawal:** backend calls `authorizeWithdrawal` on vault (TBD: full automation); user calls `withdraw(amount)`.

Deploy vault:

```bash
git clone https://github.com/Langclaw-AI/contracts.git && cd contracts
export LANGCLAW_USAGE_VAULT_OWNER=0x...
export LANGCLAW_USAGE_VAULT_WITHDRAWAL_AUTHORITY=0x...
forge script script/DeployLangclawUsageVault.s.sol:DeployLangclawUsageVaultScript \
  --rpc-url https://evmrpc.0g.ai --broadcast
```

Deploy registry (in the [backend](https://github.com/Langclaw-AI/backend) repo):

```bash
git clone https://github.com/Langclaw-AI/backend.git && cd backend
npm run deploy:registry
```

Set `LANGCLAW_USAGE_VAULT_ADDRESS` and `SIGNALGRAPH_REGISTRY_ADDRESS` in [backend `.env`](https://github.com/Langclaw-AI/backend/blob/main/.env.example).

## Deployment

| Service | Notes |
| ------- | ----- |
| Frontend | Set `NEXT_PUBLIC_SIGNALGRAPH_API_URL` to public API |
| Backend | `npm run build && npm start`; `PORT` default 3001 |
| Supabase | Apply [backend migrations](https://github.com/Langclaw-AI/backend/tree/main/supabase/migrations) |
| 0G | `OG_COMPUTE_API_KEY`, storage/chain keys as needed |

**Smoke tests after deploy**

1. `GET /health`
2. `GET /v1/models` with API key
3. Wallet connect + chat stream
4. `POST /api/discover` with short topic
5. Deposit to vault + `POST /api/usage/deposit/verify`

See env templates: [backend](https://github.com/Langclaw-AI/backend/blob/main/.env.example), [frontend](https://github.com/Langclaw-AI/frontend/blob/master/.env.example), [contracts](https://github.com/Langclaw-AI/contracts) (see deploy section in contracts README).

## Security

- Provider keys, Supabase service role, pepper, and `OG_*_PRIVATE_KEY` stay on the server only.
- API keys stored as HMAC hashes (`LANGCLAW_API_KEY_PEPPER`).
- Vault `withdrawalAuthority` is a hot wallet — use multisig for `owner` in production.
- Never commit `.env` files.

Spec for vault requirements: [SMART_CONTRACT_TEAM_NOTES.md](https://github.com/Langclaw-AI/backend/blob/main/docs/SMART_CONTRACT_TEAM_NOTES.md).

## Prerequisites

- Node.js 20+, pnpm (frontend), npm (backend)
- Supabase project
- Optional: [OpenClaw CLI](https://docs.openclaw.ai/install), [Foundry](https://book.getfoundry.sh/), 0G API keys and funded wallets

## Additional docs ([backend](https://github.com/Langclaw-AI/backend))

| Document | Description |
| -------- | ----------- |
| [API_REFERENCE.md](https://github.com/Langclaw-AI/backend/blob/main/docs/API_REFERENCE.md) | Full HTTP API |
| [SIGNALGRAPH_BLUEPRINT.md](https://github.com/Langclaw-AI/backend/blob/main/SIGNALGRAPH_BLUEPRINT.md) | Hackathon blueprint |
| [DEMO_SCRIPT.md](https://github.com/Langclaw-AI/backend/blob/main/docs/DEMO_SCRIPT.md) | 3-minute demo script |
| [openclaw/README.md](https://github.com/Langclaw-AI/backend/blob/main/openclaw/README.md) | OpenClaw workspace |

## Links

- GitHub: https://github.com/Langclaw-AI
- 0G docs: https://docs.0g.ai/
- 0G Compute Router: https://docs.0g.ai/developer-hub/building-on-0g/compute-network/router/overview
- 0G APAC Hackathon: https://www.hackquest.io/hackathons/0G-APAC-Hackathon

## License

TBD
