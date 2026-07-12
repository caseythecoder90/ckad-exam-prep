# Examples

Runnable Kubernetes manifests that accompany the notes in [`../notes/`](../notes/). Apply them against a local cluster (kind/minikube) to see a concept work end to end.

```bash
kubectl apply -f <file>.yaml
```

This folder is growing; the goal is a runnable manifest for every concept the notes cover, cross-linked from the relevant chapter.

## Contents

- `pod-definition.yml` — minimal Pod manifest.
