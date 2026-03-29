# Introduction to Solen

Solen is a **modular settlement network** designed around a few core principles:

1. **Minimal base layer** — L1 handles consensus, settlement, and verification only
2. **Native modularity** — Rollups, proofs, and cross-domain messaging are protocol primitives
3. **User safety by default** — Every account is a programmable smart account
4. **Bounded privacy** — Proof systems allow selective disclosure
5. **Decentralization over throughput** — Credible settlement over single-chain benchmarks

## What makes Solen different?

### Smart Accounts Only

There are no externally owned accounts (EOAs) in Solen. Every account is a smart account that supports:

- **Multiple authentication methods** — Ed25519 keys, passkeys (WebAuthn), threshold signatures
- **Guardians** — Social recovery for lost keys
- **Session credentials** — Temporary keys with spending limits
- **Programmable policies** — Custom authorization logic

### Native Rollups

Rollups are not bolted on — they are first-class protocol components:

- **Batch publishing** — Rollup sequencers submit state commitments to L1
- **Bridge vaults** — L1-L2 asset locking with challenge windows
- **Cross-domain messaging** — Rollups can message L1 and each other
- **Proof verification** — L1 verifies rollup proofs and updates state roots

### Intent-Aware Execution

Users can express **desired outcomes** instead of exact execution steps:

- "Swap token A for at least Y of token B"
- Solvers compete to fulfill intents under auditable rules
- Reduces MEV extraction and improves execution quality

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                        Solen Node                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │Consensus │  │Execution │  │   WASM   │  │  System       │  │
│  │  Engine  │◄─┤  Engine  │◄─┤    VM    │  │  Contracts    │  │
│  │  (BFT)   │  │          │  │(wasmtime)│  │               │  │
│  └────┬─────┘  └────┬─────┘  └──────────┘  │ ■ Staking     │  │
│       │              │                       │ ■ Bridge      │  │
│  ┌────┴─────┐  ┌────┴─────┐                │ ■ Governance  │  │
│  │   P2P    │  │ Storage  │                │ ■ Treasury    │  │
│  │(libp2p)  │  │(RocksDB) │                └───────────────┘  │
│  └──────────┘  └──────────┘                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ JSON-RPC │  │ Indexer  │  │ Explorer │                     │
│  │  :9944   │  │          │  │ API :9955│                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## Repository Structure

The Solen monorepo contains 17 Rust crates organized into layers:

| Layer | Crates |
|-------|--------|
| **Core** | `solen-types`, `solen-crypto`, `solen-storage` |
| **Protocol** | `solen-consensus`, `solen-execution`, `solen-vm` |
| **Systems** | `solen-system-contracts`, `solen-rollup-kit`, `solen-intents` |
| **Interface** | `solen-p2p`, `solen-rpc`, `solen-indexer` |
| **Applications** | `solen-node`, `solen-cli`, `solen-faucet`, `solen-contract-sdk` |

Plus two SDKs (`wallet-sdk-ts`, `wallet-sdk-rs`) and tooling for devnet/testnet deployment.

## Next Steps

- [Quick Start](quick-start.md) — Build and run a node
- [Build Your First dApp](first-dapp.md) — Deploy and interact with a smart contract
- [Architecture Overview](../architecture/overview.md) — Deep dive into system design
