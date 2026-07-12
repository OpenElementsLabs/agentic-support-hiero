# Canonical Sources

Where everything in this skill comes from, and where to verify or go deeper. When
this skill and a canonical source disagree, the source wins — protos and reference
SDKs evolve.

## Protocol definitions (the ground truth)

- **HAPI protobufs** — `hiero-consensus-node` repo, under
  `hapi/hedera-protobuf-java-api/src/main/proto/`:
  https://github.com/hiero-ledger/hiero-consensus-node/tree/main/hapi/hedera-protobuf-java-api/src/main/proto
  - `services/` — transaction/query bodies, `basic_types.proto`,
    `response_code.proto`, all `*_service.proto` gRPC definitions
  - `mirror/` — mirror-node gRPC (topic subscription, address book)
  - `platform/`, `streams/`, `block/`, `fees/` — platform internals, record/block
    streams, fee schedules (rarely needed by SDK authors)

  The envelope files an SDK author opens constantly (all under `services/`):
  | File | Contains |
  |---|---|
  | `transaction.proto` | `Transaction` envelope + `TransactionBody` with the `oneof data` |
  | `transaction_contents.proto` | `SignedTransaction` (bodyBytes + sigMap) |
  | `transaction_response.proto` | the precheck-only submit response |
  | `transaction_receipt.proto` / `transaction_record.proto` | the two-phase results |
  | `query.proto` / `query_header.proto` | `Query` oneof + payment header |
  | `response.proto` / `response_header.proto` | `Response` oneof + precheck/cost header |
  | `basic_types.proto` | entity IDs, `Key`/`KeyList`/`ThresholdKey`, `SignatureMap`, `HederaFunctionality` |
  | `response_code.proto` | the single ~388-value status enum |
  | `<operation>.proto` (e.g. `token_create.proto`, `crypto_transfer.proto`) | each operation's body message |
  | `<service>_service.proto` | the gRPC service each operation is submitted to |

- **HAPI documentation (human-readable, field-by-field)** —
  https://hashgraph.github.io/hedera-protobufs/ — the auto-generated protobuf
  reference, one page per message with every field's doc comment. This is the
  prose counterpart to the raw protos. (The older
  `docs.hedera.com/hedera/sdks-and-apis/hedera-api` pages are marked outdated and
  redirect here.)
- **SDK-level transaction docs (what the API should feel like)** — per-operation
  pages under `https://docs.hedera.com/hedera/sdks-and-apis/sdks/<service>/<operation>`,
  e.g. https://docs.hedera.com/hedera/sdks-and-apis/sdks/token-service/define-a-token
  for TokenCreateTransaction — required/optional fields plus code samples across
  SDKs. Useful as the target UX when designing a new SDK's method surface.
- **Fees** — https://docs.hedera.com/hedera/networks/mainnet/fees — what each
  operation costs; informs per-type default max fees.
- **Mirror node REST API** — OpenAPI docs: https://mainnet.mirrornode.hedera.com/api/v1/docs/
  (same paths on testnet/previewnet/local)
- **HIPs (Hedera Improvement Proposals)** — https://hips.hedera.com — the design
  rationale behind SDK-visible features. Notable for SDK authors: HIP-338 (signers/
  wallets), HIP-551 (atomic batch), HIP-15 (entity-ID checksums), HIP-32 (auto
  account creation via alias), HIP-583 (EVM address aliases), HIP-991 (topic custom
  fees).

## Reference implementations

| SDK | Repo |
|---|---|
| JS/TS (reference for this skill) | https://github.com/hiero-ledger/hiero-sdk-js |
| Python | https://github.com/hiero-ledger/hiero-sdk-python |
| Go | https://github.com/hiero-ledger/hiero-sdk-go |
| Rust | https://github.com/hiero-ledger/hiero-sdk-rust |
| C++ | https://github.com/hiero-ledger/hiero-sdk-cpp |
| Java | https://github.com/hiero-ledger/hiero-sdk-java |

When implementing anything protocol-sensitive (retryable status sets, DER prefixes,
derivation paths, chunk sizes), diff at least two SDKs — a single SDK may carry a
local bug (this skill documents several).

## Cross-SDK governance and conformance

- **SDK Collaboration Hub** — https://github.com/hiero-ledger/sdk-collaboration-hub —
  the org's cross-SDK developer documentation. Especially:
  - `guides/api-guideline.md` — the authoritative cross-language API design
    guideline (this skill's conventions were derived independently from the code;
    the hub guideline is the governance document — check it for rulings)
  - `guides/api-best-practices-<language>.md` — per-language best practices
    (python, js, ts, go, rust, java, cpp, swift)
- **TCK (Technology Compatibility Kit)** — https://github.com/hiero-ledger/hiero-sdk-tck —
  the language-neutral conformance driver and per-method test specifications
  (`docs/test-specifications/`). A new SDK "conforms" when it passes these suites
  via its own TCK JSON-RPC server.

## The lookup chain for implementing one transaction/query

When adding, say, `TokenCreateTransaction` to an SDK, consult in this order:

1. **Proto body** — `services/token_create.proto` (fields, types, oneofs):
   https://github.com/hiero-ledger/hiero-consensus-node/tree/main/hapi/hedera-protobuf-java-api/src/main/proto/services
2. **HAPI docs** for that operation — field semantics, defaults, size limits:
   https://hashgraph.github.io/hedera-protobufs/
3. **Reference implementation** — the same class in the JS SDK (and one more SDK
   to catch local bugs): `src/token/TokenCreateTransaction.js`
4. **SDK docs page** for the operation — the intended developer-facing surface:
   `docs.hedera.com/hedera/sdks-and-apis/sdks/<service>/<operation>`
5. **TCK test specification** — the conformance contract your implementation must
   pass: https://github.com/hiero-ledger/hiero-sdk-tck/tree/main/docs/test-specifications
6. **Relevant HIP** if the feature is recent — design rationale and edge cases:
   https://hips.hedera.com

## Developer-facing product docs

- Hedera docs: https://docs.hedera.com (also exposed as an MCP server —
  see this plugin's `.mcp.json` `hedera-docs` entry)
- Hiero project: https://hiero.org and https://github.com/hiero-ledger
- Local networks: `solo` — see the `hiero-solo` skill in this plugin

## How this skill was built (provenance)

Distilled July 2026 from: the JS SDK (primary reference), cross-checked against the
Python, Go, Rust, and C++ SDKs to separate essential architecture from language
accident, plus the HAPI protos at `hiero-consensus-node` and both SDKs' test
suites/TCK servers. File-level citations were verified at time of writing; treat
line numbers as approximate as the repos move.
