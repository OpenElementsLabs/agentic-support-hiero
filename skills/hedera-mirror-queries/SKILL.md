---
name: hedera-mirror-queries
license: Apache-2.0
metadata:
  source: https://github.com/hiero-hackers/agentic-support-hiero
  author: Hiero Hackers
description: >-
  How to query Hedera network data — balances, transactions, tokens,
  NFTs, HCS topic messages, smart-contract results, staking, nodes,
  fees — via the free mirror node REST API or the typed
  @hiero-enterprise/mirror TypeScript package. Use this skill whenever
  the user wants to read, look up, analyze, or monitor anything that
  happened on Hedera (mainnet or testnet), including account history,
  token holders, NFT ownership, topic messages, contract logs,
  exchange rates, network supply, and node status. Also use it when
  choosing between raw REST calls and the hiero-enterprise-js
  repositories, or when a query needs pagination, timestamp
  filtering, or historical (point-in-time) data.
---

# Querying Hedera Data

Every consensus result on Hedera — every transfer, token mint, topic
message, contract call — is ingested by **mirror nodes** and exposed
through a free public REST API. No API key, no account, no node to
run. This skill covers what data exists and how to get it.

## Choosing your route

Both routes hit the same API and return the same data — the choice is
about who handles the operational details (retries, rate limits,
pagination, response drift), not about what you can access.

**Use the REST API directly** (`curl`, `fetch`, any HTTP client) when:

- You need an answer *now*: one-off lookups, exploration, debugging,
  shell scripts, CI checks. A URL in the browser works.
- The project isn't TypeScript/JavaScript — the REST API is the only
  language-neutral route.
- You're building agent tooling that composes URLs dynamically.

**Use `@hiero-enterprise/mirror`** (TypeScript) when:

- The queries live in a **production service** — the client bakes in
  what you'd otherwise hand-roll and get wrong once: timeouts,
  retries on 429/5xx, a concurrency gate, and an optional
  requests-per-second ceiling shared across all queries.
- You're making **many queries or paginating deeply** — `Page.next()`
  / `paginate()` handle cursor-following, and the shared gate keeps a
  burst of parallel lookups from tripping the public rate limit.
- You want **types and runtime validation**: camelCase typed
  responses, and upstream field drift surfaces as a typed
  `MirrorError` instead of `undefined` three functions later. (The
  types also encode observed reality where the upstream spec is
  wrong — e.g. fields that are nullable in practice.)
- You're already on the hiero-enterprise-js stack — the express/
  fastify/nest adapters expose every repository with no extra wiring.

The usual progression: explore with REST until the query is right,
then port it into the package for anything that runs unattended.
Everything in this skill (filters, timestamps, pagination, gotchas)
applies identically to both.

**Neither** — use the Hedera hosted MCP server (see the `hedera-mcp`
skill) when an agent needs tool-calling against Testnet, including
building transactions, not just reads.

Base URLs (same API on each):

```
https://mainnet.mirrornode.hedera.com
https://testnet.mirrornode.hedera.com
https://previewnet.mirrornode.hedera.com
```

## What data exists — the map

Think in domains. Every domain below is available over REST and as a
repository in the TS package:

| You want… | REST (under `/api/v1`) | TS repository |
| --- | --- | --- |
| Account info, balance, keys, staking | `/accounts/{id}`, `/balances` | `accountRepository` |
| Account's transaction history | `/transactions?account.id=` | `transactionRepository.findByAccount` |
| Any transaction by id/type/timestamp | `/transactions`, `/transactions/{id}` | `transactionRepository` |
| Fungible tokens: info, holders, supply | `/tokens`, `/tokens/{id}`, `/tokens/{id}/balances` | `tokenRepository` |
| NFTs: ownership, serials, history | `/tokens/{id}/nfts`, `/accounts/{id}/nfts` | `nftRepository` |
| HCS topic messages (ordered, immutable) | `/topics/{id}/messages` | `topicRepository` |
| Smart contracts: results, logs, state, simulate calls | `/contracts/**`, `POST /contracts/call` | `contractRepository` |
| Scheduled transactions | `/schedules` | `scheduleRepository` |
| Blocks (record-file windows) | `/blocks` | `blockRepository` |
| Network: nodes, supply, fees, exchange rate, staking | `/network/**` | `networkRepository` |
| Staking rewards paid to an account | `/accounts/{id}/rewards` | `accountRepository.findRewards` |

For the exact contract — every endpoint's parameters, response
schemas, and enums — read the bundled OpenAPI spec
[references/openapi.yml](references/openapi.yml) (Mirror Node REST API). It's the authoritative, machine-readable source; it's large,
so grep it for the `operationId`, path, or schema name you need rather
than reading it whole. [references/rest-api.md](references/rest-api.md)
is the curated map of the same endpoints with the reality-vs-spec
gotchas the raw spec omits — read it for orientation and the OpenAPI
spec for precision. For the complete package API (config, all
repository methods, pagination, errors), read
[references/enterprise-js.md](references/enterprise-js.md).

## Quickstart — REST

```bash
# Account balance + info
curl -s "https://mainnet.mirrornode.hedera.com/api/v1/accounts/0.0.2?transactions=false"

# Last 5 transfers touching an account
curl -s "https://mainnet.mirrornode.hedera.com/api/v1/transactions?account.id=0.0.2&limit=5&order=desc"

# HCS messages on a topic
curl -s "https://mainnet.mirrornode.hedera.com/api/v1/topics/0.0.1234/messages?limit=10"
```

## Quickstart — @hiero-enterprise/mirror

Not yet on npm — install from the repo (workspace clone or git
dependency of `hiero-hackers/hiero-enterprise-js`, package
`packages/mirror`).

```ts
import {
    createMirrorNodeClient,
    createMirrorRepositories,
    paginate,
} from "@hiero-enterprise/mirror";

const client = createMirrorNodeClient({ network: "mainnet" });
const repos = createMirrorRepositories(client);

const account = await repos.accountRepository.findByAccountId("0.0.2");
const page = await repos.transactionRepository.findByAccount("0.0.2", {
    limit: 25,
    order: "desc",
});
for (const tx of page.data) console.log(tx.transactionId, tx.result);
```

The client enforces sane defaults you'd otherwise hand-roll: 10 s
timeout, 3 retries on 429/5xx, max 25 concurrent requests. Configure
via `MirrorConfig` (`network` or explicit `mirrorNodeUrl`, plus
timeout/retry/concurrency/rate fields) or `mirrorConfigFromEnv()`.

## Core patterns (both routes)

**Entity IDs.** Everything is `shard.realm.num` (`0.0.x`). Accounts
can also be addressed by EVM address (`0x…`) or alias on the accounts
endpoints; contracts by EVM address.

**Timestamps.** Consensus timestamps are `seconds.nanoseconds` strings
(`"1783641600.000000000"`). Filter params accept comparison prefixes:
`timestamp=gte:1783555200&timestamp=lt:1783641600`. Two operators on
one param = a range.

**Point-in-time (historical) queries.** Many endpoints accept
`timestamp=` to answer *as of that moment*: `/balances?timestamp=` is
the balance snapshot then, `/accounts/{id}?timestamp=` the entity
state then, `/network/supply?timestamp=` the supply then. This is how
you reconstruct history without an archive node.

**Pagination.** List responses return `links.next` (a ready-made URL,
`null` on the last page). Follow it verbatim — don't build offset
math. Default page size 25, max 100 (`limit=`). In the TS package
every list returns `Page<T>`: `page.data`, `page.links`, and
`page.next()` → the next `Page` or `null`; or stream items with
`for await (const item of paginate(...))`.

**Ordering.** `order=asc|desc` (`desc` for "latest first").

**Result filtering.** `result=success|fail` on transactions;
`transactiontype=CRYPTOTRANSFER|CONSENSUSSUBMITMESSAGE|…` narrows by
type server-side — always prefer this over filtering client-side.

## Unit and encoding gotchas

These cause most wrong answers — check them before reporting numbers:

- **HBAR amounts are in tinybars** (1 ℏ = 100,000,000 tinybar).
  Divide by 1e8. Token amounts are in the token's smallest unit —
  divide by 10^`decimals` from the token info.
- **Topic messages are base64-encoded** in `message`. Decode before
  showing content.
- **Contract data is hex** (`0x…`) — call data, logs, state values.
- Large numeric fields can exceed JS `Number.MAX_SAFE_INTEGER`
  (supplies, tinybar totals) — treat as strings/BigInt.
- Some fields are legitimately `null` (e.g. a node's
  `grpc_proxy_endpoint`, an account's `alias`) — handle before use.
- Mirror data is **eventually consistent**: expect a few seconds of
  lag behind consensus. For "did my transaction land", poll
  `/transactions/{id}` briefly rather than assuming absence = failure.

## Recipes that come up constantly

```bash
# Who holds token X, largest first
/api/v1/tokens/{id}/balances?order=desc&limit=100

# An account's HBAR balance exactly at a past moment
/api/v1/balances?account.id=0.0.2&timestamp=lt:1783641600

# All successful contract logs for a contract, newest first
/api/v1/contracts/{id}/results/logs?limit=100&order=desc

# Current HBAR/USD exchange rate (rate = cent_equivalent/hbar_equivalent cents per ℏ)
/api/v1/network/exchangerate

# Which consensus nodes were active yesterday: the end-of-day payout
# from 0.0.802 lists every node paid — an address-book node absent
# from the transfer list was inactive that day
/api/v1/transactions?transactiontype=CRYPTOTRANSFER&result=success&type=debit&account.id=0.0.802&limit=1&order=desc
```

## Rate limits and etiquette

Public mirror nodes throttle (~50 req/s per IP, subject to change;
429 = back off). For bulk work: use `limit=100`, server-side filters,
and the TS package's gate (`mirrorNodeMaxRequestsPerSecond`) — or run
your own mirror node. Do not hammer `/transactions` unfiltered.

## When the mirror node is the wrong tool

- **Submitting transactions** — mirror nodes are read-only. Use an
  SDK (see `build-hiero-sdk`) or the hosted MCP server (`hedera-mcp`).
- **Trustless verification** — mirror responses are a database's
  word. Cryptographic proof requires the signed record/block streams.
- **Full-history bulk analytics** — page-by-page REST scraping of
  years of history is slow and unkind; the stream files in the public
  buckets are the firehose for that.
