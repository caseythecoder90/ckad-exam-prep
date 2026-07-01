# Network Policies

The only core-Kubernetes firewall. **No imperative generator** — `kubectl create networkpolicy` doesn't exist; hand-write the YAML. A pod selected by any policy becomes default-deny for the covered direction; rules are an allow-list.

## Inspect

```bash
k get netpol                          # short name; policies in current namespace
k get networkpolicy -A                # all namespaces
k describe netpol db-policy           # parsed Ingress/Egress rules, peers, ports
k apply -f db-policy.yaml
k delete netpol db-policy
```

## Find the labels you'll select on (do this FIRST)

```bash
k get pods --show-labels                       # for podSelector
k get ns --show-labels                          # for namespaceSelector
k get ns prod -o jsonpath='{.metadata.labels}'  # e.g. kubernetes.io/metadata.name=prod
```

## Test connectivity

```bash
k exec -it <client-pod> -- nc -zv <db-svc> 3306        # allowed = connects; denied = hangs/times out
k run tester --image=busybox -it --rm --restart=Never -- sh -c 'nc -zvw3 db-service 3306'
```

Note: `k port-forward` bypasses NetworkPolicy (it tunnels at the node) — don't use it to test a policy.

## Policy shape (the AND/OR trap)

```yaml
spec:
  podSelector:                 # which pods this policy applies to ({} = all in namespace)
    matchLabels: { app: db }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:   # SAME list item as podSelector below = AND
            matchLabels: { kubernetes.io/metadata.name: prod }
          podSelector:
            matchLabels: { app: api }
        - ipBlock:             # SEPARATE list item (leading dash) = OR
            cidr: 10.0.0.0/16
      ports:
        - protocol: TCP
          port: 3306
```

The dash is load-bearing: two peers under one `-` = **AND** (must match both); two separate `-` entries = **OR**.

## See also

- `07-services-and-networking/01-network-policies.md`, `02-network-policy-selectors-and-rules.md`
- `labels-selectors.md` — the selector syntax
