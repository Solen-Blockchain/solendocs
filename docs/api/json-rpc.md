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
| `total_allocation` | `string` | Total tokens allocated at genesis (base units) |
| `total_staked` | `string` | Total tokens currently staked (base units) |
| `total_circulation` | `string` | Tokens in circulation, excluding system accounts and staked tokens (base units) |

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
| `account_id` | `string` | Base58 (or hex) encoded account ID |

**Returns:** Balance as a string (to avoid integer overflow in JSON).

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getBalance","params":["2ZrMqiKGz6TUvJkyBKyNMf3Y7dMrJ5JqWSCCYGn1VWbp"],"id":1}'
```

---

### `solen_getAccount`

Get full account information.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Base58 (or hex) encoded account ID |

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
  "sender": "account-id (Base58 or hex)",
  "nonce": 0,
  "actions": [
    {
      "type": "Transfer",
      "to": "account-id (Base58 or hex)",
      "amount": 1000
    }
  ],
  "max_fee": 200,
  "signature": "hex-encoded-ed25519-signature (or multi-sig)"
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

=== "SetAuth"
    ```json
    { "type": "SetAuth", "auth_methods": [{ "Threshold": { "signers": ["hex-pubkey-1", "hex-pubkey-2"], "threshold": 2 } }] }
    ```

**Signing message format:**

The signature is computed over a deterministic message that includes the chain ID to prevent cross-chain replay:

```
chain_id     [8 bytes, little-endian u64]
sender       [32 bytes]
nonce        [8 bytes, little-endian u64]
max_fee      [16 bytes, little-endian u128]
actions_hash [32 bytes, BLAKE3 hash of JSON-serialized actions]
```

Total: 96 bytes. Sign with Ed25519 to produce a 64-byte signature.

**Chain IDs:**

| Network | Chain ID |
|---------|----------|
| Mainnet | 1 |
| Testnet | 9000 |
| Devnet | 1337 |

**Multi-sig signatures:** For `Threshold` auth, concatenate `pubkey[32] + sig[64]` pairs (96 bytes each). At least `threshold` valid signatures from the signers list must be present.

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
| `address` | `string` | Validator address (Base58 or hex) |
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
| `account_id` | `string` | Base58 (or hex) encoded account ID |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `total_delegated` | `string` | Total tokens delegated by this account |
| `delegations` | `array` | List of `{validator, amount}` objects |
| `pending_undelegations` | `u64` | Number of pending undelegation requests |

---

### `solen_getGovernanceProposals`

Get all governance proposals and their current status.

**Parameters:** None

**Returns:** Array of proposal objects:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `u64` | Proposal ID |
| `proposer` | `string` | Proposer account (Base58 or hex) |
| `action` | `string` | Proposal action (e.g., `SetBlockTime { new_block_time_ms: 4000 }`) |
| `description` | `string` | Human-readable description |
| `status` | `string` | `Active`, `Passed`, `Rejected`, or `Executed` |
| `voting_end_epoch` | `u64` | Epoch when voting ends |
| `execute_after_epoch` | `u64` | Epoch when execution is allowed (after timelock) |
| `total_for` | `string` | Total stake voting for (base units) |
| `total_against` | `string` | Total stake voting against (base units) |
| `vote_count` | `u64` | Number of individual votes |

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getGovernanceProposals","params":[],"id":1}'
```

---

### `solen_callView`

Read-only contract call — no signature or transaction required. Executes a contract method in a sandboxed VM and returns the result without modifying state.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `contract_id` | `string` | Base58 (or hex) encoded contract account ID |
| `method` | `string` | Method name to call |
| `args` | `string?` | Hex-encoded arguments (optional) |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | `bool` | Whether the call succeeded |
| `return_data` | `string` | Hex-encoded return data |
| `gas_used` | `u64` | Gas consumed |
| `error` | `string?` | Error message (if failed) |

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_callView","params":["<contract-id>","total_supply"],"id":1}'
```

---

### `solen_getVestingInfo`

Get vesting schedule information for an account.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | `string` | Base58 (or hex) encoded account ID |

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
  -d '{"jsonrpc":"2.0","method":"solen_getVestingInfo","params":["2ZrMqiKGz6TUvJkyBKyNMf3Y7dMrJ5JqWSCCYGn1VWbp"],"id":1}'
```

---

### `solen_getPendingIntents`

Get pending intents available for solvers.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | `u64?` | Maximum number of intents to return (default: 50) |

**Returns:** Array of intent objects with `id`, `sender`, `constraints`, `max_fee`, `expiry_height`, `tip`, `status`.

---

### `solen_submitSolution`

Submit a solver's solution for an intent.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `intent_id` | `u64` | ID of the intent to solve |
| `solver` | `string` | Base58 (or hex) encoded solver account ID |
| `operations` | `array` | Array of `UserOperation` objects |
| `claimed_tip` | `string` | Tip amount claimed by solver |
| `score` | `u64` | Solver-reported quality score |

**Returns:** `{ accepted: bool, error: string? }`

---

### `solen_getRollupBatches`

Get verified batch history for a rollup.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rollup_id` | `u64` | Rollup domain identifier |
| `limit` | `u64?` | Maximum batches to return (default: 50) |

**Returns:** Array of `{ rollup_id, batch_index, state_root, data_hash, pre_state_root }`.

---

### `solen_submitIntent`

Submit a signed intent for solver resolution.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `sender` | `string` | Base58 (or hex) encoded sender account ID |
| `constraints` | `array` | Array of constraint objects (see below) |
| `max_fee` | `string` | Maximum fee the sender is willing to pay |
| `expiry_height` | `u64` | Block height at which the intent expires |
| `signature` | `string` | Hex-encoded signature |
| `tip` | `string` | Incentive for the solver (base units) |

**Constraint types:**

| Type | Fields |
|------|--------|
| `MinBalance` | `account`, `min_amount` |
| `MaxSpend` | `account`, `max_amount` |
| `RequireTransfer` | `from`, `to`, `min_amount` |
| `RequireCall` | `target`, `method` |
| `Custom` | `verifier`, `data` (hex) |

**Returns:** `{ accepted: bool, intent_id: u64?, error: string? }`

---

### `solen_checkSponsorship`

Check if a paymaster will sponsor an operation's fees.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `op` | `UserOperation` | The operation to check (byte array format) |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `sponsored` | `bool` | Whether a paymaster will sponsor |
| `paymaster` | `string?` | Paymaster contract address (Base58 or hex) |
| `max_gas` | `string?` | Maximum gas the paymaster will cover |
| `reason` | `string?` | Reason if not sponsored |

---

### `solen_getSnapshot`

Get a compressed state snapshot for fast sync. Cached for 500 blocks.

**Parameters:** None

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `height` | `u64` | Block height of the snapshot |
| `epoch` | `u64` | Epoch at snapshot time |
| `state_root` | `string` | Hex-encoded state root hash |
| `entries` | `u64` | Number of state entries |
| `compressed_bytes` | `usize` | Compressed data size in bytes |
| `uncompressed_bytes` | `usize` | Uncompressed data size in bytes |
| `data` | `string` | Base64-encoded snapshot binary |

**Example:**

```bash
curl -s -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getSnapshot","params":[],"id":1}'
```

---

### `solen_getRollupStatus`

Get rollup registration info and latest state commitment.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rollup_id` | `u64` | Rollup domain identifier |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| `rollup_id` | `u64` | Rollup ID |
| `registered` | `bool` | Whether the rollup is registered |
| `last_verified_state_root` | `string?` | Last verified state root (hex) |
| `last_batch_index` | `u64?` | Last verified batch index |

---

### `solen_submitBatch`

Submit a rollup batch commitment for proof verification on L1.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rollup_id` | `u64` | Rollup domain identifier |
| `batch_index` | `u64` | Sequential batch number |
| `state_root` | `string` | Post-execution state root (hex) |
| `data_hash` | `string` | Hash of batch data (hex) |
| `proof` | `string` | Proof bytes (hex) |

**Returns:** `{ accepted: bool, verified: bool, error: string? }`

---

## System Contract Calls

System contracts are invoked via `solen_submitOperation` with `Action::Call` targeting a well-known address. The executor routes these to native Rust implementations.

| Address | Contract | Available Methods |
|---------|----------|-------------------|
| `0xFFFF...FF01` | Staking | `register`, `delegate`, `undelegate`, `withdraw`, `set_commission`, `rotate_key` |
| `0xFFFF...FF02` | Governance | `propose_set_base_fee`, `propose_set_block_time`, `vote`, `finalize`, `execute` |
| `0xFFFF...FF03` | Bridge | `register_vault`, `deposit`, `register_rollup` |
| `0xFFFF...FF04` | Treasury | `status` |
| `0xFFFF...FF06` | Vesting | `claim`, `status` |
| `0xFFFF...FF07` | Paymaster Registry | `register`, `unregister`, `list` |
| `0xFFFF...FF08` | Guardian Recovery | `initiate_recovery`, `confirm_recovery`, `cancel_recovery`, `execute_recovery` |

> **Note:** System contract addresses are conventionally written in hex since they are compile-time constants. Both hex and Base58 formats are accepted in API calls.

### Staking Methods

**`register`** — Register as a new validator with self-stake.
Args: `amount[16 bytes LE u128]` (must be >= 500,000 SOLEN). The sender becomes the validator.

**`delegate`** — Delegate tokens to a validator.
Args: `validator_address[32 bytes] + amount[16 bytes LE u128]`

**`undelegate`** — Begin undelegation (7-epoch cooldown). Automatically withdraws any previously matured undelegations.
Args: `validator_address[32 bytes] + amount[16 bytes LE]`

**`withdraw`** — Withdraw all matured undelegations. Epoch is read from chain state automatically.
Args: none

**`set_commission`** — Set validator commission rate (validator only).
Args: `commission_bps[8 bytes LE]` (e.g., 1000 = 10%, max 10000 = 100%)

**`rotate_key`** — Request a validator key rotation. The new key takes effect at the next epoch boundary. All delegation records are updated automatically.
Args: `new_key[32 bytes]` (the new validator public key)

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

**`register_rollup`** — Register a rollup domain on L1. Requires a 10,000 SOLEN deposit.
Args: `rollup_id[8] + name_len[4] + name[...] + proof_type_len[4] + proof_type[...] + sequencer[32] + genesis_state_root[32]`

Creates a bridge vault, registers the rollup for proof verification, and emits a `rollup_registered` event.

### Paymaster Registry Methods

**`register`** — Register the calling contract as a fee sponsor (paymaster). The sender must be a deployed contract that implements a `willSponsor` view method. No args.

**`unregister`** — Remove the calling contract from the paymaster registry. No args.

**`list`** — Query the number of registered paymasters. No args.

**Paymaster contract interface:**

Paymaster contracts must implement a `willSponsor` method that:

- Receives the serialized `UserOperation` as input
- Returns `[1]` (1 byte) to accept, or `[1, max_gas[16 bytes LE]]` to accept with a gas limit
- Returns `[0]` or empty to reject

### Guardian Recovery Methods

Social recovery for lost keys. Users add `Guardian` auth methods to their account, designating trusted accounts that can collectively recover access.

**Setup:** Use `SetAuth` action to add guardians to your account:
```json
{
  "type": "SetAuth",
  "auth_methods": [
    { "Ed25519": { "public_key": "your-key-hex" } },
    { "Guardian": { "guardian_id": "guardian-1-hex" } },
    { "Guardian": { "guardian_id": "guardian-2-hex" } },
    { "Guardian": { "guardian_id": "guardian-3-hex" } }
  ]
}
```

**`initiate_recovery`** — A guardian initiates recovery for a target account.
Args: `target_account[32] + new_auth_methods_json[...]`
Starts a 1-week timelock (151,200 blocks at 4s block time). The initiating guardian auto-confirms.

**`confirm_recovery`** — Another guardian confirms a pending recovery.
Args: `recovery_id[8 bytes LE]`
Requires majority of guardians (minimum 2) to confirm.

**`cancel_recovery`** — The account owner cancels an unauthorized recovery.
Args: `recovery_id[8 bytes LE]`
Only the target account owner can cancel. Can be called any time before execution.

**`execute_recovery`** — Execute recovery after timelock expires and enough confirmations.
Args: `recovery_id[8 bytes LE]`
Anyone can call this. Replaces the target account's auth methods with the new ones.

**Recovery flow:**
1. Owner loses key
2. Guardian A calls `initiate_recovery` with new key → auto-confirms, starts 1-week timelock
3. Guardian B calls `confirm_recovery` → threshold reached (2 of 3)
4. After 1 week, anyone calls `execute_recovery` → account auth methods replaced
5. Owner regains access with new key

If the recovery is unauthorized, the real owner can call `cancel_recovery` during the 1-week window.

---

## CLI Commands

The `solen-cli` tool provides command-line access to all system operations:

```bash
# Chain info
solen-cli status
solen-cli balance <account>
solen-cli validators

# Transfers
solen-cli transfer <from> <to> <amount>

# Staking
solen-cli register-validator <from> <amount>
solen-cli stake <from> <validator> <amount>
solen-cli unstake <from> <validator> <amount>
solen-cli withdraw-stake <from>

# Governance
solen-cli propose-block-time <from> <ms> <description>
solen-cli vote <from> <proposal-id> --yes <weight>
solen-cli finalize-proposal <from> <proposal-id>
solen-cli execute-proposal <from> <proposal-id>

# Contracts
solen-cli deploy <from> <wasm-file>
solen-cli call <from> <target> <method> [--args <hex>]

# Rollups
solen-cli register-rollup <from> <rollup-id> <name> [--proof-type mock] [--genesis-state-root <hex>]

# Paymasters
solen-cli register-paymaster <from>
solen-cli unregister-paymaster <from>

# Guardian recovery
solen-cli initiate-recovery <from> <target> <new-public-key>
solen-cli confirm-recovery <from> <recovery-id>
solen-cli cancel-recovery <from> <recovery-id>
solen-cli execute-recovery <from> <recovery-id>

# Key management
solen-cli key generate <name>
solen-cli key import <name> <seed-hex>
solen-cli key list
```

All amount values are in SOLEN (human-readable). Decimals supported (e.g., `100.5`).
Use `--rpc <url>` to specify the RPC endpoint and `--chain-id <id>` for the network.

---

## Rate Limits

The RPC server enforces per-method rate limits to prevent abuse:

| Method | Limit |
|--------|-------|
| `solen_submitOperation` | 50 requests/second |
| `solen_submitSolution` | 20 requests/second |
| `solen_getSnapshot` | 2 requests/second |

Read-only query methods (`solen_getBalance`, `solen_getAccount`, etc.) are not rate-limited. Requests exceeding the limit receive a `-32603` (Internal error) response.

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
