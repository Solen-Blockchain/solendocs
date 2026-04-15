# Testing Smart Contracts

How to test your Solen smart contracts before deploying to testnet.

## 1. Local Testing with Devnet

Start a local single-validator node for fast iteration:

```bash
./target/release/solen-node --no-p2p
```

This gives you:
- 2-second block times
- Pre-funded faucet account (10M SOLEN)
- No network dependencies
- Clean state on each restart

## 2. Simulate Before Submitting

Always simulate operations before submitting to the network. The CLI does this automatically, but you can also use the RPC directly:

```bash
# CLI (automatic simulation)
solen call faucet <CONTRACT> my_method --args "..."
# Output: "Simulated OK (gas: 12345). Submitting..."

# RPC (manual simulation)
curl -X POST http://127.0.0.1:29944 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "solen_simulateOperation",
    "params": [{
      "sender": [YOUR_SENDER_BYTES],
      "nonce": 0,
      "actions": [{"Call": {"target": [CONTRACT_BYTES], "method": "my_method", "args": ""}}],
      "max_fee": 1000000,
      "signature": ""
    }],
    "id": 1
  }'
```

Simulation runs the exact same execution path as a real transaction but doesn't commit state changes. It returns:

| Field | Description |
|-------|-------------|
| `success` | Whether the operation would succeed |
| `gas_used` | Exact gas consumption |
| `return_data` | Hex-encoded return data |
| `error` | Error message if failed |

## 3. Read-Only View Calls

For read-only queries, use `callView` — no signature needed:

```bash
# CLI
solen call-view <CONTRACT> total_supply

# TypeScript
const result = await client.callView(contractId, "total_supply");
console.log("Return data:", result.return_data);
```

## 4. Gas Estimation

Use simulation to estimate gas costs before setting `max_fee`:

```typescript
const sim = await client.simulateOperation(op);
if (sim.success) {
  // Set max_fee to 2x the estimated gas for safety margin
  op.max_fee = BigInt(sim.gas_used) * 2n;
  await client.submitOperation(op);
}
```

**Typical gas costs:**

| Operation | Approx. Gas |
|-----------|------------|
| Native transfer | ~100 |
| Counter increment (storage read + write + event) | ~6,000 |
| Token transfer (2 balance writes + event) | ~8,000 |
| Token mint (balance write + supply update + event) | ~12,000 |
| Contract deployment | ~1,000 (plus bytecode storage) |

## 5. Testing Contract State

After calling a method, verify state changes:

```bash
# Check an account's token balance
solen call-view <TOKEN_CONTRACT> balance_of --args "<ACCOUNT_HEX>"

# Check the chain status
solen status

# View the transaction on the explorer
# https://solenscan.io/tx/<BLOCK_HEIGHT>-<TX_INDEX>
```

## 6. Testing Error Cases

Test what happens when your contract fails:

```bash
# Transfer more tokens than the balance
solen call alice <TOKEN> transfer --args "${BOB}$(python3 -c "print((999999999).to_bytes(16, 'little').hex())")"
# Should fail: "Simulation failed: ..."

# Call with wrong method name
solen call alice <CONTRACT> nonexistent_method
# Should succeed but return 0 (default match arm)

# Call without required args
solen call alice <TOKEN> mint
# Should fail if the contract validates arg length
```

## 7. Reset and Replay

To start fresh:

```bash
# Stop the node, delete state, restart
rm -rf data/devnet
./target/release/solen-node --no-p2p
```

Your contract will need to be redeployed after a state reset.

## 8. Debugging Tips

- **Check return data** — If a call "succeeds" but does nothing, check the return data. A return value of `0` with no events usually means the method wasn't matched.
- **Watch events** — Events are your best debugging tool. Emit events at key points in your contract logic.
- **Use the explorer** — [solenscan.io](https://solenscan.io) shows all transactions, events, and contract interactions.
- **Check gas** — If gas is 0 for a contract call, the method likely returned before doing any work (bad method name or early return).
- **Compare simulation vs submission** — If simulation passes but submission fails, it's usually a nonce issue.
