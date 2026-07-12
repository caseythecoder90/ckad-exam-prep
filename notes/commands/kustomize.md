# Kustomize

Template-free config management: a **base** of plain YAML + **overlays** that patch it per environment. Built into kubectl via `-k`. The one rule to remember: **`-k` runs the base+overlay merge; `-f` applies raw files** (and chokes on `kustomization.yaml`).

## Apply / render / delete

```bash
k apply -k <dir>                          # build + apply (built-in Kustomize)
k delete -k <dir>                         # build + delete
k kustomize <dir>                         # render merged output to stdout — no apply (sanity check)

# Standalone CLI equivalents
kustomize build <dir>                     # render to stdout
kustomize build <dir> | kubectl apply -f -
kustomize build <dir> | kubectl delete -f -
kustomize version
```

Always `k kustomize <dir>` (or `kustomize build`) to eyeball output **before** applying.

## Scaffold & edit `kustomization.yaml` in place

```bash
kustomize create --autodetect --recursive     # generate kustomization.yaml from existing manifests
kustomize edit add resource service.yaml       # add to resources: (a file OR a directory)
kustomize edit add resource ../../base         # overlay references the base
kustomize edit add component ../../components/db
kustomize edit add patch --path patch.yaml --group apps --version v1 --kind Deployment --name api
```

## Transformers via `kustomize edit set`

```bash
kustomize edit set image nginx=haproxy:2.4     # newName + newTag
kustomize edit set image nginx=*:2.4           # change tag only
kustomize edit set namespace dev
kustomize edit set nameprefix KodeKloud-
kustomize edit set namesuffix -dev
kustomize edit set label org:KodeKloud
kustomize edit set annotation branch:master
kustomize edit set replicas nginx-deployment=5
```

## Mental model

- `resources:` — base YAML files and/or directories (nested kustomizations recurse).
- Overlay = its own `kustomization.yaml` referencing the base + env-only resources + patches.
- Patches: **strategic merge** (partial YAML, matched by name) or **JSON6902** (`op`/`path`/`value`). Lists: JSON6902 uses positional `/0`, `/-`; strategic merge is keyed.
- No native rollback — the stand-in is `kubectl rollout undo` on the affected workload.

## Install (standalone CLI)

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
# or: brew install kustomize  /  go install sigs.k8s.io/kustomize/kustomize/v5@latest
```

## See also

- `11-kustomize/` — chapters 01-10 (problem statement → components)
- `helm.md` — the templating alternative
