# Observability — Logs & Metrics

The full `kubectl logs` toolkit and `kubectl top`. For `exec`/`describe`/`events`, see `debugging.md`.

## Logs

```bash
k logs <pod>                          # one-shot dump (single-container pod)
k logs -f <pod>                       # follow the live stream
k logs <pod> -c <container>          # target a container (multi-container pod)
k logs <pod> <container>             # container as positional arg (same effect)
k logs <pod> -c <container> --previous # PREVIOUS crashed instance — for CrashLoopBackOff
k logs <pod> --tail=50 --timestamps   # last 50 lines, timestamped
k logs <pod> --since=10m              # time-bounded (also --since-time=...)
k logs <pod> --all-containers=true    # every container in the pod
k logs -f deploy/<name>              # via a Deployment (skips the pod hash)
k logs -f -l app=myapp               # by label selector (spans pods; add --prefix)
```

Handy flags: `-f` follow · `-c` container · `--previous`/`-p` prior instance · `--tail=N` · `--since=` · `--timestamps` · `--all-containers=true` · `-l` selector.

## Metrics — `kubectl top` (needs metrics-server)

```bash
k top node                            # per-node CPU(cores), CPU%, MEM(bytes), MEM%
k top pod                             # per-pod CPU + memory
k top pod --containers                # break out per container
k top pod -l app=myapp                # filter by label
k top pod -n <namespace>
```

Verify / install metrics-server:

```bash
k get apiservice v1beta1.metrics.k8s.io       # is it registered/Available?
minikube addons enable metrics-server         # minikube
# kind/self-managed: deploy the sigs.k8s.io manifests; add --kubelet-insecure-tls for self-signed kubelet certs
```

`k top` returns "Metrics API not available" until metrics-server is running and registered.

## See also

- `04-observability/03-logging.md`, `04-monitoring-metrics-server.md`
- `debugging.md` — exec, describe, events, explain
