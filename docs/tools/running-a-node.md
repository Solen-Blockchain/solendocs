# Running a Node

This guide covers running a Solen node for different environments.

## Quick Start

```bash
# Build
cargo build --bin solen-node --release

# Run (devnet defaults)
./target/release/solen-node
```

## Configuration

### CLI Options

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

### Network Defaults

| | RPC Port | P2P Port | Explorer Port | Data Directory | Block Time |
|---|---|---|---|---|---|
| **mainnet** | 9944 | 30333 | 9955 | `data/mainnet` | 6s |
| **testnet** | 19944 | 40333 | 19955 | `data/testnet` | 2s |
| **devnet** | 29944 | 50333 | 29955 | `data/devnet` | 2s |

## Development Mode

For local development, use in-memory storage and disable P2P:

```bash
solen-node --in-memory --no-p2p --block-time 1000
```

This gives you:

- 1-second block times for fast iteration
- No disk persistence (clean state on restart)
- No peer connections needed

## Multi-Node Local Network

```bash
# Terminal 1 — Node A (seed node)
./target/release/solen-node

# Terminal 2 — Node B (connects to Node A)
./target/release/solen-node \
    --rpc-port 29945 \
    --p2p-port 50334 \
    --data-dir data/devnet-2 \
    --explorer-port 29956 \
    --bootstrap /ip4/127.0.0.1/tcp/50333
```

## Production Deployment

For production or testnet deployment, see [Testnet Deployment](testnet-deployment.md).

### systemd Service

```ini
[Unit]
Description=Solen Node
After=network.target

[Service]
Type=simple
User=solen
ExecStart=/usr/local/bin/solen-node --network mainnet
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Storage | 100 GB SSD | 500 GB NVMe |
| Network | 100 Mbps | 1 Gbps |

### Data Backup

RocksDB data is stored in the `--data-dir` directory. Back up this directory to preserve chain state.

```bash
# Stop the node before backing up
systemctl stop solen-node
cp -r data/mainnet data/mainnet-backup
systemctl start solen-node
```

## Monitoring

### JSON-RPC Health Check

```bash
curl -s http://127.0.0.1:9944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

### Explorer API Status

```bash
curl -s http://127.0.0.1:9955/api/status
```

### Logs

The node logs to stderr using the `tracing` crate. Set log level with `RUST_LOG`:

```bash
RUST_LOG=info solen-node
RUST_LOG=solen_consensus=debug solen-node
```
