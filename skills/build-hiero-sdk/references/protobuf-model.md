# The HAPI Protobuf Model

Reverse-engineered from `hiero-consensus-node/hapi/.../src/main/proto/services/`.
Everything an SDK does is shaped by these messages.

## Contents
1. The transaction envelope (three nested layers)
2. The query envelope
3. gRPC service definitions
4. The two-phase result model (response / receipt / record)
5. Shared vocabulary: IDs, keys, signatures, time
6. ResponseCodeEnum and phase-dependent interpretation
7. Design implications for SDK authors

## 1. The transaction envelope

A submitted transaction is a Russian doll of serialized byte blobs:

```
Transaction { bytes signedTransactionBytes = 5 }        // transaction.proto (fields 1-4 deprecated)
  └─ SignedTransaction {                                 // transaction_contents.proto
       bytes bodyBytes = 1                               //   a serialized TransactionBody
       SignatureMap sigMap = 2
     }
       └─ TransactionBody {                              // transaction.proto
            TransactionID transactionID = 1              //   payer + validStart timestamp
            AccountID nodeAccountID = 2                  //   the ONE node this copy targets
            uint64 transactionFee = 3                    //   max fee payer allows
            Duration transactionValidDuration = 4        //   default ~120s
            string memo = 6
            Key batch_key = 73                           //   HIP-551 atomic batch
            oneof data { ... ~70 arms ... }              //   the actual operation
          }
```

The `oneof data` arms each point at a per-operation body message in its own proto file:
`CryptoTransferTransactionBody cryptoTransfer = 14`, `TokenMintTransactionBody
tokenMint = 37`, `ConsensusSubmitMessageTransactionBody = 27`, `AtomicBatchTransactionBody
atomic_batch = 74`, etc. Reserved field numbers (30, 61-64) show the append-only,
never-renumber discipline — honor `reserved`, tolerate unknown arms.

**Why bytes-before-signing:** protobuf serialization is not canonical across
languages/versions. Signatures are computed over the exact `bodyBytes`; if any party
re-encoded the parsed body, the bytes — and the signature hash — could differ. So the
body is frozen into an opaque blob once, and everyone signs/verifies those bytes.
This single fact creates the SDK's freeze lifecycle.

**Why one body per node:** `nodeAccountID` is inside the signed bytes. Submitting to a
different node requires different bytes and fresh signatures. SDKs therefore build a
{nodeAccountId → bodyBytes} map at freeze time and sign each entry.

## 2. The query envelope

```
Query { oneof query { ... ~28 arms ... } }               // query.proto
Each *Query message embeds:
  QueryHeader {                                          // query_header.proto
    Transaction payment = 1        // a signed CryptoTransfer paying the node
    ResponseType responseType = 2  // ANSWER_ONLY | COST_ANSWER | (± STATE_PROOF)
  }

Response { oneof response { ... mirror arms ... } }      // response.proto
Each *Response embeds:
  ResponseHeader {                                       // response_header.proto
    ResponseCodeEnum nodeTransactionPrecheckCode = 1
    ResponseType responseType = 2
    uint64 cost = 3
    bytes stateProof = 4
  }
```

**The COST_ANSWER handshake:** to learn a query's price, send it with
`responseType=COST_ANSWER` (free, empty payment) — the node returns only
`header.cost`. Then resend `ANSWER_ONLY` with a payment transaction of at least that
cost (SDKs add ~10% buffer). Genuinely free queries: `TransactionGetReceiptQuery` and
network version info — which is exactly why receipt polling costs nothing.

## 3. gRPC service definitions

Every RPC has one of exactly two signatures:

- `rpc X (Transaction) returns (TransactionResponse)` — state-changing
- `rpc X (Query) returns (Response)` — read

Services (each in `*_service.proto`): **CryptoService** (accounts, transfers,
receipts/records queries), **TokenService** (create/mint/burn/kyc/freeze/pause/wipe/
airdrop/reject/updateNfts), **ConsensusService** (topics), **SmartContractService**
(contracts + callEthereum), **FileService**, **ScheduleService**, **NetworkService**,
**UtilService** (prng, atomicBatch), **FreezeService**, **AddressBookService**
(node create/update/delete).

The RPC method and the body's oneof arm are redundant — nodes route by the oneof.
The SDK's Channel abstraction exposes one lazily-created stub per service; each
concrete transaction/query names exactly one stub method.

## 4. The two-phase result model

| Message | When | Cost | Contents |
|---|---|---|---|
| `TransactionResponse` | synchronous, from submit | — | ONLY `nodeTransactionPrecheckCode` + `cost`. "Admitted to the queue", nothing more |
| `TransactionReceipt` | after consensus, via free query | free | `status` + created entity IDs (accountID/tokenID/topicID/...), topic sequence/hash, `newTotalSupply`, serials, exchange rate |
| `TransactionRecord` | after consensus, paid query | paid | embeds receipt + transactionHash, consensusTimestamp, actual fee, full transfer lists, contract results, custom fees, `oneof entropy` (prng) |

**Receipts expire (~180s).** Nodes keep receipts only briefly; finality is discovered
by polling `getTransactionReceipts(txId)` until status leaves
`UNKNOWN`/`RECEIPT_NOT_FOUND`. After the window, history lives on mirror nodes. This is
why every SDK ships receipt polling with in-place (single node) retry, and why
"execute succeeded" in SDK terms means "receipt fetched and status == SUCCESS".

## 5. Shared vocabulary (basic_types.proto)

**Entity IDs — the shard.realm.num triplet.** AccountID, TokenID, FileID, ContractID,
TopicID, ScheduleID, NftID all share the shape:

```
message AccountID {
  int64 shardNum = 1; int64 realmNum = 2;
  oneof account { int64 accountNum = 3; bytes alias = 4; }   // alias = key or EVM address
}
```

Text form `0.0.1234`. AccountID/ContractID carry an alias/EVM-address oneof arm —
an account can be referenced before its number exists. Build ONE entity-id abstraction
and stamp it per type.

**TransactionID** = `Timestamp transactionValidStart` + `AccountID accountID` (payer)
+ `scheduled` + `nonce`. The client mints it; validStart is both the dedup key and the
start of the validity window. Back-date slightly to tolerate clock skew.

**Keys are a recursive tree:**

```
message Key { oneof key {
  bytes ed25519; bytes ECDSA_secp256k1;
  ThresholdKey thresholdKey;    // { uint32 threshold; KeyList keys; }
  KeyList keyList;              // { repeated Key keys; }
  ContractID contractID; ContractID delegatable_contract_id;
}}
```

Arbitrary nesting depth (m-of-n of lists of keys...), and a key can BE a contract.
Model this as a Key interface with KeyList/ThresholdKey composites.

**Signatures mirror keys by prefix:**

```
message SignaturePair { bytes pubKeyPrefix = 1; oneof signature { bytes ed25519; bytes ECDSA_secp256k1; } }
message SignatureMap  { repeated SignaturePair sigPair = 1; }
```

`pubKeyPrefix` is the shortest unambiguous prefix of the signer's public key; the node
matches prefixes against the required key tree. SDKs typically send the full key as
prefix. Dedup on public key when signing.

**Time:** `Timestamp{int64 seconds; int32 nanos}` — nanosecond precision, because
consensus timestamps are unique and ordered. `Duration{int64 seconds}`.

## 6. ResponseCodeEnum

One flat enum (~388 values, `response_code.proto`) reused in FOUR places: transaction
precheck, query precheck, receipt status, record status. The same namespace spans two
lifecycle phases, so interpretation is phase-dependent:

- Precheck-only codes: `OK`(0), `BUSY`, `DUPLICATE_TRANSACTION`, `TRANSACTION_EXPIRED`,
  `INSUFFICIENT_TX_FEE`, `INVALID_SIGNATURE`
- Receipt-only codes: `SUCCESS`(22), entity-specific errors (`INVALID_TOKEN_ID`, ...)
- Polling signals: `UNKNOWN`(21), `RECEIPT_NOT_FOUND` → keep polling, not failure
- `OK` ≠ `SUCCESS`. Ever.

## 7. Design implications for SDK authors

1. Freeze `bodyBytes` before signing; never re-serialize; setters throw post-freeze.
2. Build and sign one body per candidate node; retry-on-another-node = different bytes.
3. Mint TransactionIDs client-side; manage clock skew and the ~120s validity window.
4. Implement receipt polling as a first-class, free, single-node-pinned query; treat
   `UNKNOWN`/`RECEIPT_NOT_FOUND` as "continue", and surface receipt status ≠ SUCCESS
   as a typed error distinct from precheck errors.
5. Implement the COST_ANSWER pre-flight and max-query-payment guardrails.
6. One entity-ID type family (shard.realm.num + alias arm + checksummed string form).
7. Recursive Key model with prefix-based SignaturePair generation and signer dedup.
8. Route by service stub but drive semantics from the body oneof; keep a
   oneof-tag→class registry for deserialization; tolerate unknown arms.
