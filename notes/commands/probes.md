# Probes — Readiness, Liveness & Startup

- **Readiness** — is the container ready to receive traffic? Failing removes the pod from Service endpoints (READY `0/1`), does NOT restart it.
- **Liveness** — is the container still healthy? Failing kills and restarts the container (RESTARTS climbs).
- **Startup** — has the container finished booting? Runs **only until it first succeeds**, then never again; while it runs, liveness and readiness are suspended. Use it for slow starters so liveness doesn't kill them mid-boot.

Exam phrasing → probe type:

| The question says | Use |
|---|---|
| "only runs during the start of the container, not continuously" | `startupProbe` |
| "checks it can receive traffic" / "should not get requests until…" | `readinessProbe` |
| "restart it if unhealthy" / "if it stops responding" | `livenessProbe` |

No imperative flag — `$do > pod.yaml` and hand-add the probe block.

## Memorize one block, not three

All three probes take the **same handlers and the same timing fields** — only the first word changes. Learn this once and you never look up a probe again:

```yaml
        startupProbe:            # or readinessProbe / livenessProbe
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 15
```

Forgot a field name mid-exam? Recall it in the terminal instead of opening docs:

```bash
k explain pod.spec.containers.startupProbe
k explain pod.spec.containers.startupProbe --recursive | head -20
```

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
    readinessProbe:            # or livenessProbe / startupProbe — identical shape
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

Handler choice from the wording: *"HTTP GET on / port 80"* → `httpGet`;
*"reachable on port 80"* with no path → `tcpSocket`; *"the file /tmp/ready
exists"* / *"the command succeeds"* → `exec`.

## Adding a probe to something that already exists

Most probe questions hand you a running Deployment. There is no `k set probe`,
so it's `k edit` (fastest) or export→edit→apply (only if the task wants a file):

```bash
k -n ceres edit deploy slow-starter    # add the block under the container
k -n ceres get deploy slow-starter -o jsonpath='{.spec.template.spec.containers[0].startupProbe}{"\n"}'
```

Indentation is the whole game in vim: the probe key is a sibling of `image:`
and `name:` inside `containers[]`. Land the cursor on the `image:` line, `yy`
to copy it as an indent reference, then `o` and type — or `:set paste` first
if you're pasting from the docs, otherwise vim auto-indents each line further
right and the YAML breaks.

## Pod conditions (readiness gates endpoints)

```bash
k get pod app                 # READY column = ready containers / total
k describe pod app            # Conditions: PodScheduled / Initialized / ContainersReady / Ready
```

## See also

- `04-observability/01-readiness-probes.md`, `02-liveness-probes.md`
- `services.md` — a Service only routes to pods whose readiness probe passes
