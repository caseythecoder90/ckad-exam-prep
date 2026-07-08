---
section: 11-kustomize
chapter: 07
title: "Patches — Dictionaries (add / replace / remove)"
course_ref: "Mumshad Mannambeth CKAD — Kustomize: Patches (Dictionaries)"
examinable: true
kind: notes
companion_diagrams:
  - diagrams/14-json6902-dictionary-ops.png
  - diagrams/15-strategic-merge-dictionary-ops.png
  - diagrams/16-json-pointer-escaping.png
---

# Patches — Dictionaries (add / replace / remove)

Ch06 split patches into two flavours — **JSON 6902** (explicit `op`/`path`) and
**strategic merge** (a fragment of the real manifest) — and two layouts (**inline**
`patch: |-` vs **separate file** `path: patch.yaml`). This chapter drills into operating
on a **dictionary** (a map field such as `metadata.labels`): the three operations
**add / replace / remove**, done both ways. Lists are a separate chapter (Ch08).

---

## 1. The mental model: same outcome, two engines

A patch always answers two questions: **which resource** and **what change**. The two
flavours differ in *how you express the change*.

| | **JSON 6902** | **Strategic merge** |
|---|---|---|
| Shape | explicit op list: `op` + `path` + `value` | a partial copy of the manifest |
| Targeting | `target:` selector (kind/name/…) in the kustomization | identifying fields (`apiVersion`,`kind`,`metadata.name`) **inside the patch** |
| Reads like | a surgical instruction | "here's the bit that's different" |
| Needs a schema? | **No** — purely structural, works on any field / CRD | **Yes-ish** — uses the type's patch metadata; CRDs without a schema fall back to JSON-merge semantics |
| Deletes by | `op: remove` | setting the key to **`null`** |

Both are listed under the **unified `patches:` field**. Kustomize decides which engine to
use: a list item with a `patch:` string containing `op:`/`path:` is JSON6902; a list item
pointing at (or inlining) a manifest fragment is strategic merge.

> **Precision flag (deprecated fields).** Older repos use the *split* fields
> `patchesJson6902:` and `patchesStrategicMerge:`. These still work but are **superseded
> by the single `patches:` field** — prefer `patches:` in anything new. The lecture's
> screenshots already use the modern `patches:` form.

---

## 2. JSON 6902 — dictionary ops

JSON6902 (RFC 6902, "JSON Patch") is a list of operations. Each op names a `path`
(an RFC 6901 **JSON Pointer**, `/`-delimited) and, for add/replace, a `value`.

![JSON6902 dictionary operations: add, replace, remove](./diagrams/14-json6902-dictionary-ops.png)

```yaml
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: add                                    # create-or-overwrite
        path: /spec/template/metadata/labels/org
        value: KodeKloud
      - op: replace                                # key MUST already exist
        path: /spec/template/metadata/labels/component
        value: web
      - op: remove                                 # no value: field
        path: /spec/template/metadata/labels/org
```

Operation semantics — the part the lecture glosses over:

| op | key missing | key present | needs `value:` |
|---|---|---|---|
| `add` | creates it | **overwrites** it | yes |
| `replace` | **errors** | overwrites it | yes |
| `remove` | **errors** | deletes it | no |

So `add` is the safe choice when you're unsure whether the key exists; `replace` is the
stricter, more self-documenting choice when you *expect* it to be there and want a loud
failure if your assumption is wrong.

---

## 3. Strategic merge — dictionary ops

Strategic merge is "**describe the delta in the shape of the manifest**." You write a
fragment that mirrors the path down to the field, and kustomize **deep-merges** it onto
the base. The fragment carries `apiVersion`/`kind`/`metadata.name` so kustomize knows
*which* resource to merge into.

![Strategic merge dictionary operations, including null removal](./diagrams/15-strategic-merge-dictionary-ops.png)

```yaml
# label-patch.yaml  (referenced by  patches: [ - path: label-patch.yaml ]  or inline)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: KodeKloud      # add: name a new key
        component: web      # replace: name an existing key with a new value
        # org: null         # remove: the ONLY way to delete with strategic merge
```

### 3.1 Why deletion needs an explicit `null`

This is the subtle bit. A merge **only ever adds or overwrites the keys you name** —
everything you *don't* mention is left exactly as it was. That's the whole point of a
merge: it's additive by default. So the engine has **no way to infer "delete"** from a
key simply being absent from your fragment — absence means "leave it alone," not "remove
it."

To delete, you have to give an **explicit directive**. For a dictionary key, that
directive is **`key: null`**. (For *lists*, the equivalent directive is `$patch: delete`
— that's a Ch08 problem.)

> **Gotcha, stated plainly:** omitting a key in a strategic-merge patch does **not**
> remove it. Only `key: null` does.

---

## 4. JSON-Pointer escaping — the slash trap

A JSON6902 `path` is `/`-delimited, so a **literal `/` inside a key name** (extremely
common in Kubernetes label/annotation keys) collides with the delimiter and must be
escaped. This is almost never shown in intro courses and it *will* waste an afternoon
the first time it bites you.

![JSON-Pointer escaping: ~1 for slash, ~0 for tilde](./diagrams/16-json-pointer-escaping.png)

RFC 6901 escapes (apply in this order):

| In the key | Write in the path |
|---|---|
| `~` | `~0`  *(do this first)* |
| `/` | `~1` |

```yaml
# Target label:  app.kubernetes.io/name: api
patch: |-
  - op: replace
    path: /spec/template/metadata/labels/app.kubernetes.io~1name   # ~1 == '/'
    value: web
```

Skip the escape and the pointer reads `…/labels/app.kubernetes.io/name` as *two* levels —
a `name` key under a non-existent `app.kubernetes.io` map — and the patch fails or
silently no-ops. **Strategic merge sidesteps this entirely**: you write the key verbatim
in YAML, no pointer involved.

---

## 5. Inline vs separate file (recap)

Either flavour can live **inline** in the kustomization or in its **own file** — purely a
layout choice, identical output.

```yaml
patches:
  # inline strategic merge
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata: { name: api-deployment }
      spec: { template: { metadata: { labels: { org: KodeKloud } } } }

  # separate-file strategic merge
  - path: label-patch.yaml

  # inline JSON6902 (note the target: selector)
  - target: { kind: Deployment, name: api-deployment }
    patch: |-
      - op: add
        path: /spec/template/metadata/labels/org
        value: KodeKloud
```

Rule of thumb: inline for one or two trivial edits, separate file once the patch is more
than a few lines or is reused/reviewed on its own.

---

## 6. Exam-pattern gotchas

- **Strategic-merge delete = `null`.** Omitting the key does nothing. If a task says
  "remove label X with a strategic-merge patch," the answer is `X: null`.
- **`replace` errors on a missing key; `add` is create-or-overwrite.** If you're not
  sure the key exists, reach for `add`.
- **`remove` takes no `value:`.** Including one isn't always rejected but it's wrong;
  don't muscle-memory a value onto a remove.
- **Escape `/` in keys as `~1`** (and `~` as `~0`, first) inside JSON6902 paths. This is
  the single most common JSON6902 mistake with real-world labels/annotations.
- **JSON6902 needs `target:`; strategic merge self-identifies.** A JSON6902 patch with no
  `target:` has nothing to attach to. A strategic-merge fragment missing `metadata.name`
  can't be matched.
- **Patch vs transformer for labels:** a `labels:`/`commonLabels` transformer rewrites
  labels *and selectors* across resources; a dictionary patch on `metadata.labels`
  touches **only that one label on that one resource** and leaves selectors alone. Pick
  the surgical tool when the task is surgical.
- **CRDs / arbitrary fields → prefer JSON6902.** Strategic merge depends on the type's
  schema; for a CRD without one it degrades to plain JSON-merge behaviour, which can
  surprise you on lists. JSON6902 is schema-agnostic.

---

## 7. Imperative / `kustomize edit` shortcuts

There's no fast imperative way to *compose* a patch body — you'll hand-write the YAML.
What you can script is **registering** a patch file in the kustomization:

```bash
# add a strategic-merge / json6902 patch entry, scoped to a target
kustomize edit add patch \
  --path label-patch.yaml \
  --group apps --version v1 --kind Deployment --name api-deployment

# preview the merged result before applying (exam-safe verification)
kubectl kustomize overlays/dev          # or:  kustomize build overlays/dev
```

Always `kubectl kustomize <dir>` to **confirm the field actually changed** in the rendered
output before `kubectl apply -k`. On the exam this catches a wrong path or a forgotten
`~1` immediately.
