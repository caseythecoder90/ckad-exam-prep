# RBAC — Roles, Bindings & `auth can-i`

Role/RoleBinding are namespaced; ClusterRole/ClusterRoleBinding are cluster-wide. A **ClusterRole bound by a RoleBinding** grants its permissions in just that one namespace (the dual-use pattern). Subjects: `--user`, `--group`, `--serviceaccount=<ns>:<name>`.

## Imperative create (exam-friendly)

```bash
# Role (namespaced)
k create role developer --verb=list,get,create,update,delete --resource=pods --namespace=default
k create role developer --verb=get,list --resource=pods,configmaps
k create role developer --verb=get,delete --resource=pods --resource-name=blue,orange   # named objects only

# RoleBinding
k create rolebinding dev-binding --role=developer --user=dev-user --namespace=default
k create rolebinding team-binding --role=developer --group=dev-team
k create rolebinding sa-binding   --role=developer --serviceaccount=default:my-sa

# Bind a BUILT-IN ClusterRole in one namespace
k create rolebinding casey-view --clusterrole=view --user=casey --namespace=default

# ClusterRole / ClusterRoleBinding (cluster-wide)
k create clusterrole node-reader --verb=get,list,watch --resource=nodes
k create clusterrolebinding casey-node-reader --clusterrole=node-reader --user=casey

# Generate YAML instead of applying
k create role developer --verb=get,list --resource=pods $do > developer-role.yaml
```

## Inspect

```bash
k get roles,rolebindings                 # current namespace
k get roles -A                            # all namespaces
k get clusterroles                        # includes built-ins (view, edit, admin, cluster-admin)
k get clusterrolebindings
k describe role developer                 # rules
k describe rolebinding dev-binding -n default   # subject + roleRef
k get clusterrole edit -o yaml            # copy a built-in's rules as a starting point
```

## `kubectl auth can-i` (permission checks)

```bash
k auth can-i create deployments           # your own permission
k auth can-i list secrets -n kube-system
k auth can-i --list                        # everything you can do here
k auth can-i --list -n finance

# Impersonate (needs admin) — the exam's "does user X have access" tool
k auth can-i create pods --as dev-user -n default
k auth can-i --list --as dev-user -n default
k auth can-i list pods --as system:serviceaccount:default:my-sa
k auth can-i list nodes --as=cluster-admin
```

## Role vs ClusterRole — which to use

```bash
k api-resources --namespaced=true         # namespaced resources -> Role
k api-resources --namespaced=false        # cluster-scoped (nodes, pv, ns) -> ClusterRole
```

## See also

- `09-security/06-rbac.md`, `07-cluster-roles.md`
- `service-accounts.md` · `kubeconfig.md` (who you authenticate as) · `authorization.md`
