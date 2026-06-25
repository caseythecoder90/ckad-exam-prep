---
section: "11-kustomize"
chapter: "02"
title: "Kustomize vs Helm"
course_ref: "Mumshad Mannambeth CKAD — Kustomize vs Helm"
examinable: partial         # Kustomize examinable; Helm = ecosystem awareness only
kind: "conceptual"
companion_diagrams:
  - "diagrams/04-kustomize-vs-helm.png"
  - "diagrams/05-helm-templating-values.png"
---

# Kustomize vs Helm

Helm is the *other* answer to the same question from ch.01 — "how do I customise
manifests per environment without copy-pasting?" Same goal, fundamentally
different mechanism. This chapter is orientation, not a Helm tutorial: know how
the two differ and why a team picks one, because that's the level the exam (and a
design review) cares about.

> **CKAD scope:** Kustomize is the examinable tool. Helm shows up only as general
> ecosystem awareness — you won't be asked to author a chart in the exam. Learn
> the *conceptual* difference here; don't rat-hole on Go-template syntax for the
> cert.

---

## 1. The one-line difference

- **Kustomize — patch model.** Start from a real, valid manifest (the base) and
  apply small YAML patches per environment. Everything stays valid YAML the whole
  way through. *Merge.*
- **Helm — substitute model.** Write a *template* full of placeholders, supply the
  values separately, and render the two together into a final manifest. The
  template is **not** valid YAML until rendered. *Substitute.*

![Side-by-side: Kustomize patches a valid base while Helm substitutes values into a template; both produce the same replicas:5 result via opposite mechanisms](./diagrams/04-kustomize-vs-helm.png)

---

## 2. How Helm actually works

Helm uses **Go templates** to assign variables to properties. Inside a template
you reference values with `{{ ... }}`:

```yaml
# templates/deployment.yaml  (a Helm chart template — NOT valid YAML on its own)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "nginx:{{ .Values.image.tag }}"
```

The placeholders pull from a **`values.yaml`** at render time:

```yaml
# values.yaml  (the variables)
name: nginx
replicaCount: 1
image:
  tag: "1.27"
```

![Helm template with {{ .Values.* }} placeholders, a values.yaml supplying each one, and the rendered concrete manifest as output](./diagrams/05-helm-templating-values.png)

Per-environment customisation in Helm = **swap the values file** (or override
keys), the template is untouched:

```bash
helm install web ./mychart -f environments/values.dev.yaml
helm install web ./mychart -f environments/values.prod.yaml
helm install web ./mychart --set replicaCount=5      # ad-hoc override
helm template web ./mychart -f values.prod.yaml       # render only, no apply
```

Built-in objects you'll see in templates: `.Values` (your values file),
`.Chart` (chart metadata), `.Release` (release name/namespace/revision),
`.Capabilities`, `.Files`.

### Helm folder layout

The instructor's simplified picture: one set of `templates/` plus a values file
per environment.

```
k8s/
├── environments/
│   ├── values.dev.yaml      # per-env variables
│   ├── values.stg.yaml
│   └── values.prod.yaml
└── templates/               # the templated manifests (shared)
    ├── nginx-deployment.yaml
    ├── nginx-service.yaml
    ├── db-deployment.yaml
    └── db-service.yaml
```

A *real* Helm chart is a bit more than that — worth knowing the canonical shape:

```
mychart/
├── Chart.yaml        # chart metadata: name, version, appVersion
├── values.yaml       # default values (overridden by -f / --set)
├── templates/        # templated manifests + _helpers.tpl (named templates)
└── charts/           # vendored subchart dependencies
```

> Helm 3 note: there is **no Tiller** (the old in-cluster server component was
> removed in v3). Release state is stored as **Secrets in the release's
> namespace**. If you hit old tutorials talking about `helm init`/Tiller, that's
> Helm 2 — ignore it.

---

## 3. Helm is more than per-env config

The lecture's key framing: Helm isn't *just* a per-environment customiser — it's
a **package manager** for Kubernetes apps. That's the real reason teams adopt it:

- **Packaging & distribution** — a chart is a versioned, shareable unit. Pull
  charts from repos / OCI registries (`helm repo add`, `helm pull`). This is how
  you install third-party software (cert-manager, ingress-nginx, Prometheus).
- **Release lifecycle** — Helm tracks what it installed as a *release* with a
  revision history: `helm upgrade`, `helm rollback`, `helm history`, `helm list`.
  Kustomize has none of this — `apply -k` is fire-and-forget; rollback is "go
  re-apply the old YAML / `kubectl rollout undo`."
- **Logic** — conditionals (`{{ if }}`), loops (`{{ range }}`), functions and
  pipelines (`{{ .Values.x | default "y" | quote }}`), named templates
  (`define`/`include`), and **hooks** (pre/post-install/upgrade jobs via
  annotations). Kustomize is deliberately logic-free.

The cost of that power (also from the lecture): **templates aren't valid YAML**,
so your editor, `kubeval`, and `git diff` see template soup until you render with
`helm template`. **Complex charts become genuinely hard to read** — whitespace
and indentation in Go templates are their own special misery (`{{- -}}` trimming,
anyone). Debug with `helm template --debug` and `helm install --dry-run`.

---

## 4. Head-to-head

| Dimension | Kustomize | Helm |
|---|---|---|
| Mechanism | overlay / patch (merge) | template + values (substitute) |
| Artifacts | always valid YAML | templates invalid until rendered |
| Learning curve | low — YAML only | higher — Go templates |
| Install | built into `kubectl` (`-k`) | separate binary |
| Packaging / sharing | none native | charts, repos, OCI registries |
| Release lifecycle | none (just `apply`) | install / upgrade / **rollback** / history |
| Logic (if / loops / fns) | no (by design) | yes |
| Best fit | in-house manifests, small per-env deltas | distributable apps, heavy parameterization, 3rd-party software |

**It's not either/or.** Extremely common pattern: **Helm for third-party charts**
(you don't want to hand-maintain cert-manager's manifests) and **Kustomize for
your own services**. They even compose — Kustomize has a `helmCharts` inflation
generator that can render a chart and then patch it, so you can apply org policy
(labels, NetworkPolicies, registry rewrites) over a vendor chart without forking
it. Don't reach for that on day one; just know the two aren't enemies.

---

## 5. Three ways to inject values (the mental model that ties it together)

This is the useful generalisation. "Customise per environment" decomposes into
three distinct techniques, and your work setup uses more than one:

1. **Structural overlay / patch** *(Kustomize)* — merge YAML fragments onto a base.
   Operates on YAML structure. No string templating, no logic.
2. **Template substitution** *(Helm)* — a templating engine evaluates placeholders
   against a values tree, *with* logic (conditionals/loops/functions).
3. **Variable injection** *(envsubst / CI systems)* — plain `${VAR}` string
   substitution from key/value pairs at build/pipeline time. Simpler than Helm
   (no engine, no logic), and it can sit on top of plain *or* Kustomized YAML.

These layer. You can have a Kustomize base+overlay whose values are filled in by
a pipeline's variable injection — which is exactly your GKP setup below.

---

## 6. JPMC / GKP grounding

What you described maps cleanly onto the model above. On **GKP (Gaia Kubernetes
Platform)** — JPMC's in-house private-cloud Kubernetes — your team uses
**Kustomize** for structural per-env config (technique #1), *and* **Jules**
(your Jenkins-based pipeline layer) reads key/value pairs from `jules.yml` and
**injects them as variables** into the resource definitions at pipeline time
(technique #3).

So you're effectively getting "both worlds" without adopting Helm's templating
engine: Kustomize handles the structural deltas (different resources/patches per
env), while Jules does the value substitution that Helm would otherwise do via
`{{ .Values.* }}`. The substitution happens in CI rather than at `helm install`
time, and it's plain key/value injection — no Go-template logic — which keeps the
manifests reviewable as (near-)plain YAML in PRs.

This is almost certainly GKP-specific platform glue rather than a stock OSS tool;
the *pattern* (CI-time variable injection over Kustomize) is common, the
`jules.yml` plumbing is JPMC's. If you drop the screenshots you mentioned, I'll
fold a concrete "how Jules + Kustomize compose on GKP" worked example into this
section — a real `jules.yml` → injected → rendered example would make this
chapter's mental model land hard, and it's the kind of thing that's genuinely
useful to have written down for your own team.

> CKAD caveat: none of the GKP/Jules specifics are on the exam. The exam wants
> Kustomize technique #1. The work context is here to make the concept stick.

---

## TL;DR

- Same problem as ch.01, opposite mechanism: Kustomize **patches** a valid base;
  Helm **substitutes** values into a template.
- Helm = Go templates + `values.yaml`, *and* a package manager with releases,
  rollback, conditionals/loops/hooks.
- Price of Helm's power: templates aren't valid YAML; complex charts get unreadable.
- Use Kustomize for in-house manifests/small deltas; Helm for distributable apps
  and third-party software. They coexist.
- General model: overlay/patch (Kustomize) · template substitution (Helm) ·
  variable injection (envsubst / CI). Your GKP setup = Kustomize + Jules injection.

## Quick recall

- [ ] Kustomize mechanism vs Helm mechanism? → patch/merge vs template/substitute.
- [ ] Why aren't Helm templates valid YAML? → Go-template `{{ }}` placeholders.
- [ ] Where do Helm's values come from? → `values.yaml` (+ `-f` / `--set` overrides).
- [ ] Two things Helm gives that Kustomize doesn't? → packaging + release rollback (also logic/hooks).
- [ ] What replaced Tiller in Helm 3? → nothing; release state lives in namespace Secrets.
- [ ] Is Helm examinable on CKAD? → no hands-on; Kustomize is the examinable tool.
- [ ] Render a chart without applying? → `helm template ./chart -f values.yaml`.

## Resolved threads

- *Why does ch.01's "advantages over Helm" claim Helm is harder?* → Go-template
  syntax + invalid-until-rendered YAML + chart complexity, detailed here.

## Open threads (carried into ch.03+)

- [ ] `kustomization.yaml` syntax: `resources:`, overlay → base reference, the
      first real hands-on (this is where ch.03 starts).
- [ ] Patch mechanisms: strategic-merge vs JSON6902 vs `replicas:`/`images:`/
      `namePrefix:`/`commonLabels:` transformers.
- [ ] ConfigMap/Secret generators + hash-suffix behaviour.
- [ ] (Optional, work) Worked `jules.yml` + Kustomize injection example on GKP —
      pending your screenshots.
