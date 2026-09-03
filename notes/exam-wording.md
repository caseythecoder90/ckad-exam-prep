# Exam wording → YAML field

> **Why this matters.** CKAD questions are written in prose, but graded on
> fields. Most lost time is spent translating a sentence into the right key,
> not typing the YAML. The exam reuses a small set of phrasings — learn these
> and a question becomes a lookup rather than a puzzle.

---

## Rollouts and replicas

| The question says | You write |
|---|---|
| "up to N additional Pods may be created above the desired count" | `spec.strategy.rollingUpdate.maxSurge: N` |
| "no Pod may be unavailable at any time" | `maxUnavailable: 0` |
| "at most N Pods unavailable during the update" | `maxUnavailable: N` |
| "all Pods are replaced at once" / "downtime is acceptable" | `spec.strategy.type: Recreate` |
| "trigger a new rollout" | change anything in the pod template (`k set env` is the one-liner) |
| "undo the last change" | `k rollout undo deploy/x` |
| "go back to revision N" | `k rollout undo deploy/x --to-revision=N` |

## Probes

| The question says | You write |
|---|---|
| "only runs during the start of the container, not continuously" | `startupProbe` |
| "checks whether the container can receive traffic" | `readinessProbe` |
| "restart the container if it becomes unhealthy" | `livenessProbe` |
| "wait N seconds before the first check" | `initialDelaySeconds: N` |
| "check every N seconds" / "periodically N" | `periodSeconds: N` |
| "reachable on port P" (no path mentioned) | `tcpSocket: {port: P}` |
| "HTTP GET on / port 80" | `httpGet: {path: /, port: 80}` |
| "fails after N consecutive failures" | `failureThreshold: N` |

## Images and lifecycle

| The question says | You write |
|---|---|
| "only pull the image if it's not already on the node" | `imagePullPolicy: IfNotPresent` |
| "never pull the image" | `imagePullPolicy: Never` |
| "always pull" | `imagePullPolicy: Always` |
| "N seconds to shut down gracefully" | `terminationGracePeriodSeconds: N` |
| "run before the app container starts" | `initContainers:` |
| "runs alongside the app container" | a second entry in `containers:` (sidecar) |

## Jobs

| The question says | You write |
|---|---|
| "N completions in total" | `spec.completions: N` |
| "M running in parallel" | `spec.parallelism: M` |
| "give up after N seconds" | `spec.activeDeadlineSeconds: N` |
| "retry at most N times" | `spec.backoffLimit: N` |
| "keep finished Pods for N seconds" | `spec.ttlSecondsAfterFinished: N` |
| "Pods are labelled `k: v`" | `spec.template.metadata.labels` (**not** the Job's own labels) |

## Identity, security, scheduling

| The question says | You write |
|---|---|
| "the Pods should run under ServiceAccount X" | `spec.template.spec.serviceAccountName: X` (**pod** level) |
| "Container SecurityContext" | `containers[].securityContext` |
| "Pod SecurityContext" | `spec.securityContext` |
| "must not be able to escalate privileges" | `allowPrivilegeEscalation: false` |
| "run as user N" | `runAsUser: N` |
| "schedule only on nodes labelled X" | `nodeSelector` (or `affinity` if the wording says "preferred") |
| "must tolerate the taint" | `tolerations:` |

## Services and networking

| The question says | You write |
|---|---|
| "available on all nodes on port P" | `type: NodePort`, `nodePort: P` (30000–32767) |
| "expose internally" / "cluster-internal only" | `type: ClusterIP` (the default) |
| "the Service should point to Deployment X's Pods" | edit `spec.selector` to X's pod labels |
| "the containers run on port P" | `targetPort: P` (≠ `port`, the Service port) |
| "route path /foo to service X" | Ingress rule with `pathType` (required in v1) |

## Storage and config

| The question says | You write |
|---|---|
| "the volume survives Pod restarts" | PVC + `persistentVolumeClaim` (not `emptyDir`) |
| "scratch space shared between containers" | `emptyDir: {}` |
| "keep the volume after the PVC is deleted" | StorageClass `reclaimPolicy: Retain` |
| "under data key `index.html`" | `--from-file=index.html=/path/file` (the `key=` prefix!) |
| "mount the same volume as the other container" | same `volumes` entry, `mountPath` may differ |

---

## Structural traps worth memorizing

- **`serviceAccountName`** is a pod-spec field — not on the container, not on
  the Deployment spec. (Most-missed field on the exam.)
- **`securityContext`** exists at *both* pod and container level, with
  different allowed keys. Read which one the question names.
- **StorageClass** puts `provisioner` and `reclaimPolicy` at the **top level**,
  not under `spec` — nearly unique among CKAD resources.
- **`pathType`** is mandatory on every Ingress path in `networking.k8s.io/v1`.
- **Job labels**: "Pods are labelled X" means `spec.template.metadata.labels`.
- **Shell syntax in commands** (`&&`, `|`, `>`) requires an explicit shell:
  `-- sh -c "sleep 2 && echo done"`. Passing it raw makes the binary receive
  the operators as literal arguments.
- **A crashed container's logs** need `k logs <pod> --previous`.
- **"Save the YAML at /path"** means the file is graded too — do both the file
  and the apply.

---

## Time triage

The exam is ~2 hours for 15–20 questions, weighted (a 3% question and a 8%
question look identical). Practical approach:

1. **First pass** — answer everything you can finish in under ~4 minutes. Flag
   the rest (the exam UI has a flag button) and move on.
2. **Second pass** — return to flagged questions, highest weight first.
3. **Never leave a question blank.** Partial credit is real: a Deployment with
   the right name, image, and replicas but a missing probe still scores.
4. **Verify as you go** — `k get`/`k describe` immediately after each change,
   because a silently-ignored edit costs the whole question.

Related: [`commands/imperative.md`](commands/imperative.md) for generating
manifests fast, [`commands/modifying-resources.md`](commands/modifying-resources.md)
for changing existing ones, and [`commands/setup.md`](commands/setup.md) for
the aliases these rely on.
