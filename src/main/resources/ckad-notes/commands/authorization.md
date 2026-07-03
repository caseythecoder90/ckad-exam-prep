# Authorization, API Groups & Admission

Inspection commands for the API request pipeline: **AuthN → AuthZ (authorization modes) → Admission controllers**. RBAC object commands are in `rbac.md`; `auth can-i` lives there too.

## Explore the API (groups, versions, resources)

Core group is `/api` (apiVersion `v1`, empty group string); named groups are `/apis` (e.g. `apps/v1`, `networking.k8s.io/v1`).

```bash
k api-resources                            # every resource: NAME, SHORTNAMES, APIVERSION, NAMESPACED, KIND
k api-resources --sort-by=name
k api-resources | grep deployment          # find a resource's group/version (apps/v1)
k api-resources --namespaced=false         # cluster-scoped resources
k api-versions                             # all group/version strings
k explain deployment                       # apiVersion + fields
k explain pod.spec.containers              # drill into nested fields
```

## Raw API via curl / proxy

```bash
kubectl proxy                              # local proxy on 127.0.0.1:8001, forwards kubeconfig auth
curl http://localhost:8001/apis           # named groups, no certs needed
curl http://localhost:8001/api/v1         # core resources

# Direct (needs client certs)
curl https://localhost:6443/version -k                              # version (unauth-ish)
curl https://localhost:6443/apis --key admin.key --cert admin.crt --cacert ca.crt | grep '"name"'
```

Useful unauth-ish endpoints: `/version`, `/healthz`, `/metrics`.

## Authorization mode (control-plane inspection)

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
k describe pod kube-apiserver-<node> -n kube-system | grep authorization-mode
ps aux | grep kube-apiserver | grep authorization-mode
```

Flag form: `--authorization-mode=Node,RBAC,Webhook` (chain — first ALLOW wins; RBAC is the examinable one). See `rbac.md` for `auth can-i`.

## Admission controllers (the gate after authorization)

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission
k exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
k get pods -n kube-system                  # confirm apiserver static pod restarted after editing its manifest

# Demo: NamespaceAutoProvision auto-creates a missing namespace on use
k run nginx --image=nginx --namespace blue
k get namespaces
```

Flags: `--enable-admission-plugins=...`, `--disable-admission-plugins=...`. Validating webhooks reject; mutating webhooks patch (`patchType: JSONPatch`).

## See also

- `09-security/04-api-groups.md`, `05-authorization.md`, `08-admission-controllers.md`, `09-validating-mutating-webhooks.md`
- `rbac.md` · `kubeconfig.md`
