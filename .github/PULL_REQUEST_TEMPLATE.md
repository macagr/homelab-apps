## What workload is this changing?

<!-- e.g. apps/grafana, platform/cert-manager-issuers, demos/nginx-test -->

## What changed and why?

<!-- Brief description of the change -->

## Checklist

- [ ] Tested with `kubectl apply --dry-run=server -f <directory>/`
- [ ] Resource limits are set on every container (`requests` and `limits` for CPU and memory)
- [ ] Containers run as non-root where the upstream image supports it (`runAsNonRoot: true`)
- [ ] No real secrets are included — placeholder values (`REPLACE_ME_BEFORE_DEPLOY`) used where needed
- [ ] Labels include `app.kubernetes.io/name`, `/component`, and `/part-of`
- [ ] If stateful: `nodeSelector: storage: ssd` is set
