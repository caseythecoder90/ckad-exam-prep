---
section: 10-helm
chapter: "03"
title: "Helm on the Exam — Task Patterns and Traps"
examinable: true
related:
  - 01-helm-intro-and-installation.md
  - 02-helm-concepts.md
  - ../11-kustomize/12-exam-patterns.md
---

# 03 – Helm on the Exam — Task Patterns and Traps

The CKAD curriculum (Application Deployment, 20%) lists *"Use the Helm
package manager to deploy existing packages"*. You will not write a chart.
You will install, upgrade, roll back, inspect and delete releases of charts
that come from a repository already reachable from the exam node, and you
will need the `helm ls` variants to find what a question hides.

Every command below takes `-n <namespace>`. Releases are namespaced, and a
release in the wrong Namespace scores zero.

---

## 1. Find the chart, then find its knobs

```bash
helm repo list
helm repo add <name> <url>                       # only if the question's repo isn't there yet
helm repo update                                 # after add, before "upgrade to newest"
helm search repo <chart>                         # newest version of each match ONLY
helm search repo <chart> --versions              # every version, newest first
helm show values <repo>/<chart> [--version X]    # what --set / -f can change
helm show chart  <repo>/<chart>                  # Chart.yaml: version, appVersion
```

`CHART VERSION` is the package version — what `--version` selects. `APP
VERSION` is the software inside; informational. Questions that say
"version 2.1.0" mean the chart version unless they say otherwise.

Never guess a value name. `helm show values` first, then set exactly the
key it prints (`replicaCount`, `service.type`, `image.tag`).

---

## 2. Install

```bash
helm install <release> <repo>/<chart> -n <ns> \
  --version 2.1.0 \                              # omit = newest
  --create-namespace \                           # helm does NOT create Namespaces otherwise
  --set replicaCount=3 --set service.type=NodePort --set service.nodePort=30080 \
  -f my-values.yaml
```

| Flag | Notes |
|---|---|
| `--version` | chart version, not app version |
| `--create-namespace` | without it, `-n newns` fails with *namespaces "newns" not found* |
| `--set a.b=c` | nested keys with dots; `--set 'env[0].name=X,env[0].value=Y'` for lists |
| `--set-string image.tag=1.27` | force a string when Helm would parse a number/bool |
| `-f file.yaml` | a values file; repeatable; later files win |
| `--dry-run` | render, print, install nothing |
| `--wait --timeout 60s` | block until Pods are ready; default returns as soon as the API accepts the manifests |

Precedence: `--set` > `-f` > chart `values.yaml`. Lists and anything nested
more than two levels are easier in a values file than on the command line.

"Configure via Helm values" is graded with `helm get values <release>`:
fixing the replica count afterwards with `kubectl scale` leaves Helm's
record unchanged, and the next `helm upgrade` puts it back.

---

## 3. Upgrade — the trap that costs the most points

`helm upgrade` renders the chart from its default `values.yaml` plus **only
the values you pass on this command**. Previous `--set`s are forgotten:

```bash
helm install web repo/nginx --set replicaCount=4
helm upgrade web repo/nginx --set image.tag=1.27       # replicaCount silently back to 1
```

Two correct ways:

```bash
helm get values web -n <ns>                            # what it was installed with
helm upgrade web repo/nginx --reuse-values --set image.tag=1.27    # carry old values forward
helm upgrade web repo/nginx -f values.yaml --set image.tag=1.27    # or re-pass everything
```

Other upgrade facts:

- No `--version` = newest chart version in the (updated) repo index.
- `helm upgrade --install` = install if missing, upgrade if present.
- Editing a values file changes nothing until `helm upgrade -f` re-reads it.
- Each upgrade is a new revision (`helm history`).

---

## 4. History and rollback — a rollback is a new revision

```bash
helm history <release> -n <ns>
helm rollback <release> 2 -n <ns>        # to revision 2
helm rollback <release> -n <ns>          # no revision = the previous one
helm history <release> -n <ns>           # revision 4: "Rollback to 2"
```

Rolling back to 2 does not make 2 current; it creates revision **4** with
revision 2's contents. A question asking "which revision is deployed now"
after a rollback wants the new number. Same idea as `kubectl rollout undo`.

Helm reports `deployed` when the API server accepted the manifests, not
when the Pods are healthy. A broken image tag produces a `deployed` revision
with Pods in `ImagePullBackOff` — check `kubectl get pod` before deciding
which revision "worked".

---

## 5. Finding releases — `helm ls` hides things

```bash
helm ls -n <ns>          # deployed and failed releases in one Namespace
helm ls -A               # all Namespaces
helm ls -a               # all statuses: also pending-install/upgrade/rollback, uninstalled (--keep-history)
helm ls -A -a            # everything, everywhere — the one to remember
helm ls --pending        # only pending-*
helm ls -o json          # scriptable
```

A helm process killed mid-operation leaves the release in `pending-install`
or `pending-upgrade`. Plain `helm ls` does not show it. "There is a broken
release somewhere, find and delete it" means `helm ls -A -a`, then
`helm uninstall <release> -n <ns>`.

---

## 6. Inspect a release

```bash
helm status <release> -n <ns>
helm get values <release> -n <ns>                # user-supplied values only
helm get values <release> -n <ns> --all          # merged with chart defaults
helm get values <release> -n <ns> --revision 3
helm get manifest <release> -n <ns>              # rendered YAML Helm applied
helm get notes <release> -n <ns>
```

Where the state lives: one Secret per revision in the release's Namespace,
named `sh.helm.release.v1.<release>.v<revision>`, type `helm.sh/release.v1`.
`kubectl get secret -n <ns>` shows them; deleting the Namespace deletes the
release.

---

## 7. Render without installing

```bash
helm template <release> <repo>/<chart> -n <ns> -f values.yaml --set replicaCount=3 > out.yaml
kubectl apply -n <ns> -f out.yaml
```

`helm template` creates no release and no Secret; `helm ls` stays empty.
`-n` sets `.Release.Namespace` for templates that use it, but most charts
render no `metadata.namespace`, so the `kubectl apply` needs `-n` too.

`helm install --dry-run` also prints manifests but wraps them in status
text (NOTES, headers) — not clean YAML for a file.

---

## 8. Get a chart onto disk, install from disk

```bash
helm pull <repo>/<chart> --version 0.6.0 --untar -d /some/dir   # /some/dir/<chart>/Chart.yaml
vim /some/dir/<chart>/values.yaml                               # change the chart's defaults
helm install <release> /some/dir/<chart> -n <ns>                # a PATH installs from disk
helm lint /some/dir/<chart>
```

The chart argument is resolved by shape: `repo/name` is looked up in the
repo index; anything with a `/` or `./` prefix, or ending in `.tgz`, is a
path. Editing `values.yaml` inside the chart changes its defaults, so
`helm get values` shows nothing user-supplied.

---

## 9. Uninstall

```bash
helm uninstall <release> -n <ns>
helm uninstall <release> -n <ns> --keep-history   # release record stays; helm ls --uninstalled
```

`helm uninstall` removes every object the release created. Deleting the
Deployment with kubectl instead leaves the release record behind, and the
next `helm upgrade` recreates the Deployment.

---

## 10. Exam wording → command

| The question says | You run |
|---|---|
| "install version X of chart Y" | `helm install <rel> repo/Y --version X` |
| "into Namespace N, which does not exist" | `-n N --create-namespace` |
| "with N replicas, set via Helm" | `helm show values`, then `--set replicaCount=N` |
| "upgrade to the newest version" | `helm repo update`, `helm upgrade <rel> repo/Y` |
| "keep the values it was installed with" | `--reuse-values`, or `helm get values` and re-pass |
| "roll back to the previous working version" | `helm history`, `helm rollback <rel> [rev]` |
| "a release is stuck / broken / pending" | `helm ls -A -a` |
| "render the manifests, do not install" | `helm template <rel> chart ... > file` |
| "download / extract / inspect the chart" | `helm pull repo/Y --untar` |
| "install from the local chart" | `helm install <rel> ./dir` |
| "remove the release" | `helm uninstall <rel> -n N` |

---

## 11. Traps, collected

- `-n` on every command. `helm ls` with no `-n` is the current context
  Namespace, not everything.
- `--version` = chart version. Without it you get the newest and fail "install 2.1.0".
- `--create-namespace`, or the install fails.
- `helm upgrade` forgets previous `--set`s: `--reuse-values` or re-pass.
- Rollback creates a new revision.
- `deployed` does not mean healthy: look at the Pods.
- `helm ls` hides `pending-*`: `-a`.
- `helm template` + `kubectl apply` needs `-n` twice.
- Quote strings that look like numbers or booleans: `--set-string`, or
  `tag: "1.27"` in a values file.
- `helm repo update` before "newest version", or the index is stale.

## References

- [Using Helm](https://helm.sh/docs/intro/using_helm/) — install/upgrade/rollback/uninstall and repo management, with the flags above
- [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) — `--reuse-values`, `--reset-values`, `--install`, `--version`
- [helm list](https://helm.sh/docs/helm/helm_list/) — the status filters (`--all`, `--pending`, `--failed`, `--uninstalled`)
- [helm template](https://helm.sh/docs/helm/helm_template/) — client-side rendering and its flags
