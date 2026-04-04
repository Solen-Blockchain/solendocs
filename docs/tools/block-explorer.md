# Block Explorer (SolenScan)

SolenScan is a web-based block explorer for the Solen blockchain, built with Next.js 15, React 19, and TypeScript.

**Live:** [solenscan.io](https://solenscan.io)
**Repository:** [solenscan](https://github.com/Solen-Blockchain/solenscan)

## Features

- **Dashboard** — Live chain statistics (block height, total blocks/transactions/events)
- **Block Explorer** — Browse and search blocks with filtering
- **Transaction Viewer** — Transaction details with status indicators
- **Event Browser** — Smart contract event viewing
- **Account Pages** — Balances, nonce, transaction history
- **Network Switcher** — Toggle between devnet, testnet, and mainnet
- **Search** — Find blocks, transactions, and accounts
- **Dark Mode** — Toggle between light and dark themes
- **Slash Events** — Validator slashing events displayed with penalty amounts and reason
- **Auto-refresh** — Data updates every 3-5 seconds

## Tech Stack

| Technology | Version |
|-----------|---------|
| Next.js | 15 |
| React | 19 |
| TypeScript | Latest |
| Tailwind CSS | 3 |

## Running Locally

```bash
cd solenscan
npm install
npm run dev
```

The explorer connects to the Solen node's [Explorer REST API](../api/explorer.md) (default port 29955 for devnet).

## Configuration

Set the API endpoint in `.env.local`:

```
NEXT_PUBLIC_API_URL=http://127.0.0.1:29955
```

## Data Sources

SolenScan reads from the Explorer REST API endpoints:

| Endpoint | Used For |
|----------|----------|
| `GET /api/status` | Dashboard statistics |
| `GET /api/blocks` | Block list and details |
| `GET /api/accounts/{id}/txs` | Account transaction history |
| `GET /api/events` | Event browser |

See [Explorer REST API](../api/explorer.md) for full endpoint documentation.
