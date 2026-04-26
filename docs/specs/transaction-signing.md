# Spec 004: Transaction Signing

**Status:** Stable (consensus-critical — any change is a hard fork)

This spec is the authoritative reference for producing a valid Solen transaction signature. It is the document any external wallet, hardware-wallet firmware, or third-party SDK should validate against. The reference implementation is [`UserOperation::signing_message`](https://github.com/Solen-Blockchain/solen/blob/main/crates/solen-types/src/transaction.rs) in `solen-types`.

## Address Format

A Solen address is **32 raw bytes**. There are two ways an address comes into existence:

1. **Externally keyed account** — the address is an Ed25519 public key. The wallet generates an Ed25519 keypair; the public key bytes *are* the address. No hashing, no derivation.
2. **Deployed contract account** — the address is `BLAKE3(deployer ‖ code_hash ‖ salt)`. Wallets do not produce these; only the `Deploy` action does.

Both forms are interchangeable on the wire — every RPC method that takes an address accepts either Base58 or hex.

| Form | Length | Example |
|------|--------|---------|
| Base58 (display, default) | 43–44 chars | `dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur` |
| Hex (interop) | 64 chars (optionally `0x`-prefixed) | `0x6c8a…7f` |

**Base58 alphabet** (Bitcoin alphabet, no checksum):
`123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`

Validation: decode → must yield exactly 32 bytes. There is **no checksum byte**; integrators relying on checksum-style validation (Base58Check) will reject valid addresses.

## Chain IDs

| Network | Chain ID |
|---------|----------|
| Mainnet | `1` |
| Testnet | `9000` |
| Devnet | `1337` |

Chain ID is encoded as a little-endian `u64` and is the first field of every signing message. It is the only cross-chain replay protection.

## UserOperation

A `UserOperation` is the unit a wallet signs and submits.

```rust
struct UserOperation {
    sender: [u8; 32],      // address of the signing account
    nonce: u64,            // must equal account.nonce at execution time
    actions: Vec<Action>,  // one or more actions, executed sequentially
    max_fee: u128,         // upper bound on fee debited from sender
    signature: Vec<u8>,    // see "Signature Encoding" below
}
```

### Action

Externally tagged enum (Rust `serde` default). The four variants:

```rust
enum Action {
    Transfer { to: [u8; 32], amount: u128 },
    Call     { target: [u8; 32], method: String, args: Vec<u8> },
    Deploy   { code: Vec<u8>, salt: [u8; 32] },
    SetAuth  { auth_methods: Vec<AuthMethod> },
}
```

JSON shape produced by `serde_json::to_vec(&actions)`:

```json
[{"Transfer":{"to":[1,2,...,32 bytes],"amount":1500000000}}]
[{"Call":{"target":[...],"method":"swap","args":[0,1,2,3]}}]
[{"Deploy":{"code":[...wasm bytes...],"salt":[...32 bytes...]}}]
[{"SetAuth":{"auth_methods":[{"Ed25519":{"public_key":[...32 bytes...]}}]}}]
```

Note: **byte arrays serialize as JSON arrays of numbers**, not hex strings. This is the serde default for `[u8; N]` and `Vec<u8>`. It is verbose but unambiguous.

## Signing Message

The bytes signed by Ed25519 are **exactly 96 bytes**, in this order:

```
offset  size  field         encoding
------  ----  ------------  --------
   0     8    chain_id      little-endian u64
   8    32    sender        raw bytes (32-byte address)
  40     8    nonce         little-endian u64
  48    16    max_fee       little-endian u128
  64    32    actions_hash  BLAKE3(serde_json::to_vec(&actions))
```

Reference (Rust):

```rust
pub fn signing_message(&self, chain_id: u64) -> Vec<u8> {
    let mut msg = Vec::with_capacity(96);
    msg.extend_from_slice(&chain_id.to_le_bytes());
    msg.extend_from_slice(&self.sender);
    msg.extend_from_slice(&self.nonce.to_le_bytes());
    msg.extend_from_slice(&self.max_fee.to_le_bytes());
    let actions_bytes = serde_json::to_vec(&self.actions).unwrap();
    msg.extend_from_slice(blake3::hash(&actions_bytes).as_bytes());
    msg
}
```

### JSON Canonicalization

`actions_hash` is the BLAKE3 digest of the bytes produced by Rust `serde_json::to_vec`. Any other JSON serializer must produce **byte-identical** output for the same logical actions. The relevant rules:

- **No insignificant whitespace.** Output is the compact form: no spaces after `:` or `,`, no newlines.
- **Field order follows Rust struct declaration order**, not alphabetical:
    - `Transfer`: `to`, then `amount`
    - `Call`: `target`, then `method`, then `args`
    - `Deploy`: `code`, then `salt`
    - `SetAuth`: `auth_methods`
- **Externally tagged enums**: `{"Variant":{...fields}}`.
- **Byte arrays as JSON number arrays.** `[u8; 32]` → `[<n>,<n>,...]` with each `<n>` an integer 0–255.
- **Integers as JSON number literals.** `u128` is written as a plain decimal integer with no quotes. JavaScript implementations must serialize via a BigInt-aware writer, since values above 2^53 will silently lose precision through `JSON.stringify` of `Number`.
- **Strings**: standard JSON escaping. `method` is ASCII in practice but the wire format is UTF-8.

The TypeScript reference signer (`solenwallet/src/lib/wallet.ts`, `solenbrowser/src/lib/wallet.ts`) and the Rust wallet SDK (`sdks/wallet-sdk-rs`) both produce byte-identical `actions_hash` against the canonical Rust implementation. New ports MUST be validated against the test vectors (see [Test Vectors](#test-vectors)).

!!! warning "JSON-of-actions is a known fragility"
    Hashing JSON is not the long-term plan. A future hard fork is expected to switch `actions_hash` to BLAKE3 of a Borsh-encoded actions array, removing the canonicalization burden entirely. External integrators should isolate the hashing step so it can be swapped at the fork height.

## Signature Encoding

The 32-byte `sender` field selects the account; the account's on-chain `auth_methods` determine how the `signature` field is interpreted.

| Auth method | `signature` field |
|-------------|-------------------|
| `Ed25519` | 64 raw bytes — Ed25519 signature over the 96-byte signing message |
| `Threshold` (M-of-N) | Concatenation of `pubkey[32] ‖ sig[64]` pairs (96 bytes each). At least `threshold` valid pairs from the configured signer set must be present |
| `Passkey` (WebAuthn / P-256) | `authenticatorData ‖ clientDataJSON ‖ r[32] ‖ s[32]`. The `clientDataJSON` challenge field MUST equal `base64url(signing_message)` |
| `Session` | 64-byte Ed25519 signature, verified against the session's ephemeral key. Subject to expiry, spending, target, and method restrictions |
| `Guardian` | Not used for `signature`; guardians authorize via the Guardian system contract |

For first-party wallets and most third-party integrations the relevant case is **Ed25519 → 64 bytes**.

## HD Derivation

**Today:** Solen's first-party wallets generate accounts from a 32-byte random seed; that seed *is* the Ed25519 secret key. No BIP-39, no derivation path. Address = Ed25519 public key.

**Planned:** BIP-39 mnemonic + SLIP-0010 Ed25519 derivation, registered SLIP-0044 coin type **`20260424`** ([SLIP PR #2010](https://github.com/satoshilabs/slips/pull/2010), pending merge). Path:

```
m/44'/20260424'/<account>'/0'
```

24-word mnemonics, no forced upgrade — random-key accounts continue to work. Hardware wallets and external integrators should target this path once the SLIP merges.

## Multi-Sig Signing Procedure

1. Each signer independently computes `signing_message(operation, chain_id)`.
2. Each signer produces an Ed25519 signature over those 96 bytes with their own key.
3. Collected signatures are concatenated as `pubkey[32] ‖ sig[64]` pairs.
4. The `signature` field of the `UserOperation` is the concatenation. Order does not matter; the verifier checks each pair independently against the on-chain signer set.

Duplicate signers are rejected at the `SetAuth` step, so a multi-sig signature with the same pubkey twice will fail signature verification.

## Test Vectors

The canonical vectors file is checked in at [`tools/vectors/vectors.json`](https://github.com/Solen-Blockchain/solen/blob/main/tools/vectors/vectors.json). Each entry contains:

- `chain_id`, `operation.sender_hex`, `operation.nonce`, `operation.max_fee_dec`
- `operation.actions_canonical_json` — the **exact** UTF-8 bytes that get hashed
- `operation.actions_hash_hex` — `BLAKE3(actions_canonical_json)`
- `signing_message_hex` — the 96-byte message that gets signed (hex)
- `signature_hex` — the Ed25519 signature produced by the documented test seed (hex)

Coverage includes `Transfer` / `Call` / `Deploy` / `SetAuth` across the three chain IDs, and edge values for `nonce` (`0`, `u64::MAX`), `max_fee` (`0`, `u128::MAX`), and `amount` (`0`, `u128::MAX`).

**Regenerate** (after a non-fork change to the implementation, e.g. test seed update):

```bash
cargo run -p solen-vectors --release -- generate --out tools/vectors/vectors.json
```

**Verify** that the current implementation reproduces the pinned file byte-for-byte:

```bash
cargo run -p solen-vectors --release -- check --in tools/vectors/vectors.json
```

A new language port MUST round-trip every vector — recompute `actions_canonical_json` → `actions_hash` → `signing_message` → signature — before being trusted to sign mainnet operations.

## Submitting

Once the operation is signed, submit it via the [`solen_submitOperation`](../api/json-rpc.md#solen_submitoperation) JSON-RPC method. The node verifies signatures, nonce, balance, and auth-method restrictions before admitting the operation to the mempool.

## Common Pitfalls

- **Wrong endianness on `max_fee`.** It is `u128` little-endian, *all 16 bytes*. A common mistake is to write only the low 8 bytes (LE u64) and leave the upper 8 bytes uninitialized — this works for small fees but produces a different signing message than the canonical encoding when junk is present.
- **`amount` as a JS `Number` above 2^53.** `JSON.stringify({amount: 1e20})` produces `1e+20`, not `100000000000000000000`, and the Rust verifier will fail to parse. Use a BigInt-aware JSON writer.
- **Address re-encoding.** Don't decode Base58 → re-encode → decode again before hashing; round-tripping is lossy if your Base58 implementation strips leading zeros.
- **Treating the signing message as the transaction hash.** It isn't. The transaction hash is a separate value computed by the node.
- **Cross-chain replay**. The `chain_id` is the only protection. Always set it to the chain you are submitting to. Mainnet is `1`, not `0`.
