# gatekeeper-policies

ConstraintTemplates and Constraint instances for the cluster's admission
policies. Architecture and lifecycle in [../../docs/gatekeeper.md](../../docs/gatekeeper.md).

## Layout

```
apps/gatekeeper-policies/
├── config.yaml         # Gatekeeper Config CR (audit scope + namespace exclusions)
├── templates/          # ConstraintTemplate CRs (define new CRD kinds + Rego)
└── constraints/        # Constraint CRs (instances; apply policies)
```

One file per resource, named after the resource. Templates and Constraints
are kept in separate directories so the Argo CD sync-wave annotations stay
mechanical:

| Sync wave | Contents                                       |
|-----------|------------------------------------------------|
| `1`       | `config.yaml`, every file in `templates/`      |
| `2`       | every file in `constraints/`                   |

The wave annotation goes on each manifest individually
(`argocd.argoproj.io/sync-wave: "2"` on Constraint files, `"1"` on Templates
and Config).

## Initial state — empty

The directories ship empty (`.gitkeep` placeholders) so the Argo Application
syncs cleanly without policy enforcement on day one. Add policies in
follow-up PRs, starting with `enforcementAction: dryrun` so existing
workloads aren't blocked while we discover non-compliance.

## Recommended first wave of policies

From the [Gatekeeper Library](https://open-policy-agent.github.io/gatekeeper-library/),
mapping to entries in [docs/conventions.md](../../docs/conventions.md):

| Library policy                  | Convention enforced                                             |
|---------------------------------|-----------------------------------------------------------------|
| `K8sRequiredLabels`             | `app.kubernetes.io/{name,component,part-of}` on every workload  |
| `K8sContainerLimits`            | Every container declares CPU + memory limits + requests         |
| `K8sPSPAllowPrivilegeEscalationContainer` | `runAsNonRoot: true` where the image supports it      |
| `K8sBlockNodePort` (optional)   | No NodePort services (we use Gateway API only)                  |

Custom policies (Rego authored locally):

| Custom policy                   | Convention enforced                                             |
|---------------------------------|-----------------------------------------------------------------|
| `HomelabStatefulNeedsSSD`       | Pods with PVCs must declare `nodeSelector: storage: ssd`        |
| `HomelabSecretNoPlaceholders`   | Secret manifests must not contain `REPLACE_ME_BEFORE_DEPLOY`    |

## Authoring + lifecycle

See [docs/gatekeeper.md](../../docs/gatekeeper.md) §4 for the full workflow:
authoring conventions, dryrun → warn → deny rollout, gator-in-CI, retiring.
