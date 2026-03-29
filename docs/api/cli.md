# CLI Reference

The `solen` CLI provides commands for interacting with Solen nodes from the terminal.

## Installation

```bash
cargo build --bin solen --release
# Binary at: ./target/release/solen
```

## Commands

### `solen status`

Show the current chain status.

```bash
solen status
```

```
Solen Network Status
────────────────────────────────────────
  Height:      12
  State root:  78c88b3eb083...805d24e9
  Pending ops: 0
  Epoch:       0
  Proposer:    8a88e3dd7409f195...
  Gas used:    0
```

---

### `solen balance <account>`

Query an account's native token balance. Account can be a key name or hex address.

```bash
solen balance faucet
# Output: 1000000000
```

---

### `solen account <account>`

Get full account details including nonce and code hash.

```bash
solen account faucet
```

```
Account
────────────────────────────────────────
  ID:        197f6b23e16c8532...
  Balance:   1000000000
  Nonce:     0
  Code hash: (none)
```

---

### `solen block [height]`

Get block information. Shows the latest block if no height is given.

```bash
solen block          # latest block
solen block 100      # block at height 100
```

```
Block #100
────────────────────────────────────────
  Epoch:      1
  Proposer:   8a88e3dd7409f195...
  State root: 78c88b3eb08313c6...
  Txs:        2
  Gas used:   200
  Time:       4s ago
```

---

### `solen transfer <from> <to> <amount>`

Transfer tokens between accounts. Simulates first, then submits.

```bash
solen transfer mykey 139c31e8543b1962... 5000
```

The sender must be a key name in your keystore. The recipient can be a key name or hex address.

---

### `solen key` — Key Management

#### `solen key generate <name>`

Generate a new Ed25519 keypair.

```bash
solen key generate alice
```

#### `solen key import <name> <hex-seed>`

Import a key from a 32-byte hex seed.

```bash
solen key import faucet 2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a
```

#### `solen key list`

List all stored keys.

```bash
solen key list
```

```
NAME         ACCOUNT ID           PUBLIC KEY
──────────────────────────────────────────────────────────────────────
alice        139c31e8543b1962...  139c31e8543b1962...
faucet       197f6b23e16c8532...  197f6b23e16c8532...
```

---

### `solen validators`

List all validators and their stake.

```bash
solen validators
```

```
STATUS ADDRESS            SELF STAKE      DELEGATED          TOTAL
──────────────────────────────────────────────────────────────────────
GENSIS 8a88e3dd7409f195...        1000000              0        1000000
GENSIS 8139770ecb104aef...        1000000              0        1000000
GENSIS ed4928c628d1c2c6...        1000000              0        1000000
GENSIS ca93ac1705187071...        1000000              0        1000000

4 validators (4 active)
```

---

### `solen stake <from> <validator> <amount>`

Delegate tokens to a validator.

```bash
solen stake mykey 8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c 10000
```

### `solen unstake <from> <validator> <amount>`

Begin undelegation from a validator. Funds are available after the unbonding period (7 epochs).

```bash
solen unstake mykey 8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c 5000
```

---

### System Contract Addresses

System contracts have well-known addresses:

| Contract | Address |
|----------|---------|
| Staking | `0xFFFF...FF01` |
| Governance | `0xFFFF...FF02` |
| Bridge | `0xFFFF...FF03` |
| Treasury | `0xFFFF...FF04` |

Call them using `solen call`:

```bash
# Delegate to a validator via system call
solen call mykey ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff01 delegate --args "<validator><amount>"
```

4 validators (4 active)
```

---

### `solen deploy <from> <wasm-file>`

Deploy a WASM contract.

```bash
solen deploy faucet target/wasm32-unknown-unknown/release/my_contract.wasm
```

The CLI will:

1. Simulate the deployment to estimate gas
2. Submit the deploy operation
3. Print the contract ID and code hash

---

### `solen call <from> <contract-id> <method> [--args <hex>]`

Call a contract method.

```bash
# No arguments
solen call faucet <CONTRACT_ID> increment

# With arguments (hex-encoded)
solen call faucet <CONTRACT_ID> mint --args "197f6b23e16c8532...40420f0000..."
```

The CLI simulates the call first, then submits if simulation succeeds.

---

### Global Options

| Option | Description |
|--------|-------------|
| `--rpc <URL>` | JSON-RPC endpoint (default: `http://127.0.0.1:9944`) |
| `--network <NETWORK>` | Use network defaults (devnet, testnet, mainnet) |

```bash
# Use a different RPC endpoint
solen --rpc http://127.0.0.1:29944 status

# Use testnet defaults
solen --network testnet status
```
