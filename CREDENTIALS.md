# CREDENTIALS — workspace stack (`coding-workspace/deploy`)

This repo is a **private in-cluster repo**; credentials are stored here on
purpose (owner policy: everything is intranet-only) so a redeploy never depends
on re-deriving them. For an actual deploy, pass `deploy-values.private.yaml`
(secrets) to `helm-deploy`; **this file is the INDEX of every secret** so none
is forgotten. Rotate deliberately — several of these invalidate live sessions.

## 1. Shared infra consumed over Service DNS (the `infra` release, `agent` ns)

| Service | Endpoint | Credentials |
|---|---|---|
| NATS (JetStream) | `nats.agent.svc.cluster.local:4222` | account `workspace` / `devpassword` — this stack's dedicated account |
| Garage S3 (agent blobs) | `http://s3.agent.svc.cluster.local` (region `garage`, bucket `workspace`, path-style) | access `GK66c166f3602ec6b5e762b21c` / secret `685de403…da58c22` |
| Postgres (agent metadata) | `postgres.agent.svc.cluster.local:5432` (db `workspace_agent`) | user `workspace` / `devpassword` |
| Forgejo (git) | `http://git.agent.svc.cluster.local` | PAT `8a582a2d…e4b3039` (root) |
| buildkitd (image builds) | `tcp://buildkitd.agent.svc.cluster.local:1234` | (none; in-cluster) |
| Selenium (playwright) | `http://selenium.agent.svc.cluster.local:4444` | (none) |
| outbound proxy (mihomo) | `http://mihomo.develop.svc.cluster.local:7890` | (none) |

> The `worker`-ns services below are the **easy-vcs/deploy** shared infra. The
> chart currently points at the `agent`-ns ones; migrating the workspace stack
> to the `worker`-ns services (as the owner asked) is tracked separately — see
> `DEVELOP.md`.

| Service | Endpoint | Credentials |
|---|---|---|
| NATS (worker ns) | `nats.worker.svc.cluster.local:4222` | accounts `abcp-agent` / `easyvcs` |
| Garage admin API (worker ns) | `http://garage.worker.svc.cluster.local:443` | bearer `adminsecret-token-0123456789` |
| Garage S3 (worker ns) | `http://garage.worker.svc.cluster.local:80` | buckets `abcp-agent` / `sandbox` |
| Postgres (worker ns) | `postgres.worker.svc.cluster.local:80` (→5432) | `root` / `devpassword` |
| artifact (packages + OCI + sandbox images) | `http://artifact.worker.svc.cluster.local` | read = **anonymous**; write token = `dev-artifact-token` |

## 2. workspace-gateway secrets (Secret `workspace-gateway-secrets`)

| key | value | used by |
|---|---|---|
| `forgejo-token` | `8a582a2d295a7d19f089b449bc29e5e4e34b3039` | repo tools + EnsureRepo/protect |
| `service-token` | `devservice-token` | workspace-extension → gateway sandbox RPCs |
| `agent-admin-token` | `dev-admin-token` | gateway → agent AdminService (mint tenant tokens) |
| `brave_api_key` (calibration) | `BSAnI5ZDD4HltcHrC3ovDHGKyPLu7Y_` | bundled brave-search tool |

## 3. workspace-agent auth

| item | value |
|---|---|
| admin token | `dev-admin-token` |
| bootstrap tenant | `workspace` |
| bootstrap token | `devworkspacetoken` |

## 4. Model gateway providers (`gateway-*` capabilities)

Shared apiKey `sk-code-dAeG7zpfYuusoIA0Z9LYumE6oDb7BNArqxTkk4th37lYtzRokkkvoY5Y`
on `https://api-gray.xueersi.com/ai-multimodal-gateway/v4/ai` (text / image /
speech / transcription / embedding / rerank / realtime). Recorded in
`deploy-values.private.yaml` under `gatewayProviders`.

> Caveat: the workspace chart does not yet consume this key (it does not seed
> model providers). Wire it into a chart value before relying on it.

## 5. Rotation

| credential | how to rotate |
|---|---|
| Garage S3 key (agent) | `POST http://<garage-admin>/v2/CreateKey {"name":"workspace-deploy"}` then `POST /v2/AllowBucketKey` (bucket `workspace`, read+write+owner) → update `deploy-values.private.yaml` → `helm upgrade`. |
| NATS `workspace` password | edit the account in the infra `nats.conf` → update here + `infra.nats.password`. |
| Forgejo PAT | Forgejo → Settings → Applications → regenerate → update here. |
| brave_api_key | reissue from Brave → update here. |
| gateway/agent tokens | change in `deploy-values.private.yaml` → `helm upgrade` (invalidates existing sessions). |
| provider apiKey | request a new key from the gateway operator → update here. |

## 6. Lost-key recovery

Garage admin API (`ListKeys` / `ListBuckets` / `CreateKey` / `AllowBucketKey`)
rebuilds a lost S3 key non-destructively; the **admin token above is the
recovery root**, so keep it here even if nothing else.
