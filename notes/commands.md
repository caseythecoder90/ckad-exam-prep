# Commands reference

> Quick-reference for the kubectl, kind, and shell commands you'll use most. Organized by resource type, with cross-cutting sections (setup, generation, debugging) at the top. For *why* a command works the way it does, see the chapter notes.

## How to use this file

- `Ctrl+F` for the resource (Pods, Deployments, Services, ...) to jump to that section.
- Each resource section follows the same order: **Generate → Inspect → Edit → Delete**.
- Assumes `k=kubectl`, `$do=--dry-run=client -o yaml`, `$now=--force --grace-period=0` are set (chapter 00).
- For deeper per-topic reference, see [`commands/`](commands/) — one focused file per resource. This page is the single-file global view that mirrors what's in there.

---

## 1. Setup at the start of every fresh shell / exam

```bash
# Aliases (the exam pre-sets k=kubectl and bash autocomplete — verify with: alias k)
echo 'export do="--dry-run=client -o yaml"' >> ~/.bashrc
echo 'export now="--force --grace-period=0"' >> ~/.bashrc
source ~/.bashrc

# Vim config for YAML (no tabs, 2-space indent, line numbers)
echo "set expandtab tabstop=2 shiftwidth=2 number" > ~/.vimrc

# Confirm context and namespace before doing anything
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## 2. Imperative commands & dry-run YAML generation (the exam time-saver)

> Typing YAML by hand is slow. Imperative commands create resources in one line, and `--dry-run=client -o yaml` turns those same commands into a manifest you can edit. Use this pattern for almost every "create a thing" question. Full reference: [`commands/imperative.md`](commands/imperative.md).

### The two imperative entry points

| Command | What it creates |
|---|---|
| `kubectl run` | **Pods only** (since 1.18 — does not create Deployments anymore) |
| `kubectl create` | Everything else: deployments, services (via `expose`), configmaps, secrets, jobs, cronjobs, namespaces, roles, ... |

### Path A — one-shot imperative (no file)

Fastest. Use when every required field has a flag.

```bash
k run nginx --image=nginx
k create deployment web --image=nginx --replicas=3
k create configmap app-cfg --from-literal=ENV=prod
k expose deployment web --port=80 --type=NodePort
k create namespace dev
```

### Path B — generate YAML, edit, apply

When the question needs fields imperative flags don't cover (env vars on a Deployment, probes, volumes, resources, securityContext).

```bash
# 1. Generate template
k <run|create> <args> $do > thing.yaml

# 2. Edit
vim thing.yaml

# 3. Validate (optional but cheap)
k apply -f thing.yaml --dry-run=client
k apply -f thing.yaml --dry-run=server   # stricter, hits the API server

# 4. Apply
k apply -f thing.yaml

# 5. Verify
k get <resource> <name>
k describe <resource> <name>
```

### What `--dry-run=client -o yaml` actually does

- `--dry-run=client` — kubectl builds the spec **without sending it to the API server**. Nothing is created.
- `-o yaml` — print the would-be-sent object as YAML.
- Combined with `> file.yaml`, you get a fully-formed manifest (apiVersion, kind, metadata, spec all populated) ready to edit.

`--dry-run=server` does the same but the API server also validates (catches missing namespaces, unknown fields, admission rejections). Still no persist.

### The `--` separator — two rules

Everything after `--` is the **container's command**, not kubectl's flags. Both failure modes are silent: you get a created resource with the wrong spec, not an error.

```bash
# Rule 1 — multi-command in -- → always sh -c "cmd1 && cmd2", never bare &&
k create job x -n batch --image=busybox -- cmd1 && cmd2              # WRONG: cmd2 runs locally
k create job x -n batch --image=busybox -- sh -c "cmd1 && cmd2"      # RIGHT

# Rule 2 — kubectl flags go BEFORE --, or they become command args
k create job x --image=busybox -- sh -c "cmd1 && cmd2" $do > job.yaml       # WRONG: job really created
k create job x -n batch --image=busybox $do -- sh -c "cmd1 && cmd2" > job.yaml   # RIGHT
```

Same for `;`, `|`, `>`, `$VAR`, globs — any shell metacharacter needs `sh -c`. Use single quotes (`'echo $HOME'`) when the variable must expand inside the container. Shape: `k create <kind> <name> <all flags> -- sh -c "…"`.

### Exam-time strategy

1. Read the whole question — note namespace, labels, image version, env vars.
2. Pick the entry point: `run` for a Pod, `create <kind>` for everything else, `expose` for a Service.
3. Try **pure imperative first**. Fewer chances to mis-indent YAML.
4. If a required field has no flag, switch to **`$do > file.yaml`**, edit, apply.
5. Always pass `-n <ns>` or set the default namespace first — forgetting this is the most common silly mistake.
6. Verify with `k get` and `k describe` (check the `Events:` section).

---

## 3. Inspecting & debugging (works for any pod)

```bash
# Logs
k logs <pod>
k logs <pod> -f                      # follow
k logs <pod> -c <container>          # multi-container pod
k logs <pod> --previous              # logs from a crashed previous container
k logs -l app=web                    # by label selector

# Shell into a container
k exec -it <pod> -- bash
k exec -it <pod> -c <container> -- sh
k exec <pod> -- <command>            # one-shot, non-interactive

# Describe — Events at the bottom shows scheduling, image pull, probe failures
k describe pod <name>
k describe deployment <name>

# Full live spec (everything Kubernetes stores about the resource)
k get pod <name> -o yaml
k get pod <name> -o yaml > pod.yaml  # extract for editing

# Wide / labeled output
k get pods -o wide                   # adds NODE + IP columns
k get pods --show-labels
k get pods -l app=web                # filter by label

# Watch live
k get pods -w

# Cluster events in chronological order
k get events --sort-by=.metadata.creationTimestamp
```

### Discovery — when you forget a field name or resource type

```bash
k explain pod                        # top-level fields
k explain pod.spec                   # drill in
k explain pod.spec.containers
k explain pod.spec.containers --recursive   # full nested tree

k api-resources                      # every resource type known to the cluster
k api-resources | grep -i ingress    # find a resource by partial name
```

---

## 4. Cluster & context

```bash
# kind (local lab)
kind create cluster --name <name> --config <file>
kind delete cluster --name <name>
kind get clusters

# Switch context / namespace
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>

# Confirm where you are
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## 5. Pods

```bash
# Generate (kubectl run — pods only)
k run nginx --image=nginx $do > pod.yaml
k run nginx --image=nginx --port=80 $do > pod.yaml
k run nginx --image=nginx --labels="app=web,tier=frontend" $do > pod.yaml
k run nginx --image=nginx --env="ENV=prod" $do > pod.yaml
k run nginx --image=nginx -n <namespace> $do > pod.yaml
k run busybox --image=busybox --command -- sleep 3600 $do > pod.yaml

# Imperative create (no file)
k run nginx --image=nginx
k run nginx --image=nginx --port=80

# Inspect
k get pods
k get pods -A                        # all namespaces
k get pods -o wide
k get pods --show-labels
k get pod <name> -o yaml

# Edit (only certain fields — see "What you can edit" below)
k edit pod <name>

# Extract live spec to a file (when you don't have the original YAML)
k get pod <name> -o yaml > pod.yaml

# Delete
k delete pod <name>                  # by name
k delete -f pod.yaml                 # by manifest
k delete pod -l app=web              # by label selector
k delete pods --all                  # all pods in current namespace
k delete pod <name> $now             # force-delete (only when stuck)
```

**Editable fields on a live pod** (everything else needs delete-and-recreate):

- `spec.containers[*].image`, `spec.initContainers[*].image`
- `spec.activeDeadlineSeconds`
- `spec.tolerations` (additions only)
- `spec.terminationGracePeriodSeconds` (lower only)

---

## 6. Deployments

```bash
# Generate
k create deployment web --image=nginx --replicas=3 $do > deploy.yaml

# Imperative
k create deployment web --image=nginx --replicas=3

# Inspect
k get deployments
k get deploy web -o yaml
k describe deploy web

# Scale
k scale deployment web --replicas=5

# Edit (deployments are fully mutable — controller rolls out new pods)
k edit deployment web
k set image deployment/web nginx=nginx:1.25     # update image only

# Rollout
k rollout status deployment/web
k rollout history deployment/web
k rollout undo deployment/web                   # roll back to previous revision
k rollout undo deployment/web --to-revision=2   # roll back to a specific one
k rollout restart deployment/web                # rolling restart (re-pulls image)

# Delete (removes deployment + replicaset + pods)
k delete deployment web
k delete -f deploy.yaml
```

---

## 7. Services

```bash
# Generate (from an existing deployment or pod)
k expose deployment web --port=80 --target-port=8080 $do > svc.yaml
k expose pod nginx --port=80 --target-port=80 $do > svc.yaml

# Specify a service type
k expose deployment web --port=80 --type=NodePort $do > svc.yaml
k expose deployment web --port=80 --type=LoadBalancer $do > svc.yaml
# (default is ClusterIP if --type is omitted)

# Inspect
k get svc
k get endpoints <svc-name>           # which pods this service points at
k describe svc <name>

# Delete
k delete svc <name>
```

---

## 8. ConfigMaps & Secrets

```bash
# ConfigMap
k create configmap app-config --from-literal=ENV=prod $do > cm.yaml
k create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=debug $do > cm.yaml
k create configmap app-config --from-file=config.properties $do > cm.yaml
k create configmap app-config --from-env-file=app.env $do > cm.yaml

# Secret (generic) — values are base64-encoded for you
k create secret generic db-creds --from-literal=username=admin --from-literal=password=s3cr3t $do > secret.yaml
k create secret generic tls-files --from-file=cert.pem --from-file=key.pem $do > secret.yaml

# Secret (TLS-specific shorthand)
k create secret tls my-tls --cert=path/to/cert --key=path/to/key $do > secret.yaml

# Inspect
k get cm
k get cm app-config -o yaml
k get secret
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d   # decode a value
```

---

## 9. Jobs & CronJobs

```bash
# Job (runs once to completion) — note $do BEFORE --, command last
k create job pi --image=perl $do -- perl -Mbignum=bpi -wle "print bpi(100)" > job.yaml

# CronJob (recurring; cron-format schedule)
k create cronjob backup --image=busybox --schedule="0 */6 * * *" $do -- echo "backup" > cron.yaml

# Multiple commands always need sh -c (see section 2, "The -- separator")
k create job migrate -n batch --image=busybox -- sh -c "date && echo done"

# Inspect
k get jobs
k get cronjobs                       # short: cj
k logs job/<name>                    # logs from the job's pod

# Trigger a CronJob manually (creates a one-off Job from its template)
k create job --from=cronjob/backup manual-backup
```

---

## 10. Namespaces

```bash
# Create
k create namespace dev
k create namespace dev $do > ns.yaml

# Make a namespace your default for this context (saves typing -n constantly)
k config set-context --current --namespace=dev

# Run a one-off in a different namespace
k get pods -n kube-system
k get pods -A                        # all namespaces

# Delete (removes everything inside the namespace — destructive)
k delete namespace dev
```

---

## 11. Labels, Selectors & Annotations

```bash
k get pods -l app=App1                        # equality selector (short for --selector)
k get pods -l app=App1,function=Front-end     # comma = AND
k get pods -l 'env in (dev,staging)'          # set-based: in / notin
k get pods -l '!tier'                         # objects WITHOUT a tier label
k get pods --show-labels
k get pods -L app -L function                 # show labels as COLUMNS (-L)

k label pod <pod> tier=frontend               # add
k label pod <pod> tier=backend --overwrite    # change
k label pod <pod> tier-                       # remove (trailing dash)
k annotate pod <pod> description="payments frontend"
```

Full reference: [`commands/labels-selectors.md`](commands/labels-selectors.md).

---

## 12. Scheduling — Taints, Tolerations, Node Selectors, Affinity

```bash
# Taints (node repels) — key=value:effect  (NoSchedule | PreferNoSchedule | NoExecute)
k taint nodes <node> app=blue:NoSchedule
k taint nodes <node> app=blue:NoSchedule-     # remove (trailing dash)
k describe node <node> | grep -A1 Taints

# Node labels + nodeSelector (pod attracts to node)
k label nodes <node> size=Large
k label nodes <node> size-                    # remove
k get nodes -l size=Large
k get nodes --show-labels
```

Toleration / nodeSelector / nodeAffinity have no imperative flag — `$do > pod.yaml` and edit. Node affinity operators: `In, NotIn, Exists, DoesNotExist, Gt, Lt`; rule types `requiredDuringSchedulingIgnoredDuringExecution` (hard) / `preferred...` (soft, weighted). **Dedicated node** = same `key=value` as both a taint and a label. Full reference: [`commands/scheduling.md`](commands/scheduling.md).

---

## 13. Security Contexts

No imperative flag — generate YAML and add the block. Pod-level sets defaults; container-level overrides. `capabilities` is container-level only.

```yaml
spec:
  securityContext: { runAsUser: 1000, runAsNonRoot: true }
  containers:
    - name: app
      securityContext:
        runAsUser: 2000
        capabilities: { add: ["NET_ADMIN"], drop: ["CHOWN"] }
```

```bash
k exec <pod> -- id            # verify uid/gid
k exec <pod> -- capsh --print # verify capabilities
```

Docker→K8s: `--user`→`runAsUser`, `--cap-add`→`capabilities.add`, `--privileged`→`privileged`. Full reference: [`commands/security-contexts.md`](commands/security-contexts.md).

---

## 14. Service Accounts

```bash
k create serviceaccount build-bot             # short: k create sa build-bot
k get sa
k create token build-bot                       # ~1h bound token (1.24+)
k create token build-bot --duration=24h
k set serviceaccount deployment/my-deploy build-bot   # attach to a workload
```

Pod YAML: `spec.serviceAccountName: build-bot` (immutable on a live bare Pod); `automountServiceAccountToken: false` to opt out. Full reference: [`commands/service-accounts.md`](commands/service-accounts.md).

---

## 15. Multi-Container & Init Containers

Init containers run to completion, in order, before app containers. Native sidecar = init container with `restartPolicy: Always`.

```bash
k get pod <pod> -w                    # watch Init:0/2 -> Init:1/2 -> Running
k describe pod <pod>                   # "Init Containers" section
k logs <pod> -c <container>           # required once >1 container
k logs <pod> --all-containers=true
k exec -it <pod> -c <container> -- sh
```

Full reference: [`commands/multi-container-pods.md`](commands/multi-container-pods.md).

---

## 16. Probes — Readiness & Liveness

Readiness failing → removed from Service endpoints (READY 0/1), no restart. Liveness failing → container killed and restarted (RESTARTS climbs). No imperative flag — edit YAML.

```bash
k explain pod.spec.containers.readinessProbe
k get pod app                          # READY 0/1 vs 1/1; RESTARTS climbing
k describe pod app                     # "probe failed" / "Killing" events
```

Handlers: `httpGet` / `tcpSocket` / `exec`. Timing: `initialDelaySeconds`, `periodSeconds`, `failureThreshold`. Full reference: [`commands/probes.md`](commands/probes.md).

---

## 17. Metrics — `kubectl top`

```bash
k top node
k top pod
k top pod --containers
k top pod -l app=myapp
k get apiservice v1beta1.metrics.k8s.io    # is metrics-server registered?
```

Needs metrics-server (kind/minikube: add `--kubelet-insecure-tls`). Richer log flags (`--previous`, `--tail`, `--since`, `deploy/<name>`) in [`commands/observability.md`](commands/observability.md).

---

## 18. Deployment Strategies — Blue/Green & Canary

```bash
# Blue/green: flip the Service selector
k patch service my-service -p '{"spec":{"selector":{"version":"v2"}}}'
k get endpoints my-service                 # confirm switch

# Canary: shared label, ratio via replica counts
k scale deployment myapp-canary --replicas=2
k set image deployment/myapp-primary app-container=myapp-image:2.0   # promote
k delete deployment myapp-canary
```

Full reference: [`commands/deployment-strategies.md`](commands/deployment-strategies.md).

---

## 19. StatefulSets & Headless Services

```bash
k get sts
k scale sts mysql --replicas=5             # ordered; scale-down removes highest ordinal first
k set image sts/mysql mysql=mysql:8.1
k patch sts mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'  # canary
k rollout status sts/mysql
k delete sts mysql --cascade=orphan        # leave pods
k delete pvc -l app=mysql                  # PVCs are NOT auto-deleted

# Headless service (clusterIP: None) → per-pod DNS
k get svc mysql-h                          # CLUSTER-IP: None
k exec -it mysql-0 -- nslookup mysql-0.mysql-h.default.svc.cluster.local
```

Full reference: [`commands/statefulsets.md`](commands/statefulsets.md).

---

## 20. Network Policies

No imperative generator — hand-write YAML. Selected pods become default-deny for the covered direction.

```bash
k get netpol
k describe netpol db-policy
k get pods --show-labels                   # find podSelector labels first
k get ns --show-labels                     # find namespaceSelector labels
k exec -it <client> -- nc -zv <db-svc> 3306   # test (allowed=connects, denied=hangs)
```

AND/OR trap: peers under one `-` = AND; separate `-` entries = OR. Full reference: [`commands/network-policies.md`](commands/network-policies.md).

---

## 21. Ingress

```bash
k create ingress web \
  --rule="my-store.com/wear*=wear-service:80" \
  --rule="my-store.com/watch*=watch-service:80"     # trailing * => pathType: Prefix
k create ingress web --rule="/wear*=wear-service:80" $do > ing.yaml
k describe ingress web                     # host/path -> backend table
```

Extra flags: `--class`, `--default-backend`, `--annotation`. v1 traps: `pathType` required; backend is `service.name` + `service.port.number`. Full reference: [`commands/ingress.md`](commands/ingress.md).

---

## 22. Storage — Volumes, PV/PVC, StorageClasses

```bash
k get pv                                   # capacity, access modes, reclaim policy, status, claim
k get pvc                                  # Pending -> Bound
k get pv,pvc,pod                            # one-shot check: Bound / Bound / Running
k describe pvc myclaim                      # why still Pending
k get pvc myclaim -o jsonpath='{.spec.volumeName}'
k get sc                                    # which is (default)?
k describe sc standard                      # provisioner, volumeBindingMode
```

No generator for PV/PVC/StorageClass — type the YAML. Apply PV+PVC and confirm `Bound` before writing the pod that consumes it. PVC must repeat the PV's `storageClassName` and `accessModes` to bind; size is `capacity.storage` on the PV but `resources.requests.storage` on the PVC. StorageClass is `storage.k8s.io/v1` (not core `v1`) and has **no `spec:`** — `provisioner` is top-level.

`WaitForFirstConsumer` keeps a PVC Pending until a pod consumes it (normal). Full reference: [`commands/storage.md`](commands/storage.md); memorizable skeleton and timed drill: [`08-state-persistance/08-pv-pvc-speed-run.md`](08-state-persistance/08-pv-pvc-speed-run.md).

---

## 23. Kubeconfig & Authentication

```bash
k config get-contexts
k config use-context <name>
k config set-context --current --namespace=<ns>
k config view --minify

# Issue a client cert for a user
openssl genrsa -out casey.key 2048
openssl req -new -key casey.key -out casey.csr -subj "/CN=casey/O=developers"
openssl x509 -req -in casey.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out casey.crt -days 365
k config set-credentials casey --client-certificate=casey.crt --client-key=casey.key
```

No User object (`k create user` doesn't exist). Full reference: [`commands/kubeconfig.md`](commands/kubeconfig.md).

---

## 24. RBAC

```bash
k create role developer --verb=get,list,create --resource=pods -n default
k create rolebinding dev-binding --role=developer --user=dev-user -n default
k create rolebinding sa-binding --role=developer --serviceaccount=default:my-sa
k create clusterrole node-reader --verb=get,list --resource=nodes
k create clusterrolebinding casey-nodes --clusterrole=node-reader --user=casey

k auth can-i create deployments
k auth can-i --list -n finance
k auth can-i create pods --as dev-user -n default      # impersonation
```

Full reference: [`commands/rbac.md`](commands/rbac.md).

---

## 25. Authorization, API Groups & Admission

```bash
k api-resources                            # NAME, SHORTNAMES, APIVERSION, NAMESPACED, KIND
k api-resources --namespaced=false          # cluster-scoped (Role vs ClusterRole decision)
k api-versions
kubectl proxy                               # local proxy, forwards kubeconfig auth
curl http://localhost:8001/apis

cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission
```

Full reference: [`commands/authorization.md`](commands/authorization.md).

---

## 26. Custom Resources (CRDs)

```bash
k create -f crd-definition.yml             # register the type first
k create -f myresource.yml                 # then instances are accepted
k api-resources | grep <name>              # confirm registration
k get <singular>   /   k get <shortname>
```

Full reference: [`commands/crd.md`](commands/crd.md).

---

## 27. Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search hub wordpress                  # Artifact Hub  |  helm search repo (local)
helm install wordpress bitnami/wordpress --set replicaCount=5 -f values.yaml
helm upgrade wordpress bitnami/wordpress --set replicaCount=5
helm rollback wordpress 1
helm uninstall wordpress
helm list -A
helm history wordpress
helm show values bitnami/wordpress
helm template <release> <chart>            # render, no install
helm pull --untar bitnami/wordpress
```

Value precedence: `--set` > `-f` > chart defaults. Full reference: [`commands/helm.md`](commands/helm.md).

---

## 28. Kustomize

Remember: `-k` runs the base+overlay merge; `-f` applies raw files.

```bash
k apply -k <dir>                           # build + apply
k delete -k <dir>
k kustomize <dir>                          # render to stdout (no apply)
kustomize build <dir> | kubectl apply -f -
kustomize create --autodetect --recursive
kustomize edit set image nginx=haproxy:2.4
kustomize edit set namespace dev
kustomize edit add resource ../../base
kustomize edit add patch --path patch.yaml --group apps --version v1 --kind Deployment --name api
```

Patches: strategic merge (keyed) or JSON6902 (`op`/`path`/`value`, positional list ops). No native rollback (`kubectl rollout undo`). Full reference: [`commands/kustomize.md`](commands/kustomize.md).

---

## 29. Force-delete & common gotchas

```bash
# Pod stuck in Terminating
k delete pod <name> $now             # = --force --grace-period=0

# A deleted pod immediately reappears
# → It's managed by a Deployment/ReplicaSet. Delete the controller, or scale to 0:
k delete deployment <name>
k scale deployment <name> --replicas=0

# "I can't change the image on this pod" / "edit failed"
# → Pods are mostly immutable. Delete and recreate, or edit the parent Deployment.
k get pod <name> -o yaml > pod.yaml
k delete pod <name>
# edit pod.yaml, then:
k apply -f pod.yaml

# Lost track of which context / namespace you're in
k config current-context
k config view --minify | grep namespace
```

---

## 30. Vim quick reference

| Action | Command |
|---|---|
| Open file | `vim file.yaml` |
| Enter Insert mode | `i` |
| Leave Insert mode | `Esc` |
| Save | `:w` |
| Save and quit | `:wq` |
| Quit without saving | `:q!` |
| Jump to line N | `:N` |
| Go to top / bottom | `gg` / `G` |
| Search forward | `/word` then Enter (then `n` for next match) |
| Delete current line | `dd` |
| Copy current line | `yy` |
| Paste below cursor | `p` |
| Undo / Redo | `u` / `Ctrl+r` |

---

## See also

- [`commands/`](commands/) — per-topic focused files (start with [`imperative.md`](commands/imperative.md))
- `00-local-lab-setup.md` — full lab setup, vim survival kit, exam environment
- `03-pods.md` — pod concepts, lifecycle, multi-container patterns
- `04-pod-yaml.md` — YAML structure, real-world walkthrough, template-edit-apply workflow
