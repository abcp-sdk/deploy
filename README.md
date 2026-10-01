# abc-protocol/deploy

The **deployment home of the ABCP agent** (`abc-protocol`): it holds the Helm
charts that wire the agent and its extensions together, plus the standalone k8s
manifests for the non-Helm workloads. **Each component's code (and its
`Dockerfile` / `build-image.sh`) lives in its OWN repository.** This repo is
deploy-only: no application code, no image build.

**Scope: `abc-protocol` only.** This repo deploys `abc-protocol/agent` and its
extensions. Other stacks (e.g. `coding-workspace`) have their own deployment and
are explicitly out of scope here.

> Mirrors the `easy-vcs/deploy` pattern: one release wires the whole platform,
> images are built by each owning repo's `build-image.sh` and pinned here by tag.

## Layout

```
deploy/
├── charts/
│   ├── infra/       # shared infrastructure (NATS / Garage S3 / Forgejo / Selenium / Postgres / buildkitd)
│   └── platform/    # the agent stack (agent + webui + playwright + fixed worker + worker-extension)
├── worker-k8s/      # standalone agent-worker deployments (linux / macOS / Windows / Android / desktop + KVM device plugin)
└── README.md
```

The `platform` release consumes the `infra` release over Service DNS.

## Components (each image built from its OWN repository)

| Component | Repository | Role |
|---|---|---|
| `agent` | `abc-protocol/agent` | session/model/turn engine (h2c); loads the in-process bundled extension |
| `webui` (agent-webui) | `abc-protocol/webui` | SPA + same-origin Caddy aggregator (`/agent.v1.*` → agent h2c) |
| `playwright-extension` | `abc-protocol/playwright-extension` | browser-automation tools (drives infra Selenium over CDP) |
| `worker-extension` | `abc-protocol/worker-extension` | run commands / read-write files in ONE fixed easyworker |
| `agent-worker` | `abc-protocol/worker` | the easyworker binary; also the sandbox worker injected at launch |

## Prerequisites

- A cluster with a node that can run `hostPath` volumes (the dev cluster) — or
  set the chart to `persistence.mode: pvc` / `agent.db.backend: pg`.

## Install

```sh
# 1) shared infrastructure (NATS + Garage + Forgejo + Selenium + Postgres + buildkitd)
helm install infra   ./charts/infra   -n agent --create-namespace

# 2) the standalone agent stack
helm install platform ./charts/platform -n agent
```

Image tags are pinned in each chart's `values.yaml` and must match tags pushed
by each repo's `build-image.sh` (`<registry>/abcp/<name>:<tag>`).

### A second, independent environment (`values-standalone2.yaml`)

Each chart ships an extra values file for a SECOND, self-contained environment
(`abcp-agent-s2` + `infra-s2`), co-located in the `agent` namespace but with its
own NATS/S3/Postgres/Selenium and PVC-backed state (`s2-` prefix). Unlike the
original, Forgejo and buildkitd are disabled and the agent's metadata DB is
Postgres.

```sh
helm install infra-s2    ./charts/infra    -n agent -f ./charts/infra/values-standalone2.yaml
helm install abcp-agent-s2 ./charts/platform -n agent -f ./charts/platform/values-standalone2.yaml
```

### Restricted deploy tools (RBAC)

An in-cluster restricted deploy tool (e.g. `helm-deploy`) renders a **whitelist**
of namespaced kinds and filters RBAC / cluster-scoped / privileged objects. The
`platform` chart ships only a namespaced `ServiceAccount` (allowed), so it applies
as-is. The standalone worker manifests under `worker-k8s/` are NOT a Helm release
and include a **privileged** device plugin (`generic-device-plugin.yaml`) and a
**cluster-scoped** local-path provisioner (`workspace-local-path.yaml`) — apply
those out-of-band with the appropriate context (see below).

## NATS isolation (do not skip)

The platform stack uses the `agent` NATS account. Any other agent stack in the
same broker MUST use a **distinct account** (`agent` / `workspace` / `class` in
`charts/infra/values.yaml`): two agents sharing an account collide on the global
`abc.discover` subject, the `abc-presence` KV keys and the fixed `ABC_MAILBOX` /
`ABC_EVENTS` / `ABC_DLQ` streams. Add an account with `nats.extra` (each
`{name,user,password}`).

## Standalone worker deployments (`worker-k8s/`)

Plain manifests (not Helm) for long-lived `agent-worker` sandboxes: linux
(`agent-worker.yaml`), macOS + Xcode, Windows, Android emulator, desktop
(noVNC), plus the KVM device plugin. The VM/Android manifests request
`squat.ai/kvm` while staying `privileged: false`. Replace `<node-name>` and
`<registry-credentials>` before applying (the image refs already point at the
in-cluster registry).

`workspace-local-path.yaml` is the **cluster-scoped** `workspace-local`
StorageClass plus its own `rancher/local-path-provisioner` (host path
`/home/develop/PVC`) — the default `persistence.storageClass` the `infra` and
`platform` charts reference. It contains a `Namespace` / `ClusterRole` /
`ClusterRoleBinding` / `StorageClass`, so a restricted deploy tool (which filters
RBAC / cluster-scoped kinds) CANNOT apply it: apply it out-of-band (drop it into
the k3s `server/manifests/` directory, or `kubectl apply -f` with
cluster-admin). `generic-device-plugin.yaml` is likewise applied out-of-band.

## Working agreement (how every repo works now)

**Code repos own code; this repo owns deployment.** Concretely:

| Repository | Keeps | No longer holds |
|---|---|---|
| `abc-protocol/agent` | `packages/`, `scripts/`, `Dockerfile`, `build-image.sh` | ~~`k8s/chart`, `k8s/infra-chart`~~ → here |
| `abc-protocol/worker` | Go source, `webui/`, `agent-toolchain/` (sandbox image catalog), `Dockerfile`, `build-image.sh` | ~~`k8s/agent-worker*.yaml`, `k8s/generic-device-plugin.yaml`~~ → here |

To change a deployment:

1. Edit the chart here and open an MR to `abc-protocol/deploy`.
2. Bump the per-component image tag in `values.yaml` after the owning repo
   builds and pushes a new image.
3. Nothing else changes in the code repos.

`build-image.sh` (per component repo) and `agent-toolchain/` (sandbox base
images, in `abc-protocol/worker`) stay with their code — they build artifacts,
they do not deploy.

