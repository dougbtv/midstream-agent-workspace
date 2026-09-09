# vLLM midstream and upstream work

This project area covers upstream vLLM contribution, downstream synchronization,
build automation, CI infrastructure, performance/evaluation, model validation,
the dashboard, and accelerator integrations.

## Primary repositories

- `repos/vllm/nm-vllm-ent`: downstream/midstream vLLM engine
- `repos/vllm/nm-cicd`: midstream build and validation automation
- `repos/vllm/ci-infra`: upstream CI infrastructure
- `repos/vllm/perf-eval`: performance and evaluation workloads
- `repos/vllm/vllm-dashboard`: results UI
- `repos/vllm/model-validation-configs`: model test configuration
- `repos/vllm/vllm-gaudi`: Gaudi integration

## Context

- `ci-performance-and-evaluation.md`: data flow, repository responsibilities,
  and current evaluation direction
- `ci-speed-optimization.md`: time-to-signal research and planning snapshot

Reusable sync, release, PR, ROCm, and blocker procedures live under `skills/`.
Treat dates, branch names, job IDs, image tags, and ticket state in imported
documents as snapshots that require live verification.
