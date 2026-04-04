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
const balance = await client.getBalance("8a88e3dd7409...");
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
const balance = await client.getBalance("8a88e3dd7409...");
// "10000"
```

#### `getAccount(accountId)`

Get full account information.

```typescript
const account = await client.getAccount("8a88e3dd7409...");
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
const alice = new SmartAccount("8a88e3dd7409...", client);

// Build a transfer operation
const op = await alice.buildTransfer("626f6200...", 500);

// Sign (with your Ed25519 key)
// The signing message includes chain_id to prevent cross-chain replay:
// msg = chain_id[8 LE] + sender[32] + nonce[8 LE] + max_fee[16 LE] + blake3(actions)[32]
// op.signature = ed25519_sign(privateKey, msg);

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

### Account Addresses

Account addresses are Base58-encoded Ed25519 public keys (~44 characters). Hex format (64 characters) is also accepted. Your address IS your public key — no derivation or hashing needed.

```typescript
// When you generate a key, the public key is your address
const keypair = generateKeypair();
const myAddress = keypair.publicKey; // this IS your account ID (Base58)
```

> **Note:** Both Base58 (Bitcoin alphabet) and hex formats are accepted as input in all SDK methods.

## Example: Full Workflow

```typescript
import { SolenClient, SmartAccount } from "@solen/wallet-sdk";

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:29944" });

// Account addresses are Ed25519 public keys.
// Both Base58 (~44 chars) and hex (64 chars) are accepted.
// Get these from `solen key list` or your wallet.
const FAUCET_ADDRESS = "2ZrMqiKGz6TUvJkyBKyNMf3Y7dMrJ5JqWSCCYGn1VWbp";
const ALICE_ADDRESS = "1vWn3XjQDrCkPqnHFSmGhxSjS3Z2CWFXwVzLHrpnZFN";

async function main() {
  // Check chain status
  const status = await client.chainStatus();
  console.log(`Chain height: ${status.height}`);

  // Check balances using public key addresses
  const faucetBalance = await client.getBalance(FAUCET_ADDRESS);
  console.log(`Faucet: ${faucetBalance}`);

  const aliceBalance = await client.getBalance(ALICE_ADDRESS);
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
