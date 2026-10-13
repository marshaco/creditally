# CrediTally

**Spend control for AI agents, with the spending limit enforced on Solana.**

AI agents pay for data and services from a shared company wallet. Each agent has a daily spending limit enforced on-chain by the Squads program, so an agent can't overspend even if our own code has a bug. Every payment is tagged with the task it was for, and a live dashboard shows finance what each agent bought, what was blocked, and how much budget is left today.

CrediTally also tracks what agents spend on AI credits from Claude, OpenAI and Gemini. For each provider you set either a hard cap or a spend target, for example to use your free start-up credits before they expire.

Built in one day at the **BUILD IRL Solana hackathon**. Runs on **Solana devnet only**.

## How it works

```
 ┌────────────┐  1. GET /weather              ┌────────────┐
 │  AI agent  │ ────────────────────────────▶ │  Paid API  │
 │ (Claude /  │ ◀──────────────────────────── │  (HTTP 402)│
 │  OpenAI /  │  2. 402: price, payTo, reqId  └─────┬──────┘
 │  Gemini)   │                                     │ 5. checks the payment
 └─────┬──────┘  4. retry with tx signature ──────▶ │    on-chain, returns data
       │ 3. pay from its allowance                  │
       ▼                                            ▼
 ┌────────────────────────────────────────────────────────┐
 │ Solana: Squads v4 vault + per-agent spending limit     │
 │ memo "task:<taskId>|req:<requestId>" on every payment  │
 │ over-limit payments are rejected on-chain (error 6026) │
 └───────────────────────────┬────────────────────────────┘
                             │ polled every 5s
                             ▼
                   ┌───────────────────┐
                   │ Finance dashboard │
                   └───────────────────┘
```

1. **Shared vault.** Company funds sit in a Squads v4 multisig vault. On devnet these are our own test stablecoin (an SPL token shown as $1 per token).
2. **On-chain limit per agent.** An agent isn't a multisig member. It can only spend through a Squads spending limit, 100 tokens a day by default. If a payment goes over the limit, the Squads program rejects it, and the failed transaction is still visible in the explorer.
3. **Pay-per-request data API.** The API replies `402 Payment Required` with a price. The agent pays and retries with the transaction signature. Before returning data, the API checks on-chain that the transaction succeeded, paid at least the price to the provider in the right token, and carries a request ID that this API issued, and that the signature hasn't been used before.
4. **Real data:** weather (Open-Meteo, 10), stock quotes (Yahoo Finance, 20), trivia (Open Trivia DB, 5), and company financials (SEC EDGAR, 150). Financials cost more than the daily limit on purpose, so buying them always shows an on-chain rejection.
5. **AI credit tracking.** Every model call is logged with its real token cost, per task and per agent. Before each call, the agent checks the provider's hard cap and stops once it's reached.
6. **Dashboard.** Shows payments with explorer links, blocked attempts in red, budget left today read from the spending-limit account, AI credits per provider, and a per-task table with on-chain purchases next to AI cost.

### What runs on Solana and what doesn't

| Part | On-chain? |
|---|---|
| Holding funds, enforcing the per-agent limit, payment receipts tagged with the task | Yes (Squads v4 + SPL memo) |
| AI credit usage, caps and targets | No. Usage data lives with each AI provider. The hard cap only stops agents that check with CrediTally before each call. |

## Stack

TypeScript (Node 20+, `tsx`) · `@solana/web3.js` v1 · `@solana/spl-token` · Squads v4 (`@sqds/multisig`) · Express · Next.js 16 + Tailwind · Anthropic, OpenAI and Gemini APIs

## Repo layout

```
wallet-service/   payAsAgent(): Squads spendingLimitUse + SPL memo
paid-api/         server.js: the HTTP 402 data API with on-chain checks (index.ts: mock-data fallback)
agent/            agent with a buy_data tool (Claude, OpenAI or Gemini); credits.ts tracks AI usage and caps
dashboard/        Next.js dashboard (live summary, usage API, settings page)
scripts/          setup, add-agents, reset-limit, demo-start, test-payments
credits.config.json   credit grant, hard cap, minimum target and expiry for each AI provider
```

## Running it

Requires Node 20+ and a devnet RPC URL (the public devnet endpoint or a Helius devnet URL).

```bash
npm install
cd dashboard && npm install && cd ..
cp .env.example .env              # set RPC_URL and ANTHROPIC_API_KEY
```

Create the on-chain setup. `scripts/setup.ts` creates the token, the multisig, the funded vault and the agent's spending limit, and writes their addresses to `.env`. It can be re-run safely. The owner keypair (`keys/owner.json`) needs some devnet SOL first.

```bash
npx tsx scripts/gen-keys.ts       # keys/owner.json, keys/agent.json (gitignored)
npx tsx scripts/setup.ts
npx tsx scripts/add-agents.ts     # optional: OpenAI and Gemini agents (set OPENAI_API_KEY / GEMINI_API_KEY)
```

Start it, in three terminals:

```bash
npm run paid-api                  # http://localhost:3001
cd dashboard && npm run dev       # http://localhost:3000
npm run agent -- "What's AAPL trading at?"
```

Payments appear on the dashboard within about 5 seconds. More examples:

```bash
npm run agent -- --provider gemini "Weather in Dublin?"
npm run agent -- "Get Apple's latest annual financials"     # over the limit: rejected on-chain
npm run agent -- --provider openai --chat "Summarise ..."   # uses AI credits only, buys nothing
npm run demo:start                                          # refill the limit and clear the dashboard view
```

In **Settings** (`/settings`), paste a Squads multisig address. The app finds the vault, spending limit, token and agents on-chain. Then add your agents' public keys with names, and choose a hard limit or a spend target for agent purchases and for each AI provider.

### Example transactions (devnet)

- [Payment by the Claude agent](https://explorer.solana.com/tx/gccafQctuqbMMwLPUeG48psdm4sELjr6YuosiEfjR4LpPDgxhihhnU1bTNWpr84PGqrdMGgsJuAEakh6Fp9rbHX?cluster=devnet)
- [Payment blocked on-chain by the spending limit](https://explorer.solana.com/tx/ZM1vtWwKurLSkjjiwmranmaoZzHsYLtAGccCh7r9DBtws5UA4q6TD4bZJcUjmfhmPfnP5ch3EdTxixWMSKVQWh9?cluster=devnet)

## Limitations and next steps

- Devnet and a test token only. Mainnet with USDC is the next step.
- The AI credit hard cap needs the agent's cooperation. Enforcing it on any agent needs a proxy that all AI traffic goes through.
- Saving a new purchase limit in Settings signs the change with the owner key on the server. In production, this would be a Squads approval flow.
- Not built yet: alerts, multi-team support, an SDK package.

## Team

- **Colin Marshall:** Squads wallet and payment service, on-chain payment checks, integration, AI credit tracking, Settings, multi-provider agents
- **Finn Mitchell:** first version of the agent and its 402 payment loop
- **Oliwia Kędzierska:** paid data API with real data sources
- **Jakub Kieblesz:** dashboard design and first version
