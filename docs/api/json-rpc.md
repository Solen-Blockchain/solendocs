# JSON-RPC API

Solen exposes a JSON-RPC 2.0 API over HTTP for querying chain state and submitting operations.

## Connection

| Network | Default Port | URL |
|---------|-------------|-----|
| Mainnet | 9944 | `http://127.0.0.1:9944` |
| Testnet | 19944 | `http://127.0.0.1:19944` |
| Devnet | 29944 | `http://127.0.0.1:29944` |

All requests use HTTP POST with `Content-Type: application/json`.

## Methods

### `solen_chainStatus`

Get the current chain status.

**Parameters:** None

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `height` | `u64` | Current block height |
| `state_root` | `string` | Hex-encoded state root hash |
| `pending_ops` | `u64` | Number of operations in the mempool |

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

---

### `solen_getBalance`

Get an account's native token balance.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Hex-encoded 32-byte account ID |

**Returns:** Balance as a string (to avoid integer overflow in JSON).

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getBalance","params":["197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61"],"id":1}'
```

---

### `solen_getAccount`

Get full account information.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Hex-encoded 32-byte account ID |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `balance` | `string` | Native token balance |
| `nonce` | `u64` | Current operation nonce |
| `code_hash` | `string?` | Contract code hash (if deployed) |

---

### `solen_getBlock`

Get a block by height.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `height` | `u64` | Block height |

**Returns:** Block header and execution summary including transaction count, gas used, and state root.

---

### `solen_getLatestBlock`

Get the latest finalized block.

**Parameters:** None

**Returns:** Same format as `solen_getBlock`.

---

### `solen_submitOperation`

Submit a signed user operation to the mempool.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `operation` | `UserOperation` | The signed operation object |

**UserOperation format:**

```json
{
  "sender": "hex-encoded-account-id",
  "nonce": 0,
  "actions": [
    {
      "type": "Transfer",
      "to": "hex-encoded-account-id",
      "amount": 1000
    }
  ],
  "max_fee": 200,
  "signature": "hex-encoded-ed25519-signature"
}
```

**Action types:**

=== "Transfer"
    ```json
    { "type": "Transfer", "to": "account-id", "amount": 1000 }
    ```

=== "Call"
    ```json
    { "type": "Call", "target": "contract-id", "method": "mint", "args": "hex-data" }
    ```

=== "Deploy"
    ```json
    { "type": "Deploy", "code": "hex-encoded-wasm" }
    ```

---

### `solen_simulateOperation`

Dry-run an operation without committing state changes.

**Parameters:** Same as `solen_submitOperation`.

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | `bool` | Whether the operation would succeed |
| `gas_used` | `u64` | Estimated gas consumption |
| `return_data` | `string?` | Hex-encoded return data (for calls) |
| `error` | `string?` | Error message (if failed) |

Use this to estimate gas costs before submitting.

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
