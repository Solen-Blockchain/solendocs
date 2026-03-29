# Solen Documentation

Documentation website for [Solen](https://github.com/solenchain/solen) — a modular settlement network with native rollups, smart accounts, and intent-aware execution.

Built with [MkDocs](https://www.mkdocs.org/) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Local Development

### Prerequisites

- Python 3.8+

### Install dependencies

```bash
pip install mkdocs mkdocs-material
```

### Preview

```bash
mkdocs serve
```

Opens at [http://127.0.0.1:8000](http://127.0.0.1:8000). Changes reload automatically.

### Build

```bash
mkdocs build
```

Static site output goes to `site/`.

## Structure

```
docs/
├── index.md                    # Home page
├── getting-started/            # Quick start, tutorial
├── architecture/               # Consensus, execution, accounts, rollups, intents
├── smart-contracts/            # Contract SDK, host functions, examples
├── api/                        # JSON-RPC, Explorer REST API, CLI
├── tools/                      # SDKs, node operation, explorer
├── tokenomics/                 # Supply, staking, governance, fees
└── specs/                      # Protocol specifications
```

## License

MIT OR Apache-2.0
