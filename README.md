# @stellar-ai-payments/sdk

[![CI](https://github.com/Devcyprian/stellar-ai-payments-sdk/actions/workflows/ci.yml/badge.svg)](https://github.com/Devcyprian/stellar-ai-payments-sdk/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Stellar Wave](https://img.shields.io/badge/Stellar-Wave%205-7b2d8b)](https://communityfund.stellar.org)
[![npm](https://img.shields.io/badge/npm-%40stellar--ai--payments%2Fsdk-blue)](https://www.npmjs.com/package/@stellar-ai-payments/sdk)

> **Alpha TypeScript SDK enabling AI agents to prepare and integrate micropayments on the Stellar network — including x402, Multi-Party Payments (MPP), session keys, and fee sponsorship interfaces.**

> **Status:** this package is under active development and is not audited. API
> compatibility, contract integrations, and production credential handling may
> change between releases. Use testnet credentials only until the security and
> integration review is complete.

---

## Why This Matters for the Stellar Ecosystem

The rise of autonomous AI agents creates an urgent need for machine-native payment infrastructure. Today, AI agents cannot easily pay for APIs, split revenue with collaborators, or operate without holding XLM for transaction fees. This SDK solves all three problems directly on Stellar:

- **x402 protocol** — Agents pay per API request using HTTP 402, the standard Stellar and Coinbase have adopted for agentic payments
- **MPP (Machine Payment Protocol)** — Atomic multi-party splits so agents can pay multiple recipients in one transaction
- **Session keys** — Short-lived, scoped signing keys so agents never hold long-lived secrets
- **Fee sponsorship** — Agents transact with zero XLM balance; a sponsor vault covers fees

This SDK is the foundational TypeScript layer for the entire `stellar-ai-payments` ecosystem, consumed by the adapters package and referenced by the documentation site.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     AI Agent / Application                      │
├──────────────────────────────────────────────────────────────── ┤
│                  @stellar-ai-payments/sdk                       │
│                                                                 │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────┐  ┌──────────┐ │
│  │  x402       │  │   MPP    │  │  Session    │  │   Fee    │ │
│  │  Protocol   │  │ Payments │  │    Keys     │  │ Sponsor  │ │
│  │  Client     │  │ Builder  │  │  Manager   │  │  Vault   │ │
│  └──────┬──────┘  └────┬─────┘  └──────┬──────┘  └────┬─────┘ │
│         └──────────────┴───────────────┴───────────────┘       │
│                              │                                  │
│                   @stellar/stellar-sdk                          │
└──────────────────────────────┼──────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │       Stellar Network            │
              │  Horizon API  │  Soroban RPC     │
              │  Testnet / Mainnet               │
              └─────────────────────────────────┘
```

---

## Repository Structure

```
stellar-ai-payments-sdk/
├── src/
│   ├── x402/
│   │   ├── client.ts          # HTTP 402 protocol client, retry, backoff, proof encoding
│   │   └── index.ts
│   ├── mpp/
│   │   └── index.ts           # Multi-party payment builder, asset parser, fee calculator
│   ├── account/
│   │   └── index.ts           # Account helpers, Friendbot, balance queries, reserve calc
│   ├── session/
│   │   └── index.ts           # Session key manager: create, validate, revoke, extend, TTL
│   ├── sponsorship/
│   │   └── index.ts           # Fee-bump envelope builder, estimateFeeBump
│   ├── version.ts             # SDK_VERSION export
│   └── index.ts               # Single entry point — re-exports all modules
├── tests/
│   ├── x402.test.ts           # x402 client, backoff, memo validation, error types
│   ├── mpp.test.ts            # Asset parsing, split validation, fee calculation
│   ├── account.test.ts        # Account helpers, balance mapping, reserve calculation
│   └── session.test.ts        # Session key lifecycle, validation, TTL, label lookup
├── .github/
│   ├── workflows/ci.yml       # Lint → Test → Build on push/PR to main and develop
│   ├── CODEOWNERS             # @Devcyprian
│   └── ISSUE_TEMPLATE/
│       └── stellar_wave_task.md
├── .env.example
├── .eslintrc.js
├── jest.config.js
├── tsconfig.json
├── package.json               # Ready for npm publish as @stellar-ai-payments/sdk
├── CONTRIBUTING.md
├── SECURITY.md
├── CHANGELOG.md
└── LICENSE                    # MIT
```

---

## Installation

```bash
npm install @stellar-ai-payments/sdk
```

Copy environment config:

```bash
cp .env.example .env
```

```env
STELLAR_NETWORK=testnet
HORIZON_URL=https://horizon-testnet.stellar.org
AGENT_SECRET_KEY=S...
SPONSOR_SECRET_KEY=S...
```

---

## Usage

### x402 — Pay-per-request API access

```typescript
import { X402Client } from '@stellar-ai-payments/sdk';

const client = new X402Client();

// Automatically handles 402 → pay → retry
const response = await client.fetch(
  'https://api.example.com/premium-data',
  async (details) => {
    // details: { amount, asset, destination, memo, expiresAt }
    const txHash = await submitStellarPayment(details);
    return { txHash, ledger: 1, paidAt: Date.now() };
  }
);
```

### MPP — Split payments across multiple recipients

```typescript
import { buildMppTransaction, validateSplits } from '@stellar-ai-payments/sdk';
import { Keypair } from '@stellar/stellar-sdk';

const { valid, errors } = validateSplits(splits);
if (!valid) throw new Error(errors.join(', '));

const { xdr, totalAmount, splitCount } = buildMppTransaction({
  sourceKeypair: Keypair.fromSecret(process.env.AGENT_SECRET_KEY!),
  sourceAccount: account,
  splits: [
    { destination: 'GDEST1...', amount: '7' },  // 70%
    { destination: 'GDEST2...', amount: '3' },  // 30%
  ],
});
// Submit xdr to Horizon
```

### Session Keys — Scoped, time-limited agent signing

```typescript
import { SessionKeyManager } from '@stellar-ai-payments/sdk';

const mgr = new SessionKeyManager();

// Create a key valid for 5 minutes, max 5 XLM per tx, restricted destinations
const key = mgr.create({
  ttlSeconds: 300,
  maxAmountPerTx: '5',
  allowedDestinations: ['GDEST1...', 'GDEST2...'],
  label: 'my-agent-session',
});

// Validate before every signing operation
const { valid, reason } = mgr.validate(key.id, destination, amount);
if (!valid) throw new Error(reason);

// Revoke immediately after use
mgr.revoke(key.id);
```

### Fee Sponsorship — Zero-fee agent transactions

```typescript
import { buildFeeBump, estimateFeeBump } from '@stellar-ai-payments/sdk';
import { Keypair } from '@stellar/stellar-sdk';

// Estimate cost before building
const fee = estimateFeeBump(operationCount, 100); // stroops

// Wrap agent's signed tx in a fee-bump envelope
const { xdr, feeAccount, innerTxHash } = buildFeeBump({
  sponsorKeypair: Keypair.fromSecret(process.env.SPONSOR_SECRET_KEY!),
  innerTxXdr: agentSignedTxXdr,
  baseFee: '200',
});
// Submit xdr — sponsor pays, agent pays nothing
```

### Account Helpers

```typescript
import { StellarAccountHelper, NETWORKS } from '@stellar-ai-payments/sdk';

const helper = new StellarAccountHelper(NETWORKS.TESTNET.horizonUrl);

// Generate keypair
const { publicKey, secretKey } = StellarAccountHelper.generateKeypair();

// Fund on testnet
await helper.fundTestnet(publicKey);

// Check balance
const xlm = await helper.getXlmBalance(publicKey);
const usdc = await helper.getAssetBalance(publicKey, 'USDC:GISSUER...');

// Guard minimum reserve
await helper.requireXlmBalance(publicKey, '2'); // throws if < 2 XLM

// Calculate minimum reserve
const minXlm = StellarAccountHelper.minReserve(subentryCount); // (2 + n) * 0.5
```

---

## Development

```bash
# Install dependencies
npm install

# Run tests with coverage
npm test

# Lint
npm run lint

# Build to dist/
npm run build
```

All tests run without network access (mocked with Jest). Coverage threshold: 80% lines.

---

## Stellar Ecosystem Alignment

This SDK directly implements protocols championed by the Stellar Development Foundation:

| Protocol | SDF Reference | This SDK |
|----------|--------------|----------|
| x402 | [x402 on Stellar](https://stellar.org/x402) | `X402Client` |
| MPP | [MPP on Stellar](https://developers.stellar.org/docs/build/agentic-payments/mpp) | `buildMppTransaction` |
| Fee Sponsorship | [Stellar Fee Bump](https://developers.stellar.org/docs/learn/encyclopedia/transactions-specialized/fee-bump-transactions) | `buildFeeBump` |
| Soroban | [Soroban Docs](https://developers.stellar.org/docs/build/smart-contracts) | Used by contracts package |

---

## Related Packages

| Package | Description |
|---------|-------------|
| [`stellar-ai-payments-contracts`](https://github.com/Devcyprian/stellar-ai-payments-contracts) | Rust/Soroban contracts: AgentEscrow, SponsoredFeeVault, PaymentRouter |
| [`stellar-ai-payments-adapters`](https://github.com/Devcyprian/stellar-ai-payments-adapters) | LangChain, OpenAI, Claude, Express adapters |
| [`stellar-ai-payments-docs`](https://github.com/Devcyprian/stellar-ai-payments-docs) | Full documentation site (Docusaurus) |

---

## Contributing

This project participates in **Stellar Wave 5** via [Drips Wave](https://drips.network/wave). Issues labeled `wave5` are open for community contributors.

See [CONTRIBUTING.md](CONTRIBUTING.md) for branch model, commit format, and PR checklist.

**Branch model:** all PRs target `develop` → squash merge → `main` for releases.

---

## License

MIT © 2026 [Devcyprian](https://github.com/Devcyprian)
