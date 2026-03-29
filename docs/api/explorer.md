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
| `limit` | `u32` | 20 | Number of blocks to return |

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
| `id` | `string` | Hex-encoded account ID |

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
