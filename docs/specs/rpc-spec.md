# Spec 003: RPC API Specification

**Status:** Draft

## Overview

The Solen JSON-RPC API is organized into five groups: state queries, write operations, simulation, rollup operations, and WebSocket subscriptions. The server accepts both HTTP POST and WebSocket connections on the same port.

## State Queries

### `solen_getBalance(account_id)`

Returns the native token balance for an account.

- **Parameters:** `account_id` — Account ID (Base58) or public key (hex)
- **Returns:** Balance as string

### `solen_getAccount(account_id)`

Returns full account state including balance, nonce, code hash, and authentication configuration.

- **Parameters:** `account_id` — Account ID (Base58) or public key (hex)
- **Returns:** Account object

### `solen_getNextNonce(account_id)`

Returns the next usable nonce for an account, accounting for pending mempool transactions.

- **Parameters:** `account_id` — Account ID (Base58) or public key (hex)
- **Returns:** Next nonce (u64) = on-chain nonce + pending count

### `solen_getBlock(height | "latest")`

Returns a block by height or the latest finalized block.

- **Parameters:** `height` — block number (u64) or string `"latest"`
- **Returns:** Block header and execution summary

### `solen_getValidators()`

Returns the current validator set with stake and status information.

- **Parameters:** none
- **Returns:** List of validators with self-stake, delegated stake, active status, genesis flag, and commission rate

## Write Operations

### `solen_submitOperation(signed_op)`

Submits a signed user operation to the mempool. Returns immediately after mempool acceptance — does not wait for block inclusion.

- **Parameters:** `signed_op` — complete signed UserOperation
- **Returns:** `{ accepted, error }`

### `solen_submitOperationConfirm(signed_op, timeout_secs?)`

Submits a signed user operation and blocks until it is included in a finalized block (or until `timeout_secs` elapses, default 60s, max 180s). Designed for exchange integrations that want a single HTTP round-trip with full confirmation data.

- **Parameters:** `signed_op` — complete signed UserOperation; `timeout_secs` — optional u64 wait cap
- **Returns:** `{ accepted, confirmed, success, block_height, tx_hash, sender, nonce, gas_used, error }`
- **Semantics:** A reverted on-chain tx returns `confirmed: true, success: false` — exchanges must not credit funds on revert.
- **Concurrency:** Server caps simultaneous confirm-waiters at 200; excess calls reject with `accepted: false`.
- **Status:** Implemented

### `solen_submitIntent(signed_intent)`

Submits a signed intent for solver resolution. Intents are stored in an in-memory pool and matched with solver solutions.

- **Parameters:** `signed_intent` — Intent with sender, constraints, max_fee, expiry_height, signature, and tip
- **Returns:** `{ accepted, intent_id, error }`
- **Status:** Implemented

## Simulation

### `solen_simulateOperation(op)`

Dry-runs a user operation without state changes.

- **Parameters:** `op` — UserOperation (signature optional)
- **Returns:** Gas estimate, success/failure, action summary, return data

### `solen_checkSponsorship(op)`

Checks if a registered paymaster contract will sponsor the given operation's fees. Queries each paymaster's `willSponsor` view method.

- **Parameters:** `op` — UserOperation (byte array format)
- **Returns:** `{ sponsored, paymaster, max_gas, reason }`
- **Status:** Implemented

## Rollup Operations

### `solen_getRollupStatus(rollup_id)`

Returns rollup registration info and latest verified state root.

- **Parameters:** `rollup_id` — numeric rollup identifier
- **Returns:** `{ rollup_id, registered, last_verified_state_root, last_batch_index }`
- **Status:** Implemented

### `solen_submitBatch(batch)`

Submits a rollup batch commitment for proof verification on L1. Auto-registers the rollup in the in-memory proof registry if it was registered on-chain.

- **Parameters:** `batch` — `{ rollup_id, batch_index, state_root, data_hash, proof }` (hex-encoded hashes)
- **Returns:** `{ accepted, verified, error }`
- **Status:** Implemented

## WebSocket Subscriptions

Connect via `ws://host:port` (same port as HTTP RPC). All RPC methods above are also callable over WebSocket. Subscriptions use the JSON-RPC 2.0 subscription protocol.

### `solen_subscribeNewBlocks`

Streams every finalized block.

- **Parameters:** none
- **Notification (`solen_newBlock`):** `{ height, epoch, block_hash, state_root, proposer, timestamp_ms, tx_count, gas_used }`
- **Unsubscribe:** `solen_unsubscribeNewBlocks`

### `solen_subscribeTxConfirmation(sender, nonce)`

Watches for a specific transaction confirmation. Auto-closes after delivery.

- **Parameters:** `sender` — Account ID (Base58 or hex), `nonce` — transaction nonce (u64)
- **Notification (`solen_txConfirmation`):** `{ block_height, tx_hash, sender, nonce, success, gas_used }`
- **Unsubscribe:** `solen_unsubscribeTxConfirmation`

### `solen_subscribeValidatorChanges`

Emitted at epoch boundaries when the validator set changes.

- **Parameters:** none
- **Notification (`solen_validatorChange`):** `{ epoch, active_count }`
- **Unsubscribe:** `solen_unsubscribeValidatorChanges`

## Error Codes

| Code | Description |
|------|-------------|
| `-32600` | Invalid request |
| `-32601` | Method not found |
| `-32602` | Invalid params |
| `-32603` | Internal error |
| `-32000` | Account not found |
| `-32001` | Insufficient balance |
| `-32002` | Invalid nonce |
| `-32003` | Signature verification failed |
| `-32004` | Rollup not found |
| `-32005` | Intent expired |
