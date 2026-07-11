# SDK Conventions and Best Practices

Named conventions shared across the JS, Python, and Go Hiero SDKs, with the reasoning.
Apply the rule; keep the language's own casing (`setTokenId` / `set_token_id` /
`SetTokenId`).

## Contents
1. Setters and construction
2. Factories and conversions
3. Test formats (unit / integration / TCK / round-trip)
4. Errors and validation
5. Examples and docs
6. Release and compatibility

## 1. Setters and construction

- **Fluent setters return self.** One field per setter; chaining is the primary
  construction style. Rationale: transactions have many optional fields; chaining
  reads as a builder without builder machinery.
- **`require_not_frozen` is the first line of every setter.** Post-freeze mutation
  would invalidate signatures; fail fast with an "immutable" error instead of letting
  the network reject a corrupted body.
- **Coerce convenience inputs, validate cheap invariants.** Accept `str | AccountId`,
  `str | bytes`, `int | Long`; normalize immediately to the canonical type so
  downstream code sees one type. Enforce local bounds (e.g. metadata ≤ 100 bytes)
  with typed errors. Example:

  ```python
  def set_metadata(self, metadata: bytes | str):
      self._require_not_frozen()
      if isinstance(metadata, str): metadata = metadata.encode("utf-8")
      if len(metadata) > 100: raise ValueError("Metadata must not exceed 100 bytes")
      self._metadata = metadata
      return self
  ```

- **Constructor/setter parity, constructor delegates to setters.** Everything
  constructible in one shot is also settable incrementally; the constructor forwards
  to setters so validation exists once. Required-field validation is deferred to
  build/freeze time so partial construction never throws.

## 2. Factories and conversions

- **`from_string` / `to_string` round-trip** the canonical `shard.realm.num` text
  form — the interop contract for configs, env vars, and docs.
- **`from_bytes` / `to_bytes` are exact inverses**, wrapping proto encode/decode.
- **`_to_proto` / `_from_proto` are private** (underscore, `@internal` tags). Proto
  shapes change with HAPI versions; the public surface must not expose them. This is
  the single most consistent convention across all SDKs.
- **Checksum methods are separate and client-aware**: `to_string_with_checksum(client)`
  / `validate_checksum(client)` need the ledger id (mainnet vs testnet checksums
  differ), so plain `to_string` stays dependency-free. Refuse checksums for
  alias/EVM-address IDs (undefined).
- **Named factories over overloaded constructors**: `from_string`, `from_bytes`,
  `from_evm_address`, `from_solidity_address` — each input has distinct validation
  and error messages.

## 3. Test formats

- **Unit tests run a real in-process gRPC server**, not method mocks. Bind a local
  server on port 0, register every service, queue canned proto responses per call
  (a list per simulated node); the client connects over real insecure gRPC. This
  exercises serialization, node selection, retry, and precheck handling for real.
  Error entries in the queue abort with a chosen gRPC code to test transport retry.
- **Proto-body assertion tests**: build the transaction, call the body builder
  directly, assert field-by-field — including `HasField` checks proving unset
  optionals stay unset (critical for update transactions where set vs unset differ).
- **Round-trip tests** for every value type: string↔object, bytes↔object,
  object↔proto — including alias/EVM variants.
- **Integration tests**: one shared env object that builds the client from
  env vars (operator id/key), refuses to run against mainnet, provides factory
  helpers (`create_account`, `create_fungible_token`) accepting per-test mutator
  lambdas, and is closed in `finally`/afterAll. Mirror-node eventual consistency is
  handled with an explicit polling helper, not sleeps.
- **The TCK server is the cross-SDK conformance layer.** Each SDK ships `tck/` — a
  JSON-RPC 2.0 server with three parallel packages:
  - `param/` — request dataclasses with `parse_json_params` classmethods
  - `handlers/` — functions registered by method name (decorator/registry),
    wrapped in an SDK-error→JSON-RPC-error adapter
  - `response/` — response dataclasses serialized back to JSON
  One language-neutral driver suite runs against every SDK; passing it is the
  definition of behavioral parity. When implementing a TCK method: read the spec's
  request/response tables literally, match the reference SDK's semantics, validate
  required params (proper invalid-params errors, no crash paths), and return
  spec-exact field names with unset optionals omitted.

## 4. Errors and validation

- **Typed errors carry the machine-readable context**: status code, transaction id,
  node id, and (for receipt errors) the full receipt. Distinct classes per lifecycle
  phase (precheck vs receipt vs exhausted-retries) so callers know *where* it failed.
- **Fail fast locally only on cheap, unambiguous checks** (types, byte lengths,
  positive counts). Defer business validation (does the account exist, is the fee
  enough) to the network — duplicating ledger rules locally is brittle and steals
  authority from the source of truth.
- **Checksum validation** catches cross-network ID mixups before spending a
  transaction fee.

## 5. Examples and docs

- **One runnable example file per operation**, self-contained: top docstring with
  purpose, required env vars, and the exact run command; small named step functions
  (`setup_client`, `build_transaction`, ...); client from `Client.from_env()`;
  progress prints; nonzero exit on failure.
- **Docstrings/JSDoc on every public member** in the language's canonical style
  (Google-style Args/Returns/Raises; JSDoc @param/@returns/@internal/@deprecated) —
  they feed type checkers and generated API docs from one source.
- **Contributor docs are separate from user docs** (`docs/sdk_developers/` vs
  `docs/sdk_users/`): how to extend vs how to use.

## 6. Release and compatibility

- **Deprecate in place, never remove within a major**: old symbol stays, emits a
  deprecation warning pointing at the replacement (with the caller's line via
  stacklevel), and delegates to the new implementation. Name the removal version in
  the message.
- **SemVer with tagged releases** (`v*.*.*` tags trigger CI publish after tests).
- **Keep-a-Changelog format**: newest first, Added/Changed/Fixed groups, every
  bullet links its PR.
- **Pin the HAPI proto version** the SDK is generated against; proto upgrades are
  deliberate, reviewed events (new oneof arms → new registry entries → new leaf
  classes), not silent regenerations.
