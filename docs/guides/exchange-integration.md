# Exchange Integration Guide

This guide covers everything an exchange needs to integrate SOLEN deposits, withdrawals, and balance tracking.

## Network Information

| | |
|---|---|
| **Chain** | Solen Mainnet |
| **Chain ID** | 1 |
| **Native Token** | SOLEN |
| **Decimals** | 8 (1 SOLEN = 100,000,000 base units) |
| **Block Time** | 6 seconds |
| **Finality** | Deterministic (single-slot BFT) |
| **Address Format** | Base58-encoded Ed25519 public key (32 bytes) |
| **Signature Algorithm** | Ed25519 |
| **RPC Endpoint** | `https://rpc.solenchain.io` |
| **Explorer API** | `https://api.solenchain.io` |
| **Explorer** | [solenscan.io](https://solenscan.io) |

!!! info "Finality"
    Solen uses BFT consensus with deterministic finality. Once a block is finalized (confirmed by 2/3+ stake-weighted attestations), it is irreversible. There are no reorgs or rollbacks. A single confirmation is sufficient.

## RPC Setup

All exchange operations use the JSON-RPC 2.0 API over HTTP POST. The public endpoint is `https://rpc.solenchain.io`. For high-volume operations, we recommend running your own node — see [Running a Node](../tools/running-a-node.md).

```bash
# Test connectivity
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

---

## 1. Check Network Status

### Get Chain Status

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

**Response:**

```json
{
  "height": 5000,
  "state_root": "8f32465c...",
  "pending_ops": 0,
  "total_allocation": "200000000000000000",
  "total_staked": "1100000000000000",
  "total_circulation": "600001989000000",
  "config": {
    "block_time_ms": 6000,
    "min_validator_stake": "50000000000000",
    "unbonding_period_epochs": 7,
    "epoch_length": 100,
    "base_fee_per_gas": "1",
    "burn_rate_bps": 5000
  }
}
```

Use `height` to track chain progress and detect stalls. The node is healthy if height advances every ~6 seconds.

---

## 2. Get Block by Height

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getBlock","params":[5000],"id":1}'
```

**Response:**

```json
{
  "height": 5000,
  "epoch": 50,
  "parent_hash": "a1b2c3...",
  "state_root": "d4e5f6...",
  "transactions_root": "...",
  "receipts_root": "...",
  "proposer": "2PmB6h8i...",
  "timestamp_ms": 1745085600000,
  "tx_count": 3,
  "gas_used": 600
}
```

### Get Latest Block

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getLatestBlock","params":[],"id":1}'
```

---

## 3. Get Transaction by ID

Transactions are identified by `block_height` and `tx_index` within the block.

### Via Explorer API

```bash
# Get transaction at block 5000, index 0
curl -s https://api.solenchain.io/api/tx/5000/0
```

**Response:**

```json
{
  "block_height": 5000,
  "index": 0,
  "sender": "dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur",
  "nonce": 42,
  "success": true,
  "gas_used": 200,
  "error": null,
  "events": [
    {
      "topic": "transfer",
      "data": "..."
    }
  ]
}
```

### Get All Transactions in a Block

```bash
curl -s https://api.solenchain.io/api/blocks/5000/txs
```

### Get Transactions for an Account

```bash
curl -s https://api.solenchain.io/api/accounts/dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur/txs
```

---

## 4. Get Account Balance

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getBalance","params":["dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur"],"id":1}'
```

**Response:**

```json
{
  "result": "1500000000"
}
```

The balance is in **base units** (8 decimals). Divide by 100,000,000 to get SOLEN. So `1500000000` = 15.0 SOLEN.

### Get Full Account Info

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getAccount","params":["dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur"],"id":1}'
```

**Response:**

```json
{
  "id": "dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur",
  "balance": "1500000000",
  "nonce": 42,
  "code_hash": "0000000000000000000000000000000000000000000000000000000000000000",
  "staked": "0"
}
```

---

## 5. Generate a New Address

Solen uses Ed25519 keypairs. The **account address IS the public key**, Base58-encoded.

### Using Node.js

```typescript
import { ed25519 } from "@noble/curves/ed25519";
import { randomBytes } from "crypto";
import bs58 from "bs58";

// Generate a random 32-byte seed (private key)
const seed = randomBytes(32);

// Derive the public key
const publicKey = ed25519.getPublicKey(seed);

// The address is the Base58-encoded public key
const address = bs58.encode(Buffer.from(publicKey));

console.log("Address:", address);
console.log("Private Key (hex):", Buffer.from(seed).toString("hex"));
console.log("Public Key (hex):", Buffer.from(publicKey).toString("hex"));
```

### Using Python

```python
from nacl.signing import SigningKey
import base58

# Generate keypair
signing_key = SigningKey.generate()
public_key = signing_key.verify_key.encode()

address = base58.b58encode(public_key).decode()
private_key = signing_key.encode().hex()

print(f"Address: {address}")
print(f"Private Key: {private_key}")
```

!!! warning "Key Storage"
    The private key (seed) must be stored securely. It is the only way to sign transactions from this address. Today the seed directly generates the keypair (no derivation path); a BIP-39 / SLIP-0010 path under SLIP-0044 coin type `20260424` is planned once the [SLIP registration](https://github.com/satoshilabs/slips/pull/2010) merges. See [Transaction Signing → HD Derivation](../specs/transaction-signing.md#hd-derivation).

---

## 6. Validate an Address

A valid Solen address is a Base58-encoded 32-byte Ed25519 public key.

### Validation Rules

1. Must be valid Base58 (characters: `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`)
2. Must decode to exactly 32 bytes
3. No checksum — Base58 encoding only (not Base58Check)

### Node.js

```typescript
import bs58 from "bs58";

function isValidAddress(address: string): boolean {
  try {
    const decoded = bs58.decode(address);
    return decoded.length === 32;
  } catch {
    return false;
  }
}
```

### Python

```python
import base58

def is_valid_address(address: str) -> bool:
    try:
        decoded = base58.b58decode(address)
        return len(decoded) == 32
    except:
        return False
```

---

## 7. Fee Estimation

Solen has a simple fee model. Fees are deterministic based on the operation type:

| Operation | Typical Gas | Fee (base units) |
|-----------|------------|-----------------|
| Transfer | 200 | 200 |
| Contract Call | 200-10,000+ | varies |
| Contract Deploy | 1,000+ | varies |

The base fee is 1 base unit per gas unit. Fees are extremely low — a transfer costs 200 base units = 0.000002 SOLEN.

### Get Current Fee Parameters

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_chainStatus","params":[],"id":1}'
```

Check `config.base_fee_per_gas` in the response. The fee for a transfer is `base_fee_per_gas * 200`.

### Simulate a Transaction

To get an exact gas estimate before submitting:

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0","method":"solen_simulateOperation","id":1,
    "params":[{
      "sender": [sender_bytes],
      "nonce": 0,
      "actions": [{"Transfer": {"to": [recipient_bytes], "amount": 1000000000}}],
      "max_fee": 100000,
      "signature": []
    }]
  }'
```

**Response:**

```json
{
  "success": true,
  "gas_used": 200,
  "error": null
}
```

---

## 8. Send a Transaction (Withdrawal)

Sending SOLEN requires building a `UserOperation`, signing it with Ed25519, and submitting via RPC.

### Step-by-Step

#### 1. Get the sender's current nonce

```bash
curl -s -X POST https://rpc.solenchain.io \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"solen_getNextNonce","params":["SENDER_ADDRESS"],"id":1}'
```

#### 2. Build the operation

```json
{
  "sender": [32-byte public key as array],
  "nonce": <current_nonce>,
  "actions": [
    {
      "Transfer": {
        "to": [32-byte recipient public key as array],
        "amount": 1500000000
      }
    }
  ],
  "max_fee": 100000,
  "signature": [64-byte ed25519 signature as array]
}
```

#### 3. Build the signing message

The signing message is 96 bytes:

```
chain_id    [8 bytes, little-endian u64]  = 1 for mainnet
sender      [32 bytes]                    = public key
nonce       [8 bytes, little-endian u64]
max_fee     [16 bytes, little-endian u128]
actions_hash [32 bytes]                   = BLAKE3 hash of JSON-serialized actions
```

For the canonical byte-level spec (JSON canonicalization rules, multi-sig encoding, edge cases, test vectors) see [Transaction Signing](../specs/transaction-signing.md).

The `actions_hash` is computed by JSON-serializing the actions array in the Rust serde format and hashing with BLAKE3:

```typescript
// Actions in Rust serde format:
const rustActions = [{
  Transfer: {
    to: Array.from(recipientPublicKeyBytes),  // 32-byte array
    amount: 1500000000
  }
}];

const actionsJson = JSON.stringify(rustActions);
const actionsHash = blake3(new TextEncoder().encode(actionsJson));
```

#### 4. Sign with Ed25519

```typescript
import { ed25519 } from "@noble/curves/ed25519";

const signature = ed25519.sign(signingMessage, privateKeySeed);
```

#### 5. Submit the operation

You have two endpoints. **For exchange withdrawals, use `solen_submitOperationConfirm`** — it submits and waits for finalized block inclusion in a single round-trip, which is exactly what a withdrawal pipeline needs.

=== "Recommended: submit + wait (one round-trip)"

    ```bash
    curl -s -X POST https://rpc.solenchain.io \
      -H "Content-Type: application/json" \
      -d '{
        "jsonrpc":"2.0","method":"solen_submitOperationConfirm","id":1,
        "params":[
          {
            "sender": [<32 bytes>],
            "nonce": 42,
            "actions": [{"Transfer": {"to": [<32 bytes>], "amount": 1500000000}}],
            "max_fee": 100000,
            "signature": [<64 bytes>]
          },
          60
        ]
      }'
    ```

    **Response (confirmed + successful):**

    ```json
    {
      "accepted": true,
      "confirmed": true,
      "success": true,
      "block_height": 98421,
      "tx_hash": "ef56...",
      "sender": "2ZrMqiKGz6TUvJkyBKyNMf3Y7dMrJ5JqWSCCYGn1VWbp",
      "nonce": 42,
      "gas_used": 21000,
      "error": null
    }
    ```

=== "Fire-and-forget submit"

    ```bash
    curl -s -X POST https://rpc.solenchain.io \
      -H "Content-Type: application/json" \
      -d '{
        "jsonrpc":"2.0","method":"solen_submitOperation","id":1,
        "params":[{
          "sender": [<32 bytes>],
          "nonce": 42,
          "actions": [{"Transfer": {"to": [<32 bytes>], "amount": 1500000000}}],
          "max_fee": 100000,
          "signature": [<64 bytes>]
        }]
      }'
    ```

    **Response:**

    ```json
    { "accepted": true, "error": null }
    ```

    `accepted: true` means mempool received the op — **not** that it's on-chain yet. Use this only if you have a separate confirmation path (block scanner, `solen_subscribeTxConfirmation` over WebSocket, or a poll of `solen_getAccount`).

### Withdrawal outcome handling

`solen_submitOperationConfirm` returns four distinct outcomes that your withdrawal pipeline must handle differently:

| `accepted` | `confirmed` | `success` | What it means | Action |
|---|---|---|---|---|
| `false` | `false` | `false` | Mempool rejected (bad nonce, zero balance, duplicate, rate-limit). | Re-fetch nonce with `solen_getNextNonce`, fix the issue, resubmit. The withdrawal **did not happen**. |
| `true` | `false` | `false` | Submit ok, no inclusion within timeout. May still land later. | Treat as **pending**, never as failed. Poll `solen_getAccount` for the sender's nonce — if it advanced past the operation's nonce, the tx landed. Do NOT re-sign with the same nonce until you've confirmed the original is gone. |
| `true` | `true` | `false` | Included in a block but **execution reverted** (gas paid, nonce consumed, no funds moved). | Mark the withdrawal as **failed**, surface the `error` to the user, **do not retry with the same nonce** (it's already burned). Build a new op with the next nonce. |
| `true` | `true` | `true` | Confirmed and successful. | Mark withdrawal as **paid**. `block_height` and `tx_hash` are safe to record for audit/customer support. |

!!! warning "Reverted ≠ failed-to-submit"
    A reverted tx (`confirmed: true, success: false`) consumed the nonce. The user paid gas, no funds moved, and you cannot reuse that nonce. **Do not credit deposits or mark withdrawals as paid when `success: false`.**

!!! info "Why prefer the confirm endpoint over polling"
    Solen has deterministic single-slot finality (no reorgs). Once `confirmed: true, success: true` returns, the result is irreversible — there is no need for a confirmation count. One round-trip replaces the typical "submit → poll for inclusion → poll for confirmations" pattern.

### Idempotency and retries

If your withdrawal worker crashes between calls, retrying with the same `(sender, nonce, actions, signature)` is safe:

- If the original op already landed, `solen_submitOperationConfirm` detects this via the on-chain backfill check and returns `accepted: true, confirmed: true, success: true` with `block_height: 0` and an empty `tx_hash` (the backfill path can't reconstruct the hash without the receipt's block placement). De-dupe by `(sender, nonce)` instead, or query `solen_getAccount` + scan recent blocks if you need the exact placement.
- If the original was never submitted, the retry submits and waits normally.
- If a different op with the same nonce landed (e.g. an operator submitted from two workers), `validate_and_submit` rejects with `nonce too low` and `accepted: false`. Treat this as "nonce already used" and check `solen_getAccount` to see what actually happened.

### Complete Node.js Example

```typescript
import { ed25519 } from "@noble/curves/ed25519";
import { blake3 } from "@noble/hashes/blake3";
import bs58 from "bs58";

const CHAIN_ID = 1; // mainnet
const RPC = "https://rpc.solenchain.io";

async function sendTransfer(
  senderSeed: Uint8Array,    // 32-byte private key seed
  recipientAddress: string,  // Base58 address
  amount: bigint             // base units (1 SOLEN = 100000000n)
) {
  const senderPubKey = ed25519.getPublicKey(senderSeed);
  const recipientPubKey = bs58.decode(recipientAddress);
  const senderAddress = bs58.encode(Buffer.from(senderPubKey));

  // 1. Get nonce
  const nonceResp = await rpc("solen_getNextNonce", [senderAddress]);
  const nonce = nonceResp.result;

  // 2. Build actions (Rust serde format)
  const actions = [{
    Transfer: {
      to: Array.from(recipientPubKey),
      amount: Number(amount)
    }
  }];

  const maxFee = 100000;

  // 3. Build signing message (96 bytes)
  const msg = new Uint8Array(96);
  const view = new DataView(msg.buffer);
  view.setBigUint64(0, BigInt(CHAIN_ID), true);        // chain_id LE
  msg.set(senderPubKey, 8);                             // sender
  view.setBigUint64(40, BigInt(nonce), true);            // nonce LE
  view.setBigUint64(48, BigInt(maxFee), true);           // max_fee LE (lower 8 bytes)
  // upper 8 bytes of max_fee are 0

  const actionsJson = JSON.stringify(actions);
  const actionsHash = blake3(new TextEncoder().encode(actionsJson));
  msg.set(actionsHash.slice(0, 32), 64);                // actions_hash

  // 4. Sign
  const signature = ed25519.sign(msg, senderSeed);

  // 5. Submit and wait for finalized block inclusion (one round-trip).
  //    Returns once the tx lands or after `timeout_secs`.
  const op = {
    sender: Array.from(senderPubKey),
    nonce,
    actions,
    max_fee: maxFee,
    signature: Array.from(signature),
  };
  const { result } = await rpc("solen_submitOperationConfirm", [op, 60]);

  // Branch on the outcome — see the table above.
  if (!result.accepted) {
    throw new Error(`Mempool rejected: ${result.error}`);
  }
  if (!result.confirmed) {
    // Still pending — caller decides whether to wait longer or treat as pending.
    return { status: "pending", op, timeoutError: result.error };
  }
  if (!result.success) {
    // On-chain revert — nonce is consumed, do NOT retry with the same nonce.
    return { status: "reverted", txHash: result.tx_hash, error: result.error };
  }
  return {
    status: "confirmed",
    txHash: result.tx_hash,
    blockHeight: result.block_height,
    gasUsed: result.gas_used,
  };
}

async function rpc(method: string, params: any[]) {
  const resp = await fetch(RPC, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ jsonrpc: "2.0", id: 1, method, params }),
  });
  return resp.json();
}
```

---

## Deposit Detection

For exchanges managing many deposit addresses, the recommended approach is to scan each finalized block for transfer events matching your address set. This is more efficient than polling individual accounts and guarantees no deposits are missed.

### Recommended: Block Scanner (Many Addresses)

Scan every new block and check transfer events against your set of deposit addresses. This is the standard pattern used by exchanges.

```typescript
import bs58 from "bs58";

const RPC = "https://rpc.solenchain.io";
const API = "https://api.solenchain.io";

// Your set of deposit addresses (loaded from database)
const depositAddresses = new Set<string>([
  "dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur",
  "H9we5VhjE42uCXXgy1YTcFV5bZReBcqxznMNkryHz8KY",
  // ... thousands of addresses
]);

let lastProcessedHeight = 0; // persist this to disk/database

async function getChainHeight(): Promise<number> {
  const resp = await fetch(RPC, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0", id: 1,
      method: "solen_chainStatus", params: [],
    }),
  });
  const json = await resp.json();
  return json.result.height;
}

async function getBlockTxs(height: number): Promise<any[]> {
  const resp = await fetch(`${API}/api/blocks/${height}/txs`);
  if (!resp.ok) return [];
  return resp.json();
}

async function scanForDeposits() {
  const currentHeight = await getChainHeight();
  if (currentHeight <= lastProcessedHeight) return;

  for (let h = lastProcessedHeight + 1; h <= currentHeight; h++) {
    const txs = await getBlockTxs(h);

    for (const tx of txs) {
      if (!tx.success) continue;

      for (const event of (tx.events || [])) {
        // Transfer events: topic="transfer", data = recipient[32] + amount[16] (hex)
        if (event.topic === "transfer" && event.data && event.data.length >= 96) {
          const recipientHex = event.data.substring(0, 64);
          const recipientBytes = Buffer.from(recipientHex, "hex");
          const recipientAddress = bs58.encode(recipientBytes);

          if (depositAddresses.has(recipientAddress)) {
            // Parse amount (little-endian u128, bytes 32-48 = hex chars 64-96)
            const amountHex = event.data.substring(64, 96);
            const amount = parseLEu128(amountHex);

            console.log(`Deposit detected!`);
            console.log(`  Block: ${h}`);
            console.log(`  To: ${recipientAddress}`);
            console.log(`  From: ${tx.sender}`);
            console.log(`  Amount: ${Number(amount) / 1e8} SOLEN`);
            console.log(`  Tx: ${h}-${tx.index}`);

            // Credit the deposit in your database
            await creditDeposit({
              address: recipientAddress,
              amount: amount,
              blockHeight: h,
              txIndex: tx.index,
              sender: tx.sender,
            });
          }
        }
      }
    }

    lastProcessedHeight = h;
    // Persist lastProcessedHeight to database/disk
  }
}

function parseLEu128(hex: string): bigint {
  let val = 0n;
  for (let i = hex.length - 2; i >= 0; i -= 2) {
    val = (val << 8n) | BigInt(parseInt(hex.substring(i, i + 2), 16));
  }
  return val;
}

// Poll every 6 seconds (one block interval)
setInterval(scanForDeposits, 6000);
```

!!! tip "Performance"
    This approach scales to millions of deposit addresses since it only does a `Set.has()` lookup per transfer event. The bottleneck is the API call per block, not the address count. For high throughput, run your own node and query the explorer API locally on port 9955.

### Alternative: Poll Individual Accounts

For a small number of addresses, you can poll each account directly:

```bash
# Get recent transactions for a deposit address
curl -s https://api.solenchain.io/api/accounts/YOUR_DEPOSIT_ADDRESS/txs?limit=20
```

### Alternative: WebSocket Subscription

For real-time notification on a specific address:

```typescript
const ws = new WebSocket("wss://rpc.solenchain.io");

ws.onopen = () => {
  ws.send(JSON.stringify({
    jsonrpc: "2.0", id: 1,
    method: "solen_subscribeTxConfirmation",
    params: ["YOUR_DEPOSIT_ADDRESS"]
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.params?.result) {
    const tx = data.params.result;
    console.log(`Deposit confirmed at height ${tx.height}`);
  }
};
```

### Confirmation Requirement

Solen has **deterministic finality**. Once a block is finalized, it cannot be reverted. A single confirmation (the block being finalized) is sufficient for crediting deposits. There is no risk of reorgs or double-spends.

---

## Supply Endpoints

Plain-text endpoints for aggregators and monitoring:

| Endpoint | Description |
|----------|-------------|
| `https://api.solenchain.io/api/totalsupply` | Total supply as a number |
| `https://api.solenchain.io/api/circulatingsupply` | Circulating supply as a number |
| `https://api.solenchain.io/api/richlist?limit=100` | Top accounts by balance |

---

## wSOLEN on Base Chain

SOLEN is also available as an ERC-20 token (wSOLEN) on Base chain via the official bridge.

| | |
|---|---|
| **Token Contract** | `0x14C84e576EDDb3e24b3dA3659843b585285f9fD9` |
| **Bridge Contract** | `0x67c369a8FC8fd099158df035F1bE9A8cc29f66Ea` |
| **Decimals** | 8 |
| **Network** | Base (Chain ID 8453) |
| **BaseScan** | [View Token](https://basescan.org/token/0x14C84e576EDDb3e24b3dA3659843b585285f9fD9) |

---

## Support

For integration support, contact the Solen team:

- Telegram: [t.me/solenblockchain](https://t.me/solenblockchain)
- Discord: [discord.com/invite/QGxv8bebJc](https://discord.com/invite/QGxv8bebJc)
- GitHub: [github.com/Solen-Blockchain](https://github.com/Solen-Blockchain)
