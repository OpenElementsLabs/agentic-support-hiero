---
name: hiero-solo
license: Apache-2.0
metadata:
  source: https://github.com/OpenElementsLabs/agentic-support-hiero
  author: Open Elements
description: Deploy and manage local or multi-node Hiero/Hedera test networks with the solo CLI. Use when the user wants to spin up a local Hiero network to develop or test an app, run a Hedera/Hiero SDK against a throwaway network, set up consensus nodes, mirror node, JSON-RPC relay, or explorer, or deploy larger multi-node test networks. Also use when the user mentions solo, `solo one-shot`, hiero-solo-action, or replacing hiero-local-node.
---

# Hiero Solo — Test Network Deployment

`solo` is the Hiero Ledger CLI for deploying and managing standalone Hiero/Hedera test networks on a local Kubernetes cluster (Kind by default). It deploys the full stack — consensus node(s), mirror node, JSON-RPC relay, and explorer — and is the preferred tool over the deprecated `hiero-local-node`. See the `hiero-info` skill for ecosystem context.

There are two ways to use solo:

- **One-shot** — a single command brings up a complete network. This is the right choice for app developers who just want an endpoint to test against.
- **Manual / step-by-step** — deploy each component individually. Use this for multi-node networks, CI/CD, custom builds, or when you need control over each stage.

> The solo CLI and its default ports change between releases. Always confirm exact syntax and endpoints for the installed version with `solo --version`, `solo --help`, `solo <group> --help`, and `solo one-shot show`. Prefer these over assuming the values below.

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Node.js | ≥ 22 (lts/jod) | Required to run solo |
| Docker (or Podman) | daemon running | Grant Docker Desktop ≥ 12 GB RAM and ≥ 6 CPUs |
| kubectl / Helm / Kind | recent | Auto-provisioned by solo at deploy time |

Single-node needs ~12 GB RAM / 4–6 CPU; multi-node (3+) needs ≥ 16 GB / 8 CPU. Insufficient Docker resources is the most common failure — check this first when a deploy hangs or crashes.

Install:

```bash
brew install hiero-ledger/tools/solo    # macOS / Linux (recommended)
npm install -g @hiero-ledger/solo@latest # cross-platform alternative
solo --version
```

## One-shot: the fast path for app developers

Use this when someone wants to develop or test an application against a Hiero network. One command builds the cluster and the whole network, then writes endpoints and pre-funded account keys to `~/.solo/one-shot-<deployment-name>/accounts.json`.

```bash
# Single consensus node — the default developer setup
solo one-shot single deploy

# EVM / smart-contract profile — creates 20 pre-funded ECDSA accounts (1,000,000 HBAR each)
solo one-shot single deploy --evm

# Multi-node from one command
solo one-shot multi deploy --num-consensus-nodes 3

# Inspect live endpoints and account keys at any time
solo one-shot show

# Tear it all down
solo one-shot single destroy
solo one-shot multi destroy
```

Useful one-shot flags: `--evm`, `--no-explorer`, `--explorer blockscout|mirror-node`, `--num-consensus-nodes N`.

### Falcon (YAML-driven one-shot)

For a repeatable one-shot defined in a values file — the recommended shape for team-shared or CI setups:

```bash
solo one-shot falcon prepare --values-file falcon-values.yaml   # generate the values file
solo one-shot falcon deploy  --values-file falcon-values.yaml
solo one-shot falcon destroy
```

The values file has `network`, `setup`, `consensusNode`, `mirrorNode`, `explorerNode`, `relayNode`, and optional `blockNode` sections.

## Connecting an app to the network

After a deploy, get the real endpoints from `solo one-shot show` or `solo deployment config ports --deployment <name>`. Do not hardcode ports — they are version-dependent (recent releases default to the values below; older ones use `50211` / `8084`).

| Service | Typical local endpoint |
|---------|------------------------|
| Consensus node gRPC (account `0.0.3`) | `127.0.0.1:35211` |
| Mirror node (REST / gRPC ingress) | `127.0.0.1:38081` |
| JSON-RPC relay | `http://localhost:37546` |
| Explorer UI | `http://localhost:38080` |

The genesis operator account is `0.0.2` (its key is in `accounts.json`). Example with the Hiero JS SDK:

```javascript
const network = { "127.0.0.1:35211": AccountId.fromString("0.0.3") };
const client = Client.fromConfig({ network, mirrorNetwork: "127.0.0.1:38081", scheduleNetworkUpdate: false });
```

If connections fail after they previously worked, the port-forwards may have dropped — re-establish them with `solo deployment refresh port-forwards` and verify with `nc -zv localhost 35211`.

## Manual deployment: larger and custom networks

Use the step-by-step workflow when you need multiple nodes, custom node software versions, multi-cluster, locally built components, or CI/CD control. Current releases namespace commands under resources (`cluster-ref config`, `deployment config`, `keys consensus generate`, `consensus network`, `consensus node`, `mirror node`, `relay node`, `explorer node`). Older tutorials use flat names (`solo network deploy`, `solo node setup`, `solo mirror-node add`) — a CLI migration reference is in the docs.

Set node count with `--num-consensus-nodes` at the `deployment cluster attach` step.

```bash
export SOLO_CLUSTER_NAME=solo
export SOLO_DEPLOYMENT=solo-deployment
export CONSENSUS_NODE_VERSION=v0.66.0   # confirm a current release tag

# 1. Connect the kube context and create a deployment
solo cluster-ref config connect --cluster-ref kind-${SOLO_CLUSTER_NAME} --context kind-${SOLO_CLUSTER_NAME}
solo deployment config create -n solo-namespace --deployment ${SOLO_DEPLOYMENT}

# 2. Attach the cluster and set the number of consensus nodes
solo deployment cluster attach --deployment ${SOLO_DEPLOYMENT} --cluster-ref kind-${SOLO_CLUSTER_NAME} --num-consensus-nodes 3

# 3. Generate gossip + TLS keys, then install shared cluster components
solo keys consensus generate --gossip-keys --tls-keys --deployment ${SOLO_DEPLOYMENT}
solo cluster-ref config setup --cluster-setup-namespace solo-cluster

# 4. Deploy and start the consensus network
solo consensus network deploy --deployment ${SOLO_DEPLOYMENT}
solo consensus node setup --deployment ${SOLO_DEPLOYMENT} --release-tag ${CONSENSUS_NODE_VERSION}
solo consensus node start --deployment ${SOLO_DEPLOYMENT}

# 5. Add the supporting services
solo mirror node add   --deployment ${SOLO_DEPLOYMENT} --cluster-ref kind-${SOLO_CLUSTER_NAME} --enable-ingress --pinger
solo explorer node add --deployment ${SOLO_DEPLOYMENT} --cluster-ref kind-${SOLO_CLUSTER_NAME}
solo relay node add    --deployment ${SOLO_DEPLOYMENT} -i node1
```

Tear down in reverse order (`relay node destroy` → `explorer node destroy` → `mirror node destroy` → `consensus network destroy`), then `solo cluster-ref config reset` to remove shared cluster components.

### Command groups reference

| Group | Purpose |
|-------|---------|
| `solo init` | Initialize the local environment |
| `solo cluster-ref config` | Map kube contexts to cluster refs: `connect`, `disconnect`, `list`, `setup`, `reset` |
| `solo deployment config` | `create`, `ports`; plus `deployment cluster attach`, `deployment refresh port-forwards` |
| `solo keys consensus generate` | Gossip / TLS key generation |
| `solo consensus network` | `deploy`, `destroy`, `freeze`, `upgrade` (all nodes) |
| `solo consensus node` | Per-node `setup`, `start`, `stop`, `add`, `destroy`, `upgrade` |
| `solo block node` | `add`, `destroy`, `upgrade` |
| `solo mirror node` / `relay node` / `explorer node` | `add`, `destroy` |
| `solo one-shot` | `single`, `multi`, `falcon`, `show` |

## CI/CD

For automated pipelines, use the `hiero-solo-action` GitHub Action (donated to Hiero by Open Elements) instead of scripting the CLI by hand. Repo examples under `hiero-ledger/solo/examples` (e.g. `one-shot-falcon`, `one-shot-local-build`, `hardhat-with-solo`) are driven by `Taskfile`s (`task deploy` / `task test` / `task destroy`) and are good starting points.

## References

| Resource | URL |
|----------|-----|
| Solo docs | https://solo.hiero.org/docs/ |
| Solo repository | https://github.com/hiero-ledger/solo |
| Solo examples | https://github.com/hiero-ledger/solo/tree/main/examples |
| Solo docs source | https://github.com/hiero-ledger/solo-docs |
| hiero-solo-action | https://github.com/hiero-ledger/hiero-solo-action |
