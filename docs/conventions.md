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
to nodes with SSD storage. The cluster has one node whose boot media is a microSD
card; landing stateful workloads there leads to data loss and card wear.

```yaml
nodeSelector:
  storage: ssd
```

This selector must appear on every `Pod` spec (directly or via a Deployment/
StatefulSet template) that mounts a PersistentVolumeClaim.

Nodes with SSDs should have this label applied via the Ansible homelab repo. Verify
with `kubectl get nodes --show-labels` before deploying.
