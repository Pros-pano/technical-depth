# Production‑Ready MLOps for Kubernetes AI Infra

This guide outlines a **CI/CD pipeline** that builds, tests, and deploys AI workloads on the DGX Spark Kubernetes clusters.

## Core Topics
- **GitOps** using Argo CD / Tekton – automated manifests validation, linting (`kube‑val`), and progressive roll‑outs.
- **Security** – RBAC templating, OPA/Gatekeeper policies, secret management with Vault.
- **Testing** – unit tests for Helm charts, integration tests with Kind clusters, performance regression suites.
- **Observability** – Prometheus exporter for pod‑level GPU metrics, Grafana dashboards, alerting on latency / OOM.
- **Rollback** – idempotent manifests, canary releases, health‑check hooks, automated revert on failure.
- **Packaging** – building reusable Helm charts, versioned container images, and publishing to an internal chart museum.

*This file is a scaffold; detailed steps will be added later.*
