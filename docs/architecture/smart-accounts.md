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

### Threshold Signatures

Require M-of-N signatures to authorize an operation. Useful for multi-sig wallets, team treasuries, and DAOs.

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
