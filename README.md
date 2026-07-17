# FIP-104: Init-Only Minimum Stake and Incremental Top-Ups for Index Mining

```
  FIP: 104
  Title: Init-Only Minimum Stake and Incremental Top-Ups for Index Mining
  Author: TheLonelyBit (@MrShiddy @fractal_tlb) on X, Fractal Community Contributors
  Status: Draft
  Type: Standard Proposal (Staking Policy & Implementation)
  Created: 2026-07-17
  Requires: FIP-101
  Layer: Staking System (non-consensus)
  Activation: Coordinated staking portal, documentation, and client update
```

## Review Note

1. **Init-only 50 FB** — Keep the official 50 FB minimum for first-time stake binding per `(payout_address, indexer_id)`; do not lower the cold-start gate in this FIP.
2. **Top-Up path** — After a binding exists, official interfaces must support funding the stake address below 50 FB (balance-weighted accounting already used by FIP-101 / `stake-indexer`).
3. **No consensus change** — No node upgrade, no Indexer-block / script / 1:1:1 / settlement changes; staking portal + docs (+ optional clients) only.
4. **Portal / docs owners** — Activation owners are the official Index Mining portal / tx-builder maintainers and FIP-101 staking documentation maintainers.
5. **Stake-address discovery (required UX)** — Official portal **must** support UniSat wallet connect → read address + pubkey → derive stake address for the selected `indexer_id`, plus an on-site / open-source derive path (`pubkey` + `indexer_id` → stake address). Pools may reuse that flow; Stratum stays `payout_address.worker_name`.

**Open for core decision:** Exact value of `MIN_TOPUP` (recommended default **1 FB**, must be ≥ network dust). Please confirm or set an alternate floor before portal implementation freezes.

## Abstract

This proposal refines the **minimum stake participation rules** of the FIP-101 Index Mining staking system.

FIP-101 established Taproot-based non-custodial staking, balance-weighted stake accounting, and a documented participation threshold of **50 FB**. In current practice, that threshold is enforced as an **official portal and documentation policy**, while `stake-indexer` recognizes bindings and weights the **confirmed balance** of each derived stake address.

FIP-104 makes the intended rule explicit and operationally consistent:

- The **50 FB minimum applies to Initial Stake** (creation of a stake binding for a given user address and indexer).
- **Incremental Top-Ups** to an already-bound stake address may be smaller than 50 FB, subject only to a documented dust / operational floor.
- Official interfaces must distinguish **payout address** from **stake address**, and **must** expose a deterministic discovery path: **UniSat wallet connect** (or equivalent) to obtain the pubkey and **derive** the stake address for a chosen indexer, with an open on-site / script fallback for advanced users.

This proposal does **not** modify consensus Indexer blocks, reward allocation ratios, Taproot script templates, proof verification, commission bounds, issuance, or settlement delay.


## Proposal Summary

- **Init-Only Minimum:** Retain **50 FB** as the official minimum for first-time stake binding under a given indexer.
- **Incremental Top-Ups:** After a binding exists, additional funding of the stake address below 50 FB is permitted and must be supported by official Top-Up flows.
- **Balance-Weighted Synergy:** Align portal policy with FIP-101’s existing model: stake weight equals stake-address balance after binding.
- **Deterministic Address Discovery:** Stake address is computed from `(indexer_id, pubkey, address_type)` via UniSat connect and/or an open derive tool — not guessed from Stratum and not pasted into miner config.
- **Non-Consensus Activation:** Deploy via staking portal, documentation, and client updates. No node hard fork.


## Motivation

1. Participation Friction for Small-Scale Miners

Permissionless Mining and Index Mining are co-equal pillars under FIP-101’s 1:1:1 design. Many home and small-scale miners receive daily pool payouts well below 50 FB. After they have initialized a stake position, requiring another 50 FB action to compound mining rewards into Index Mining creates unnecessary friction and weakens the intended economic loop between hashrate contributors and indexing infrastructure.

2. Ambiguity Between Policy Minimum and Protocol Accounting

FIP-101 states that minimum stake thresholds apply and that weighting rules are specified in implementation documents. The live staking system weights **stake-address balances** after a valid binding. Official product surfaces, however, often present 50 FB as a blanket requirement for every stake action. This ambiguity causes incorrect integrator assumptions and blocks legitimate compounding paths that the accounting layer already supports.

3. Address Confusion in Mining Integrations

Mining pools commonly identify workers by payout address (for example, Stratum worker `payout_address.worker_name`). That address is the user’s wallet identity and mining receipt address. It is **not** the FIP-101 stake address. Without a clear standard, pools and miners incorrectly expect a “staking address” field in miner configuration, which would break payouts and couple firmware to a specific indexer.


## Design Overview

1. Compatibility with FIP-101 Pillars

FIP-104 preserves FIP-101’s architecture:

- **Merged Mining** — unchanged
- **Permissionless Mining** — unchanged
- **Indexing / Index Mining** — unchanged at consensus; staking participation UX and policy clarified

Indexer block validation (Chain ID `0x2026`, proof structures, quantity constraints), reward formulas, and settlement remain exactly as specified by FIP-101 and its technical notes.

2. Init Versus Top-Up

FIP-101 staking already has two natural phases:

- **Binding creation** via a `stake` inscription that associates a user address with an indexer and a derived stake address
- **Balance maintenance** via confirmed credits and debits on that stake address

FIP-104 assigns the 50 FB minimum to binding creation in official flows, and treats subsequent credits as Top-Ups.

3. Non-Custodial Continuity

Top-Ups do not introduce custody, operator approval, or slashing. Users retain key control of stake-address spends. Exit remains a user-authored spend from the stake address, consistent with FIP-101 and the emergency unstake tooling.

4. Deterministic Stake-Address Discovery

The stake address is a pure function of FIP-101 parameters already used by `stake-indexer` and `unstake-tool`. Discovery **must not** depend on pools inventing a second miner credential.

Canonical discovery order:

1. **Wallet connect (primary):** User connects UniSat (Fractal mainnet) → client reads `address` + `publicKey` → derives stake address for selected `indexer_id` (client-side Taproot derivation **or** official `POST /tx/stake-address`).
2. **On-site / open-source derive (fallback):** User pastes compressed or x-only pubkey (+ address type if required) and `indexer_id` → same derivation, no custody of keys.
3. **Binding confirmation (secondary):** After Initial Stake, stake-indexer user-stakings APIs return the bound `stake_address` for display and pool dashboards.

5. Pool-Neutral Integration

Pools continue to pay mining rewards to the payout address. They **may** embed the same wallet-connect derive UI or deep-link to the official derive / Top-Up page. Stratum identity remains payout-address-based.


## Relationship to FIP-101

| FIP-101 Element | FIP-104 Effect |
| --- | --- |
| 1:1:1 block reward structure | Unchanged |
| Indexer block consensus rules | Unchanged |
| Taproot stake leaf / NUMS internal key | Unchanged |
| Stake inscription `fip101,1,stake,<indexer_id>` | Unchanged |
| Stake weight = stake-address balance | Affirmed; Top-Ups rely on this |
| Documented 50 FB minimum | Scoped to **Initial Stake** in official flows |
| ~20,160-block settlement / manual claim | Unchanged |
| Commission / proof validity / delayed proof penalties | Unchanged |
| Non-custodial exit | Unchanged |

FIP-104 is an **additive clarification** of staking participation parameters contemplated by FIP-101 (“minimum stake thresholds” / implementation weighting rules). It does not supersede FIP-101 consensus text.


## Goals & Non-Goals

### Goals

- **Clarify** that the official 50 FB minimum is an Initial Stake rule
- **Enable** frequent sub-50 FB compounding into Index Mining after initialization
- **Align** portal admission policy with stake-indexer balance accounting
- **Standardize** payout-address vs stake-address terminology for miners, pools, and wallets
- **Require** UniSat (or equivalent) wallet-connect derivation plus an open pubkey→stake-address tool
- **Preserve** FIP-101 security, liveness, and non-custodial guarantees

### Non-Goals

This proposal does **not** modify:

- Consensus, PoW, difficulty, block structure, or UTXO / script language
- Index Mining coinbase allocation or emission schedule
- Stake script templates or derivation constants
- Proof formats, proof windows, or reward quantization rules
- Settlement delay or claim authorization model

This proposal does **not**:

- Lower the 50 FB Initial Stake minimum for never-initialized addresses
- Require mining-pool attestations as a protocol predicate
- Mandate that pools auto-forward payouts to stake addresses
- Place stake addresses into Stratum worker configuration
- Replace UniSat; other wallets **may** be added if they expose address + pubkey equivalently


## Specification

### 1. Definitions

- **Payout Address:** The user wallet address used as mining receipt identity and as the FIP-101 actor / portal identity.
- **Stake Address:** The Taproot address derived from `(indexer_id, user_pubkey, address_type)` that holds staked principal for that indexer.
- **Binding:** A valid FIP-101 `stake` inscription recognized by stake-indexer, associating payout address and indexer with a stake address.
- **Initialized Position:** A Binding exists for `(payout_address, indexer_id)`.
- **Initial Stake:** The official flow that creates a Binding and funds the stake address.
- **Top-Up:** Any confirmed transaction that increases the balance of an existing stake address for an Initialized Position.
- **`MIN_STAKE_INIT`:** `50 FB`.
- **`MIN_TOPUP`:** Implementation parameter. Recommended default `1 FB`. Must be at least the network dust threshold. **Open for core decision** before implementation freeze (see Review Note).

### 2. FIP-101 Mechanics Affirmed (Informative Reference)

Stake address derivation (unchanged):

- Internal key: `50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`
- Leaf: `<indexer_id> OP_DROP <address_type> OP_DROP <x-only_pubkey> OP_CHECKSIG`

Binding inscription body (unchanged):

```text
fip101,1,stake,<indexer_id>
```

After Binding recognition:

```text
stake_weight(payout_address, indexer_id) = confirmed_balance(stake_address)
```

Top-Ups therefore do not require a second `stake` inscription. Unstake remains a spend from the stake address. Claim and proof flows remain as in FIP-101.

### 3. Init-Only Minimum (Official Flows)

1. If `(payout_address, indexer_id)` is not Initialized, official Stake builders **must** reject amounts below `MIN_STAKE_INIT`.
2. If the position is Initialized, official interfaces **must not** require `MIN_STAKE_INIT` for further funding and **should** direct users to Top-Up.
3. This section constrains official portal / documentation policy. It does not add a new consensus rule.

### 4. Incremental Top-Up Rules

1. For an Initialized Position, official Top-Up flows **must** accept amounts greater than or equal to `MIN_TOPUP`.
2. Top-Up **must** be implemented as a payment to the bound stake address.
3. A new `stake` inscription **should not** be required for Top-Up.
4. Initializing under a different indexer requires a separate Binding; that Initial Stake remains subject to `MIN_STAKE_INIT`.
5. If Binding status cannot be determined, official Stake flows **must** fail closed to `MIN_STAKE_INIT`.

### 5. Stake-Address Discovery (Normative)

Payout address and stake address are distinct and **must** be labeled distinctly in official interfaces.

#### 5.1 Derivation inputs

```text
stake_address = DeriveStakeAddress(indexer_id, user_pubkey, address_type)
```

Where leaf and NUMS internal key are exactly as in FIP-101 / `stake-indexer` / `unstake-tool` (see §2). `address_type` is inferred from the connected payout address format (P2TR / P2WPKH / P2SH-P2WPKH / P2PKH) consistent with existing FIP-101 tooling.

#### 5.2 Official portal — wallet connect (**must**)

Official Index Mining Stake and Top-Up surfaces **must**:

1. Support **UniSat wallet connect** on Fractal Bitcoin mainnet (or a documented equivalent wallet API that exposes address + pubkey).
2. On connect, read:
   - payout `address`
   - `publicKey` (compressed or x-only, normalized as in current portal / unstake-tool)
3. Let the user select `indexer_id`.
4. **Derive and display** the stake address before the user signs Initial Stake or Top-Up.
5. For Top-Up, build a payment to that derived (or Binding-confirmed) stake address — not a second 50 FB Stake gate.

Derivation **must** match one of:

- Client-side Taproot derivation identical to `fractal-bitcoin/unstake-tool`, or
- Official `POST /tx/stake-address` with `{ address, pubKey, indexerId }` returning `{ stakeAddress }`

Displayed stake address **must** be copyable and, when a Binding already exists, **should** be cross-checked against stake-indexer user-stakings for that payout address.

#### 5.3 On-site / open-source derive tool (**must**)

In addition to wallet connect, official or Fractal-sponsored tooling **must** provide a **keyless derive page or script** that:

1. Accepts `indexer_id` and user `pubkey` (optional: payout address for address-type inference).
2. Outputs the deterministic stake address.
3. Never requests seed phrases or private keys.
4. Documents that Top-Up before Binding creates **no** stake weight.

This covers advanced users, integrators, and pools that cannot embed UniSat, and mirrors the trust model of the existing unstake emergency tool.

#### 5.4 Binding lookup (**should**)

After Initial Stake, clients **should** read `stake_address` from stake-indexer (`GET /users/:payout_address/stakings` or equivalent) to confirm the on-chain Binding matches the derived address.

#### 5.5 Default vs advanced pool payout configuration

**Default (recommended for most miners):**

```text
Stratum worker / pool reward address = payout_address (control wallet)
```

Mine → pool pays payout wallet → user Top-Ups via portal (or manual send) to stake address.  
UniSat connect, Initial Stake, Claim, and Unstake all use the **same** payout wallet.

**Advanced auto-compound (optional, not weak if done correctly):**

After an Initialized Position exists, a miner **may** set the pool reward destination to the **derived stake address** (for example worker `stake_address.worker_name` on pools that pay the address prefix), so mining payouts land directly on the stake address and increase stake weight automatically.

This is cryptographically sound: any confirmed credit to a bound stake address is a Top-Up under FIP-101 accounting. It is **not** required by this FIP.

**Hard requirements if using advanced mode:**

1. Binding **must** already exist for that stake address / indexer (or be created immediately); credits before Binding do not earn Index Mining weight until Binding is recognized.
2. Pool **must** support paying **P2TR** (`bc1p…`) destinations — FIP-101 stake addresses are Taproot.
3. Docs **must** warn: stake address is **per `indexer_id`**. Switching indexer requires a new derive + pool payout update.
4. Control wallet (UniSat) remains the payout/actor address used for Initial Stake, Claim, and Unstake — it is **not** replaced by the stake address.
5. Pools **must not** make stake-address-as-reward the silent default for new miners.

#### 5.6 What is forbidden

1. Requiring stake address as the only allowed Stratum identity for all miners.
2. Inventing stake addresses from payout address alone (no pubkey / no `indexer_id`).
3. Treating “miner connected to pool” as proof of an Initialized Position.
4. Asking for seed phrases or private keys in derive tools.

### 6. Optional Pool Compounding Assistance (Non-Mandatory)

Mining pools **may**:

- Embed UniSat-connect derive, or deep-link to the official derive / Top-Up page
- After Binding, show the stake address and an explicit **“Pay mining rewards to stake address (auto Top-Up)”** opt-in
- Server-side forward payouts to stake address only with opt-in + Binding checks

Pool features are not required for FIP-104 activation.

### 7. Rewards, Proofs, and Settlement

Unchanged from FIP-101:

- Effective stake and commission split
- Proof validity and delayed-proof penalties
- Reward release tiers
- Settlement delay of approximately 20,160 blocks
- Manual claim flow

Official interfaces **should** present claim readiness in a weekly cadence consistent with settlement, without implying a separate emission schedule.

### 8. Compatibility Matrix

| Component | Required Change |
| --- | --- |
| Consensus / node software | None |
| Indexer proof pipeline | None |
| Stake script templates | None |
| `stake-indexer` min-amount validation | None required |
| Official staking portal / tx builder | Required: UniSat connect derive + Stake vs Top-Up |
| Official / open derive page or script | Required: pubkey + `indexer_id` → stake address |
| Official documentation | Required |
| Third-party pools | Recommended: embed or deep-link the same derive flow |


## Security & Backward Compatibility

### Security Considerations

- **Anti-dust onboarding:** Initial Stake retains the 50 FB official gate.
- **Wrong-address Top-Ups:** Stake addresses must come from wallet-connect derivation, the open derive tool, or a confirmed Binding — never from payout address alone or miner config.
- **Pubkey handling:** Derive tools must accept only pubkey / address inputs; never seed phrases or private keys.
- **Pre-binding transfers:** Credits to an unbound address do not create stake weight until Binding exists; official Top-Up must check Initialization first.
- **Pool forwarding:** Opt-in only; never require stake private keys; always show the destination stake address.
- **Liveness:** No change to block production or proof critical paths.
- **Fund control:** Non-custodial Taproot control and unilateral exit remain intact.

### Upgrade Impact

After activation of official staking-system updates:

- Existing Bindings and balances remain valid
- Reward math and settlement remain valid
- Non-upgraded third-party UIs may continue to over-enforce 50 FB until they adopt Top-Up
- Nodes require no upgrade

No changes are made to emission, PoW, difficulty, block structure, UTXO model, or script language.


## FIP-104 Execution Process

1. Community and core-contributor review of this draft.
2. Core decision on `MIN_TOPUP` (recommended default: 1 FB; see Review Note), then parameter freeze.
3. Official portal update: UniSat connect → derive stake address; Initial Stake vs Top-Up; dual-address presentation.
4. Ship / document open derive page or script (`pubkey` + `indexer_id`), aligned with `unstake-tool` derivation.
5. Documentation update: staking guide covers wallet-connect discovery and init-only minimum.
6. Public testing validation against the vectors below.
7. Activation via staking-system and documentation release notes (no consensus flag day).


## Test Vectors

| ID | Initialized? | Action | Amount | Expected (official flow) |
| --- | --- | --- | --- | --- |
| T1 | No | Initial Stake | 49 FB | Reject |
| T2 | No | Initial Stake | 50 FB | Accept; Binding created |
| T3 | Yes | Top-Up | 1 FB | Accept |
| T4 | Yes | Payment to stake address | 2 FB | Stake weight increases |
| T5 | Yes | Stratum worker remains payout address | — | Mining payouts unchanged |
| T6 | No | Payment to unbound derived address | any | No stake weight until Binding |
| T7 | Yes for indexer A only | Initial Stake to indexer B | 20 FB | Reject |
| T8 | Yes | Second official Stake to same indexer | any | Redirect to Top-Up; no 50 FB Top-Up gate |
| T9 | — | UniSat connect + indexer_id | — | Portal displays stake address matching `unstake-tool` / `/tx/stake-address` |
| T10 | — | Open derive tool with same pubkey + indexer_id | — | Output equals portal-derived stake address |


## Reference Implementation

Reference changes are expected in:

- Official Index Mining portal: UniSat `requestAccounts` / `getPublicKey` (or current UniSat Fractal APIs) → derive → Stake / Top-Up
- Official or Fractal-sponsored **Derive Stake Address** page/script (pubkey + indexer_id; no keys)
- Staking user documentation and rewards documentation
- Optional pool dashboards reusing the same derivation module or deep-link

Open-source references for derivation and accounting (already correct):

- `fractal-bitcoin/unstake-tool` (`DeriveStakeAddress` / leaf script)
- `fractal-bitcoin/stake-indexer` (`DeriveStakeAddress`, Binding + balance weight)
- Official `POST /tx/stake-address` (portal API)
- `fractal-bitcoin/fips` (FIP-101 and technical notes)

Operational guidance for miners:

1. Mine to payout address as today (`payout_address.worker_name`).
2. Open Index Mining → **Connect UniSat** with that same payout wallet → select indexer → portal shows derived **stake address**.
3. Complete Initial Stake (≥ 50 FB) once.
4. Compound via Top-Up to the displayed stake address (or use the open derive tool if verifying offline).
5. Claim Index Mining rewards after settlement as under FIP-101.


## Rationale

Assigning the 50 FB minimum to Initial Stake preserves FIP-101’s anti-dust onboarding posture while unlocking the compounding loop that balance-weighted accounting already permits.

Keeping stake addresses out of Stratum preserves mining payout safety and indexer portability. The technically correct discovery path is the same one FIP-101 staking already uses: **wallet pubkey + indexer_id → deterministic Taproot stake address**. UniSat connect is the primary UX; an open derive script is the auditable fallback; Binding lookup is confirmation — not a substitute for derivation.

Implementing FIP-104 outside consensus minimizes risk, matches FIP-101’s non-blocking settlement philosophy, and allows progressive portal rollout during public testing or full Index Mining operation.


## Acknowledgments

This draft incorporates community direction associated with PrimeTriad / @MrShiddy and public discussion with Fractal contributors, and is intended as a compatible refinement of the FIP-101 staking participation model.


## References

1. FIP-101: Fractal Standard Indexing Service — https://github.com/fractal-bitcoin/fips/blob/main/fip-101.md
2. FIP-101 Technical Notes (Node Changes) — https://github.com/fractal-bitcoin/fips/blob/main/fip-101-technical-notes-part-1-node-changes.md
3. Stake Indexer — https://github.com/fractal-bitcoin/stake-indexer
4. FIP-101 Unstake Tool — https://github.com/fractal-bitcoin/unstake-tool
5. FIP-101 Guide for Staking Users — https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-guide-for-staking-users
6. FIP-101 Rewards Allocation — https://docs.fractalbitcoin.io/overview/fip-101-fractal-standard-indexing-service/fip-101-rewards-allocation

---

Fractal Community
