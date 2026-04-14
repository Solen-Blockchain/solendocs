# Execution Engine

The execution engine processes user operations, manages account state, and coordinates with the WASM VM for contract execution.

## User Operations

All transactions are submitted as **user operations** — signed action bundles from smart accounts:

```rust
UserOperation {
    sender: AccountId,       // 32-byte account identifier
    nonce: u64,              // Replay protection
    actions: Vec<Action>,    // One or more actions
    max_fee: u128,           // Maximum fee willing to pay
    signature: Vec<u8>,      // Ed25519 signature
}
```

### Action Types

| Action | Description | Base Gas |
|--------|-------------|----------|
| `Transfer` | Send native tokens to another account | 100 |
| `Call` | Invoke a deployed contract method | 500 + VM cost |
| `Deploy` | Deploy new WASM contract bytecode | 1,000 |

## Execution Flow

1. **Signature verification** — Ed25519 signature check against the sender's public key
2. **Nonce check** — Operation nonce must equal account nonce
3. **Balance check** — Sender must have sufficient balance for `max_fee`
4. **Action execution** — Each action is executed sequentially
5. **Fee deduction** — Actual gas used is deducted from sender's balance
6. **Fee distribution** — 50% burned, 50% to treasury
7. **State commit** — Updated state root is computed

## Transaction Simulation

The `solen_simulateOperation` RPC method performs a dry-run of an operation without committing state changes. This is useful for:

- Estimating gas costs
- Checking if an operation will succeed
- Previewing state changes

## State Management

Account state is stored using a key-value store with a Merkle-ized state root:

- **MemoryStore** — In-memory storage for testing and development
- **RocksDB** — Persistent storage for production nodes

The `StateStore` trait abstracts over both backends, allowing pluggable storage.

## Genesis

Genesis initialization creates the initial state:

- Deploys system contracts (staking, bridge, governance, treasury)
- Creates initial accounts with balances
- Sets up the initial validator set
- Configures protocol parameters

## Performance

Benchmarked on a live 4-validator testnet with BFT consensus and full Ed25519 signature verification.

| Benchmark | TPS | Finality |
|-----------|-----|----------|
| Transfers (live testnet) | ~1,200 | < 1s |
| Contract calls (WASM, live testnet) | ~188 | < 1s |
| Max ops per block | 5,000 | — |
| RPC ingestion rate | 10,000+ tx/s | — |

All numbers measured with full BFT quorum (2/3+ stake-weighted attestations), not single-node execution.
