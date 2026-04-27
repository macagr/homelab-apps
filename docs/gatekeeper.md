# Gatekeeper — admission policy

This document covers the **OPA Gatekeeper** deployment: why it exists, how it
fits the cluster, the operational decisions baked into the values, and how
policies are authored, tested, rolled out, and retired. It does **not** cover
the OPA-as-PDP (Policy Decision Point) deployment — that is a separate concern
addressed in a future doc when a real Policy Enforcement Point materialises.

Verified against **Gatekeeper v3.22.2** (current stable as of writing) and
its corresponding Helm chart on
`https://open-policy-agent.github.io/gatekeeper/charts`.

---

## 1. Why Gatekeeper

Gatekeeper is a Kubernetes-native admission controller built on top of OPA
(Open Policy Agent). It validates (and optionally mutates) every object the
API server sees, rejecting requests that violate cluster policy. We use it
to turn the conventions documented in [conventions.md](conventions.md) from
"things we promise we'll do in PRs" into hard guarantees enforced at admission
time.

Why Gatekeeper specifically (vs. alternatives):

- **Kyverno** — also valid; uses YAML policies, no Rego required. We chose
  Gatekeeper because Rego transfers 1:1 to the future OPA-as-PDP work, so
  policy-authoring skill investment compounds.
- **PSA (Pod Security Admission)** — built into Kubernetes, but limited to
  pod-security concerns. Doesn't cover "stateful workloads must use
  `storage: ssd`" or "every workload must declare resource limits".
- **Validating Admission Policies (CEL)** — newer, in-tree, lighter-weight.
  Less mature ecosystem and no shared library yet. Gatekeeper supports CEL
  policies *in addition* to Rego from v3.21+, so this isn't an exclusive choice.

## 2. How it fits the existing GitOps patterns

Gatekeeper is deployed using **two of the three packaging patterns** already
in use in this repo (see [README.md](../README.md)):

| Component                 | Pattern         | Path                      | Reason                                                                                              |
|---------------------------|-----------------|---------------------------|-----------------------------------------------------------------------------------------------------|
| Gatekeeper operator       | Umbrella chart  | `apps/gatekeeper/`        | Third-party app with hundreds of values. Pin the upstream chart; override locally.                   |
| Policies (Templates + CRs)| Raw manifests   | `apps/gatekeeper-policies/`| First-party YAML, simple, single-environment. No templating overhead needed.                         |

The two are **deliberately separate Argo Applications** because their change
cadence differs: chart bumps are infrequent (every few months), but policies
evolve weekly. Splitting them lets us roll either side independently — useful
during incident response, when you may want to pause policy enforcement
without touching the operator, or vice versa.

## 3. Architecture decisions

### 3.1 Placement

We use the documented **HA-across-Tailscale scheme** for admission-webhook
controllers (see homelab repo's
[`k3s_labels` role README](../../homelab/ansible/roles/k3s_labels/README.md)).
The same scheme already backs cert-manager and external-secrets.

| Component             | Placement                                          | Reasoning                                                                                                                                    |
|-----------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `controllerManager`   | pi-03 + node-4, **2 replicas** with hostname spread | Synchronous admission round-trips need low apiserver latency. pi-03 satisfies steady-state; node-4 is the HA partner over Tailscale.         |
| `audit`               | node-4, **1 replica**                              | Crawls every object every 5 min. node-4 is the only SSD-backed node and has 8 GB RAM headroom. Tailscale latency on a 5-minute loop is trivial. |

Selectors use the existing label taxonomy (`workload`, `node-location`) — see
the [shared affinity block](../../homelab/ansible/group_vars/all.yml) (variable
`affinity_app_workloads_node_affinity`).

### 3.2 Failure mode — `failOpen` during ramp

`validatingWebhookFailurePolicy: Ignore` (the chart default) is kept for
initial rollout. Rationale:

- Pi-cluster + new deployment ⇒ optimise for **not bricking ourselves**.
- `failClosed` (Fail) means: if Gatekeeper webhooks crash or the network
  blips, **all admissions halt**. On a Tailscale-stretched cluster with HDD
  control plane, that's a real risk.
- Cost of `Ignore`: during a Gatekeeper outage, policies are silently
  bypassed. Acceptable while we're still learning what the policy set should
  look like.

We will revisit per-Constraint once a baseline is stable. Specific
high-stakes Constraints can override to `Fail` individually via the
ValidatingWebhookConfiguration's `namespaceSelector` if we ever need that.

### 3.3 Validation only — no mutation

`disableMutation: true`. We don't apply mutation webhooks for now because:

- Mutation has its own failure modes (idempotency, ordering with other
  mutators) and is harder to reason about.
- Validation alone covers every convention in [conventions.md](conventions.md).
- Adding mutation later is a one-flag change.

### 3.4 Audit interval — 5 minutes (300 s)

Default 60 s is fine on a fleet but excessive for a homelab. At 60 s, audit
hits the apiserver continuously and writes status updates back to kine on
pi-01 (USB HDD). 5 min cuts the load by 5× with no real loss — we're not
hunting policy violations in real time, we're auditing for trends.

### 3.5 Logging

- `logLevel: WARN` — default `INFO` is chatty during ArgoCD reconciliation
  churn (every applied object hits the webhook, every webhook decision is
  logged at INFO).
- `logDenies: true` — when a policy denies a request, we want to know.
  Surfaces in cluster logs, picked up by the future log stack.

### 3.6 Resource budget

| Workload            | Requests          | Limits              | Notes                                                |
|---------------------|-------------------|---------------------|------------------------------------------------------|
| controller-manager  | 50 m / 128 Mi     | (none) / 256 Mi     | Bursty; default chart limits are fine.                |
| audit               | 100 m / 256 Mi    | (none) / 512 Mi     | Spikes during the periodic crawl; idle between.       |

The chart applies sensible CPU/memory defaults; we trim memory specifically
because the homelab nodes are memory-constrained. CPU limits are intentionally
left empty (Linux CFS unfair when CPU-throttled).

### 3.7 Exempt namespaces

Two layers of exemption — they look at different things:

- **Webhook exemption** (`controllerManager.exemptNamespaces`): stops the
  admission webhook from gating object writes in those namespaces.
- **Config CR exclusion** (`match.excludedNamespaces`): stops audit from
  reporting violations on those namespaces.

We keep both lists aligned. Exempted namespaces are infrastructure that:
- runs *before* Gatekeeper (chicken-and-egg risk)
- has its own admission controllers we don't want to interfere with
- is operated by upstream charts whose manifests we don't control

| Namespace          | Why exempt                                                          |
|--------------------|---------------------------------------------------------------------|
| `kube-system`      | k3s + Cilium core. Bootstraps before Gatekeeper.                    |
| `gatekeeper-system`| Self-exempt to avoid circular admission.                            |
| `argocd`           | Reconciles everything else; if Argo's own pods are blocked, the cluster's recovery loop dies. |
| `cert-manager`     | Issues certs for everything; webhook-backed, has bootstrap order constraints. |
| `external-secrets` | Pulls 1Password secrets; bootstrap-critical.                        |
| `metallb-system`   | LoadBalancer for in-cluster services. ARP-critical.                 |
| `traefik`          | Ingress for everything.                                             |
| `external-dns`     | DNS automation; chart manifests we don't author.                    |
| `registry-mirror`  | Docker Hub pull-through; image-pull-critical.                       |

User-facing application namespaces (`homepage`, `gitea`, `monitoring`, `rss`,
`k8bot`, future apps) are **not** exempt — that's where policies actually
matter.

### 3.8 Sync waves

Gatekeeper has a known timing wrinkle: a `ConstraintTemplate` CR generates a
new CRD at runtime, which the matching `Constraint` then uses. So we need
ordering between three layers:

| Wave | Resource                            | Path                                |
|------|-------------------------------------|-------------------------------------|
| 0    | Operator chart (installs `ConstraintTemplate` CRD + webhook) | `apps/gatekeeper/`                  |
| 1    | `ConstraintTemplate` CRs + Config CR | `apps/gatekeeper-policies/templates/`, `apps/gatekeeper-policies/config.yaml` |
| 2    | `Constraint` CRs                    | `apps/gatekeeper-policies/constraints/` |

Implementation:
- The two bootstrap Applications carry `argocd.argoproj.io/sync-wave`
  annotations (gatekeeper = 0, gatekeeper-policies = 1).
- Inside `apps/gatekeeper-policies/`, individual manifests carry their own
  sync-wave annotations to enforce the templates → constraints order.

Sync options on `apps/gatekeeper-policies/`:

```yaml
syncOptions:
  - CreateNamespace=true
  - ServerSideApply=true              # ConstraintTemplates are large CRDs
  - Replace=true                       # required for CRD updates
  - SkipDryRunOnMissingResource=true   # CRDs created at runtime aren't visible to client-side dry-run
```

## 4. Policy management

### 4.1 Source of policies — library + custom

Two sources, in priority order:

1. **[Gatekeeper Library](https://open-policy-agent.github.io/gatekeeper-library/)** — for universal policies (required labels, no privileged containers, runAsNonRoot, container resource limits, allowed registries). Vendor verbatim into `apps/gatekeeper-policies/templates/`.
2. **Custom Rego** — only for homelab-specific rules that the library doesn't cover:
   - Stateful workloads must declare `nodeSelector: storage: ssd` (per [conventions.md](conventions.md))
   - Secret manifests must not contain `REPLACE_ME_BEFORE_DEPLOY` placeholders

Don't rewrite what's already battle-tested. Rule of thumb: if you find
yourself writing more than 20 lines of Rego, check the library first.

### 4.2 Authoring — ConstraintTemplate + Constraint pairs

A policy is two manifests:

- `ConstraintTemplate`: defines a new CRD plus the Rego that backs it.
- `Constraint`: an instance of that CRD with parameters (which kinds it
  applies to, which labels are required, etc.).

Naming convention in this repo:
```
apps/gatekeeper-policies/
  templates/
    required-labels.yaml          # ConstraintTemplate: K8sRequiredLabels
    container-limits.yaml         # ConstraintTemplate: K8sContainerLimits
    storage-ssd-required.yaml     # custom ConstraintTemplate: HomelabStorageSelector
  constraints/
    apps-must-have-standard-labels.yaml   # Constraint: K8sRequiredLabels instance
    workloads-must-declare-limits.yaml    # Constraint: K8sContainerLimits instance
    stateful-workloads-need-ssd.yaml      # Constraint: HomelabStorageSelector instance
```

One file per resource. Templates and Constraints in separate directories so
the sync-wave annotation pattern stays mechanical.

### 4.3 Roll-out strategy — `dryrun` → `warn` → `deny`

Every Constraint lives through three lifecycle states, controlled by the
`enforcementAction` field:

```yaml
spec:
  enforcementAction: dryrun   # → warn → deny
```

| State      | Behavior                                                   | When to use                                        |
|------------|------------------------------------------------------------|----------------------------------------------------|
| `dryrun`   | Audit reports violations; admission **never** rejected.    | Always start here. Discover existing non-compliance. |
| `warn`     | Admission reports violations as warnings; still accepted.  | Used by humans during PR review. Useful in CI.     |
| `deny`     | Admission rejected on violation.                           | Final state once we trust the policy.              |

Lifecycle in practice:

1. **Land Constraint as `dryrun`** in a PR. Argo syncs.
2. **Read the audit results**: `kubectl get <constraintkind> -A -o yaml | yq .status.violations`. This is the real-time map of how non-compliant the cluster is.
3. **Fix violators in-repo** via PRs (add the missing label, declare the missing limit, etc.).
4. **Wait one audit cycle** (5 min default). Confirm violations list is empty.
5. **Promote to `deny`** in a follow-up PR. Argo syncs. From now on, new violations are blocked at admission.

This is the GitOps version of feature-flagging. It also doubles as cluster
auditing: even before flipping to `deny`, the `status.violations` field is a
real, current view of how compliant the cluster is today.

### 4.4 CI integration — `gator test`

`gator` is Gatekeeper's offline test CLI (ships with each Gatekeeper release).
Same ConstraintTemplate + Constraint files we deploy to the cluster, evaluated
locally against arbitrary YAML — zero drift between what CI checks and what
the admission webhook enforces.

Wired into Gitea Actions on PR: every PR that touches `apps/` or `platform/`
runs `gator test apps/gatekeeper-policies/templates/ apps/gatekeeper-policies/constraints/ apps/<...changed dirs...>/`. Failures block merge.

`gator` interprets `enforcementAction` as follows:
- `dryrun`/`warn` Constraints → violations reported but exit code 0
- `deny` Constraints → violations cause non-zero exit code, blocking the PR

So once a Constraint is promoted to `deny` at runtime, it also starts blocking
PRs in CI — same source of truth on both sides.

A complementary alternative for the same job is `conftest`, which uses plain
Rego rather than Gatekeeper's CRD format. We use `gator` because it shares
the deployment artifacts; `conftest` would mean maintaining policies twice.

### 4.5 Audit results & observability

`kubectl get <constraintkind>` shows each Constraint's `.status.violations` —
the running audit's current findings. For human-readable summary across all
Constraints:

```bash
kubectl get constrainttemplates,constraints -A
```

Once monitoring is up under `apps/monitoring/`:
- A `ServiceMonitor` will scrape Gatekeeper's Prometheus endpoint
  (`controllerManager.metricsPort: 8888`).
- The two panels that matter on the Grafana dashboard:
  - **Constraint violation count by Constraint** — trend down as we fix.
  - **Audit lag / cycle duration** — grows with constraint count; budget for it.

### 4.6 Updating, retiring, and rolling back policies

- **Update a policy:** edit the ConstraintTemplate or Constraint manifest, PR.
  Argo syncs. Audit re-evaluates within one cycle.
- **Retire a policy:** delete the Constraint manifest first; let Argo prune
  it. Then delete the ConstraintTemplate. Don't reverse — deleting the
  template while Constraints still exist orphans them.
- **Roll back enforcement:** flip `deny` → `dryrun` in the same Constraint
  manifest. Audit keeps reporting; admission stops blocking. Buys time to
  investigate without breaking deploys.

## 5. What's deferred

- **Mutation webhook**: useful for "auto-add `runAsNonRoot: true` if missing"
  but not yet. Validation only at start.
- **External data providers**: Rego that calls out to external systems
  (e.g. image vulnerability lookups). Out of scope until there's a concrete
  need.
- **VAP enforcement scope**: Gatekeeper v3.21+ can sync constraints to
  Validating Admission Policies (CEL). Useful for performance at scale; we
  don't need it at homelab volume.
- **OPA-PDP** (standalone OPA serving Rego over HTTP for application-level
  authz): documented separately when the first PEP appears.

## 6. References

- [Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/website/docs/)
- [Gatekeeper Helm chart](https://github.com/open-policy-agent/gatekeeper/tree/master/charts/gatekeeper)
- [Gatekeeper Library](https://open-policy-agent.github.io/gatekeeper-library/)
- [Rego language docs](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [`gator` CLI reference](https://open-policy-agent.github.io/gatekeeper/website/docs/gator)
- Cluster placement scheme: [`k3s_labels` role README](../../homelab/ansible/roles/k3s_labels/README.md) (homelab repo)
