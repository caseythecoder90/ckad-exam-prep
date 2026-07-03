# Service Accounts

A ServiceAccount is a pod's identity to the API server. Every pod gets one (`default` if unset). Tokens changed a lot across versions — since 1.24 there's no auto-generated Secret; you mint short-lived bound tokens with `kubectl create token`.

## Create & inspect

```bash
k create serviceaccount build-bot          # short: k create sa build-bot
k get sa                                     # note SECRETS column is 0 on 1.24+
k get sa build-bot -o yaml
k describe sa build-bot
```

## Tokens (1.24+)

```bash
k create token build-bot                                   # ~1h bound token, printed to stdout
k create token build-bot --duration=24h
k create token build-bot --audience=https://my-app.example.com
k create token build-bot --bound-object-kind=Secret --bound-object-name=my-secret
```

Long-lived token via a manually-created Secret (when you truly need a static token):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-bot-token
  annotations:
    kubernetes.io/service-account.name: build-bot
type: kubernetes.io/service-account-token
```

```bash
k apply -f secret.yaml
k get secret build-bot-token -o jsonpath='{.data.token}' | base64 -d
```

## Attach to a workload

```bash
# Deployment (safe, rolls new pods)
k set serviceaccount deployment/my-deploy build-bot

# Pod at creation (via YAML — serviceAccountName is immutable on a live bare Pod)
k run nginx --image=nginx --dry-run=client -o yaml     # then add serviceAccountName
```

```yaml
spec:
  serviceAccountName: build-bot        # on spec, NOT metadata; NOT "serviceAccount"
  automountServiceAccountToken: false  # opt out of token injection (Pod or SA level; Pod wins)
```

## Verify the injected token (inside the pod)

```bash
k describe pod <pod>                                                            # Mounts: kube-api-access-*
k exec -it <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount           # ca.crt namespace token
k exec -it <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

## See also

- `02-configuration/09-service-accounts.md`
- `rbac.md` — grant the SA permissions with a RoleBinding (`--serviceaccount=<ns>:<name>`)
