# Quick Start

Get a Solen node running locally in a few minutes.

## Prerequisites

- **Rust 1.78+** — Install via [rustup](https://rustup.rs):
  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```
- **C build tools** — `build-essential` and `clang`:
  ```bash
  sudo apt install build-essential clang  # Ubuntu/Debian
  ```
- **WASM target** (for compiling contracts):
  ```bash
  rustup target add wasm32-unknown-unknown
  ```

## Build

```bash
git clone <repo-url> solen && cd solen
cargo build --workspace
```

!!! tip "RocksDB Compilation"
    On some Linux systems, RocksDB may need:
    ```bash
    export C_INCLUDE_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/include
    ```

## Run Tests

```bash
cargo test --workspace
```

This runs 135 tests including unit tests, 4 property-based invariant tests, and integration tests across all crates.

## Start a Node

```bash
cargo run --bin solen-node
```

This starts a single-validator devnet with:

| Service | Address |
|---------|---------|
| JSON-RPC | `http://127.0.0.1:29944` |
| Explorer API | `http://127.0.0.1:29955` |
| P2P | port `50333` |

The node produces blocks every 2 seconds and creates three genesis accounts:

| Account | Balance |
|---------|---------|
| `faucet` | 1,000,000,000 |
| `alice` | 10,000 |
| `bob` | 5,000 |

## Network Environments

Use `--network` to select the environment:

```bash
solen-node --network devnet    # default
solen-node --network testnet
solen-node --network mainnet
```

| | RPC Port | P2P Port | Explorer Port | Data Directory | Block Time |
|---|---|---|---|---|---|
| **mainnet** | 9944 | 30333 | 9955 | `data/mainnet` | 6s |
| **testnet** | 19944 | 40333 | 19955 | `data/testnet` | 2s |
| **devnet** | 29944 | 50333 | 29955 | `data/devnet` | 2s |

Any default can be overridden:

```bash
solen-node --network testnet --rpc-port 8888
```

## CLI Options

```
solen-node [OPTIONS]

Options:
    --network <NETWORK>        devnet, testnet, or mainnet [default: devnet]
    --rpc-port <PORT>          JSON-RPC port
    --p2p-port <PORT>          P2P listen port
    --data-dir <DIR>           RocksDB data directory
    --block-time <MS>          Block interval in ms
    --bootstrap <MULTIADDR>    Bootstrap peer address (repeatable)
    --validator-seed <HEX>     32-byte hex seed for validator key
    --no-p2p                   Disable P2P networking
    --in-memory                Use in-memory storage (no persistence)
    --explorer-port <PORT>     Explorer API port (0 to disable)
```

## Multi-Node Local Network

Run multiple nodes locally by assigning different ports and data directories:

```bash
# Terminal 1 — Node A
cargo run --bin solen-node

# Terminal 2 — Node B
cargo run --bin solen-node -- \
    --rpc-port 29945 \
    --p2p-port 50334 \
    --data-dir data/devnet-2 \
    --explorer-port 29956 \
    --bootstrap /ip4/127.0.0.1/tcp/50333
```

## Verify It Works

Once the node is running, check the chain status:

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

Or use the CLI:

```bash
cargo run --bin solen -- status
```

## Next Steps

- [Build Your First dApp](first-dapp.md) — Deploy a token contract
- [Running a Node](../tools/running-a-node.md) — Production node configuration
- [CLI Reference](../api/cli.md) — Full CLI command reference
