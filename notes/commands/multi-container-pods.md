# Multi-Container Pods & Init Containers

Containers in one pod share network (localhost) and can share volumes. **Init containers** run to completion, in order, before the app containers start. **Native sidecar** = an init container with `restartPolicy: Always` (starts, stays running alongside the app).

No imperative flag for multiple containers or init containers — `$do > pod.yaml`, then hand-add the extra `containers:` / `initContainers:` entries.

## Build

```bash
k run app --image=web-app --port=8080 $do > pod.yaml   # scaffold, then add containers/initContainers
k apply -f pod.yaml --dry-run=client                    # validate before applying
k apply -f pod.yaml
k explain pod.spec.containers
k explain pod.spec.initContainers
```

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup db; do sleep 2; done']
  containers:
    - name: app
      image: web-app
    - name: log-shipper           # sidecar
      image: log-shipper
```

## Inspect & logs

```bash
k get pod <pod> -w                     # watch Init:0/2 -> Init:1/2 -> Running
k describe pod <pod>                    # "Init Containers" section: state, exit code, restarts
k logs <pod> -c <container>            # logs from ONE container (required once >1 container)
k logs <pod> --all-containers=true     # every container at once
k logs <pod> -c init-myservice        # init container logs
k exec -it <pod> -c <container> -- sh # shell into a specific container
```

## Gotchas

- With >1 container, `k logs <pod>` errors without `-c` — it doesn't know which container.
- An init container that never exits blocks the pod at `Init:` forever. Check its logs with `-c`.
- A pod stuck at `Init:0/2` → the first init container hasn't completed; `describe` shows why.

## See also

- `03-multi-container-pods/01-multi-container-pods-intro.md`, `02-design-patterns.md`, `03-init-containers.md`
- `observability.md` — full `kubectl logs` variants
