# Agent SDK

`@solen/agent-sdk` is the **agent-side** library. It lets an AI agent transact as a session key on the owner's account — signing, sponsorship, and limit-aware pre-flight — without ever holding the owner's primary key.

For the **owner side** (creating and revoking the grant), see [Granting & Revoking Agents](granting.md).

## Install

```bash
npm install @solen/agent-sdk
```

The agent's signing is verified byte-for-byte against the node's canonical [transaction-signing](../specs/transaction-signing.md) vectors, so its operations are accepted exactly like any other.

## Create an agent

```typescript
import { SolenAgent } from "@solen/agent-sdk";

const SOLEN = 100_000_000n; // 1 SOLEN = 1e8 base units

const agent = await SolenAgent.create({
  rpcUrl: "https://rpc.solenchain.io",
  chainId: 1,                         // 1 mainnet · 9000 testnet · 1337 devnet
  owner: "5Z6Ay5NEcbg3xhopc522sBCRXQujkTiuDRnHGfQdcnSf",
  sessionSecretKey: agentSecretHex,   // the scoped session key (32-byte seed, hex or bytes)
  grant: {                            // optional local mirror for can()/budget()
    budgetTotal: 10n * SOLEN,
    allowedMethods: ["transfer"],
  },
});
```

| Config | Type | Notes |
|--------|------|-------|
| `rpcUrl` | `string` | JSON-RPC endpoint. |
| `chainId` | `number` | `1` mainnet · `9000` testnet · `1337` devnet. |
| `owner` | `string` | The account the agent acts on (Base58 or hex). |
| `sessionSecretKey` | `string \| Uint8Array` | The agent's Ed25519 secret (32-byte seed, or 64-byte seed+public). |
| `grant` | `SessionGrant?` | Local copy of the granted limits. The chain is the source of truth; this only powers `can()` / `budget()`. |
| `sponsor` | `boolean?` | Try paymaster sponsorship (default `true`). |
| `maxFee` | `bigint?` | Fee ceiling when not sponsored (default `100000`). |

## Transact

The agent acts *as the owner* (`sender = owner`), authorized by its session key. Submission does `getNextNonce → sign → checkSponsorship → submit`.

```typescript
// Native transfer
const res = await agent.transfer(recipient, 1n * SOLEN);
// → { opHash, sponsored, paymaster?, nonce }

// Contract call
await agent.call(tokenContract, "transfer", argsBytes);

// Arbitrary actions
await agent.submit([{ kind: "transfer", to, amount: 1n * SOLEN }]);
```

## Pre-flight and limits

`simulate()` is the **authoritative** check — it runs the operation against current chain state (budget, targets, methods, expiry, balance) without spending anything:

```typescript
const sim = await agent.simulate([{ kind: "transfer", to: recipient, amount: 1n * SOLEN }]);
if (!sim.success) {
  console.error("would be rejected:", sim.error);
}
```

`can()` / `explain()` are fast, local checks against the grant you configured (no network):

```typescript
if (agent.can({ kind: "transfer", to: recipient, amount: 1n * SOLEN })) {
  await agent.transfer(recipient, 1n * SOLEN);
}

agent.budget();       // { total, spentThisSession, remaining }
await agent.isExpired(); // checks the session expiry against current height
```

## Method reference

| Method | Returns | Description |
|--------|---------|-------------|
| `SolenAgent.create(config)` | `Promise<SolenAgent>` | Derive the public key and build the agent. |
| `agent.transfer(to, amount, opts?)` | `Promise<SubmitResult>` | Build + sign + submit a native transfer. |
| `agent.call(target, method, args?, opts?)` | `Promise<SubmitResult>` | Build + sign + submit a contract call. |
| `agent.submit(actions, opts?)` | `Promise<SubmitResult>` | Submit arbitrary actions. |
| `agent.simulate(actions)` | `Promise<SimulationResult>` | Authoritative server-side pre-flight. |
| `agent.checkSponsorship(actions)` | `Promise<SponsorshipResult>` | Whether a paymaster will cover fees. |
| `agent.can(action)` / `agent.explain(action)` | `boolean` / `string?` | Local check against the configured grant. |
| `agent.budget()` | `BudgetView` | Best-effort budget view from this instance. |
| `agent.isExpired()` | `Promise<boolean>` | Expiry check against current chain height. |
| `agent.sessionPublicKey` / `agent.sessionAccountId` | `string` | The session key's public key (hex) and id (Base58). |

!!! note "Amounts"
    Amounts are **base units** (1 SOLEN = 10<sup>8</sup>). The JSON signing path reproduces the node's encoding losslessly up to 2<sup>53</sup>-1 base units (~90M SOLEN per field); larger single amounts throw a clear error. Agent budgets are bounded by design, so this is not a practical limit.

!!! warning "`budget()` is local"
    `budget()` reflects spend tracked by *this* SDK instance. The chain is authoritative — it rejects over-budget operations regardless. Use `simulate()` to confirm an operation will be accepted across processes or restarts.
