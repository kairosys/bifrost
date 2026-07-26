# AGENTS.md

This is a local workspace for **Bifrost** (a MaxKB-style proxy service). The repository ships no source code, build scripts, or tests — it contains only generated runtime data and a Kubernetes deployment manifest.

## Layout

- `data/` — generated at runtime; SQLite files (`config.db`, `logs.db`) plus `-wal`/`-shm` sidecars. **Never edit by hand while the service is running.**
  - `.gitignore` ignores everything under `data/`, so DB changes are not tracked and will be lost on redeploy.
  - The empty `data/logs/` directory (owned by `root:root`, mode `0755`) must remain an accessible directory for log output.
- `k8s/bifrost-deployment.yaml` — single Deployment + Service + Ingress in the `furseal` namespace; container port **8080**, image `maximhq/bifrost:latest`.

## Data path mapping (important)

The container mounts a hostPath volume:

```yaml
volumeMounts: [{name: bifrost-data, mountPath: /app/data}]
volumes:  [{name: bifrost-data, hostPath: {path: "/mnt/workspaces/bifrost/data", type: DirectoryOrCreate}}]
```

- A new cluster node will not have `/mnt/workspaces/bifrost/data` populated. Either pre-seed that path or create the directory before first `kubectl apply`.
- All persisted state (config + logs) lives under `/mnt/workspaces/biftrast...`/`data` on each host — back it up there, **not** from inside this repo.

## What to change and how

| Need | Where | Notes |
|------|-------|-------|
| Deploy changes   | `k8s/bifrost-deployment.yaml` then `kubectl apply -f k8s/` | Image tag is `:latest`; pin a digest if you need reproducibility instead of the moving tag. |
| Change port      | container + readiness/liveness probes (none defined) and the Service `port`/`targetPort`. The Ingress assumes 8080 — update it too. |
| Environment vars / secrets | not present yet — add under `containers[].env` or a referenced Secret. No `.env` file is loaded by this repo alone; this repo does **not** contain app code to edit. |

## Gotchas for agents

- There are no tests, lint, typecheck, build, or package manifests in this workspace. Commands like `npm install`, `go test`, or `make dev` do nothing here.
- `data/*.db*` grow unbounded; a large workspace may exhaust disk on the node before hitting project limits. Watch `/mnt/workspaces/bifrost` free space.