# The Mirror Node Layer

Consensus nodes hold current state and accept transactions; mirror nodes hold
history and serve streams. Every SDK needs a second, distinct network layer for
mirrors — with three different channel types.

## Contents
1. The three mirror channels
2. Topic subscriptions: the resumable stream
3. Chunked message reassembly
4. Lazy entity population from mirror REST
5. What must come from where (consensus vs mirror)
6. Design lessons

## 1. The three mirror channels

1. **Mirror gRPC streaming** — `ConsensusService.subscribeTopic` (server-streaming):
   live + historical HCS topic messages. The only long-lived stream in the SDK.
2. **Mirror REST API** (HTTPS/JSON, paginated with `?limit=&order=` and
   `links.next`) — endpoints SDKs actually use:
   - `/api/v1/network/nodes` — address-book bootstrap for the consensus pool
   - `/api/v1/accounts/{idOrEvmAddress}` — resolve EVM address ↔ account num
   - `/api/v1/contracts/{evmAddress}` — resolve contract num
   - `/api/v1/contracts/call` — eth_call-style reads / gas estimation (JS)
3. **AddressBookQuery over mirror gRPC** (`NetworkService.getNodes`, JS) — the
   streaming alternative to the REST nodes endpoint. Either is fine; pick one and
   schedule periodic refresh (JS refreshes every 24h).

Port quirks for local networks: REST on 5551, contract-call on 8545, mirror gRPC on
5600. Centralize base-URL derivation in one helper.

## 2. Topic subscriptions: the resumable stream

The load-bearing mechanic is **resubscribe-from-last-timestamp**: on every received
message, record its consensus timestamp; when the stream drops and reconnects, the
rebuilt request sets `consensusStartTime = lastTimestamp + 1ns` and decrements
`limit` by messages already delivered. No duplicates, no gaps — without server-side
session state.

- Handlers: onMessage, onError (also receives listener exceptions), onComplete
  (clean stream end). Expose a cancellable SubscriptionHandle.
- Stream retry set mirrors the unary one: NOT_FOUND (topic not yet mirrored),
  UNAVAILABLE, RESOURCE_EXHAUSTED, INTERNAL+RST_STREAM; plus non-gRPC transport
  errors. Exponential backoff capped ~8s.
- Known gap in existing SDKs (don't copy it): the attempt counter never resets on
  successful messages, so a long-lived stream that reconnects often can exhaust
  attempts. Reset attempts on progress.

## 3. Chunked message reassembly

Messages >~1KB arrive as chunks carrying `ConsensusMessageChunkInfo`
(initialTransactionID, number, total):

- Key pending chunks by the initial TransactionId.
- Emit only when `received == total`; sort by chunk number; concatenate bytes.
- Metadata convention: consensus timestamp / running hash / sequence number from the
  **last** chunk; transaction id from the **first**.
- Decide up front whether reassembly is always-on (JS) or opt-in (Python's
  `chunking_enabled`) — it changes the callback contract.

## 4. Lazy entity population from mirror REST

Accounts/contracts created from an EVM address have an unknown entity num until
resolved. Provide explicit populate methods
(`populate_account_num`, `populate_evm_address`) that hit the mirror REST accounts/
contracts endpoints. Two cautions:

- **Mirror ingestion lag**: mirror data trails consensus by seconds. Prefer
  poll-with-timeout over fixed sleeps (JS hardcodes a 3s delay — don't copy).
- Treat NOT_FOUND as "not yet indexed", not a hard failure, in these paths.

## 5. What must come from where

| Data | Source |
|---|---|
| Transaction submission, paid queries | consensus nodes |
| Live state (balances, info, receipts <~3min) | consensus nodes |
| Topic message history + subscriptions | mirror only |
| Expired receipts/records, historical transactions | mirror only |
| Address book / node discovery | mirror (REST or gRPC) |
| EVM address ↔ entity num resolution | mirror REST |
| Cheap contract reads / gas estimates | mirror REST (`/contracts/call`) |

Keep the pools separate on the client (`network` vs `mirror_network`) — different
endpoints, ports, TLS rules, and health characteristics.

## 6. Design lessons

1. A separate managed pool for mirror nodes, with its own health/rotation.
2. A first-class resumable-stream abstraction (data/status/error/cancel behind a
   channel interface) with timestamp-based resubscription.
3. A real REST client with pagination, timeouts, and typed error mapping — half the
   mirror surface is JSON over HTTPS, not gRPC.
4. Explicit, deterministic chunk reassembly keyed by initial transaction id.
5. Tolerate ingestion lag: retry/poll NOT_FOUND on recently-created entities.
