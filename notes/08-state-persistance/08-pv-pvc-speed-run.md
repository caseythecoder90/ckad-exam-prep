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

## The run

Apply storage first and confirm it binds **before** writing the pod. A wrong `storageClassName` shows up instantly here; if you apply all three at once you debug a `Pending` pod instead of a `Pending` claim.

```bash
vim pv.yaml                    # PV + PVC, --- between them
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

| # | PV | PVC | Pod |
|---|---|---|---|
| 1 | `log-volume`, class `manual`, RWX, 1Gi, hostPath `/opt/volume/nginx` | `log-claim`, 200Mi | `logger`, `nginx:alpine`, mount at `/var/www/nginx` |
| 2 | `data-pv`, class `slow`, RWO, 500Mi, hostPath `/mnt/data` | `data-pvc`, 100Mi | `app`, `busybox`, `sleep 3600`, mount at `/data` |
| 3 | `cache-pv`, no class, ROX, 2Gi, hostPath `/mnt/cache` | `cache-pvc`, 1Gi | `web`, `nginx`, mount read-only at `/usr/share/nginx/html` |
| 4 | *(none — use the default StorageClass)* | `dyn-pvc`, RWO, 1Gi | `dyn-pod`, `nginx`, mount at `/app` |

Prompt 3 forces `readOnly: true` on the mount and `storageClassName: ""` on both objects (empty string means "static PVs with no class only" — omitting the field entirely would invoke the default StorageClass instead). Prompt 4 forces the opposite: no PV at all, omit the class, let dynamic provisioning create the PV for you.

When two minutes is comfortable, drill the same way against `volumeClaimTemplates` in `06-statefulsets.md` — the per-pod PVC is the same spec nested one level deeper.

## References

- [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/) — the one page with all three manifests together; the right bookmark if you do have to look
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — access modes, class matching, and the binding rules behind the traps above
- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) — `persistentVolumeClaim` and `hostPath` volume sources
