---
hide:
  - navigation
---

<p align="center">
  <img src="assets/solenlogo.png" alt="Solen" width="300" />
</p>

# Solen Documentation

**A modular settlement network with native rollups, smart accounts, privacy primitives, and intent-aware execution.**

Solen narrows the responsibilities of the base layer and treats execution domains as first-class components. The design combines a minimal settlement chain, native rollups, smart accounts by default, privacy-capable proof interfaces, and intent-oriented transaction flow.

!!! success "Mainnet Live"
    Solen mainnet launched April 2026 with 11 validators, 6-second block times, and a cross-chain bridge to Base. RPC: `https://rpc.solenchain.io` | Explorer: [solenscan.io](https://solenscan.io)

---

## Quick Links

| | |
|---|---|
| **[Getting Started](getting-started/quick-start.md)** | Set up your environment and run your first node in minutes. |
| **[Build Your First dApp](getting-started/first-dapp.md)** | Deploy a token contract, mint, and transfer — in 15 minutes. |
| **[Architecture](architecture/overview.md)** | Understand how consensus, execution, rollups, and intents fit together. |
| **[API Reference](api/json-rpc.md)** | JSON-RPC methods, Explorer REST API, and CLI commands. |
| **[Smart Contracts](smart-contracts/overview.md)** | Write, deploy, and call WASM contracts on Solen. |
| **[Tokenomics](tokenomics/overview.md)** | Supply, staking, fees, and governance. |

---

## Project Stats

| Metric | Value |
|--------|-------|
| Rust crates | 17 |
| TypeScript packages | 2 |
| Rust source lines | ~21,000 |
| Test functions | 198 |
| Property tests | 4 (hundreds of cases each) |
| Fuzz targets | 3 |
| RPC methods | 20 |
| Explorer endpoints | 18 |
| Transfer TPS (live testnet) | ~1,200 |
| Contract call TPS (live testnet) | ~188 |
| Finality | < 1 second (single-slot BFT) |

---

## Ecosystem

| Project | Description |
|---------|-------------|
| [Solen](https://github.com/Solen-Blockchain/solen) | Core blockchain node, runtime, and tools |
| [SolenScan](https://github.com/Solen-Blockchain/solenscan) | Block explorer web application ([solenscan.io](https://solenscan.io)) |
| [Solen Wallet](https://github.com/Solen-Blockchain/solenwallet/releases) | Desktop wallet application (download from GitHub releases) |
| [Solen Website](https://solenchain.io) | Project landing page |

---

## License

Licensed under MIT OR Apache-2.0 at your option.
