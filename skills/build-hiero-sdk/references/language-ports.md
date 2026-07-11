# Language Adaptation: How Go, Rust, and C++ Express the Same Architecture

The architecture (see SKILL.md) is constant; each language re-expresses the contracts
with its own tools. Use the closest analogue when porting to a new language.

## Contents
1. Go: interfaces + embedding + a function-pointer Method struct
2. Rust: traits + generics + blanket impls
3. C++: CRTP templates + PIMPL
4. Idiom translation table
5. Unique mechanisms worth stealing
6. Per-language porting lessons

## 1. Go (`hiero-sdk-go`)

- **Contract**: an `Executable` interface (~20 methods) consumed by a free function
  `_Execute(client, e Executable)` — the spine dispatches purely through the
  interface. Shared state lives in an embedded `executable` struct.
- **RPC dispatch**: a `_Method` struct with two nullable function-pointer fields
  (`query func(...)`, `transaction func(...)`); each concrete type's `getMethod`
  returns the generated stub method. The spine picks whichever pointer is non-nil.
- **Fluent chaining with type preservation**: `Transaction[T TransactionInterface]`
  is generic over the concrete child and stores `childTransaction T`; base setters
  return `T`, so chains keep the concrete type without casts. (The Go SDK's `Query`
  predates generics and uses interface-embedding — an internal inconsistency; use
  generics uniformly in a new port.)
- **Errors without exceptions**: `(value, error)` returns everywhere; sentinel
  errors + typed structs (`ErrHederaPreCheckStatus{Status, TxID}`). The notable
  adaptation: **deferred-error fields** — fluent setters can't return errors, so
  validation failures are stashed on the struct (`freezeError`, `keyError`) and
  surfaced at `Execute()`.
- **Concurrency**: synchronous loop with `time.Sleep` backoff and `context`
  deadlines; one background goroutine for the scheduled address-book refresh;
  `sync.RWMutex` around node health.
- **Codegen**: transaction classes are hand-written; Go AST rewriting keeps
  proto-derived enums/switches in sync with generated `.pb.go` instead of
  maintaining parallel lists.

## 2. Rust (`hiero-sdk-rust`)

- **Async**: tokio + tonic throughout; per-attempt outcome modeled as
  `ControlFlow<Response, Error>` (std type) instead of a hand-rolled state enum;
  retries driven by the `backoff` crate with `max_elapsed_time = request_timeout`.
- **Contract**: one `trait Execute` with associated types
  (`GrpcRequest/GrpcResponse/Context/Response`) plus a **blanket impl** over generic
  `Transaction<D>` / `Query<D>`. Concrete types are just type aliases
  (`type AccountCreateTransaction = Transaction<AccountCreateTransactionData>`);
  per-type code shrinks to a small `execute()` returning a boxed gRPC future.
  Layered data traits: `TransactionData` → `TransactionExecute` →
  `TransactionExecuteChunked` (marker).
- **Setters**: `&mut self -> &mut Self` chaining; all mutation funnels through a
  single `body_mut()` chokepoint that asserts not-frozen.
- **Freeze**: deliberately runtime (`is_frozen: bool` + `#[track_caller]` assert),
  NOT type-state — preserves cross-SDK ergonomics and error messages.
- **Errors**: public `#[non_exhaustive]` thiserror enum with structured variants
  (`TransactionPreCheckStatus { status, transaction_id, cost }` — cost populated on
  `InsufficientTxFee`, richer than other SDKs); a separate internal
  `Transient / Permanent / EmptyTransient` retry taxonomy classified in one place.
- **Client sharing**: `#[derive(Clone)] struct Client(Arc<ClientInner>)`; swappable
  config (operator, network) behind `ArcSwap` so background address-book refresh is
  lock-free — executes snapshot the network per attempt.
- **Codegen**: prost/tonic at build time over the vendored HAPI protos (with a small
  build.rs preprocessing pass).

## 3. C++ (`hiero-sdk-cpp`)

- **Contract**: a 4-parameter class template
  `Executable<SdkRequestType, ProtoRequestType, ProtoResponseType, SdkResponseType>`
  mixing **CRTP** (the derived type is a template param; fluent setters
  `return static_cast<SdkRequestType&>(*this)`) with **virtual hooks**
  (`makeRequest`, `mapResponse`, `submitRequest`, `determineStatus` →
  SUCCESS/SERVER_ERROR/REQUEST_ERROR/RETRY/RETRY_WITH_ANOTHER_NODE). Two template
  layers: `AccountCreateTransaction : Transaction<AccountCreateTransaction> :
  Executable<...>`.
- **The C++-specific chore**: template bodies live in `.cc`, so there's a
  hand-maintained explicit-instantiation list of ~75 request types — every new
  transaction/query edits it (forgetting = link error).
- **Ownership**: PIMPL (`unique_ptr<Impl>`) on all public types keeps grpc/proto
  headers out of the public API (the C++ equivalent of "generated code never
  leaks"); `shared_ptr` for shared graph objects (nodes, networks, keys);
  entity IDs are plain value types.
- **Errors**: exceptions — `PrecheckStatusException` (status + txId),
  `ReceiptStatusException`, `MaxAttemptsExceededException`, `IllegalStateException`
  for freeze violations.
- **Async**: `executeAsync` returns `std::future`, plus callback overloads; also
  request/response **interceptor hooks** (`setRequestListener`/`setResponseListener`)
  no other SDK has.
- **Codegen**: CMake FetchContent pulls the HAPI repo at a pinned tag at configure
  time, runs protoc twice (cpp + grpc); vcpkg manifest pins native deps.
- Wart to avoid copying: `execute(const Client&)` `const_cast`s the client to
  refresh the address book mid-retry — design the client parameter as mutable/shared
  from the start.

## 4. Idiom translation table

| Concern | JS | Python | Go | Rust | C++ |
|---|---|---|---|---|---|
| Contract | abstract class | ABC | interface + embedded struct | trait + blanket impl over generic | CRTP template + virtuals |
| Fluent setter | `return this` | `return self` | pointer receiver returns `T` (generic child) | `&mut self -> &mut Self` | `static_cast<Derived&>(*this)` |
| Freeze guard | throw | raise | deferred-error field, checked at Execute | `assert!` + `#[track_caller]` | throw `IllegalStateException` |
| Retry state | ExecutionState enum | IntEnum | shared `_ExecutionState` | `ControlFlow` + Transient/Permanent | `ExecutionStatus` enum |
| Async | async-first | sync | sync + one goroutine | tokio async-first | sync + `std::future`/callbacks |
| Error surface | typed Error subclasses | typed Exceptions | sentinel vars + typed structs | `#[non_exhaustive]` thiserror enum | exception hierarchy |
| Proto privacy | `@internal` JSDoc | `_` prefix | unexported | `pub(crate)` | PIMPL + forward declarations |
| Client sharing | single instance | single instance | mutex-guarded | `Arc` + `ArcSwap` (lock-free) | PIMPL + shared_ptr internals |

## 5. Unique mechanisms 

- **Rust: active node pinging** — before executing, candidate nodes are filtered by
  a real `PingQuery`, not just passive backoff-based health. Best-in-class node
  selection.
- **Rust: transient/permanent retry taxonomy** — clearer than a retryable-bool.
- **Rust: cost attached to InsufficientTxFee errors** — better DX on the most
  common fee failure.
- **Go: deferred-error accumulation** — the honest answer for fluent APIs in
  no-exception languages.
- **Go: AST-synced enums** — generated-code drift protection when classes are
  hand-written.
- **C++: request/response interceptors** — a wire-level observation/mutation seam
  useful for debugging and middleware.
- **C++: PIMPL discipline** — the strongest enforcement of "generated proto never
  leaks" of any SDK.

## 6. Per-language porting lessons

Porting to a Go-like language (no exceptions, no inheritance):
interface for the contract, embedded struct for shared state, function-pointer
struct for RPC dispatch, generic-child pattern for type-preserving chains,
deferred-error fields validated at execute time.

Porting to a Rust-like language (traits, ownership):
one trait with associated types + blanket impl over a generic transaction/query;
concrete types as aliases; runtime freeze (not type-state); public non-exhaustive
error enum + internal transient/permanent split; Arc-wrapped client with lock-free
config swap.

Any language: keep the retryable status sets, freeze semantics, per-node bodies,
and receipt-polling behavior byte-identical to the reference SDKs — these are
protocol contracts, not style choices.
