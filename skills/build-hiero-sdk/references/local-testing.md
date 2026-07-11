# Testing an SDK Against a Local Network

How SDK test suites wire to a local solo network. For *deploying/operating* the
network itself (install, one-shot, multi-node, teardown), use the `hiero-solo`
skill — this doc covers only the SDK side of the contract.

## Contents
1. The local network shape and the port trap
2. The test-env contract
3. CI wiring (hiero-solo-action)
4. The TCK loop
5. The developer inner loop
6. What to verify at each level

## 1. The local network shape — and the port trap

A local network exposes: consensus gRPC (node account `0.0.3`), mirror gRPC (5600),
mirror REST, and optionally a JSON-RPC relay. The genesis operator is **`0.0.2`**
with the well-known genesis Ed25519 key; CI setups mint a separate funded test
operator per run and hand it to tests — prefer that over operating as 0.0.2.

**The trap: SDK presets disagree on ports.** Python's `solo` preset expects
consensus `localhost:35211` and REST `:38081`; JS's `local-node` preset expects
consensus `127.0.0.1:50211` and REST `:5551`. Both use mirror gRPC `:5600`. Your
`.env`/port-forwards must match the SDK under test — and a new SDK should document
its choice loudly. Don't hardcode assumptions; confirm actual ports from the
running network (e.g. `solo one-shot show`).

TLS is **off** for local networks: SDKs enable transport security only for
mainnet/testnet/previewnet presets.

## 2. The test-env contract

Both SDKs use the same shape — an integration-test environment object that:

- reads the target network from env (`NETWORK=solo` / `HEDERA_NETWORK=local-node`),
- reads `OPERATOR_ID` / `OPERATOR_KEY` from env (fail loudly if missing; load a
  local `.env` for convenience),
- **hard-refuses mainnet** (a cheap guard that has definitely saved someone),
- exposes factory helpers (`create_account`, `create_fungible_token`, ...) that
  accept per-test mutator hooks,
- provides a mirror-node polling helper (poll with timeout/interval — mirror data
  trails consensus by seconds; never fixed sleeps),
- is torn down in `finally`/afterAll.

## 3. CI wiring

The shared GitHub Action `hiero-ledger/hiero-solo-action` (pin a version) spins up
solo with mirror node, and outputs a funded operator: map its outputs to env —
`OPERATOR_ID={{outputs.accountId}}`, `OPERATOR_KEY={{outputs.privateKey}}`,
`NETWORK=solo` — then run the integration suite. Gate integration on unit tests
passing first, and parallelize with per-file distribution (tests share network
state; per-file isolation avoids cross-test interference).

## 4. The TCK loop

The TCK server is a JSON-RPC service the SDK ships (`tck/`, default port 8544).
The external, language-neutral TCK driver calls its `setup` method with node IP,
node account id, and mirror IP — the handler builds a client for exactly that
network (`Client.for_network({nodeIp: nodeAccountId})` + mirror address + operator
from params). So the TCK server has no hardcoded network: the driver decides.
Running conformance locally = solo up → SDK's TCK server up → driver suite pointed
at both.

## 5. The developer inner loop

1. **Unit** (no network): mocked in-process gRPC server — seconds, run constantly.
2. **Integration** (solo): bring the network up once, keep it running, iterate.
3. **TCK** (conformance): run the driver suites for the methods you touched.
4. **Testnet** (pre-release): the same integration suite against the public
   testnet with a funded account — catches anything solo's single node hides
   (real latency, node rotation, fee variance).

## 6. What to verify at each level

| Level | Catches |
|---|---|
| Unit + mock server | serialization, retry classification, freeze/sign logic |
| Proto-body assertions | field mapping, set-vs-unset semantics |
| Solo integration | end-to-end signing validity, receipts, real gRPC framing |
| TCK | cross-SDK behavioral parity, spec-exact response shapes |
| Testnet | multi-node behavior, node health/backoff, real fees/throttles |

A change to the execution spine or crypto layer needs all five; a new leaf
transaction type usually needs the first four.
