# Deployment Strategies — Blue/Green & Canary

Rolling update, rollout, and rollback commands live in `deployments.md`. This file covers the two multi-deployment patterns. Both are driven by **labels + a Service selector**, not by a special resource.

## Blue/Green — two deployments, flip the Service selector

Blue (v1) serves traffic; green (v2) runs alongside with no traffic until you cut over.

```bash
k apply -f myapp-blue.yml                      # v1, label version=v1
k apply -f myapp-green.yml                     # v2, label version=v2 (no traffic yet)

k port-forward deployment/myapp-green 8080:8080   # validate green privately

# Cutover: flip the Service selector v1 -> v2 (all traffic to green)
k patch service my-service -p '{"spec":{"selector":{"version":"v2"}}}'
# or edit the selector in the Service YAML and: k apply -f service-definition.yaml

k describe service my-service                  # confirm Selector=version=v2
k get endpoints my-service                     # confirm backing pod IPs switched to green

# Rollback: flip back to blue
k patch service my-service -p '{"spec":{"selector":{"version":"v1"}}}'
k delete -f myapp-blue.yml                      # retire blue once confident
```

## Canary — shared label, split by pod count

Both versions carry the **same** Service-selected label (e.g. `app=front-end`), so one Service load-balances across both. The traffic split ≈ the pod-count ratio.

```bash
k apply -f myapp-primary.yml                    # primary: replicas 5, v1
k apply -f myapp-canary.yml                     # canary:  replicas 1, v2  (~5:1 split)
k apply -f service-definition.yaml              # selects the shared label app=front-end

k get endpoints my-service                      # backed by pods from BOTH deployments (e.g. 6 IPs)
k get pods -l app=front-end --show-labels       # see v1 + v2 mixed under the shared label

k scale deployment myapp-canary --replicas=2    # widen the split (~5:2)

# Promote: roll primary to v2, then drop the canary
k set image deployment/myapp-primary app-container=myapp-image:2.0
k rollout status deployment/myapp-primary
k delete deployment myapp-canary
```

## Key idea

- **Blue/green** = one deployment live at a time; switch is atomic (selector flip). Instant rollback.
- **Canary** = both live at once; ratio controlled by replica counts. Native Service load-balancing only gives coarse, count-based splitting (finer % splitting needs an ingress/mesh).

## See also

- `05-pod-design/03-blue-green-deployment.md`, `04-canary-deployment.md`
- `deployments.md` — rollout status/history/undo · `labels-selectors.md` · `services.md`
