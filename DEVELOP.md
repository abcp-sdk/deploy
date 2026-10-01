# DEVELOP

## Editing the chart

The chart is a plain Helm v3 chart. Render it locally before opening an MR:

```sh
helm template abcp-platform ./charts/platform -n worker
# the second stack:
helm template abcp-agent-s2 ./charts/platform -n worker \
  -f ./charts/platform/values-standalone2.yaml
```

There is no cluster access requirement to render. `helm lint` is optional.

## Conventions

- **One chart** (`platform`), self-contained (`Chart.yaml` + `values.yaml` +
  `templates/`). It renders a ServiceAccount, the agent + its extensions, and a
  self-deployed Selenium node. It does NOT deploy NATS / Garage / Postgres —
  those are the SHARED `worker`-namespace services owned by `easy-vcs/deploy`
  and consumed over Service DNS.
- **Release name**: install as `abcp-platform`, NOT `platform` — `platform` is
  the easy-vcs stack's release name in this cluster, and Helm would upgrade it
  in place. The deploy tool (`helm-deploy`) also manages other tenants'
  releases, so always check `helm-list` before choosing a name.
- **No Forgejo, no buildkitd.** The standalone agent needs neither. Do not
  re-add them.
- **Namespace**: `.Values.namespaceOverride | default .Release.Namespace`. Every
  template renders into the release namespace (the shared infra is cross-
  namespace and addressed by its Service DNS, e.g. `*.worker.svc.cluster.local`).
- **Shared-infra names are ABCP-specific**: the NATS account, S3 bucket and
  Postgres database must be distinct from every other stack on the shared
  services (see README). The password / access key are supplied at install time,
  not committed.
- **Image refs**: `<registry.host>/<namespace>/<repo>:<tag>`, tags pinned in
  `values.yaml`. Selenium uses the upstream image name directly.
- **Selenium**: the URL the playwright extension uses comes from the
  `abcp-agent.seleniumUrl` helper — an explicit `.Values.selenium.url` wins,
  otherwise the in-chart Service. Set `selenium.enabled=false` only if you point
  `selenium.url` at an external node.
- **No committed credentials**: the shared NATS password / S3 keys and the
  gateway `providerApiKey` are placeholders (`REPLACE_ME`) passed at install
  time. `providerApiKey` is injected by the `abcp-agent.providersSeed` helper
  into every provider in `.Values.providers` that has no `apiKey` of its own.
- **GUI worker endpoints**: `values-standalone2.yaml`'s `workerExtension.sandboxes`
  points the linux/macOS/windows entries at the standalone manifests in
  `worker-k8s/`, which live in the `worker` namespace
  (`agent-worker-desktop` / `agent-worker-macos` / `agent-worker-windows`).
  Keep those in sync if the manifests move. The manifests' `WORKER_TOKEN` and
  the `sandboxes` `token` MUST match (`devdesktop-token` / `devmac-token` /
  `devwin-token`) or the sandbox answers 401.
- **Restricted deploy tools**: keep RBAC and privileged/hostPath kinds out of the
  default release path (see README). The platform chart ships only a namespaced
  ServiceAccount.

## Renaming an object leaves an orphan

Helm only tracks the objects in the CURRENT revision; renaming a Deployment/
Service (or rolling back past a rename) leaves the OLD object running in the
cluster, owned by nobody. It keeps answering on its Service DNS, which is
confusing. After any rename, check and delete the stale objects:

```sh
kubectl -n worker get deploy,svc -l app.kubernetes.io/name=abcp-agent
# delete any name that is not in the current `helm get manifest abcp-platform`
```

## Migrating a chart change back to a code repo

If a chart template still references a component's internals, prefer fixing the
template here over re-adding k8s/ to the code repo. The code repo's README
"Deploy" section should point at this repo.
