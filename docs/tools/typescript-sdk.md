# TypeScript SDK

The TypeScript SDK (`@solen/wallet-sdk`) provides a client for interacting with Solen nodes from JavaScript/TypeScript applications.

## Installation

```bash
cd sdks/wallet-sdk-ts
npm install
npm run build
```

## Quick Start

```typescript
import { SolenClient, SmartAccount } from "@solen/wallet-sdk";

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

// Query chain status
const status = await client.chainStatus();
console.log(`Height: ${status.height}`);

// Query balance
const balance = await client.getBalance("616c69636500...");
```

## SolenClient

The main client class for JSON-RPC communication.

### Constructor

```typescript
const client = new SolenClient({
  rpcUrl: "http://127.0.0.1:9944",
});
```

### Methods

#### `chainStatus()`

Get the current chain status.

```typescript
const status = await client.chainStatus();
// { height: 42, state_root: "abc...", pending_ops: 0 }
```

#### `getBalance(accountId)`

Get an account's native token balance.

```typescript
const balance = await client.getBalance("616c69636500...");
// "10000"
```

#### `getAccount(accountId)`

Get full account information.

```typescript
const account = await client.getAccount("616c69636500...");
// { balance: "10000", nonce: 0, code_hash: null }
```

#### `getLatestBlock()`

Get the latest finalized block.

```typescript
const block = await client.getLatestBlock();
// { height: 42, tx_count: 3, gas_used: 1500 }
```

#### `submitOperation(operation)`

Submit a signed user operation.

```typescript
const result = await client.submitOperation(signedOp);
```

#### `simulateOperation(operation)`

Dry-run an operation without state changes.

```typescript
const sim = await client.simulateOperation(op);
// { success: true, gas_used: 500 }
```

## SmartAccount

Helper class for building and submitting operations.

```typescript
const alice = new SmartAccount("616c69636500...", client);

// Build a transfer operation
const op = await alice.buildTransfer("626f6200...", 500);

// Sign (with your Ed25519 key)
// op.signature = sign(op, privateKey);

// Submit
const result = await alice.submit(op);
```

## PasskeyAuth

WebAuthn/passkey authentication support for browser environments.

```typescript
import { PasskeyAuth } from "@solen/wallet-sdk";

const auth = new PasskeyAuth();

// Register a new passkey
const credential = await auth.register("my-wallet");

// Sign an operation with passkey
const signature = await auth.sign(operationBytes);
```

## Utilities

### `nameToHex(name)`

Convert a human-readable name to a 32-byte hex account ID (left-padded with zeros).

```typescript
import { nameToHex } from "@solen/wallet-sdk";

nameToHex("faucet");
// "6661756365740000000000000000000000000000000000000000000000000000"

nameToHex("alice");
// "616c696365000000000000000000000000000000000000000000000000000000"
```

## Example: Full Workflow

```typescript
import { SolenClient, SmartAccount, nameToHex } from "@solen/wallet-sdk";

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

async function main() {
  // Check chain status
  const status = await client.chainStatus();
  console.log(`Chain height: ${status.height}`);

  // Check balances
  const faucetBalance = await client.getBalance(nameToHex("faucet"));
  console.log(`Faucet: ${faucetBalance}`);

  const aliceBalance = await client.getBalance(nameToHex("alice"));
  console.log(`Alice: ${aliceBalance}`);

  // Get latest block
  const block = await client.getLatestBlock();
  console.log(`Block #${block.height}: ${block.tx_count} txs`);
}

main().catch(console.error);
```

Run with:

```bash
npx tsx demo.ts
```
