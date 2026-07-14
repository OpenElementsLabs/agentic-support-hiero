# @hiero-enterprise/mirror — package reference

Typed TypeScript client for the mirror node REST API, from
[hiero-hackers/hiero-enterprise-js](https://github.com/hiero-hackers/hiero-enterprise-js)
(`packages/mirror`). ESM, `sideEffects: false`, Apache-2.0. Not yet on
npm — consume it as a workspace package from a clone, or as a git
dependency.

## Setup

```ts
import {
    createMirrorNodeClient,
    createMirrorRepositories,
    mirrorConfigFromEnv,
    paginate,
} from "@hiero-enterprise/mirror";

const client = createMirrorNodeClient({ network: "mainnet" });
const repos = createMirrorRepositories(client);
```

`MirrorConfig` fields (all optional):

| Field | Default | Meaning |
| --- | --- | --- |
| `network` | — | `"mainnet"` / `"testnet"` / `"previewnet"` (also `hedera-*` aliases); resolves the URL |
| `mirrorNodeUrl` | — | Explicit base URL; wins over `network` |
| `mirrorNodeTimeoutMs` | 10000 | Per-request timeout |
| `mirrorNodeMaxRetries` | 3 | Retries on 429 / 5xx / timeout |
| `mirrorNodeMaxConcurrent` | 25 | Concurrency gate (`Infinity` disables) |
| `mirrorNodeMaxRequestsPerSecond` | unlimited | Sustained rate ceiling |

`mirrorConfigFromEnv()` builds the same config from environment
variables. All repositories share the one client, so the gate applies
across everything you do.

## Pagination model

Every list method returns `Page<T>`:

```ts
page.data        // T[] — the items
page.links       // raw links (persistable; next is a URL or null)
page.timestamp   // snapshot consensus timestamp on /balances-style endpoints
page.next()      // Promise<Page<T>> | null — fetch the next page
```

`page.next` is a function, so it does not survive JSON serialization —
persist `links.next` if you checkpoint. To stream across pages:

```ts
for await (const holder of paginate(() =>
    repos.tokenRepository.findHolders(tokenId, { limit: 100 }),
)) { /* … */ }
```

## Repositories and methods

Method names follow the convention: `findByX` (one entity),
`find*`/`list` (a `Page`), `get*` (a scalar/value object).

### accountRepository
- `findByAccountId(accountId, options?)` — account info (id, EVM
  address, or alias accepted)
- `findByAlias(alias)` — resolve an alias
- `getBalance(accountId, options?)` — balance (pass `timestamp` in
  options for point-in-time)
- `findTokens(accountId, options?)` — token relationships
- `findRewards(accountId, options?)` — staking rewards paid
- `list(options?)` — accounts (filter by balance, key, id range)
- `listBalances(options?)` — the `/balances` snapshot endpoint
- `findPendingAirdrops(accountId, options?)` / `findOutstandingAirdrops(...)`
- `findCryptoAllowances(...)` / `findTokenAllowances(...)` / `findNftAllowances(...)`
- `findHooks(accountId, options?)` / `findHookStorage(...)`

### transactionRepository
- `find(options?)` — all transactions; filter by `transactionType`,
  `result`, `timestamp`, `account.id`
- `findByAccount(accountId, options?)`
- `findById(transactionId)` — includes child/scheduled duplicates

### tokenRepository
- `findById(tokenId, options?)` — token info (supply, decimals, keys)
- `list(options?)` — filter by type, name, account
- `findByAccountId(accountId, options?)`
- `findHolders(tokenId, options?)` — `/tokens/{id}/balances`

### nftRepository
- `findByType(tokenId, options?)` — serials of a collection
- `findBySerial(tokenId, serialNumber)`
- `findByOwner(accountId, options?)` / `findByOwnerAndType(...)`
- `findTransactions(tokenId, serialNumber, options?)` — provenance

### topicRepository
- `findById(topicId)` — topic info
- `findByTopicId(topicId, options?)` — messages (a `Page`)
- `findByTopicIdAndSequenceNumber(topicId, seq)`
- `findMessageByTimestamp(timestamp)`

### contractRepository
- `list(options?)` / `findById(idOrEvmAddress, options?)`
- `findResults(idOrEvmAddress, options?)` / `listResults(options?)` —
  execution results
- `findResult(transactionIdOrHash)` / `findResultByTimestamp(id, timestamp)`
- `findActions(txIdOrHash, options?)` / `findOpcodes(txIdOrHash, options?)`
- `findState(idOrEvmAddress, options?)` — contract storage
- `findLogs(idOrEvmAddress, options?)` / `listLogs(options?)`
- `call(request)` — `POST /contracts/call`: simulate/eth_call without
  submitting a transaction

### networkRepository
- `findNodes(options?)` — address book + per-node stake (`nodeId`
  filter supports ranges)
- `findRegisteredNodes(options?)`
- `findNetworkSupplies(options?)` — total/released supply, historical
  via `timestamp`
- `findStakingRewards()` — network stake totals & reward rate
- `findExchangeRates(options?)` — HBAR/cent rate, historical via `timestamp`
- `findFees(options?)` / `estimateFees(request)` — fee schedule + POST estimation

### scheduleRepository
- `list(options?)` / `findById(scheduleId)`

### blockRepository
- `list(options?)` / `findByHashOrNumber(hashOrNumber)`

## Query options

Options interfaces mirror the REST params in camelCase: `limit`
(max 100), `order: "asc" | "desc"`, `timestamp` (string or a
`RangeFilter` for `gte:`/`lt:` semantics), plus per-endpoint filters
(`nodeId`, `fileId`, `transactionType`, `result`, `serialNumber`, …).
Range-style fields accept `{ gte, gt, lte, lt, eq }` objects.

## Errors

Failures throw `MirrorError` with a `code` from `MirrorErrorCodes`
(config errors, HTTP failures after retries, response-shape
violations). Responses are runtime-validated (`assert*Response`), so a
drifting upstream field surfaces as a typed error rather than
`undefined` downstream.

## Raw vs public types

Each domain has a raw wire type (snake_case, exactly what the REST API
returns) and a public camelCase type the repositories map to. When a
field seems missing, check whether you're reading the raw or the
mapped shape. Nullable-in-reality fields are typed `| null`
(e.g. `NetworkNode.grpcProxyEndpoint`) even where the upstream OpenAPI
spec disagrees — the types follow observed reality.

## Framework adapters

`@hiero-enterprise/express`, `/fastify`, and `/nest` in the same repo
expose these repositories as the read side of their integration —
construct once, inject/mount, and every repository is available
without wiring each one.
