# Granting & Revoking Agents

Granting an agent is an **owner-side** action: you add a scoped `Session` auth method to your account with a [`SetAuth`](../architecture/smart-accounts.md) operation, signed by your primary key. Revoking is the same operation with the session removed.

`SetAuth` *replaces* the account's full auth-method list, so a grant must be composed from your **current** methods plus the new session — never clobbering your primary key or other agents. The wallet and SDKs handle this for you.

## From a dapp — `@solen/web-sdk`

Granting a session key is delegated to the wallet (it knows your current methods and holds your key). The web SDK forwards the request:

```typescript
import { getSolenProvider, generateAgentKey } from "@solen/web-sdk";

const provider = await getSolenProvider();
await provider.connect();

// 1. Mint the agent's session key. Keep the secret for the agent runtime
//    (feed it to @solen/agent-sdk); hand the wallet only the public key.
const agentKey = generateAgentKey();

// 2. Ask the wallet to grant it (amounts are base units; 1 SOLEN = 1e8).
provider.on("grantResult", ({ result, error }) => {
  if (error) return console.error("grant failed:", error);
  console.log("granted, op:", result.opHash);
});
await provider.grantAgent({
  agentPublicKey: agentKey.publicKey,
  grant: {
    budgetTotal: "1000000000",      // 10 SOLEN lifetime
    spendingLimit: "100000000",     // 1 SOLEN per op
    allowedMethods: ["transfer"],
    expiresAt: currentHeight + 100_000,
    restrictSubcalls: false,
  },
});

// Later: revoke it.
await provider.revokeAgent({ agentPublicKey: agentKey.publicKey });
```

On desktop these resolve through the Solen Browser extension's `window.solen`. On mobile they navigate to `solen://grant` / `solen://revoke`; listen via `provider.on("grantResult", …)` for the result.

## From the desktop wallet

The Solen Wallet desktop app has an **Agents** tab where you can, for the active account:

- **List** every granted agent with its budget, per-op cap, methods, targets, expiry, and sub-call lockdown — read live from the chain.
- **Grant** a new agent: generate a key (the secret is shown **once** — save it for your agent runtime), set the limits, and submit.
- **Revoke** any agent in one click.

## Building the grant yourself

`@solen/agent-sdk` exposes the builders if you want to compose the `SetAuth` directly (e.g. in a custodial backend that holds the owner key):

```typescript
import { grantAgentAction, setAuthAction, ed25519AuthMethod } from "@solen/agent-sdk";

const SOLEN = 100_000_000n;

const action = grantAgentAction(
  [ed25519AuthMethod(ownerPublicKey)],   // your current methods — keep them!
  agentPublicKey,
  {
    budgetTotal: 10n * SOLEN,
    spendingLimit: 1n * SOLEN,
    allowedMethods: ["transfer"],
    expiresAt: currentHeight + 100_000,
    restrictSubcalls: true,
  },
);
// → sign an operation carrying `action` with the owner key and submit it.
// Revoke later with setAuthAction([ed25519AuthMethod(ownerPublicKey)]).
```

## The Session auth method

A grant adds this to your account's `auth_methods`:

```json
{
  "Session": {
    "session_key": "agent-ed25519-public-key",
    "expires_at": 800000,
    "spending_limit": 100000000,
    "budget_total": 1000000000,
    "allowed_targets": ["contract-id"],
    "allowed_methods": ["transfer"],
    "restrict_subcalls": false
  }
}
```

| Field | Meaning |
|-------|---------|
| `session_key` | The agent's Ed25519 public key. |
| `expires_at` | Block height after which the session is rejected. |
| `spending_limit` | Per-operation spend cap (base units). `0` = none. |
| `budget_total` | Cumulative lifetime spend cap (base units). `0` = unlimited. |
| `allowed_targets` | Allowed call targets / transfer recipients. Empty = all. |
| `allowed_methods` | Allowed call methods. Empty = all. |
| `restrict_subcalls` | Enforce the allowlist on contract sub-calls too (whole call tree). |

You can read an account's current auth methods — to list agents or compose a `SetAuth` without clobbering — from [`solen_getAccount`](../api/json-rpc.md), which returns them in the `auth_methods` field.

!!! note "How the budget is tracked"
    The cumulative spend for a session key is tracked on-chain and **charged only when an operation succeeds**. A reverted or limit-rejected operation consumes no budget, and a budget-rejected operation doesn't consume the account nonce — so an agent can simply retry with a smaller amount.

!!! warning "Layout note for integrators"
    `budget_total` and `restrict_subcalls` are part of the on-chain `Session` encoding. Build the method with the fields in the order shown above so the operation signature verifies.
