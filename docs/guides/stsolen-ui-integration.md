# stSOLEN UI Integration Plan

This is the build plan for surfacing the stSOLEN liquid-staking contract through Solen's user-facing wallets and a dedicated dapp. It is written to be picked up *after* the v1 contract deploys on mainnet — nothing here is meant to ship before then.

stSOLEN itself is a liquid-staking derivative: users deposit SOLEN, receive `stSOLEN` (an SRC-20 receipt token), and the contract round-robins delegations across an operator allowlist. Rewards auto-credit each epoch and raise the exchange rate against `stSOLEN`. Withdrawals run through a queue that respects the staking system contract's 7-epoch unbonding period.

## Three integration surfaces

Ranked by leverage:

1. **Standalone dapp at `stake.solenchain.io`** — primary entry point. Uses `window.solen.signAndSubmit` from solenbrowser. Builds independently; doesn't touch the wallet repos.
2. **`StakeStsolenCard` in solenwallet** — desktop power users. Mirrors the existing `StakingCard.tsx` pattern (which handles native validator delegation, not stSOLEN).
3. **stSOLEN balance row + APY pill in solenbrowser popup** — passive surfacing only. No actions; deep-links to the dapp.

Ship in that order. The dapp covers most users without touching wallet code; the wallet additions are convenience.

---

## 1 — `solenstake/` dapp

**Stack:** Vite + React, matching the rest of the dapp lineup. Hosted at `stake.solenchain.io`. Mainnet-only (single network) for v1.

**Pages:**

- `/` — main stake/unstake. Tabs for **Stake**, **Unstake**, **Claims**.
- `/claims/[seq]` — direct deep-link for a queue entry (used in email/notif flows).
- `/admin` — owner-only: operator allowlist editor, fee config, slash oracle key. Gated by reading `owner()` via `solen_callView`.

### Stake tab

- Inputs: `amount` (SOLEN). Reads the user's available balance from `window.solen.getBalance()`.
- Reads `exchange_rate()` via `solen_callView` → returns `(total_pooled[16] || total_supply[16])`. Compute `mint_preview = amount * total_supply / total_pooled` client-side at full precision (use `BigInt` throughout — never coerce to `Number` for u128 math).
- Show the current APY (described below) and a "you'll receive" stSOLEN amount.
- Button → builds `[Transfer(stSOLEN, amount), Call(stSOLEN, "deposit", &[])]` and sends via `window.solen.signAndSubmit`.

### Unstake tab

- Input: stSOLEN to burn.
- Preview: `solen_owed = stsolen_burn * total_pooled / total_supply` and the **eligible epoch** (`current_epoch + UNBONDING_EPOCHS + 1` ≈ 8 epochs ≈ ~27 minutes at 100-block × 2-s × 8). Show the absolute time, not just "8 epochs."
- Button → `Call(stSOLEN, "request_withdrawal", stsolen_burn[16])`.

### Claims tab

- Lists the user's pending withdrawals from `wq/{seq}` storage.
- Per claim: status badge (Unbonding / Ready / Claimed), countdown to eligible epoch, "Claim" button when ready.
- "Claim" button auto-handles the buffer state: if `withdrawal_buffer < solen_owed`, the dapp first sends a `Call(stSOLEN, "crank_undelegations", &[])`, then `Call(stSOLEN, "claim_withdrawal", seq[8])`. Bundle into a single `signAndSubmit` if both fit; otherwise two ops with quiet status feedback. **The user never sees "crank required" — this is auto.**
- "How can I exit faster?" footer linking to the solenswap stSOLEN/SOLEN pair.

### APY computation (off-chain, indexed)

- Subscribe to `recompounded` events emitted by the stSOLEN contract via the explorer.
- For each: parse `amount[16] || operator[32]`. Sum amounts over the rolling window.
- APY ≈ `sum(amount) / avg(total_pooled_solen) × (epochs_per_year / N)`.
- **Lead with 7-day** as the headline number; render a smaller "30-day: X%" beneath it for context.
- During the first 7 days post-deploy, fall back to **"since launch"** so we don't extrapolate from a thin sample. Avoid "annualized" projections off a single epoch.
- This is purely cosmetic; never use the displayed APY in transaction-build math.

### File layout

```
solenstake/
├── src/
│   ├── lib/
│   │   ├── rate.ts          # exchange_rate, BigInt-safe mint/burn previews
│   │   ├── stsolen.ts       # contract address, ABI-ish helpers
│   │   ├── claims.ts        # parses wq/{seq} blobs from solen_callView
│   │   └── apy.ts           # event-derived APY
│   ├── components/
│   │   ├── StakeForm.tsx
│   │   ├── UnstakeForm.tsx
│   │   ├── ClaimsList.tsx
│   │   └── ConnectButton.tsx
│   └── pages/
│       ├── index.tsx
│       └── admin.tsx
└── package.json
```

**Effort:** ~5 days of focused work. The hard part is making the BigInt math airtight; the UI is small.

---

## 2 — `StakeStsolenCard` in solenwallet

Drops next to the existing `StakingCard.tsx` (which is for native validator delegation, not stSOLEN). New file, shares no code with the existing card.

### Files

- `~/solenwallet/src/components/StakeStsolenCard.tsx` — UI component, ~250 lines.
- `~/solenwallet/src/lib/stsolen.ts` — pure helpers (no React):
    - `STSOLEN_ADDRESS_BY_NETWORK: Record<NetworkName, string>` (mainnet only for v1)
    - `readExchangeRate(rpc): Promise<{ pool: bigint; supply: bigint }>`
    - `previewMint(amount, pool, supply): bigint`
    - `previewOwed(burn, pool, supply): bigint`
    - `buildDepositOp(sender, nonce, contract, amount): UserOperation`
    - `buildRequestWithdrawalOp(sender, nonce, contract, burn): UserOperation`
    - `buildClaimOp(sender, nonce, contract, seq): UserOperation`
    - `buildCrankOp(sender, nonce, contract): UserOperation`
    - `readWithdrawalQueue(rpc, contract, account): Promise<ClaimRow[]>`

### UI layout

- Top: stSOLEN balance + current backing in SOLEN ("You hold 12.34 stSOLEN ≈ 13.50 SOLEN"). Backing = `bal × total_pooled / total_supply`.
- Tabs: **Stake** / **Unstake** / **Claims**, identical UX to the dapp.
- Submit path: build the same UserOps but sign locally with the wallet's keystore — no `window.solen` indirection since solenwallet has direct key access.

### Wiring

- Add `StakeStsolenCard` to `App.tsx` as a new card alongside `StakingCard`. Hide on testnet/devnet (gate by `STSOLEN_ADDRESS_BY_NETWORK[currentNetwork]`).
- Reuse `lib/rpc.ts` for `solen_callView`, `solen_getNextNonce`, `solen_submitOperationConfirm`.
- Reuse `lib/wallet.ts` for op-signing — same Ed25519/BLAKE3 path the existing `StakingCard` uses.

### Visual reuse

Match the design language already established in `StakingCard.tsx` (left-aligned numeric labels, validator chips, action buttons in the bottom row). Don't reinvent.

**Effort:** ~2 days. Most of the substance is already in the dapp's `lib/`; this is a port + sign-locally adaptation.

---

## 3 — solenbrowser popup integration

Minimal — popup is space-constrained. Goal: make stSOLEN visible to the user without forcing them through the dapp to check.

### Changes

1. **`token-registry.ts`** — register stSOLEN with a logo:
    ```ts
    "<stSOLEN_contract_hex_lowercase_no_0x>": "stsolen-logo.svg",
    ```
    Add the SVG under `~/solenbrowser/public/`.

2. **Backing-aware balance pill on `TokensList`** — when a token row is stSOLEN (matched by contract address), show a second line: "≈ N.NN SOLEN" computed from the same exchange-rate read the dapp does. Cache the rate for 30 s in `chrome.storage.local`.

3. **APY badge on the same row** — small "APY 5.2%" pill. Reads from a public oracle endpoint (`api.solenchain.io/api/stsolen/apy` — needs to be added to the indexer; trivial endpoint).

4. **"Stake →" deep link** in the row's overflow menu, opening `https://stake.solenchain.io?account=<base58>` in a new tab.

**No transaction-build paths in solenbrowser.** Keep it a passive viewer; let the dapp drive activity through `window.solen.signAndSubmit`. This matches how solenswap/launchpad already work.

### Files

- `~/solenbrowser/src/lib/token-registry.ts` — one entry
- `~/solenbrowser/src/popup/App.tsx` — extend the `TokensList` row renderer (~30 lines)
- `~/solenbrowser/src/lib/rate-cache.ts` — new ~40 lines
- `~/solenbrowser/public/stsolen-logo.svg` — design asset

**Effort:** ~1 day.

---

## Cross-cutting concerns

### Reads that need adding to the contract before launch

The contract today exposes `exchange_rate`, `op_stake_of`, `pending_undelegate_op_of`, `owner`, `treasury`, `slash_oracle`, `paused`. Missing for the UI:

- **`withdrawal_at(seq[8])`** — return the raw 56-byte queue entry (`account[32] || solen_owed[16] || requested_epoch[8]`). The dispatcher accepts the method name but the body is currently a stub. Wire it to call `read_wq(seq)` and return the encoded bytes.
- **`pending_withdrawals_of(account[32]) -> u64`** — count of unclaimed entries belonging to an account. Iterate `wq_head..wq_tail`. Acceptable gas cost since it's only called by the Claims tab.

These are small contract additions; flag them now so they're not forgotten in the final pre-deploy pass.

### Network-aware contract address

Each surface needs to know the stSOLEN address per network. Centralize once:

- Add to `~/solenwallet/src/lib/networks.ts` as a `stsolen_address` field per network entry.
- Same in `~/solenbrowser/src/lib/networks.ts`.
- Dapp reads it from a config file or env.

Mainnet-only at v1; testnet/devnet entries can stay null, hiding the UI on those networks until a separate test deploy exists.

### Pause-state handling

Every action UI must check `paused()` first. Render a banner ("Deposits and withdrawals are paused. SRC-20 transfers and existing claims are unaffected.") and disable submit buttons. SRC-20 `transfer` remains available even when paused — this is intentional per the spec and shouldn't be hidden.

### Auto-crank on claim

Per the build decision: when a user clicks Claim and the `withdrawal_buffer` is too small, the dapp **automatically** calls `crank_undelegations` first, then `claim_withdrawal`. The user sees a single "Claim" button and a brief progress state. Log the auto-crank in the dapp's console for transparency, but don't surface it as a separate UX step.

### Solenswap pair seeding

Initial liquidity for the stSOLEN/SOLEN pair on solenswap is provided directly from the protocol/founder allocation. Build the dapp's "exit faster via solenswap" footer to link to the existing solenswap UI — no special seeding flow needed in the stSOLEN UI itself.

### Slashing alert surface

When the slash-oracle bot (`~/solen-stsolen-oracle/`) detects a slashed allowlisted operator, the operational team needs to act. Add a low-effort surface:

- Slash events publish to a Discord webhook (configurable in the bot's `.env`).
- Optional: a small `ops.solenchain.io` page that lists recent `slash_reported` events from stSOLEN.

Out of scope for stSOLEN UI proper; flagged here so it's not forgotten.

### Cranker bot

The Claims tab depends on `crank_undelegations` running with reasonable cadence (the auto-crank inside the dapp covers individual user payouts, but a background cranker also helps fill the buffer ahead of demand). Ship a tiny cranker bot alongside the dapp:

- Polls every minute.
- If any operator has `pending_undelegate_op > 0` OR `un_log_head` has matured entries, calls `crank_undelegations`.
- Run by the protocol team; permissionless so anyone can take over if it stalls.
- Sized similarly to the slashing-oracle bot (~150 lines TS).

### Wallet HD migration

The stSOLEN UI doesn't depend on HD-derived keys — single-key accounts work fine. HD migration (blocked on SLIP-0044 PR #2010) touches keystore code, not UI surfaces. They're orthogonal; build the UI now.

---

## Sequencing recommendation

| Order | Item | Effort | Gate |
|---|---|---|---|
| 0 | Add `withdrawal_at` + `pending_withdrawals_of` to the contract | 1 hr | Pre-deploy |
| 1 | Mainnet deploy of stSOLEN + cranker bot running | (out of scope here) | Mainnet |
| 2 | `solenstake/` dapp (read-only paths first, then write) | 5 days | Step 1 |
| 3 | solenbrowser popup balance row + APY pill | 1 day | Step 2 (logo, APY endpoint exist) |
| 4 | solenwallet `StakeStsolenCard` | 2 days | Step 2 (lib helpers) — copy them in |

Total: ~8–9 days of focused UI work after deployment. The dapp is the single biggest user-facing payoff.

---

## Locked design decisions

- **Branding:** "Staked SOLEN" / `stSOLEN`. No rebrand.
- **APY display:** 7-day headline, 30-day subtext, "since launch" fallback during the first 7 days.
- **Solenswap liquidity:** seeded directly from the protocol/founder allocation.
- **Auto-crank on claim:** user sees a single button; dapp transparently sends a crank op when needed.
- **Deployment scope:** mainnet only at v1. Testnet/devnet UI surfaces gated off until/unless a test deployment exists.
