# Security Contexts

Who the container process runs as and what kernel privileges it holds. **No imperative flag** — `kubectl run` has no `--security-context`. Always `$do > pod.yaml` and hand-edit the `securityContext` block.

## Two levels

```yaml
spec:
  securityContext:            # POD level — defaults for every container
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
  containers:
    - name: app
      image: nginx
      securityContext:        # CONTAINER level — overrides the pod for this container
        runAsUser: 2000
        capabilities:         # capabilities are CONTAINER-LEVEL ONLY
          add: ["NET_ADMIN"]
          drop: ["CHOWN"]
        privileged: false
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
```

Precedence: container-level wins over pod-level for that container. `capabilities` under a **pod**-level `securityContext` is a manifest error.

Key fields:
- `runAsUser` / `runAsGroup` — UID/GID the process runs as.
- `runAsNonRoot: true` — refuses to start if the resolved user is UID 0 → `CreateContainerConfigError`.
- `capabilities.add` / `.drop` — tune Linux capabilities (container level only).
- `privileged: true` — all capabilities + host device access; last resort.

## Verify (inside the running container)

```bash
k exec <pod> -- id                # effective uid/gid, e.g. uid=1000 gid=3000
k exec <pod> -- whoami
k exec <pod> -- capsh --print     # capabilities actually held (needs capsh in image)
```

## Docker → Kubernetes mapping

| Docker | Kubernetes `securityContext` |
|---|---|
| `docker run --user=1001` | `runAsUser: 1001` |
| `docker run --cap-add=NET_ADMIN` | `capabilities.add: ["NET_ADMIN"]` |
| `docker run --cap-drop=CHOWN` | `capabilities.drop: ["CHOWN"]` |
| `docker run --privileged` | `privileged: true` |
| (no equivalent) | `runAsNonRoot: true` |

Docker capability flags for reference: `--cap-add=<CAP>`, `--cap-drop=<CAP>` (repeatable, they tune the default set); `--privileged` grants all.

## See also

- `02-configuration/06-docker-security.md`, `07-security-contexts.md`
- `service-accounts.md` — the pod's API identity (separate from its Linux user)
