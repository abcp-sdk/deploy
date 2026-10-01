# abc-protocol/deploy

The **deployment home of the ABCP agent** (`abc-protocol`): it holds the Helm
chart that wires the agent and its extensions together, plus the standalone k8s
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
├── charts/platform/   # the standalone agent stack
├── worker-k8s/        # standalone agent-worker deployments (linux / macOS / Windows / Android / desktop + KVM device plugin)
└── README.md
```

## Shared infrastructure (do NOT deploy your own)

NATS, Garage S3 and Postgres are **shared services in the `worker` namespace**,
deployed and operated by `easy-vcs/deploy`. The `platform` chart consumes them
over Service DNS and **never deploys them itself**:

| Service | Endpoint |
|---|---|
| NATS (JetStream) | `nats.worker.svc.cluster.local:4222` |
| Postgres | `postgres.worker.svc.cluster.local:80` (→5432) |
| Garage S3 | `http://garage.worker.svc.cluster.local:80` (region `garage`, path-style) |

Because the services are shared, the account / bucket / database names MUST be
ABCP-specific (distinct from every other stack on the same services):

| Knob | Default stack | Second stack (`-s2`) |
|---|---|---|
| NATS account (user) | `abcp-agent` | `abcp-agent-s2` |
| S3 bucket | `abcp-agent` | `abcp-agent-s2` |
| Postgres database | `abcp_agent` | `abcp_agent_s2` |

The NATS **password** and S3 **access/secret key** are issued by
`easy-vcs/deploy`; pass them at install time (never commit them).

**Selenium** is NOT shared — the `platform` chart deploys its own Selenium node
(the `playwright-extension` drives it over CDP). **Forgejo and buildkitd are not
deployed at all**: the standalone agent needs neither a git server nor an image
builder.

## Components (each image built from its OWN repository)

| Component | Repository | Role |
|---|---|---|
| `agent` | `abc-protocol/agent` | session/model/turn engine (h2c); loads the in-process bundled extension |
| `webui` (agent-webui) | `abc-protocol/webui` | SPA + same-origin Caddy aggregator (`/agent.v1.*` → agent h2c) |
| `playwright-extension` | `abc-protocol/playwright-extension` | browser-automation tools (drives the in-chart Selenium over CDP) |
| `worker-extension` | `abc-protocol/worker-extension` | run commands / read-write files in ONE fixed easyworker |
| `agent-worker` | `abc-protocol/worker` | the easyworker binary; also the sandbox worker injected at launch |

## Install

```sh
# Request your NATS account, S3 bucket+key and Postgres database from
# easy-vcs/deploy first, then (release name MUST NOT be `platform`: that is the
# easy-vcs stack's release — use `abcp-platform`):
helm install abcp-platform ./charts/platform -n worker --create-namespace \
  --set infra.nats.password='<from easy-vcs>' \
  --set infra.s3.accessKey='<from easy-vcs>' \
  --set infra.s3.secretKey='<from easy-vcs>'
```

**Release name / namespace gotcha**: deploy into the `worker` namespace (where
the shared infra lives) and pick a release name that is NOT already used in the
cluster. `platform` belongs to the `easy-vcs` stack — installing a second
`platform` release silently UPGRADES theirs. Use `abcp-platform`.

**Object names are ABCP-prefixed** (`abcp-agent`, `abcp-webui`, …) for the same
reason: Helm does not enforce object ownership across releases, so a
cluster-unique name (not `agent`, not `agent-webui`) prevents two releases from
silently overwriting each other's objects. Before installing, run `helm-list`
and `kubectl get` to confirm the names are free.

The agent's metadata DB is the shared **Postgres** (`agent.db.backend: pg`), so
no `/data` volume is used. Image tags are pinned in `values.yaml` and must match
tags pushed by each repo's `build-image.sh` (`<registry>/abcp/<name>:<tag>`).

### A second, independent stack (`values-standalone2.yaml`)

`charts/platform/values-standalone2.yaml` defines a SECOND, independent stack
with its OWN NATS account / S3 bucket / Postgres database and its own
`s2-`-prefixed Selenium (Selenium is self-deployed, so two stacks need distinct
Service names):

```sh
helm install abcp-agent-s2 ./charts/platform -n worker \
  -f ./charts/platform/values-standalone2.yaml \
  --set infra.nats.password='<from easy-vcs>' \
  --set infra.s3.accessKey='<from easy-vcs>' \
  --set infra.s3.secretKey='<from easy-vcs>'
```

> **Not currently provisioned.** `easy-vcs/deploy` has only opened ONE ABCP
> tenant (`abcp-agent`); there is no `abcp-agent-s2` account / bucket / database
> yet. Request those before running the second stack.

### Restricted deploy tools (RBAC)

An in-cluster restricted deploy tool (e.g. `helm-deploy`) renders a **whitelist**
of namespaced kinds and filters RBAC / cluster-scoped / privileged objects. The
`platform` chart ships only a namespaced `ServiceAccount` (allowed), so it applies
as-is. The standalone worker manifests under `worker-k8s/` are NOT a Helm release
and include a **privileged** device plugin (`generic-device-plugin.yaml`) and a
**cluster-scoped** local-path provisioner (`workspace-local-path.yaml`) — apply
those out-of-band with the appropriate context (see below).

## NATS isolation (do not skip)

The shared NATS broker serves several stacks. Each agent stack MUST use a
**distinct account**: two agents sharing an account collide on the global
`abc.discover` subject, the `abc-presence` KV keys and the fixed `ABC_MAILBOX` /
`ABC_EVENTS` / `ABC_DLQ` streams. ABCP's account (`abcp-agent`) is opened on the
shared broker by `easy-vcs/deploy`; the password is supplied at install time.

## Standalone worker deployments (`worker-k8s/`)

Plain manifests (not Helm) for long-lived `agent-worker` sandboxes: linux
(`agent-worker.yaml`), macOS + Xcode, Windows, Android emulator, desktop
(noVNC), plus the KVM device plugin. The VM/Android manifests request
`squat.ai/kvm` while staying `privileged: false`. Replace `<node-name>` and
`<registry-credentials>` before applying (the image refs already point at the
in-cluster registry).

`workspace-local-path.yaml` is the **cluster-scoped** `workspace-local`
StorageClass plus its own `rancher/local-path-provisioner` (host path
`/home/develop/PVC`). It contains a `Namespace` / `ClusterRole` /
`ClusterRoleBinding` / `StorageClass`, so a restricted deploy tool (which filters
RBAC / cluster-scoped kinds) CANNOT apply it: apply it out-of-band (drop it into
the k3s `server/manifests/` directory, or `kubectl apply -f` with
cluster-admin). `generic-device-plugin.yaml` is likewise applied out-of-band.

The desktop / macOS / Windows manifests back the `linux` / `macos` / `windows`
sandboxes the second stack's worker-extension registers
(`charts/platform/values-standalone2.yaml`), so their `WORKER_TOKEN` MUST match
that `sandboxes` list. The macOS / Windows (and Android) manifests need a KVM
node plus the device plugin; fill `<node-name>` / `<registry-credentials>` before
applying.

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
