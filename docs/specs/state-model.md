# Spec 002: State Model

**Status:** Draft

## Canonical State Objects

The Solen L1 maintains these core state objects:

### Account

| Field | Type | Description |
|-------|------|-------------|
| `id` | `[u8; 32]` | Unique account identifier (Ed25519 public key). Displayed as Base58 (~44 characters); hex (64 characters) is also accepted. |
| `code_hash` | `[u8; 32]` | WASM contract code hash (zero if no contract deployed) |
| `auth_methods` | `Vec<AuthMethod>` | Authorized signing/recovery methods (see below) |
| `nonce` | `u64` | Operation counter (replay protection) |
| `balance` | `u128` | Native token balance (base units, 8 decimals) |

**AuthMethod variants:**

| Variant | Fields | Description |
|---------|--------|-------------|
| `Ed25519` | `public_key: [u8; 32]` | Standard Ed25519 signing key |
| `Threshold` | `signers: Vec<[u8; 32]>, threshold: u16` | M-of-N multi-sig |
| `Guardian` | `guardian_id: [u8; 32]` | Social recovery guardian (must be existing account) |
| `Passkey` | `credential_id: Vec<u8>` | WebAuthn passkey (planned) |

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
