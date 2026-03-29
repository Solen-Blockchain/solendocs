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
  "indexed_height": 42,
  "total_blocks": 42,
  "total_transactions": 15,
  "total_events": 23
}
```

---

### `GET /api/blocks`

Get recent blocks.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 10 | Number of blocks to return |

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

**Response:** Same format as a single block from the blocks list, plus full transaction details.

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
| `limit` | `u32` | 10 | Number of transactions to return |

**Response:**

```json
[
  {
    "hash": "tx123...",
    "block_height": 42,
    "sender": "account-id",
    "actions": [...],
    "gas_used": 500,
    "success": true
  }
]
```

---

### `GET /api/events`

Get recent events (emitted by contracts).

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | `u32` | 10 | Number of events to return |

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
