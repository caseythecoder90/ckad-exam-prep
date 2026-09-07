---
section: 11-kustomize
chapter: 13
title: "Memorize Cold — no Kustomize docs on the exam"
examinable: true
kind: reference
related:
  - 05-transformers.md
  - 07-patches-dictionaries.md
  - 08-patches-lists.md
  - 11-generators.md
  - 12-exam-patterns.md
---

# Memorize Cold — no Kustomize docs on the exam

`kubectl.docs.kubernetes.io` is **not** on the allowed-docs list. The only
Kustomize page reachable is
`kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/` — it has a
worked example of every field below, so it is the fallback, but it is slow to
search under time. Everything on this page should come out of your fingers
without opening it.

Deliberate subset of Ch03–Ch12. Nothing new, nothing optional.

---

## 1. The four commands

```bash
kubectl kustomize <dir>            # render to stdout — do this before every apply
kubectl apply -k <dir>             # render + apply
kubectl delete -k <dir>            # render + delete
kubectl diff -k <dir>              # what apply would change
```

`-k` takes the **directory**, never the file, and points at the **overlay**.
`kubectl kustomize . > out.yaml` is the answer to "save the rendered manifests".
Assume the standalone `kustomize` binary does not exist.

---

## 2. `kustomization.yaml` from blank

Exact spellings. `unknown field` always means a typo here, never in the manifests.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod
namePrefix: stg-
nameSuffix: -v2
commonAnnotations: {owner: web}

resources:
  - ../../base            # a base is just a directory in resources:
  - extra.yaml

labels:                   # metadata.labels only — selectors untouched
  - pairs: {env: staging}
commonLabels: {team: web} # labels AND selectors — immutable on live Deployments

replicas:
  - name: web             # name in the BASE, before namePrefix
    count: 2

images:
  - name: nginx           # matched by IMAGE name, not container name
    newName: haproxy
    newTag: "1.27"

components:
  - ../../components/monitoring
```

Wrong more often than anything else here: `replicas[].name` and a patch's
`metadata.name` use the **base** name (prefix not applied yet), and
`images[].name` is the **image**, not the container.

---

## 3. Generators

```yaml
configMapGenerator:
  - name: app-config
    files:
      - app.properties                # key = file name
      - index.html=/tmp/page.html     # key = index.html
    literals:
      - LOG_LEVEL=warn
    envs:
      - app.env                       # one key per KEY=VALUE line
    options: {disableNameSuffixHash: true}

secretGenerator:
  - name: db-creds
    literals: [username=lapis, password=ruby42]
    type: Opaque                      # or kubernetes.io/tls
    behavior: merge                   # create (default) | merge | replace

generatorOptions:
  disableNameSuffixHash: true         # every generator in this file
```

- The name gets a **content hash** suffix, and every reference in the same build
  (`configMapRef`, `secretKeyRef`, `envFrom`, `volumes[].configMap.name`,
  `volumes[].secret.secretName`) is rewritten to match. Never hard-code the hash.
- "Must be named exactly X" → `disableNameSuffixHash: true`.
- File paths are relative to the kustomization declaring the generator.
- `behavior:` only when overriding a generator the **base** already declares.

---

## 4. Strategic merge — the four moves

A fragment of the real manifest. Self-identifying: needs `apiVersion`, `kind`,
`metadata.name`. No `target:`.

```yaml
patches:
  - path: patch.yaml     # file
  - patch: |-            # or inline
      apiVersion: apps/v1
      kind: Deployment
      metadata: {name: web}
      spec: {replicas: 3}
```

```yaml
# patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    org: kk            # 1. ADD/REPLACE a map key — just name it
    stale: null        # 2. REMOVE a map key — ONLY null does this
spec:
  template:
    spec:
      containers:
        - name: web    # 3. EDIT — match by merge key, supply changed fields
          image: nginx:1.27
        - name: cache  # 4a. ADD — a key not in the base is appended
          image: redis
        - name: db     # 4b. REMOVE a list element
          $patch: delete
```

Merge is **additive** — omitting a key never deletes it. Deletion is always an
explicit directive: `key: null` for maps, `$patch: delete` for keyed lists.

Merge key for `containers`, `initContainers`, `env`, `volumes`, `ports`,
`volumeMounts` is `name`. **Scalar lists have no merge key** — `command:` and
`args:` are **replaced wholesale**, never merged element-by-element.

---

## 5. JSON 6902 — anatomy

```yaml
patches:
  - target:                 # REQUIRED — this is what identifies the resource
      group: apps           # omit for core/v1
      version: v1
      kind: Deployment
      name: web             # a REGEX: web.* matches web and web-canary
      namespace: prod       # optional
      labelSelector: "env=prod"   # optional
    patch: |-               # inline op list
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value: {name: DEBUG, value: "true"}
  - target: {kind: Deployment, name: web}
    path: ops.yaml          # or the op list in its own file
```

`kind` + `name` is enough almost every time. The patch body is a **list of ops** —
no `apiVersion`/`kind` inside it.

| op | path missing | path present | takes `value:` |
|---|---|---|---|
| `add` | creates it | **overwrites** it | yes |
| `replace` | **errors** | overwrites it | yes |
| `remove` | **errors** | deletes it | **no** |

When unsure whether the target exists, use `add`. A `remove` on something that
isn't there fails the whole build — render and grep first.

---

## 6. JSON 6902 — the path vocabulary

Paths are the whole game. Learn these; everything else is a variation.

```
/metadata/name
/metadata/labels/<key>
/metadata/annotations/<key>
/spec/replicas
/spec/template/metadata/labels/<key>
/spec/template/metadata/annotations/<key>
/spec/template/spec/serviceAccountName
/spec/template/spec/containers/0/image
/spec/template/spec/containers/0/env/-
/spec/template/spec/containers/0/env/0/value
/spec/template/spec/containers/0/args/-
/spec/template/spec/containers/0/ports/0/containerPort
/spec/template/spec/containers/0/resources/limits/memory
/spec/template/spec/containers/0/volumeMounts/-
/spec/template/spec/volumes/-
/spec/ports/0/port                        # Service
/spec/rules/0/http/paths/0/path           # Ingress
```

Deployment fields sit under `/spec`; Pod fields sit under
`/spec/template/spec`. Getting that one hop wrong is the most common silent
failure.

---

## 7. JSON 6902 — maps vs lists

**Maps** — the last segment is a key:

```yaml
- op: add                                       # create or overwrite
  path: /spec/template/metadata/labels/org
  value: kk
- op: remove                                    # no value:
  path: /metadata/labels/stale
```

`add` writes a key **into a map that already exists**. It does not create the
map. If the resource has no `annotations:` at all, add the map itself:

```yaml
- op: add
  path: /metadata/annotations
  value: {owner: web}       # keys written VERBATIM here — escaping is path-only
```

**Lists** — the last segment is a position:

| Segment | Meaning |
|---|---|
| `/0`, `/1` | that index |
| `/-` | the slot after the last element → **append** |

```yaml
- op: add                                  # APPEND
  path: /spec/template/spec/containers/-
  value: {name: cache, image: redis}

- op: add                                  # INSERT at the front, shifts the rest
  path: /spec/template/spec/containers/0
  value: {name: sidecar, image: envoy}

- op: replace                              # swaps the ENTIRE element
  path: /spec/template/spec/containers/0
  value: {name: web, image: nginx}

- op: replace                              # edit ONE field — path INTO the element
  path: /spec/template/spec/containers/0/image
  value: nginx:1.27

- op: remove
  path: /spec/template/spec/containers/1
```

The two that cost points: `add /list/0` **inserts** (overwriting is `replace`),
and `replace /containers/0` **wipes the whole container** when you meant to
change only the image.

Indexes are build-time positions. `kubectl kustomize . | grep -n "name:"` and
read the real order before trusting `/1`.

---

## 8. Escaping — RFC 6901

The path is `/`-delimited, so a `/` inside a key must be escaped.

| In the key | In the path |
|---|---|
| `~` | `~0` |
| `/` | `~1` |

**Order matters: replace `~` first, then `/`.** The other way round turns the
`~` you just wrote in `~1` into `~01` and corrupts the pointer.

Dots, dashes and underscores are **not** special — never escape them.

```
app.kubernetes.io/name                            -> app.kubernetes.io~1name
app.kubernetes.io/managed-by                      -> app.kubernetes.io~1managed-by
nginx.ingress.kubernetes.io/rewrite-target        -> nginx.ingress.kubernetes.io~1rewrite-target
kubectl.kubernetes.io/last-applied-configuration  -> kubectl.kubernetes.io~1last-applied-configuration
cost~center                                       -> cost~0center
a/~b                                              -> a~1~0b
```

Worked — remove an annotation whose key contains a slash:

```yaml
patches:
  - target: {kind: Deployment, name: web}
    patch: |-
      - op: remove
        path: /metadata/annotations/nginx.ingress.kubernetes.io~1rewrite-target
```

Escape **only in `path:`**. Inside `value:` the key is ordinary YAML:

```yaml
- op: add
  path: /metadata/labels/app.kubernetes.io~1version   # escaped
  value: "1.2.3"
- op: add
  path: /metadata/labels                              # the whole map
  value:
    app.kubernetes.io/version: "1.2.3"                # NOT escaped
```

Miss the `~1` and the pointer reads as two levels — a `name` key under a
nonexistent `app.kubernetes.io` map — so the build fails or the patch no-ops.
Strategic merge sidesteps escaping entirely: if the task lets you choose and the
key has a `/`, take strategic merge.

---

## 9. Traps

- `apply -f <dir>` is wrong. Kustomize is `-k`.
- "Do not modify the base." The change goes in the overlay.
- **Env values are strings.** `value: "true"`, `value: "8080"` — unquoted they
  become bool/int and the API server rejects the Pod.
- `remove` and `replace` error when the path doesn't exist; `add` never does.
- Relative paths resolve from the kustomization that lists them — `../../base`
  from `overlays/prod/`.
- `apply -k` **never prunes**. Deleting a resource from the kustomization leaves
  it running; scale it to 0 or `kubectl delete` it.
- Make the change **before** rendering to a file.
- `commonLabels` on a live Deployment → `field is immutable`. "Without changing
  selectors" means `labels:`.

---

## 10. Drill from blank

Cover the page. Target is under a minute each.

1. Overlay `kustomization.yaml` on `../../base`: namespace `prod`, prefix
   `stg-`, 3 replicas of `web`, image `nginx:1.27`.
2. Strategic-merge patch: set `web`'s memory limit to `256Mi` and delete the
   `db` container.
3. JSON 6902: append env `MODE=debug` to container 0 of Deployment `web`.
4. JSON 6902: remove annotation `app.kubernetes.io/managed-by` from Service
   `api`.
5. JSON 6902: change the image of container 1 without touching its other fields.
6. ConfigMap `nginx-conf` from file `default.conf`, named exactly that.
7. Add a label to every resource without touching any selector.

## References

- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) — the only Kustomize page reachable from the exam browser; generators, transformers, patches, bases and overlays with worked examples
- [kubectl kustomize](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/) — the built-in render command
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/) — `-k` on the apply side
