# Ditto Vault — Yield Strategies

> Canton Network · CIP-56 · Multi-strategy USDCx vault

This document describes the yield strategies the vault router allocates across, the target weights, and the rebalance and risk triggers that drive allocation decisions. All strategies are USDCx-denominated and live on Canton-native protocols.

---

## Table of Contents

- [1. Design intent](#1-design-intent)
- [2. Strategy legs](#2-strategy-legs)
- [3. Target allocation](#3-target-allocation)
- [4. Rebalance and risk triggers](#4-rebalance-and-risk-triggers)
- [5. Execution and accounting](#5-execution-and-accounting)
- [6. Future strategies](#6-future-strategies)

---

## 1. Design intent

The vault is a multi-strategy aggregator, not a single-pool wrapper. The four legs are chosen to be *mechanically distinct* — each responds to a different market regime, so the blended NAV is more stable than any individual strategy.

| Leg type | Pays out from | Hurts when |
|---|---|---|
| Passive lending | Interest spread on lent USDCx | Borrow demand collapses, oracle failure |
| Looped lending | Magnified interest spread | Rate spreads invert, leverage liquidation |
| Delta-hedged LP | Swap fees minus hedge funding | Hedge mismatches, hedge funding spikes |
| Naked LP | Swap fees plus directional exposure | Impermanent loss in directional moves |

The router's job is to keep each leg sized correctly given live conditions, harvest accrued yield into NAV, and ensure withdrawals can always be served from the idle buffer plus immediately-recallable strategy liquidity.

---

## 2. Strategy legs

### Leg 1 — Passive lending (Alpend)

USDCx is supplied directly to the Alpend lending market at the variable supply rate. This is the lowest-risk leg and the largest weight in the default allocation.

**Mechanism**
- Adapter calls Alpend's `Supply` choice on USDCx, receiving aTokens (or equivalent receipt position) representing supplied principal plus accrued interest.
- Interest accrues continuously at a two-slope kink rate model based on pool utilization.
- Withdrawal is permissionless and immediate up to available pool liquidity.

**Risk surface**
- **Protocol risk** — Alpend is in beta with audit not yet complete. Allocation caps will reflect audit status.
- **Oracle risk** — Alpend's price oracle source is not currently disclosed. The `AlpendSupply` adapter will require oracle source disclosure as a precondition for production allocation.
- **Utilization risk** — when utilization approaches 100%, withdrawals queue at protocol level. The router caps allocation so vault buffer stays accessible.

### Leg 2 — Looped lending (Alpend)

Recursive deposit-borrow-resupply loop on Alpend, capped to a safe LTV well below the liquidation threshold.

**Mechanism**
1. Supply USDCx to Alpend.
2. Borrow CC against USDCx collateral at conservative LTV.
3. Swap CC → USDCx via Cantex.
4. Re-supply USDCx, return to step 2 until target loop count is reached.

**Net APY** depends on the spread between Alpend USDCx supply rate and CC borrow rate, plus Cantex swap costs. The leg is enabled only when:

```
USDCx_supply_apy − (CC_borrow_apy × loop_factor) − swap_friction > minimum_spread
```

This is a *modeled* return, not a measured one — it depends on Alpend's CC borrow market having sufficient depth and a stable rate spread, which are not yet published. The leg ships in the **off-by-default** state and is enabled per measured market data.

**Risk surface**
- **Rate-spread inversion** — if the borrow rate exceeds the supply rate × loop factor, the leg becomes unprofitable. Triggered unwind in §4.
- **Liquidation risk** — collateral deteriorates if CC appreciates rapidly. Health-factor floor enforced by trigger.
- **CC borrow depth** — looping consumes Alpend's CC borrow inventory. Allocation capped against published utilization.

### Leg 3 — Delta-hedged LP (Cantex + Alpend)

USDCx/CC liquidity provision on Cantex with a short-CC hedge sized to neutralize the LP position's delta exposure to CC.

**Mechanism**
1. Provide USDCx + CC to a Cantex USDCx/CC pool, receiving LP receipt.
2. Compute LP position's CC exposure (depends on pool curve and current price).
3. Borrow CC on Alpend in size matching the long-CC component of the LP, swap to USDCx, re-supply USDCx as collateral.
4. Net result: LP earns swap fees, hedge cost is the CC borrow rate, market direction is neutralized.

**Net APY** = realized Cantex pool fee APY − CC borrow rate − hedge funding friction. The realized fee APY for Cantex pools is unpublished and will be measured via small live LP positions before this leg is sized to target weight.

**Risk surface**
- **Imperfect hedge** — pool curve nonlinearities mean delta drifts as price moves. Rehedge trigger in §4 caps drift.
- **Hedge funding** — CC borrow rate spikes erode the leg's margin. Disengagement trigger in §4.
- **Cantex pool risk** — protocol-level smart contract risk and any oracle dependency on Cantex's side.

### Leg 4 — Naked LP (Cantex)

USDCx/CC LP without hedging. Directional exposure to CC is retained in exchange for a higher fraction of swap fees and the chance for impermanent gain in mean-reverting regimes.

**Mechanism** — same as Leg 3 step 1, no hedge.

**Status** — **off by default**. Enabled only with explicit operator opt-in and capped allocation. This leg exists for completeness; the prudent default is for the vault not to hold directional CC exposure.

**Risk surface**
- **Impermanent loss** — full IL exposure to CC moves.
- All Leg 3 protocol-level risks.

### Idle buffer

Unallocated USDCx held by the vault operator party, available for immediate withdrawal serving.

**Sizing**
- Default floor: ≥10% of NAV.
- Dynamically increased during high redemption pressure.
- Replenished from the lowest-yielding deployed strategy when below floor.

---

## 3. Target allocation

The default allocation when all preconditions (audits, oracle disclosure, sufficient borrow depth, measured fee APY) are met:

| Leg | Default weight | Notes |
|---|---|---|
| Passive lending (Alpend) | ~45% | Largest weight; lowest risk |
| Looped lending (Alpend) | ~15% | Off until rate spread measured |
| Delta-hedged LP (Cantex) | ~20% | Off until pool fee APY measured |
| Naked LP (Cantex) | 0–10% | Off by default |
| Idle buffer | ≥10% | Floor, dynamically scaled |

These weights are **target ranges**, not hard allocations. The router moves capital between legs based on live APY readings, utilization, and risk triggers (§4). Total deployed never exceeds `1 − buffer_floor` of NAV.

---

## 4. Rebalance and risk triggers

The router evaluates these continuously. Each trigger has a deterministic action — no operator discretion required for routine rebalancing.

| # | Trigger | Action |
|---|---|---|
| T1 | Alpend USDCx supply APY < (Alpend CC borrow APY × 0.6) | Unwind Leg 2 immediately |
| T2 | Leg 2 health factor < 1.5 | Auto-deleverage to HF ≥ 1.8 |
| T3 | Leg 3 pool delta drift > 10% from neutral | Rehedge |
| T4 | Any Alpend pool utilization > 95% | Pause new Alpend deposits, route to Cantex / buffer |
| T5 | Cantex pool fee APY (rolling 7d) < CC borrow APY | Pause Leg 3, route to Leg 1 / buffer |
| T6 | Realized 7-day blended APY < 2% | Disclose to vault holders, pause growth marketing |
| T7 | Idle buffer < 5% of NAV | Recall from lowest-yielding strategy until ≥10% |
| T8 | Idle buffer < 1% of NAV (withdrawal-pressure floor) | Pause new deposits, recall from all strategies until restored |
| T9 | Strategy adapter reports unhealthy oracle / protocol state | Pause that adapter, recall to buffer |

Triggers T1–T8 are evaluated each accounting tick (currently every 10s aligned with the indexer poll). T9 evaluates on every DVN attestation update.

---

## 5. Execution and accounting

### Adapter interface

Every strategy implements the same interface:

```typescript
interface StrategyAdapter {
  id: string;
  protocol: 'alpend' | 'cantex' | ...;
  asset: 'USDCx' | 'CC' | ...;
  deploy(amount: bigint): Promise<void>;
  harvest(): Promise<{ realized: bigint, deployed: bigint }>;
  withdraw(amount: bigint): Promise<void>;
  getCurrentValue(): Promise<bigint>;
  getApy(): Promise<{ apy7d: number, apy30d: number }>;
  getHealth(): Promise<HealthReport>;
}
```

### NAV and share price

```
NAV = vault_reserve + Σ strategy_allocations.last_value
share_price = NAV / total_shares
```

`last_value` is updated on every harvest tick or DVN attestation, never set arbitrarily by the operator (the operator cannot, e.g., type a number into an admin form). NAV is fully derived; the operator cannot set NAV manually.

### Phased operator integration

For v1 (Phase 4a), each adapter wraps a **manual operator workflow**: the operator deposits and withdraws on the protocol's UI directly, and `last_value` is updated from the operator's read of the protocol's reported balance. This is honest about the team's actual operational burden in the early weeks of MainNet, and it ships in days rather than months.

The trust gap (operator-driven NAV updates) is mitigated by the on-chain NAV publication mechanism described in [ARCHITECTURE.md §8](./ARCHITECTURE.md#8-nav-derivation-and-on-chain-publication). Every published NAV is a verifiable receipt that auditors and counterparties cross-check against protocol-side state.

Phase 4b adds CLI-driven semi-automation; Phase 4c is the fully-autonomous adapter that requires only protocol-side maturity (SDKs, on-chain state readers).

### Harvest cadence

Adapters are harvested on a per-protocol cadence:
- Alpend supply: continuous interest accrual, no on-chain harvest required (value is read from the supply receipt; in Phase 4a, operator reads from Alpend UI).
- Alpend looped: same.
- Cantex LP: fees compound into the pool position; periodic harvest claim required to realize them into NAV.

Each harvest tick produces a new `NavAnchor` on-chain (see ARCHITECTURE §8).

### Withdrawal serving

Withdrawals are served in this order:
1. From idle buffer if sufficient.
2. From lowest-yielding strategy, sized to leave that strategy at its target floor.
3. Cascading to higher-yielding strategies if buffer + lowest leg insufficient.
4. Queued with published SLA if cumulative recall would breach a strategy's exit liquidity.

---

## 6. Future strategies

Strategies under evaluation, gated on Canton ecosystem developments:

| Strategy | Gate |
|---|---|
| **Stable/stable pool** | Counter-stable to USDCx (USDT, USDY, FOBXX) needs to ship on Canton. This becomes the lowest-risk leg in any USDCx-denominated vault and supplants part of Leg 1. |
| **Multi-asset lending** | Additional Canton lending markets beyond Alpend. Diversifies protocol risk; adds parallel passive-lending capacity. |
| **CC staking** | When permissionless CC delegation becomes available beyond institutional Super Validators. |
| **Tokenized RWA tranche** | Phase 8 — SPV-wrapped private credit fund tranche, deployed only inside the `dvUSDCx-LOCK1Y` locked vault. Targets stable double-digit yield from actively-managed credit. Subject to legal SPV setup and accreditation gating. |

Each new strategy is added as a separate adapter behind the same interface and goes through the same staged-enablement process: audit / disclosure prerequisites, small-size live test, measured APY, then ramp to target weight.

---

## 7. Vault product variants

The strategy framework above supports a lineup of vault products, each its own CIP-56 dvToken with its own fee schedule and strategy mix:

| Vault | Base asset | Lock term | Strategy mix | Target APY | Indicative fees |
|---|---|---|---|---|---|
| **`dvUSDCx-CORE`** *(launch product)* | USDCx | Daily-liquid | Legs 1–4 with conservative weights, ≥10% buffer | ~5% measured (passive Alpend USDCx supply); 5–9% modeled, gated on Alpend CC-borrow rate disclosure | 1.5% mgmt / 10% carry |
| **`dvUSDCx-LOCK90`** | USDCx | 90 days | Legs 1–4 with looser buffer, higher Leg 2 weight | 6–11% modeled; same gating | 1.5% mgmt / 15% carry |
| **`dvUSDCx-LOCK1Y`** | USDCx | 1 year | Legs 1–4 plus tokenized RWA tranche (when live) | Double-digit target, contingent on private credit fund partnership and SPV setup | 2.0% mgmt / 20% carry |
| **`dvCC`** *(planned)* | CC | Daily-liquid | Alpend CC supply + future CC strategies | TBD | 1.0% mgmt / 10% carry |
| **`dvCBTC`** *(planned)* | CBTC | Daily-liquid | Alpend CBTC supply + future CBTC strategies | TBD | 1.0% mgmt / 10% carry |

**Reading these numbers**: Only the Alpend USDCx passive-supply rate is currently *measured* (~4.85% per April 2026 Alpend disclosure). Every other range is *modeled* — derived from rate-spread assumptions that depend on Alpend's CC borrow market, Cantex realized fee APY, and protocol-level data not yet published. Numbers will be updated as live measurement replaces modeling.

All vaults share the same backend, indexer, oracle, and accounting infrastructure. They differ only in strategy mix, base asset, lock term, and fee schedule. Adding a new vault is a configuration change and a new dvToken instrument — not a new code base.

Fee accrual is per-vault: each vault's management and performance fee streams are independent. Asset-issuer marker rewards are per-dvToken — under the Featured App Coupon Guidance (revised 22 April 2026, effective 27 April 2026 21:00 UTC), each instrument earns 1 marker per third-party / transaction-originator / venue-submitted transaction involving that dvToken (peer-to-peer transfers, DVP swaps, composable use), regardless of how many dvToken movements that transaction batches. Vault deposit and redeem clearings are operator-submitted and therefore **do not** generate the asset-issuer marker; the marker stream is driven by secondary-market activity, not deposit/redeem volume.
