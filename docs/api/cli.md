# CLI Reference

The `solen` CLI provides commands for interacting with Solen nodes from the terminal.

## Installation

```bash
cargo build --bin solen --release
# Binary at: ./target/release/solen
```

## Commands

### `solen status`

Show the current chain status.

```bash
solen status
```

```
Solen Network Status
────────────────────────────────────────
  Height:      12
  State root:  78c88b3eb083...805d24e9
  Pending ops: 0
  Epoch:       0
  Proposer:    8a88e3dd7409f195...
  Gas used:    0
```

---

### `solen balance <account>`

Query an account's native token balance. Account can be a key name, Base58 address, or hex address.

```bash
solen balance faucet
# Output: 1000000000
```

---

### `solen account <account>`

Get full account details including nonce and code hash.

```bash
solen account faucet
```

```
Account
────────────────────────────────────────
  ID:        197f6b23e16c8532...
  Balance:   1000000000
  Nonce:     0
  Code hash: (none)
```

---

### `solen block [height]`

Get block information. Shows the latest block if no height is given.

```bash
solen block          # latest block
solen block 100      # block at height 100
```

```
Block #100
────────────────────────────────────────
  Epoch:      1
  Proposer:   8a88e3dd7409f195...
  State root: 78c88b3eb08313c6...
  Txs:        2
  Gas used:   200
  Time:       4s ago
```

---

### `solen transfer <from> <to> <amount>`

Transfer tokens between accounts. Simulates first, then submits.

```bash
solen transfer mykey 139c31e8543b1962... 5000
```

The sender must be a key name in your keystore. The recipient can be a key name, Base58 address, or hex address.

---

### `solen key` — Key Management

#### `solen key generate <name>`

Generate a new Ed25519 keypair.

```bash
solen key generate alice
```

> **Security note:** The seed is stored in the keystore file and is never printed to the console. Use `solen key lock` to encrypt the keystore with a password (Argon2id + AES-256-GCM) for additional protection.

#### `solen key import <name> <hex-seed>`

Import a key from a 32-byte hex seed.

```bash
solen key import faucet 2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a
```

#### `solen key list`

List all stored keys.

```bash
solen key list
```

```
NAME         ACCOUNT ID           PUBLIC KEY
──────────────────────────────────────────────────────────────────────
alice        139c31e8543b1962...  139c31e8543b1962...
faucet       197f6b23e16c8532...  197f6b23e16c8532...
```

---

### `solen validators`

List all validators and their stake.

```bash
solen validators
```

```
STATUS ADDRESS            SELF STAKE      DELEGATED          TOTAL
──────────────────────────────────────────────────────────────────────
GENSIS 8a88e3dd7409f195...        1000000              0        1000000
GENSIS 8139770ecb104aef...        1000000              0        1000000
GENSIS ed4928c628d1c2c6...        1000000              0        1000000
GENSIS ca93ac1705187071...        1000000              0        1000000

4 validators (4 active)
```

---

### `solen register-validator <from> <amount>`

Register as a new validator with self-stake. Requires minimum 500,000 SOLEN.

```bash
solen register-validator mykey 500000
```

The sender's account becomes the validator. The stake amount is deducted from the account balance. The validator is eligible for block production starting the next epoch.

---

### `solen stake <from> <validator> <amount>`

Delegate tokens to a validator.

```bash
solen stake mykey 8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c 10000
```

### `solen unstake <from> <validator> <amount>`

Begin undelegation from a validator. Automatically withdraws any previously matured undelegations. New unstaked funds are available after the unbonding period (7 epochs).

```bash
solen unstake mykey 8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c 5000
```

---

### `solen withdraw-stake <from>`

Withdraw all matured unstaked tokens. Tokens that have passed the 7-epoch unbonding period are credited to your balance.

```bash
solen withdraw-stake mykey
```

---

### `solen unjail <from>`

Reactivate a jailed validator. A validator is jailed after being slashed for downtime (50 missed blocks). The validator rejoins the active consensus set at the next epoch boundary.

```bash
solen unjail my-validator
```

The sender must be the jailed validator's key.

---

### `solen propose-block-time <from> <new-block-time-ms> <description>`

Submit a governance proposal to change the block time.

```bash
solen propose-block-time mykey 3000 "Increase block time to 3 seconds for stability"
```

Proposals require a deposit and go through a voting period before they can be finalized and executed.

---

### `solen propose-min-stake <from> <new-min-stake> <description>`

Submit a governance proposal to change the minimum validator self-stake.

```bash
solen propose-min-stake mykey 15000 "Lower min stake for testnet onboarding"
```

The amount is in SOLEN (e.g., 15000 = 15,000 SOLEN). Takes effect after voting period + timelock.

---

### `solen vote <from> <proposal-id> --yes [--weight <amount>]`

Vote on a governance proposal. Use `--yes` to vote in favor, omit it to vote against.

```bash
# Vote yes with default weight (1 SOLEN)
solen vote mykey 1 --yes

# Vote yes with 1000 SOLEN weight
solen vote mykey 1 --yes --weight 1000

# Vote no
solen vote mykey 1
```

---

### `solen finalize-proposal <from> <proposal-id>`

Finalize a governance proposal after the voting period has ended. Tallies votes and determines if the proposal passed or was rejected.

```bash
solen finalize-proposal mykey 1
```

---

### `solen execute-proposal <from> <proposal-id>`

Execute a passed governance proposal after the timelock period. Applies the proposed parameter change on-chain.

```bash
solen execute-proposal mykey 1
```

---

### `solen register-paymaster <from>`

Register a deployed contract as a paymaster (fee sponsor). Paymasters can pay transaction fees on behalf of users, enabling gasless transactions.

```bash
solen register-paymaster my-contract
```

The sender must be the key of a deployed contract account.

---

### `solen unregister-paymaster <from>`

Remove a contract's paymaster registration.

```bash
solen unregister-paymaster my-contract
```

---

### `solen initiate-recovery <from> <target> <new-public-key>`

Initiate guardian recovery for a lost account. The sender must be a guardian of the target account.

```bash
solen initiate-recovery guardian-key <target-account-hex> <new-public-key-hex>
```

This creates a pending recovery request. Other guardians must confirm it before it can be executed.

---

### `solen confirm-recovery <from> <recovery-id>`

Confirm a pending guardian recovery. The sender must be a guardian of the target account.

```bash
solen confirm-recovery guardian-key 1
```

A recovery requires confirmations from a majority of guardians before it can be executed.

---

### `solen cancel-recovery <from> <recovery-id>`

Cancel a pending recovery. The sender must be the owner of the account being recovered.

```bash
solen cancel-recovery mykey 1
```

---

### `solen execute-recovery <from> <recovery-id>`

Execute a recovery after the timelock expires and sufficient guardian confirmations have been received. Anyone can call this once the conditions are met.

```bash
solen execute-recovery mykey 1
```

The target account's public key is replaced with the new key specified during initiation.

---

### `solen register-rollup <from> <rollup-id> <name> [--proof-type <type>]`

Register a rollup domain on L1. Requires a 10,000 SOLEN deposit.

```bash
solen register-rollup mykey 1 "My Rollup" --proof-type mock
```

---

### `solen key lock`

Encrypt all keys in the local keystore with a password (Argon2id + AES-256-GCM).

```bash
solen key lock
# Prompts for new password (min 8 chars) + confirmation
```

### `solen key unlock`

Decrypt all keys, removing password protection.

```bash
solen key unlock
# Prompts for current password
```

### `solen key change-password`

Change the keystore password.

```bash
solen key change-password
# Prompts for current password, then new password + confirmation
```

---

### `solen multisig <from> --threshold <N> --signers <key1,key2,...>`

Convert an account to multi-sig (threshold signing). The account must currently be owned by `<from>`.

```bash
# Create a 2-of-3 multi-sig
solen multisig mykey --threshold 2 --signers "aabb...01,ccdd...02,eeff...03"
```

After conversion, all future operations on this account require at least `N` valid signatures from the signers list. The signature field must contain concatenated `pubkey[32] + sig[64]` pairs (96 bytes each).

---

### `solen claim-vesting <from>`

Claim vested tokens from your vesting schedule (team or investor allocations).

```bash
solen claim-vesting mykey
```

If tokens are available to claim, they will be credited to your account balance. The command calls the vesting system contract (`0xFFFF...FF06`) with the `claim` method.

---

### System Contract Addresses

System contracts have well-known addresses. These are conventionally written in hex since they are compile-time constants, but Base58 format is also accepted.

| Contract | Address |
|----------|---------|
| Staking | `0xFFFF...FF01` |
| Governance | `0xFFFF...FF02` |
| Bridge | `0xFFFF...FF03` |
| Treasury | `0xFFFF...FF04` |
| Intent Pool | `0xFFFF...FF05` |
| Vesting | `0xFFFF...FF06` |
| Paymaster Registry | `0xFFFF...FF07` |

Call them using `solen call`:

```bash
# Delegate to a validator via system call
solen call mykey ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff01 delegate --args "<validator><amount>"
```

---

### `solen deploy <from> <wasm-file>`

Deploy a WASM contract.

```bash
solen deploy faucet target/wasm32-unknown-unknown/release/my_contract.wasm
```

The CLI will:

1. Simulate the deployment to estimate gas
2. Submit the deploy operation
3. Print the contract ID and code hash

---

### `solen call <from> <contract-id> <method> [--args <hex>]`

Call a contract method.

```bash
# No arguments
solen call faucet <CONTRACT_ID> increment

# With arguments (hex-encoded)
solen call faucet <CONTRACT_ID> mint --args "197f6b23e16c8532...40420f0000..."
```

The CLI simulates the call first, then submits if simulation succeeds.

---

### Global Options

| Option | Description |
|--------|-------------|
| `--rpc <URL>` | JSON-RPC endpoint (overrides `--network` default) |
| `--network <NETWORK>` | Network preset: `devnet` (default), `testnet`, or `mainnet`. Sets RPC endpoint and chain ID. |
| `--chain-id <ID>` | Chain ID for signing (overrides `--network` default) |

> **Address format:** All commands accept account addresses in either Base58 (Bitcoin alphabet, ~44 characters) or hex (64 characters) format. Key names from the local keystore are also accepted wherever an address is expected.

Network defaults:

| Network | RPC Endpoint | Chain ID |
|---------|-------------|----------|
| devnet | `http://127.0.0.1:29944` | 1337 |
| testnet | `https://testnet-rpc.solenchain.io` | 9000 |
| mainnet | `https://rpc.solenchain.io` | 1 |

```bash
# Use a different RPC endpoint
solen --rpc http://127.0.0.1:29944 status

# Use testnet defaults
solen --network testnet status
```
