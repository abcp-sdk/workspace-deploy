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

Expected: **16 objects** (5 Deployments, 5 Services — incl. the webui alias,
1 ConfigMap, 1 Secret, 1 ServiceAccount, Role + RoleBinding, 1 PodDisruptionBudget).

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
- **Component images are pulled from ARTIFACT** (`artifact.worker.svc.cluster.local/abcp/...`) —
  `agent` / `workspace-extension` / `playwright-extension` / `workspace-gateway`
  / `workspace-webui`. Artifact is a plaintext-HTTP, insecure in-cluster
  registry and serves `/abcp/*` anonymously (pull works with no creds).
  `repo-build-image` pushes to the repo's own org (`coding-workspace/<name>`),
  so after a build you must copy each new tag into `abcp/` (e.g.
  `skopeo copy --dest-creds root:dev-artifact-token --dest-tls-verify=false
  docker://artifact.../coding-workspace/<name>:<tag> docker://artifact.../abcp/<name>:<tag>`).
  Sandbox images already come from artifact (`sandbox/*`).
- **Forgejo (`git.agent.svc.cluster.local`) stays for git + the build/push
  registry** (`infra.forgejo.url`, `infra.registry.host`) — that is the code
  source of truth and where builds push; only the *sandbox/component image
  source* moved to artifact. Migrating Forgejo itself is a separate project.
- **Verifying a registry tag:** the registry API requires auth — anonymous
  `GET /v2/...` returns 401. Use the Forgejo PAT from
  `charts/workspace/values.yaml` (`gateway.forgejo.token`):
  ```sh
  curl -u root:<pat> http://git.agent.svc.cluster.local/v2/abcp/workspace-gateway/tags/list
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
