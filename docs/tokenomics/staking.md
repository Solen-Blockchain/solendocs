# Staking

Solen uses proof-of-stake consensus where validators and delegators stake SOLEN tokens to secure the network and earn rewards.

## Validators

Validators run nodes, propose blocks, and participate in consensus.

| Parameter | Value |
|-----------|-------|
| Minimum self-stake | 500,000 SOLEN |
| Slashing (double sign) | 10% of stake + jailing |
| Slashing (downtime) | 1% after 50 missed blocks |
| Unbonding period | 7 epochs |

### Validator Rewards

Validators earn from two sources:

1. **Epoch rewards** — Distributed from the staking allocation at each epoch boundary, proportional to total stake (self-stake + delegations)
2. **Transaction fees** — The treasury share of fees (50%) is governed by on-chain governance and can be directed to validators

### Reward Calculation

Each epoch, a fixed reward pool is distributed across all active validators:

```
validator_reward = epoch_reward_pool × (validator_total_stake / network_total_stake)
```

## Delegators

Any token holder can delegate to a validator without running infrastructure.

| Parameter | Value |
|-----------|-------|
| Minimum delegation | No minimum |
| Reward share | Proportional to stake relative to validator's total |
| Unbonding period | 7 epochs (same as validators) |
| Slashing risk | Shared with chosen validator |

### Delegator Rewards

Within a validator, rewards are split proportionally:

```
delegator_reward = validator_reward × (delegator_stake / validator_total_stake)
```

### Choosing a Validator

Delegators should consider:

- **Uptime** — Validators with downtime get slashed, affecting delegators
- **Commission** — Validators may charge a percentage of delegator rewards
- **Stake concentration** — Distributing delegation improves decentralization

## Slashing

### Double Signing

Signing two different blocks at the same height:

- **10% stake penalty** — Applied to validator and all delegators proportionally
- **Jailing** — Validator removed from the active set

### Downtime

Missing 50 consecutive blocks:

- **1% stake penalty** — Applied to validator and delegators

## Unbonding

When undelegating, tokens enter a 7-epoch unbonding period before they can be transferred. During unbonding:

- Tokens do not earn rewards
- Tokens are still subject to slashing
- The unbonding period provides security against long-range attacks

## Target Staking Ratio

The network targets **40-60%** of circulating supply staked:

- Below 40%: Governance may increase epoch rewards
- Above 60%: Governance may decrease rewards
