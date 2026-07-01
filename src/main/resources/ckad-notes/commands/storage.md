# Storage — Volumes, PV/PVC, StorageClasses

Pod volumes are declared in `spec.volumes` and mounted via `containers[*].volumeMounts`. Persistent storage decouples into PersistentVolume (the disk) + PersistentVolumeClaim (the request), bound 1:1. StorageClasses provision PVs dynamically. None have a clean imperative generator — scaffold a pod and hand-edit, or `k create -f`.

## Pod volumes

```bash
k explain pod.spec.volumes                        # discover volume types
k explain pod.spec.containers.volumeMounts        # mountPath, name, readOnly, subPath
k describe pod <name>                             # Mounts: section shows what's mounted where
```

```yaml
spec:
  containers:
    - name: app
      image: alpine
      volumeMounts:
        - name: data
          mountPath: /opt/data
  volumes:
    - name: data
      emptyDir: {}                # scratch, dies with pod
    # - hostPath: { path: /data } # node path — NOT portable across nodes (multi-node trap)
    # - persistentVolumeClaim: { claimName: myclaim }
```

## PersistentVolumes & Claims

```bash
k create -f pv.yaml
k create -f pvc.yaml
k get pv                                  # capacity, access modes, reclaim policy, status, claim, SC
k get pvc                                 # status Pending -> Bound
k describe pv pv-vol1
k describe pvc myclaim                    # why it's still Pending
k explain pv.spec                         # accessModes, capacity, reclaim policy
k explain pvc.spec
k get pvc myclaim -o jsonpath='{.spec.volumeName}'   # which PV it bound to
k delete pvc myclaim                      # may hang Terminating if a pod still mounts it
```

Binding needs the PVC's `accessModes` + requested size to fit the PV, and matching `storageClassName`.

## StorageClasses (dynamic provisioning)

No `kubectl create storageclass` — always declarative.

```bash
k get sc                                   # which one is (default)?
k describe sc standard                     # provisioner, volumeBindingMode, params, reclaim policy
k apply -f pvc.yaml                        # references storageClassName
k get pvc -w                               # watch Pending -> Bound
k apply -f pod.yaml                        # with WaitForFirstConsumer, the pod triggers binding
k get pv                                   # see the auto-created PV
```

Gotchas: `volumeBindingMode: WaitForFirstConsumer` keeps a PVC `Pending` until a pod consumes it — that's normal, not an error. A typo in `storageClassName` → PVC stuck Pending with "no StorageClass" in `describe`.

## See also

- `08-state-persistance/03-kubernetes-volumes.md`, `04-persistent-volumes-and-claims.md`, `05-storage-classes.md`
- `statefulsets.md` — per-pod PVCs via volumeClaimTemplates
