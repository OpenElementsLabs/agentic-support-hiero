# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-07-11

### Added

- `build-hiero-sdk` skill — an architecture guide for building a Hedera/Hiero-style SDK
  in any language: the five-layer architecture (value types → crypto → execution spine →
  network stack → service surface), the HAPI protobuf envelope model and its design
  implications (frozen `bodyBytes` signing, per-node bodies, two-phase receipts, paid
  queries), the `Executable` retry engine, and the setter/factory/test/release
  conventions shared across the official SDKs. Ships with a set of reference
  documents distilled from the JS, Python, Go, Rust, and C++ SDKs and the
  consensus-node protobuf definitions: the protobuf envelope model, the execution
  layer, shared conventions, language-adaptation patterns (Go/Rust/C++ idiom
  translation), the mirror-node layer (topic streams, REST, address-book bootstrap),
  exact cryptographic constants (DER prefixes, keccak/compact-signature rules, HD
  derivation paths), a milestone walkthrough from empty repo to first confirmed
  transfer, and local-network test wiring (solo ports/env contract, CI, TCK loop —
  complementing the existing `hiero-solo` deployment skill).

## [0.2.0] - 2026-07-10

### Added

- `hedera-testnet` MCP server in `.mcp.json` — the Hedera hosted MCP server
  (`https://agentic-testnet-mcp.hedera.com/mcp`) for building Hedera Testnet transactions
  and running read-only queries. Authenticated with the `x-hedera-account-id` header
  (via the `HEDERA_ACCOUNT_ID` environment variable); operates in `RETURN_BYTES` mode so
  signing stays client-side.
- `hedera-mcp` skill — documents how to use the hosted MCP server: authentication, the
  `RETURN_BYTES` signing model, tool categories, and a typical Testnet workflow.

### Changed

- Moved the repository to the `hiero-hackers` GitHub organization and rebranded the plugin
  and marketplace away from Open Elements. Owner/author is now "Hiero Hackers" and all
  homepage/repository URLs point to `https://github.com/hiero-hackers/agentic-support-hiero`.
- **Breaking:** renamed the marketplace from `open-elements-hiero` to `hiero-hackers`.
  Anyone who added the old marketplace must re-add it and reinstall:

  ```
  /plugin marketplace add hiero-hackers/agentic-support-hiero
  /plugin install agentic-support-hiero@hiero-hackers
  ```

## [0.1.0] - 2026-07-05

### Added

- Initial release as a standalone Claude Code plugin, split out of `claude-base`.
- `hedera-info` skill — background knowledge about the Hedera public network.
- `hiero-info` skill — background knowledge about the Hiero open-source DLT project.
- `hiero-solo` skill — deploy and manage local/multi-node test networks with the `solo` CLI.
- `.mcp.json` — the Hedera documentation MCP server (`https://docs.hedera.com/mcp`).
- `.claude-plugin/plugin.json` — plugin manifest.
- `.claude-plugin/marketplace.json` — marketplace catalog (this repo is its own marketplace).
