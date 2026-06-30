---
section: "11-kustomize"
chapter: "04"
title: "Managing Directories (Nested kustomization.yaml)"
course_ref: "Mumshad Mannambeth CKAD — Kustomize: Managing Directories"
examinable: true
kind: "hands-on"
companion_diagrams:
  - "diagrams/08-managing-directories-evolution.png"
  - "diagrams/09-nested-kustomization-mechanic.png"
---

# Managing Directories (Nested kustomization.yaml)

Continuation of ch.03. Once a project has several services, you split them into
subdirectories (`api/`, `db/`, …). This chapter is about how Kustomize scales that
up: the one new mechanic is that **a `kustomization.yaml` can reference a
directory, not just a file** — which lets you nest kustomizations.

---

## 1. The setup and the naive apply

Files organised by service under `k8s/`:

```
k8s/
├── api-depl.yaml
├── api-service.yaml
├── db-depl.yaml
└── db-service.yaml
```

…or, more realistically, split into per-service subdirectories:

```
k8s/
├── api/
│   ├── api-depl.yaml
│   └── api-service.yaml
└── db/
    ├── db-depl.yaml
    └── db-service.yaml
```

You *can* still deploy this with plain kubectl, one directory at a time:

```bash
kubectl apply -f k8s/api/
kubectl apply -f k8s/db/
```

That's fine for two directories. Add `cache/`, `kafka/`, `worker/`… and it's one
more command every time — tedious and easy to forget one.

---

## 2. Step 1 — a single flat kustomization.yaml

Put one `kustomization.yaml` at `k8s/` listing every manifest (note the subdir
path prefixes), and Kustomize does the applying for you in **one** command:

```yaml
# k8s/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - api/api-depl.yaml
  - api/api-service.yaml
  - db/db-depl.yaml
  - db/db-service.yaml
```

```bash
kustomize build k8s/ | kubectl apply -f -      # or: kubectl apply -k k8s/
```

Better — one command regardless of directory count. **But** it doesn't fully
scale either: every new manifest is another line you must remember to add to this
central list. The toil moved from "+1 command per dir" to "+1 line per file."

![Three approaches side by side: per-dir kubectl apply (grows with dirs), one flat kustomization listing every file (grows with files), nested kustomizations (top stays small)](./diagrams/08-managing-directories-evolution.png)

---

## 3. Step 2 — nested kustomizations (the actual lesson)

Give **each subdirectory its own `kustomization.yaml`** that lists *that
directory's* files, then have the **top-level** `kustomization.yaml` reference the
**directories** instead of individual files:

```yaml
# k8s/kustomization.yaml   (top level)
resources:
  - api/
  - db/
  - cache/
  - kafka/
```

```yaml
# k8s/db/kustomization.yaml   (per-service)
resources:
  - db-depl.yaml
  - db-service.yaml
```

![Nested mechanic: the top kustomization references directories api/ db/ cache/ kafka/, each of which contains its own kustomization.yaml listing its files; build recurses](./diagrams/09-nested-kustomization-mechanic.png)

`kubectl apply -k k8s/` now builds the top level, **recurses into each referenced
directory**, builds each one, and applies the combined result — all in one shot.
The top-level file stays tiny (one line per service); each service owns and
manages its own manifests. That's the structure that actually scales.

> The key realisation: `resources:` entries can be **files or directories**. A
> directory entry is itself built as a kustomization and folded in. That single
> rule is the whole nesting feature.

---

## 4. Mechanics & gotchas

- **A directory in `resources:` must contain its own `kustomization.yaml`.** It is
  *not* a loose `-f` folder. `kubectl apply -f some/dir/` applies every YAML in
  that dir as-is; `resources: [ some/dir/ ]` requires `some/dir/kustomization.yaml`
  to exist, or build fails. Different rules — don't conflate them.
- **`build` recurses.** Building the top level builds every nested kustomization
  beneath it, depth permitting. One `-k` covers the whole tree.
- **Relative paths** are relative to the location of the `kustomization.yaml` that
  names them (so `../../base/` walks up from an overlay).
- **Transformers cascade.** A transformer at the top level (e.g. `commonLabels`,
  `namespace`) applies to **everything** pulled in from nested kustomizations.
  Each nested kustomization can *also* run its own transformers first, at its own
  level. Inner transforms happen, then outer transforms apply over the merged
  result — this layering is exactly what makes overlays work (next point).
- **`bases:` was the old nesting field** — it existed *specifically* to reference
  base directories. It's deprecated; modern kustomize folds base dirs straight
  into `resources:`. (Flagged in ch.03; this is *why* it existed.)

---

## 5. Imperative / exam shortcuts

```bash
kubectl apply  -k k8s/         # build top, recurse, apply
kubectl kustomize k8s/         # render the whole nested tree to stdout (inspect!)
kustomize build k8s/           # same render via the standalone CLI

# scaffold a kustomization.yaml in the current dir from the YAML already there
kustomize create --autodetect          # in k8s/db/, picks up db-*.yaml
kustomize edit add resource cache/      # add a dir/file to resources: in place
```

When a nested build misbehaves, `kubectl kustomize k8s/` (render-only) is the
fastest way to see the fully-merged output and spot the missing/duplicated
resource before you ever apply.

---

## 6. JPMC / GKP grounding — you already nest, by environment

What you described at work *is* nested kustomizations — you just nest **vertically
by environment** instead of horizontally by service:

```
kube/kustomize/
├── base/
│   └── kustomization.yml          # resources: [ service.yml, ... ]   (Image 5)
└── overlays/
    ├── dev/
    │   └── kustomization.yml       # bases: [ ../../base/ ]; resources/images/namespace/configMapGenerator  (Image 4)
    ├── stg/
    │   └── kustomization.yml
    └── prod/
        └── kustomization.yml
```

Two levels of `kustomization.yml`: one at `base/`, one in each overlay. The
overlay's `kustomization.yml` **references the base directory** (`bases: ../../base/`,
i.e. the deprecated spelling of `resources: ../../base/`) — that reference *is* a
nested kustomization. So the base+overlay model from ch.01 is the same directory-
nesting mechanic this lecture teaches; ch.01 nests by env, this lecture nests by
service.

Your own realisation is the right one: if a single environment's manifest count
keeps growing, you can nest **again inside** each overlay — split `dev/` into
`dev/api/`, `dev/db/`, … each with its own `kustomization.yml`, and have
`dev/kustomization.yml` reference those subdirs. You'd then be nesting on **both**
axes (environment × service), which is exactly how large GKP-style repos stay
organised. Transformer cascade makes this clean: env-level concerns
(`namespace: ${namespace}`, the image rewrite) sit in the overlay and apply down
over every service folder automatically.

> CKAD scope: directory references in `resources:`, nested builds, and recursion
> are fair game. The `base/ + overlays/` split and `${...}` injection are GKP
> flavour — the mechanism underneath is the examinable part.

---

## TL;DR

- A `kustomization.yaml` `resources:` entry can be a **file or a directory**; a
  directory entry must itself contain a `kustomization.yaml`.
- Evolution: per-dir `kubectl apply` (grows with dirs) → one flat kustomization
  (grows with files) → **nested** kustomizations (top lists only dirs, each dir
  self-manages) — only the last scales.
- `kubectl apply -k k8s/` builds the top and **recurses** into every referenced
  directory, applying the whole tree in one command.
- Top-level transformers **cascade** onto everything pulled in from children.
- Your GKP base+overlay layout already is nested kustomization (by environment);
  you can nest again by service inside an overlay as files grow.

## Quick recall

- [ ] Can `resources:` reference a directory? → yes, if it holds a `kustomization.yaml`.
- [ ] File-dir vs `-f` dir? → `resources: dir/` needs a kustomization.yaml; `-f dir/` applies loose YAML.
- [ ] Does `build` recurse into nested kustomizations? → yes.
- [ ] Where do relative paths resolve from? → the location of the referencing `kustomization.yaml`.
- [ ] Do top-level transformers affect nested resources? → yes, they cascade over the merged output.
- [ ] One command to deploy a nested tree? → `kubectl apply -k k8s/`.
- [ ] Deprecated nesting field? → `bases:` (now use `resources:`).

## Resolved threads

- *How an overlay references its base / the base+overlay structure* (open since
  ch.01) → it's directory nesting via `resources:` (old `bases:`), resolved here.

## Open threads (carried into ch.05+)

- [ ] Patch mechanisms in depth: strategic-merge patch vs JSON6902 patch vs the
      inline `replicas:`/`images:` transformers — when to use which.
- [ ] `namePrefix`/`nameSuffix`, `replacements:` (modern replacement for `vars:`).
- [ ] Full `jules.yml → injected → kustomize build → rendered` trace — pending the `jules.yml`.
