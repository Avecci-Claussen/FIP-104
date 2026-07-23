# FIP-104 DRAFT: Mining → Index Mining Compounding via Allowlisted Pools

```
  FIP: 104
  Title: Mining → Index Mining Compounding via Allowlisted Pools
  Author: TheLonelyBit, MrShiddy, Fractal Community Contributors
  Status: Draft
  Revision: Rev-1.3
  Type: Standard Proposal (Staking Policy & Implementation)
  Created: 2026-07-17
  Updated: 2026-07-23
  Requires: FIP-101
  Layer: Staking System (non-consensus)
  Activation: Staking portal, pool allowlist, documentation, client updates, and at least one allowlisted pool attestation path
```

## Language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in RFC 2119.

## Review Note

1. **Binding minimum remains 50 FB** for all users (first Binding per indexer). Self-funded Binding is the default path; accumulate-to-50 is optional for small miners.
2. **After Binding**, confirmed credits to the bound stake address increase stake weight (FIP-101 balance model). Official portal Top-Ups below 50 FB are gated to **ActiveMiner** only.
3. **Primary miner product:** opt-in pool pay-to-stake or forward-to-stake. Portal Top-Up is secondary. Silent Stratum redirection to a stake address is forbidden.
4. **No consensus or node change.** This FIP defines staking-portal policy and pool integration rails only.
5. **Owners:** staking portal / tx-builder, staking documentation, Fractal core (allowlist), and allowlisted pools (attestation; optional compound).
6. **Stake address discovery:** UniSat (or equivalent) wallet connect, and/or an open pubkey derive tool. Pools MUST NOT hold miner stake keys.

**Parameters open for core:** `MIN_TOPUP` (recommended **1 FB**); `MINER_ACTIVITY_WINDOW` (default **7 days**).

## Abstract

This proposal extends FIP-101 Index Mining staking with an official **Mining → Index Mining compounding path** for **active miners**, verified through **Fractal-allowlisted mining pools**.

The Initial Stake (Binding) minimum remains **50 FB** for all users. After a Binding exists, stake weight equals the confirmed balance of the bound stake address. Official portal Top-Ups below 50 FB require both an existing Binding and ActiveMiner status. Non-miner stakers retain today’s ≥ 50 FB rules for official adds.

Allowlisted pools MAY offer opt-in pay-to-stake (or forward mining rewards to the bound stake address) so miners can compound without repeating the 50 FB gate on every add. Permissionless and merged-mining FB payouts both qualify when the pool can attest the miner’s Fractal payout address.

This proposal does not change consensus Indexer blocks, scripts, proofs, 1:1:1 reward allocation, or settlement.

## Proposal Summary

- **Binding:** `MIN_STAKE_INIT = 50 FB` for the first Binding (all users). Default: miner self-funds and signs. Optional: accumulate, then bind once.
- **Compound:** after Binding, credits to the stake address increase weight. The recommended rail is opt-in pool pay-to-stake or forward-to-stake.
- **Portal Top-Up:** amounts ≥ `MIN_TOPUP` and < 50 FB only if ActiveMiner.
- **Miner proof:** allowlisted pool attestation (not self-declaration), for permissionless or merged-mining FB addresses.
- **Mechanics:** Top-Up and compound are payments to the bound stake address; no new consensus operation; no new stake inscription per credit.
- **Deployment:** portal, allowlist, attestation schema, documentation, and ≥1 live attestation path; no hard fork.

## Motivation

1. Many home miners and pool participants earn well below 50 FB per payout cycle. After Initial Stake they need a compounding path into Index Mining without another 50 FB gate on every add.
2. Fractal’s block economy targets 1:1:1 (merged mining : permissionless mining : indexing). Active miners already participate in the mining pillars; without a compound rail they rarely capture Index Mining yield from the same hashrate.
3. Opening official sub-50 Top-Ups to all stakers would weaken the 50 FB anti-dust gate. Miner privilege therefore requires verifiable mining activity—consistent with Index Mining’s reliance on proofs rather than self-declaration.
4. Allowlisted pools already observe shares and FB payouts (permissionless or merged). They are the natural attestation layer and the practical payout rail for opt-in compound.
5. The FIP-101 stake-indexer already treats stake weight as the stake-address balance after Binding. This FIP makes that compounding path official, gated, and operable by pools—without consensus risk.

## Goals and Non-Goals

**Goals**

- Provide an official Mining → Index Mining compounding path for ActiveMiners
- Preserve the 50 FB Binding minimum for all users
- Define allowlist attestation for permissionless and merged-mining FB payout addresses
- Require clear stake-address discovery (wallet connect and/or open pubkey derive)
- Recommend opt-in pool pay-to-stake / forward-to-stake as the primary rail
- Provide ActiveMiner-gated portal Top-Up as a secondary rail
- Avoid consensus risk

**Non-goals**

- Official sub-50 Top-Ups for all stakers
- Self-declaration as sole proof of miner status
- Lowering Binding below 50 FB
- Changes to consensus, scripts, or settlement
- Silent default of Stratum / pool payout to a stake address
- Pool custody of miner stake keys
- Requiring every pool to ship pay-to-stake for activation (a working attestation path is sufficient; pay-to-stake is strongly recommended)

## Protocol Basis (FIP-101, Unchanged)

1. **Binding.** A valid `fip101,1,stake,<indexer_id>` inscription binds `(user_address, indexer_id)` to a derived Taproot stake address. Official builders enforce `amount ≥ MIN_STAKE_INIT` (50 FB) for this first Binding.
2. **Weight.** After Binding, `stake_weight = confirmed_balance(stake_address)`. Balance deltas update weight. Later credits do not require a new stake inscription.
3. **Exit.** The miner retains private-key control. Unstake and claim remain miner-signed. Pools MUST NOT hold stake keys.
4. **Portal policy versus chain.** This FIP constrains official portal and builder UX (the same layer as today’s 50 FB gate). Raw on-chain credits to a bound stake address still affect balance. ActiveMiner gating applies to official sub-50 Top-Up offers.

## Specification

### 1. Definitions

| Term | Meaning |
| --- | --- |
| Payout address | Miner’s Fractal mining receipt address (permissionless or merged FB) and the FIP-101 actor / portal wallet identity used for Binding |
| Stake address | Derived Taproot vault for `(indexer_id, pubkey, address_type)` |
| Binding / Initialized | A valid `fip101,1,stake,<indexer_id>` linking the user to a stake address for that indexer |
| Top-Up | A confirmed credit to a bound stake address that increases weight; the official portal path MAY amount-gate this action |
| Compound / pay-to-stake | Opt-in pool payout or forward of mining rewards to the bound stake address |
| ActiveMiner | A valid attestation from at least one allowlisted pool (see §5) |
| `MIN_STAKE_INIT` | 50 FB |
| `MIN_TOPUP` | Recommended 1 FB (≥ dust); final value set by core |
| `MINER_ACTIVITY_WINDOW` | Default 7 days; final value set by core |

### 2. Miner flows

#### 2.1 Default — self-fund Binding, then compound

1. The miner connects a wallet (UniSat or equivalent) on Fractal mainnet. The portal reads address and pubkey.
2. The miner selects an indexer. The portal derives and displays the stake address.
3. The miner stakes `≥ MIN_STAKE_INIT` and signs the Binding. ActiveMiner status is **not** required for Binding.
4. The miner opts in to compound on an allowlisted pool (pay-to-stake or forward-to-stake).
5. Subsequent mining rewards credit the stake address; weight increases. The miner claims and unstakes later with their own keys.

#### 2.2 Optional — accumulate-to-50 cold start

For miners who do not yet hold 50 FB:

1. Perform the same opt-in (address, pubkey, indexer).
2. Mining rewards MAY accumulate (in wallet and/or pool ledger) until Binding principal reaches ≥ 50 FB.
3. The miner signs the Binding stake (`≥ MIN_STAKE_INIT`). A pool MAY help fund principal only if the miner remains the inscription actor and retains sole exit-key control.
4. After Binding, the same compound rules as §2.1 apply.

Cold start is optional. Miners who can fund 50 FB themselves SHOULD use §2.1.

#### 2.3 Portal Top-Up (secondary)

ActiveMiners with a Binding MAY use official portal Top-Up for amounts `≥ MIN_TOPUP` and `< MIN_STAKE_INIT`. Non-ActiveMiners MUST be offered official adds only at `≥ MIN_STAKE_INIT`.

### 3. Admission (official portal and builders)

```text
TopUpAllowed = Initialized(user, indexer_id) AND ActiveMiner(payout_address)
```

1. If not Initialized, stake builders MUST reject `amount < MIN_STAKE_INIT`. Initial Stake MUST NOT require ActiveMiner.
2. Official Top-Up with `amount < MIN_STAKE_INIT` MUST require TopUpAllowed and `amount ≥ MIN_TOPUP`, and MUST pay the bound stake address (no new stake inscription).
3. If Initialized but not ActiveMiner, official builders MUST NOT offer sub-50 Top-Up and MUST require `≥ MIN_STAKE_INIT` for official adds.
4. If Binding or attestation is unknown or unavailable, builders MUST fail closed (no sub-50 Top-Up; enforce the Binding minimum).

### 4. Stake-address discovery

```text
stake_address = DeriveStakeAddress(indexer_id, user_pubkey, address_type)
```

Derivation MUST match `stake-indexer` / `unstake-tool` (NUMS internal key and leaf committing `indexer_id`, `address_type`, and x-only pubkey).

The official portal MUST:

1. Connect a Fractal mainnet wallet (UniSat or equivalent)
2. Read address and pubkey
3. Require indexer selection
4. Derive and display the stake address before Stake, Top-Up, or compound opt-in guidance (client-side derive or `POST /tx/stake-address`)

Official or Fractal-sponsored tooling MUST also provide a keyless derive page or script (`pubkey`, `indexer_id`, and address type as required). Tools MUST NOT request seeds or private keys.

The following are forbidden:

- Self-declaration as sole proof of miner status
- Deriving a stake address from payout address alone
- Treating “on a non-allowlisted pool” as ActiveMiner

### 5. Pool allowlist, attestation, and compound rails

1. Fractal core MUST publish a Pool Allowlist including at least `pool_id`, attestation URL, status, and supported mining modes. Core MAY add, suspend, or remove pools.
2. An address is Active at a pool if, within `MINER_ACTIVITY_WINDOW`, it has accepted shares or a pool FB payout at that pool (permissionless or merged-mining FB credit).
3. Allowlisted pools MUST expose attestation over HTTPS, for example:

```text
GET /v1/fip104/miner-attestation?address=<fractal_address>
```

Minimum JSON fields: `pool_id`, `address`, `active`, `expires_at`. Signed attestations MAY be added later.

4. The portal MUST query allowlisted endpoints. ActiveMiner is true if any allowlisted response returns `active: true` with a future `expires_at`. Cache TTL MUST NOT exceed one hour. On error or timeout, the portal MUST fail closed for sub-50 Top-Up.
5. Attestation is portal policy, not consensus. Pools MUST NOT hold stake keys. False attestations MUST be grounds for delisting.
6. Default pool configuration MUST pay the miner’s payout address, not the stake address.
7. Opt-in compound is strongly recommended. After Binding and explicit miner opt-in recorded by the pool, the miner MAY set:
   - pool reward address = bound stake address (P2TR), or
   - pool forward of mining credits to the bound stake address in chunks `≥ MIN_TOPUP`.

   Compound MUST NOT be a silent default. The miner’s control wallet remains the Stake / Claim / Unstake actor. The miner selects the indexer at Binding; pools MUST NOT silently choose or change indexer.
8. Allowlisted pools MAY deep-link portal Top-Up. Pay-to-stake is not required for FIP activation if attestation works, but it is the intended primary miner experience.
9. Example eligible operator when listed (non-exclusive): PrimeTriad (`pool.primetriad.com`).

### 6. Unchanged FIP-101 surfaces

Rewards, proofs, commission, settlement (~20,160 blocks), claim, scripts, and 1:1:1 allocation remain unchanged.

| Component | Change under FIP-104 |
| --- | --- |
| Node / consensus | None |
| stake-indexer minimums | None required (weight already follows balance) |
| Portal | Required: derive, Binding init, ActiveMiner Top-Up, compound guidance |
| Allowlist + pool attestation APIs | Required |
| Pool pay-to-stake / forward | Strongly recommended |
| Derive tool + documentation | Required |

## Worked Example — On-Chain Miner Flow

Illustrative amounts and timing. Addresses are abbreviated. Indexer example: `1818297:10` (PrimeIndexer).

### Actors

| Actor | Role |
| --- | --- |
| Alice | Home miner; wallet `bc1qalice…`; pubkey `P_alice` |
| Pool P | Allowlisted pool; pays FB; exposes attestation API |
| Portal | Official Index Mining UI / transaction builder |
| Chain / stake-indexer | Confirms transactions; maintains Bindings; `stake_weight = balance(stake_address)` |

### Day 0 — Discovery and Binding (self-fund)

1. Alice connects UniSat on Fractal mainnet. The portal reads `bc1qalice…` and `P_alice`.
2. Alice selects indexer `1818297:10`.
3. The portal derives:

```text
S = DeriveStakeAddress(1818297:10, P_alice, address_type)
# → bc1pstake… (Taproot script-path vault; NUMS internal key; leaf CHECKSIG(P_alice))
```

4. Alice stakes **50 FB**. The builder produces the stake flow whose inscription and first non-`OP_RETURN` output satisfy:

```text
inscription: fip101,1,stake,1818297:10
output:      50 FB → S
actor:       P_alice (tapscript)
```

5. Alice signs. Transactions confirm.
6. stake-indexer records Binding `(S → user=bc1qalice…, indexer=1818297:10)` and sets weight from `balance(S)` ≈ 50 FB.

**Checkpoint:** only a transaction whose actor pubkey derives to Alice and whose stake output equals **S** creates this Binding. Pool P cannot forge Alice’s Binding without Alice’s key.

### Day 0 — Opt-in compound (pool)

7. On Pool P, Alice enables compound-to-Index-Mining and confirms that displayed **S** matches portal (or pool) derive.
8. Pool stores `{ payout_address, stake_address: S, compound: true, indexer_id }`.
9. Default payout remains the wallet address unless Alice explicitly selected pay-to-stake. Compound remains opt-in.

### Days 1–7 — Mining credits increase Index Mining weight

10. Alice continues mining. Pool P accepts shares for `bc1qalice…` (permissionless or merged FB accounting).
11. At payout, Alice earned **3.2 FB**. With `compound=true` and Binding present, Pool P broadcasts an ordinary payment:

```text
inputs:  pool hot-wallet UTXOs
output:  3.2 FB → S
# no FIP-101 inscription required
```

12. After confirmation, stake-indexer applies the balance delta:

```text
stake_weight(Alice, 1818297:10) ← balance(S) ≈ 53.2 FB
```

13. The pattern repeats each payout (or in forward chunks ≥ `MIN_TOPUP`). Alice does not sign compound credits.

**Checkpoint:** Pool P can send to **S** but cannot spend **S**. Spending requires Alice’s script-path signature. Compound is not custody.

### Day 7+ — Attestation and optional portal Top-Up

14. Portal queries:

```text
GET /v1/fip104/miner-attestation?address=bc1qalice…
→ { "pool_id": "...", "address": "bc1qalice…", "active": true, "expires_at": "..." }
```

15. If Alice later adds **2 FB** via portal Top-Up and `TopUpAllowed` holds, the builder pays **S**.
16. If attestation times out, portal fails closed on sub-50 Top-Up; pool pay-to-stake continues to work on-chain.

### Day 30 — Claim and unstake (unchanged FIP-101 control)

17. After the settlement window (~20,160 blocks from allocation), Alice claims Index Mining rewards with a wallet signature. Claim output is controlled by Alice, not Pool P.
18. Alice unstakes via portal or `unstake-tool` by spending UTXOs on **S** with `P_alice`. `balance(S)` falls and weight falls. Pool P cannot block exit.

### Edge cases

| Event | Result |
| --- | --- |
| No Binding | Credits to an unbound address are ordinary funds, not Index Mining weight |
| Pool pays wrong address | Alice’s weight unchanged; mitigate with derive match before enabling compound |
| Alice disables compound | Subsequent payouts return to `bc1qalice…` |
| Pool falsifies attestation | Delist; portal gate only; stake keys unaffected |
| Third party pays **S** | Weight rises (balance model); portal still requires attestation for ActiveMiner Top-Up |

## Security Considerations and Threat Model

### On-chain and cryptographic properties (FIP-101)

| Property | Mechanism | Implication for FIP-104 |
| --- | --- | --- |
| Stake vault ownership | Taproot script path `OP_CHECKSIG` on miner x-only pubkey; NUMS internal key | Pool payments to **S** are not spendable by the pool |
| Binding authenticity | Stake output MUST equal `DeriveStakeAddress(...)`; actor from tapscript | Pool cannot bind a miner without the miner’s signature |
| Weight integrity | stake-indexer: weight = confirmed balance of bound **S** | Compound uses ordinary confirmed credits; no new cryptographic primitive |
| Exit | Miner-signed unstake and claim | Non-custodial after Binding |
| Reorganizations | Same Binding and balance indexing as FIP-101 | No new reorg surface |

### Portal and attestation (off-chain policy)

| Threat | Mitigation |
| --- | --- |
| False miner self-declaration | Allowlisted pools only; activity window; fail closed; delist |
| Sold or rented attestations | Delist and pool policy; optional signatures later; cannot move stake keys |
| Attestation outage | Fail closed for sub-50 portal Top-Up; pay-to-stake unaffected |
| Wrong stake address / phishing | Display derive; require wallet pubkey; provide open derive tool; Binding-only display |
| Silent payout redirection to stake | Forbidden; compound requires explicit opt-in |
| Pool choosing indexer for the miner | Forbidden after Binding; miner selects indexer at Binding |
| On-chain dust credits after Binding | Portal keeps ≥ 50 FB for non-ActiveMiner official adds; raw credits may still affect balance (inherent limit of policy-layer FIPs) |
| Pool funds Binding but retains keys | Forbidden; miner MUST remain inscription actor and sole exit-key holder |

### Trust boundaries

```text
Trust pool for:     share accounting, payout execution, attestation honesty
Do not trust pool for: stake custody, unstake, claim, indexer choice after Binding

Trust portal for:   official minimums, ActiveMiner gating UX
Do not trust portal for: consensus validity (nodes and stake-indexer define weight)
```

This matches existing miner trust in pools for mining payouts, while keeping Index Mining principal under FIP-101’s Taproot non-custodial model.

### Backward compatibility

- The 50 FB Binding minimum is preserved. Non-miners cannot use official sub-50 Top-Up.
- Existing Bindings remain valid. Nodes require no upgrade.
- Activation requires portal support plus ≥1 allowlisted attestation path. Pay-to-stake is recommended and not activation-blocking.

## Design Rationale

Effective onboarding rails in proof-of-stake and mining ecosystems tend to share one property: they remove repeated friction between an existing cashflow and a yield sink, without taking user keys.

FIP-104 applies that pattern to Fractal’s 1:1:1 design:

- **Cashflow:** mining FB (permissionless or merged)
- **Yield sink:** Index Mining stake weight
- **Friction removed:** repeated 50 FB official add gate and manual restake
- **Keys retained:** Taproot stake vault and miner-signed exit

This proposal intentionally does not introduce a new consensus asset, liquid staking token, or restaking marketplace. It remains a staking-system policy change with pool rails, so it can activate without a hard fork. Impact scales with allowlisted pool adoption of opt-in pay-to-stake.

## Expected Impact

| Outcome | Expectation |
| --- | --- |
| Home miners | Compound into Index Mining after one Binding without repeated 50 FB gates |
| Pools | Standard attestation and optional pay-to-stake on-ramp |
| Permissionless and merged mining | Both, via FB payout-address attestation |
| Index Mining stake | Gradual hashrate-backed credit flow, not a one-time dump |
| Non-miners | Unchanged official ≥ 50 FB add rules |

Suggested 30-day success metrics after activation: compound / ActiveMiner Top-Up volume (FB), unique compounding addresses, and share of allowlisted-pool hashrate opted into compound mode.

## Execution Process

1. Core review; freeze `MIN_TOPUP` and `MINER_ACTIVITY_WINDOW`.
2. Publish allowlist format and attestation schema.
3. Ship portal: derive, Binding init, ActiveMiner-gated Top-Up, and compound guidance.
4. Onboard ≥1 allowlisted pool with live attestation; recommend pay-to-stake.
5. Update documentation; execute test vectors; release as a staking-system update (no flag day).

## Test Vectors

| ID | Init? | ActiveMiner? | Action | Amt | Expected |
| --- | --- | --- | --- | --- | --- |
| T1 | No | — | Stake | 49 | Reject |
| T2 | No | — | Stake | 50 | Accept (Binding) |
| T3 | Yes | Yes | Portal Top-Up | 1 | Accept |
| T4 | Yes | No | Portal Top-Up | 1 | Reject (require ≥ 50) |
| T5 | Yes | No | Official add | 50 | Accept |
| T6 | No | Yes | Portal Top-Up | 5 | Reject (no Binding) |
| T7 | Yes | timeout | Portal Top-Up | 1 | Reject (fail closed) |
| T8 | Yes | self-declare only | Portal Top-Up | 1 | Reject |
| T9 | — | — | UniSat derive vs open derive | — | Identical stake address |
| T10 | Yes | Yes | Opt-in pay-to-stake credit | 1 | Weight increases |
| T11 | No | Yes | Accumulate only, no Binding | — | No Index Mining weight |
| T12 | Yes | Yes | Merged-mining FB payout attested | — | ActiveMiner true if pool allowlisted |

## References

1. https://github.com/fractal-bitcoin/fips/blob/main/fip-101.md
2. https://github.com/fractal-bitcoin/stake-indexer
3. https://github.com/fractal-bitcoin/unstake-tool
4. https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-guide-for-staking-users
