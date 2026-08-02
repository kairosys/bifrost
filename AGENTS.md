# AGENTS.md

This workspace ships **no application code**: only `k8s/` manifests and generated runtime data under `data/`. No build/lint/test/typecheck/Package.json exists (npm/go/make do nothing here). Bifrost is a MaxKB-style OpenAI-compatible proxy on a Kind cluster; config + logs persist to SQLite files mounted via hostPath.

## Layout
- `k8s/bifrost-deployment.yaml` — single source of truth: Service (`port`/`targetPort 8080`), Deployment (image `maximhq/bifrost:latest`, 1 replica), Ingress (`bifrost.localhost`). No namespace is declared in this file; resources land in whatever namespace the kubectl context targets.
- `data/` — ignored by `.gitignore`; holds runtime SQLite: `config.db` (~26 MB) + WAL/SHM sidecars; `logs.db` (currently ~189 MB on host); and a kept `logs/` dir. All state is on each cluster node, not in this repo.
- `.gitignore` — ignores `data/`, `k8s/*-secret.yaml`, `*.log`.

## HostPath mapping (critical)
Container mounts `/app/data`; host directory is created by Kubernetes at:
```yaml
volumes: [{name: bifrost-data, hostPath: {path: /mnt/workspaces/bifrost/data, type: DirectoryOrCreate}}]
```
- **New Kind nodes will not have this path populated.** Pre-seed/create `/mnt/workspaces/bifrost/data` before first `kubectl apply`, otherwise the gateway pod has no place to write config/logs.
- Back up from `/mnt/workspaces/bifrost/data` on each host — never here (deleted files are runtime-generated only).

## Deploy flow (default namespace unless your kubectl context overrides)
```bash
kubectl apply -f k8s/bifrost-deployment.yaml         # creates Service + Deployment + Ingress
# wait for the Deployment to come up:
kubectl rollout status deployment/bifrost --timeout=90s
```
Changing the Service `port`/`targetPort` requires updating the Ingress `service.port.number` and container `containerPort`/`name: http` together.

## Safe test/verification commands
```bash
# Reach gateway in-cluster (no backend needed to confirm Bifrost answers):
kubectl run -i --rm debug-bifrost --image=curlimages/curl --restart=Never -- \
  sh -c 'curl -fsS http://bifrost.default.svc.cluster.local:8080/v1/models'

# Local check without touching ingress host:
kubectl port-forward svc/bifrost 8080:8080 & curl -s http://localhost:8080/v1/models
```

## Data hygiene (where agents get paged)
- SQLite: `data/*.db*` are locked by the running pod. Inspect them only with the pod scaled to zero, or copy out first via `kubectl cp` — do not open `-wal`/`-shm` while live.
- Growth: `logs.db` grows unbounded on host disk (`/mnt/workspaces/bifrost`); if a node's free space is low after changes, prune it from `/mnt/workspaces/bifrost/data/logs*` (not here) and restart the pod — no in-repo tool tracks this.

## Environment / secrets
None present yet. To add backend URL/env vars: edit `containers[].env` and apply; reference a Secret under `k8s/*-secret.yaml` (gitignored). Do **not** expect a `.env` loader — none exists in the image or manifest.

## Conventions that differ from defaults
No readiness/liveness probes are defined, so Kubernetes reports an unhealthy container as healthy until it exits. Add explicit probes before production traffic (there is no existing health endpoint wired here).