# Logging in Kubernetes

> **Section:** 04-observability
> **Course chapter:** 3 (Logging Mechanisms)
> **Why this is in CKAD:** `kubectl logs` is a primary debugging tool and a guaranteed exam touchpoint. You must know how to follow logs, target a specific container in a multi-container pod, and read previous-instance logs after a crash.
> **Companion files:** `02-liveness-probes.md` (a liveness restart is one reason you reach for `--previous` logs); `../03-multi-container-pods/02-design-patterns.md` (multi-container pods are why `-c` exists)

---

## 1. Logging in Docker vs Kubernetes (instructor's framing)

In Docker you run a container (e.g. the `event-simulator` app) and read its stream with `docker logs -f <id>`, where `-f` follows the live stream.

Kubernetes gives you the equivalent at the pod level:

```bash
kubectl create -f event-simulator.yaml
kubectl logs -f event-simulator-pod        # -f follows the live log stream
```

`kubectl logs` reads the container's stdout/stderr, exactly what the app writes to the console. The `-f` flag streams new lines as they appear (same meaning as in `docker logs -f` and `tail -f`).

## 2. Multi-container pods: you must name the container

If a pod runs a single container, `kubectl logs <pod>` is unambiguous. The moment a pod has **more than one** container, you must say which one - otherwise kubectl errors and asks you to choose.

The instructor's example pod has two containers (`event-simulator` and `image-processor`):

```bash
kubectl logs -f event-simulator-pod event-simulator
#                  <pod>              <container>
```

Note the slide passes the container name as a **positional argument** (`logs -f <pod> <container>`). The more explicit and more common form uses `-c`:

```bash
kubectl logs -f event-simulator-pod -c event-simulator
```

Both work. `-c <container>` is the form to memorize - it is identical to the flag used for `kubectl exec`, init containers, and sidecars, so it is one rule across everything.

## 3. Why your pod names have a random hash (your work question)

You noticed work pod names look like `myapp-7d4f9c8b6d-x7k2p`, while the lecture's pod is just `event-simulator-pod`. That difference is the tell for **how the pod was created**:

- **Bare pod** (created directly from a pod YAML, like the lecture): you choose the exact name, no suffix. `event-simulator-pod`.
- **Pod owned by a Deployment** (almost everything in production, like your work apps): the name is generated through a two-level chain.

```
Deployment:   myapp
  -> ReplicaSet:  myapp-7d4f9c8b6d          # hash = stable encoding of the pod template
       -> Pod:        myapp-7d4f9c8b6d-x7k2p   # final 5 chars = random, per pod
```

So the long hash identifies the ReplicaSet's pod template (it changes when you change the template, e.g. a new image), and the short trailing suffix makes each replica's name unique. There is no way to predict it - copying the full name out of `kubectl get pods` is the correct workflow, which is exactly what you are doing.

Forward pointer (covered properly in the Deployments material): once you are dealing with Deployments you get friendlier options that avoid the hash entirely -

```bash
kubectl logs -f deploy/myapp                 # logs from a pod managed by the Deployment
kubectl logs -f -l app=myapp                 # logs by label selector (can span pods with --prefix)
```

These are not in this lecture, but they are why you rarely need to type the hash once Deployments are in play.

## 4. Useful flags beyond the lecture

The lecture only shows `-f`. These are the ones worth having for the exam and for work:

| Flag | What it does |
|---|---|
| `-f` | follow/stream live (the one you use at work) |
| `-c <container>` | target a specific container in a multi-container pod |
| `--previous` / `-p` | logs from the **previous** container instance - essential after a crash/restart to see why it died |
| `--tail=N` | only the last N lines |
| `--since=10m` / `--since-time=...` | time-bounded output |
| `--timestamps` | prefix each line with its timestamp |
| `--all-containers=true` | logs from every container in the pod at once |
| `-l <selector>` | logs from pods matching a label (multi-pod) |

`--previous` is the high-value one: when a container is in `CrashLoopBackOff`, the current instance may have no useful output yet, but `kubectl logs <pod> -c <container> --previous` shows the logs from the instance that just died - usually where the actual error is.

## 5. What `kubectl logs` does and does not do

Depth worth knowing so you are not surprised at work:

- It reads **stdout/stderr only**. If the app writes to a log *file* inside the container instead of the console, `kubectl logs` shows nothing - that is exactly the problem a logging sidecar solves (tail the file, ship it; see the EFK sidecar in `../03-multi-container-pods/02-design-patterns.md`).
- Logs live and die with the container. When a pod is deleted, its logs are gone; `--previous` only reaches back **one** instance. Durable history needs a real log backend (Elastic, Loki, etc.) - kubectl is for live/recent debugging, not retention.

## 6. Exam-pattern gotchas

- **Multi-container pod without `-c` fails.** kubectl returns an error listing the containers and refuses to guess. Add `-c <container>`.
- **`-c` is the universal container flag** - same as `exec`, init/sidecar logs. One rule everywhere.
- **`--previous` for crash debugging.** If a container restarted (liveness kill, OOM, panic), the cause is in the previous instance's logs, not the current one.
- **No log output is a real signal**, not always a bug: the app may be logging to a file, not the console.

## 7. Imperative shortcuts / command reference

```bash
kubectl logs <pod>                              # one-shot dump (single-container pod)
kubectl logs -f <pod>                           # follow live
kubectl logs -f <pod> -c <container>            # specific container (multi-container)
kubectl logs <pod> -c <container> --previous    # previous crashed instance
kubectl logs <pod> --tail=50 --timestamps       # last 50 lines, timestamped
kubectl logs -f deploy/<name>                    # via a Deployment (skips the hash)
```

## 8. TL;DR / takeaways

- `kubectl logs -f <pod>` follows a pod's stdout/stderr live (the Kubernetes equivalent of `docker logs -f`).
- In a **multi-container pod you must name the container** - positionally (`logs <pod> <container>`) or, preferably, with `-c <container>`.
- Random-hash pod names (`myapp-<hash>-<rand>`) mean the pod is owned by a **Deployment -> ReplicaSet**; bare pods keep the literal name you gave them.
- `--previous` shows the prior crashed instance's logs - the go-to for `CrashLoopBackOff`.
- `kubectl logs` only sees the console stream and only keeps live/recent output; file logs and long-term retention need a sidecar/backend.

---

### Open threads
- [ ] **Deployments** section: revisit `logs deploy/<name>` and `-l` selector logging; explain the ReplicaSet hash properly when ownership/labels are covered.
- [ ] Cross-link the **EFK logging sidecar** (`../03-multi-container-pods/02-design-patterns.md`) as the answer to "app logs to a file, not stdout."
