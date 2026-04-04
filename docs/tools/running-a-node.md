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
    --prune                    Enable block pruning (default: archive mode)
    --explorer-port <PORT>     Explorer API port (0 to disable)
    --genesis <PATH>           Path to genesis.json config file
    --snapshot <URL|PATH>      Bootstrap from a state snapshot
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

## Snapshot Sync

New nodes can skip replaying the entire chain history by downloading a compressed state snapshot from an existing node. This is significantly faster than syncing block-by-block.

### Automatic (Recommended)

When a node starts with an empty data directory and has bootstrap peers configured, it **automatically** requests a snapshot from the seed nodes:

```bash
solen-node --network testnet \
    --bootstrap /dns4/testnet-seed1.solenchain.io/tcp/40333
```

The node will:

1. Detect that the store is empty
2. Call `solen_getSnapshot` on the seed node's RPC endpoint
3. Download and decompress the snapshot
4. Verify the state root matches
5. Resume normal sync from the snapshot height

### Manual

You can also provide a snapshot explicitly:

```bash
# From a file
solen-node --snapshot /path/to/snapshot.bin

# From an RPC endpoint
solen-node --snapshot https://testnet-rpc.solenchain.io
```

### Creating Snapshots

Any archive node can serve snapshots via the `solen_getSnapshot` RPC method:

```bash
curl -s -X POST https://testnet-rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getSnapshot","params":[],"id":1}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin)['result']; print(f'Height: {r[\"height\"]}, Entries: {r[\"entries\"]}, Size: {r[\"compressed_bytes\"]} bytes')"
```

The response includes:

| Field | Description |
|-------|-------------|
| `height` | Block height of the snapshot |
| `epoch` | Epoch at snapshot time |
| `state_root` | State root hash (verified on restore) |
| `entries` | Number of state entries |
| `compressed_bytes` | Compressed snapshot size |
| `data` | Base64-encoded snapshot (header + deflate-compressed KV pairs) |

### How It Works

- **Snapshot format:** 56-byte header (`SNAP` magic + version + height + epoch + state root) followed by deflate-compressed key-value pairs
- **State root verification:** After restoring, the node computes the state root over all loaded entries and verifies it matches the header. Corrupted or tampered snapshots are rejected.
- **Caching:** Archive nodes cache snapshots for 500 blocks (~16 minutes) and pre-warm the cache on startup. Serving a snapshot to new nodes is instant.
- **Non-blocking:** Snapshot creation uses RocksDB's native checkpoint API (hard-linked SST files) so the chain never pauses during snapshot generation.
- **Storage mode:** By default, nodes run in archive mode (keep all blocks). Use `--prune` to enable block pruning for nodes that only need recent history.

### Snapshot vs Full Sync

| | Snapshot Sync | Full Sync |
|---|---|---|
| **Speed** | Seconds to minutes | Hours to days (at scale) |
| **Data** | Current state only | Full block history |
| **Verification** | State root match | Every block re-executed |
| **When to use** | New validators, API nodes | Archive nodes, auditing |

!!! note
    Snapshot sync restores the current account state but does not include historical blocks. The node will have the correct balances and contract state, but transaction history before the snapshot height won't be available locally. The explorer indexer will only show transactions from the snapshot height forward.

## Becoming a Validator

Validators produce blocks and earn staking rewards. You need the minimum stake of **500,000 SOLEN** to register.

### 1. Generate a Validator Key

```bash
# Generate a new keypair
solen key generate my-validator

# Or import from a 32-byte hex seed
solen key import my-validator <64-char-hex-seed>
```

Your validator's account ID = public key. Fund this account with at least 500,000 SOLEN.

### 2. Register as Validator

Register with the minimum stake (500,000 SOLEN):

```bash
solen --rpc https://testnet-rpc.solenchain.io --chain-id 9000 \
  register-validator my-validator 500000
```

All CLI amounts are in **SOLEN** (not base units). Decimals are supported: `100.5` = 100.5 SOLEN.

This calls the staking system contract to register your account as a validator with self-stake. Your validator will be eligible for block production starting the next epoch (~100 blocks).

### 3. Run the Validator Node

```bash
solen-node \
    --network testnet \
    --genesis /path/to/genesis.json \
    --data-dir /opt/solen/data/testnet \
    --validator-seed <your-32-byte-hex-seed> \
    --bootstrap /dns4/testnet-seed1.solenchain.io/tcp/40333 \
    --bootstrap /dns4/testnet-seed2.solenchain.io/tcp/40333 \
    --bootstrap /dns4/testnet-seed3.solenchain.io/tcp/40333 \
    --bootstrap /dns4/testnet-seed4.solenchain.io/tcp/40333
```

The node will sync with the network, and once your stake is active, it will begin participating in consensus.

### 4. Verify

Check that your validator appears in the validator list:

```bash
solen --rpc https://testnet-rpc.solenchain.io validators
```

### Requirements

| Requirement | Value |
|-------------|-------|
| Minimum stake | 500,000 SOLEN |
| Unbonding period | 7 epochs |
| Genesis validator lock | ~1 year (157,680 epochs) |
| Default commission | 10% (1000 bps) |

### Validator Operations

#### Status and Monitoring

```bash
# List all validators and their stake/status
solen validators

# Check your account balance and nonce
solen account my-validator
```

#### Staking

```bash
# Add more self-stake to your validator
solen stake my-validator <your-validator-address> <amount>

# Delegate to another validator
solen stake my-validator <other-validator-address> <amount>

# Begin unstaking (enters 7-epoch unbonding period)
solen unstake my-validator <validator-address> <amount>

# Withdraw matured unstaked tokens after unbonding completes
solen withdraw-stake my-validator
```

All amounts are in **SOLEN** (not base units). Decimals are supported: `100.5` = 100.5 SOLEN.

#### Slashing and Unjailing

If your validator misses **50 consecutive blocks**, it will be automatically slashed:

- **1% of self-stake** is deducted and sent to the Foundation Treasury
- The validator is **jailed** (removed from the active consensus set)
- The validator stops earning rewards and producing blocks

To reactivate a jailed validator:

```bash
solen unjail my-validator
```

The validator rejoins the active set at the **next epoch boundary** (every 100 blocks). Slashed funds held in the treasury can be restored via a governance proposal if the community deems it appropriate.

!!! warning
    While jailed, your node is still running but not participating in consensus. Delegators staked with a jailed validator do not earn rewards. Reactivate promptly to minimize downtime for yourself and your delegators.

#### Exiting (Full Unstake)

To permanently exit as a validator:

1. Unstake your entire self-stake:
    ```bash
    solen unstake my-validator <your-validator-address> <full-amount>
    ```
2. Wait for the 7-epoch unbonding period
3. Withdraw the matured tokens:
    ```bash
    solen withdraw-stake my-validator
    ```

Once your self-stake drops below the minimum (500,000 SOLEN), the validator is removed from the active set at the next epoch boundary. You can stop the node after withdrawing.

!!! note
    Genesis validators are locked for ~1 year (157,680 epochs) and cannot unstake during the lock period.

#### Re-entry

A validator that previously exited can re-register with a new stake:

```bash
solen register-validator my-validator 500000
```

The same key and node can be reused. The validator rejoins the active set at the next epoch boundary after registration.

### Important Notes

- **Never share your validator seed.** It controls your validator identity and staked funds.
- **Never delete the data directory** while the chain is running. Restart is fine, but wiping data requires a full resync (or snapshot sync).
- **Keep your node online.** Extended downtime (50+ consecutive missed blocks) results in a 1% slash and jailing. Use `solen unjail` to reactivate.
- Your validator earns epoch rewards proportional to its stake. Default commission is 10% on delegator rewards.
- Slashing is **deterministic and on-chain** — it executes as a system transaction in a block, so all nodes apply it identically.

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
