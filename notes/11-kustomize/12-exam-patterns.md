---
section: 11-kustomize
chapter: 12
title: "Kustomize on the Exam — Environment, Workflow, Field Cheat-Sheet, Traps"
examinable: true
kind: notes
related:
  - 03-install-kustomization-build-output.md
  - 05-transformers.md
  - 06-patches-intro.md
  - 09-overlays.md
  - 11-generators.md
  - ../10-helm-fund/03-helm-exam-patterns.md
---

# Kustomize on the Exam — Environment, Workflow, Field Cheat-Sheet, Traps

Chapters 01–11 cover the mechanics. This one is the exam view: what the
terminal actually gives you, the three-command workflow, one table that maps
a requirement to a `kustomization.yaml` field, and the mistakes that cost
points under time pressure.

---

## 1. What you have on the exam node

`kubectl` with Kustomize built in. That is the only guarantee:

```bash
kubectl kustomize <dir>       # render to stdout
kubectl apply -k <dir>        # render + apply
kubectl delete -k <dir>       # render + delete
kubectl diff -k <dir>         # what apply would change
```

The standalone `kustomize` binary — and with it every `kustomize edit set
...` / `kustomize create` shortcut shown in earlier chapters — is **not**
promised. Practise editing `kustomization.yaml` in vim, and know the field
names cold. kubectl 1.33 embeds Kustomize v5, so the modern fields
(`patches`, `labels`, `replicas`, a base listed under `resources`) all work;
the deprecated spellings (`bases`, `commonLabels`, `patchesStrategicMerge`,
`patchesJson6902`) still work too and appear in existing files.

There is no `kubectl explain` for `kustomization.yaml`. The field reference
at kubectl.docs.kubernetes.io is allowed during the exam; a wrong field name
is also caught instantly by `kubectl kustomize` (*unknown field*).

---

## 2. The workflow, every time

```bash
cd <dir>; ls -R; cat kustomization.yaml       # what's there, where's the base
kubectl kustomize .                           # render — read what you're about to apply
vim kustomization.yaml                        # change
kubectl kustomize . | grep -E "replicas:|image:|namespace:"   # confirm the change landed
kubectl apply -k .
kubectl -n <ns> get all                       # the cluster is graded too
```

Render before apply. It is free, it catches typos, and when a task asks for
the rendered output in a file, it *is* the answer:
`kubectl kustomize . > /path/out.yaml`. Count resources with
`kubectl kustomize . | grep -c "^kind:"`.

Apply the **overlay**, not the base, and point every command at the
directory containing `kustomization.yaml`, never at the file.

---

## 3. Requirement → field

| The task wants | Field | Notes |
|---|---|---|
| everything in Namespace X | `namespace: X` | overrides what resources say |
| names prefixed/suffixed | `namePrefix: stg-` / `nameSuffix: -v2` | references are rewritten too |
| a label on every resource | `labels: [{pairs: {k: v}}]` | selectors untouched (`includeSelectors: false` default) |
| a label on everything **and** in selectors | `commonLabels: {k: v}` | immutable-selector risk on live Deployments |
| an annotation everywhere | `commonAnnotations: {k: v}` | |
| N replicas for Deployment D | `replicas: [{name: D, count: N}]` | `name` is the name in the base, before any prefix |
| image tag/name change | `images: [{name: nginx, newTag: "1.27"}]` | matched by image name, not container name |
| change one field of one resource | `patches:` | strategic merge fragment, or JSON 6902 op list with `target:` |
| ConfigMap/Secret from files or literals | `configMapGenerator` / `secretGenerator` | hash suffix on, references rewritten (Ch11) |
| this overlay builds on the base | `resources: [../../base]` | a directory is a resource |
| include extra manifests only here | `resources: [../../base, extra.yaml]` | |
| opt into a reusable feature | `components: [../../components/x]` | `kind: Component` on the other side |

Skeleton of an overlay that uses most of it:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: zircon
namePrefix: stg-
resources:
  - ../../base
replicas:
  - name: web
    count: 2
images:
  - name: nginx
    newTag: 1.27-alpine
labels:
  - pairs:
      env: staging
patches:
  - path: patch-resources.yaml            # strategic merge: a Deployment fragment
  - target: {kind: Deployment, name: web} # JSON 6902: needs a target
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value: {name: DEBUG, value: "true"}
configMapGenerator:
  - name: app-config
    files: [app.properties]
```

---

## 4. Choosing the patch type

| | Strategic merge | JSON 6902 |
|---|---|---|
| Looks like | a slice of the manifest (`kind`, `metadata.name`, the fields) | `op` / `path` / `value` list + `target:` |
| Lists of maps (`containers`, `env`) | matched by `name` | by index (`/0`) or append (`/-`) |
| Delete | `key: null`, `$patch: delete` | `op: remove` |
| Best for | "set these fields on container X" | "remove this annotation", "change env[0]", CRDs |

Both go under `patches:`. Reach for strategic merge when the change reads as
"here is the bit that's different"; reach for JSON 6902 when you need a
precise operation on a list position or a key with a `/` in it.

---

## 5. Traps that cost points

- **`-f` vs `-k`.** `kubectl apply -f <dir>` applies raw files and fails on
  `kustomization.yaml`. Kustomize is `-k`.
- **The base is not the deliverable.** "Do not modify the base" is common;
  the overlay is where the change goes, and graders can checksum the base.
- **`replicas.name` / patch `metadata.name` use the base name.** With
  `namePrefix: stg-`, the Deployment is still `web` to the kustomization.
- **`images.name` is the image, not the container.** `- name: web` matches
  nothing when the image is `nginx`.
- **`commonLabels` rewrites selectors.** On a Deployment that already exists,
  the apply fails with *field is immutable*. "Without touching selectors"
  means `labels:`.
- **`apply -k` never prunes.** Removing a resource from a kustomization and
  re-applying leaves it running. Set a canary to 0 instead of deleting it;
  clean up with `delete -k` or `kubectl delete`.
- **`delete -k` deletes exactly what the overlay renders.** Deleting the
  Namespace instead is usually forbidden by the task and always removes more.
- **JSON pointers:** `/` inside a key is `~1`, `~` is `~0`; `/-` appends,
  `/0` inserts at the front; `replace` needs the path to exist.
- **Env values are strings.** `value: true` unquoted becomes a boolean and
  the API server rejects it; write `"true"`. Same for `"8080"`.
- **Paths are relative to the kustomization that lists them.** From
  `overlays/prod/`, the base is `../../base`. Count the hops.
- **Strict field names.** `commonLabels` (plural), `namePrefix`,
  `configMapGenerator`, `patches`. *Unknown field* means a typo in
  `kustomization.yaml`, never in the manifests.
- **Order for "save the render to a file":** make the change first, then
  render. A file rendered before the edit is graded as wrong.

---

## 6. Wording → field

| The question says | You write |
|---|---|
| "deploy the overlay" / "apply the Kustomization" | `kubectl apply -k <dir>` |
| "show / save what it would deploy" | `kubectl kustomize <dir> [> file]` |
| "remove everything the overlay deployed" | `kubectl delete -k <dir>` |
| "all resources into Namespace X (in the Kustomization)" | `namespace: X` |
| "prefix every name with" | `namePrefix:` |
| "N replicas" | `replicas:` |
| "use image tag T" / "replace image A by B" | `images:` (`newTag` / `newName`) |
| "add label L to every resource" | `labels:` (or `commonLabels` if selectors should change) |
| "add a probe / resources / env to container C" | strategic merge `patches:` naming `- name: C` |
| "remove annotation / change the first env var" | JSON 6902 `patches:` with `target:` |
| "generate a ConfigMap from file F" | `configMapGenerator: files: [F]` |
| "the Secret must be named exactly S" | `options: {disableNameSuffixHash: true}` |
| "enable feature X only for prod" | `components:` in the prod overlay |

## References

- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) — the kubernetes.io task page: `kubectl kustomize`, `apply -k`, `delete -k`, generators, transformers, patches, bases and overlays
- [kubectl kustomize](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/) — the built-in render command and its flags
- [The Kustomization File](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) — every field in one place
- [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/) — strategic merge vs JSON 6902, `target:` selectors
