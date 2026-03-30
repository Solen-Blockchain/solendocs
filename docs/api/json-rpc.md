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

---

### `solen_getValidators`

List all validators and their stake.

**Parameters:** None

**Returns:** Array of validator objects:

| Field | Type | Description |
|-------|------|-------------|
| `address` | `string` | Validator address (public key hex) |
| `self_stake` | `string` | Validator's own stake |
| `total_delegated` | `string` | Total delegated to this validator |
| `total_stake` | `string` | Self stake + delegated |
| `is_active` | `bool` | Whether the validator is active |
| `is_genesis` | `bool` | Whether this is a genesis validator |
| `commission_bps` | `u64` | Commission rate in basis points (1000 = 10%) |

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getValidators","params":[],"id":1}'
```

---

### `solen_getStakingInfo`

Get staking information for an account (delegations, undelegations).

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Hex-encoded account address |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `total_delegated` | `string` | Total tokens delegated by this account |
| `delegations` | `array` | List of `{validator, amount}` objects |
| `pending_undelegations` | `u64` | Number of pending undelegation requests |

---

### `solen_getVestingInfo`

Get vesting schedule information for an account.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Hex-encoded account address |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `has_schedule` | `bool` | Whether this account has a vesting schedule |
| `total_amount` | `string` | Total tokens allocated to this account |
| `vested` | `string` | Tokens vested so far (based on elapsed epochs) |
| `claimed` | `string` | Tokens already claimed |
| `claimable` | `string` | Tokens available to claim now (`vested - claimed`) |
| `vesting_type` | `string` | `"team"` or `"investor"` |

If the account has no vesting schedule, `has_schedule` is `false` and all amounts are `"0"`.

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getVestingInfo","params":["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"],"id":1}'
```

---

## System Contract Calls

System contracts are invoked via `solen_submitOperation` with `Action::Call` targeting a well-known address. The executor routes these to native Rust implementations.

| Address | Contract | Available Methods |
|---------|----------|-------------------|
| `0xFFFF...FF01` | Staking | `delegate`, `undelegate`, `withdraw`, `set_commission` |
| `0xFFFF...FF02` | Governance | `propose_set_base_fee`, `vote` |
| `0xFFFF...FF03` | Bridge | `register_vault`, `deposit` |
| `0xFFFF...FF04` | Treasury | `status` |
| `0xFFFF...FF06` | Vesting | `claim`, `status` |

### Staking Methods

**`delegate`** — Delegate tokens to a validator.
Args: `validator_address[32 bytes] + amount[16 bytes LE u128]`

**`undelegate`** — Begin undelegation (7-epoch cooldown).
Args: `validator_address[32 bytes] + amount[16 bytes LE] + epoch[8 bytes LE]`

**`withdraw`** — Withdraw completed undelegations.
Args: `current_epoch[8 bytes LE]`

**`set_commission`** — Set validator commission rate (validator only).
Args: `commission_bps[8 bytes LE]` (e.g., 1000 = 10%, max 10000 = 100%)

### Reward Distribution

Rewards are distributed every epoch (~100 blocks / ~3.3 minutes):

| Recipient | Amount | Event |
|-----------|--------|-------|
| Validator | Self-stake share + commission on delegator rewards | `epoch_reward` |
| Delegators | Proportional share minus commission | `delegator_reward` |

Default commission: **10%** (1000 bps). Validators can change this via `set_commission`.

**Event data format** (both `epoch_reward` and `delegator_reward`):
`recipient_address[32 bytes] + amount[16 bytes LE u128]`

### Vesting Methods

**`claim`** — Claim vested tokens that have passed the cliff and vesting schedule.
Args: none (sender is identified from the operation).

The claimed tokens are credited directly to the sender's account balance.

**`status`** — Query vesting status (read-only, use `solen_getVestingInfo` RPC instead).
Args: none.

**Vesting Schedules:**

| Type | Cliff | Total Duration |
|------|-------|----------------|
| Team & Founders | 1 year (approx. 365 epochs) | 4 years (approx. 1460 epochs) |
| Early Investors | 6 months (approx. 182 epochs) | 2.5 years (approx. 912 epochs) |

Tokens vest linearly after the cliff period. Unclaimed tokens accumulate and can be claimed at any time after vesting.

---

### Governance Methods

**`propose_set_base_fee`** — Create a proposal to change the base fee.
Args: `new_fee[16 bytes LE] + description[UTF-8 string]`

**`vote`** — Vote on a proposal.
Args: `proposal_id[8 bytes LE] + support[1 byte: 0=no, 1=yes] + stake_weight[16 bytes LE]`

### Bridge Methods

**`register_vault`** — Register a bridge vault for a rollup.
Args: `rollup_id[8 bytes LE]`

**`deposit`** — Deposit tokens into a rollup bridge vault.
Args: `rollup_id[8 bytes LE] + amount[16 bytes LE]`

---

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
