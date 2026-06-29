# Post-Quantum Security

Solen accounts can opt in to **quantum-resistant signatures** — built in, today, made practical by the [smart-account](smart-accounts.md) model. A large-scale quantum computer would break the elliptic-curve cryptography behind Ed25519 and WebAuthn (P-256); ML-DSA does not rely on any assumption a quantum computer is known to defeat.

## What is — and isn't — at risk

Only **signatures** are vulnerable to quantum attack. The rest of the stack already survives:

| Primitive | Used for | Quantum status |
|-----------|----------|----------------|
| Ed25519 / P-256 ECDSA | account signatures | **Broken by Shor's algorithm** — a quantum computer recovers the private key from the public key |
| BLAKE3, SHA-256 | hashing, state root, Merkle | Safe — Grover only halves security (256→128 bit) |
| AES-256-GCM, Argon2id | keystore encryption | Safe — Grover only halves security (still 128-bit) |

So "quantum-proofing" Solen is precisely "post-quantum **signatures**" — which is exactly what an account can opt into.

!!! info "Why now — harvest-now, decrypt-later"
    A public key visible on-chain today can be attacked once a cryptographically-relevant quantum computer exists. Rotating high-value accounts to a post-quantum key **before** that day is the defensive move. Solen makes that a single command, with no change of address.

## The scheme: ML-DSA-65 (FIPS 204)

Solen uses **ML-DSA-65** — the NIST-standardized, module-lattice signature scheme (FIPS 204, formerly CRYSTALS-Dilithium), at NIST **security category 3**.

| | Ed25519 (classical) | ML-DSA-65 (post-quantum) |
|---|---|---|
| Public key | 32 bytes | 1952 bytes |
| Signature | 64 bytes | 3309 bytes |
| Verification | deterministic | deterministic (consensus-safe) |

The larger key and signature are the cost of quantum resistance — fine for high-assurance, opt-in accounts. Verification is a deterministic pure function, so every validator reaches the same verdict.

## Two opt-in modes

Both are added with a [`SetAuth`](smart-accounts.md#authentication-methods) operation, exactly like any other auth method.

| Mode | Auth method | A signature must carry… | Use when |
|------|-------------|-------------------------|----------|
| **Pure post-quantum** | `MlDsa` | a valid ML-DSA-65 signature | you want full quantum resistance |
| **AND-hybrid** | `Hybrid` | **both** a valid Ed25519 **and** a valid ML-DSA-65 signature | transition-period defense-in-depth — secure unless *both* schemes break |

Hybrid is the conservative choice: an attacker must break *both* the classical and the post-quantum scheme to forge a signature. Conveniently, a single 32-byte seed derives both keys (breaking Ed25519 reveals its scalar, not the seed, so the ML-DSA key stays safe).

## Upgrading an account

One command rotates an existing account to post-quantum auth. **The address never changes** — there is no fund migration, unlike chains where the address is a hash of the public key.

=== "Pure ML-DSA"

    ```bash
    solen --chain-id 1 key quantum-upgrade mykey
    ```

=== "Hybrid (Ed25519 + ML-DSA)"

    ```bash
    solen --chain-id 1 key quantum-upgrade mykey --hybrid
    ```

This generates a fresh post-quantum key, submits a `SetAuth` **signed by your current key**, and — only after the network accepts it — rewrites the local key so the same name now signs with the new scheme.

Equivalent `SetAuth` actions:

```json
{ "SetAuth": { "auth_methods": [ { "MlDsa": { "public_key": [/* 1952 bytes */] } } ] } }
```

```json
{ "SetAuth": { "auth_methods": [ { "Hybrid": {
    "ed25519_public_key": [/* 32 bytes */],
    "ml_dsa_public_key":  [/* 1952 bytes */]
} } ] } }
```

## Availability & activation

Post-quantum authentication is **opt-in and network-gated**. It is a consensus-affecting capability, so it ships dormant and is enabled fleet-wide at a coordinated activation height (`pq_auth_height`).

!!! warning "Activate before you rotate"
    On a network where post-quantum auth is **not yet active**, an account rotated to ML-DSA-only (or Hybrid) cannot transact until activation. Upgrade accounts only on a chain that has post-quantum auth enabled. The CLI prints this warning.

## Signing & interoperability

The signed message is the **same 96-byte [signing message](../specs/transaction-signing.md#signing-message)** as Ed25519 — only the signature scheme over it changes. Signature encodings:

- `MlDsa` — 3309-byte ML-DSA-65 signature (empty FIPS 204 context).
- `Hybrid` — `ed25519[64] ‖ ml_dsa[3309]` = 3373 bytes; both halves must verify.

The signers are **cross-implementation verified**: the Rust node (`fips204`) and the TypeScript SDK (`@noble/post-quantum`) produce keys and signatures that validate against each other, pinned by committed test vectors. See the [Transaction Signing spec](../specs/transaction-signing.md#signature-encoding).

## SDK support

- **Rust** ([SDK](../tools/rust-sdk.md)): `sign_operation_ml_dsa`, `sign_operation_hybrid`, `build_quantum_upgrade`, `build_hybrid_upgrade`.
- **TypeScript** ([SDK](../tools/typescript-sdk.md)): `signOperationMlDsa`, `signOperationHybrid`, `mlDsaKeygenFromSeed`, `signingMessage`, `verifyMlDsa`, `verifyHybrid`.

## See also

- [Smart Accounts](smart-accounts.md) — the pluggable auth model post-quantum builds on
- [Transaction Signing](../specs/transaction-signing.md) — authoritative signature spec
