# Walkthrough: Empty Repo → First Confirmed Transfer

The milestone path for a brand-new SDK, in language-neutral pseudocode. Each
milestone is independently verifiable; don't start the next until the check passes.
Total scope to "first confirmed transfer": roughly the 10-class minimum from
SKILL.md.

## Milestone 0 — Codegen

Pin a HAPI proto version (a git tag). Generate message types + gRPC stubs for the
`services/` protos. Wrap access behind one internal module; nothing outside it may
import generated types.

*Check:* you can construct a `TransactionBody`, set `memo`, serialize, parse back.

## Milestone 1 — Value types and keys

```
AccountId { shard, realm, num }        from_string("0.0.2") / to_string / _to_proto
Hbar { tinybars }                      from_tinybars / of(hbar) / negated()
Timestamp { seconds, nanos }           now() / plus_nanos
TransactionId { account_id, valid_start }   generate(operator_id)  # back-date ~10s
PrivateKey / PublicKey                 ed25519 first; sign(bytes) -> 64B; from_string DER+raw
```

*Check:* round-trip tests string↔object↔proto↔bytes; sign a test vector and verify
against another SDK's output for the same key.

## Milestone 2 — Network stack (minimal)

```
Channel(address)          # lazy grpc client; one stub accessor per service (crypto first)
Node(account_id, address) # owns channel; is_healthy() always true for now
Network                   # hardcode 2-4 testnet/solo nodes; select() round-robins
Client                    # operator(id, key); for_testnet()/for_solo(); defaults
```

Skip TLS, health backoff, and address-book refresh for now — they bolt on later
without redesign (that's what the layering buys you).

*Check:* open a channel to a local solo node; call any stub with an empty query and
get a gRPC response (even an error proves transport works).

## Milestone 3 — The transaction path

```
class Transaction(Executable):
  freeze_with(client):
    fee     = self.fee or client.default_fee
    tx_id   = TransactionId.generate(client.operator.id)
    nodes   = network.pick(N)                       # redundancy
    for node in nodes:                              # ONE BODY PER NODE
      body       = build_body(tx_id, node.account_id, fee, memo, data_oneof)
      body_bytes = serialize(body)                  # FROZEN — never re-serialize
      self.signed[node] = SignedTransaction(body_bytes, sig_map={})
    self.frozen = true

  sign(key):
    require_frozen(); dedup_by_public_key(key)
    for st in self.signed.values():
      st.sig_map.add(key.public.prefix, key.sign(st.body_bytes))

  execute(client):
    freeze+sign with operator if needed
    loop attempts:
      node = next healthy node with a signed body
      resp = node.channel.crypto.<stub_method>(wrap(self.signed[node]))
      match precheck(resp):
        OK                          -> return TransactionResponse(tx_id, node, hash)
        BUSY | PLATFORM_* | INVALID_NODE_ACCOUNT -> backoff, next node
        TRANSACTION_EXPIRED         -> regenerate tx_id, re-freeze, re-sign, retry
        else                        -> raise PrecheckError(status, tx_id)
```

First leaf: `TransferTransaction` (hbar transfers list; body oneof `cryptoTransfer`;
stub `crypto.cryptoTransfer`). Enforce transfers sum to zero locally.

*Check:* execute a 1-tinybar self-transfer on solo; precheck returns OK.
(You still can't prove it *succeeded* — that's the next milestone, and the reason
receipt polling is not optional.)

## Milestone 4 — Receipt polling (the mandatory query)

```
class TransactionReceiptQuery(Query):
  payment_required = false                 # receipts are free
  node = SAME node the tx went to          # pin; back off in place, never round-robin
  loop:
    resp = channel.crypto.getTransactionReceipts(query_for(tx_id))
    if precheck in {RECEIPT_NOT_FOUND, UNKNOWN} or receipt.status == UNKNOWN:
      backoff; continue                    # consensus hasn't happened yet
    return TransactionReceipt(resp)        # entity ids, status

TransactionResponse.get_receipt(client):
  receipt = TransactionReceiptQuery(tx_id, node).execute(client)
  if receipt.status != SUCCESS: raise ReceiptStatusError(status, tx_id, receipt)
  return receipt
```

*Check:* the transfer's receipt returns `SUCCESS`. **This is "hello world" for the
SDK** — submit + confirm is the loop every user runs.

## Milestone 5 — First read: AccountBalanceQuery

Free query (no payment): build `cryptogetAccountBalance` with an empty header, call
the stub, map the response.

*Check:* balance of `0.0.2` on solo returns a positive number; balance of the
receiver reflects your transfer.

## Milestone 6 — Paid queries (the last core mechanic)

Add to Query base: COST_ANSWER pre-flight → `header.cost`, `max_query_payment`
guard, one signed CryptoTransfer payment per candidate node, `ANSWER_ONLY` resend.
First paid leaf: `AccountInfoQuery` or `FileContentsQuery`.

*Check:* a paid query succeeds with auto-payment; setting `max_query_payment` below
cost raises the typed error without submitting.

## After the walkthrough

Now harden in any order: node health/backoff + readmission, TLS with cert-hash
pinning, address-book refresh, `to_bytes`/`from_bytes` + the oneof registry,
chunked transactions, mirror layer (see `mirror-layer.md`), remaining ~90 leaf
types (mechanical — see "Adding one new transaction type" in SKILL.md), the TCK
server (see `conventions.md` §3), ECDSA + EVM addresses (see `crypto-details.md`).

## Definition of done for "core SDK"

- [ ] Transfer + receipt round-trip on solo and testnet
- [ ] Kill a node mid-run → SDK fails over (per-node bodies prove out)
- [ ] Unit suite runs against an in-process mock gRPC server
- [ ] `to_bytes`/`from_bytes` round-trips a signed transaction across processes
- [ ] A second SDK can verify your signatures (cross-SDK vector test)
- [ ] TCK `setup` + `createAccount` + `transferCrypto` methods pass the driver
