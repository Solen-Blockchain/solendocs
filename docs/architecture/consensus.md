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

Validators take turns proposing blocks in round-robin order. A proposed block requires 2/3+ stake-weighted votes to be finalized.

### Epochs

An **epoch** is a fixed interval of blocks used for:

- Validator set rotation
- Reward distribution
- Checkpoint creation

At each epoch boundary, rewards from the staking allocation pool are distributed proportionally to validator stake.

## Slashing

Validators can be slashed for two types of misbehavior:

### Double Signing

Signing two different blocks at the same height results in:

- **10% stake penalty** applied to the validator and their delegators
- **Jailing** — the validator is removed from the active set

### Downtime

Missing 50 consecutive blocks results in:

- **1% stake penalty**
- The validator remains in the active set but may be rotated out

## Checkpoints

The consensus engine periodically creates checkpoints — snapshots of the validator set and finalized state. These are used for:

- Fast sync for new nodes
- Cross-domain proof anchoring
- State verification

## Configuration

Consensus parameters can be modified through [governance](../tokenomics/governance.md) proposals:

- Block time
- Epoch length
- Reward amounts
- Staking minimums
- Slashing percentages
