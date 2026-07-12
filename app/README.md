# app (planned)

A real Java (Spring Boot) application deployed on Kubernetes end to end — the single worked example that ties the CKAD curriculum together.

Planned scope:

- Container image build (multi-stage Dockerfile) and a pinned image tag.
- Configuration via ConfigMap and Secret, injected as env vars and mounted files.
- Readiness, liveness, and startup probes on the app's actuator endpoints.
- A Service and an Ingress in front of it.
- Resource requests/limits and a HorizontalPodAutoscaler.
- Packaging with both Helm and Kustomize (base + per-environment overlays).

Nothing is here yet — this directory is a placeholder for that build. See the Roadmap in the [top-level README](../README.md).
