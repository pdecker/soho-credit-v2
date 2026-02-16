# AgentCredit Protocol

Onchain credit system for AI agents. Stablecoin-powered micro-payments and everyday spend, backed by a yield-generating liquidity vault.

Built on **Base** (primary, ~$0.001/tx) and **Solana** (secondary, ~$0.00025/tx).

## Architecture

```
Agent ──API──→ Permission Engine ──→ MPC Co-Sign ──→ USDC Transfer ──→ Merchant
                   │                                       │
              Credit Check                            Fee (1-2%)
              KYA / Sanctions                              │
              Risk Score                                   ▼
              Merchant Verify                     Liquidity Vault
                                                 (Lenders earn yield)
```

**Payment flow:** Agent calls API → permission checks → MPC 2-of-2 signature → USDC sent from vault to merchant on Base/Solana → fee stays in vault as yield → agent repays weekly.

**Yield model:** Lenders deposit USDC into vault, receive shares. Merchant fees (1-2%) increase total vault assets, raising share price. When lenders withdraw, they get their principal + accumulated yield.

## Quick Start

```bash
npm install && cp .env.example .env
npm run migrate          # Create database
npm run seed             # Optional: test data ($100k vault, test agent + merchants)
npm run dev              # Start on :3000
```

## API Endpoints

**Agents:** `POST /api/v1/agents` (register), `GET /api/v1/agents/me` (profile)

**Payments:** `POST /api/v1/payments` (pay), `GET /api/v1/payments` (list), `GET /api/v1/payments/:id` (detail), `POST /api/v1/payments/repay` (repay), `GET /api/v1/payments/credit/summary` (credit line)

**Merchants:** `POST /api/v1/merchants` (register), `GET /api/v1/merchants` (list)

**Vault:** `GET /api/v1/vault` (stats), `POST /api/v1/vault/deposit` (lend), `POST /api/v1/vault/withdraw` (withdraw), `GET /api/v1/vault/positions/:address` (lender positions)

Auth: `Authorization: Bearer acp_xxx` on all authenticated endpoints.

## SDK Usage

```typescript
import { AgentCreditClient } from "@agentcredit/protocol/sdk";

const client = new AgentCreditClient({
  apiKey: "acp_...",
  mpcShard: "your-hex-shard",
  chain: "base",  // or "solana"
});

// Pay a merchant
await client.pay({ to: "0xMerchant...", amount: 25.00, memo: "API credits" });

// Check credit
const credit = await client.getCreditSummary();

// Repay
await client.repay(100, "0xTxHash...");
```

## Smart Contracts

**Base (Solidity):** `contracts/AgentCreditVault.sol` — ERC-4626-style vault with credit management, disbursement, and repayment. Uses OpenZeppelin AccessControl + ReentrancyGuard.

**Solana (Anchor):** `contracts/solana/agent_credit_vault.rs` — PDA-based vault with SPL token transfers, credit accounts, and lender positions.

Both contracts handle: deposits/withdrawals with share accounting, USDC disbursement with fee capture, agent credit limits, utilization caps, and pause functionality.

## Permission Checks (every payment)

1. **Credit check** — sufficient available credit for the amount
2. **Sanctions screening** — recipient address checked against OFAC/Chainalysis
3. **Merchant verification** — recipient is approved and active
4. **KYA (Know Your Agent)** — agent passed identity verification
5. **Risk score** — agent's behavioral risk score below threshold

All five must pass. Failures return the specific reasons.

## Project Structure

```
src/
  api/          Routes (agents, payments, merchants, vault)
  chains/       Base + Solana providers (multi-chain abstraction)
  compliance/   KYA, sanctions screening, risk scoring
  config/       Environment config with Zod validation
  core/         Payment engine, agent management, scheduler
  db/           SQLite schema, queries, migrations
  middleware/   Auth (API key + JWT), request tracking
  mpc/          2-of-2 threshold signature engine
  sdk/          Drop-in TypeScript client for agents
  types/        Shared type definitions + Zod schemas
  vault/        Liquidity pool, yield calculation, share pricing
contracts/
  AgentCreditVault.sol          Base (EVM)
  solana/agent_credit_vault.rs  Solana (Anchor)
```

## Deployment

```bash
npm run build
docker compose up -d    # API + Redis
```

Set `VAULT_CONTRACT` in .env after deploying the smart contract to Base or Solana.
