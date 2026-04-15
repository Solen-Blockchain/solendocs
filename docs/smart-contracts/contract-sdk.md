# Contract SDK

The `solen-contract-sdk` crate provides the tools and bindings needed to write WASM contracts in Rust for Solen.

## Setup

Add the SDK to your contract's `Cargo.toml`:

```toml
[package]
name = "my-contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
solen-contract-sdk = { path = "../../crates/solen-contract-sdk" }

[profile.release]
opt-level = "z"       # Optimize for size
lto = true
strip = true
```

Contracts use `#![no_std]` — no standard library, keeping the WASM binary small.

## Building

```bash
cargo build --target wasm32-unknown-unknown --release
```

The compiled WASM module will be at:
```
target/wasm32-unknown-unknown/release/my_contract.wasm
```

## Contract Structure

Every contract must export a `call` function:

```rust
#![no_std]
use solen_contract_sdk::{sdk, storage, events};

#[no_mangle]
pub extern "C" fn call(input_ptr: i32, input_len: i32) -> i32 {
    let input = sdk::read_input(input_ptr, input_len);

    // Parse method name (everything before the first \0 byte).
    let method_end = input.iter().position(|&b| b == 0).unwrap_or(input.len());
    let method = &input[..method_end];
    let args = if method_end + 1 < input.len() { &input[method_end + 1..] } else { &[] };

    match method {
        b"init" => {
            // Initialize contract state
            storage::set(b"initialized", &[1]);
            0
        }
        b"increment" => {
            let count = storage::get_u64(b"count").unwrap_or(0);
            let new_count = count + 1;
            storage::set_u64(b"count", new_count);
            events::emit(b"incremented", &new_count.to_le_bytes());
            sdk::return_value(&new_count.to_le_bytes())
        }
        b"get_count" => {
            let count = storage::get_u64(b"count").unwrap_or(0);
            sdk::return_value(&count.to_le_bytes())
        }
        _ => 0,
    }
}
```

## Input Format

The `call` function receives a pointer and length to the input buffer. The input format is:

```
[method_name] [0x00] [arguments...]
```

- **Method name**: UTF-8 bytes, terminated by a null byte (`0x00`)
- **Arguments**: Raw bytes after the null terminator. Format depends on the method.

**Parsing pattern:**

```rust
let input = sdk::read_input(input_ptr, input_len);
let method_end = input.iter().position(|&b| b == 0).unwrap_or(input.len());
let method = &input[..method_end];
let args = if method_end + 1 < input.len() { &input[method_end + 1..] } else { &[] };
```

**Common argument patterns:**

```rust
// Read a 32-byte account ID from args
fn read_account(args: &[u8], offset: usize) -> [u8; 32] {
    let mut id = [0u8; 32];
    id.copy_from_slice(&args[offset..offset + 32]);
    id
}

// Read a u128 amount (16 bytes, little-endian) from args
fn read_u128(args: &[u8], offset: usize) -> u128 {
    let mut buf = [0u8; 16];
    buf.copy_from_slice(&args[offset..offset + 16]);
    u128::from_le_bytes(buf)
}

// Read a u64 from args
fn read_u64(args: &[u8], offset: usize) -> u64 {
    let mut buf = [0u8; 8];
    buf.copy_from_slice(&args[offset..offset + 8]);
    u64::from_le_bytes(buf)
}
```

## SDK Modules

### `sdk` — Input, Output, and Context

| Function | Description |
|----------|-------------|
| `sdk::read_input(ptr, len)` | Read the raw input buffer |
| `sdk::return_value(data)` | Set return data and return its length |
| `sdk::caller()` | Get the 32-byte account ID of the caller |
| `sdk::block_height()` | Get the current block height |
| `sdk::self_id()` | Get this contract's own account ID |
| `sdk::transfer(to, amount)` | Transfer native SOLEN from the contract (max 50 per call) |

### `storage` — Persistent Key-Value Storage

Each contract has its own isolated storage namespace. Other contracts cannot read your storage.

| Function | Description |
|----------|-------------|
| `storage::get(key)` | Read raw bytes. Returns `Option<&[u8]>` (`None` if key doesn't exist) |
| `storage::set(key, value)` | Write raw bytes |
| `storage::get_u64(key)` | Read a u64 |
| `storage::set_u64(key, value)` | Write a u64 |
| `storage::get_u128(key)` | Read a u128 |
| `storage::set_u128(key, value)` | Write a u128 |

!!! warning "Scratch Buffer"
    `storage::get()` reads into a shared 4096-byte scratch buffer. If your values exceed 4KB, use the raw `storage_read` host function directly.

### `events` — Event Emission

| Function | Description |
|----------|-------------|
| `events::emit(topic, data)` | Emit an event with a topic string and data payload |

Events are indexed by the block explorer and available through the Explorer REST API. Use them for all state changes that off-chain applications need to track.

**Event data encoding convention:**

```rust
// Transfer event: recipient[32] + amount[16]
let mut data = Vec::new();
data.extend_from_slice(&recipient);
data.extend_from_slice(&amount.to_le_bytes());
events::emit(b"transfer", &data);
```

## Gas Costs

Every host function call consumes fuel (gas). If fuel is exhausted, the contract execution is halted and all state changes are rolled back.

| Operation | Base Cost | Per-Byte Cost |
|-----------|-----------|---------------|
| Storage write | 2,000 | 10 per byte (key + value) |
| Storage read | 500 | 10 per byte (key) |
| Event emission | 2,000 | 10 per byte (topic + data) |
| Native transfer | 5,000 | — |
| Return data | 500 | 10 per byte |

**Default fuel limit:** 1,000,000 per contract call (~100-200 storage operations).

## Access Control Pattern

```rust
fn require_owner(args_or_storage: &[u8]) -> bool {
    let caller = sdk::caller();
    let owner = storage::get(b"owner");
    match owner {
        Some(o) if o.len() >= 32 => caller == o[..32],
        _ => false,
    }
}

// In your method:
b"admin_method" => {
    if !require_owner(&[]) {
        return 0; // unauthorized
    }
    // ... admin logic
}
```

## Deploying

```bash
# Deploy using the CLI
solen deploy <FROM_ACCOUNT> path/to/contract.wasm

# Call a method
solen call <FROM_ACCOUNT> <CONTRACT_ID> <METHOD> --args <HEX_ARGS>

# Read-only view call (no signature needed)
solen call-view <CONTRACT_ID> <METHOD>
```

## Best Practices

1. **Keep contracts small** — Use `opt-level = "z"` and LTO in release builds
2. **Validate all inputs** — Check argument lengths before reading bytes
3. **Use events** — Emit events for all state changes for indexability
4. **Check callers** — Always verify authorization for sensitive operations
5. **Handle missing keys** — `storage::get()` returns `None` for unset keys
6. **Use base units** — 1 SOLEN = 100,000,000 base units (8 decimal places)
