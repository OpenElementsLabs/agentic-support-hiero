---
name: build-hiero-sdk
license: Apache-2.0
metadata:
  source: https://github.com/hiero-hackers/agentic-support-hiero
  author: Hiero Hackers
description: >-
  How to architect and build a Hiero-style blockchain SDK from scratch in any
  language — the layered architecture (value types → crypto → execution spine → network
  stack → service surface), the protobuf envelope model, transaction/query lifecycles,
  retry/error semantics, and the coding conventions shared across the official SDKs.
  Use this skill whenever the user is building, porting, extending, or reviewing an SDK
  for Hedera/Hiero (any language), adding a new transaction or query type, implementing
  a TCK server or handler, debugging signature/serialization/retry behavior in an SDK,
  or asks how the Hiero SDKs work internally — even if they don't say "SDK
  architecture" explicitly.
---

# Building a Hiero-style SDK

This skill distills the architecture shared by the official Hiero SDKs (JS, Python, Go,
Java) and the HAPI protobuf design they all serve. The architecture is essentially
identical across languages — same layers, same lifecycles, same conventions in each
language's casing — so this generalizes to building the SDK in any new language, and to
extending an existing one.

## The five layers (build them in this order)

```
5. SERVICE SURFACE      TokenCreateTransaction, AccountBalanceQuery, ... (~100 leaf classes)
4. EXECUTION SPINE      Executable → Transaction / Query   (retry loop, lifecycle)
3. NETWORK STACK        Client → Network → Node → Channel  (gRPC, health, TLS)
2. CRYPTO + ERRORS      PrivateKey/PublicKey, Status enum, typed errors
1. VALUE TYPES          AccountId, Hbar, Timestamp, TransactionId, Key tree
   ─────────────────────────────────────────────────────────────────────
0. GENERATED PROTO      pinned HAPI version → codegen → never leaked to users
```

Each layer depends only on the ones below it. Layer 5 classes are nearly mechanical
once 1–4 exist: each new transaction type is ~4 methods plus a registry entry.

**Starting from an empty repo?** Follow `references/walkthrough.md` — six verifiable
milestones from codegen to a confirmed transfer, with a check gating each step.

**Minimum viable SDK, in dependency order:** hex/sha384 helpers → value types
(each with `_to_proto`/`_from_proto`) → PrivateKey/PublicKey → Status + error types →
Executable → Channel (one concrete impl) → Node (health/backoff) → Network (selection)
→ Client (operator + defaults) → Transaction + SignatureMap + TransactionResponse →
Query + CostQuery → then three concrete leaves to be end-to-end usable:
`TransferTransaction`, `AccountBalanceQuery`, and `TransactionReceiptQuery`
(the receipt query is mandatory — nothing confirms success without it).

## The protobuf reality that shapes everything

Read `references/protobuf-model.md` before designing anything. The non-negotiables it
imposes:

1. **Sign bytes, not objects.** `TransactionBody` is serialized ONCE into
   `bodyBytes`; signatures are over those exact bytes. Protobuf serialization is not
   canonical across languages, so re-encoding a parsed body invalidates signatures.
   Freeze = serialize; after freeze, all setters must throw.
2. **One signed body per node.** `nodeAccountID` lives *inside* the signed bytes, so
   a transaction bound for N candidate nodes is N distinct `bodyBytes`, each signed.
   This is what makes node-failover retry possible.
3. **The client mints identity.** `TransactionID` = payer + client-chosen
   `validStart` timestamp. It drives dedup and the validity window (~120s default).
4. **Two-phase results.** Submitting returns only a precheck code (`OK` ≠ success).
   Real outcomes come from polling the free `TransactionGetReceipt` query until the
   status leaves `UNKNOWN`/`RECEIPT_NOT_FOUND`. Receipts expire in ~3 minutes;
   history lives on mirror nodes.
5. **Paid queries with a COST_ANSWER pre-flight.** Most queries embed a signed
   CryptoTransfer payment in their `QueryHeader`. To learn the price, send the same
   query with `responseType=COST_ANSWER` first (free), read `header.cost`, then
   resend `ANSWER_ONLY` with payment. Receipt queries are free — special-case them.
6. **One `ResponseCodeEnum` (~388 values), interpreted by phase.** `BUSY` and
   `DUPLICATE_TRANSACTION` are precheck-time; `SUCCESS` and entity errors are
   receipt-time. `OK` means "admitted", never "succeeded".

## The execution spine

The heart of every Hiero SDK is one abstract class (JS `Executable`, Python
`_Executable`) that owns the entire submit/retry loop, generic over
(request, response, output). Transaction and Query are its only two specializations.
Read `references/execution-layer.md` for the full contract; the shape:

- The loop per attempt: select a healthy node → build the request → gRPC call with a
  per-attempt deadline → classify the outcome as `FINISHED | RETRY | ERROR | EXPIRED`
  → map to output, backoff-and-retry, or throw a typed error.
- Subclasses implement ~8 abstract hooks (`_make_request`, `_get_method`,
  `_should_retry`, `_map_response`, `_map_status_error`, ...). Every concrete
  transaction/query only fills hooks; none touches the loop.
- Two distinct timers: per-attempt gRPC deadline vs whole-loop request timeout.
- Two distinct backoffs: per-request exponential (250ms→8s) vs per-node health
  backoff with readmission (8s→1h). Unhealthy nodes are skipped, not hammered.
- Retryable precheck codes are a fixed, protocol-specific set — copy them from a
  reference SDK, do not guess: `BUSY`, `PLATFORM_TRANSACTION_NOT_CREATED`,
  `PLATFORM_NOT_ACTIVE`, `INVALID_NODE_ACCOUNT` (transactions; the last also
  triggers an address-book refresh), plus transport-level
  `UNAVAILABLE`/`DEADLINE_EXCEEDED`/`RESOURCE_EXHAUSTED`/RST_STREAM.
  `TRANSACTION_EXPIRED` → regenerate the transaction ID and retry.
- Receipt/record polling is single-node and backs off in place — it must NOT
  round-robin, or polling flaps across nodes that haven't seen the receipt yet.

## Transaction lifecycle (the user-visible contract)

```
construct → set (fluent, validated) → freeze_with(client) → sign(key)* → execute(client)
          → TransactionResponse (precheck only) → get_receipt(client) → receipt.status
```

- Every setter: guard `_require_not_frozen()`, coerce convenience inputs
  (`"0.0.5"` → AccountId), validate cheap local invariants, `return self`.
- `freeze_with(client)` resolves defaults (fee, node list, transaction ID),
  serializes one body per node, and locks the object.
- Signatures are stored in a map keyed by body bytes / public-key prefix; signing
  the same key twice must dedup.
- `to_bytes`/`from_bytes` round-trip the full signed state (for offline signing and
  transport between processes). Deserialization dispatches on the body's protobuf
  `oneof` tag through a registry — every new transaction type registers itself.
- Chunked transactions (FileAppend, TopicMessageSubmit — payloads >~6KB) split into
  sequential chunks, one transaction ID per chunk (nanosecond-offset), each re-frozen
  and re-signed. Prefer a shared ChunkedTransaction base (JS-style) over per-class
  duplication (Python's early mistake).

## Adding one new transaction type (the recurring task)

1. Find its `TransactionBody.data` oneof arm and body message in the protos.
2. Create the class extending Transaction: private fields, fluent setters
   (guard + coerce + validate + return self), constructor delegating to setters.
3. Implement the ~4 hooks: build the body message, name the oneof case, pick the
   gRPC stub method, (optionally) parse receipt extras.
4. Register in the oneof→class deserialization registry.
5. Tests: proto-body construction assertions (including `HasField` checks that unset
   optionals stay unset), round-trip bytes test, mock-server execute test, one
   integration test, and a TCK handler if the spec defines one.

## Conventions (the part reviewers enforce)

Read `references/conventions.md` when writing any public API or tests. Headlines:

- **One API, three casings**: `setTokenId` / `set_token_id` / `SetTokenId` — design
  names once, apply each language's convention mechanically.
- **Underscore = internal**, everywhere: `_to_proto`/`_from_proto` conversions are
  private; generated proto types must never appear in public signatures.
- **Everything round-trips**: string↔object, bytes↔object, object↔proto are tested
  inverse pairs. This is what cross-SDK interop (and the TCK) stands on.
- **Unit tests use a real in-process gRPC server** with queued canned responses —
  not method mocks — so the serialization + retry path is actually exercised.
- **Integration tests**: shared env object, operator from env vars, mainnet refusal,
  teardown in finally.
- **TCK server** (`tck/` with param/handlers/response trio) is the cross-SDK
  conformance layer: same JSON-RPC driver runs against every SDK.
- **Deprecate in place**: keep old symbols warning-and-delegating; never remove
  within a major version.

## Reference files

| File | Read when |
|---|---|
| `references/protobuf-model.md` | Designing anything; understanding envelopes, services, receipts, keys, IDs |
| `references/execution-layer.md` | Building Executable/Transaction/Query/Network; debugging retry, signatures, timeouts; porting pitfalls |
| `references/conventions.md` | Writing public API, tests, examples, errors, releases |
| `references/language-ports.md` | Porting to Go/Rust/C++-like languages; choosing idioms for contracts, setters, errors, async |
| `references/mirror-layer.md` | Building topic subscriptions, address-book bootstrap, REST queries, chunk reassembly |
| `references/crypto-details.md` | Implementing keys/signing/mnemonics — exact DER prefixes, keccak+compact-r‖s signing, derivation paths |
| `references/walkthrough.md` | Starting from an empty repo — six verifiable milestones to a confirmed transfer |
| `references/local-testing.md` | Wiring tests to a local solo network, CI recipes, the TCK loop (deploying the network itself: `hiero-solo` skill) |
| `references/sources.md` | Links to the canonical protos, HIPs, all six SDK repos, the TCK, and the org's API guidelines — verify against these when in doubt |
