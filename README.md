# FIP-104 DRAFT : Miner-Only Index Mining Top-Ups via Allowlisted Pools

```
  FIP: 104
  Title: Miner-Only Top-Ups via Allowlisted Pools for Index Mining
  Author: TheLonelyBit , MrShiddy, Fractal Community Contributors
  Status: Draft
  Type: Standard Proposal (Staking Policy & Implementation)
  Created: 2026-07-17
  Updated: 2026-07-18
  Requires: FIP-101
  Layer: Staking System (non-consensus)
  Activation: Coordinated staking portal, pool allowlist, documentation, and client update
```


## Review Note

1. **50 FB init** for everyone (first Binding per indexer).
2. **Sub-50 Top-Up only for ActiveMiner** — allowlisted pool attestation; non-miners stay ≥ 50 FB.
3. **No consensus / node change.**
4. **Owners:** portal/tx-builder, staking docs, Fractal core (allowlist).
5. **Stake address:** UniSat connect or pubkey derive; optional pool pay-to-stake for attested miners.

**Open for core:** `MIN_TOPUP` (recommend **1 FB**); `MINER_ACTIVITY_WINDOW` (default **7 days**).

## Abstract

This proposal refines FIP-101 Index Mining staking so official **Top-Ups below 50 FB** are available only to **active miners**, proven by **Fractal-allowlisted mining pools**.

**50 FB** remains the Initial Stake minimum for all users. After a Binding exists, sub-50 Top-Ups require `Initialized ∧ ActiveMiner`. Non-miner stakers keep ≥ 50 FB funding rules. Stake weight remains stake-address balance (FIP-101 unchanged). Stake address discovery uses UniSat (or equivalent) wallet connect and/or an open pubkey derive tool.

No changes to consensus Indexer blocks, scripts, proofs, 1:1:1 rewards, or settlement.

## Proposal Summary

- **Init:** `MIN_STAKE_INIT = 50 FB` for first Binding (all users).
- **Top-Up:** `amount ≥ MIN_TOPUP` and `< 50 FB` only if ActiveMiner.
- **Proof of miner:** allowlisted pool attestation (not self-declare).
- **Mechanics:** Top-Up = pay bound stake address; no new consensus ops.
- **Deploy:** portal + allowlist + docs; no hard fork.

## Motivation

1. Home miners often earn ≪ 50 FB/day. After Initial Stake they need a compounding path into Index Mining without another 50 FB gate.
2. Opening that path to all stakers weakens the 50 FB anti-dust gate. Miner privilege needs verifiable hashrate activity—same spirit as Index Mining requiring real proofs, not self-declaration.
3. Allowlisted pools already see shares and payouts; they are the natural attestation layer. Non-miners stay on today’s ≥ 50 FB posture.

## Goals & Non-Goals

**Goals:** miner-only sub-50 Top-Ups; keep 50 FB init; allowlist attestation; UniSat/pubkey stake-address discovery; no consensus risk.

**Non-goals:** Top-Ups for all stakers; self-declare as miner proof; lower cold-start below 50 FB; consensus/script/settlement changes; silent Stratum = stake address default.

## Specification

### 1. Definitions

| Term | Meaning |
| --- | --- |
| Payout address | Mining receipt + FIP-101 actor / portal wallet |
| Stake address | Derived Taproot vault for `(indexer_id, pubkey, address_type)` |
| Binding / Initialized | Valid `fip101,1,stake,<indexer_id>` linking payout → stake address |
| Top-Up | Confirmed credit to bound stake address (increases weight) |
| ActiveMiner | Valid attestation from ≥1 allowlisted pool (§5) |
| `MIN_STAKE_INIT` | 50 FB |
| `MIN_TOPUP` | Recommend 1 FB (≥ dust); open for core |
| `MINER_ACTIVITY_WINDOW` | Default 7 days; open for core |

FIP-101 weight (unchanged): `stake_weight = confirmed_balance(stake_address)`.

Official sub-50 Top-Up UX is gated by ActiveMiner. On-chain credits still affect balance; this FIP constrains **portal policy** (same layer as today’s 50 FB gate).

### 2. Admission

```text
TopUpAllowed = Initialized(payout, indexer_id) AND ActiveMiner(payout)
```

1. Not Initialized → Stake builders **must** reject `amount < MIN_STAKE_INIT`. Initial Stake does **not** require ActiveMiner.
2. Official Top-Up with `amount < MIN_STAKE_INIT` **must** require TopUpAllowed and `amount ≥ MIN_TOPUP`; pay bound stake address (no new stake inscription required).
3. Initialized but not ActiveMiner → **must not** offer sub-50 Top-Up; require `≥ MIN_STAKE_INIT` for official adds.
4. Unknown Binding or attestation → **fail closed** (no sub-50 Top-Up / enforce init minimum).

### 3. Stake-address discovery

```text
stake_address = DeriveStakeAddress(indexer_id, user_pubkey, address_type)
```

Same as `stake-indexer` / `unstake-tool` (NUMS internal key + leaf with indexer_id / address_type / x-only pubkey).

Official portal **must**: UniSat connect (Fractal mainnet) → read address + pubkey → select indexer → derive and show stake address before Stake/Top-Up (client derive or `POST /tx/stake-address`).

Official or Fractal-sponsored tooling **must** also provide a keyless derive page/script (`pubkey` + `indexer_id`). Never request seeds/private keys.

**Default pool config:** Stratum reward = payout address.  
**Optional auto-compound:** after Initialized ∧ ActiveMiner, miner may set pool reward to stake address (P2TR); not silent default; control wallet remains Stake/Claim/Unstake actor.

**Forbidden:** self-declare as sole miner proof; derive stake address from payout alone; treat non-allowlisted “on a pool” as ActiveMiner.

### 4. Pool allowlist and attestation

1. Fractal core **must** publish a Pool Allowlist (`pool_id`, attestation URL, status); may add/suspend/remove pools.
2. Active if, within `MINER_ACTIVITY_WINDOW`, the address has **accepted shares** or a **pool payout** at that pool.
3. Pools **should** expose e.g. `GET /v1/fip104/miner-attestation?address=...` returning `{ pool_id, address, active, expires_at, ... }` over HTTPS (optional signatures later).
4. Portal **must** query allowlisted endpoints; ActiveMiner if any returns `active: true` with future `expires_at`; cache ≤ 1 hour; errors → fail closed.
5. Attestation is portal policy, not consensus. Pools never hold stake keys. False attestations → delist. PrimeTriad (`pool.primetriad.com`) is an example eligible operator when listed—not exclusive.

Allowlisted pools **may** deep-link Top-Up or offer opt-in pay-to-stake; not required for activation.

### 5. Unchanged FIP-101 surfaces

Rewards, proofs, commission, settlement (~20,160 blocks), claim, scripts, 1:1:1: **unchanged**.

| Component | Change |
| --- | --- |
| Node / consensus | None |
| stake-indexer mins | None required |
| Portal | Required (ActiveMiner + Top-Up) |
| Allowlist + pool APIs | Required for miner Top-Up |
| Derive tool + docs | Required |

## Security & Backward Compatibility

- 50 FB init preserved; non-miners cannot use official sub-50 Top-Up.
- Fake miners blocked by allowlist + activity window + fail-closed + delist.
- Wrong stake address mitigated by wallet-connect / Binding-only display.
- Existing Bindings valid; nodes need no upgrade; activates with portal + ≥1 allowlisted attestation path.

## Execution Process

1. Core review; freeze `MIN_TOPUP` and `MINER_ACTIVITY_WINDOW`.
2. Publish allowlist + attestation schema.
3. Portal: derive, init, ActiveMiner-gated Top-Up.
4. Onboard pools; update docs; run test vectors; ship as staking-system release (no flag day).

## Test Vectors

| ID | Init? | ActiveMiner? | Action | Amt | Expected |
| --- | --- | --- | --- | --- | --- |
| T1 | No | — | Stake | 49 | Reject |
| T2 | No | — | Stake | 50 | Accept |
| T3 | Yes | Yes | Top-Up | 1 | Accept |
| T4 | Yes | No | Top-Up | 1 | Reject (≥ 50) |
| T5 | Yes | No | Add | 50 | Accept |
| T6 | No | Yes | Top-Up | 5 | Reject (no Binding) |
| T7 | Yes | timeout | Top-Up | 1 | Reject (fail-closed) |
| T8 | Yes | self-declare only | Top-Up | 1 | Reject |
| T9 | — | — | UniSat vs open derive | — | Same stake address |

## References

1. https://github.com/fractal-bitcoin/fips/blob/main/fip-101.md
2. https://github.com/fractal-bitcoin/stake-indexer
3. https://github.com/fractal-bitcoin/unstake-tool
4. https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-guide-for-staking-users
5. Companion audit: `docs/fips/FIP-104-TECHNICAL-AUDIT.md`

---

Fractal Community
