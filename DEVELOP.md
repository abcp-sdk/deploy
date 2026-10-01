# DEVELOP

## Editing the chart

The chart is a plain Helm v3 chart. Render it locally before opening an MR:

```sh
helm template abcp-platform ./charts/platform -n worker
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
- **Default model**: `agent.defaultModel` seeds the tenant `default_model` via
  `AGENT_CONFIG_SEED` (create-if-absent) so a new session works on its first
  turn. Distinct from `AGENT_EXT_CONFIG_SEED` (that writes the extension `cfg`
  bucket; the default model lives in the `abcp-agent-config` KV). Requires an
  agent image that implements `AGENT_CONFIG_SEED`.
- **Upgrading a live release**: the infra secrets (`infra.nats.password`,
  `infra.s3.accessKey`, `infra.s3.secretKey`) and `providerApiKey` are NOT in
  `values.yaml`. Always `helm upgrade … --reuse-values` (or re-`--set` all of
  them), otherwise the upgrade resets them to `REPLACE_ME` and breaks the agent.
- **GUI worker endpoints**: to use the `computer-*` tools, set
  `workerExtension.sandboxes` to the standalone manifests under `worker-k8s/`
  (they live in the `worker` namespace: `agent-worker-desktop` /
  `agent-worker-macos` / `agent-worker-windows`). The manifests' `WORKER_TOKEN`
  and the `sandboxes` `token` MUST match or the sandbox answers 401.
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

## Package installs: use the in-cluster `artifact` mirror (do not hit the public internet)

Every package manager in a dev container / sandbox should fetch through the
shared **`artifact`** pull-through registry instead of the public internet. It
is an easy-vcs service in the `worker` namespace:

```
ARTIFACT=http://artifact.worker.svc.cluster.local
```

(Plain HTTP, anonymous **pull**. On a cache miss artifact fetches the upstream
once — through the cluster proxy — and caches it; later installs are local.)

This mirrors what `easy-vcs/easyops` does for freshly created sandboxes
(`internal/opssvc/upstream.go` + `bootstrap.go`): it injects a standard-env
subset into the sandbox Pod **and** runs a one-shot setup job that writes the
file-configured tools' configs. In an interactive container you do the same
thing by hand.

**Ownership (who configures the package sources):**

- **easy-vcs** — its `easyops` bootstrap configures the package sources for
  sandboxes *easyops creates*. Not ABCP's concern.
- **abc-protocol** — ABCP's own `agent-worker` sandboxes (the worker-extension
  drives a fixed easyworker) get their image / runtime config from
  **`abc-protocol/worker`** (`agent-toolchain/`, `sandbox-images/`). That is
  where an ABCP-side mirror config would go — **not** in this deploy repo.
- **This repo** (`abc-protocol/deploy`) — the `platform` chart deploys no
  package-manager config at all; it only wires the agent + extensions.
- The **shared `artifact`** itself (incl. its git-proxy fix) is transparent:
  both stacks benefit with no change.

### Environment variables (the "env half")

```sh
export ARTIFACT_UPSTREAM=http://artifact.worker.svc.cluster.local
export PIP_INDEX_URL=$ARTIFACT_UPSTREAM/artifacts/pypi/simple/
export PIP_TRUSTED_HOST=artifact.worker.svc.cluster.local
export NPM_CONFIG_REGISTRY=$ARTIFACT_UPSTREAM/artifacts/npm/
export GOPROXY=$ARTIFACT_UPSTREAM/artifacts/go
export GOSUMDB=off
export HEX_MIRROR=$ARTIFACT_UPSTREAM/artifacts/hex/
export PUB_HOSTED_URL=$ARTIFACT_UPSTREAM/artifacts/pub
```

### Config files (the "file half")

`easyops` writes these with a single idempotent script; run the equivalent
yourself (all writes overwrite, so re-running is safe). `$A` is the base URL:

```sh
A=http://artifact.worker.svc.cluster.local

# apt (Debian): artifact-only, drop the image's own sources
mkdir -p /etc/apt/sources.list.d
rm -f /etc/apt/sources.list /etc/apt/sources.list.d/debian.sources
echo "deb [trusted=yes] $A/artifacts/debian/debian trixie main" \
  > /etc/apt/sources.list.d/artifact.list

# apk (Alpine 3.24)
echo "$A/artifacts/apk/v3.24/main" > /etc/apk/repositories

# maven (java)
mkdir -p ~/.m2 && printf '<settings><mirrors><mirror><id>artifact</id><mirrorOf>*</mirrorOf><url>%s/artifacts/maven/</url></mirror></mirrors></settings>' "$A" > ~/.m2/settings.xml

# gradle. `allowInsecureProtocol = true` is REQUIRED (Gradle 7+ rejects
# plain-HTTP repos). The DEFAULT maven mount is Maven Central ONLY; Google Maven
# (androidx / com.android.tools.build / AGP) and the Gradle Plugin Portal are
# NAMED targets. THREE repositories, and the ORDER MATTERS — list maven.google /
# maven.gradle BEFORE maven (see gotcha): Gradle pins a plugin's JAR to the
# repository that resolved its marker, and a cold AGP jar is 404 on the default
# `maven`. Two blocks: `allprojects` for project deps + the plugin MARKER lookup;
# `settings.pluginManagement` for plugin JARs (its default is plugins.gradle.org).
mkdir -p ~/.gradle
cat > ~/.gradle/init.gradle <<GRADLE
def mirrors = [
  "$A/artifacts/maven.google/",   // dl.google.com/dl/android/maven2 (androidx/AGP) — FIRST
  "$A/artifacts/maven.gradle/",   // plugins.gradle.org/m2 (Gradle Plugin Portal)
  "$A/artifacts/maven/",          // repo.maven.apache.org/maven2 (Central) — LAST
]
allprojects {
  repositories {
    clear()
    mirrors.each { u -> maven { url u; allowInsecureProtocol = true } }
  }
}
settingsEvaluated { settings ->
  settings.pluginManagement.repositories {
    clear()
    mirrors.each { u -> maven { url u; allowInsecureProtocol = true } }
  }
}
GRADLE

# cargo (rust)
mkdir -p ~/.cargo && printf '[source.crates-io]\nreplace-with="artifact"\n[source.artifact]\nregistry="sparse+%s/artifacts/cargo/index/"\n' "$A" > ~/.cargo/config.toml

# rubygems
printf -- '---\n:sources:\n- %s/artifacts/rubygems/\n' "$A" > ~/.gemrc

# composer
mkdir -p ~/.composer && printf '{"repositories":{"packagist":{"type":"composer","url":"%s/artifacts/composer/"}}}' "$A" > ~/.composer/config.json

# nuget
mkdir -p ~/.nuget/NuGet && printf '<?xml version="1.0"?><configuration><packageSources><clear/><add key="artifact" value="%s/artifacts/nuget/v3/index.json"/></packageSources></configuration>' "$A" > ~/.nuget/NuGet/NuGet.Config

# conda
printf 'channels:\n  - %s/artifacts/conda/pkgs/main\n' "$A" > ~/.condarc

# git (CLI): rewrite github.com clones to the artifact mirror
git config --global url."$A/artifacts/git/github.com/".insteadOf https://github.com/

# Swift Package Manager: SPM uses libgit2 and IGNORES git's insteadOf, so it
# needs its own native mirror file (an OBJECT, no wildcards):
mkdir -p ~/.swiftpm/configuration
cat > ~/.swiftpm/configuration/mirrors.json <<JSON
{ "version": 1, "object": [
  { "original": "https://github.com/apple/swift-argument-parser.git",
    "mirror":   "$A/artifacts/git/github.com/apple/swift-argument-parser.git" }
] }
JSON
```

Per-protocol client setup (one-off commands, hosted-vs-upstream notes):
`easy-vcs/artifact` → `CLIENTS.md`.

### Verified in-cluster (2026-10-01, artifact `20261001-2`)

| Ecosystem | Config | Result |
|---|---|---|
| npm | `NPM_CONFIG_REGISTRY=$A/artifacts/npm/` | ✅ `npm install left-pad` |
| pub (Dart) | `PUB_HOSTED_URL=$A/artifacts/pub` | ✅ `dart pub get` resolved 11 deps |
| pip | `PIP_INDEX_URL=$A/artifacts/pypi/simple/` + `PIP_TRUSTED_HOST=…` | ✅ index 200 |
| Go | `GOPROXY=$A/artifacts/go` + `GOSUMDB=off` | ✅ 200 |
| Gradle/Maven | `$A/artifacts/maven/` + `allowInsecureProtocol = true`; **both** the `allprojects` and the `settings.pluginManagement` block | ✅ all downloads via artifact (plugin JARs too) |
| apt/Debian | `deb [trusted=yes] $A/artifacts/debian/debian trixie main` | ✅ `apt-get update` + install |
| git (large repos) | `$A/artifacts/git/github.com/…` | ✅ `apple/swift-log`, `apple/swift-nio` clone |
| SwiftPM | `~/.swiftpm/configuration/mirrors.json` | ✅ `swift package resolve` via artifact |

**Gotchas:**

- **Gradle** rejects plain-HTTP repositories unless you opt in — the
  `init.gradle` above sets `allowInsecureProtocol = true` (without it:
  "Using insecure protocols with repositories … is unsupported").
- **Gradle/Maven: the default `maven` mount is Maven Central ONLY**, and the
  repository ORDER matters. Google Maven (`androidx/*`, `com.android.tools.build`
  / AGP) is the NAMED target `$A/artifacts/maven.google/`; the Gradle Plugin
  Portal is `$A/artifacts/maven.gradle/`. Verified on a COLD cache: a fresh
  `androidx` coordinate → 404 via `maven/`, 200 via `maven.google/`. All maven
  mirrors share ONE cache, so after a pull through any target the others serve it
  (that is why the default can *look* like it works — the 404 shows only cold).
  **Put `maven.google` / `maven.gradle` BEFORE `maven`**: Gradle resolves a
  plugin's *marker* from the first repository that has it (the default `maven`
  proxies markers) and then pins that plugin's JAR to the SAME repository — where
  a cold AGP jar is 404 ("Could not find gradle-9.4.0.jar … Searched in … maven/
  com/android/tools/build/gradle/9.4.0/…"). Reordering fixed a full cold build.
- **Gradle plugins need the SECOND block**: `allprojects { repositories { … } }`
  only covers project deps + the plugin *marker* lookup; the plugin JARs resolve
  through `settings.pluginManagement`, whose default is `plugins.gradle.org`. Keep
  it in `~/.gradle/init.gradle` (not the repo's generated `settings.gradle.kts`,
  which should stay mirror-free). Verified against `agent-sdk-kotlin` (artifact
  `20261001-2`, cache wiped): without it `gradle build` still pulled ~30 artifacts
  (`kotlin-gradle-plugin-2.4.20-gradle96.jar`, …) from the public internet.
- **Swift Package Manager** does NOT honor `git config url.<base>.insteadOf`
  (SPM resolves with libgit2, not the `git` CLI); use the `mirrors.json` above.
  The `easy-vcs/easyops` bootstrap pre-seeds the common `apple/*` mirrors.
  (artifact `20261001-2` fixed its git proxy for large/many-ref repos — earlier
  `apple/swift-log` clones failed with `bad line length character`.)
- The mirror is **shared** and plain HTTP: `PIP_TRUSTED_HOST` / gradle's
  `allowInsecureProtocol` / apt's `[trusted=yes]` are the opt-ins for that.

### Verify it works

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://artifact.worker.svc.cluster.local/healthz          # 200
curl -s -o /dev/null -w '%{http_code}\n' http://artifact.worker.svc.cluster.local/artifacts/npm/left-pad
```

> The mirror is **shared** — pull is anonymous; a **push** needs a write-level
> token (never hard-code one). Point each repo at it so CI and interactive
> containers stop re-downloading the same packages from the public internet.
