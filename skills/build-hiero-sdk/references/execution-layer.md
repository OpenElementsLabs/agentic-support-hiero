# The Execution Layer: Executable, Transaction, Query, Network

## Contents
1. Executable: the template-method engine
2. Transaction specialization and lifecycle internals
3. Query specialization and the payment flow
4. The network stack (Client → Network → Node → Channel)
5. Error model
6. Essential vs incidental: what varies across languages
7. Porting pitfalls (the mistakes a naive port makes)

## 1. Executable: the template-method engine

One abstract class owns the entire execute loop; nothing domain-specific lives in it.
Generic over `(RequestT, ResponseT, OutputT)`.

**State it owns:** `max_attempts` (default 10), `min_backoff`/`max_backoff`
(250ms/8s), `grpc_deadline` (per attempt), `request_timeout` (whole loop),
`node_account_ids` (a lockable round-robin list), operator, logger.

**The loop, per attempt:**
1. Check whole-loop `request_timeout`.
2. Select a node: locked `node_account_ids` round-robin, else ask the Network for a
   healthy node. Skip unhealthy nodes (`node.is_healthy()` = readmit time passed).
3. Get the node's channel; apply the per-attempt gRPC deadline.
4. Build the request via the `_make_request` hook.
5. Call the one stub method the subclass names (`_get_method` / `_execute`).
6. On transport error: `_should_retry_exceptionally` — retry on
   UNAVAILABLE / DEADLINE_EXCEEDED / RESOURCE_EXHAUSTED / INTERNAL+RST_STREAM
   (both SDKs share the same RST_STREAM regex); increase the node's backoff.
7. On response: classify via `_should_retry` into an ExecutionState:
   - `FINISHED` → `_map_response` → return OutputT
   - `RETRY` → exponential delay `min(min_backoff * 2^attempt, max_backoff)`, next node
   - `EXPIRED` → regenerate transaction ID, retry (transactions only)
   - `ERROR` → throw `_map_status_error`'s typed error
8. On success, decrease the node's backoff (health recovery).

**Abstract hooks subclasses implement:** `_make_request`, `_get_method`/`_execute`,
`_should_retry`, `_map_response`, `_map_status_error`, `_get_transaction_id`,
`_get_log_id`, plus `to_bytes`. Concrete leaf classes implement only body-building
hooks — the loop is written once.

**Special cases baked into the loop:**
- `INVALID_NODE_ACCOUNT` → trigger a client network/address-book refresh.
- Receipt/record requests: back off **in place** on the same node instead of
  advancing — other nodes may not have the receipt yet; flapping breaks polling.
- Local-network mode (127.0.0.1 nodes): disable backoff, raise attempts.

## 2. Transaction internals

- **Freeze (`freeze_with(client)`)** — the pivotal step: resolve max fee
  (explicit → client default → per-type default), pull N node account IDs from the
  network (redundancy), generate `TransactionId.withValidStart(operator, now)`,
  then serialize one `SignedTransaction{bodyBytes, sigMap}` per node. Frozen-ness is
  detected by the presence of body bytes / signed transactions — not a boolean.
- **Signing:** `sign(key)` → dedup by public key (hex of raw key) → append a
  SignaturePair to every node variant's sigMap. `sign_with_operator(client)` freezes
  if needed. Expose canonical signable bytes for external/HSM signing. JS also has
  sign-on-demand (store the operator, re-sign per attempt); Python signs eagerly —
  pick ONE model; mixing them yields duplicate SignaturePairs.
- **SignatureMap shape:** nested map AccountId → TransactionId → PublicKey → bytes,
  reconstructed from the flattened per-node array. The flattened index is
  `txIndex * nodeCount + nodeIndex` (relevant only for chunked transactions).
- **Serialization:** `to_bytes()` emits a `TransactionList` of all node variants;
  `from_bytes()` decodes, dedups, reads the body's `oneof` case, and dispatches
  through a registry map (oneof tag → concrete class `_from_proto`). Every new
  transaction type MUST register; a missed registration breaks round-trips silently.
- **Response mapping:** `TransactionResponse{nodeId, transactionHash(sha384),
  transactionId}` with `get_receipt(client)` (throws ReceiptStatusError unless
  SUCCESS) and `get_record(client)` (fetches receipt first, then the paid record).
- **Chunking (FileAppend, TopicMessageSubmit):** payloads >~6KB split into ≤20 chunks;
  one transaction ID per chunk, offset by nanoseconds from the base validStart;
  chunks execute sequentially, each re-frozen and re-signed (signatures bind to body
  bytes, and each chunk's bytes differ). `execute` returns the first chunk's
  response; `execute_all` returns all. **Make chunking a shared base-class concern**
  (JS) — Python re-implemented it per class and the two copies drifted (different
  default chunk sizes, duplicated freeze/sign logic).

## 3. Query internals

- `_is_payment_required()` defaults true; receipt/version queries override to false.
- **Cost discovery:** `get_cost(client)` wraps the query in a CostQuery sharing its
  config, sends with `COST_ANSWER` and a zero payment, reads `header.cost`; callers
  add a ~10% buffer. A `QueryBase`/shared helper builds the signed CryptoTransfer
  payment so Query and CostQuery avoid a circular dependency.
- **Before execute:** resolve payment (explicit `query_payment` → `get_cost`,
  compared against `max_query_payment`, throwing a MaxQueryPaymentExceeded error),
  then build one signed payment transaction per candidate node.
- **Retry:** same retryable set as transactions minus the expired/regenerate branch.
  The receipt query additionally inspects the *inner* receipt status
  (`RECEIPT_NOT_FOUND`/`UNKNOWN` → keep polling) and honors a validate-status flag.

## 4. The network stack

Ownership: **Client → Network (+ MirrorNetwork) → Node → Channel**.

- **Client:** operator (payer id + key), all defaults (fees, backoffs, deadlines,
  max attempts), factory presets (`for_testnet`, `for_mainnet`, `from_env`), and a
  scheduled address-book refresh (JS: every 24h via mirror AddressBookQuery; Python:
  fetch at construction from mirror REST `/api/v1/network/nodes` — schedule it).
- **Network:** the healthy-node pool. Diffs old/new node sets on update (closing
  removed nodes' channels), random/round-robin selection over healthy nodes,
  `max_nodes_per_transaction` trimming, TLS toggle (ports 50211 plaintext / 50212 TLS).
- **Node:** one endpoint. Owns its lazily-created channel and its health state:
  exponential backoff (default 8s, doubling to 1h cap) with a readmit time;
  `is_healthy()` = readmit time passed; success halves the backoff.
- **Channel:** lazily instantiates one gRPC stub per service (crypto, token,
  consensus, file, contract, schedule, network, util, freeze, addressBook). The only
  platform-specific class: Node gRPC vs grpc-web vs native each provide the unary
  client; everything above is shared.
- **TLS reality:** consensus nodes are addressed by IP but their certs aren't issued
  for those IPs. SDKs pin the node certificate hash from the address book
  (SHA-384 of the PEM) and override the target name instead of standard hostname
  verification. A naive TLS setup either fails or silently disables verification.

## 5. Error model

| Error | Thrown when | Carries |
|---|---|---|
| PrecheckStatusError / PrecheckError | node rejects at admission | status, transactionId, nodeId |
| ReceiptStatusError | consensus receipt status ≠ SUCCESS | status, transactionId, full receipt |
| MaxQueryPaymentExceeded | query cost > configured max | cost, max |
| MaxAttemptsOrTimeoutError | loop exhausted attempts / request_timeout | attempts, last status |
| GrpcServiceError | transport failure | normalized grpc status |

Callers branch on `err.status` programmatically. The precheck/receipt split tells the
user *which lifecycle phase* failed — preserve it in any port.

## 6. Essential vs incidental

Essential (in every implementation): the five layers; the Executable hook contract and
ExecutionState classification; freeze/sign/execute lifecycle with per-node bodies; the
receipt-polling model; node health/backoff/readmit; cost pre-flight; oneof registry;
`_to_proto`/`_from_proto` privacy; proto codegen pinned to a HAPI version with one stub
per service behind a Channel.

Incidental (language choices): async-first (JS) vs sync (Python) — pick per language
idiom; platform client subclasses (JS browser/native) vs single client; external proto
package (JS) vs in-tree generated code with import rewriting (Python — the rewriting is
protoc-in-Python friction, copy the intent not the machinery); signer/wallet/provider
abstraction (HIP-338, JS-only so far); address book via gRPC AddressBookQuery vs mirror
REST.

## 7. Porting pitfalls

1. Serializing the body once instead of per node → breaks failover and
   INVALID_NODE_ACCOUNT recovery.
2. Signing once then looping chunks → invalid signatures; each chunk re-freezes and
   re-signs with stored keys.
3. Reusing one transaction ID across chunks or executing chunks concurrently → fails;
   IDs are sequential nanosecond offsets, submitted in order.
4. Guessing the retryable status set → copy it from a reference SDK verbatim.
5. Letting receipt polling round-robin nodes → flaps; pin the node, back off in place.
6. Conflating the per-attempt gRPC deadline with the whole-loop request timeout →
   premature or never-firing timeouts; deadline ≤ timeout, warn otherwise.
7. Standard TLS hostname verification against node IPs → fails; use cert-hash pinning
   from the address book.
8. Forgetting the COST_ANSWER pre-flight → paid queries fail with missing/insufficient
   payment.
9. Forgetting to register a new transaction type in the oneof→class registry →
   from_bytes silently returns the wrong thing.
10. Mixing eager signing with sign-on-demand → duplicate SignaturePairs; dedup by
    public key regardless.
