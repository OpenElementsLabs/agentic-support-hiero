# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-07-10

### Added

- `hedera-testnet` MCP server in `.mcp.json` — the Hedera hosted MCP server
  (`https://agentic-testnet-mcp.hedera.com/mcp`) for building Hedera Testnet transactions
  and running read-only queries. Authenticated with the `x-hedera-account-id` header
  (via the `HEDERA_ACCOUNT_ID` environment variable); operates in `RETURN_BYTES` mode so
  signing stays client-side.
- `hedera-mcp` skill — documents how to use the hosted MCP server: authentication, the
  `RETURN_BYTES` signing model, tool categories, and a typical Testnet workflow.

## [0.1.0] - 2026-07-05

### Added

- Initial release as a standalone Claude Code plugin, split out of `claude-base`.
- `hedera-info` skill — background knowledge about the Hedera public network.
- `hiero-info` skill — background knowledge about the Hiero open-source DLT project.
- `hiero-solo` skill — deploy and manage local/multi-node test networks with the `solo` CLI.
- `.mcp.json` — the Hedera documentation MCP server (`https://docs.hedera.com/mcp`).
- `.claude-plugin/plugin.json` — plugin manifest.
- `.claude-plugin/marketplace.json` — marketplace catalog (this repo is its own marketplace).
