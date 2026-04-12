# Secrets handling

**Rule: no real secrets are ever committed to this repository.**

This rule holds whether the repository is private or public, now or in the future.
A committed secret is a permanent liability — git history is forever.

---

## Placeholder convention

Secret manifests are committed with obvious placeholder values so it is impossible
to accidentally apply them without noticing:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-credentials
  namespace: my-app
type: Opaque
stringData:
  username: "REPLACE_ME_BEFORE_DEPLOY"
  password: "REPLACE_ME_BEFORE_DEPLOY"
  api-key:  "REPLACE_ME_BEFORE_DEPLOY"
```

The string `REPLACE_ME_BEFORE_DEPLOY` is intentional. If you see it in a `kubectl get
secret -o yaml` output you know the secret was never properly populated.

`stringData` is used (not `data`) so values are human-readable in code review.
Kubernetes converts `stringData` to base64-encoded `data` at apply time.

---

## Applying secrets with real values

**Never edit the committed placeholder file.** Instead, apply the real secret out of
band using one of these approaches:

### Option 1 — `kubectl create secret` (quickest for one-off secrets)

```bash
kubectl create secret generic my-app-credentials \
  --namespace my-app \
  --from-literal=username='actualuser' \
  --from-literal=password='actualpassword' \
  --from-literal=api-key='sk-...' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Using `--dry-run=client -o yaml | kubectl apply -f -` rather than bare `create`
makes the operation idempotent (safe to re-run).

### Option 2 — local override file (never committed)

Copy the placeholder manifest to a local file that is git-ignored, fill in real
values, and apply that file:

```bash
cp apps/my-app/secret.yaml /tmp/my-app-secret-real.yaml
# edit /tmp/my-app-secret-real.yaml with real values
kubectl apply -f /tmp/my-app-secret-real.yaml
rm /tmp/my-app-secret-real.yaml
```

---

## .gitignore backstop

`.gitignore` contains patterns like `*secret*.yaml` and `*secrets*.yaml`. This is a
backstop, not the primary defense. The primary defense is discipline: never put real
values in files that live in this repo.

---

## Future: Sealed Secrets / External Secrets

Neither [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) nor
[External Secrets Operator](https://external-secrets.io/) is configured on the
cluster yet. One of them will likely be added later to enable fully GitOps-driven
secret management. When that happens, this document will be updated and the
placeholder convention may be retired.

Until then, the manual out-of-band approach above is the required workflow.
