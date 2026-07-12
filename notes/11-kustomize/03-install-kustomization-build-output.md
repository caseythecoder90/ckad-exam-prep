---
section: "11-kustomize"
chapter: "03"
title: "Install, kustomization.yaml, Build & Output"
course_ref: "Mumshad Mannambeth CKAD — Install + kustomization.yaml + Output (3 short lectures merged)"
examinable: true
kind: "hands-on"
note: "Three short lectures folded into one (installation / the kustomization.yaml file / build + output), following the Helm-chapter merge precedent. Next chapter is 04."
companion_diagrams:
  - "diagrams/06-build-output-apply-delete.png"
  - "diagrams/07-commonlabels-transformer.png"
---

# Install, kustomization.yaml, Build & Output

Installing the standalone CLI, the anatomy of `kustomization.yaml`, and how
`build` produces output you then apply or delete.

---

## 1. Installation (the 60-second version)

**Prereqs:** a running cluster + `kubectl` on your machine. "Cluster up and
running" really means *reachable from this machine* — the control plane and nodes
live wherever they live; you just need kubeconfig access. Kustomize itself is a
client-side binary; it does nothing on the cluster, it only renders YAML locally.

**Install** (Linux/macOS/Windows) — the team ships a script that detects your OS
and grabs the right binary:

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
```

**Validate:**

```bash
kustomize version
```

Other install paths worth knowing (the script isn't the only way):

```bash
brew install kustomize                 # macOS / Linuxbrew
go install sigs.k8s.io/kustomize/kustomize/v5@latest
# or just download the pinned release binary from the GitHub releases page
```

> Recall from ch.01: `kubectl` already has kustomize **built in** (`kubectl apply
> -k`, `kubectl kustomize`). You only install the standalone CLI to get a
> **newer** version than the one vendored into your `kubectl`. If your tasks only
> use `-k`, you may not need the separate binary at all.

> Don't `curl | bash` in prod. Piping a `master`-branch script straight into a
> shell executes whatever's there today, unpinned and unverified. Fine for a lab;
> in a regulated environment you'd pull a **version-pinned, checksum-verified**
> binary from an approved internal mirror.

---

## 2. The `kustomization.yaml` file

Kustomize looks for a file named **exactly** `kustomization.yaml` (also accepts
`kustomization.yml` or `Kustomization`). **You write it yourself** — it's the
entry point from ch.01, and it does two jobs:

1. **`resources:`** — the list of manifests Kustomize should manage.
2. **customizations / transformers** — what to change across those resources
   (the instructor keeps it to one: a `commonLabels` applied to everything).

```yaml
# k8s/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1   # optional but good practice (see §6)
kind: Kustomization

# 1) kubernetes resources to be managed by kustomize
resources:
  - nginx-depl.yaml
  - nginx-service.yaml

# 2) customizations that should be applied to all of them
commonLabels:
  company: KodeKloud
```

Folder for that example:

```
k8s/
├── nginx-depl.yaml
├── nginx-service.yaml
└── kustomization.yaml
```

---

## 3. What `commonLabels` actually does (it's more than you think)

One `commonLabels` entry doesn't just add a label to each resource's
`metadata.labels`. It fans **into every label site**: resource labels, **the
Service selector**, **the Deployment `selector.matchLabels`**, *and* the **pod
template labels** — in every managed resource.

![commonLabels: company: KodeKloud injected into 5 separate label sites across a rendered Service and Deployment](./diagrams/07-commonlabels-transformer.png)

That's the mirror image of ch.01's fan-out *problem*: here the fan-out is the
*feature* — declare once, applied everywhere consistently.

> **Gotcha worth more than the lecture gives it:** because `commonLabels` rewrites
> **selectors**, and a Deployment's `spec.selector` is **immutable after
> creation**, applying a new `commonLabels` to an *already-running* Deployment is
> rejected: `field is immutable`. It's painless on first create, painful as a
> retrofit.
>
> **Modern alternative (flag the deprecation drift):** newer kustomize prefers
> the list-form `labels:` transformer, which by default **does not touch
> selectors**:
> ```yaml
> labels:
>   - pairs:
>       company: KodeKloud
>     includeSelectors: false      # default false -> safe on live Deployments
>     includeTemplates: true
> ```
> Mumshad teaches `commonLabels` (still valid, still likely what the exam shows).
> Know it for the cert; reach for `labels:` in real life when you don't want to
> rewrite immutable selectors. (`commonAnnotations` is the annotation-side twin
> and has no selector hazard.)

Other transformers you'll meet immediately (all live in `kustomization.yaml`):

| Field | Effect |
|---|---|
| `namespace: foo` | sets `metadata.namespace` on every resource |
| `namePrefix:` / `nameSuffix:` | prepends/appends to every resource name |
| `commonLabels:` / `commonAnnotations:` | labels/annotations across all (labels also hit selectors) |
| `images:` | rewrite image `name`/`newName`/`newTag`/`digest` |
| `configMapGenerator:` / `secretGenerator:` | generate ConfigMaps/Secrets (with a content-hash name suffix) |

---

## 4. Build & output: render first, deploy second

`kustomize build k8s/` reads the `kustomization.yaml`, **combines** all the listed
manifests, **applies the transformations**, and prints the **final YAML to
stdout**. Critically, **`build` does not deploy anything.** It's a pure render.

![kustomize build renders to stdout and never touches the cluster; pipe to kubectl apply -f - to create or kubectl delete -f - to remove, or use the kubectl -k one-step shortcut](./diagrams/06-build-output-apply-delete.png)

To actually act on the cluster, **redirect the output into kubectl** — output of
one command becomes input of the next (`-` = read from stdin):

```bash
kustomize build k8s/ | kubectl apply -f -      # render, then create/update
kustomize build k8s/ | kubectl delete -f -     # render, then delete (same pattern)
```

Deleting works exactly the same way — just pipe into `delete` instead of `apply`.

**One-step equivalents** using kubectl's built-in kustomize (no separate
`build`):

```bash
kubectl apply  -k k8s/
kubectl delete -k k8s/
kubectl kustomize k8s/        # = "kustomize build k8s/" via kubectl (render only)
```

---

## 5. Imperative / exam shortcuts

```bash
# render & inspect (do this constantly before applying)
kubectl kustomize <dir>            # or: kustomize build <dir>

# apply / delete
kubectl apply  -k <dir>
kubectl delete -k <dir>

# edit kustomization.yaml fast without opening an editor (big exam time-saver)
kustomize edit set image nginx=nginx:1.27
kustomize edit set namespace dev
kustomize edit add resource service.yaml
kustomize edit set replicas nginx-deployment=5

# scaffold a kustomization.yaml from the manifests already in a dir
kustomize create --autodetect --recursive
```

`kustomize edit ...` mutates the `kustomization.yaml` in the current directory in
place — faster and less error-prone than hand-editing under time pressure.

---

## 6. apiVersion & kind

You **can** (and in stricter tooling, should) put a header on the file:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```

It's optional for plain `kustomize build` — kustomize infers it — but it makes the
file self-describing, satisfies schema validators/linters, and is what `kustomize
create` writes for you. Cheap to include; include it.

## References

- [Kustomize — Install Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) — official install methods for the standalone CLI
- [The Kustomization File](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) — the fields that go in `kustomization.yaml` (resources, transformers, generators)
- [kustomize build](https://kubectl.docs.kubernetes.io/references/kustomize/cmd/build/) — how `build` hydrates a kustomization into rendered manifests on stdout
