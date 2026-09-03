---
section: 11-kustomize
chapter: 11
title: "Generators — ConfigMaps and Secrets from files and literals"
examinable: true
kind: notes
related:
  - 05-transformers.md
  - 09-overlays.md
  - ../02-configuration/04-configmaps.md
  - ../02-configuration/05-secrets.md
---

# Generators — ConfigMaps and Secrets from files and literals

Ch03 listed `configMapGenerator` / `secretGenerator` in the transformer table
and moved on. They deserve their own page: "generate a ConfigMap from this
file in the overlay" is exactly the kind of Kustomize task the exam can pose,
and the hash suffix they add surprises everyone once.

---

## 1. What they do

A generator turns inputs listed in `kustomization.yaml` into a ConfigMap or
Secret at build time — the declarative version of `kubectl create configmap
--from-file/--from-literal/--from-env-file`:

```yaml
configMapGenerator:
  - name: app-config
    files:
      - app.properties            # key = file name, value = file content
      - settings.json=config.json # key = settings.json, value = content of config.json
    literals:
      - LOG_LEVEL=warn
    envs:
      - app.env                   # one key per KEY=VALUE line

secretGenerator:
  - name: db-creds
    literals:
      - username=lapis
      - password=ruby-lapis-42
    type: Opaque                  # default; kubernetes.io/tls etc. also possible
```

| Input | `kubectl create` equivalent | Key |
|---|---|---|
| `files: [f]` | `--from-file=f` | the file name |
| `files: [k=f]` | `--from-file=k=f` | `k` |
| `literals: [K=V]` | `--from-literal=K=V` | `K` |
| `envs: [f]` | `--from-env-file=f` | each `K` in the file |

File paths are relative to the kustomization that declares the generator,
so a file "in the overlay directory" is just its name. Secret values are
base64-encoded for you.

---

## 2. The hash suffix, and why references still work

Generated objects get a content hash appended to their name:

```
app-config-6ck5t9h8f2
db-creds-2m4g7hb5kd
```

In the same build, Kustomize rewrites every reference to the plain name —
`configMapRef`, `secretKeyRef`, `envFrom`, `volumes[].configMap.name`,
`volumes[].secret.secretName` — to the hashed name. So a base Deployment
that says `configMap: {name: app-config}` ends up pointing at
`app-config-6ck5t9h8f2` in the rendered output.

That is the feature: change a file, rebuild, and the name changes, so the
Deployment's Pod template changes, so a rollout happens. Plain ConfigMap
edits never trigger rollouts; generated ones do. Old hashed objects are not
deleted by `apply -k` (nothing is pruned), which is harmless but visible in
`kubectl get cm`.

---

## 3. Turning the suffix off

When something outside the build reads the object by a fixed name:

```yaml
secretGenerator:
  - name: db-creds
    literals: [username=lapis, password=ruby-lapis-42]
    options:
      disableNameSuffixHash: true      # this generator only
```

```yaml
generatorOptions:
  disableNameSuffixHash: true          # every generator in this kustomization
  labels:
    generated-by: kustomize
```

Per-generator `options:` and top-level `generatorOptions:` take the same
keys (`disableNameSuffixHash`, `labels`, `annotations`, `immutable`).

---

## 4. Overriding a base's generator from an overlay

`behavior` decides what happens when an overlay declares a generator with
the same name as one in the base:

```yaml
configMapGenerator:
  - name: app-config
    behavior: merge          # add/override keys, keep the rest   (or: replace)
    literals:
      - LOG_LEVEL=debug
```

`create` (the default) errors if the name already exists in the build;
`merge` and `replace` require that it does.

---

## 5. Worked example

Base Deployment referencing both objects by plain name:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: {name: db-creds, key: password}
          volumeMounts:
            - {name: config, mountPath: /etc/app-config}
      volumes:
        - name: config
          configMap: {name: app-config}
```

Overlay:

```yaml
resources:
  - ../../base
configMapGenerator:
  - name: app-config
    files: [app.properties]
    literals: [LOG_LEVEL=warn]
secretGenerator:
  - name: db-creds
    literals: [username=lapis, password=ruby-lapis-42]
    options: {disableNameSuffixHash: true}
```

```bash
kubectl kustomize overlays/prod | grep -E "^  name:|configMap:|secretKeyRef" 
kubectl apply -k overlays/prod
kubectl -n <ns> get cm,secret
```

The rendered Deployment references `app-config-<hash>` and `db-creds`.

---

## 6. Exam-pattern gotchas

- **The file must sit next to the kustomization** (or be given by relative
  path). A missing file is a build error, not an empty ConfigMap.
- **Key = file name** unless you write `key=file`. "Under key `index.html`"
  means `files: [index.html=/path/to/file]`, the same `key=` trap as
  `kubectl create cm --from-file`.
- **Hash suffix on, references rewritten.** Do not hard-code the hashed name
  anywhere; do not "fix" the base to use the hashed name.
- **`disableNameSuffixHash` lives under `options:`** for one generator or
  under `generatorOptions:` for all — not at the generator's top level.
- **Literals are `KEY=VALUE` strings**, one per list item. Quoting values
  with spaces: `- "MOTD=hello world"`.

## References

- [configMapGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/configmapgenerator/) — `files`, `literals`, `envs`, `behavior`, `options`
- [secretGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/secretgenerator/) — the same inputs plus `type`
- [generatorOptions](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/generatoroptions/) — `disableNameSuffixHash` and friends, globally
- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) — generators in the kubernetes.io task page, including the hash-suffix behaviour
