# Commands — per-topic reference

This directory splits the global `../commands.md` into focused files, one per resource type or task area. Use this when you're studying or drilling a single topic; use `../commands.md` when you want a single page to Ctrl+F across everything.

Both stay in sync — when a notes chapter adds new commands, update **both** the relevant file here and the global reference.

## Index

### Core & cross-cutting

| File | What's in it |
|---|---|
| [`imperative.md`](imperative.md) | **The exam time-saver.** Imperative one-liners + `--dry-run=client -o yaml` to generate manifests. Read this first. |
| [`setup.md`](setup.md) | Aliases (`k`, `$do`, `$now`), `.bashrc`, vim config, context/namespace verification |
| [`cluster-context.md`](cluster-context.md) | `kind` clusters, contexts, switching namespace defaults |
| [`debugging.md`](debugging.md) | `logs`, `exec`, `describe`, `events`, `explain`, `api-resources` |
| [`observability.md`](observability.md) | Full `kubectl logs` variants + `kubectl top` (metrics-server) |
| [`vim.md`](vim.md) | Vim survival kit for editing YAML on the exam |

### Workloads

| File | What's in it |
|---|---|
| [`pods.md`](pods.md) | Pod create/inspect/edit/delete, which fields are mutable |
| [`multi-container-pods.md`](multi-container-pods.md) | Sidecars, init containers, per-container logs/exec |
| [`deployments.md`](deployments.md) | Deployment generation, scaling, rollout, rollback, restart |
| [`deployment-strategies.md`](deployment-strategies.md) | Blue/green (selector flip) and canary (shared label) |
| [`probes.md`](probes.md) | Readiness vs liveness probes, handlers, timing |
| [`jobs-cronjobs.md`](jobs-cronjobs.md) | Job, CronJob, manual trigger from a CronJob |
| [`statefulsets.md`](statefulsets.md) | StatefulSets, ordinals, partitioned rollout, headless service DNS |

### Configuration & scheduling

| File | What's in it |
|---|---|
| [`configmaps-secrets.md`](configmaps-secrets.md) | All `--from-*` variants, TLS shorthand, decoding |
| [`security-contexts.md`](security-contexts.md) | runAsUser, capabilities, privileged; Docker→K8s mapping |
| [`service-accounts.md`](service-accounts.md) | SAs, `create token`, attaching to workloads |
| [`labels-selectors.md`](labels-selectors.md) | Equality/set-based selectors, `label`, `annotate` |
| [`scheduling.md`](scheduling.md) | Taints/tolerations, node selectors, node affinity, dedicated nodes |

### Networking & storage

| File | What's in it |
|---|---|
| [`services.md`](services.md) | `expose`, ClusterIP vs NodePort, endpoints |
| [`namespaces.md`](namespaces.md) | Create, default-switch, cross-namespace queries, delete |
| [`network-policies.md`](network-policies.md) | Ingress/egress rules, the AND/OR dash trap, testing |
| [`ingress.md`](ingress.md) | `kubectl create ingress --rule`, v1 structure traps |
| [`storage.md`](storage.md) | Volumes, PV/PVC binding, StorageClasses, dynamic provisioning |

### Security

| File | What's in it |
|---|---|
| [`kubeconfig.md`](kubeconfig.md) | kubeconfig sections, contexts, client certs, KUBECONFIG merge |
| [`rbac.md`](rbac.md) | Roles/bindings, ClusterRoles, `auth can-i` (+ `--as`) |
| [`authorization.md`](authorization.md) | API groups/resources exploration, authz modes, admission controllers |
| [`crd.md`](crd.md) | CRDs, custom resource create/get, operators |

### Packaging

| File | What's in it |
|---|---|
| [`helm.md`](helm.md) | Repos, install/upgrade/rollback, values, `template`, `pull`, the `ls -a` / `--reuse-values` / rollback-revision traps |
| [`kustomize.md`](kustomize.md) | `-k` vs `-f`, an annotated `kustomization.yaml` (transformers, patches, generators), `kustomize edit` where available |

## Conventions

All files assume the aliases from `setup.md` are sourced:

- `k` = `kubectl`
- `$do` = `--dry-run=client -o yaml`
- `$now` = `--force --grace-period=0`
