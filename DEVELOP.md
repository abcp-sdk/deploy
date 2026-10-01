# DEVELOP

## Editing the charts

Every chart is a plain Helm v3 chart. Render it locally before opening an MR:

```sh
helm template platform  ./charts/platform  -n agent
helm template infra     ./charts/infra     -n agent
```

There is no cluster access requirement to render. `helm lint` is optional.

## Conventions

- **One chart per stack**, each self-contained (`Chart.yaml` + `values.yaml` +
  `templates/`). The `platform` chart renders a ServiceAccount and depends on the
  shared `infra` release over Service DNS; it never deploys infra itself.
- **Namespace**: `.Values.namespaceOverride | default .Release.Namespace`. Every
  template renders into the release namespace.
- **Image refs**: `<registry.host>/<namespace>/<repo>:<tag>`, tags pinned in
  `values.yaml`. The infra chart uses upstream image names directly.
- **Secrets in values**: the dev cluster commits dev credentials in `values.yaml`
  (matching the pre-existing `agent` chart). A production values file must
  override them; never commit a real credential.
- **Restricted deploy tools**: keep RBAC and privileged/hostPath kinds out of the
  default release path (see README). The platform chart ships only a
  namespaced ServiceAccount.

## Migrating a chart change back to a code repo

If a chart template still references a component's internals, prefer fixing the
template here over re-adding k8s/ to the code repo. The code repo's README
"Deploy" section should point at this repo.
