# Persistent Volumes & Persistent Volume Claims

Core, heavily examinable. You'll write PVs and PVCs, bind them, consume a PVC from a pod, and reason about access modes and reclaim policy. The **1:1 binding** rule and the **`persistentVolumeClaim.claimName`** wiring are common task pieces. Builds on `03-kubernetes-volumes.md` (volumes/volumeMounts, the multi-node `hostPath` problem this solves).

---

## 1. Why PV/PVC exist

In the last chapter the volume was defined **inside each pod**. That doesn't scale: on a big cluster with many users, every user would have to know and configure storage details in every pod — lots of duplicated, error-prone config. Better to **manage storage centrally**: an **administrator** provisions a pool of storage, and **users carve out pieces as they need them**.

Kubernetes splits this into **two separate cluster objects**:

- **PersistentVolume (PV)** — a cluster-wide piece of storage, provisioned by the **admin**.
- **PersistentVolumeClaim (PVC)** — a **user's request** for storage. Kubernetes **binds** the claim to a matching PV.

![PV + PVC: admin provisions, developer claims and consumes](./diagrams/08-pv-pvc-pod-chain.png)

This is a clean separation of concerns: the admin worries about *what storage exists and how it's backed*; the developer just says *"I need 500Mi, read-write"* and consumes the result. The chain is **Pod → PVC → PV**.

---

## 2. PersistentVolume (PV)

Admin-created. Key fields: `accessModes`, `capacity.storage`, and a backend (the lecture shows `awsElasticBlockStore`; for labs use `hostPath`).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:                 # labs; in prod this is normally a CSI/StorageClass-provisioned backend
    path: /data
```

```bash
kubectl create -f pv-definition.yaml
kubectl get pv
# NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
# pv-vol1   1Gi        RWO            Retain           Available                          3m
```

**Access modes** (how the volume may be mounted):

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | mounted read-write by a **single node** (not a single *pod* — see gotcha) |
| `ReadOnlyMany` | ROX | mounted read-only by many nodes |
| `ReadWriteMany` | RWX | mounted read-write by many nodes (needs a backend that supports it, e.g. NFS) |
| `ReadWriteOncePod` | RWOP | mounted read-write by a **single pod** (beyond the lecture; GA in 1.29) |

> **Correction (in-tree plugin):** the slide's inline `awsElasticBlockStore:` is a deprecated in-tree plugin (same note as `03`). On a current cluster, PVs for cloud storage are normally **dynamically provisioned by a StorageClass via CSI**, not hand-written with inline cloud blocks. For practice on kind, use `hostPath`.

**PV status lifecycle:** `Available` → `Bound` (claimed) → `Released` (PVC deleted, but data may remain) → `Failed`.

---

## 3. PersistentVolumeClaim (PVC)

User-created. It's a *request*: how much storage, which access mode.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl create -f pvc-definition.yaml
kubectl get pvc
# NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES
# myclaim   Pending                                        <- then Bound once a PV matches
```

Note the asymmetry: a **PV** declares `capacity` (what it *has*); a **PVC** declares `resources.requests` (what it *wants*).

---

## 4. Binding — and resolving your confusion

When you create a PVC, Kubernetes looks for a PV that satisfies **all** of these criteria, then binds them:

![Binding is 1:1, with criteria, wasted capacity, and the carve-up resolution](./diagrams/09-pv-pvc-binding.png)

- **Sufficient capacity** — PV `capacity` ≥ PVC `request`
- **Access modes** — compatible (the PV offers a mode the PVC asks for)
- **Volume mode** — `Filesystem` (default) vs `Block` must match
- **Storage class** — `storageClassName` must be equal
- **Selector (optional)** — if you set `spec.selector.matchLabels` on the PVC, only PVs with matching labels qualify (used to pick a *specific* PV when several would otherwise match)

```yaml
# PV
metadata:
  labels:
    name: my-pv
---
# PVC
spec:
  selector:
    matchLabels:
      name: my-pv
```

### 1:1 binding vs "carving up a pool"

A large PV isn't split among users — the catch is **what gets carved**:

- **Binding is strictly 1 PVC : 1 PV.** Once bound, that PV belongs to that one claim, exclusively.
- If a **500Mi** claim is matched to a **1Gi** PV (because that's the smallest available that satisfies it), the claim binds to the **whole** PV. The extra **500Mi is not handed to anyone else** — it's locked to that bound PV and effectively wasted. A single PV is **never split** across multiple claims.
- So **"carving up a large pool" is not "splitting one PV."** The pool is either **many PV objects** (the admin pre-creates a bunch of differently-sized PVs, each claimed 1:1), or — far more common today — a **StorageClass** that **dynamically provisions a right-sized PV per claim** from the backing storage system. *That* dynamic, per-claim provisioning is the real "carving," and it's the next lecture. The 1:1 rule is about **object-to-object binding**, while "carving the pool" is about the **backend** being subdivided into many PV objects.

**If no PV satisfies the claim, the PVC stays `Pending`** until a matching PV appears (or a StorageClass provisions one).

---

## 5. Using a PVC in a pod

Once the PVC is bound, a pod consumes it by naming the claim under `volumes` (then mounting it like any volume):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myfrontend
    image: nginx
    volumeMounts:
    - mountPath: /var/www/html
      name: mypd
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim          # <- the link to the PVC
```

Same pattern for Deployments/ReplicaSets — put it in the **pod template** (`spec.template.spec`). Notice this is the exact `volumes`/`volumeMounts` wiring from `03`; the only change is the volume *source* is now `persistentVolumeClaim` instead of `hostPath`/`emptyDir`.

---

## 6. Reclaim policy — what happens when the PVC is deleted

`kubectl delete pvc myclaim` removes the claim. What happens to the **PV** (and its data) is governed by the PV's **`persistentVolumeReclaimPolicy`**:

| Policy | What happens to the PV |
|---|---|
| **Retain** (default for static PVs) | PV and its data are **kept**; PV moves to `Released`. It is **not** auto-reused — an admin must manually clean up and recreate/reclaim it. |
| **Delete** | the PV (and usually the underlying storage asset) is **deleted**. Common for dynamically provisioned volumes. |
| **Recycle** (**deprecated**) | data was scrubbed (`rm -rf /scrub/*`) and the PV made `Available` again. |

> **Why Recycle is deprecated:** a naive `rm -rf` is **not a real reclaim**. It doesn't guarantee the space is truly free and safe to reuse — leftover **filesystem metadata, snapshots, and inode entries** can remain, you can hit **permission issues**, and there's no secure-wipe guarantee (a real concern for multi-tenant clusters where the next claimant must not see prior data). Properly reclaiming storage needs far more than file deletion, so Kubernetes dropped Recycle in favor of **dynamic provisioning + `Delete`** (let the storage system handle teardown) or admin-managed **`Retain`**.

> **Storage-object-in-use protection (exam-adjacent gotcha):** if you try to delete a PVC that's **still mounted by a running pod**, the PVC won't disappear — it stays `Terminating` (a finalizer holds it) until the pod stops using it. Same protection keeps a bound PV from vanishing out from under a PVC. So "my PVC won't delete" usually means a pod is still using it.

---

## Imperative / command reference

No clean generator — write the YAML (small specs, fast to type):

```bash
kubectl get pv ; kubectl get pvc
kubectl describe pv pv-vol1          # shows Status, Claim, Reclaim Policy, events
kubectl describe pvc myclaim         # shows Bound volume, capacity, events (why Pending?)
kubectl explain pv.spec              # discover fields: accessModes, capacity, reclaim policy...
kubectl explain pvc.spec
kubectl get pvc myclaim -o jsonpath='{.spec.volumeName}'   # which PV it bound to
kubectl delete pvc myclaim           # may hang (Terminating) if a pod still uses it
```

---

## Exam-pattern gotchas

- **PV and PVC are separate objects; binding is 1:1.** An oversized PV is claimed whole — spare capacity is wasted, never re-shared.
- **`RWO` = one *node*, not one *pod*.** Multiple pods on the same node can share an RWO volume. Use **`ReadWriteOncePod`** for true single-pod.
- **`storageClassName` must match** to bind. Empty `""` binds only to PVs with no class (pure static); **omitting** it lets the **default StorageClass** provision dynamically — different behavior, easy to trip on.
- **PVC `Pending`** = no PV matches (capacity/access/class/selector) **or** no provisioner for dynamic. `describe pvc` tells you which.
- **Consume via** `volumes.persistentVolumeClaim.claimName`; for Deployments it goes in the **pod template**.
- **`Retain`** leaves the PV `Released` and **not reusable** until an admin intervenes. **`Recycle`** is deprecated.
- **Capacity isn't always enforced** as a hard quota by the backend (e.g. `hostPath` won't stop you exceeding 1Gi) — it's primarily a *matching* attribute. (Beyond lecture; true enforcement depends on the storage type.)
- Deleting a PVC **in use by a pod** blocks until the pod releases it (finalizer).
