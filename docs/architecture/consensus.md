# Consensus

Solen uses a BFT (Byzantine Fault Tolerant) proof-of-stake consensus engine with deterministic finality.

## Overview

- **Algorithm:** BFT with round-robin block proposers
- **Finality:** Deterministic (no probabilistic confirmation)
- **Quorum:** 2/3+ of stake-weighted validator votes
- **Block times:** 6s (mainnet), 2s (testnet/devnet)

## Validators

Validators run nodes, propose blocks, and participate in consensus voting.

| Parameter | Value |
|-----------|-------|
| Minimum self-stake | 500,000 SOLEN |
| Slashing (double sign) | 10% of stake + jailing |
| Slashing (downtime) | 1% after 50 consecutive missed blocks |
| Unbonding period | 7 epochs |

### Block Production

Validators are selected to propose blocks using **stake-weighted randomized selection**. The proposer for each block height is determined by:

1. An **epoch seed** derived from the previous epoch's last block hash (unpredictable until the epoch ends)
2. A deterministic hash of the seed + height, mapped to the validator set proportional to stake
3. Validators with more stake are selected proportionally more often

A proposed block requires **2/3+ stake-weighted attestation quorum** to be finalized. If the designated proposer is offline, backup proposers take over in stake-weighted priority order.

### Proposer Signatures

Block headers include an Ed25519 signature from the proposer, proving authorship. This prevents relay attacks where a malicious peer changes the proposer field.

### Epochs

An **epoch** is a fixed interval of 100 blocks used for:

- Validator set rotation
- Reward distribution
- Checkpoint creation and finalization
- Epoch seed rotation (for proposer selection randomization)

At each epoch boundary, rewards from the staking allocation pool are distributed proportionally to validator stake.

### Finalized Checkpoints

At every epoch boundary, a **finalized checkpoint** is created:

1. The block at the epoch boundary becomes the checkpoint candidate
2. Each validator signs and broadcasts their attestation of the checkpoint
3. Once 2/3+ of stake attests, the checkpoint is **finalized** and persisted
4. All blocks at or before a finalized checkpoint are irreversible
5. New nodes syncing via snapshot verify the checkpoint's validator attestations

Finalized checkpoints prevent **long-range attacks** — an attacker cannot rewrite history past a finalized checkpoint without controlling 2/3+ of stake.

### Fork Scoring

When competing blocks arrive at the same height from different proposers, the node uses **stake-weighted fork scoring** to choose the best block. The block from the higher-priority proposer (earlier in the stake-weighted order for that height) replaces any lower-priority pending block.

### Peer Reputation

The P2P layer tracks peer behavior and temporarily bans misbehaving peers:

- Decode failures, invalid blocks, and rate limit violations decrease a peer's score
- Valid blocks and attestations increase score
- Peers below a threshold score are banned for 5 minutes
- Scores decay toward zero over time, allowing recovery

## Slashing

Validators can be slashed for two types of misbehavior:

### Double Signing

Signing two different blocks at the same height results in:

- **10% stake penalty** applied to the validator and their delegators
- **Jailing** — the validator is removed from the active set

### Downtime

Missing 50 consecutive blocks results in:

- **1% stake penalty** applied to the validator and their delegators
- **Jailing** — the validator is removed from the active set

### Unjailing

A jailed validator can be reactivated by submitting an `unjail` transaction. The validator rejoins the active consensus set at the next epoch boundary and resumes block production in the round-robin rotation.

### Backup Proposers

When the designated proposer is offline, backup proposers take over in deterministic order — the next validator in the round-robin sequence after the designated proposer. The first backup waits **3x block_time** (18s on mainnet) before proposing. Each subsequent backup waits an additional **2x block_time** (12s on mainnet) beyond the previous backup's deadline. This ensures liveness even when multiple consecutive proposers are unavailable.

All slashed funds are sent to the **Foundation Treasury** (`0xFFFF...FF04`).

> **Note:** Slashing is deterministic and on-chain — penalties are executed as system transactions within blocks, not applied out-of-band.

## Dynamic Validator Set

The consensus engine syncs with the staking contract at each epoch boundary (every 100 blocks). New validators who register via the `register` staking method are automatically added to the active proposer rotation at the next epoch. Validators earn rewards starting one epoch after registration.

## Checkpoints

The consensus engine periodically creates checkpoints — snapshots of the validator set and finalized state. These are used for:

- Fast sync for new nodes
- Cross-domain proof anchoring
- State verification

## Security

The consensus engine includes several hardened security measures:

- **State root verification:** Every finalized block includes a state root that is independently verified. During initial sync, the node validates state roots against the received block data to detect tampered state.
- **RPC rate limiting:** Write operations are rate-limited to prevent denial-of-service attacks on the mempool (see [JSON-RPC Rate Limits](../api/json-rpc.md#rate-limits)).
- **Attestation validation:** Block attestations (votes) are validated against the active validator set — only validators with stake in the current epoch can participate in consensus voting. Duplicate or invalid attestations are rejected.
- **Proposer validation:** Block proposals are verified to come from the expected proposer in the round-robin rotation. Blocks from unauthorized proposers are rejected.

## Configuration

Consensus parameters can be modified through [governance](../tokenomics/governance.md) proposals:

- Block time
- Epoch length
- Reward amounts
- Staking minimums
- Slashing percentages
