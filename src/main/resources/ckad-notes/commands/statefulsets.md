# StatefulSets & Headless Services

StatefulSets give pods stable ordinal names (`mysql-0`, `mysql-1`), ordered startup/teardown, and per-pod PVCs. A **headless Service** (`clusterIP: None`) gives each pod a stable DNS name. No full imperative generator — declarative YAML.

## Manage

```bash
k apply -f statefulset.yaml
k get sts                              # short for statefulset
k describe sts mysql                   # replicas, update strategy, events
k get pods -l app=mysql -w             # watch ordered create/delete
k scale sts mysql --replicas=5         # adds mysql-3, mysql-4 sequentially
k scale sts mysql --replicas=2         # removes HIGHEST ordinals first (4, then 3, then 2)
```

## Rollout & partitioned canary

Updates roll in **reverse ordinal** order. `partition: N` updates only pods with ordinal ≥ N.

```bash
k set image sts/mysql mysql=mysql:8.1                                       # triggers rolling update
k patch sts mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'  # only >=2 update
k exec mysql-2 -- mysql --version                                          # validate the canary
k patch sts mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'  # roll to all

k rollout status sts/mysql
k rollout history sts/mysql
k rollout undo sts/mysql
k rollout restart sts/mysql
```

## Delete (PVCs are NOT auto-removed)

```bash
k delete pod mysql-1                    # recreated with same name/ordinal/PVC
k delete sts mysql --cascade=orphan     # delete controller, leave pods running
k delete sts mysql                      # delete controller + pods
k delete pvc -l app=mysql               # per-pod PVCs must be deleted manually
k delete pod mysql-1 --force --grace-period=0   # stuck Terminating on a dead node (split-brain risk)
```

## Headless Service + per-pod DNS

`clusterIP: None` → DNS returns pod IPs directly instead of one VIP. The StatefulSet's `serviceName` must point at it.

```bash
k apply -f headless-service.yaml        # create before/with the StatefulSet
k get svc mysql-h                       # CLUSTER-IP shows None
k get endpoints mysql-h                 # registered pod IPs
k describe endpoints mysql-h

# Per-pod DNS: <pod>.<svc>.<ns>.svc.cluster.local
k exec -it mysql-0 -- nslookup mysql-0.mysql-h.default.svc.cluster.local   # one pod's A record
k exec -it mysql-0 -- nslookup mysql-h.default.svc.cluster.local           # all pod IPs
k exec -it mysql-0 -- hostname          # mysql-0
k exec -it mysql-0 -- hostname -f       # mysql-0.mysql-h.default.svc.cluster.local
k exec -it mysql-0 -- cat /etc/resolv.conf   # search domains + CoreDNS
```

## See also

- `08-state-persistance/06-statefulsets.md`, `07-headless-services.md`
- `storage.md` — PV/PVC concepts · `services.md` — normal (VIP) Services
