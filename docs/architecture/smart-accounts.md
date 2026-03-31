# Smart Accounts

In Solen, **every account is a smart account**. There are no externally owned accounts (EOAs). This is a core design decision that prioritizes user safety and flexibility.

## Account Structure

```rust
Account {
    account_id: AccountId,           // 32-byte unique identifier
    code_hash: Option<Hash>,         // WASM contract code (if deployed)
    nonce: u64,                      // Operation counter
    balance: u128,                   // Native token balance
    auth_methods: Vec<AuthMethod>,   // Authentication configuration
}
```

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
- Multi-sig accounts work with all operation types: transfers, contract calls, staking, and governance votes

### Guardians

Trusted accounts that can help recover access if primary keys are lost. Social recovery enables account recovery without seed phrases.

### Session Credentials

Temporary keys with:

- **Time limits** — Auto-expire after a set duration
- **Spending limits** — Cap the total value of operations
- **Method restrictions** — Limit which contract methods can be called

Useful for dApp sessions where users grant limited permissions without exposing their primary keys.

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
