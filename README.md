# Ditto Vault

**Yield-Bearing Vault Token on Canton Network**

---

## Overview

Ditto Vault bridges risk-adjusted yield from EVM DeFi to Canton Network through a CIP-56-compliant vault token (**dvUSDC**). Users deposit supported stablecoins on Canton and receive dvUSDC — a yield-bearing token backed by returns from leading DeFi money markets. Yield generation is fully autonomous and secured by Ditto Network's decentralized operator set of 16 operators across Eigenlayer and Symbiotic, securing over $200M in TVL.

The vault operates with a hybrid on-chain/off-chain architecture: **CIP-56 token contracts** handle all token movements on Canton, while **PostgreSQL** manages vault accounting, deposit/withdrawal queues, and user state. This separation minimizes on-chain complexity and cost while preserving full CIP-56 compliance for token interoperability.

---

## Architecture

```
Canton Network                                EVM Yield Engine
┌───────────────────────────────────┐         ┌──────────────────┐
│  Splice Validator Node            │         │  Aave · Morpho   │
│  ├── Participant (Ledger API)     │         │  Fluid · Spark   │
│  └── CIP-56 Token Contracts      │  Bridge │  (16 operators)  │
│      ├── dvUSDC Holdings/Factory  │ ◄─────► │                  │
│      └── USDCx Holdings/Factory   │         └──────────────────┘
└────────────┬──────────────────────┘
             │ JSON Ledger API v2
┌────────────┴──────────────────────┐
│  Application Server               │
│                                   │
│  PostgreSQL (Off-Chain State)     │
│  ├── vault_state (singleton)     │  ← NAV, shares, price
│  ├── deposit_queue               │  ← Pending deposits
│  ├── withdrawal_queue            │  ← Pending withdrawals
│  ├── users                       │  ← Auth, custodial balances
│  └── supported_deposit_tokens    │  ← Configurable tokens
│                                   │
│  Express.js Backend (API + Auth) │
│  ├── React App        (/app)    │  ← User-facing UI
│  └── Controller       (/demo)   │  ← Operator interface
└───────────────────────────────────┘
```

---

## CIP-56 Tokens

| Token | Purpose | Standard |
|---|---|---|
| **dvUSDC** | Vault share token — proportional ownership of vault NAV | CIP-56 Holding + BurnMintFactory |
| **USDCx** | Stablecoin — deposited by users, held by operator | CIP-56 Holding + BurnMintFactory |

Both tokens implement the Splice CIP-56 token standard interfaces. Holdings follow the UTXO model — minting burns inputs and creates outputs. Factories are nonconsuming, allowing multiple mint/burn operations in a single atomic batch.

---

## Key Flows

### Memo-Based Routing

All deposits and withdrawals use a self-describing memo: `{targetPartyId} {optionalMemo}`. The backend parses the target address and forwarding memo from this format. No Canton party registration is required for core deposit/withdraw functionality.

### Deposit USDCx → Get Shares

Any Canton party sends supported stablecoins to the operator address with a memo specifying where to mint dvUSDC shares.

1. Sender transfers USDCx to operator (CIP-56 burn/mint on-chain)
2. Backend queues the deposit in PostgreSQL with the parsed mint target
3. Operator clears the queue: CIP-56 mints dvUSDC to the target address, updates vault state in DB
4. If the target is the treasury (custodial), the user's DB balance is credited automatically

### Deposit Shares → Custodial Account

Anyone can send dvUSDC directly to a custodial user's account by transferring shares to the treasury address with the user's ID as memo. The transfer happens on-chain via CIP-56 and the user's balance is credited in PostgreSQL immediately — no queue needed.

### Withdraw dvUSDC → Get USDCx

Any Canton party sends dvUSDC to the operator address with a memo specifying where to send USDCx redemption.

1. Sender transfers dvUSDC to operator (CIP-56 burn/mint on-chain)
2. Backend queues the withdrawal in PostgreSQL
3. Operator clears the queue: burns dvUSDC, transfers USDCx to the target, updates vault state

### Transfer Shares

Authenticated users can transfer dvUSDC between wallets (custodial-to-custodial, custodial-to-non-custodial, or vice versa) through the application API.

---

## Interaction Modes

### Non-Custodial

Users provide their own Canton party ID and interact permissionlessly. Shares are held directly by their party on-chain. No registration is required — the memo carries all routing information.

### Custodial

Users register through the application and receive a deposit memo containing the treasury address and their user ID. Shares are held by a shared treasury party on-chain; individual balances are tracked in PostgreSQL. This mode supports users who prefer a managed experience without running their own Canton participant.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Token Contracts | Daml 3.x / CIP-56 (Holding + BurnMintFactory interfaces) |
| Network | Canton Network (Splice validator, DevNet + MainNet) |
| Ledger API | Canton JSON Ledger API v2 (HTTP) |
| Backend | Node.js, TypeScript, Express |
| Database | PostgreSQL |
| User Auth | JWT (bcryptjs + jsonwebtoken) |
| Frontend (App) | React, Vite, TypeScript, Tailwind CSS, shadcn/ui |
| Frontend (Controller) | Vanilla HTML/JS + Tailwind CSS |
| Deployment | Docker Compose |

---

## User Interface

### User Dashboard (`/app`)

- **Deposit USDCx** — modal with vault address and memo for depositing stablecoins
- **Deposit Shares** — modal with treasury address and user ID for receiving dvUSDC
- **Withdraw** — redeem dvUSDC for stablecoins via the withdrawal queue
- **Send Shares** — transfer dvUSDC to any Canton party or custodial user
- Balance cards showing dvUSDC holdings, share value, and pending items
- Vault statistics (NAV, total shares, share price)

### Controller Dashboard (`/demo`)

- Operator and test-user send forms with intelligent routing
- Queue management (view/clear pending deposits and withdrawals)
- Vault controls (fund reserve, bridge to/from EVM, update NAV, pause)
- Configurable supported deposit tokens
- All balances (on-chain + custodial DB)

---

## Design Principles

1. **CIP-56 only on-chain** — Token Holdings and Factories are the sole on-chain contracts. Vault accounting lives in PostgreSQL, reducing Canton traffic cost and UTXO complexity.
2. **Permissionless memo routing** — No registration needed. Deposit/withdrawal memos carry all routing information, enabling any Canton party to interact.
3. **Custodial option** — Users who prefer a managed experience share a treasury party. Individual balances tracked off-chain with DB rollback protection on chain failures.
4. **Configurable deposit tokens** — Operator can add/remove accepted tokens via the database, enabling seamless migration from test tokens to production stablecoins.
5. **Atomic CIP-56 operations** — Multi-command submissions ensure all-or-nothing execution for token operations.
6. **Single operator party** — The operator acts as both vault manager and CIP-56 token admin, simplifying authorization for atomic transactions.

---

## Vault Accounting

All vault state management happens in PostgreSQL:

| Field | Description |
|---|---|
| `nav` | Total vault value in USDC terms |
| `total_shares` | Total dvUSDC shares outstanding |
| `share_price` | `nav / total_shares` — updated on every deposit/withdrawal clearing |
| `vault_reserve` | USDCx available for withdrawals |
| `evm_vault_balance` | Capital deployed to EVM yield strategies |
| `is_paused` | Emergency pause flag — blocks all operations |

**Share price calculation:**
```
On deposit:  sharesToMint    = depositAmount / sharePrice
On withdraw: redemptionAmount = sharesToBurn × sharePrice
Yield:       sharePrice increases as NAV grows from EVM yield
```

---

## API

### Public (No Authentication)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/deposit` | Deposit stablecoins `{ senderPartyId, amount, memo }` |
| POST | `/api/deposit-shares` | Deposit dvUSDC to custodial user `{ senderPartyId, amount, memo }` |
| POST | `/api/withdraw` | Withdraw dvUSDC `{ senderPartyId, dvUsdcAmount, memo }` |

### User (JWT Required)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Register (custodial or non-custodial) |
| POST | `/api/auth/login` | Login, returns JWT |
| GET | `/api/user/portfolio` | Balances, pending items, vault stats |
| POST | `/api/user/transfer-shares` | Transfer dvUSDC to another wallet |
| POST | `/api/user/custodial-withdraw` | Custodial withdrawal to destination |

### Operator

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/state` | Full vault state + on-chain balances + queues |
| POST | `/api/process-deposits` | Clear pending deposits |
| POST | `/api/process-withdrawals` | Clear pending withdrawals |
| POST | `/api/update-nav` | Update NAV and share price |
| POST | `/api/transfer` | Raw CIP-56 transfer between parties |

---

## Featured App Alignment

Ditto Vault is designed for Canton Featured App compliance:

- Full CIP-56 compliance — dvUSDC uses BurnMintFactory for standard-compatible mint, burn, and transfer operations
- Economically motivated transactions — every CIP-56 operation serves a genuine user need (deposit, withdrawal, transfer)
- Institutional controls — operator-gated clearing with full audit trail
- Composable ecosystem asset — dvUSDC is available as a CIP-56 token for any Canton application
- Active validator presence — Ditto operates validators on Canton DevNet, TestNet, and MainNet

---

## Revenue Model

Self-sustaining revenue from protocol operations.

| Source | Mechanism |
|---|---|
| **Management Fee** | 0.5–2% annual on AUM, deducted from yield |
| **Performance Fee** | Share of yield above benchmark |

---

## Roadmap

| Phase | Status | Scope |
|---|---|---|
| **Phase 1 — MVP** | **Complete** | CIP-56 tokens, memo-based deposit/withdraw, custodial/non-custodial modes, PostgreSQL queues, React UI, Docker deployment, DevNet + MainNet validators |
| **Phase 2 — Bridge** | Planned | Circle xReserve integration for USDCx ↔ USDC bridging, end-to-end fund movement |
| **Phase 3 — Featured App** | Planned | Committee review, CIP-56 compliance validation, security audit, Featured App markers |
| **Phase 4 — Liquidity** | Planned | On-chain secondary market for instant exits, yield distribution mechanism |

---

## About Ditto Network

**Canton Network Presence**
- Validator operator on Canton DevNet, TestNet, and MainNet
- Active participant in Canton ecosystem since early access
- CIP-56 token integration with working deposit/withdrawal/transfer flows

**DeFi Infrastructure Track Record**
- 16 node operators across Eigenlayer and Symbiotic restaking protocols
- Over $200M in TVL secured by a decentralized operator set
- Autonomous yield generation across Aave, Morpho, Fluid, and Spark
- Live, battle-tested cross-chain automation and vault management platform

---

[dittonetwork.io](https://dittonetwork.io) · [@Ditto_Network](https://x.com/Ditto_Network) · [GitHub](https://github.com/dittonetwork)
