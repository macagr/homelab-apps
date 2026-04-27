# Conventions

Standards that apply to every manifest in this repository. Following them makes
workloads predictable, safe to deploy, and easy to operate.

---

## Namespace strategy

Every workload gets its own dedicated namespace named after the workload itself.

```
workload name: grafana   →   namespace: grafana
workload name: vaultwarden → namespace: vaultwarden
```

Rationale: namespace-scoped RBAC and NetworkPolicy are far easier to reason about
when each workload is isolated. It also makes `kubectl -n <workload> get all` the
one command you always reach for.

Do not reuse namespaces across workloads. Do not use `default`.

---

## Label conventions

Every Kubernetes object (Deployment, Service, ConfigMap, etc.) must carry the
following labels:

| Label | Value | Notes |
|---|---|---|
| `app.kubernetes.io/name` | workload name, e.g. `grafana` | The workload itself |
| `app.kubernetes.io/component` | component role, e.g. `server`, `agent`, `exporter` | Omit only if the workload is a single component |
| `app.kubernetes.io/part-of` | parent app name if component is a sub-piece | Same as `name` when the workload stands alone |

Example:

```yaml
labels:
  app.kubernetes.io/name: grafana
  app.kubernetes.io/component: server
  app.kubernetes.io/part-of: grafana
```

These labels are used by dashboards, monitoring, and will eventually drive Argo CD
sync status views.

---

## Resource limits

Every container must declare both `requests` and `limits` for CPU and memory. There
are no exceptions. An unset limit is a potential DoS vector on a three-node Pi cluster.

Example values for a lightweight workload:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Guidelines:
- Start conservative and tune upward based on observed usage.
- Avoid CPU limits that are orders of magnitude above requests — on a Pi this causes
  real throttling.
- Memory limits must be set; a container hitting its memory limit is OOM-killed
  (expected and safe); one with no limit can take down a node.

---

## Non-root containers

Containers must run as non-root where the upstream image supports it.

Minimum securityContext on every container:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

If the upstream image requires root (e.g. some legacy containers), document why in a
comment alongside the manifest and consider whether a rootless alternative exists.

Recommended pod-level defaults to set in addition:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

---

## Stateful workloads

Workloads that need persistent storage must include a `nodeSelector` that pins them
to a node with SSD storage. The home nodes boot from USB HDDs (pis) or SD/eMMC
(odroids) — neither is appropriate for write-heavy persistent state. Only the cloud
node (node-4) has SSD storage.

```yaml
nodeSelector:
  storage: ssd
```

This selector must appear on every `Pod` spec (directly or via a Deployment/
StatefulSet template) that mounts a PersistentVolumeClaim. In the current cluster
this resolves to node-4 (Hetzner) — accept the Tailscale latency or redesign the
workload to be stateless.

The `storage` label is applied per-node by the Ansible `k3s_labels` role. Verify
with `kubectl get nodes -L storage` before deploying.

---

## App of Apps pattern

Workloads are deployed via the Argo CD
[App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern)
pattern. A single **root Application** (created by Ansible) watches the
`bootstrap/` directory. Each file there is a child Argo CD Application
pointing at a workload directory under `apps/`, `platform/`, or `demos/`.

### Directory mapping

| Directory | Contains | Managed by |
|-----------|----------|------------|
| `bootstrap/` | Child Argo CD `Application` manifests (one per workload) | Root Application |
| `apps/<name>/` | Workload Kubernetes manifests | Child Application in `bootstrap/<name>.yaml` |
| `platform/<name>/` | Platform service manifests | Child Application in `bootstrap/<name>.yaml` |
| `demos/<name>/` | Demo/experiment manifests | Child Application in `bootstrap/<name>.yaml` |

### Naming conventions

- **File name:** `bootstrap/<workload>.yaml` (matches the workload directory name)
- **Application name:** `metadata.name` matches the workload name (e.g. `miniflux`)
- **Namespace:** `spec.destination.namespace` matches the workload name
- All child Applications live in the `argocd` namespace (`metadata.namespace: argocd`)

### Sync policy

All Applications (root and children) use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

- **prune:** Argo CD deletes resources removed from git
- **selfHeal:** Argo CD reverts manual `kubectl` changes to match git

Child Applications use `CreateNamespace=true` (the namespace is part of the
workload). The root Application uses `CreateNamespace=false` (the `argocd`
namespace already exists).
