# vLLM-Omni Midstream Build — Status Update

**Date:** 2026-04-29 | **Author:** Doug Smith | **Epic:** [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288)

---

## TL;DR

We have a first-pass build pipeline for vLLM-Omni working end-to-end through the midstream infrastructure. The image is built and pushed, but the pipeline code is still prototype-quality — we're working towards cleaning it up for team review.

```
quay.io/vllm/automation-vllm-omni:cuda-24846094460
```

---

## What is nm-cicd?

[nm-cicd](https://github.com/neuralmagic/nm-cicd) is our GitHub Actions-based CI/CD system for producing midstream vLLM releases. It builds wheels, assembles UBI9-based container images, and runs model validation — all with Red Hat-focused tooling to prepare for AIPCC compatibility and managed upstream fork lifecycle. This is how we already ship standard vLLM (nm-vllm-ent). We have a first-pass extension of this pipeline for vLLM-Omni on a working branch — not yet merged to main.

---

## Architecture: Two-Wheel Build

vLLM-Omni doesn't replace vLLM — it layers on top. The build uses a two-wheel strategy:

```
 Step 1    Build vLLM wheel (nm-vllm-ent)              213 MB, compiled CUDA/C++
              |
 Step 2    Build vllm-omni wheel (nm-vllm-omni-ent)    1.5 MB, pure Python
              |
 Step 3    Build Docker image — installs BOTH wheels    UBI9-minimal + CUDA 12.9
```

This keeps standard vLLM untouched — omni is purely additive. The image reuses pre-built wheels, so Dockerfile iteration doesn't require rebuilding anything.

---

## First Build Results

| Step | Status | Run |
|------|--------|-----|
| vLLM wheel | **SUCCESS** | [24809747786](https://github.com/neuralmagic/nm-cicd/actions/runs/24809747786) |
| vllm-omni wheel | **SUCCESS** | [24811068149](https://github.com/neuralmagic/nm-cicd/actions/runs/24811068149) |
| Docker image | **SUCCESS** | [24846094460](https://github.com/neuralmagic/nm-cicd/actions/runs/24846094460) |

**Image contents:**
- vllm: `0.18.1.dev37+rhaiv.2`
- vllm-omni: `0.1.dev1067+g2a36e235c`
- Base: UBI 9 minimal, CUDA 12.9, Python 3.12

---

## Repositories

### [neuralmagic/nm-cicd](https://github.com/neuralmagic/nm-cicd) — Build Pipeline

Branch: [`feature/vllm-omni`](https://github.com/neuralmagic/nm-cicd/tree/feature/vllm-omni)

GitHub Actions workflows for wheel builds, image builds, and model validation. The vllm-omni additions (on a feature branch) plug into the existing `build-whl.yml` and `build-image.yml` workflows via dynamic action selection — no forked workflows.

> **Note:** The original branch was `vllm-omni-build` — this was renamed to `feature/vllm-omni`. The `midstream-build.yml` workflow default and all trigger commands should use `feature/vllm-omni`. If you pass `nm_cicd_ref` explicitly, use `feature/vllm-omni`.

### [neuralmagic/nm-vllm-omni-ent](https://github.com/neuralmagic/nm-vllm-omni-ent) — Midstream Fork

Branch: [`midstream-main`](https://github.com/neuralmagic/nm-vllm-omni-ent/tree/midstream-main)

Carries UBI9 Dockerfiles (CUDA, ROCm, CPU) and build config (`docker-bake.hcl`) on top of upstream vllm-omni. Minimal carry — currently 2 commits on top of upstream v0.18.0.

---

## JIRA

| Issue | Title | Purpose |
|-------|-------|---------|
| [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288) | vLLM-Omni Midstream: Initial Build | Epic — build, test, packaging scope |
| [INFERENG-6279](https://redhat.atlassian.net/browse/INFERENG-6279) | First pass build pipeline | Story — the work tracked here |
| [RHAISTRAT-1242](https://redhat.atlassian.net/browse/RHAISTRAT-1242) | vLLM-Omni Tracker | Initiative — cross-team tracker |

---

## Upstream Activity

Alex Brooks has two PRs that clean up the vLLM/vllm-omni install story:

- [vllm#40744](https://github.com/vllm-project/vllm/pull/40744) — vLLM CLI natively delegates to omni when `--omni` is passed
- [vllm-omni#3082](https://github.com/vllm-project/vllm-omni/pull/3082) — removes entrypoint hijacking from vllm-omni

Once merged, install order between vLLM and vllm-omni will no longer matter.

---

## How to Bump the vLLM Wheel Version

When upstream vllm-omni syncs and requires a newer vLLM, you need to update **two files** in nm-vllm-omni-ent — missing either one will silently use the old wheel:

### 1. Build the vLLM wheel first (nm-cicd)

Trigger a `build whl image` run in nm-cicd for the new vLLM version. Note the **run ID** when it succeeds.

### 2. Update `midstream/vllm-wheels.yml` — add the mapping

Add a new entry at the top with the run ID from step 1:

```yaml
v0.23.0:
  run_id: "27551614162"
  branch: "doug/v0.23.0"
  note: "v0.23.0 wheel, built 2026-06-15 for INFERENG-7911"
```

### 3. Update `midstream/vllm-version` — bump the pointer

This file tells the resolve step which version to look up in the mapping:

```
v0.23.0
```

**Both files must be updated together.** The resolve step reads `vllm-version`, then looks it up in `vllm-wheels.yml`. If you add a new entry to the YAML but forget to bump the version file, the build will silently use the old wheel.

### 4. PR, build, verify

```bash
# Branch and commit both changes
git checkout -b doug/bump-vllm-wheels-v0.X.Y
# ... edit both files ...
git add midstream/vllm-wheels.yml midstream/vllm-version
git commit -m "Bump vllm-wheels to v0.X.Y"
git push -u origin doug/bump-vllm-wheels-v0.X.Y

# Create PR
gh pr create --repo neuralmagic/nm-vllm-omni-ent --base main ...

# Trigger full-chain build from the branch
gh workflow run "Midstream Build" \
  --repo neuralmagic/nm-vllm-omni-ent \
  --ref doug/bump-vllm-wheels-v0.X.Y \
  -f build_step=full-chain \
  -f nm_cicd_ref=feature/vllm-omni
```

The build will resolve the wheel, build the omni wheel, build the Docker image, and run a smoke test.

### Gotcha log

| Issue | Symptom | Fix |
|-------|---------|-----|
| Forgot to bump `vllm-version` | Image silently uses old wheel, version mismatch ImportError at startup | Update both files together |
| `curand.h` missing | FlashInfer JIT fails on startup (v0.23.0+) | Need `libcurand-devel` in Dockerfile — currently only in `cuda-develsdk` variant |
| Smoke test timeout | Pod stuck in ContainerCreating on cold image pull | Retry — image caches on node after first pull; consider bumping `health_timeout` in nm-cicd `test-vllm-omni` action (default 600s) |
| Wrong nm-cicd branch | Workflows dispatch but use stale code | Use `feature/vllm-omni`, NOT `vllm-omni-build` (renamed) |

---

## What's Next

- **Cleanup pass** on nm-cicd branch — get it review-ready before proposing merge to main
- **Repo access grants** for collaborators (list being compiled)
- **Smoke testing** the image with actual models (image gen, TTS, video gen) — image is built but not yet validated with real workloads
- **Onboarding docs** so the team can trigger builds independently
- **ROCm / CPU** image builds (CUDA-only so far)

---

## Status

This is **dev-preview-oriented work**. We are not yet assessing tech preview readiness — that depends on upstream model support maturity, multimodal testing coverage, and software lifecycle considerations.
