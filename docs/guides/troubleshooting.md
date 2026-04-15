# Troubleshooting

Common issues and how to fix them.

## Transaction Errors

### "Simulation failed: invalid nonce"

Your nonce is out of sync. This usually happens when sending multiple transactions quickly.

**Fix:** Use `getNextNonce` instead of reading the nonce from `getAccount`:

```bash
solen --network testnet call v1 <CONTRACT> <METHOD>
# The CLI does this automatically, but if sending rapidly, wait for the previous tx to land.
```

### "insufficient balance"

Your account doesn't have enough SOLEN to cover the transaction fee.

**Fix:** Get testnet tokens from the faucet:

```bash
curl -X POST https://testnet-faucet.solenchain.io/faucet \
  -H "Content-Type: application/json" \
  -d '{"address": "YOUR_ADDRESS"}'
```

### "mempool full or duplicate"

Either the mempool is at capacity or you submitted the same (sender, nonce) pair twice.

**Fix:** Wait a few seconds for the mempool to clear, then retry. If using rapid submissions, use `getNextNonce` to get the correct nonce.

### "nonce too far ahead"

Your transaction's nonce is more than 64 ahead of your on-chain nonce. This is a spam protection limit.

**Fix:** Wait for your pending transactions to be included in blocks before submitting more.

### "signature verification failed"

The operation signature doesn't match any of the account's auth methods.

**Fix:**
- Verify you're signing with the correct key for the account
- Check the chain ID matches the network (testnet = 9000, devnet = 1337)
- Ensure the signing message format matches: `chain_id[8] + sender[32] + nonce[8] + max_fee[16] + blake3(actions_json)[32]`

## Contract Errors

### "invalid WASM bytecode"

The WASM module doesn't have the required exports or is malformed.

**Fix:**
- Ensure your contract exports `call` and `memory`
- Build with `--target wasm32-unknown-unknown --release`
- Check `crate-type = ["cdylib"]` in Cargo.toml

### "out of gas"

Your contract exceeded the 1,000,000 fuel limit.

**Fix:**
- Reduce the number of storage operations per call
- Batch fewer operations per transaction
- Check for infinite loops in your contract logic

### "execution trapped"

The WASM contract panicked (e.g., array out of bounds, explicit `unreachable`).

**Fix:**
- Check array access bounds in your contract
- Ensure `storage::get()` return values are checked for `None`
- Use `if args.len() < EXPECTED_LEN { return 0; }` before reading args

### Contract return data is empty

The contract didn't call `sdk::return_value()`.

**Fix:** Add `sdk::return_value(&data)` as the last line of your method, and return its value from `call`.

## Node Issues

### "Too many open files"

The node or service hit the OS file descriptor limit.

**Fix:**
```bash
# Temporary
ulimit -n 65536

# Permanent (systemd service)
# Add to [Service] section:
LimitNOFILE=65536
```

### Node stuck after "starting block production"

The node can't reach peers or thinks it's partitioned.

**Fix:**
1. Check peer connectivity: other validators must be reachable on the P2P port (40333 for testnet)
2. Restart the node — this resets the partition detection counter
3. If data is corrupted: stop the node, delete the data directory, restart to resync from snapshot

### "state root mismatch — requesting snapshot resync"

Your node's state diverged from the network.

**Fix:** This triggers automatic resync. Wait for the snapshot download to complete. If it fails, manually resync:

```bash
systemctl stop solen-node
rm -rf /opt/solen/data/testnet
systemctl start solen-node
```

## RPC Errors

| Code | Meaning | Common Cause |
|------|---------|-------------|
| `-32600` | Invalid request | Malformed JSON-RPC body |
| `-32601` | Method not found | Typo in method name |
| `-32602` | Invalid params | Wrong parameter types or count |
| `-32603` | Internal error | Rate limited or server error |
| `-32000` | Account not found | Address doesn't exist on-chain |
| `-32001` | Insufficient balance | Not enough SOLEN for tx |
| `-32002` | Invalid nonce | Nonce already used or too far ahead |
| `-32003` | Signature failed | Wrong key or chain ID |

## Getting Help

- **Discord:** [discord.com/invite/QGxv8bebJc](https://discord.com/invite/QGxv8bebJc)
- **Telegram:** [t.me/solenblockchain](https://t.me/solenblockchain)
- **GitHub Issues:** [github.com/Solen-Blockchain/solen/issues](https://github.com/Solen-Blockchain/solen/issues)
