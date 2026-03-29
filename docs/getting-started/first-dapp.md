# Build Your First dApp on Solen

Deploy a token contract, mint tokens, and transfer them between accounts — all from the command line in about 15 minutes.

**By the end you will have:**

- A running local Solen node
- A deployed SRC-20 token contract (ERC20-equivalent)
- Minted tokens to an account
- Transferred tokens between accounts
- Queried balances and chain state

## Step 1: Build the Tools

```bash
# Install the WASM target
rustup target add wasm32-unknown-unknown

# Build the node and CLI
cargo build --bin solen-node --bin solen --release
```

## Step 2: Start the Node

Open a terminal and start a local devnet:

```bash
./target/release/solen-node --no-p2p
```

You should see:

```
INFO solen_node: === Solen Node v0.1.0 ===
INFO solen_storage::rocks: RocksDB opened path=data/solen-db
INFO solen_node: genesis state initialized
INFO solen_rpc::server: JSON-RPC server started addr=127.0.0.1:9944
INFO solen_node: Node running. Press Ctrl+C to stop.
INFO solen_consensus::engine: block finalized height=1 ops=0 gas=0
```

The node produces blocks every 2 seconds. Leave this running.

## Step 3: Set Up Your Wallet

In a second terminal, import the devnet faucet key (pre-funded with 1 billion tokens):

```bash
./target/release/solen key import faucet \
  2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a
```

Generate a second key for another user:

```bash
./target/release/solen key generate alice
```

Check your keys:

```bash
./target/release/solen key list
```

```
NAME         ACCOUNT ID           PUBLIC KEY
──────────────────────────────────────────────────────────────────────
alice        616c696365000000...  139c31e8543b1962...
faucet       6661756365740000...  197f6b23e16c8532...
```

## Step 4: Check the Chain

```bash
./target/release/solen status
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

Check the faucet balance:

```bash
./target/release/solen balance faucet
# Output: 1000000000
```

## Step 5: Build the Token Contract

```bash
cd examples/contracts/token
cargo build --target wasm32-unknown-unknown --release
cd ../../..
```

The compiled contract is at `target/wasm32-unknown-unknown/release/solen_example_token.wasm`.

## Step 6: Deploy the Token

```bash
./target/release/solen deploy faucet \
  target/wasm32-unknown-unknown/release/solen_example_token.wasm
```

```
Simulated OK (gas: 1000). Deploying...
Contract deployed successfully.
  Contract ID: a7f3b2c1d4e5f6a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1
  Code hash:   9c8d7e6f5a4b3c2d...
```

!!! important
    Save the **Contract ID** — you'll need it for the next steps. The examples below use `TOKEN_ID` as a placeholder.

## Step 7: Initialize the Token

```bash
./target/release/solen call faucet <TOKEN_ID> init
```

## Step 8: Mint Tokens

Mint 1,000,000 tokens to the faucet account:

```bash
# Faucet account ID (32 bytes hex)
TO="6661756365740000000000000000000000000000000000000000000000000000"

# 1,000,000 as u128 little-endian (16 bytes hex)
AMOUNT=$(python3 -c "print((1000000).to_bytes(16, 'little').hex())")

./target/release/solen call faucet <TOKEN_ID> mint --args "${TO}${AMOUNT}"
```

## Step 9: Check the Token Balance

```bash
ACCOUNT="6661756365740000000000000000000000000000000000000000000000000000"
./target/release/solen call faucet <TOKEN_ID> balance_of --args "${ACCOUNT}"
```

The return data contains the balance as a 16-byte little-endian u128.

## Step 10: Transfer Tokens

Transfer 50,000 tokens from faucet to alice:

```bash
ALICE="616c696365000000000000000000000000000000000000000000000000000000"
AMOUNT=$(python3 -c "print((50000).to_bytes(16, 'little').hex())")

./target/release/solen call faucet <TOKEN_ID> transfer --args "${ALICE}${AMOUNT}"
```

Wait for the next block (~2 seconds), then check alice's token balance:

```bash
./target/release/solen call faucet <TOKEN_ID> balance_of --args "${ALICE}"
```

## Step 11: Use the TypeScript SDK

You can also interact with the node programmatically:

```bash
cd sdks/wallet-sdk-ts
npm install
```

Create a script `demo.ts`:

```typescript
import { SolenClient, nameToHex } from "./src/index";

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

async function main() {
  const status = await client.chainStatus();
  console.log(`Chain height: ${status.height}`);

  const balance = await client.getBalance(nameToHex("faucet"));
  console.log(`Faucet balance: ${balance}`);

  const block = await client.getLatestBlock();
  console.log(`Block #${block.height}: ${block.tx_count} txs, ${block.gas_used} gas`);

  const alice = await client.getAccount(nameToHex("alice"));
  console.log(`Alice nonce: ${alice.nonce}, balance: ${alice.balance}`);
}

main().catch(console.error);
```

Run it:

```bash
npx tsx demo.ts
```

## Step 12: Clean Up

Stop the node with ++ctrl+c++. To reset state:

```bash
rm -rf data/solen-db
```

## What You Built

1. **Started a local blockchain** — single-validator devnet with 2-second blocks
2. **Managed keys** — imported a pre-funded faucet key, generated a new key
3. **Deployed a smart contract** — compiled Rust to WASM and deployed on-chain
4. **Called contract methods** — initialized, minted, transferred tokens
5. **Queried state** — checked balances via CLI and TypeScript SDK

## Next Steps

- [Write your own contract](../smart-contracts/contract-sdk.md) — Start from the counter example
- [TypeScript SDK](../tools/typescript-sdk.md) — Connect a web app
- [JSON-RPC API](../api/json-rpc.md) — Explore all RPC methods
- [Running multiple nodes](../tools/running-a-node.md) — Set up a multi-node network
