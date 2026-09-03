# CKAD Study Guide

Comprehensive, exam-focused notes for the **Certified Kubernetes Application Developer (CKAD)** exam — built from working through the full curriculum, expanded with the mechanics, gotchas, and real-world context that make each topic stick.

Every chapter keeps only what helps you pass: the concept, worked YAML, the imperative shortcuts, and the traps that show up on the exam. No lecture transcription, no filler.

## Who this is for

- Anyone preparing for the CKAD exam who wants dense, skimmable notes plus a fast command reference.
- Engineers who want a practical refresher on Kubernetes application concepts (config, probes, storage, networking, RBAC, Helm, Kustomize).

You do not need to read it front to back — jump to a section, or `Ctrl+F` the command reference during a lab.

## Repository layout

```
ckad-study-guide/
├── README.md
├── notes/                     # the study notes, one folder per curriculum section
│   ├── 01-core-concepts/      #   each with its own diagrams/ subfolder
│   ├── 02-configuration/
│   ├── 03-multi-container-pods/
│   ├── 04-observability/
│   ├── 05-pod-design/
│   ├── 07-services-and-networking/
│   ├── 08-state-persistance/
│   ├── 09-security/
│   ├── 10-helm-fund/
│   ├── 11-kustomize/
│   ├── commands/              # per-topic command references (study one resource at a time)
│   ├── commands.md            # global single-file command reference (Ctrl+F across everything)
│   ├── exam-wording.md        # question phrasing → YAML field, plus time triage
│   └── NOTES-WORKFLOW.md      # how these notes are authored
├── examples/                  # runnable manifests that accompany the notes (growing)
└── app/                       # planned: a real Java app deployed on Kubernetes (see Roadmap)
```

## Contents

| Section | Covers |
|---|---|
| [01 · Core Concepts](notes/01-core-concepts/) | Architecture, pods, ReplicaSets, Deployments, namespaces, Services, `explain`/output formats |
| [02 · Configuration](notes/02-configuration/) | Images, commands/args, env vars, ConfigMaps, Secrets, security contexts, resources, service accounts, scheduling (taints/affinity) |
| [03 · Multi-Container Pods](notes/03-multi-container-pods/) | Sidecar/adapter/ambassador patterns, init containers |
| [04 · Observability](notes/04-observability/) | Readiness, liveness, and startup probes, logging, metrics-server |
| [05 · Pod Design](notes/05-pod-design/) | Labels/selectors/annotations, rollouts/rollbacks, blue-green, canary, Jobs, CronJobs |
| [07 · Services & Networking](notes/07-services-and-networking/) | Network policies, ingress |
| [08 · State Persistence](notes/08-state-persistance/) | Volumes, PV/PVC, storage classes, StatefulSets, headless services |
| [09 · Security](notes/09-security/) | AuthN/AuthZ, kubeconfig, RBAC, admission control, API versioning/deprecation, CRDs, operators |
| [10 · Helm](notes/10-helm-fund/) | Charts, values, releases, lifecycle commands, exam task patterns and traps |
| [11 · Kustomize](notes/11-kustomize/) | Bases/overlays, transformers, patches, components, generators, exam field cheat-sheet |

## How to use it

1. **Learn a topic** — read its chapter in `notes/<section>/`. Each ends with exam-pattern gotchas.
2. **Drill the commands** — [`notes/commands/`](notes/commands/) has one focused file per resource; [`notes/commands.md`](notes/commands.md) is the single-page version to `Ctrl+F`.
3. **Work imperatively** — start with [`notes/commands/imperative.md`](notes/commands/imperative.md). On the exam, generating YAML with `kubectl ... --dry-run=client -o yaml` and editing it beats hand-writing manifests. Then [`notes/commands/modifying-resources.md`](notes/commands/modifying-resources.md) for the other half of the job: changing resources that already exist (`set` vs `patch` vs `edit` vs export→apply).
4. **Translate the question** — [`notes/exam-wording.md`](notes/exam-wording.md) maps the exam's recurring phrasings ("no Pod may be unavailable", "only runs during start") onto the fields they mean, so a question becomes a lookup instead of a puzzle.
5. **Reference the official docs** — kubernetes.io is allowed during the exam. Each chapter links the canonical pages so you can practice navigating them under time pressure.

## About the CKAD exam

The CKAD is a 2-hour, hands-on, performance-based exam: you solve tasks in a live cluster from a terminal. The official kubernetes.io (and Helm) documentation is available during the exam, so knowing *how to find* a spec quickly matters as much as recall.

Current curriculum domains and weights:

| Domain | Weight |
|---|---|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

Always confirm the current version, duration, and passing score on the official pages:

- CKAD curriculum: https://github.com/cncf/curriculum
- Exam page & candidate handbook: https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/

## Roadmap

- **`examples/`** — runnable manifests for every concept, so notes link straight to something you can `kubectl apply`.
- **`app/`** — a real Java (Spring Boot) application deployed on Kubernetes end to end: image build, ConfigMaps/Secrets, probes, a Service and Ingress, an HPA, and Helm/Kustomize packaging — a single worked example that ties the whole curriculum together.
- **Docs links** on every chapter (in progress) pointing at the canonical kubernetes.io pages.

## Attribution & license

Notes were built while working through a CKAD video course and expanded well beyond it with additional depth. They are original write-ups intended for study and sharing.

This repository is dual-licensed (see [`LICENSE`](LICENSE)):

- **Notes & documentation** (`notes/`, diagrams, this README) — [CC BY 4.0](LICENSE-CC-BY-4.0). Reuse and adapt freely, including commercially, with attribution.
- **Code** (`examples/`, `app/`, scripts, manifests) — [MIT](LICENSE-MIT).
