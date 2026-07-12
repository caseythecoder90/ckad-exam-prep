# Probes — Readiness & Liveness

- **Readiness** — is the container ready to receive traffic? Failing removes the pod from Service endpoints (READY `0/1`), does NOT restart it.
- **Liveness** — is the container still healthy? Failing kills and restarts the container (RESTARTS climbs).

No imperative flag — `$do > pod.yaml` and hand-add the probe block.

## Build & inspect

```bash
k run app --image=simple-webapp --port=8080 $do > pod.yaml   # then add the probe
k apply -f pod.yaml --dry-run=client
k apply -f pod.yaml
k explain pod.spec.containers.readinessProbe
k explain pod.spec.containers.readinessProbe.httpGet
k explain pod.spec.containers.livenessProbe

k get pod app                 # READY 0/1 vs 1/1 (readiness); RESTARTS climbing (liveness)
k describe pod app            # Conditions + "Readiness/Liveness probe failed" / "Killing" events
```

## Probe YAML (three handler types)

```yaml
    readinessProbe:            # or livenessProbe — same handlers, same timing fields
      httpGet:                 # handler 1: HTTP GET, 200-399 = pass
        path: /healthz
        port: 8080
      # tcpSocket: { port: 3306 }        # handler 2: TCP connect succeeds
      # exec: { command: ["cat", "/tmp/ready"] }  # handler 3: exit 0 = pass
      initialDelaySeconds: 10   # wait before first probe
      periodSeconds: 5          # how often
      timeoutSeconds: 1
      failureThreshold: 3       # consecutive fails before acting
      successThreshold: 1
```

## Pod conditions (readiness gates endpoints)

```bash
k get pod app                 # READY column = ready containers / total
k describe pod app            # Conditions: PodScheduled / Initialized / ContainersReady / Ready
```

## See also

- `04-observability/01-readiness-probes.md`, `02-liveness-probes.md`
- `services.md` — a Service only routes to pods whose readiness probe passes
