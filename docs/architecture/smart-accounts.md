# Smart Accounts

In Solen, **every account is a smart account**. There are no externally owned accounts (EOAs). This is a core design decision that prioritizes user safety and flexibility.

## Account Structure

```rust
Account {
    account_id: AccountId,           // 32-byte Ed25519 public key (displayed as Base58)
    code_hash: Option<Hash>,         // WASM contract code (if deployed)
    nonce: u64,                      // Operation counter
    balance: u128,                   // Native token balance
    auth_methods: Vec<AuthMethod>,   // Authentication configuration
}
```

> **Address format:** Account IDs are Base58-encoded Ed25519 public keys (~44 characters). Hex format (64 characters) is also accepted as input.

## Authentication Methods

Smart accounts support multiple authentication mechanisms:

### Ed25519 Keys

The default authentication method. Each account can have one or more Ed25519 public keys authorized to sign operations.

### Passkeys (WebAuthn)

Browser-native authentication using device biometrics (fingerprint, face ID) or security keys. The TypeScript SDK provides `PasskeyAuth` for WebAuthn integration.

### Threshold Signatures (Multi-sig)

Require M-of-N signatures to authorize an operation. Useful for multi-sig wallets, team treasuries, and DAOs. This is implemented natively at the protocol level — no contract deployment needed.

#### Setting up a Multi-sig Account

**Step 1:** Generate keys for each signer:

```bash
solen key generate signer-alice
solen key generate signer-bob
solen key generate signer-carol
```

Note the public keys from `solen key list` — these are the signer addresses.

**Step 2:** Convert an existing account to multi-sig:

```bash
solen --chain-id 9000 multisig mykey \
  --threshold 2 \
  --signers "alice_pubkey,bob_pubkey,carol_pubkey"
```

This creates a **2-of-3** multi-sig: any 2 of the 3 signers must sign to authorize operations. The account must already exist and have balance for the gas fee.

**Step 3:** Sign operations with multiple keys. Each signer computes the signing message and produces their Ed25519 signature. The signatures are concatenated as `pubkey[32] + sig[64]` pairs (96 bytes each) in the operation's `signature` field. At least `threshold` valid pairs must be present.

#### Multi-sig via RPC

Submit a `SetAuth` action:

```json
{
  "type": "SetAuth",
  "auth_methods": [{
    "Threshold": {
      "signers": [
        "alice_pubkey_hex_64chars",
        "bob_pubkey_hex_64chars",
        "carol_pubkey_hex_64chars"
      ],
      "threshold": 2
    }
  }]
}
```

#### Reverting to Single-sig

To convert back to a standard Ed25519 account, submit a `SetAuth` action signed by the current multi-sig with a single `Ed25519` auth method:

```json
{
  "type": "SetAuth",
  "auth_methods": [{ "Ed25519": { "public_key": "your_pubkey_hex" } }]
}
```

#### Considerations

- The account must be funded before converting — gas fees still apply to the `SetAuth` operation
- A threshold of 0 is rejected. Threshold cannot exceed the number of signers
- **Duplicate signer prevention:** The signer list is validated for uniqueness — submitting the same public key multiple times in the `signers` array is rejected to prevent a single key from counting toward the threshold more than once
- Multi-sig accounts work with all operation types: transfers, contract calls, staking, and governance votes

### Guardians

Trusted accounts that can help recover access if primary keys are lost. Social recovery enables account recovery without seed phrases.

**Setup:** Add `Guardian` auth methods alongside your primary key using `SetAuth`:

```json
{
  "auth_methods": [
    { "Ed25519": { "public_key": "your-key" } },
    { "Guardian": { "guardian_id": "friend-1" } },
    { "Guardian": { "guardian_id": "friend-2" } },
    { "Guardian": { "guardian_id": "friend-3" } }
  ]
}
```

Guardian accounts must exist on-chain at the time of setup (validated during `SetAuth`).

**Recovery process:**

1. A guardian calls `initiate_recovery` on the Guardian system contract (`0xFFFF...FF08`) with the target account and new auth methods
2. The guardian list is **captured at initiation time** — subsequent changes to the account's guardians don't affect an in-progress recovery
3. Other guardians call `confirm_recovery` — a majority (minimum 2) must confirm
4. A **1-week timelock** (151,200 blocks at 4s block time) must pass before execution
5. During the timelock, the account owner can call `cancel_recovery` to block unauthorized attempts
6. After timelock + confirmations, anyone can call `execute_recovery` to replace the account's auth methods

**Security properties:**

- No single guardian can recover alone (majority required, minimum 2)
- Account owner can always cancel during the timelock window
- Guardian list is frozen at initiation — changing guardians mid-recovery doesn't affect the outcome
- Guardians never get access to funds — they can only change auth methods

### Passkeys (WebAuthn)

Hardware-backed authentication using platform biometrics (Face ID, Touch ID, Windows Hello). No seed phrases needed.

**How it works:**

- Uses P-256 (secp256r1) ECDSA — the standard WebAuthn algorithm
- The public key coordinates (x, y) are stored on-chain alongside the credential ID
- Each transaction's signing message is embedded as the WebAuthn challenge
- Signature includes authenticatorData + clientDataJSON + ECDSA (r, s)

**Setup:** Use `SetAuth` to add a passkey:

```json
{
  "auth_methods": [
    {
      "Passkey": {
        "credential_id": [/* bytes */],
        "public_key_x": [/* 32 bytes */],
        "public_key_y": [/* 32 bytes */]
      }
    }
  ]
}
```

**Verification checks:**

- Challenge in clientDataJSON matches base64url(signing_message)
- User Present (UP) flag is set in authenticatorData
- P-256 ECDSA signature is valid over authenticatorData || SHA-256(clientDataJSON)

### Session Credentials

Temporary keys with:

- **Time limits** — Auto-expire after a set block height
- **Spending limits** — Cap the total value of transfers per operation
- **Target restrictions** — Limit which contracts can be called
- **Method restrictions** — Limit which contract methods can be called

Useful for dApp sessions where users grant limited permissions without exposing their primary keys.

**Setup:** Use `SetAuth` to add a session key alongside your primary key:

```json
{
  "auth_methods": [
    { "Ed25519": { "public_key": "primary-key" } },
    {
      "Session": {
        "session_key": "temp-ed25519-key",
        "expires_at": 100000,
        "spending_limit": 1000000000000,
        "allowed_targets": ["contract-id"],
        "allowed_methods": ["transfer", "swap"]
      }
    }
  ]
}
```

Session keys use standard Ed25519 signatures. The executor validates restrictions at execution time — expired sessions are rejected, spending over the limit fails, and calls to unauthorized targets/methods are blocked.

> **Security restriction:** Session keys are blocked from performing privileged operations: `SetAuth` (changing account authentication), `Deploy` (deploying contracts), guardian recovery actions (`initiate_recovery`, `confirm_recovery`, `cancel_recovery`, `execute_recovery`), and governance proposal actions (`propose`, `finalize`, `execute`). These operations always require the primary account key.

## Programmable Policies

Accounts can define custom authorization logic:

- Spending limits per time period
- Whitelisted destination accounts
- Required delays for large transfers
- Custom validation logic via WASM

## Why No EOAs?

Traditional blockchains distinguish between EOAs (controlled by private keys) and contract accounts. This creates problems:

1. **Lost keys = lost funds** — No recovery mechanism for EOAs
2. **No programmability** — EOAs can't enforce spending policies
3. **Poor UX** — Users must manage raw private keys
4. **No batching** — EOAs execute one action per transaction

Solen's smart-account-only model solves all of these by making every account programmable from the start.
