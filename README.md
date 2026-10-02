# coding-workspace/deploy

Deployment assets for the **workspace stack** (the `coding-workspace/*`
components), moved here from `workspace-gateway/k8s/` so the whole
coding-workspace chart lives in one deploy repo.

## Layout

```
charts/workspace/        the Helm chart (release "workspace")
provisioner/             cluster StorageClass (workspace-local, local-path)
manifests/               one-time / legacy objects
  rbac.yaml              cross-namespace RBAC (worker namespace Role/RoleBinding)
  workspace-gateway.yaml legacy hand-applied gateway objects (pre-Helm)
```

## What `charts/workspace` deploys

- `workspace-agent` — a dedicated agent (h2c) that loads the workspace role
  presets (`SYSTEM_PRESETS_FILE`), bootstraps the workspace tenant, and seeds
  the extensions' config into its own cfg KV.
- `workspace-extension` — `sandbox-*` / `repo-*` tools (NATS only, no Service).
- `workspace-playwright` — browser-automation extension (drives infra Selenium).
- `workspace-gateway` — the trusted session-creation + sandbox/service lifecycle
  surface (`workspace.v1`); manages objects in the `worker` namespace.
- `workspace-webui` — Caddy aggregator (static SPA + same-origin RPC).

Shared infrastructure is consumed over Service DNS from the separate `infra`
release (NATS, Garage S3, Forgejo, buildkitd, Selenium). Nothing infra-ish is
deployed here.

## Install

```sh
# The worker namespace must exist first (the chart only creates RBAC inside it).
kubectl create namespace worker

helm install workspace ./charts/workspace -n agent --set namespaceOverride=agent
```

See `charts/workspace/README.md` for prerequisites (the `workspace` NATS
account, image build/push) and the one-time cleanup of pre-Helm objects.

## Package source (sandboxes)

Set `gateway.sandbox.packageUpstream` to the in-cluster registry so every new
sandbox fetches packages from it instead of the public internet:

```sh
helm upgrade workspace ./charts/workspace -n agent \
  --set gateway.sandbox.packageUpstream=http://artifact.worker.svc.cluster.local
```

Empty (default) = no injection. Changing the URL is a values change only — no
image rebuild. Implemented by the gateway (env + a bootstrap init-container);
see `workspace-gateway`'s `internal/sandboxbootstrap`.

