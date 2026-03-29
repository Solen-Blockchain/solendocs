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
use solen_contract_sdk::*;

#[no_mangle]
pub extern "C" fn call(input_ptr: i32, input_len: i32) -> i32 {
    let input = read_input(input_ptr, input_len);
    let method = parse_method(&input);

    match method {
        "my_method" => {
            // Your logic here
            0
        }
        _ => 0,
    }
}
```

## Input Format

The `call` function receives a pointer and length to the input buffer. The input contains:

1. **Method name** (variable length, null-terminated)
2. **Arguments** (remaining bytes after the method name)

Use `parse_method()` and `parse_args()` to extract these.

## Storage

Contracts have their own isolated storage namespace:

```rust
// Read a value
let value = storage_read(b"my_key");

// Write a value
storage_write(b"my_key", b"my_value");

// Convenience helpers for common types
let count: u64 = storage_read_u64(b"count");
storage_write_u64(b"count", count + 1);
```

## Events

Emit events for off-chain indexing:

```rust
emit_event(b"transfer", &data);
```

Events are indexed by the `solen-indexer` and available through the Explorer API.

## Return Data

Set return data for the caller:

```rust
set_return_data(&result_bytes);
```

The `call` function's return value is the length of the return data.

## Context

Access execution context:

```rust
let caller: AccountId = get_caller();      // Who called this contract
let height: u64 = get_block_height();      // Current block height
```

## Deploying

```bash
# Deploy using the CLI
solen deploy <FROM_ACCOUNT> path/to/contract.wasm

# Call a method
solen call <FROM_ACCOUNT> <CONTRACT_ID> <METHOD> --args <HEX_ARGS>
```

## Best Practices

1. **Keep contracts small** — Use `opt-level = "z"` and LTO in release builds
2. **Validate inputs** — Check argument lengths and formats
3. **Use events** — Emit events for all state changes for indexability
4. **Test locally** — Use `--in-memory` node mode for fast iteration
