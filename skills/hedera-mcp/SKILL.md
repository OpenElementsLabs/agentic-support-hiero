---
name: hedera-mcp
license: Apache-2.0
metadata:
  source: https://github.com/hiero-hackers/agentic-support-hiero
  author: Hiero Hackers
description: How to use the Hedera hosted MCP server — the official agentic MCP endpoint that builds unsigned Hedera Testnet transactions (accounts, HTS tokens, HCS topics, smart contracts) and runs read-only queries. Use when the user wants to interact with the Hedera Testnet through MCP, build or submit Hedera transactions, or asks how to configure/authenticate the Hedera MCP server. This plugin ships the server as `hedera-testnet` in `.mcp.json`.
---

# Hedera Hosted MCP Server

The Hedera hosted MCP server is an official, Hashgraph-operated Model Context Protocol
endpoint that lets an agent interact with the **Hedera Testnet**. It builds transactions
and answers queries — it never holds or uses your private key.

- **Endpoint:** `https://agentic-testnet-mcp.hedera.com/mcp`
- **Transport:** Streamable HTTP (`type: "http"`)
- **Network:** Hedera **Testnet only**
- **Docs:** https://docs.hedera.com/solutions/ai/hosted-mcp-server

This plugin already registers the server in `.mcp.json` under the name `hedera-testnet`.

## How it works: RETURN_BYTES mode

The server operates exclusively in `RETURN_BYTES` mode. It does **not** sign or execute
transactions on-chain. Every transaction-building tool returns the constructed transaction
as **hex-encoded bytes**. Your client is responsible for signing those bytes with your
private key and submitting them to the network using a Hedera SDK.

- **Read-only queries** (balances, account info, token info, …) execute directly and
  return results.
- **State-changing operations** (transfers, token mints, topic messages, contract calls)
  return unsigned transaction bytes for you to sign and submit locally.

> **Never send private keys to the hosted server.** Only your account ID is required.
> The server constructs unsigned transactions; signing happens entirely on your side.

## Authentication

The server needs a single HTTP header to know which account to build transactions for:

| Header | Value | Example |
|--------|-------|---------|
| `x-hedera-account-id` | Your Hedera Testnet account ID | `0.0.12345` |

In this plugin's `.mcp.json` the header value is `${HEDERA_ACCOUNT_ID}`, so set that
environment variable to your Testnet account ID before starting Claude Code:

```bash
export HEDERA_ACCOUNT_ID=0.0.12345
```

Get a free Testnet account (with test HBAR) from the Hedera Developer Portal
(https://portal.hedera.com).

## Manual client configuration

If you register the server outside this plugin (e.g. another MCP client), use:

```json
{
  "mcpServers": {
    "hedera": {
      "url": "https://agentic-testnet-mcp.hedera.com/mcp",
      "headers": {
        "x-hedera-account-id": "0.0.YOUR_ACCOUNT_ID"
      }
    }
  }
}
```

Common config locations:

- **Claude Code (this plugin):** shipped in `.mcp.json` as `hedera-testnet`.
- **Cursor:** `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project).
- **Claude Desktop:** `claude_desktop_config.json` (OS-dependent path).

## Available tool categories

- **Accounts** — creation, transfers, balances, allowances
- **Tokens / HTS** — fungible and non-fungible token operations
- **Smart contracts / EVM** — deployment and interactions
- **Consensus / HCS** — topic creation and messaging
- **Transactions & queries** — miscellaneous read-only lookups

## Typical workflow

1. Ensure `HEDERA_ACCOUNT_ID` is set to your Testnet account ID.
2. Ask for a read-only query (e.g. "what's the HBAR balance of 0.0.12345?") — the server
   returns the answer directly.
3. Ask for a state-changing action (e.g. "transfer 5 HBAR to 0.0.6789") — the server
   returns hex-encoded, unsigned transaction bytes.
4. Sign those bytes locally with your private key using a Hedera SDK
   (see the `hiero-info` skill for the SDKs) and submit them to Testnet.
5. Verify the result on HashScan (https://hashscan.io, Testnet).

## Notes

- This server is Testnet-only. Do not expect Mainnet operations.
- For official documentation lookups, use the separate `hedera-docs` MCP server (also
  shipped in this plugin).
- For local development networks, see the `hiero-solo` skill.
