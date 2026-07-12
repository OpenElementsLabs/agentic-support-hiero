# agentic-support-hiero

A Claude Code **plugin** that focuses on [Hiero](https://hiero.org) and [Hedera](https://hedera.com). It bundles domain-knowledge skills and the Hedera documentation MCP server so any project working with Hiero/Hedera gets the same baseline of context and tooling.

## What's included

- **`skills/hedera-info`** — background knowledge about the Hedera public network: HBAR, network services (HTS, HCS, smart contracts, file service), the Hedera Council, and the surrounding ecosystem.
- **`skills/hiero-info`** — background knowledge about the Hiero open-source DLT project under LF Decentralized Trust: hashgraph consensus, architecture, SDKs, tooling, and governance.
- **`skills/hiero-solo`** — deploy and manage local or multi-node Hiero/Hedera test networks with the `solo` CLI (one-shot and manual workflows).
- **`skills/hedera-mcp`** — how to use the Hedera hosted MCP server: authentication, the `RETURN_BYTES` signing model, tool categories, and a typical Testnet workflow.
- **`skills/build-hiero-sdk`** — how to architect and build a Hedera/Hiero-style SDK in any language: the five-layer architecture, the HAPI protobuf envelope model, transaction/query lifecycles, retry and error semantics, and the coding/test conventions shared by the official SDKs. Distilled from the JS, Python, and Go SDKs and the HAPI protobuf definitions.
- **`.mcp.json`** — two MCP servers:
  - **`hedera-docs`** (`https://docs.hedera.com/mcp`) — search and retrieve official Hedera documentation.
  - **`hedera-testnet`** (`https://agentic-testnet-mcp.hedera.com/mcp`) — the Hedera hosted MCP server for building Testnet transactions and running read-only queries. It never signs or executes: transaction-building tools return unsigned hex-encoded bytes for you to sign locally. Set `HEDERA_ACCOUNT_ID` to your Testnet account ID (sent as the `x-hedera-account-id` header); see the `hedera-mcp` skill for details.

## Installation

Install from inside Claude Code:

```
/plugin marketplace add hiero-hackers/agentic-support-hiero
/plugin install agentic-support-hiero@hiero-hackers
```

Then restart Claude Code (or run `/reload-plugins`).

Skills are **namespaced** under the plugin, e.g. `/agentic-support-hiero:hiero-solo`, `/agentic-support-hiero:hedera-info`.

## Keeping up to date

Because the plugin is versioned, updates arrive when the `version` in `plugin.json` is bumped. Pull the latest release with:

```
/plugin marketplace update hiero-hackers
```

## Repository layout

```
agentic-support-hiero/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace catalog (this repo is its own marketplace)
├── skills/                  # all skills, flat (Claude Code discovers them here)
├── .mcp.json                # Hedera MCP servers (docs + hosted Testnet)
└── CHANGELOG.md
```

## For maintainers

- **Releasing:** bump `version` in **both** `.claude-plugin/plugin.json` and the `agentic-support-hiero` entry in `.claude-plugin/marketplace.json`, update `CHANGELOG.md`, and tag the release. Validate with `claude plugin validate ./ --strict`.

## License

This project is licensed under the Apache License 2.0 — see [LICENSE](LICENSE) for details.
