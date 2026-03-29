# Rust SDK

The Rust wallet SDK (`solen-wallet-sdk`) provides programmatic access to Solen nodes from Rust applications.

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
solen-wallet-sdk = { path = "path/to/solen/sdks/wallet-sdk-rs" }
```

## Usage

```rust
use solen_wallet_sdk::{SolenClient, SmartAccount};

#[tokio::main]
async fn main() {
    let client = SolenClient::new("http://127.0.0.1:9944");

    // Query chain status
    let status = client.chain_status().await.unwrap();
    println!("Height: {}", status.height);

    // Query balance
    let balance = client.get_balance(&account_id).await.unwrap();
    println!("Balance: {}", balance);

    // Build and submit a transfer
    let account = SmartAccount::new(sender_id, &client);
    let op = account.build_transfer(&recipient_id, 500).await.unwrap();
    // Sign the operation...
    let result = account.submit(op).await.unwrap();
}
```

## API

The Rust SDK mirrors the [TypeScript SDK](typescript-sdk.md) API:

| Method | Description |
|--------|-------------|
| `chain_status()` | Get chain height, state root, pending ops |
| `get_balance(id)` | Get account balance |
| `get_account(id)` | Get full account info |
| `get_latest_block()` | Get latest finalized block |
| `submit_operation(op)` | Submit a signed operation |
| `simulate_operation(op)` | Dry-run without state changes |

## Key Management

```rust
use solen_wallet_sdk::Keypair;

// Generate a new keypair
let keypair = Keypair::generate();

// Import from seed
let keypair = Keypair::from_seed(&seed_bytes);

// Sign an operation
let signature = keypair.sign(&operation_bytes);
```
