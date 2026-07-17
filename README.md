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

1. **Init-only 50 FB (everyone)** — Keep the official **50 FB** minimum for first-time stake binding per `(payout_address, indexer_id)`. Do not lower the cold-start gate.
2. **Sub-50 Top-Up only for ActiveMiner** — After Binding exists, official Top-Up below 50 FB is allowed **only** if an **allowlisted mining pool** attests the payout address is an active miner. Non-miner stakers stay on ≥ 50 FB funding rules.
3. **No consensus change** — No node upgrade; no Indexer-block / script / 1:1:1 / settlement changes. Portal + allowlist + docs (+ clients) only.
4. **Owners** — Index Mining portal / tx-builder maintainers, staking docs maintainers, and Fractal core as **pool allowlist** publisher.
5. **Stake-address discovery** — UniSat connect (address + pubkey) → derive stake address for `indexer_id`, plus open pubkey→stake-address tool. Optional pool auto-compound for attested miners only.

**Open for core decision:**

- Exact value of `MIN_TOPUP` (recommended default **1 FB**, must be ≥ network dust).
- Exact miner activity window if other than the draft default of **7 days**.

## Abstract

This proposal refines FIP-101 Index Mining staking participation so that **incremental Top-Ups below 50 FB** are available to **active miners only**, proven by **allowlisted mining pools**.

FIP-101 established Taproot-based non-custodial staking, balance-weighted stake accounting, and a documented **50 FB** participation threshold (enforced today as portal / documentation policy). Stake weight equals the confirmed balance of a derived stake address after a valid Binding.

FIP-104 makes the following rules explicit:

- **50 FB** remains the official minimum for **Initial Stake** for all users.
- **Top-Up** below 50 FB (above `MIN_TOPUP`) is an official path **only** when:
  1. a Binding already exists for `(payout_address, indexer_id)`, and
  2. the payout address is attested as an **ActiveMiner** by at least one pool on the Fractal-published allowlist.
- Non-miner stakers **must not** receive the sub-50 Top-Up exception.
- Official interfaces **must** distinguish payout address from stake address and expose UniSat (or equivalent) wallet-connect derivation plus an open derive tool.

This mirrors FIP-101’s economic spirit: Index Mining rewards follow real indexing work and stake; miner Top-Ups follow **real hashrate activity**, not self-declaration.

This proposal does **not** modify consensus Indexer blocks, reward ratios, Taproot scripts, proof verification, commission bounds, issuance, or settlement delay.


## Proposal Summary

- **Init-Only Minimum:** **50 FB** for first Binding under a given indexer (all users).
- **Miner-Only Top-Ups:** Sub-50 FB Top-Ups require **Initialized ∧ ActiveMiner**.
- **Allowlisted Pool Attestation:** Fractal publishes a pool allowlist; pools expose an attestation API for active miner addresses.
- **Balance-Weighted Synergy:** Top-Up remains a payment to the bound stake address; weight = balance (FIP-101 unchanged).
- **Deterministic Address Discovery:** UniSat connect and/or open derive tool; never Stratum-as-stake-address by default.
- **Non-Consensus Activation:** Portal, allowlist, docs, and optional pool integrations. No hard fork.


## Motivation

1. Align Compounding with Hashrate Contributors

Permissionless Mining and Index Mining are co-equal under FIP-101’s 1:1:1 design. Home and small-scale miners often earn well below 50 FB per day. After they initialize a stake, they need a path to compound mining rewards into Index Mining without another 50 FB gate.

Opening that path to **all** stakers would dilute the anti-dust intent of the 50 FB threshold and grant a privilege meant for hashrate contributors to passive capital alone.

2. Prove “Miner” the Same Way Index Mining Proves Work

Index Mining does not reward self-declared indexers: operators must run hardware/software and submit valid proofs; stakers bind capital to those operators. Likewise, miner-only Top-Ups must not rely on a checkbox. **Allowlisted pools** that already observe shares and payouts are the natural attestation layer—analogous to an allowlist of who may certify activity.

3. Keep the 50 FB Gate Meaningful for Non-Miners

Non-miner stakers continue under the existing ≥ 50 FB funding posture. The exception is narrow, fail-closed, and revocable by delisting abusive pools.


## Design Overview

1. Compatibility with FIP-101 Pillars

- Merged Mining, Permissionless Mining, and Indexing consensus rules: **unchanged**
- Stake scripts, Binding, balance weight, proofs, settlement: **unchanged**
- FIP-104 adds **portal admission policy** for who may use `MIN_TOPUP`

2. Admission Predicates

```text
TopUpAllowed(address, indexer_id) =
  Initialized(address, indexer_id) AND ActiveMiner(address)
```

```text
Initialized  = Binding exists for (payout_address, indexer_id)
ActiveMiner  = ≥1 allowlisted pool returns valid active attestation
```

3. Fail-Closed

If Binding or attestation cannot be verified, official flows treat the user as **not** eligible for sub-50 Top-Up.

4. Deterministic Stake-Address Discovery

Same as FIP-101 tooling: wallet pubkey + `indexer_id` → stake address (UniSat connect primary; open derive fallback; Binding lookup for confirmation).


## Relationship to FIP-101

| FIP-101 Element | FIP-104 Effect |
| --- | --- |
| 1:1:1 / Indexer blocks / proofs | Unchanged |
| Taproot stake leaf / NUMS key | Unchanged |
| Stake inscription / Binding | Unchanged |
| Stake weight = stake-address balance | Affirmed |
| 50 FB documented minimum | Init for all; sub-50 Top-Up **miners only** |
| Settlement / claim / commission | Unchanged |
| Non-custodial exit | Unchanged |

FIP-104 does not supersede FIP-101 consensus text. It specifies staking participation policy contemplated by FIP-101’s “minimum stake thresholds” language.


## Goals & Non-Goals

### Goals

- Enable sub-50 FB compounding for **active miners** after Initial Stake
- Keep **50 FB** init for everyone and **no** sub-50 Top-Up for non-miners
- Define **ActiveMiner** via Fractal-allowlisted pool attestation (not self-declare)
- Require UniSat/pubkey stake-address discovery
- Preserve FIP-101 security, liveness, and non-custody

### Non-Goals

This proposal does **not** modify consensus, emission, scripts, proofs, or settlement.

This proposal does **not**:

- Grant sub-50 Top-Ups to all stakers with a Binding
- Accept self-declaration as miner proof
- Lower `MIN_STAKE_INIT` for never-initialized addresses
- Mandate that pools auto-forward payouts to stake addresses
- Place stake addresses into Stratum as the silent default
- Make pool attestation a consensus / node validation rule


## Specification

### 1. Definitions

- **Payout Address:** Wallet used for mining receipt and FIP-101 actor / portal identity.
- **Stake Address:** Taproot address from `(indexer_id, user_pubkey, address_type)` holding stake principal.
- **Binding / Initialized Position:** Valid FIP-101 `stake` inscription linking payout address + indexer → stake address.
- **Initial Stake:** Official flow creating a Binding; subject to `MIN_STAKE_INIT`.
- **Top-Up:** Confirmed credit to an existing stake address that increases stake weight.
- **Pool Allowlist:** Published set of mining pools authorized to issue ActiveMiner attestations for FIP-104.
- **ActiveMiner:** Payout address with at least one valid attestation from an allowlisted pool under §6.
- **`MIN_STAKE_INIT`:** `50 FB`.
- **`MIN_TOPUP`:** Recommended default `1 FB` (≥ dust). **Open for core decision.**
- **`MINER_ACTIVITY_WINDOW`:** Default **7 days**. **Open for core decision.**

### 2. FIP-101 Mechanics Affirmed (Informative)

Stake address (unchanged):

- Internal key: `50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`
- Leaf: `<indexer_id> OP_DROP <address_type> OP_DROP <x-only_pubkey> OP_CHECKSIG`

Binding body: `fip101,1,stake,<indexer_id>`

```text
stake_weight(payout_address, indexer_id) = confirmed_balance(stake_address)
```

On-chain, any credit to a bound stake address increases weight. **Official sub-50 Top-Up UX and builders** are gated by ActiveMiner (§4–§6). Plain transfers remain possible at the protocol layer; this FIP constrains **official product policy**, consistent with how the 50 FB gate is enforced today.

### 3. Init-Only Minimum (All Users)

1. If not Initialized, official Stake builders **must** reject `amount < MIN_STAKE_INIT`.
2. Initial Stake does **not** require ActiveMiner (miners and non-miners may initialize at ≥ 50 FB).
3. Fail closed to `MIN_STAKE_INIT` if Binding status is unknown.

### 4. Miner-Only Incremental Top-Up Rules

1. Official Top-Up with `amount < MIN_STAKE_INIT` **must** be allowed only if:
   - Initialized for the selected indexer, and
   - ActiveMiner attestation is valid for the payout address, and
   - `amount ≥ MIN_TOPUP`.
2. Top-Up **must** pay the bound stake address (transfer/PSBT). A new `stake` inscription **should not** be required.
3. If Initialized but **not** ActiveMiner, official interfaces **must not** offer sub-50 Top-Up. They **must** require `amount ≥ MIN_STAKE_INIT` for further official funding (or equivalent messaging).
4. If not Initialized, Top-Up **must** be rejected; user must Initial Stake first.
5. New indexer → new Binding → `MIN_STAKE_INIT` again; Top-Up eligibility re-checked per indexer position + ActiveMiner.

### 5. Stake-Address Discovery (Normative)

#### 5.1 Derivation

```text
stake_address = DeriveStakeAddress(indexer_id, user_pubkey, address_type)
```

Identical to `stake-indexer` / `unstake-tool`.

#### 5.2 Official portal — UniSat connect (**must**)

1. Connect UniSat on Fractal mainnet (or documented equivalent exposing address + pubkey).
2. Read payout `address` and `publicKey`.
3. User selects `indexer_id`.
4. Derive and display stake address before Stake or Top-Up.
5. For eligible Top-Up, build payment to that stake address.

Derivation via client-side parity with `unstake-tool` **or** `POST /tx/stake-address`.

#### 5.3 Open derive tool (**must**)

Keyless page/script: `indexer_id` + `pubkey` → stake address; no seeds/private keys; warn that credits before Binding earn no weight.

#### 5.4 Binding lookup (**should**)

Confirm `stake_address` via stake-indexer user-stakings after Initial Stake.

#### 5.5 Default vs advanced pool payout

**Default:** Stratum / pool reward = **payout address** (control wallet).

**Advanced auto-compound (optional):** After `Initialized ∧ ActiveMiner`, miner **may** set pool reward to the **stake address** (P2TR). Requirements:

1. Binding exists (or created immediately).
2. Pool supports P2TR payouts.
3. Stake address is per `indexer_id`; switching indexer requires update.
4. Control wallet remains actor for Stake / Claim / Unstake.
5. Not the silent default for new miners.
6. Official Top-Up UX still requires ActiveMiner for sub-50 amounts.

#### 5.6 Forbidden

1. Self-declaration as sole miner proof for sub-50 Top-Up.
2. Inventing stake address from payout address alone.
3. Treating “connected to any pool” without allowlisted attestation as ActiveMiner.
4. Requesting seed phrases in derive tools.

### 6. Pool Allowlist and ActiveMiner Attestation

#### 6.1 Allowlist

1. Fractal core / Index Mining maintainers **must** publish a **Pool Allowlist** (pool_id, attestation base URL, status).
2. Only allowlisted pools may issue attestations consumed by official Top-Up admission.
3. Core **may** add, suspend, or remove pools (abuse, downtime, false attestations).
4. Example eligible operator (non-exclusive): PrimeTriad (`pool.primetriad.com`) when listed.

#### 6.2 Activity rule (default)

An address is active at a pool if, within `MINER_ACTIVITY_WINDOW` (default **7 days**), it has either:

- accepted shares attributed to that payout address, or
- at least one pool payout to that address.

#### 6.3 Attestation API (informative sketch)

Allowlisted pools **should** expose HTTPS:

```http
GET /v1/fip104/miner-attestation?address=<payout_address>
```

Example JSON:

```json
{
  "pool_id": "primetriad",
  "address": "bc1q...",
  "active": true,
  "since": "2026-07-11T00:00:00Z",
  "expires_at": "2026-07-18T02:00:00Z",
  "activity_window_seconds": 604800
}
```

Optional later: pool-key signature over the payload. v1 **may** rely on HTTPS + allowlist URL authenticity.

#### 6.4 Portal verification

1. Official Top-Up **must** query one or more allowlisted attestation endpoints for the connected payout address.
2. ActiveMiner if **any** allowlisted pool returns `active: true` with `expires_at` in the future.
3. Portal **may** cache results ≤ **1 hour**.
4. On error, timeout, or `active: false` from all queried pools → **fail closed** (not ActiveMiner).

#### 6.5 Trust and abuse

- Attestation is **portal policy**, not consensus validation.
- Pools attest activity only; they **must not** receive stake private keys.
- False or indiscriminate `active: true` is grounds for allowlist removal.

### 7. Optional Pool Product Features

Allowlisted pools **may**:

- Embed UniSat derive or deep-link to official Top-Up
- Show Binding + stake address after init
- Offer opt-in “pay mining rewards to stake address” for ActiveMiner + Binding

Not required for FIP-104 activation.

### 8. Rewards, Proofs, and Settlement

Unchanged from FIP-101 (effective stake, commission, proofs, release tiers, ~20,160-block settlement, manual claim).

### 9. Compatibility Matrix

| Component | Required Change |
| --- | --- |
| Consensus / node | None |
| Indexer proofs / scripts | None |
| `stake-indexer` min validation | None required |
| Official portal / tx builder | Required: ActiveMiner check + Stake vs Top-Up |
| Pool allowlist publication | Required |
| Allowlisted pool attestation API | Required for pools that want miners to use Top-Up |
| Open derive page/script | Required |
| Docs | Required |


## Security & Backward Compatibility

### Security Considerations

- **Anti-dust:** 50 FB init remains for all; non-miners cannot use sub-50 Top-Up officially.
- **Anti-fake-miner:** Allowlisted attestation + activity window; fail-closed; delist abusive pools.
- **Spam Top-Ups:** Only after 50 FB init **and** recent mining activity; network fees still apply.
- **Wrong address:** Derive from wallet connect / open tool / Binding only.
- **Pre-binding credits:** No weight until Binding.
- **Liveness / custody:** Unchanged from FIP-101.

### Upgrade Impact

- Existing Bindings remain valid.
- Non-miner stakers: no behavioral change vs today’s ≥ 50 FB portal posture for adds.
- Miner Top-Up activates when portal + ≥1 allowlisted attestation path is live.
- Nodes: no upgrade.


## FIP-104 Execution Process

1. Community / core review of this draft.
2. Core decisions: `MIN_TOPUP`, `MINER_ACTIVITY_WINDOW` (defaults 1 FB / 7 days).
3. Publish initial Pool Allowlist and attestation URL schema.
4. Portal: UniSat derive; Initial Stake; Top-Up gated by ActiveMiner.
5. Onboard allowlisted pools (e.g. PrimeTriad) with attestation endpoint.
6. Docs: miner-only Top-Up + address model.
7. Validate test vectors; activate via staking-system release notes (no consensus flag day).


## Test Vectors

| ID | Initialized? | ActiveMiner? | Action | Amount | Expected (official flow) |
| --- | --- | --- | --- | --- | --- |
| T1 | No | — | Initial Stake | 49 FB | Reject |
| T2 | No | — | Initial Stake | 50 FB | Accept; Binding created |
| T3 | Yes | Yes | Top-Up | 1 FB | Accept |
| T4 | Yes | No | Top-Up | 1 FB | Reject (require ≥ 50 FB) |
| T5 | Yes | No | Official add | 50 FB | Accept under ≥ 50 FB rule |
| T6 | Yes | Yes | Payment to stake address | 2 FB | Weight increases |
| T7 | No | Yes | Top-Up | 5 FB | Reject (no Binding) |
| T8 | Yes | Attestation timeout | Top-Up | 1 FB | Reject (fail-closed) |
| T9 | Yes A only | Yes | Init stake indexer B | 20 FB | Reject |
| T10 | — | — | UniSat derive | — | Matches unstake-tool / `/tx/stake-address` |
| T11 | — | — | Open derive same inputs | — | Same stake address as T10 |
| T12 | Yes | Yes | Self-declare miner, no pool | 1 FB | Reject |


## Reference Implementation

- Official portal: UniSat connect → derive → Binding check → allowlist attestation → Top-Up PSBT
- Pool Allowlist registry (published JSON/docs)
- Allowlisted pool: `GET /v1/fip104/miner-attestation?address=`
- Open derive page/script (pubkey + indexer_id)
- References: `fractal-bitcoin/stake-indexer`, `fractal-bitcoin/unstake-tool`, FIP-101 docs

Operational path for miners:

1. Mine on an allowlisted pool (`payout_address.worker_name`).
2. Connect same payout wallet → Initial Stake ≥ 50 FB.
3. When attestation is active, Top-Up ≥ `MIN_TOPUP` to stake address (or advanced: pool pays stake address).
4. Claim after settlement as under FIP-101.


## Rationale

Restricting sub-50 Top-Ups to ActiveMiners preserves the 50 FB anti-dust gate for general stakers while restoring the miner→Index Mining compounding loop.

Allowlisted pool attestation matches FIP-101’s pattern of rewarding verifiable participation (proofs / stake) rather than claims. Keeping attestation at the portal layer avoids consensus risk and allows delisting without a hard fork.

Wallet-connect derivation remains the correct stake-address discovery path; Stratum continues to identify the payout wallet unless the miner explicitly opts into stake-address auto-compound after Binding.


## Acknowledgments

Draft direction from PrimeTriad / @MrShiddy and Fractal community discussion; intended as a compatible refinement of FIP-101 staking participation policy.


## References

1. FIP-101 — https://github.com/fractal-bitcoin/fips/blob/main/fip-101.md
2. FIP-101 Technical Notes — https://github.com/fractal-bitcoin/fips/blob/main/fip-101-technical-notes-part-1-node-changes.md
3. Stake Indexer — https://github.com/fractal-bitcoin/stake-indexer
4. Unstake Tool — https://github.com/fractal-bitcoin/unstake-tool
5. FIP-101 Staking Users Guide — https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-guide-for-staking-users
6. FIP-101 Rewards Allocation — https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-rewards-allocation
7. Companion audit — `docs/fips/FIP-104-TECHNICAL-AUDIT.md`

---

Fractal Community
