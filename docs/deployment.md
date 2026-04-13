# Deployment guide

How to get workloads onto the cluster.

---

## Current workflow — manual `kubectl`

Workloads in this repo are currently applied manually. There is no CI pipeline
that does this automatically.

### Prerequisites

- `kubectl` configured to talk to the homelab cluster (`~/.kube/config` or
  `KUBECONFIG` env var pointing at the right context)
- `kubectl config current-context` should show the homelab cluster

### Steps

1. **Create the namespace** (first deploy only):

   ```bash
   kubectl create namespace <workload>
   ```

2. **Populate secrets** with real values before applying (see [secrets.md](secrets.md)).
   Never apply a manifest that still contains `REPLACE_ME_BEFORE_DEPLOY`.

3. **Dry-run — always do this first**:

   ```bash
   kubectl apply --dry-run=server -f apps/<workload>/
   ```

   Fix any errors before proceeding.

4. **Apply**:

   ```bash
   kubectl apply -f apps/<workload>/
   ```

5. **Verify**:

   ```bash
   kubectl -n <workload> get all
   kubectl -n <workload> describe pod   # if anything looks wrong
   kubectl -n <workload> logs <pod>     # check application logs
   ```

### Updating a workload

Same steps as above — `kubectl apply` is idempotent. Always dry-run first.

### Deleting a workload

```bash
kubectl delete -f apps/<workload>/
# then optionally:
kubectl delete namespace <workload>
```

---

## GitOps workflow — Argo CD App of Apps

Argo CD watches this repository and automatically deploys workloads using the
App of Apps pattern. No manual `kubectl apply` is needed for normal operations.

### Adding a new workload

1. **Create the workload directory** with Kubernetes manifests:

   ```
   apps/<name>/
   ├── namespace.yaml
   ├── deployment.yaml
   ├── service.yaml
   └── ...
   ```

   Follow [conventions.md](conventions.md) for namespaces, labels, resource
   limits, and secrets.

2. **Create the child Application** in `bootstrap/<name>.yaml`:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <name>
     namespace: argocd
     finalizers:
       - resources-finalizer.argocd.argoproj.io
   spec:
     project: default
     source:
       repoURL: https://github.com/macagr/homelab-apps.git
       targetRevision: main
       path: apps/<name>
     destination:
       server: https://kubernetes.default.svc
       namespace: <name>
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. **Commit and push** to `main`:

   ```bash
   git add apps/<name>/ bootstrap/<name>.yaml
   git commit -m "deploy <name>"
   git push
   ```

4. **Monitor** in the Argo CD UI at https://argocd.mcagr.com or via CLI:

   ```bash
   kubectl get application -n argocd <name>
   ```

### Removing a workload

Delete `bootstrap/<name>.yaml`, commit, and push. Argo CD auto-prunes the
child Application, which cascades to delete all managed resources including
the namespace.

### Forcing a manual sync

```bash
kubectl -n argocd patch application <name> \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

Or use the Argo CD UI: select the Application → Sync → Synchronize.

### Emergency out-of-band changes

The manual `kubectl apply` workflow from above still works for emergencies.
Be aware that Argo CD's self-heal will revert manual changes within ~3 minutes
unless you first disable auto-sync on the affected Application.
