# homelab-apps - Hosted in Gitea

Kubernetes application manifests for a personal k3s homelab cluster running on
three Raspberry Pi 4 Model B nodes.

## Relationship to the homelab infra repo

Cluster infrastructure — the k3s installation itself, MetalLB, Traefik, cert-manager,
and Argo CD — is managed by a separate private Ansible-based repository ("homelab").
This repo is purely for **application workloads**: things you deploy *onto* the cluster,
not things that constitute the cluster itself. If you are looking for load-balancer
configuration, ingress controller setup, or TLS issuers, they are not here.

## Directory layout

```
homelab-apps/
├── apps/        # One subdirectory per application workload
├── platform/    # Cluster-wide services that sit above raw infra but below
│                # individual apps: cert-manager ClusterIssuers, monitoring
│                # stack, security tooling (Falco, Trivy, Kyverno), etc.
├── demos/       # Ad-hoc lab experiments and blog post source material;
│                # throwaway by nature — nothing here is production
└── docs/        # Conventions, deployment guide, secrets handling
```

Each application under `apps/` and `platform/` lives in its own subdirectory and
deploys into its own dedicated Kubernetes namespace (see [Conventions](docs/conventions.md)).

## Conventions

| Topic | Rule |
|---|---|
| Namespaces | One namespace per workload, named after the workload (e.g. `app: grafana` → namespace `grafana`) |
| Labels | `app.kubernetes.io/name`, `/component`, and `/part-of` required on every object |
| Resource limits | Every container must declare `requests` and `limits` for both CPU and memory |
| Non-root | Containers must run as non-root where the upstream image supports it (`runAsNonRoot: true`) |
| Secrets | No real secrets ever committed — use `REPLACE_ME_BEFORE_DEPLOY` placeholders (see [Secrets](docs/secrets.md)) |
| Stateful workloads | Must include `nodeSelector: storage: ssd` to avoid landing on the microSD node |

Full details in [docs/conventions.md](docs/conventions.md).

## Deploying a workload (manual)

```bash
# Dry-run first — always
kubectl apply --dry-run=server -f apps/<workload>/

# Apply for real
kubectl apply -f apps/<workload>/

# Verify
kubectl -n <workload> get all
```

Replace any `REPLACE_ME_BEFORE_DEPLOY` placeholder values in Secret manifests before
applying. See [docs/secrets.md](docs/secrets.md) for the full workflow.

## GitOps via Argo CD

Argo CD watches this repository using the
[App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern)
pattern:

```
Ansible (homelab repo)
  └─ creates root Application → watches bootstrap/
       └─ bootstrap/<name>.yaml (child Application) → watches apps/<name>/
            └─ Kubernetes manifests auto-synced to the cluster
```

To deploy a new workload: create manifests in `apps/<name>/`, add a child
Application in `bootstrap/<name>.yaml`, commit, and push. Argo CD detects
the change and deploys automatically. See [docs/deployment.md](docs/deployment.md)
for the full workflow.

## Docs

- [docs/conventions.md](docs/conventions.md) — naming, labels, namespaces, resource limits
- [docs/secrets.md](docs/secrets.md) — how secrets are handled and why nothing real is committed
- [docs/deployment.md](docs/deployment.md) — step-by-step deployment guide (manual + future GitOps)

