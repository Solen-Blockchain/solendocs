# Testnet Deployment

Deploy a multi-validator Solen testnet across multiple servers.

## Overview

The testnet deployment in `deploy/testnet/` sets up a 4-validator network:

- **Server 1** — Seed node + faucet
- **Servers 2-4** — Additional validators

## Files

| File | Purpose |
|------|---------|
| `genesis.json` | Initial state with validators and faucet |
| `solen-node.service` | systemd unit for the node |
| `solen-faucet.service` | systemd unit for the faucet HTTP service |
| `nginx.conf` | Reverse proxy for public endpoints |
| `setup.sh` | Deploy seed node (server 1) |
| `setup-validator.sh` | Deploy validators (servers 2-4) |

## DNS

| Network | RPC | API | Seed Node |
|---------|-----|-----|-----------|
| Mainnet | `rpc.solenchain.com` | `api.solenchain.com` | `seed1.solenchain.com` |
| Testnet | `testnet-rpc.solenchain.com` | `testnet-api.solenchain.com` | `testnet-seed1.solenchain.com` |

## Setup

### Server 1 (Seed Node)

```bash
# On server 1
cd deploy/testnet
./setup.sh
```

This will:

1. Build the node and faucet binaries
2. Initialize genesis state
3. Start the node and faucet as systemd services
4. Configure nginx as a reverse proxy

### Servers 2-4 (Validators)

```bash
# On each additional server
cd deploy/testnet
./setup-validator.sh --bootstrap /ip4/<SEED_IP>/tcp/40333
```

## Genesis Configuration

The `genesis.json` file defines:

- Initial validator set and their stakes
- Pre-funded accounts (faucet)
- Protocol parameters (block time, fees, etc.)

## Faucet

The faucet runs as an HTTP service for distributing testnet tokens:

```bash
# Request tokens
curl -X POST http://testnet-faucet.solenchain.com/drip \
  -H "Content-Type: application/json" \
  -d '{"account": "your-account-id-hex"}'
```

## Monitoring

Check node status on each server:

```bash
systemctl status solen-node
journalctl -u solen-node -f
```

Check the faucet:

```bash
systemctl status solen-faucet
```
