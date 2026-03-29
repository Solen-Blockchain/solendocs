# Host Functions

Host functions are the interface between WASM contracts and the Solen runtime. They provide access to storage, events, execution context, and return data.

## Available Functions

### `storage_read`

Read a value from contract storage.

```
storage_read(key_ptr: i32, key_len: i32, val_ptr: i32) -> i32
```

| Parameter | Description |
|-----------|-------------|
| `key_ptr` | Pointer to the storage key in WASM memory |
| `key_len` | Length of the key in bytes |
| `val_ptr` | Pointer to buffer where the value will be written |
| **Returns** | Length of the value, or -1 if key not found |

### `storage_write`

Write a value to contract storage.

```
storage_write(key_ptr: i32, key_len: i32, val_ptr: i32, val_len: i32)
```

| Parameter | Description |
|-----------|-------------|
| `key_ptr` | Pointer to the storage key |
| `key_len` | Length of the key in bytes |
| `val_ptr` | Pointer to the value to store |
| `val_len` | Length of the value in bytes |

### `emit_event`

Emit an event for off-chain indexing.

```
emit_event(topic_ptr: i32, topic_len: i32, data_ptr: i32, data_len: i32)
```

| Parameter | Description |
|-----------|-------------|
| `topic_ptr` | Pointer to the event topic string |
| `topic_len` | Length of the topic |
| `data_ptr` | Pointer to the event data |
| `data_len` | Length of the event data |

Events are recorded in the block and indexed by `solen-indexer`.

### `get_caller`

Get the account ID of the operation sender.

```
get_caller(out_ptr: i32)
```

| Parameter | Description |
|-----------|-------------|
| `out_ptr` | Pointer to a 32-byte buffer where the caller's AccountId is written |

### `get_block_height`

Get the current block height.

```
get_block_height() -> i64
```

Returns the height of the block being executed.

### `set_return_data`

Set the return data for the contract call.

```
set_return_data(ptr: i32, len: i32)
```

| Parameter | Description |
|-----------|-------------|
| `ptr` | Pointer to the return data |
| `len` | Length of the return data in bytes |

The `call` function's return value should equal `len`.

## SDK Wrappers

The `solen-contract-sdk` provides Rust wrappers around these raw host functions:

```rust
// High-level storage API
let value: Vec<u8> = storage_read(b"key");
storage_write(b"key", b"value");

// Typed helpers
let count: u64 = storage_read_u64(b"count");
storage_write_u64(b"count", 42);

// Events
emit_event(b"topic", &data_bytes);

// Context
let caller: [u8; 32] = get_caller();
let height: u64 = get_block_height();

// Return data
set_return_data(&result_bytes);
```

## Gas Costs

Host function calls consume gas in addition to regular WASM instruction fuel:

| Function | Approximate Gas |
|----------|----------------|
| `storage_read` | 50 + value size |
| `storage_write` | 100 + value size |
| `emit_event` | 50 + data size |
| `get_caller` | 10 |
| `get_block_height` | 5 |
| `set_return_data` | 10 + data size |
