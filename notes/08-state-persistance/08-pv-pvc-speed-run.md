# PV → PVC → Pod: the speed run

`04-persistent-volumes-and-claims.md` covers *what* these objects are. This is how to **build the chain from an empty terminal in under two minutes** without opening the docs.

## The problem this solves

PersistentVolume, PersistentVolumeClaim, and pod volume wiring are the highest-frequency exam pattern that has **no imperative generator**. There is no `k create pv`, no `k create pvc`, no `k set volume`. `$do` scaffolding — the strategy that carries most other questions — does not apply.

That leaves two options mid-question: search kubernetes.io (60-90s, and the search results bury the one page you want under CSI/StorageClass/snapshot docs), or type it from memory (~90s including verification). Memory wins, and the thing to memorize is small: **one 10-line spec, mirrored twice.**

> This is the deliberate exception to *prefer discovery over memorization*. `kubectl explain` is the right tool for the long tail of fields you meet once. It is the wrong tool for the four or five shapes you will type on every practice test — PV, PVC, NetworkPolicy, and pod-level `volumes`. Those get memorized; everything else gets discovered.

---

## The skeleton

Three documents, one file. This is the whole thing:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteMany"]
  hostPath:
    path: /opt/volume/nginx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-claim
spec:
  storageClassName: manual
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 200Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: logger
spec:
  containers:
  - name: logger
    image: nginx:alpine
    volumeMounts:
    - name: log
      mountPath: /var/www/nginx
  volumes:
  - name: log
    persistentVolumeClaim:
      claimName: log-claim
```

**`accessModes: ["ReadWriteMany"]` in flow style** is valid YAML and saves a line plus an indentation decision every time. Same for any short single-item list.

---

## Why you only have to memorize one spec

Every manifest is `apiVersion` / `kind` / `metadata.name` / `spec`. Only the spec differs, and the three specs are not three separate things to learn:

| | PV — what storage **has** | PVC — what the pod **wants** |
|---|---|---|
| class | `storageClassName: manual` | `storageClassName: manual` — *identical* |
| modes | `accessModes: [...]` | `accessModes: [...]` — *identical* |
| size | `capacity.storage` | `resources.requests.storage` |
| backend | `hostPath` / CSI / NFS | *(none — the PV owns the backend)* |

**A PVC is a PV with the backend deleted and `capacity` renamed to `resources.requests`.** So don't type it twice — type the PV, duplicate the whole buffer, and edit four lines:

```vim
:%y        " yank the entire file (the PV is all there is so far)
G p        " jump to last line, paste the copy below it
```

Then in the copy: change `kind`, change `name`, delete the `hostPath` block (`2dd`), swap `capacity:` → `resources:` + `requests:`, and add the `---` separator.

Avoid `V}y` here — `}` jumps to the next blank line, and well-formed YAML rarely has one, so it silently selects to the end of the file.

The two identical rows are not a coincidence — **they are the binding contract**. Class and access mode must match for the PVC to bind at all. If you find yourself typing a different value on either row, stop: you are about to create a PVC that hangs `Pending`.

The pod half is simpler: **two mirrored lists joined by a name you invent.**

```
containers[].volumeMounts[].name  ─── "log" ───  spec.volumes[].name
                  mountPath                       persistentVolumeClaim.claimName
                (path in container)                    (the PVC object)
```

That name (`log`) is pod-local and arbitrary. It is not the PV name, not the PVC name, and nothing outside the pod ever sees it.

---

## The fourth object: when you have to create the StorageClass

The skeleton above assumes `manual` already exists — questions often hand it to you ("already created for you, do not create it again"). When they don't, it's five lines:

```yaml
apiVersion: storage.k8s.io/v1              # NOT v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
```

Two things about this object break the pattern every other manifest follows:

- **`apiVersion: storage.k8s.io/v1`.** PV, PVC and Pod are all core `v1`. StorageClass is the one object in the chain that isn't, and `kind: StorageClass` under `apiVersion: v1` fails with "no matches for kind".
- **There is no `spec:`.** `provisioner`, `parameters`, `reclaimPolicy`, `volumeBindingMode` and `allowVolumeExpansion` are all top-level keys, siblings of `metadata`. Indenting them under a `spec:` out of habit is the single most common StorageClass error — the API rejects it as an unknown field. `provisioner` is the only required one.

Always check before writing it:

```bash
k get sc                    # does it exist? is one marked (default)?
```

### Static vs dynamic — which shape the question wants

`kubernetes.io/no-provisioner` provisions nothing. A class using it exists purely as a **matching label** so a hand-written PV and a PVC find each other; you still write the PV yourself. That's the pictured question, and it's why the class name has to appear identically on both objects.

A class with a *real* provisioner (`rancher.io/local-path` on kind, a CSI driver on a cloud cluster) creates the PV for you. You write the SC and the PVC and **never write a PV at all**.

| Question says | You write | `k get pv` shows |
|---|---|---|
| "create a PersistentVolume named X" | SC (maybe) + PV + PVC + Pod | the PV you wrote |
| names a class but no PV | SC (maybe) + PVC + Pod | a PV that appeared on its own |
| names neither | PVC + Pod, omit `storageClassName` | a PV from the default class |

The tell is simply whether a PV is named in the task. If one is, the class is a label; if not, expect dynamic provisioning.

> With `no-provisioner`, adding `volumeBindingMode: WaitForFirstConsumer` leaves the PVC `Pending` until a pod mounts it. That is correct behavior, not a broken binding — don't spend time debugging it.

---

## The run

Apply storage first and confirm it binds **before** writing the pod. A wrong `storageClassName` shows up instantly here; if you apply all three at once you debug a `Pending` pod instead of a `Pending` claim.

```bash
k get sc                       # exists already? default? -> decides if you write one

vim pv.yaml                    # SC (if needed) + PV + PVC, --- between them
k apply -f pv.yaml
k get pv,pvc                   # both must say Bound before continuing

k run logger --image=nginx:alpine $do > pod.yaml
vim pod.yaml                   # add volumeMounts + volumes
k apply -f pod.yaml

k get pv,pvc,pod               # Bound / Bound / Running
```

`k run ... $do` is still worth it for the pod: it hands you `apiVersion`, `kind`, `metadata`, `spec.containers[].name` and `image` for free. Ignore the `resources: {}` and `status: {}` noise it leaves behind — both are valid and cost nothing.

In vim, the generated pod ends at `image:`, so the edit is: `G`, then `o` and type the two blocks. `volumeMounts` indents under the container (a sibling of `image`); `volumes` indents under `spec` (a sibling of `containers`). Getting those two indent levels right is the only hard part of the pod file.

If the pod won't start:

```bash
k describe pod logger | tail -20      # Events: unbound claim, wrong claimName
k describe pvc log-claim              # Events: no matching PV and why
```

---

## Traps that cost the question

- **StorageClass written as core `v1`, or with a `spec:` block.** It is `storage.k8s.io/v1`, and `provisioner` sits at the top level next to `metadata`. Both mistakes fail at apply time, so at least they're loud.
- **PVC missing `storageClassName`.** The single most common failure. "Binds to the PV created above" means *copy the class and the access mode*. Worse than a stuck claim: on a cluster with a default StorageClass, an omitted class silently provisions a **brand-new** PV, the PVC reports `Bound`, everything looks correct, and you bound to the wrong volume.
- **Access mode narrowed.** A PVC asking `ReadWriteOnce` does **not** bind to a PV that offers only `ReadWriteMany`. The mode must be one the PV actually lists — matching, not "less than".
- **`capacity` vs `resources.requests`.** The one asymmetric field. PV declares capacity; PVC requests it. `capacity` on a PVC is not a valid field.
- **Requesting less than the PV holds is fine.** 200Mi against a 1Gi PV binds and consumes the entire PV; the spare 800Mi is wasted, not shared. Do not "fix" this by resizing the PV to match.
- **Two paths, two meanings.** `hostPath.path` is on the node. `mountPath` is inside the container. They are unrelated and are frequently near-identical strings in the question text.
- **Mount name mismatch.** `volumeMounts[].name` must equal `volumes[].name` or the pod fails validation.
- **Namespace.** PV is cluster-scoped; PVC and Pod are namespaced. If the question names a namespace, the PVC and Pod need it and the PV does not.

---

## Self-drill

Treat this like a code kata — the goal is a clean run with no reference material, not understanding, which you already have.

1. `k delete pv,pvc,pod --all` to reset.
2. Start a timer. Close every doc tab.
3. Build the chain from the prompt below. Verify with `k get pv,pvc,pod`.
4. Reset and repeat until you are consistently under two minutes.

Rotate prompts so you are reciting the shape, not a specific file:

| # | StorageClass | PV | PVC | Pod |
|---|---|---|---|---|
| 1 | *(given)* `manual` | `log-volume`, class `manual`, RWX, 1Gi, hostPath `/opt/volume/nginx` | `log-claim`, 200Mi | `logger`, `nginx:alpine`, mount at `/var/www/nginx` |
| 2 | *(given)* `slow` | `data-pv`, class `slow`, RWO, 500Mi, hostPath `/mnt/data` | `data-pvc`, 100Mi | `app`, `busybox`, `sleep 3600`, mount at `/data` |
| 3 | *(none)* | `cache-pv`, no class, ROX, 2Gi, hostPath `/mnt/cache` | `cache-pvc`, 1Gi | `web`, `nginx`, mount read-only at `/usr/share/nginx/html` |
| 4 | *(default)* | *(none — let it be provisioned)* | `dyn-pvc`, RWO, 1Gi | `dyn-pod`, `nginx`, mount at `/app` |
| 5 | **create** `local-sc`, `no-provisioner`, `WaitForFirstConsumer` | `local-pv`, class `local-sc`, RWO, 5Gi, hostPath `/mnt/disks/ssd1` | `local-pvc`, 2Gi | `db`, `postgres:alpine`, mount at `/var/lib/postgresql/data` |

Each prompt drills a different failure mode:

- **3** forces `readOnly: true` on the mount and `storageClassName: ""` on both objects — the empty string means "static PVs with no class only", while omitting the field entirely would invoke the default StorageClass instead. Two different things that look the same.
- **4** is the inverse: no PV at all, omit the class, let dynamic provisioning create it. Check `k get pv` afterwards to see the volume you never wrote.
- **5** adds the fourth object, and its `WaitForFirstConsumer` means the PVC sits `Pending` until the pod exists. Resist "fixing" it — apply the pod and watch it bind. This one also catches the `storage.k8s.io/v1` and no-`spec:` traps.

When two minutes is comfortable, drill the same way against `volumeClaimTemplates` in `06-statefulsets.md` — the per-pod PVC is the same spec nested one level deeper.

## References

- [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/) — the one page with all three manifests together; the right bookmark if you do have to look
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — access modes, class matching, and the binding rules behind the traps above
- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) — `persistentVolumeClaim` and `hostPath` volume sources
