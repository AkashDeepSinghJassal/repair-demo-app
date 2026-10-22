# repair-demo-app

GitOps target for the repair-loop prototype. Argo CD syncs `deploy/demo` (namespace `demo`) and `deploy/demo-b` (namespace `demo-b`). `workflows/` holds the Argo Workflows `WorkflowTemplate` `release-pipeline`.

Version: see `VERSION`. Every workload carries the annotation `repair.example.dev/source-path` with its own manifest path.

## Services (`demo`)

| Service | Image | Port | What it does |
|---|---|---|---|
| `redis` | `redis:7-alpine` | 6379 | Key-value store, capped at 32 MB. Reachable from `api` and `worker` only. |
| `api` | `python:3.12-alpine` | 8000 | Stdlib `http.server` app from ConfigMap `api-code`. Endpoints: `/healthz`, `/version`, `/items` (in-memory counter). Holds ~60 MiB at start. Exits 1 with `DB_URL not set` if `DB_URL` is missing (from ConfigMap `api-config`). Limit 128Mi. |
| `worker` | `python:3.12-alpine` | none | Logs a heartbeat every 10 seconds (ConfigMap `worker-code`). |
| `frontend` | `nginx:1.27-alpine` | NodePort 30080 | Serves `index.html` (ConfigMap `frontend-html`) and proxies `/api/` to `api:8000`. |

NetworkPolicies in `demo`: default-deny ingress, then allow frontend to api, workflow pods to api (for the smoke test), api and worker to redis, and anywhere to frontend port 80.

## Services (`demo-b`)

`api-b` is a second copy of `api` with its own ConfigMaps (`api-b-code`, `api-b-config`) and the same 128Mi limit.

## Workflow

`release-pipeline` (namespace `demo`) runs three steps: `check-config` (checks the `mode` parameter, default `strict`, and exits 3 on an unknown mode), `smoke-test` (GET `http://api.demo.svc:8000/healthz`), and `report`.

## Constraints

- Public, multi-arch (arm64) images only. No image builds; app code lives in ConfigMaps.
- Manifests target Kubernetes 1.31. Total memory requests are under 600Mi.
- No secrets in this repo.
