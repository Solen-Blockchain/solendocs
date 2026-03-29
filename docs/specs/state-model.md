# Spec 002: State Model

**Status:** Draft

## Canonical State Objects

The Solen L1 maintains these core state objects:

### Account

| Field | Type | Description |
|-------|------|-------------|
| `account_id` | `[u8; 32]` | Unique account identifier |
| `code_hash` | `Option<Hash>` | WASM contract code hash (if deployed) |
| `owners` | `Vec<PublicKey>` | Authorized signing keys |
| `threshold` | `u8` | Required signatures for M-of-N |
| `guardians` | `Vec<AccountId>` | Recovery accounts |
| `nonce` | `u64` | Operation counter (replay protection) |
| `spending_policies` | `Vec<Policy>` | Spending limits and restrictions |
| `session_keys` | `Vec<SessionKey>` | Temporary authorized keys |

### Validator

| Field | Type | Description |
|-------|------|-------------|
| `validator_id` | `AccountId` | Validator's account |
| `pubkeys` | `Vec<PublicKey>` | Consensus signing keys |
| `stake` | `u128` | Self-staked amount |
| `status` | `ValidatorStatus` | Active, jailed, unbonding |
| `slashing_history` | `Vec<SlashEvent>` | Record of past slashing |
| `reward_destination` | `AccountId` | Where to send rewards |

### Rollup

| Field | Type | Description |
|-------|------|-------------|
| `rollup_id` | `Hash` | Unique rollup identifier |
| `vm_type` | `VmType` | WASM, EVM, or custom |
| `proof_type` | `ProofType` | Validity, fraud, or none |
| `da_mode` | `DaMode` | On-chain or external DA |
| `bridge_config` | `BridgeConfig` | Bridge parameters |
| `sequencer_set` | `Vec<AccountId>` | Authorized sequencers |
| `governance_params` | `GovernanceParams` | Rollup-specific governance |

### Bridge Vault

| Field | Type | Description |
|-------|------|-------------|
| `asset_id` | `Hash` | Asset being bridged |
| `origin_domain` | `DomainId` | Source chain/rollup |
| `supply` | `u128` | Total locked supply |
| `custody_state` | `CustodyState` | Current custody status |
| `pending_exits` | `Vec<Exit>` | Withdrawals in progress |
| `challenge_windows` | `Vec<Challenge>` | Active challenge periods |

### Message Receipt

| Field | Type | Description |
|-------|------|-------------|
| `source` | `DomainId` | Sending domain |
| `destination` | `DomainId` | Receiving domain |
| `nonce` | `u64` | Message sequence number |
| `payload_hash` | `Hash` | Hash of message content |
| `timeout` | `u64` | Block height deadline |
| `proof_reference` | `Hash` | Proof for message validity |
| `execution_status` | `Status` | Pending, executed, expired |
