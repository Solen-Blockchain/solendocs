# Intent-Aware Execution

Solen supports **intent-based transactions** where users express desired outcomes rather than specifying exact execution steps.

## How Intents Work

1. **User submits an intent** — A declarative expression of what they want to achieve
2. **Solvers compete** — Off-chain solvers find optimal execution paths
3. **Best solution selected** — The protocol selects the best solver submission
4. **Execution and settlement** — The winning solution is executed on-chain

## Example

Instead of constructing a swap transaction with exact parameters:

```
Intent: "Swap 100 SOLEN for at least 50 USDC"
```

Solvers compete to find the best execution, potentially:

- Routing through multiple liquidity pools
- Splitting orders across venues
- Timing execution for optimal pricing

## Benefits

### MEV Reduction

By letting solvers compete openly, intent-based execution reduces the ability of validators to extract value through transaction reordering.

### Better Execution

Solvers can access off-chain liquidity and use sophisticated routing that individual users cannot replicate.

### Simpler UX

Users express what they want, not how to achieve it. No need to understand AMM mechanics, slippage settings, or gas optimization.

## Intent Structure

```rust
Intent {
    sender: AccountId,
    conditions: Vec<Condition>,    // What the user wants
    constraints: Vec<Constraint>,  // Bounds on acceptable outcomes
    deadline: u64,                 // Block height deadline
    signature: Vec<u8>,
}
```

## Solver Framework

The `solen-intents` crate provides:

- Intent parsing and validation
- Solver registration and reputation tracking
- Solution scoring and selection
- Auditable execution traces

Solvers must stake tokens to participate, creating economic accountability for solution quality.
