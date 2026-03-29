# Fee Model

Solen uses a simple, transparent fee model with a deflationary burn mechanism.

## Parameters

| Parameter | Default | Governance |
|-----------|---------|------------|
| Base fee per gas | 1 | Adjustable |
| Burn rate | 50% | Adjustable |
| Treasury rate | 50% | Adjustable |

## Gas Costs

| Operation | Base Gas |
|-----------|----------|
| Transfer | 100 |
| Contract call | 500 + VM execution cost |
| Deploy | 1,000 |

Contract execution gas depends on the number of WASM instructions executed, measured by wasmtime's fuel mechanism.

## Fee Flow

```
User pays fee
├── 50% burned (removed from supply permanently)
└── 50% to treasury (governed by on-chain governance)
```

### Deflationary Pressure

The burn creates deflationary pressure that partially offsets staking rewards. At high transaction volumes, the burn rate can exceed reward issuance, making the token **net-deflationary**.

## Fee Calculation

```
fee = gas_used × base_fee_per_gas
```

The `max_fee` field in a user operation caps the maximum fee. If actual gas used is less than `max_fee`, only the actual cost is deducted.

If `gas_used × base_fee_per_gas > max_fee`, the operation fails.

## Gas Abstraction

Users don't need to hold native tokens to transact. **Paymasters** can sponsor fees on behalf of users, enabling:

- **Sponsored transactions** — dApps pay for their users
- **Alternative fee tokens** — Pay fees in approved tokens
- **Session-based policies** — Pre-authorized spending limits

## Rollup Fees

Rollups pay L1 fees for:

- Batch publication
- Proof verification
- Bridge operations

These follow the same burn/treasury split. Rollup sequencers may charge additional L2 fees independently.
