# DEVELOP.md

Developer notes for this repo. It is a **deployment-assets repo**: a Helm chart
plus a few one-time manifests. There is no application code to build or test.

## Layout

```
charts/workspace/        Helm chart (release "workspace") — the main artifact
  templates/             one file per component; object names prefixed `workspace-`
  values.yaml            ALL tunables; the single source of deployment config
  system-presets.json    role presets, loaded via .Files.Get into a ConfigMap
provisioner/             cluster-scoped StorageClass provisioner (applied out-of-band)
manifests/               one-time / legacy objects (pre-Helm; see below)
```

## Rendering / verifying a change

The chart targets namespace `agent` via `namespaceOverride` (the release
namespace alone is not enough — every object hardcodes the helper):

```sh
helm lint ./charts/workspace --set namespaceOverride=agent
helm template workspace ./charts/workspace -n agent --set namespaceOverride=agent
```

Expected: **17 objects** (5 Deployments, 5 Services — incl. the webui alias,
1 ConfigMap, 2 Secrets (gateway secrets + the gateway DOCKER_CONFIG),
1 ServiceAccount, Role + RoleBinding, 1 PodDisruptionBudget).

Always render with `packageUpstream` set as well, since that branch is
conditional:

```sh
helm template workspace ./charts/workspace -n agent --set namespaceOverride=agent \
  --set gateway.sandbox.packageUpstream=http://artifact.worker.svc.cluster.local
```

## Conventions / gotchas

- **Two documents:** `README.md` is for users/deployers (what it deploys, how to
  install). Keep it current when behavior or deployment changes. This file is
  for developers.
- **Every object name is prefixed `workspace-`** so this release can share the
  `agent` namespace with the standalone `abcp-agent` release.
- **The `worker` namespace must pre-exist.** The chart only creates the Role /
  RoleBinding inside it (`templates/worker-rbac.yaml`,
  `gateway.workerRbac.create`), and only when `gateway.enabled`.
- **`gateway.sandbox.packageUpstream` empty = true no-op.** All three envs
  (`SANDBOX_PACKAGE_UPSTREAM` / `SANDBOX_BOOTSTRAP_IMAGE` / `SANDBOX_HOME`) are
  inside ONE `{{- if .Values.gateway.sandbox.packageUpstream }}` guard
  (`templates/gateway.yaml`). Keep them together: an unconditional env would
  change the render even when the feature is off.
- **Changing the package-source URL is a values change only — no image
  rebuild** (the gateway derives the bootstrap at pod-create time). But the
  feature needs a gateway image that implements it: the `gateway.image.tag` in
  `values.yaml` must be at or after `20261002-sandboxpkgsrc2`.
- **`packageUpstream` is ON** (`http://artifact.worker.svc.cluster.local`):
  every NEW sandbox gets the artifact package env + the bootstrap
  init-container. It only affects sandboxes created after the upgrade; existing
  ones keep their config. Set it back to `""` to disable.
- **Preset whitelists use the tool's OWN (self-namespaced) name** (e.g.
  `todo-write`, `sandbox-submit-mr`, `browser-navigate`) — see
  `abc-protocol/agent`'s `toolQualifiedName`, which uses the name as-is with
  only out-of-charset characters sanitized to `-` (`model.image` →
  `model-image`). `DISABLED_TOOLS` entries stay `<extId>.<name>` (a dot).
  `charts/workspace/system-presets.json` is a COPY of `workspace-gateway`'s
  `presets/system-presets.json` (regenerate there with `go run ./cmd/gen-presets`,
  then copy it here). **The agent image tag and the presets MUST ship
  together**: a mismatched pair (qualified ids against a bare agent, or vice
  versa) matches NO tools and every session loses its tool list.
  `agent.image.tag` must be at/after `20261004-4` (the bare-name revert).
- **Component images are pulled from ARTIFACT, namespaced by the OWNING REPO's
  org** (`artifact.worker.svc.cluster.local/<org>/<name>`):
  `abc-protocol/agent`, `abc-protocol/playwright-extension`,
  `coding-workspace/workspace-{extension,gateway,webui}`. The old `abcp/`
  namespace is LEGACY — do not add new refs to it. (Selenium/Caddy-style
  upstreams use a neutral namespace such as `library/`.) Artifact is a
  plaintext-HTTP, insecure in-cluster registry; `/abcp/*`, `/abc-protocol/*`
  and `/coding-workspace/*` all serve pulls anonymously (no creds needed).
  `repo-build-image` pushes to the repo's own org, so after a build you copy
  each new tag within artifact to the org the chart references (e.g.
  `skopeo copy --src-creds root:dev-artifact-token --dest-creds root:dev-artifact-token
  --src-tls-verify=false --dest-tls-verify=false
  docker://artifact.../abcp/<name>:<tag> docker://artifact.../coding-workspace/<name>:<tag>`).
  Sandbox images come from artifact (`sandbox/*`).
- **Forgejo (`git.agent.svc.cluster.local`) stays for git + the build/push
  registry** (`infra.forgejo.url`, `infra.registry.host`) — that is the code
  source of truth and where builds push; only the *sandbox/component image
  source* moved to artifact. Migrating Forgejo itself is a separate project.
- **Repushing the SAME tag does not update running nodes.** kubelet/containerd
  caches images keyed by tag, so re-pushing `sandbox-base:debian-trixie` (or any
  component tag) with new content can still resolve to the cached old digest on
  a node — `imagePullPolicy: IfNotPresent` then never pulls. To ship new content
  reliably, bump the TAG (e.g. `…:debian-trixie-v2`). After verifying the node
  cache was refreshed (e.g. confirm the new worker behaviour — `go version`
  works, i.e. `toolchain-install` is present), the `-v2` workaround tag can be
  dropped again (done in #18). When in doubt, keep the fresh tag.
- **Verifying a registry tag:** Forgejo (`git.agent...`) requires auth —
  anonymous `GET /v2/...` returns 401; use the PAT (`gateway.forgejo.token`).
  ARTIFACT (`artifact.worker...`) serves pulls anonymously:
  ```sh
  # Forgejo (auth):
  curl -u root:<pat> http://git.agent.svc.cluster.local/v2/coding-workspace/workspace-gateway/tags/list
  # Artifact (anonymous):
  curl -s http://artifact.worker.svc.cluster.local/v2/coding-workspace/workspace-gateway/tags/list
  ```
- **Credentials live IN THIS REPO (private, in-cluster).** Owner policy: the
  intranet repo is the store of record, so a redeploy never has to re-derive a
  secret. Deploy-time values (incl. secrets) go in `deploy-values.private.yaml`
  — pass it to `helm-deploy` (`--values @deploy-values.private.yaml`);
  `CREDENTIALS.md` is the index of every credential (endpoint, use, rotation,
  lost-key recovery). When you add a credential, record it in BOTH files. The
  chart's `values.yaml` keeps non-secret working defaults, but the private file
  is authoritative if they diverge.
- **`manifests/` is legacy.** `manifests/workspace-gateway.yaml` predates the
  chart (superseded by `templates/gateway.yaml`); `manifests/rbac.yaml` mirrors
  `templates/worker-rbac.yaml`. They are kept for the one-time migration only.
- **`provisioner/workspace-local-path.yaml` is cluster-scoped** and applied
  out-of-band (dropped into the k3s auto-apply manifests dir), not by Helm.
