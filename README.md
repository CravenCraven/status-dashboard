# status-dashboard

A small, read-only status page for my self-hosted Kubernetes homelab. It shows live totals from the cluster (nodes, pods, CPU, memory, restarts, app health) and nothing else.

Live at **https://status.projectpattie.com** (public, no login, on purpose: it only reads, and it only shows totals).

> **Status: in progress.** The page and API are being built now. Container image, CI and deployment come next. Sections below marked *(planned)* are not done yet.

Part of my homelab, which runs on three repurposed laptops with k3s and is managed with Flux (GitOps). The cluster and its manifests live in [homelab-kubecraft](https://github.com/CravenCraven/homelab-kubecraft). This repo holds only the app code.

---

## What it shows

- Nodes ready
- Pods running
- CPU % and memory % across the cluster
- 7-day availability
- Restarts in the last 24 hours
- A node table
- Health per app (healthy, degraded, planned)

App names, descriptions and the "planned" cards come from a config file in this repo (`apps.yaml`), not from the cluster. Moving an app from planned to live is a one-line edit.

Design: Brutalist, black and magenta.

## How it works

```
Visitor's browser
      |
      v
status.projectpattie.com  --> Cloudflare Tunnel --> Traefik --> status-dashboard (FastAPI)
                                                                     |
                                                  +------------------+------------------+
                                                  v                                     v
                                          Prometheus (in cluster)            Kubernetes API (in cluster)
```

1. The page is plain HTML and JavaScript served by the app.
2. The page calls one endpoint, `GET /api/status`, on load and every 30 seconds.
3. The backend asks Prometheus for the numbers (CPU, memory, pod and restart counts, app health) and the Kubernetes API for the node list.
4. The result is cached for 30 seconds so a busy page can't hammer Prometheus.
5. If a fetch fails, the page shows STALE instead of blank numbers.

Only the page and `/api/status` leave the cluster. Prometheus and the Kubernetes API stay internal.

## Security model

This page is public, so it is locked down before it is exposed.

- **Read-only.** The app runs with a ServiceAccount that can only `get` and `list` nodes and pods. No secrets, no deleting, no writing.
- **GET-only routes.** Anything else returns 405.
- **Totals only.** The JSON leaves out IP addresses, pod names and anything not listed in `apps.yaml`.
- **No credentials in this repo.** Nothing secret lives here, and nothing ever should. Secrets for the cluster are SOPS-encrypted in the homelab repo.
- **Non-root container** with a read-only filesystem *(planned)*.

## Stack

| Piece | Choice |
|---|---|
| Language | Python 3.11+ |
| Web framework | FastAPI, served with uvicorn |
| Metrics | Prometheus HTTP API (from kube-prometheus-stack) |
| Cluster data | Kubernetes Python client |
| Image registry | GitHub Container Registry, public *(planned)* |
| Architectures | linux/amd64 and linux/arm64, because the cluster mixes Intel and Apple Silicon laptops *(planned)* |
| Deploy | Flux, from the homelab-kubecraft repo *(planned)* |

## Repo layout

*(Target layout. Files appear as each block is built.)*

```
status-dashboard/
├── docs/
│   └── data-contract.md     # every number on the page, and where it comes from
├── static/
│   └── index.html           # the page
├── main.py                  # FastAPI app
├── apps.yaml                # app names, descriptions, namespaces, planned apps
├── mock-status.json         # example response shape
├── requirements.txt
├── Dockerfile               # (planned)
└── .github/workflows/       # image build on version tags (planned)
```

## Configuration

| Variable | Purpose |
|---|---|
| `PROM_URL` | Where to reach Prometheus. `localhost` on my Mac (via port-forward), the in-cluster address in the Deployment. |

The Kubernetes client tries in-cluster config first and falls back to my local kubeconfig, so the same code runs on my Mac and inside the pod.

## Run locally *(planned)*

Steps will be filled in once the backend runs. In short: create a virtual environment, install `requirements.txt`, port-forward Prometheus to `localhost:9090`, and start uvicorn.

## Roadmap

- [x] Plan, design and data contract
- [ ] Backend: `/api/status` with real numbers
- [ ] Frontend: live page
- [ ] Container image and CI (multi-architecture)
- [ ] Deploy with Flux and expose through the Cloudflare Tunnel
- [ ] Flux sync status on the ticker
- [ ] Renovate watching the image
- [ ] Runbook and write-up on [projectpattie.com](https://projectpattie.com)

## Related

- [homelab-kubecraft](https://github.com/CravenCraven/homelab-kubecraft): the cluster, manifests and GitOps setup this page reports on
- [projectpattie.com](https://projectpattie.com): write-ups of what I build and what broke
