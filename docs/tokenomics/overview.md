# Token Overview

**SOLEN** is the native token of the Solen network, used for staking, fees, governance, and settlement.

## Supply

| Parameter | Value |
|-----------|-------|
| Total initial supply | 2,000,000,000 (2B) |
| Token symbol | SOLEN |
| Decimals | 8 (1 SOLEN = 100,000,000 base units) |
| Inflation | None at launch |

Validator rewards come from a pre-allocated staking pool. Governance can vote to enable inflation if the staking pool is depleted (expected around year 10).

## Initial Distribution

| Allocation | Tokens | % | Vesting |
|-----------|--------|---|---------|
| Staking & Validator Rewards | 500,000,000 | 25% | Released over 10 years via epoch rewards |
| Foundation Treasury | 400,000,000 | 20% | Governed by on-chain governance |
| Team & Founders | 300,000,000 | 15% | 1-year cliff, 3-year linear vest |
| Ecosystem Fund | 300,000,000 | 15% | dApp incentives, developer grants, partnerships |
| Community & Airdrops | 200,000,000 | 10% | Distributed at and after launch |
| Early Investors | 200,000,000 | 10% | 6-month cliff, 2-year linear vest |
| Liquidity & Market Making | 100,000,000 | 5% | Available at launch |

## Vesting Schedule

```
Year 0 (launch)
├── Circulating: Community (200M) + Liquidity (100M) = 300M (15%)
├── Staking rewards begin accruing
└── Team, investors locked

Year 1
├── Investor cliff ends → linear unlock begins
├── Team cliff ends → linear unlock begins
├── Staking rewards: ~50M released
└── Estimated circulating: ~400-500M

Year 2
├── Investors ~50% unlocked
├── Team ~25% unlocked
├── Staking rewards: ~100M cumulative
└── Estimated circulating: ~600-800M

Year 4
├── Team fully vested
├── Investors fully vested
└── Estimated circulating: ~1.0-1.2B

Year 10
└── Staking pool fully distributed → governance decides on inflation
```

## Economic Design

The Solen token has a **fixed initial supply** with no inflation at launch. The [fee burn](fee-model.md) creates deflationary pressure as network usage grows. Staking rewards come from a pre-allocated pool distributed over 10 years, after which governance decides whether to enable modest inflation.

The design prioritizes **long-term sustainability** over short-term incentive complexity.

## Deep Dives

- [Staking](staking.md) — Validators, delegators, and rewards
- [Governance](governance.md) — On-chain voting and proposals
- [Fee Model](fee-model.md) — Gas, burns, and treasury
