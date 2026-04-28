# Descheduler — re-spreading workloads after node flaps

Kubernetes places pods at scheduling time; it does **not** rebalance running
pods when conditions change. So whenever pi-03 has a hiccup, every webhook
controller using soft `podAntiAffinity` (cert-manager, external-secrets, and
historically gatekeeper before we switched to TopologySpreadConstraints)
collapses onto node-4. When pi-03 returns, nothing pulls them back. Over
weeks of small outages, the cluster's HA workloads accumulate on whichever
node won the most recent race.

[Descheduler](https://github.com/kubernetes-sigs/descheduler) is the
upstream answer: a controller that periodically scans for pods violating
placement constraints and evicts them via the Eviction API. The owning
controller (Deployment, StatefulSet) recreates each pod, and the scheduler
gets a fresh shot at honouring its affinity rules.

Verified against **descheduler v0.35.1** / Helm chart **0.35.0** (current
stable as of writing).

## 1. What it fixes here

| Problem                                                                    | Strategy that fixes it                          |
|----------------------------------------------------------------------------|-------------------------------------------------|
| TSC violation: Gatekeeper replicas collapsed to one node                   | `RemovePodsViolatingTopologySpreadConstraint`   |
| Soft podAntiAffinity collapse (cert-manager, external-secrets)             | `RemovePodsViolatingInterPodAntiAffinity`       |
| Two replicas of the same ReplicaSet on the same node (defence-in-depth)    | `RemoveDuplicates`                              |
| Required nodeAffinity drifted (e.g. label retracted from a node)           | `RemovePodsViolatingNodeAffinity`               |

All four are descheduler **balance** or **deschedule** plugins enabled in
our profile.

## 2. What we deliberately do *not* enable

| Strategy                          | Why disabled                                                                                            |
|-----------------------------------|---------------------------------------------------------------------------------------------------------|
| `LowNodeUtilization`              | Default-on. Would fight our deliberate asymmetric placement (heavy on node-4 by design).                |
| `HighNodeUtilization`             | Inverse of the above. Same problem.                                                                     |
| `RemovePodsHavingTooManyRestarts` | Could cause flapping with poorly-tuned probes; orthogonal to the placement problem we're solving here.   |
| `RemovePodsViolatingNodeTaints`   | Default-on. Useful in theory, but pi-02's `dedicated=lightweight:NoSchedule` taint is stable; not worth the surface area. |

## 3. Eviction safety

Descheduler's `DefaultEvictor` decides which pods are *eligible* to be
evicted. Defaults are aggressive — they allow eviction of pods with local
storage and PVCs. On this cluster that's risky:

- **registry-mirror** has a local-path PV bound to pi-01. Evicting it would
  trigger a re-schedule, but the new pod can only land on pi-01 (PV anchor)
  → at best a no-op restart, at worst a gap in pull-through caching.
- **gitea, monitoring, valkey-cluster, rss-db** all have PVCs. Same anchor
  problem.

We tighten the evictor:

| Field                       | Our value | Effect                                                                                  |
|-----------------------------|-----------|-----------------------------------------------------------------------------------------|
| `nodeFit`                   | `true`    | **Load-bearing safety.** Verifies a *different* node can actually take the pod before evicting. For local-path-backed PVCs (RWO, node-anchored), no other node has the volume → no other node "fits" → eviction is skipped automatically. |
| `evictLocalStoragePods`     | `false`   | Skip pods with `emptyDir` / hostPath / local-path-anchored volumes.                      |
| `evictDaemonSetPods`        | `false`   | DaemonSets are pinned by definition; eviction is meaningless.                            |
| `evictSystemCriticalPods`   | `false`   | Don't touch kube-system / Argo / cert-manager system-critical pods.                      |

We deliberately do **not** set `ignorePvcPods: true`. That filter requires
`list`/`watch` on PVCs cluster-wide, which the chart's default ClusterRole
does not grant — enabling it caused descheduler to spam `persistentvolumeclaims is forbidden`
errors on every reconcile. `nodeFit: true` covers the same ground without
the extra permission, since on this cluster every PVC is local-path
(RWO, node-pinned) and therefore can't fit anywhere except its origin node.

## 4. Cadence

CronJob mode, every 15 minutes (`*/15 * * * *`). Default is every 2 minutes,
which is overkill for a homelab. 15 minutes converges quickly enough for
node flaps without putting steady churn on the apiserver.

We don't use Deployment mode (continuous, with `deschedulingInterval`).
CronJob is simpler, leaves no idle controller pod between runs, and fits
the "infrequent batch action" shape of the work.

## 5. Placement of descheduler itself

Single replica on **pi-03** (`workload: apps`). Reasoning:

- Descheduler is API-call-heavy during its run cycle. Local LAN to pi-01's
  apiserver is meaningfully cheaper than Tailscale to node-4 → pi-01.
- Stateless, no PVCs, no eviction concerns about itself.
- Pi-03 has the headroom (~2 GB free).
- Co-locates with the steady-state replica of every webhook controller it
  manages — same cache locality story.

If pi-03 is down, descheduler can land on node-4 (the eligible set is the
same `workload=apps OR node-location=cloud` block other workloads use).
While down, no rebalancing happens — but that's fine; the rebalancing only
matters *after* pi-03 returns anyway.

## 6. Resource budget

Trimmed from chart defaults (500m/256Mi, both requests and limits).

| Component   | Requests       | Limits         |
|-------------|----------------|----------------|
| descheduler | 50m / 64Mi     | 200m / 128Mi   |

Descheduler is bursty — most of its 15-minute cycle is idle. Tightening
the requests lets the scheduler put it next to other small workloads on
pi-03.

## 7. How to verify it's working

```bash
# CronJob fires every 15 min — last run logs
kubectl -n descheduler logs -l app.kubernetes.io/name=descheduler --tail=50

# Force a run now
kubectl -n descheduler create job --from=cronjob/descheduler descheduler-manual-$(date +%s)

# Watch what it evicts
kubectl get events -n cert-manager -n external-secrets -n gatekeeper-system \
  --field-selector reason=EvictionByDescheduler -w
```

When everything's healthy and pods are properly spread, descheduler is a
no-op — it scans, finds no violations, exits clean. The interesting events
are after a node flap recovery, when you should see eviction traffic
followed by re-scheduling onto the recovered node.

## 8. What this *doesn't* solve

- **PVC-anchored pods staying on a single node.** That's by design — see §3.
  If pi-01 dies and stays dead, registry-mirror / gitea / etc. need manual
  intervention (recycle PVC → cache rebuilds).
- **Soft podAntiAffinity tie-scoring on initial scheduling.** Descheduler
  cleans up *after* the scheduler makes a poor choice. The longer-term fix
  is to switch cert-manager and external-secrets in the homelab Ansible
  roles to use `topologySpreadConstraints` like Gatekeeper now does. That's
  a separate PR worth doing.

## 9. References

- [Descheduler GitHub](https://github.com/kubernetes-sigs/descheduler)
- [Descheduler Helm chart](https://github.com/kubernetes-sigs/descheduler/tree/master/charts/descheduler)
- [Policy v1alpha2 reference](https://github.com/kubernetes-sigs/descheduler#policy-default-evictor)
