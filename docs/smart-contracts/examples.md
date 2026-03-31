# Example Contracts

Solen includes two reference contracts in `examples/contracts/`.

## Counter

A minimal contract demonstrating storage, events, and return values.

**Source:** `examples/contracts/counter/src/lib.rs`

### Methods

| Method | Arguments | Description |
|--------|-----------|-------------|
| `increment` | — | Increment the counter by 1 |
| `get` | — | Return the current count |

### Key Concepts

- Reading and writing to storage (`storage_read`, `storage_write`)
- Emitting events (`emit_event`)
- Returning data to the caller (`set_return_data`)

### Usage

```bash
# Build
cd examples/contracts/counter
cargo build --target wasm32-unknown-unknown --release

# Deploy
solen deploy faucet target/wasm32-unknown-unknown/release/solen_example_counter.wasm

# Increment
solen call faucet <CONTRACT_ID> increment

# Get current value
solen call faucet <CONTRACT_ID> get
```

---

## Token (SRC-20)

A full ERC20-equivalent token contract with minting, transfers, and allowances.

**Source:** `examples/contracts/token/src/lib.rs`

### Methods

| Method | Arguments | Description |
|--------|-----------|-------------|
| `abi` | — | Returns JSON array of all methods |
| `init` | `name_len[1]+name[]+symbol_len[1]+symbol[]` | Initialize with metadata |
| `mint` | `to (32 bytes) + amount (u128 LE)` | Mint tokens to an account (owner only) |
| `transfer` | `to (32 bytes) + amount (u128 LE)` | Transfer tokens from caller to recipient |
| `approve` | `spender (32 bytes) + amount (u128 LE)` | Approve spender to use caller's tokens |
| `transfer_from` | `from (32 bytes) + to (32 bytes) + amount (u128 LE)` | Transfer using allowance |
| `balance_of` | `account (32 bytes)` | Query token balance |
| `allowance` | `owner (32 bytes) + spender (32 bytes)` | Query allowance |
| `total_supply` | — | Query total token supply |
| `name` | — | Token name (UTF-8 string) |
| `symbol` | — | Token symbol/ticker |
| `decimals` | — | Decimal places (default 8) |
| `owner` | — | Contract owner address |

### Argument Encoding

Arguments are passed as raw bytes. Account IDs are 32 bytes, amounts are 16-byte little-endian u128 values.

```bash
# Build the amount argument
AMOUNT=$(python3 -c "print((1000000).to_bytes(16, 'little').hex())")

# Account address = public key (from `solen key list`)
ACCOUNT="197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61"
```

### Usage

```bash
# Build
cd examples/contracts/token
cargo build --target wasm32-unknown-unknown --release

# Deploy
solen deploy faucet target/wasm32-unknown-unknown/release/solen_example_token.wasm

# Initialize with name and symbol
NAME_ARGS=$(python3 -c "
name = b'My Token'
symbol = b'MTK'
print((bytes([len(name)]) + name + bytes([len(symbol)]) + symbol).hex())
")
solen call faucet <TOKEN_ID> init --args "$NAME_ARGS"

# Mint 1,000,000 tokens to faucet
TO="197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61"
AMOUNT=$(python3 -c "print((1000000).to_bytes(16, 'little').hex())")
solen call faucet <TOKEN_ID> mint --args "${TO}${AMOUNT}"

# Check balance
solen call faucet <TOKEN_ID> balance_of --args "${TO}"

# Transfer 50,000 to alice (address = her public key)
ALICE="139c31e8543b19629ea93c90b291d684aec0ca432cc0efda170570572c62e519"
AMOUNT=$(python3 -c "print((50000).to_bytes(16, 'little').hex())")
solen call faucet <TOKEN_ID> transfer --args "${ALICE}${AMOUNT}"
```

### ABI Discovery

Contracts implement an `abi()` method that returns a JSON description of available methods and events. The explorer uses this to auto-populate the "Read Contract" and "Write Contract" UI, and to decode events.

**ABI format:**

```json
{
  "methods": [
    {"name": "transfer", "args": "to[32]+amount[16]", "mutates": true},
    {"name": "balance_of", "args": "account[32]", "mutates": false}
  ],
  "events": [
    {"topic": "transfer", "data": "to[32]+amount[16]"},
    {"topic": "mint", "data": "to[32]+amount[16]"}
  ]
}
```

| Field | Description |
|-------|-------------|
| `methods[].name` | Method name (passed as first part of input before null byte) |
| `methods[].args` | Argument format: `name[size]` separated by `+` |
| `methods[].mutates` | `true` for write methods, `false` for read-only |
| `events[].topic` | Event topic string |
| `events[].data` | Data format: `name[size]` separated by `+` |

Query it via `solen_callView`:

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_callView","params":["<TOKEN_ID>","abi"],"id":1}'
```

### Source Code Publishing

Contract deployers can publish source code for verification on the explorer:

```bash
curl -X POST https://testnet-api.solenchain.com/api/contracts/<CODE_HASH>/source \
  -H "Content-Type: application/json" \
  -d '{"code_hash":"<CODE_HASH>","source_code":"<RUST_SOURCE>","language":"rust","compiler_version":"1.78.0"}'
```

Published source code is displayed on the contract's page in the explorer.

---

## NFT (SRC-721)

A non-fungible token contract with sequential token IDs, minting, and transfers.

**Source:** `examples/contracts/nft/src/lib.rs`

### Methods

| Method | Arguments | Description |
|--------|-----------|-------------|
| `abi` | — | Returns JSON array of all methods |
| `init` | `name_len[1]+name[]+symbol_len[1]+symbol[]` | Initialize collection |
| `mint` | `to (32 bytes)` | Mint next NFT to address (owner only) |
| `transfer` | `to (32 bytes) + token_id (u64 LE)` | Transfer NFT |
| `owner_of` | `token_id (u64 LE)` | Query owner of NFT |
| `balance_of` | `account (32 bytes)` | Query NFT count |
| `total_supply` | — | Total minted NFTs |
| `name` | — | Collection name |
| `symbol` | — | Collection symbol |
| `owner` | — | Contract owner |

### Usage

```bash
# Build
cd examples/contracts/nft
cargo build --target wasm32-unknown-unknown --release

# Deploy
solen deploy mykey target/wasm32-unknown-unknown/release/solen_example_nft.wasm

# Initialize
NAME_ARGS=$(python3 -c "
name = b'My NFT Collection'
symbol = b'MNFT'
print((bytes([len(name)]) + name + bytes([len(symbol)]) + symbol).hex())
")
solen call mykey <NFT_ID> init --args "$NAME_ARGS"

# Mint NFT #1 to an address
RECIPIENT="197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61"
solen call mykey <NFT_ID> mint --args "$RECIPIENT"

# Query owner of NFT #1
TOKEN_ID=$(python3 -c "print((1).to_bytes(8, 'little').hex())")
solen call mykey <NFT_ID> owner_of --args "$TOKEN_ID"
```

## Writing Your Own Contract

Start from the counter example:

1. Copy `examples/contracts/counter/` to a new directory
2. Update `Cargo.toml` with your contract name
3. Implement your methods in `src/lib.rs`
4. Build with `cargo build --target wasm32-unknown-unknown --release`
5. Deploy and test on a local devnet

See [Contract SDK](contract-sdk.md) for the full API reference.
