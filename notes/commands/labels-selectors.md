# Labels, Selectors & Annotations

Labels = identifying key/values you select on. Annotations = non-identifying metadata (never selectable). Selectors are how Services, Deployments, and `kubectl` target sets of objects.

## Query by label

```bash
k get pods -l app=App1                       # equality (short form; --selector is long form)
k get pods -l app=App1,function=Front-end    # comma = AND
k get pods -l app!=App1                       # not-equal
k get pods -l 'env in (dev,staging)'          # set-based: in
k get pods -l 'env notin (prod)'              # set-based: notin
k get pods -l '!tier'                         # objects WITHOUT a tier label
k get pods --show-labels                      # show each object's labels
k get pods -L app -L function                 # show named labels as COLUMNS (-L, not -l)
```

Equality selectors: `key=value`, `key!=value`. Set-based: `in`, `notin`, `exists` (`key`), `!key`.

## Mutate labels / annotations

```bash
k label pod <pod> tier=frontend               # add a label
k label pod <pod> tier=backend --overwrite    # change an existing label
k label pod <pod> tier-                       # remove a label (trailing dash)
k annotate pod <pod> description="payments frontend"
k annotate pod <pod> description-             # remove an annotation
```

Also works on nodes (`k label nodes ...`) — see `scheduling.md`.

## Where selectors show up

- Service `spec.selector` → which pods get traffic.
- Deployment/ReplicaSet `spec.selector.matchLabels` → which pods it manages (immutable after create).
- Blue-green cutover = editing a Service selector; canary = a shared label across two deployments (see `deployment-strategies.md`).

## See also

- `05-pod-design/01-labels-selectors-annotations.md`
- `scheduling.md` — node labels + selectors · `deployment-strategies.md` — selectors for traffic control
