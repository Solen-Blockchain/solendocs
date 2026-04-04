# Governance

Token holders participate in on-chain governance by voting with their staked tokens.

## Parameters

| Parameter | Value |
|-----------|-------|
| Quorum | 30% of staked supply must participate |
| Pass threshold | 66.67% supermajority |
| Proposal deposit | 1,000 SOLEN (returned if passed, sent to treasury if rejected) |
| Voting period | 14 days (configurable in genesis) |
| Timelock | 3 epochs after passing before execution |

The voting period is set per-network in the genesis config (`governance_voting_period` in epochs). Testnet uses 14 epochs (~46 min) for fast testing. Mainnet will use ~604,800 epochs (~14 days at 2s blocks).

## Proposal Lifecycle

1. **Submission** — Any staked account can submit a proposal
2. **Voting** — Stakers vote yes/no for the voting period
3. **Tallying** — Quorum and supermajority thresholds checked
4. **Timelock** — Passed proposals wait 3 epochs before execution
5. **Execution** — Proposal is applied automatically

## What Governance Can Change

All network parameters are configurable via governance proposals. Current values are displayed on the [explorer homepage](https://solenscan.io).

| Parameter | CLI Command | Description |
|-----------|------------|-------------|
| Block time | `propose-block-time <key> <ms> "<desc>"` | Block production interval in milliseconds |
| Base fee | `propose-set-base-fee <key> <fee> "<desc>"` | Minimum fee per gas unit (base units) |
| Fee burn rate | `propose-set-burn-rate <key> <bps> "<desc>"` | Percentage of fees burned (basis points, 0-10000) |
| Epoch rewards | `propose-set-epoch-reward <key> <amount> "<desc>"` | Staking rewards per epoch (base units) |
| Min validator stake | `propose-min-stake <key> <solen_amount> "<desc>"` | Minimum self-stake to register as validator |
| Unbonding period | `propose-set-unbonding-period <key> <epochs> "<desc>"` | Cooldown before unstaked tokens can be withdrawn |
| Emergency pause | via system call | Halt all transaction processing |
| Emergency resume | via system call | Resume from emergency pause |

### Governance CLI Commands

```bash
# Propose changing the block time to 3 seconds
solen propose-block-time mykey 3000 "Increase block time for stability"

# Propose changing minimum validator stake to 1M SOLEN
solen --network testnet call mykey <governance-address> propose_set_min_validator_stake \
  --args "<new_min_stake_16_bytes_le><description>"

# Propose changing unbonding period to 14 epochs
solen --network testnet call mykey <governance-address> propose_set_unbonding_period \
  --args "<new_period_8_bytes_le><description>"

# Vote on a proposal (--yes or omit for no)
solen vote <keyname> <proposal_id> --yes --weight <solen_amount>

# Finalize after voting period ends
solen finalize-proposal <keyname> <proposal_id>

# Execute after timelock expires
solen execute-proposal <keyname> <proposal_id>
```

### Vote Weight Verification

Vote weight is verified against actual staked tokens at the time of voting. Voters must have staked tokens, and only stake that was active in the prior epoch is eligible for voting. This prevents flash-stake attacks where tokens are staked momentarily just to influence a vote.

## What Governance Cannot Change

These require a hard fork:

- Total supply cap
- Vesting schedules for already-allocated tokens
- Consensus mechanism

## Emergency Actions

Governance can perform emergency pause/resume of the network. This requires the same quorum and supermajority but may have an expedited voting period.

## System Contracts

Governance is implemented as a [system contract](../architecture/overview.md) with privileged access. The governance contract manages:

- Proposal storage and lifecycle
- Vote tallying and threshold checking
- Timelock enforcement
- Parameter update execution
