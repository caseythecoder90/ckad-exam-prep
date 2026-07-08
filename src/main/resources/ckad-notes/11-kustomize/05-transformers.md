---
section: "11-kustomize"
chapter: "05"
title: "Transformers (Common + Image)"
course_ref: "Mumshad Mannambeth CKAD — Kustomize: Common Transformers + Image Transformer (2 lectures merged)"
examinable: true
kind: "hands-on"
note: "Refactor: the short Image Transformer lecture is merged in here with the Common Transformers lecture. This file REPLACES the earlier 05-common-transformers.md. Patches come next and will be grouped on their own."
companion_diagrams:
  - "diagrams/10-common-transformers.png"
  - "diagrams/11-image-transformer.png"
related_diagrams:
  - "diagrams/07-commonlabels-transformer.png"   # commonLabels deep-dive (ch.03)
---

# Transformers (Common + Image)

A **transformer** is how Kustomize changes your configs during `build`. Kustomize
ships several **built-in** transformers and lets you write **custom** ones. Two
built-in families here:

- **Common transformers** — *blanket*: stamp the same thing onto **every**
  resource (`commonLabels`, `commonAnnotations`, `namespace`, `namePrefix`/
  `nameSuffix`).
- **Image transformer** — *targeted*: find a specific image by name and rewrite
  it (`images:`).

> Useful mental model for the rest of the section: transformers run on a
> blanket → targeted → surgical spectrum. Common transformers are **blanket**
> (apply X to all). The image transformer is **targeted** (match this image,
> change it). **Patches** (next chapters) are **surgical** (change this exact
> field in this exact resource). Same `build`, increasing precision.
>
> Field names matter on the exam — memorise the exact spellings.

---

## Part A — Common transformers

### The four

| Field (exact spelling) | What it does | Where it lands |
|---|---|---|
| `commonLabels` | add a label to **all** resources | `metadata.labels` **+ selectors + pod template** |
| `commonAnnotations` | add an annotation to all resources | `metadata.annotations` (only) |
| `namespace` | put all resources in one namespace | `metadata.namespace` |
| `namePrefix` / `nameSuffix` | prepend / append to every resource **name** | `metadata.name` (+ name references) |

![One kustomization.yaml with namePrefix/nameSuffix/namespace/commonLabels/commonAnnotations and color-coded arrows showing each landing in a rendered Service](./diagrams/10-common-transformers.png)

> The real fields are **`commonLabels`** and **`commonAnnotations`** (plural) —
> build fails otherwise. Don't write them singular.

### Each one, concretely

**`commonLabels`** (recap from ch.03) — adds the label to every resource's
`metadata.labels` **and** into Deployment `selector.matchLabels`, pod
`template.metadata.labels`, and Service `selector`:

```yaml
commonLabels:
  org: KodeKloud
```

![commonLabels injected into 5 label sites across a Service and Deployment](./diagrams/07-commonlabels-transformer.png)

> **Hazard:** it rewrites **selectors**, and a Deployment's `spec.selector` is
> **immutable after creation** → adding it to a live Deployment is rejected
> (`field is immutable`). Modern, safer alternative: list-form `labels:` with
> `includeSelectors: false`.

**`commonAnnotations`** — adds to `metadata.annotations` on all resources; does
**not** touch selectors, so no immutability trap. Good for git branch/commit,
owner, ticket, build URL.

```yaml
commonAnnotations:
  branch: master
```

**`namespace`** — stamps `metadata.namespace` onto every **namespaced** resource,
overriding what was there. Cluster-scoped kinds (ClusterRole, PV, `Namespace`)
are left alone; it also fixes some cross-references (e.g. a RoleBinding's
ServiceAccount subject namespace).

```yaml
namespace: lab
```

**`namePrefix` / `nameSuffix`** — wrap every resource name
(`api-service` → `KodeKloud-api-service-dev`). Two surprises: it only changes
`metadata.name` (not labels/selectors, so label wiring still works), and
**Kustomize auto-updates name references** to the renamed resource (Service in an
Ingress backend, `configMapRef`/`secretRef`, etc.). Rename once; pointers follow.

```yaml
namePrefix: KodeKloud-
nameSuffix: -dev
```

---

## Part B — Image transformer

The image transformer changes the **image** a Deployment/container uses, through
Kustomize, without editing the manifest. Declared with `images:`:

```yaml
images:
  - name: nginx          # the EXISTING image to match
    newName: haproxy     # (optional) replace the repo/name
    newTag: "2.4"        # (optional) replace the tag
    # digest: sha256:...  # (optional) pin to a digest instead of a tag
```

![Image transformer: match by image name, with three outcomes — newName only -> haproxy, newTag only -> nginx:2.4, both -> haproxy:2.4](./diagrams/11-image-transformer.png)

How matching works (the part people trip on):

- **`name:` matches the container's image NAME, not the container name.** In a pod
  with `containers: [ - name: web, image: nginx ]`, the matcher is `nginx`, never
  `web`.
- **It rewrites every container** using that image, across all managed resources.
- The three modes (see diagram): `newName` only → `haproxy`; `newTag` only →
  `nginx:2.4`; both → `haproxy:2.4`. `digest:` pins to an immutable `sha256`
  instead of a tag.

Why use this instead of a patch to change an image: it's concise, it's the
intended tool, and it matches by image across the whole bundle in one declaration.

**Imperative (exam gold):**

```bash
kustomize edit set image nginx=haproxy:2.4     # writes the images: block for you
kustomize edit set image nginx=*:2.4           # just the tag
```

---

## Gotchas & depth (both families)

- **`commonLabels` → immutable selectors** — the single most likely Kustomize
  footgun on a live cluster.
- **Plural fields**: `commonLabels`, `commonAnnotations`.
- **Modern spellings**: `labels:`/`annotations:` (list form) give `includeSelectors`
  / `includeTemplates` control; prefer them when you must avoid selectors.
- **`namespace`** is ignored on cluster-scoped kinds (no error).
- **Affixes hit generated names** too: a `configMapGenerator` output can become
  `KodeKloud-app-config-dev-7c9f...` (prefix/suffix **and** content-hash).
- **Image transformer matches the image name** value, and only rewrites image
  references — it won't touch labels/selectors.

---

## Imperative / exam shortcuts

```bash
kustomize edit set namespace lab
kustomize edit set nameprefix KodeKloud-
kustomize edit set namesuffix -dev
kustomize edit set label org:KodeKloud
kustomize edit set annotation branch:master
kustomize edit set image nginx=haproxy:2.4
kubectl kustomize .            # render and verify the transforms landed
```

Always render (`kubectl kustomize <dir>`) and eyeball the output before applying.
