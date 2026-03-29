# Smart Contracts

Solen contracts are **WASM modules** compiled from any language that targets WebAssembly. Contracts execute in a sandboxed environment with deterministic behavior and fuel-based gas metering.

## How It Works

1. **Write** a contract in Rust (or any WASM-targeting language)
2. **Compile** to `wasm32-unknown-unknown`
3. **Deploy** via a `Deploy` action in a user operation
4. **Call** via `Call` actions, passing method name and arguments

## Contract Interface

Every contract must export:

- **`memory`** — WebAssembly linear memory
- **`call(input_ptr: i32, input_len: i32) -> i32`** — Entry point, returns output length

The input contains the method name and arguments. The contract processes the call and optionally sets return data via the `set_return_data` host function.

## Gas Metering

Gas is metered using wasmtime's **fuel mechanism**:

- Each WASM instruction consumes fuel
- Operations have base gas costs (transfer: 100, call: 500, deploy: 1,000)
- Contract execution adds VM fuel consumption on top of the base cost
- Operations fail if they exceed `max_fee`

## Determinism

All contract execution is fully deterministic:

- No floating point operations
- No access to system time or randomness (use block height for ordering)
- No network access
- Memory is isolated per contract call

## Quick Example

```rust
#![no_std]
use solen_contract_sdk::*;

#[no_mangle]
pub extern "C" fn call(input_ptr: i32, input_len: i32) -> i32 {
    let input = read_input(input_ptr, input_len);
    let method = parse_method(&input);

    match method {
        "increment" => {
            let count: u64 = storage_read_u64(b"count");
            storage_write_u64(b"count", count + 1);
            emit_event(b"incremented", &(count + 1).to_le_bytes());
            set_return_data(&(count + 1).to_le_bytes());
            8 // return data length
        }
        "get" => {
            let count: u64 = storage_read_u64(b"count");
            set_return_data(&count.to_le_bytes());
            8
        }
        _ => 0,
    }
}
```

## Next Steps

- [Contract SDK](contract-sdk.md) — SDK reference and setup
- [Host Functions](host-functions.md) — Available host function APIs
- [Example Contracts](examples.md) — Counter and Token examples
