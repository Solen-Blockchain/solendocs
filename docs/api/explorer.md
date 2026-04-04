# Explorer REST API

The Solen indexer provides a REST API for querying blocks, transactions, and events. This powers the [SolenScan](../tools/block-explorer.md) block explorer.

## Connection

| Network | Default Port | Base URL |
|---------|-------------|----------|
| Mainnet | 9955 | `http://127.0.0.1:9955` |
| Testnet | 19955 | `http://127.0.0.1:19955` |
| Devnet | 29955 | `http://127.0.0.1:29955` |

Built with [Axum](https://github.com/tokio-rs/axum). All responses are JSON.

## Endpoints

### `GET /api/status`

Get the indexer status.

**Response:**

```json
{
  "latest_height": 42,
  "total_blocks": 42,
  "total_txs": 15,
  "total_events": 23
}
```

---

### `GET /api/blocks`

Get recent blocks.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 20 | Number of items to return |
| `offset` | `u32` | 0 | Number of items to skip (for pagination) |

**Response:**

```json
[
  {
    "height": 42,
    "hash": "abc123...",
    "parent_hash": "def456...",
    "state_root": "789abc...",
    "timestamp": 1700000000,
    "tx_count": 3,
    "gas_used": 1500
  }
]
```

---

### `GET /api/blocks/{height}`

Get a specific block by height.

**Response:** Same format as a single block from the blocks list.

---

### `GET /api/blocks/{height}/txs`

Get all transactions in a specific block.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `height` | `u64` | Block height |

**Response:**

```json
[
  {
    "block_height": 42,
    "index": 0,
    "sender": "account-id",
    "actions": [...],
    "gas_used": 500,
    "success": true
  }
]
```

---

### `GET /api/tx/{height}/{index}`

Get a specific transaction by block height and index within the block.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `height` | `u64` | Block height |
| `index` | `usize` | Transaction index within the block |

**Response:**

```json
{
  "block_height": 42,
  "index": 0,
  "sender": "account-id",
  "actions": [...],
  "gas_used": 500,
  "success": true
}
```

---

### `GET /api/txs`

Get recent transactions across all blocks.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 20 | Number of transactions to return |

**Response:**

```json
[
  {
    "block_height": 42,
    "index": 0,
    "sender": "account-id",
    "actions": [...],
    "gas_used": 500,
    "success": true
  }
]
```

---

### `GET /api/accounts/{id}/txs`

Get transaction history for an account.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Base58 (or hex) encoded account ID |

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 20 | Number of transactions to return |

**Response:**

```json
[
  {
    "block_height": 42,
    "index": 0,
    "sender": "account-id",
    "actions": [...],
    "gas_used": 500,
    "success": true
  }
]
```

---

### `GET /api/events`

Get recent events emitted by contracts.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 20 | Number of events to return |

**Response:**

```json
[
  {
    "block_height": 42,
    "tx_hash": "tx123...",
    "contract": "contract-id",
    "topic": "transfer",
    "data": "hex-encoded-data"
  }
]
```

---

### `GET /api/validators`

Get the current validator set.

**Response:**

```json
{
  "validators": [
    {
      "id": "validator-account-id",
      "stake": "10000",
      "status": "active"
    }
  ],
  "total_active_stake": "40000",
  "active_count": 4,
  "total_count": 4
}
```

---

### `GET /api/accounts/:account/tokens`

Get token contracts associated with an account (contracts that have sent tokens to or received tokens from this account).

**Response:** Array of contract ID strings.

```json
["7efd4515fcea2a83b0b0c12a154b82ca7fc432d1a125406f5973fa7a72a1ccdf"]
```

---

### `GET /api/contracts`

List all deployed contracts.

**Response:** Array of contract ID strings.

```json
["7efd4515fcea2a83...", "12f2f058c9809155..."]
```

---

### `GET /api/validators/stats`

Get block production statistics for each validator.

**Response:**

```json
[
  {
    "validator": "8a88e3dd7409...",
    "blocks_proposed": 1234,
    "last_proposed_height": 5678,
    "uptime_pct": 20.5
  }
]
```

`uptime_pct` is the percentage of total blocks proposed by this validator.

---

### `POST /api/contracts/:code_hash/source`

Publish source code for a contract.

**Request body:**

```json
{
  "code_hash": "869c407df2e5...",
  "source_code": "//! Contract source...",
  "language": "rust",
  "compiler_version": "1.78.0"
}
```

### `GET /api/contracts/:code_hash/source`

Retrieve published source code for a contract.

**Response:**

```json
{
  "code_hash": "869c407df2e5...",
  "source_code": "//! Contract source...",
  "language": "rust",
  "compiler_version": "1.78.0",
  "published_at": 1711900000
}
```

Returns `null` if no source has been published.

---

### `GET /api/rollups`

List all registered rollup domains.

**Response:** Array of rollup objects:

```json
[
  {
    "rollup_id": 1,
    "name": "Solen Test Rollup",
    "proof_type": "mock",
    "sequencer": "43a72e71...",
    "genesis_state_root": "",
    "registered_at_height": 34200
  }
]
```

---

### `GET /api/rollups/:rollup_id`

Get rollup detail including batch count and latest batch.

**Response:**

```json
{
  "rollup_id": 1,
  "name": "Solen Test Rollup",
  "proof_type": "mock",
  "sequencer": "43a72e71...",
  "registered_at_height": 34200,
  "total_batches": 3,
  "latest_batch": { "batch_index": 3, "state_root": "f6c344...", "verified": true }
}
```

---

### `GET /api/rollups/:rollup_id/batches`

Get verified batch history for a rollup.

**Response:** Array of batch objects with `rollup_id`, `batch_index`, `state_root`, `data_hash`, `verified`, `block_height`.

---

### `GET /api/intents`

Get recently fulfilled intents.

**Query parameters:** `limit` (default 20)

**Response:** Array of intent objects:

```json
[
  {
    "intent_id": 0,
    "sender": "43a72e71...",
    "block_height": 36316,
    "tx_index": 0,
    "transfer_to": "094c8cce...",
    "transfer_amount": "10000000000",
    "solver_tip": "100000000",
    "solver": "8a88e3dd..."
  }
]
```

---

## Pagination

All list endpoints (`/api/blocks`, `/api/txs`, `/api/events`, `/api/accounts/:id/txs`) support pagination:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 20 | Items per page |
| `offset` | `u32` | 0 | Items to skip |

**Example:** `GET /api/txs?limit=25&offset=50` returns transactions 51-75 (newest first).
