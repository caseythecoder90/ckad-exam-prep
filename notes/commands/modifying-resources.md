# Modifying existing resources — set vs patch vs edit vs apply

> **Why this matters on the exam.** [`imperative.md`](imperative.md) covers creating resources fast. But a large share of CKAD tasks hand you a resource that already exists and ask you to *change* it — add a probe, swap an image, fix a strategy, attach a ServiceAccount. There are four ways to do that and they differ by up to a minute each. Picking the right one on reflex is worth more marks than knowing any single YAML field.

---

## The decision tree

Ask these in order:

| # | Question | Use | Typical time |
|---|---|---|---|
| 1 | Does a `kubectl set` subcommand exist for this field? | `k set ...` | ~5s |
| 2 | Do I know the exact field path? | `k patch` | ~15s |
| 3 | Otherwise, is the change small? | `k edit` | ~30s |
| 4 | Does the task want a **file**, or is the field **immutable**? | export → edit → `apply` / `replace` | ~60s |

Steps 1–3 change the live cluster only. Step 4 is mandatory when the question
says "save the YAML at `/path/file.yaml`" — graders check the file *and* the
cluster state.

---

## 1. `kubectl set` — six subcommands, memorize them

These are the only fields with a dedicated verb. If your task touches one of
them, nothing else is faster.

```bash
k set image deploy/web nginx=nginx:1.31-alpine     # container image
k set env deploy/web APP_VERSION=2                 # env var (also triggers a rollout)
k set resources deploy/web --requests=memory=20Mi --limits=memory=50Mi
k set serviceaccount deploy/web my-sa              # serviceAccountName
k set selector svc/web-svc 'app=web,version=blue'  # Service selector (blue-green flip)
k set subject rolebinding/rb --user=jane           # RBAC subjects
```

Useful variants:

```bash
k set env deploy/web --list                    # show current env
k set env deploy/web KEY-                      # remove a var (trailing dash)
k set env deploy/web --from=configmap/app-cfg  # import all keys
k set image deploy/web '*=nginx:1.31'          # every container in the pod
```

> `k set env` on a Deployment changes the pod template, which triggers a new
> rollout. That makes it the one-liner answer to "update X **and** trigger a
> new rollout" questions.

---

## 2. `kubectl patch` — when you know the path

Default patch type is **strategic merge** for built-in resources: you supply
only the nested fragment you want changed.

```bash
# rolling update strategy (no `set` subcommand exists for this)
k patch deploy cassini -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":2,"maxUnavailable":0}}}}'

# ClusterIP -> NodePort with a fixed port
k patch svc jupiter-crew-svc -p '{"spec":{"type":"NodePort","ports":[{"port":8080,"targetPort":80,"nodePort":30100}]}}'

# make a StorageClass the cluster default
k patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Lists are the trap.** A strategic merge patch *replaces* a list unless the
API defines a merge key — that's why the Service example above must repeat
`port` and `targetPort`, not just `nodePort`. When you need surgical list
edits, use a JSON patch instead:

```bash
k patch deploy web --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/ports/0/hostPort","value":80}]'
```

JSON patch ops: `add`, `replace`, `remove`, `copy`, `move`, `test`. Array
indexes are zero-based; `/-` appends to a list.

Sanity-check a patch before running it for real:

```bash
k patch deploy web -p '<json>' --dry-run=server -o yaml | less
```

---

## 3. `kubectl edit` — the safe default

No syntax to recall, works on any field. Opens the live object in `$KUBE_EDITOR`
(default `vi`); saving applies immediately.

```bash
k edit deploy web
k edit svc web-svc -n venus
```

- If your YAML is invalid or the change is rejected, kubectl **keeps your work**
  in a temp file and prints the path — re-edit that file and `k apply -f` it.
- Some fields are silently ignored rather than rejected (see immutability below);
  always verify with `k get -o yaml` afterwards.
- Set `export KUBE_EDITOR=vim` in your exam shell if you prefer vim's keymap.

---

## 4. Export → edit → apply — when you must

Required in exactly two situations:

**(a) The task wants a file.** Very common phrasing: *"Save the Deployment YAML
at `/course/9/holy-api-deployment.yaml`"*.

```bash
k get deploy web -o yaml > d.yaml     # live object, includes status/metadata noise
vim d.yaml
k apply -f d.yaml
```

**(b) The field is immutable.** The API rejects the change in place, so the
object must be recreated:

```bash
k replace -f d.yaml --force           # = delete + recreate, one command
```

Commonly immutable fields on the CKAD:

| Resource | Immutable |
|---|---|
| Deployment / ReplicaSet | `spec.selector` |
| Job | almost all of `spec` (`completions`, `parallelism` are editable-ish; template is not) |
| Pod | nearly everything except `image`, `activeDeadlineSeconds`, tolerations (additions) |
| Service | `spec.clusterIP` |
| PVC | `storageClassName`, `accessModes` (size can grow if the SC allows) |

> Editing a **Pod** is the classic trap: to change almost anything you must
> `k get pod x -o yaml > p.yaml`, edit, then `k replace -f p.yaml --force`.
> Deployments don't have this problem because the controller recreates pods.

---

## Cleaning exported YAML

A live object carries fields you usually want to strip before re-applying —
`status`, `metadata.uid`, `resourceVersion`, `creationTimestamp`,
`managedFields`, and the `kubectl.kubernetes.io/last-applied-configuration`
annotation. They're harmless with `apply` in most cases but will break
`create` and can conflict on `replace`.

```bash
# kubectl 1.21+ removed --export; do it by hand, or generate fresh instead:
k get deploy web -o yaml | grep -v 'creationTimestamp\|resourceVersion\|uid:\|selfLink' > d.yaml
```

Faster alternative when the object is simple: regenerate it from scratch with
`k create ... $do` and copy over only the parts you need.

---

## Exam gotchas

- **Namespace on every command.** A perfect edit in the wrong namespace scores
  zero. Consider `k config set-context --current --namespace=<ns>` when a
  question has several parts in one namespace.
- **`k set env` triggers a rollout; `k patch` on `spec.strategy` does not.**
  If a task says "afterwards trigger a new rollout", the env change *is* the
  trigger — you don't need `rollout restart` as well.
- **`k rollout restart deploy/x`** restarts pods without changing spec — use it
  when a task wants pods recreated (e.g. to pick up a changed ConfigMap).
- **Verify with jsonpath, not eyeballs**, when a field is nested:
  ```bash
  k get deploy cassini -o jsonpath='{.spec.strategy.rollingUpdate}{"\n"}'
  ```
- **Find the path without leaving the terminal:**
  ```bash
  k explain deploy.spec.strategy.rollingUpdate
  k explain pod.spec.containers.securityContext --recursive | head -30
  ```

---

## Worked comparison

Task: *"Deployment `cassini` (4 replicas) should allow up to 2 surge Pods, 0
unavailable. Then trigger a rollout by setting `APP_VERSION=2`."*

```bash
# fastest — two commands, no editor (~20s)
k -n mercury patch deploy cassini -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":2,"maxUnavailable":0}}}}'
k -n mercury set env deploy cassini APP_VERSION=2
```

```bash
# also correct — one editor session + one command (~40s)
k -n mercury edit deploy cassini        # add strategy.rollingUpdate block
k -n mercury set env deploy cassini APP_VERSION=2
```

```bash
# slowest, and only needed if the task asked for a file (~70s)
k -n mercury get deploy cassini -o yaml > c.yaml
vim c.yaml
k apply -f c.yaml
```

Caveat on the patch: it assumes `strategy.type` is already `RollingUpdate`
(the default). If the Deployment uses `Recreate`, include
`"type":"RollingUpdate"` in the same patch or the `rollingUpdate` block is
rejected.
