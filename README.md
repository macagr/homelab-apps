# homelab-apps - Hosted in Gitea

Kubernetes application manifests for a personal k3s homelab cluster running on
three Raspberry Pi 4 Model B nodes.

> **Note:** This repo is a learning exercise. The goal is to practice the
> architectural patterns senior platform engineers use to run real clusters —
> GitOps, secret management, progressive delivery, observability — at a scale
> small enough to fit on three Pis. Expect opinionated choices and occasional
> over-engineering in the name of getting the reps in.

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

### The three deployment patterns in use

Not every workload needs the same packaging. This repo intentionally uses three
different patterns, matched to the complexity and ownership of each app. Picking
the right pattern for each workload is part of the learning goal.

#### 1. Raw manifests — [apps/homepage/](apps/homepage/)

Plain Kubernetes YAML (Deployment, Service, ConfigMap, Gateway, HTTPRoute, etc.)
with no Chart.yaml or templating. Argo CD applies the files as-is.

- **Use when:** the app is simple, single-environment, and has no dynamic config
  worth templating.
- **Why here:** the homepage is a static nginx container serving one HTML page.
  Helm would add indirection without buying anything.

#### 2. Umbrella chart — [apps/gitea/](apps/gitea/)

A local `Chart.yaml` declares an upstream chart as a dependency and a local
`values.yaml` overrides it. Argo CD renders the combined chart at sync time.

- **Use when:** deploying a complex third-party app (databases, CI/CD, monitoring
  stacks) where you need Helm's configurability but don't own the source.
- **Why here:** the upstream Gitea chart has hundreds of knobs. Pinning the
  version in [Chart.yaml](apps/gitea/Chart.yaml) and keeping overrides in
  [values.yaml](apps/gitea/values.yaml) gives reproducible, reviewable config.

#### 3. Remote blueprint — [apps/k8bot/](apps/k8bot/)

The Argo Application here points to a Helm chart living in the app's **own**
source repo ([git.mcagr.com/macagr/k8bot.git](https://git.mcagr.com/macagr/k8bot.git),
path `deploy/helm/k8bot`). This repo only holds the Argo Application manifest,
the namespace, and the ESO SecretStore wiring.

- **Use when:** deploying a first-party app you wrote yourself.
- **Why here:** k8bot is custom Go code. Keeping its Helm chart next to the Go
  source means a single PR can update code *and* deployment shape together —
  no cross-repo dance.

## Docs

- [docs/conventions.md](docs/conventions.md) — naming, labels, namespaces, resource limits
- [docs/secrets.md](docs/secrets.md) — how secrets are handled and why nothing real is committed
- [docs/deployment.md](docs/deployment.md) — step-by-step deployment guide (manual + future GitOps)

