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

## Future workflow — Argo CD GitOps

> **Status: not yet active.** Argo CD is installed on the cluster but is not
> yet pointing at this repository.

When GitOps wiring is in place, the workflow will change to:

1. Open a pull request with your manifest changes.
2. Get it reviewed and merged to `main`.
3. Argo CD detects the change and automatically syncs the affected Application.
4. Monitor the sync status in the Argo CD UI or via `argocd app get <workload>`.

No manual `kubectl apply` will be needed for normal operations. The manual workflow
above will remain useful for emergency out-of-band intervention.

### TODO — things to document once GitOps is live

- [ ] How to create an Argo CD Application pointing at `apps/<workload>/`
- [ ] Sync policy (manual vs automatic, prune, self-heal settings)
- [ ] How to force a manual sync from the CLI
- [ ] Rollback procedure
- [ ] What happens to secrets — likely requires Sealed Secrets or External Secrets
      to be set up first (see [secrets.md](secrets.md))
