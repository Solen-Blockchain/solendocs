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

#### `getNextNonce(accountId)`

Get the next usable nonce, accounting for pending mempool transactions. Use this instead of `getAccount().nonce` when sending multiple transactions rapidly.

```typescript
const nonce = await client.getNextNonce("2ZrMqiKGz...");
// Returns: 5 (on-chain nonce + pending count)
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
// msg = chain_id[8 LE] + sender[32] + nonce[8 LE] + max_fee[16 LE] + blake3(actions)[32]
// op.signature = ed25519_sign(privateKey, msg);
// Full byte-level spec: ../specs/transaction-signing.md

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

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

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

## WebSocket Subscriptions

The RPC server supports WebSocket connections on the same port for real-time event streaming. Connect via `ws://` (or `wss://` for mainnet):

```typescript
// Subscribe to new blocks via WebSocket
const ws = new WebSocket("wss://rpc.solenchain.io");

ws.onopen = () => {
  // Subscribe to new finalized blocks
  ws.send(JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "solen_subscribeNewBlocks",
    params: []
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  // Subscription confirmation (returns subscription ID)
  if (msg.result && msg.id === 1) {
    console.log("Subscribed:", msg.result);
    return;
  }

  // Block notification
  if (msg.method === "solen_newBlock") {
    const block = msg.params.result;
    console.log(`Block #${block.height} | ${block.tx_count} txs | ${block.gas_used} gas`);
  }
};
```

### Available Subscriptions

| Method | Notification | Parameters | Description |
|--------|-------------|-----------|-------------|
| `solen_subscribeNewBlocks` | `solen_newBlock` | none | Every finalized block |
| `solen_subscribeTxConfirmation` | `solen_txConfirmation` | `sender`, `nonce` | Specific tx confirmation (auto-closes) |
| `solen_subscribeValidatorChanges` | `solen_validatorChange` | none | Validator set changes at epoch boundaries |

### Watch for Transaction Confirmation

```typescript
// After submitting a transaction, watch for confirmation
ws.send(JSON.stringify({
  jsonrpc: "2.0",
  id: 2,
  method: "solen_subscribeTxConfirmation",
  params: ["SENDER_ADDRESS_HEX", NONCE]
}));

// The notification arrives once when the tx lands in a block, then the subscription closes automatically.
```

## Building and Signing Operations

To submit transactions, you need to build a `UserOperation`, sign it, and submit:

```typescript
import { SmartAccount, SolenClient, nameToHex } from "@solen/wallet-sdk";

const client = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

// Create a smart account from a seed
const alice = SmartAccount.fromSeed("your-seed-hex-64-chars", client);

// Transfer native SOLEN
const op = await alice.buildTransfer(
  nameToHex("bob"),  // recipient (hex account ID)
  500_00000000       // 500 SOLEN in base units
);

// Submit to the network
const result = await client.submitOperation(op);
console.log("Accepted:", result.accepted);
```

### Simulate Before Submitting

```typescript
// Dry-run to check gas cost and success
const sim = await client.simulateOperation(op);
if (sim.success) {
  console.log(`Gas estimate: ${sim.gas_used}`);
  const result = await client.submitOperation(op);
} else {
  console.error("Simulation failed:", sim.error);
}
```

## Multi-Sig Operations

Solen supports threshold multi-sig at the account level. Set up a 2-of-3 multi-sig:

```bash
# Set up threshold auth (CLI)
solen multisig alice 2 \
  <PUBKEY_1_HEX> \
  <PUBKEY_2_HEX> \
  <PUBKEY_3_HEX>
```

To sign a multi-sig transaction, concatenate `pubkey[32] + signature[64]` pairs (96 bytes each). At least `threshold` valid signatures from the signers list must be present:

```typescript
// Build the operation
const op = await alice.buildTransfer(bobAddress, 1000_00000000);

// Sign with key 1
const sig1 = key1.sign(op.signingMessage());
// Sign with key 2
const sig2 = key2.sign(op.signingMessage());

// Concatenate: pubkey1[32] + sig1[64] + pubkey2[32] + sig2[64]
op.signature = Buffer.concat([
  key1.publicKeyBytes, sig1,
  key2.publicKeyBytes, sig2,
]);

await client.submitOperation(op);
```

Other auth methods available: `Ed25519`, `Passkey` (WebAuthn), `Session` (time-limited keys), `Guardian` (social recovery).

## Base Units

1 SOLEN = 100,000,000 base units (8 decimal places). All API amounts use base units.

```typescript
const SOLEN = 100_000_000n; // BigInt
const amount = 500n * SOLEN; // 500 SOLEN
```

## Testnet Configuration

```typescript
// Local devnet
const devClient = new SolenClient({ rpcUrl: "http://127.0.0.1:9944" });

// Public testnet
const testClient = new SolenClient({ rpcUrl: "https://testnet-rpc.solenchain.io" });
```

| Network | RPC URL | Chain ID |
|---------|---------|----------|
| Devnet (local) | `http://127.0.0.1:9944` | 1337 |
| Testnet | `https://testnet-rpc.solenchain.io` | 9000 |
