# Spec 001: Terminology

**Status:** Draft

## Definitions

| Term | Definition |
|------|-----------|
| **Settlement Layer (L1)** | The Solen base chain responsible for consensus, finality, and canonical state. |
| **Execution Domain** | A rollup or app-specific environment that settles to Solen L1. |
| **Smart Account** | A programmable account with authentication policies; the only account type in Solen. No externally owned accounts exist. |
| **User Operation** | A signed action bundle submitted by a smart account. Contains one or more actions (transfer, call, deploy). |
| **Intent** | A declarative expression of a desired outcome, resolved by solvers/bundlers rather than executed directly. |
| **Batch Commitment** | A state root and proof published by a rollup sequencer to L1, attesting to the result of L2 execution. |
| **Epoch** | A fixed interval of blocks used for validator rotation, reward distribution, and checkpoint creation. |
| **Solver** | An off-chain agent that competes to fulfill user intents with optimal execution. |
| **Paymaster** | An account that sponsors transaction fees on behalf of other accounts, enabling gasless UX. |
| **Bridge Vault** | An L1 contract that holds assets locked for cross-domain transfers between L1 and rollups. |
| **Slashing** | Penalty applied to validators (and their delegators) for misbehavior such as double signing or downtime. |
| **Unbonding** | The cooldown period after unstaking, during which tokens cannot be transferred but are still subject to slashing. |
