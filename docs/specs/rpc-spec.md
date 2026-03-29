# Spec 003: RPC API Specification

**Status:** Draft

## Overview

The Solen JSON-RPC API is organized into four groups: state queries, write operations, simulation, and rollup operations.

## State Queries

### `solen_getBalance(account_id)`

Returns the native token balance for an account.

- **Parameters:** `account_id` — hex-encoded 32-byte account ID
- **Returns:** Balance as string

### `solen_getAccount(account_id)`

Returns full account state including balance, nonce, code hash, and authentication configuration.

- **Parameters:** `account_id` — hex-encoded 32-byte account ID
- **Returns:** Account object

### `solen_getBlock(height | "latest")`

Returns a block by height or the latest finalized block.

- **Parameters:** `height` — block number (u64) or string `"latest"`
- **Returns:** Block header and execution summary

### `solen_getValidatorSet(epoch?)`

Returns the validator set for a given epoch, or the current epoch if omitted.

- **Parameters:** `epoch` — optional epoch number
- **Returns:** List of validators with stakes and status

## Write Operations

### `solen_submitOperation(signed_op)`

Submits a signed user operation to the mempool.

- **Parameters:** `signed_op` — complete signed UserOperation
- **Returns:** Operation hash

### `solen_submitIntent(signed_intent)`

Submits a signed intent for solver resolution.

- **Parameters:** `signed_intent` — complete signed Intent
- **Returns:** Intent hash

## Simulation

### `solen_simulateOperation(op)`

Dry-runs a user operation without state changes.

- **Parameters:** `op` — UserOperation (signature optional)
- **Returns:** Gas estimate, success/failure, action summary, return data

### `solen_checkSponsorship(op)`

Checks if a paymaster will sponsor the given operation's fees.

- **Parameters:** `op` — UserOperation
- **Returns:** Sponsorship status and paymaster details

## Rollup Operations

### `solen_getRollupStatus(rollup_id)`

Returns rollup registration info and latest state commitment.

- **Parameters:** `rollup_id` — hex-encoded rollup identifier
- **Returns:** Rollup object with latest commitment

### `solen_submitBatch(batch)`

Submits a rollup batch commitment to L1.

- **Parameters:** `batch` — batch commitment with state root and proof
- **Returns:** Batch hash

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
