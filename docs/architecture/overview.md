# Architecture Overview

Solen is designed as a minimal settlement layer. The base chain handles consensus and finality while execution domains (rollups) handle application-specific computation.

## Node Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Solen Node                               │
│                                                                 │
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
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ JSON-RPC │  │ Indexer  │  │ Explorer │                     │
│  │  :9944   │  │          │  │ API :9955│                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## Components

### Settlement Layer

The **consensus engine** implements BFT proof-of-stake with deterministic finality, stake-weighted randomized proposer selection, 2/3+ quorum, epoch-based rewards, finalized checkpoints, and slashing.

### Execution Engine

Processes user operations (transfer, contract call, deploy), manages account state, verifies signatures, deducts fees, and supports transaction simulation.

### WASM VM

Contracts execute as WASM bytecode via [wasmtime](https://wasmtime.dev/). Host functions provide storage read/write, event emission, caller identity, and block height. Gas metering uses wasmtime's fuel mechanism.

### Smart Accounts

Every account is programmable. There are no externally owned accounts. Supports Ed25519 keys, passkeys, threshold authorization, guardians, and session credentials.

### System Contracts

Built-in privileged contracts for:

- **Staking** — Validator registration, delegation, rewards
- **Bridge** — L1-L2 asset transfers and challenge windows
- **Governance** — On-chain proposals, voting, timelocks
- **Treasury** — Protocol fund management

### Networking

**P2P** uses libp2p gossipsub for block and transaction propagation, peer discovery, and validator communication.

**JSON-RPC** provides a standard JSON-RPC 2.0 interface for querying state and submitting operations.

**Indexer** tracks blocks, transactions, and events for the Explorer REST API.

## Crate Map

```
solen/crates/
├── solen-types/             # Core types: Hash, AccountId, BlockHeader, UserOperation
├── solen-crypto/            # Ed25519 signing, BLAKE3 hashing
├── solen-storage/           # StateStore trait + MemoryStore + RocksDB backend
├── solen-consensus/         # BFT engine, validator set, epochs, slashing
├── solen-execution/         # Block executor, state manager, fees, genesis
├── solen-vm/                # Wasmtime WASM runtime with host functions
├── solen-system-contracts/  # Staking, bridge, governance, treasury
├── solen-rollup-kit/        # Sequencer, batch publisher, prover, messenger
├── solen-intents/           # Intent resolution and solver framework
├── solen-p2p/               # libp2p gossipsub networking
├── solen-rpc/               # JSON-RPC server (jsonrpsee)
├── solen-indexer/           # Event indexer + explorer REST API
├── solen-node/              # Node binary
├── solen-cli/               # CLI tool
├── solen-faucet/            # Testnet faucet
├── solen-contract-sdk/      # Contract development SDK
└── solen-wallet-sdk/        # Rust wallet SDK
```

## Design Principles

| Principle | Implication |
|-----------|-------------|
| Minimal base layer | L1 handles consensus, settlement, and verification — not general-purpose execution |
| Native modularity | Rollups, proofs, and messaging are protocol-native |
| User safety by default | Every account is a smart account with recovery and policy |
| Bounded privacy | Proof systems allow selective disclosure without full transparency |
| Decentralization over throughput | Credible settlement over single-chain benchmarks |

## Deep Dives

- [Consensus](consensus.md) — BFT proof-of-stake details
- [Execution Engine](execution.md) — Transaction processing pipeline
- [Smart Accounts](smart-accounts.md) — Account model and authentication
- [Rollups](rollups.md) — Native rollup framework
- [Intents](intents.md) — Intent-aware execution
