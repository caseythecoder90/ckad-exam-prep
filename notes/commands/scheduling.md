# Scheduling — Taints/Tolerations, Node Selectors, Node Affinity

How you steer pods onto (or away from) specific nodes. None of these have an imperative generator on the pod side — you `$do > pod.yaml` and hand-edit the `spec`. The **node** side (taint/label) is all imperative.

- **Taint** = node repels pods (opt-out). **Toleration** = pod is allowed onto a tainted node.
- **nodeSelector / nodeAffinity** = pod is attracted to a labeled node (opt-in).
- Taint alone doesn't pin a pod to a node; affinity alone doesn't keep others off. For a **dedicated node**, use both.

## Taints (node side)

```bash
# Add a taint: key=value:effect
k taint nodes <node> app=blue:NoSchedule
k taint nodes node1 hardware=gpu:NoSchedule

# Effects: NoSchedule (block new) | PreferNoSchedule (soft) | NoExecute (block new + evict non-tolerating)

# Remove a taint — trailing minus
k taint nodes <node> app=blue:NoSchedule-

# Inspect
k describe node <node> | grep -A1 Taints
k get node <node> -o jsonpath='{.spec.taints}'
```

Common auto-applied taints:
- `node-role.kubernetes.io/control-plane:NoSchedule` — why pods avoid the control plane (older key: `node-role.kubernetes.io/master`).
- `node.kubernetes.io/unreachable:NoExecute` — added on node failure.

Toleration block (pod side — no flag, edit YAML):

```yaml
spec:
  tolerations:
    - key: "app"
      operator: "Equal"     # Equal (default, needs value) | Exists (omit value)
      value: "blue"
      effect: "NoSchedule"
```

## Node labels + nodeSelector

```bash
# Label a node
k label nodes <node> size=Large
k label nodes <node> size=Medium --overwrite     # change existing value
k label nodes <node> size-                        # remove label (trailing minus)

# Inspect / query
k get nodes --show-labels
k get node <node> -o jsonpath='{.metadata.labels}'
k get nodes -l size=Large                          # equality selector
k get nodes -l 'size in (Large,Medium)'            # set-based selector
```

nodeSelector (pod side — simple exact-match map, implicit AND, no OR/NOT):

```yaml
spec:
  nodeSelector:
    size: Large
```

Built-in node labels you can select on: `kubernetes.io/hostname`, `kubernetes.io/arch`, `kubernetes.io/os`, `node.kubernetes.io/instance-type`, `topology.kubernetes.io/zone`.

## Node affinity

Same node labels; more expressive than nodeSelector (operators, required vs preferred). No imperative form — edit YAML, then confirm placement:

```bash
k label nodes <node> size=Large                    # label first
k get pod <name> -o wide                            # confirm which NODE it landed on
k describe pod <name>                               # FailedScheduling event = no node matched
```

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard: Pending if no match
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In            # In | NotIn | Exists | DoesNotExist | Gt | Lt
                values: ["Large"]       # omit values for Exists/DoesNotExist
      # preferredDuringSchedulingIgnoredDuringExecution: soft, takes weight 1-100 + preference
```

List nesting: multiple `nodeSelectorTerms` = **OR**; multiple `matchExpressions` in one term = **AND**.

## Dedicated node (taint + affinity together)

Use the **same** `key=value` as both a taint (keeps others off) and a label (draws the target pod in):

```bash
k label nodes node1 color=blue          # affinity matches this
k taint nodes node1 color=blue:NoSchedule   # repels pods that don't tolerate it
```

The pod then carries **both** a matching `tolerations` entry and a `nodeAffinity` rule (`operator: In`). Taint = "keep others out"; affinity = "put me here."

## Diagnosis

```bash
k get pod <name>                          # Pending?
k describe pod <name> | grep -A10 Events  # "didn't match node affinity/selector" / "had untolerated taint"
k get pod <name> -o jsonpath='{.spec.tolerations}'
k describe pod <name> | grep -A10 Tolerations
```

## See also

- `02-configuration/10-taints-and-tolerations.md`, `11-node-selectors.md`, `12-node-affinity.md`, `13-affinity-vs-taints-tolerations.md`
- `labels-selectors.md` — the label/selector syntax these all build on
