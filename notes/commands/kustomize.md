# Kustomize

Template-free config management: a **base** of plain YAML + **overlays** that patch it per environment. Built into kubectl via `-k`. The one rule to remember: **`-k` runs the base+overlay merge; `-f` applies raw files** (and chokes on `kustomization.yaml`).

The exam node guarantees only kubectl's built-in Kustomize. The standalone `kustomize` binary (and `kustomize edit ...`) may not exist — edit `kustomization.yaml` in vim and know the field names.

## Apply / render / delete (kubectl only — always available)

```bash
k kustomize <dir>                         # render merged output to stdout — no apply (do this first, every time)
k kustomize <dir> > /path/rendered.yaml   # "save the rendered manifests"
k kustomize <dir> | grep -c "^kind:"      # how many resources it renders
k apply -k <dir>                          # build + apply (never prunes removed resources)
k delete -k <dir>                         # build + delete exactly what the overlay renders
k diff -k <dir>                           # what apply would change
```

Point `-k` at the **directory** containing `kustomization.yaml`, and at the **overlay**, not the base.

## kustomization.yaml — one requirement, one field

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: zircon                     # every resource into this Namespace
namePrefix: stg-                      # (nameSuffix: -v2) names + references rewritten
resources:
  - ../../base                        # a base is a directory entry (bases: is the deprecated spelling)
  - extra.yaml                        # overlay-only manifests
replicas:
  - name: web                         # the name in the BASE, before any prefix
    count: 2
images:
  - name: nginx                       # matched by IMAGE name, not container name
    newTag: 1.27-alpine               # newName: for the repository, digest: to pin
labels:
  - pairs: {env: staging}             # every resource; selectors untouched (includeSelectors: false)
commonLabels: {team: web}             # also injects into selectors — immutable on live Deployments
commonAnnotations: {owner: web}
patches:
  - path: patch-resources.yaml        # strategic merge: a Deployment fragment with kind + metadata.name
  - target: {kind: Deployment, name: web}     # JSON 6902 needs a target
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value: {name: DEBUG, value: "true"}   # env values are strings: quote them
      - op: remove
        path: /metadata/annotations/example.com~1owner   # "/" in a key is ~1
configMapGenerator:
  - name: app-config
    files: [app.properties]           # key = file name  (key=file to rename)
    literals: [LOG_LEVEL=warn]
secretGenerator:
  - name: db-creds
    literals: [username=x, password=y]
    options: {disableNameSuffixHash: true}    # exact name; default adds a content hash and rewrites references
components:
  - ../../components/monitoring       # kind: Component, opt-in feature bundle
```

## Patches: strategic merge vs JSON 6902

| | Strategic merge | JSON 6902 |
|---|---|---|
| Form | partial manifest (`kind`, `metadata.name`, changed fields) | `op` / `path` / `value` list + `target:` |
| Lists of maps (`containers`, `env`) | keyed by `name` | positional: `/0`, append with `/-` |
| Delete | `key: null` / `$patch: delete` | `op: remove` |
| Use for | "set these fields on container X" | precise list ops, keys with `/`, CRDs |

Legacy fields `patchesStrategicMerge:` / `patchesJson6902:` still work; write `patches:`.

## Standalone CLI (where it exists)

```bash
kustomize build <dir>                     # render to stdout
kustomize build <dir> | kubectl apply -f -
kustomize build <dir> | kubectl delete -f -
kustomize create --autodetect --recursive     # generate kustomization.yaml from existing manifests
kustomize edit add resource service.yaml       # add to resources: (a file OR a directory)
kustomize edit add resource ../../base
kustomize edit add component ../../components/db
kustomize edit add patch --path patch.yaml --group apps --version v1 --kind Deployment --name api
kustomize edit set image nginx=haproxy:2.4     # newName + newTag
kustomize edit set image nginx=*:2.4           # change tag only
kustomize edit set namespace dev
kustomize edit set nameprefix KodeKloud-
kustomize edit set namesuffix -dev
kustomize edit set label org:KodeKloud
kustomize edit set annotation branch:master
kustomize edit set replicas nginx-deployment=5
kustomize version
```

Install: `curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash` (or `brew install kustomize`).

## Mental model

- `resources:` — base YAML files and/or directories (nested kustomizations recurse).
- Overlay = its own `kustomization.yaml` referencing the base + env-only resources + patches.
- Nothing prunes: removing a resource from the kustomization leaves it in the cluster. Scale to 0 or `delete -k`.
- No native rollback — the stand-in is `kubectl rollout undo` on the affected workload.
- Errors are specific: *no such file or directory* = a path in `resources:` (relative to that kustomization); *unknown field* = a typo in `kustomization.yaml`.

## See also

- `11-kustomize/` — chapters 01-10 (problem statement → components), 11 (generators), 12 (exam patterns and traps)
- `helm.md` — the templating alternative
