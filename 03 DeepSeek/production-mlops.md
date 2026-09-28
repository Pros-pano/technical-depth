# Production‑Ready MLOps for DeepSeek

This scaffold outlines a CI/CD and ops framework for the DeepSeek curriculum:

- **GitOps** – Argo CD pipelines that render `kustomize`/Helm manifests for DeepSeek serving.
- **Secret‑Ops** – Vault‑backed API‑key injection for HuggingFace, OpenAI, and internal data sources.
- **Testing** – Molecule‑style role tests, integration tests on a local Kind cluster, and performance regression suites.
- **Observability** – Export vLLM/DeepSeek metrics to Prometheus, Grafana dashboards for token latency, KV‑cache health, and GPU utilisation.
- **Rollback** – Canary deployments, health‑probe gating, and automatic rollback on failure.
- **Packaging** – Build reusable Ansible collections and Helm charts for model weight caching, NVMe storage, and inference pods.

*Detailed chapters will be added progressively.*
