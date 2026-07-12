# Ingress

L7 HTTP routing to Services. Unlike NetworkPolicy, `kubectl create ingress` **does** exist — use it.

## Imperative create

```bash
# Each --rule is host/path=service:port. Trailing * => pathType: Prefix, else Exact.
k create ingress ingress-wear-watch \
  --rule="my-online-store.com/wear*=wear-service:80" \
  --rule="my-online-store.com/watch*=watch-service:80"

# Scaffold to a file (omit the host for a no-host rule)
k create ingress web --rule="/wear*=wear-service:80" $do > ing.yaml
```

Extra flags:

```bash
--class=nginx                          # spec.ingressClassName
--default-backend=wear-service:80      # spec.defaultBackend
--annotation nginx.ingress.kubernetes.io/rewrite-target=/   # add an annotation
```

## Inspect

```bash
k get ingress                          # short: ing
k get ing -A
k describe ingress ingress-wear-watch  # the useful one: host/path -> backend table + endpoint errors
```

## Structure (networking.k8s.io/v1 traps)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  ingressClassName: nginx
  rules:
    - host: my-online-store.com
      http:
        paths:
          - path: /wear
            pathType: Prefix          # required field in v1
            backend:
              service:                # v1 nests name/port under service:
                name: wear-service
                port:
                  number: 80
```

v1 gotchas: `pathType` is required; backend is `service.name` + `service.port.number` (not the old flat `serviceName`/`servicePort`).

## See also

- `07-services-and-networking/03-ingress.md`
- `services.md` — the Services an Ingress routes to
