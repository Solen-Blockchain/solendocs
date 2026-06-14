# AI Agents

Solen lets an AI agent act on your behalf with limits the **chain itself enforces**. You delegate a *scoped session key* — never your main account key — and consensus guarantees the agent can only do what you allowed, for as long as you allow it.

This is built on [smart accounts](../architecture/smart-accounts.md): an agent is just a `Session` auth method added to your account, with a budget, an allowlist, and an expiry. Because the rules live in consensus, a compromised agent key **cannot** widen its own scope, drain your account, or touch a contract you didn't approve.

## The lifecycle

The agent lifecycle is three steps:

=== "1. Grant"

    From your wallet or a dapp, mint a fresh Ed25519 **session key** and scope it: a lifetime budget, a per-operation cap, an allowlist of contracts and methods, an expiry height, and optional sub-call lockdown. You sign the grant with your account key (a `SetAuth` operation). The agent receives only the scoped key.

    See [Granting & Revoking Agents](granting.md).

=== "2. Spend"

    The agent transacts autonomously through [`@solen/agent-sdk`](agent-sdk.md) — **gasless** when a paymaster sponsors it, so the agent needs no native balance. Every operation is validated against the session's limits *during execution*. Anything over budget, off the allowlist, or past expiry is rejected on-chain.

=== "3. Revoke"

    Remove the agent at any time with another `SetAuth` (one click in the desktop wallet's **Agents** tab). Session keys also expire automatically at the block height you set, so an agent's authority is always bounded.

## What the chain enforces

Every operation an agent signs is checked in consensus before it executes:

| Limit | Field | Behavior |
|-------|-------|----------|
| **Lifetime budget** | `budget_total` | Cumulative spend across all of the agent's operations. Tracked on-chain; charged only when an operation succeeds (reverts don't burn budget). `0` = unlimited. |
| **Per-operation cap** | `spending_limit` | Maximum spend in a single operation. `0` = no per-op cap. |
| **Allowed targets** | `allowed_targets` | Contracts the agent may call and addresses it may transfer to. Empty = all. |
| **Allowed methods** | `allowed_methods` | Contract methods the agent may call. Empty = all. |
| **Expiry** | `expires_at` | Block height after which the key is dead. |
| **Sub-call lockdown** | `restrict_subcalls` | When `true`, the target/method allowlist is enforced on the *entire* call tree — including contract→contract sub-calls — not just the top-level operation. |

Amounts are in **base units** (1 SOLEN = 10<sup>8</sup> base units).

## Why it's safe

- **Scoped key, not your key.** The agent holds a session key. Your primary key never leaves your wallet, and the agent's signature is only valid for operations that satisfy its restrictions.
- **Can't escalate.** Session keys are blocked at the protocol level from `SetAuth`, contract deploys, guardian recovery, and creating/finalizing/executing governance proposals — so an agent can never widen its own cage or rotate your auth. (It *can* vote.)
- **Sub-calls can't touch your funds.** A queued contract→contract sub-call runs as the *calling contract*, on its own behalf — never as your account — so it cannot move your balance. `restrict_subcalls` adds defense-in-depth on top of this by bounding which contracts an agent's operations may transitively reach.
- **Budget is honest.** The cumulative budget is charged only when an operation actually succeeds; a reverted or rejected operation consumes none of it.

## Gasless by design

Agents transact through the [paymaster](../architecture/smart-accounts.md) sponsorship mechanism. Before submitting, the SDK calls `solen_checkSponsorship`; if a registered paymaster will cover the fee, the agent transacts with **zero native balance**. Fund the work, not the wallet.

## Next steps

- [Agent SDK](agent-sdk.md) — build an agent with `@solen/agent-sdk`.
- [Granting & Revoking Agents](granting.md) — the owner side: scope, grant, and revoke session keys.
- [Smart Accounts](../architecture/smart-accounts.md) — the account model agents are built on.
