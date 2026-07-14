# Mirror node REST API — endpoint catalog

All paths under `/api/v1` on any mirror node base URL. Everything is
GET unless noted. Common params everywhere: `limit` (default 25, max
100), `order=asc|desc`, and comparison prefixes `eq:` `gt:` `gte:`
`lt:` `lte:` `ne:` on id/timestamp filters. List responses carry
`links.next` (follow verbatim; `null` = last page).

The authoritative contract is the OpenAPI spec, bundled alongside this
file as [openapi.yml](openapi.yml) (Mirror Node REST API) — grep it
for an `operationId`, path, or schema for exact parameters and
response shapes. The live version is served at
`https://testnet.mirrornode.hedera.com/api/v1/docs/openapi.yml`. This
catalog is the curated map; where reality drifts from the spec, the
gotchas at the end are the source of truth.

## Accounts & balances

| Endpoint | Notes |
| --- | --- |
| `/accounts` | Filter: `account.id`, `account.balance`, `account.publickey`. |
| `/accounts/{idOrAliasOrEvmAddress}` | Add `transactions=false` to skip the embedded recent-transactions block. `timestamp=` for historical entity state. |
| `/accounts/{id}/rewards` | Staking rewards paid to the account. |
| `/accounts/{id}/nfts` | NFTs owned. |
| `/accounts/{id}/tokens` | Token relationships (balance, kyc, freeze). |
| `/accounts/{id}/allowances/crypto` · `/tokens` · `/nfts` | Spending allowances. |
| `/accounts/{id}/airdrops/pending` · `/outstanding` | Unclaimed airdrops (receiver / sender view). |
| `/balances` | Balance snapshot; `account.id`, `account.balance`, `timestamp=` for **historic snapshots**. Response `timestamp` field = the snapshot moment. |

## Transactions

| Endpoint | Notes |
| --- | --- |
| `/transactions` | Filters: `account.id`, `transactiontype` (e.g. `CRYPTOTRANSFER`, `CONSENSUSSUBMITMESSAGE`, `CONTRACTCALL`, `TOKENMINT`, `ETHEREUMTRANSACTION`), `result=success|fail`, `timestamp`, `type=debit|credit` (with `account.id`). |
| `/transactions/{id}` | Id format `0.0.X-seconds-nanos`; returns the whole family (scheduled, child transactions). |
| `/transactions/{id}/stateproof` | Record-stream proof material for one transaction. |

## Tokens & NFTs

| Endpoint | Notes |
| --- | --- |
| `/tokens` | Filter: `token.id`, `account.id`, `type=FUNGIBLE_COMMON|NON_FUNGIBLE_UNIQUE`, `name`. |
| `/tokens/{id}` | Decimals, supply, keys, custom fees. `timestamp=` for historical. |
| `/tokens/{id}/balances` | Holders; `order=desc` → largest holders first. Snapshot semantics like `/balances`. |
| `/tokens/{id}/nfts` | Serials; filter `account.id` for one owner. |
| `/tokens/{id}/nfts/{serial}` | One NFT (owner, metadata — base64). |
| `/tokens/{id}/nfts/{serial}/transactions` | Provenance trail. |

## Consensus (HCS)

| Endpoint | Notes |
| --- | --- |
| `/topics/{id}` | Topic info (admin/submit keys, running hash). |
| `/topics/{id}/messages` | Ordered messages; `sequencenumber` filter; `message` field is **base64**. |
| `/topics/{id}/messages/{sequence}` | One message. |
| `/topics/messages/{consensusTimestamp}` | Look up a message by its exact timestamp. |

## Smart contracts

| Endpoint | Notes |
| --- | --- |
| `/contracts` · `/contracts/{idOrAddress}` | Bytecode included with `?bytecode=true`-style params on detail. |
| `/contracts/{idOrAddress}/results` | Executions against one contract. |
| `/contracts/results` | All executions; filter `from`, `block.number`, `timestamp`. |
| `/contracts/results/{transactionIdOrHash}` | One execution (accepts EVM tx hash). |
| `/contracts/{idOrAddress}/results/{timestamp}` | Execution at a timestamp. |
| `/contracts/results/{txIdOrHash}/actions` · `/opcodes` | Call traces / opcode trace. |
| `/contracts/{idOrAddress}/state` | Storage slots (`slot=` filter). |
| `/contracts/{idOrAddress}/results/logs` · `/contracts/results/logs` | Event logs; filter `topic0..topic3`, `timestamp`. |
| `POST /contracts/call` | eth_call-style simulation: `{to, data, estimate?, block?, …}` — read contract state without a transaction. |

## Schedules, blocks, network

| Endpoint | Notes |
| --- | --- |
| `/schedules` · `/schedules/{id}` | Scheduled transactions and their signatures. |
| `/blocks` · `/blocks/{hashOrNumber}` | Record-file windows (~2 s each): hash, tx count, gas. |
| `/network/nodes` | Address book: node ids/accounts, service endpoints, public keys, per-node stake, staking period. `node.id`/`file.id` filters. |
| `/network/stake` | Network-wide staking totals and reward rate. |
| `/network/supply` | Released/total supply (tinybars, as strings). `timestamp=` for historical. |
| `/network/exchangerate` | Current+next HBAR/cents rate; `timestamp=` historical. |
| `/network/fees` | Gas prices per transaction class; `POST /network/fees` estimates a specific transaction's fee. |

## Reality-vs-spec gotchas (observed on mainnet)

- `network/nodes[].grpc_proxy_endpoint` is `null` for nodes without a
  web proxy although the OpenAPI spec marks it non-nullable required.
- The address book lists nodes regardless of liveness — a node can be
  inactive for months and still appear. The daily activity signal is
  the end-of-day 0.0.802 payout transfer list (see SKILL.md recipe).
- Balance/holder endpoints are periodic **snapshots**, not live reads
  — check the response's `timestamp` field for staleness before
  presenting figures as current.
- `/transactions/{id}` for a scheduled or contract-initiated
  transaction returns multiple entries (parent + children) — filter
  by `nonce`/`scheduled` when you need exactly one.
- Amounts: tinybars everywhere HBAR is involved (÷1e8); token amounts
  in smallest units (÷10^decimals); supplies exceed JS safe integers.
- Timestamp filters compare against **consensus** timestamps; ranges
  need two params (`timestamp=gte:X&timestamp=lt:Y`), not one.
