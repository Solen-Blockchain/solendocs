# Native Rollups

Rollups are first-class protocol components in Solen, not external add-ons. The `solen-rollup-kit` crate provides a complete framework for building and operating rollups that settle to Solen L1.

## Components

### Sequencer

Collects L2 transactions, orders them, and produces L2 blocks. The sequencer is responsible for:

- Transaction ordering and inclusion
- L2 block production
- State transition computation

### Batch Publisher

Periodically submits **batch commitments** to L1:

- State root after executing the batch
- Transaction data or data availability proof
- Proof of correct execution (if applicable)

### Prover Adapter

Interfaces with proof systems to generate validity proofs for L2 state transitions. Supports pluggable proof backends.

### Cross-Domain Messenger

Enables messaging between:

- L2 → L1 (withdrawal messages)
- L1 → L2 (deposit messages)
- L2 → L2 (cross-rollup messages, relayed through L1)

## Bridge Vaults

Each rollup has an associated **bridge vault** on L1 that manages asset transfers:

| Operation | Mechanism |
|-----------|-----------|
| **Deposit** (L1 → L2) | Instant — tokens locked in vault, minted on L2 |
| **Withdrawal** (L2 → L1) | Challenge window (100 blocks) + withdrawal delay (50 blocks) |
| **Dispute** | Disputed withdrawals are blocked pending resolution |

## Rollup Registration

Rollups are registered on L1 through governance proposals. Each rollup record includes:

```
Rollup {
    rollup_id: Hash,
    vm_type: VmType,           // WASM, EVM, custom
    proof_type: ProofType,     // Validity, fraud, none
    da_mode: DaMode,           // On-chain, external DA
    bridge_config: BridgeConfig,
    sequencer_set: Vec<AccountId>,
    governance_params: GovernanceParams,
}
```

## Economics

Rollups pay L1 fees for:

- Batch publication transactions
- Proof verification transactions
- Bridge operations

These fees follow the same [burn/treasury split](../tokenomics/fee-model.md) as regular L1 transactions.

Rollup sequencers may charge additional L2 fees which are independent of L1 tokenomics.
