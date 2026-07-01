# Kubeconfig & Authentication

kubeconfig has three sections — **clusters** (where), **users** (credentials), **contexts** (which user + cluster + namespace). Kubernetes has no User object: `k create user` / `k get users` don't exist. Users are proven by X.509 certs / tokens / OIDC; only ServiceAccounts are cluster-managed.

## Contexts & namespace (the everyday commands)

```bash
k config current-context
k config get-contexts                     # active marked with *
k config use-context <name>               # switch (persists)
k config set-context --current --namespace=<ns>   # set default ns — most-used
k config rename-context <old> <new>
k config delete-context <name>
```

## View

```bash
k config view                             # certs redacted
k config view --raw                       # un-redacted (actual cert data)
k config view --minify                    # only the current context
k config view --minify | grep namespace
```

## Wire up a cluster / user / context by hand

```bash
k config set-cluster prod --server=https://api.example.com:6443 --certificate-authority=/path/ca.crt
k config set-credentials casey --client-certificate=casey.crt --client-key=casey.key
k config set-context casey@prod --cluster=prod --user=casey --namespace=dev
```

## One-off overrides (don't switch context)

```bash
k get pods --context=dev@development
k get pods --kubeconfig=/tmp/other-config
k -n kube-system get pods

# Merge multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/work-config
k config get-contexts
```

## Issue a client cert for a user (X.509)

```bash
openssl genrsa -out casey.key 2048
openssl req -new -key casey.key -out casey.csr -subj "/CN=casey/O=developers"   # CN=user, O=group
openssl x509 -req -in casey.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out casey.crt -days 365
k config set-credentials casey --client-certificate=casey.crt --client-key=casey.key
```

Inline a CA cert into kubeconfig: `cat ca.crt | base64 -w0` (Linux; drop `-w0` on macOS).

## Talk to the API server directly (debug auth)

```bash
curl https://<apiserver>:6443/api/v1/pods --header "Authorization: Bearer <token>" --cacert ca.crt
curl https://localhost:6443/api --key admin.key --cert admin.crt --cacert ca.crt
```

## See also

- `09-security/02-authentication.md`, `03-kubeconfig.md`
- `cluster-context.md` — kind + quick switching · `rbac.md` — what the identity is allowed to do
