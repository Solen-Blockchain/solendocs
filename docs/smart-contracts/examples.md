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
| `init` | — | Initialize the token, setting caller as owner |
| `mint` | `to (32 bytes) + amount (u128 LE)` | Mint tokens to an account (owner only) |
| `transfer` | `to (32 bytes) + amount (u128 LE)` | Transfer tokens from caller to recipient |
| `approve` | `spender (32 bytes) + amount (u128 LE)` | Approve spender to use caller's tokens |
| `transfer_from` | `from (32 bytes) + to (32 bytes) + amount (u128 LE)` | Transfer using allowance |
| `balance_of` | `account (32 bytes)` | Query token balance |
| `allowance` | `owner (32 bytes) + spender (32 bytes)` | Query allowance |
| `total_supply` | — | Query total token supply |

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

# Initialize (sets caller as owner)
solen call faucet <TOKEN_ID> init

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

## Writing Your Own Contract

Start from the counter example:

1. Copy `examples/contracts/counter/` to a new directory
2. Update `Cargo.toml` with your contract name
3. Implement your methods in `src/lib.rs`
4. Build with `cargo build --target wasm32-unknown-unknown --release`
5. Deploy and test on a local devnet

See [Contract SDK](contract-sdk.md) for the full API reference.
