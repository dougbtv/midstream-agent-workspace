# Field Guide to Midstream vLLM-Omni

A practical reference for anyone working on vLLM-Omni midstream builds, testing, and packaging. Start here to understand what exists, where it lives, and how to use it.

**Last updated:** 2026-06-04

## What Is This

[vLLM-Omni](https://github.com/vllm-project/vllm-omni) extends vLLM for omni-modality inference — audio, video, image generation, TTS, and diffusion models. The midstream effort produces official builds (Python wheels + container images) of vllm-omni through the same infrastructure that ships standard vLLM (nm-vllm-ent).

This is **developer-preview / prototype** work, built on feature branches. The goal is to get a working end-to-end pipeline, learn the rough edges, and use it to inform what a production-grade integration looks like. Some of this may graduate directly into mainline; some may be rewritten. Either way, the patterns and lessons are the point.

## Repositories

### [`neuralmagic/nm-vllm-omni-ent`](https://github.com/neuralmagic/nm-vllm-omni-ent) — The Omni Midstream Fork

The midstream fork of vllm-omni, analogous to how `nm-vllm-ent` is the midstream fork of vLLM. This is where Dockerfiles, carry commits, build tooling, and version mappings live.

**Key branches:**
- **`main`** — current midstream head, synced to upstream v0.21.0
- **`midstream-main`** — legacy name from the first pass, may still be referenced in older docs

**What's in this repo:**
- `Dockerfile.ubi` / `Dockerfile.rocm.ubi` / `Dockerfile.cpu.ubi` — UBI 9 container builds
- `docker-bake.hcl` — Buildx bake config
- `.github/workflows/midstream-build.yml` — the "macro" workflow (see below)
- `midstream/` — midstream-only content, safe from upstream rebases (see below)

**Carry commits** (patches on top of upstream):
- EPEL installation fix (rpm instead of microdnf)
- libsndfile dependency fix (replaces ffmpeg/sox RPMs)
- Two CUDA image variants (jit-cache default + develsdk)
- Upstream test inclusion in container image

### [`neuralmagic/nm-cicd`](https://github.com/neuralmagic/nm-cicd) — The Build Pipeline

The shared CI/CD infrastructure that builds wheels and images for all midstream projects (nm-vllm-ent, nm-vllm-omni-ent, etc.).

**Key branch:**
- **`feature/vllm-omni`** — the prototype branch with all omni build + test additions

This branch adds omni-specific actions, scripts, payload preparation, and test workflows. The standard `build-whl.yml` and `build-image.yml` workflows on this branch use dynamic dispatch (`jenseng/dynamic-uses`) to select omni-specific actions when `CLIENT_REPO` is `nm-vllm-omni-ent`.

### [`neuralmagic/nm-vllm-ent`](https://github.com/neuralmagic/nm-vllm-ent) — Standard vLLM (Base Wheel)

The existing productized vLLM — not omni. vLLM-Omni is built *on top of* an nm-vllm-ent wheel. This repo is not modified for omni; we just consume its wheel as a build dependency.

## Architecture: Two-Wheel Build

This is the key design decision. vLLM-Omni doesn't replace vLLM — it layers on top. The build produces two separate wheels that get installed into a single container image.

```
Step 1: Build vLLM wheel        (nm-vllm-ent)          → GCS
             ↓ run_id
Step 2: Build vllm-omni wheel   (nm-vllm-omni-ent)     → GCS
             ↓ run_id + vllm_run_id
Step 3: Build Docker image      (nm-vllm-omni-ent)     → quay.io
```

### Why Two Wheels

- vllm-omni's `setup.py` imports from vLLM at build time — the vLLM wheel must be installed first
- The final image needs both: `vllm` for core serving, `vllm_omni` for multimodal extensions
- Standard vLLM builds are untouched — omni is purely additive

### Where Artifacts Land

- **Wheels:** `gs://nm-gha-cache/neuralmagic/<repo>/assets/<run_id>/<device>/`
- **Images:** `quay.io/vllm/automation-vllm-omni:<tag>`
  - Tags: `cuda-<run_id>`, `cuda-<commit_sha>`, `<omni_version>`

## The `midstream/` Directory (nm-vllm-omni-ent)

Everything in `midstream/` is ours — it doesn't exist upstream, so it's safe from rebases. This is where midstream-specific config and tooling lives.

```
midstream/
├── README.md                   # You-are-here docs
├── vllm-version                # Single line: "v0.21.0" — which vLLM version this code targets
├── vllm-wheels.yml             # Flat-file wheel index — maps vLLM versions to known-good run IDs
├── test-excludes.txt           # Packages to exclude from test image (UBI 9 glibc compat)
├── install-ffmpeg-shim.py      # Symlinks imageio-ffmpeg's bundled binary into PATH
└── .github-upstream-policy.md  # Rebase rules: which .github/ files are upstream vs midstream
```

### The Flat-File Wheel Index (`vllm-wheels.yml`)

This is a simple YAML mapping of vLLM versions to pre-built wheel run IDs:

```yaml
v0.21.0:
  run_id: "26183985711"
  branch: "main"
  note: "CUDA 13.0 wheel, built 2026-05-20 for INFERENG-7152/7117"

v0.20.0:
  run_id: "25021945246"
  branch: "main"
  note: "reused from deepseek effort, built 2026-05-10"
```

The `midstream-build.yml` workflow reads `vllm-version` to pick the current target version, then looks up the corresponding `run_id` from this file. This means you can trigger a build without remembering (or even knowing) the vLLM wheel run ID — it's resolved automatically.

**When to update:**
- Rebased to a new upstream version → update `vllm-version`, build a new vLLM wheel, add entry to `vllm-wheels.yml`
- Built a better wheel for an existing version → update the `run_id`

## nm-vllm-omni-ent GHA: "Macros" for nm-cicd

The `.github/workflows/midstream-build.yml` workflow in nm-vllm-omni-ent is not a build system — it's a **remote control** for nm-cicd. It dispatches `gh workflow run` calls against nm-cicd's `build-whl.yml` and `build-image.yml` using a cross-repo GitHub PAT (`CICD_OMNI_PAT`).

### Why This Pattern

The build logic lives in nm-cicd, and that's where it should stay — it's a shared pipeline that builds wheels and images for multiple projects. But triggering nm-cicd builds requires repo access, knowledge of the right parameters, runner labels, and run ID chaining.

The macro workflow solves this by:

1. **Encapsulating the nm-cicd interface** — you pick a build step from a dropdown, the workflow fills in the right parameters, runner labels, and version mappings
2. **Enabling the OCTO ET team** — people working on vllm-omni upstream (Emerging Tech, model teams) can trigger midstream builds and see results directly on the nm-vllm-omni-ent repo, without needing nm-cicd access
3. **Setting a boundary** — the Midstream team owns nm-cicd changes; OCTO ET folks interact through the macro surface. We could grant nm-cicd access if needed, but this separation lets us iterate on the pipeline without coordination overhead

### Build Steps

| Step | What It Does | When to Use |
|------|-------------|-------------|
| **full-chain** | Omni wheel → wait → Docker image → smoke test | Default. Tag-push triggers use this automatically |
| **vllm-wheel** | Build base vLLM wheel from nm-vllm-ent | When you need a fresh vLLM wheel for a new version |
| **omni-wheel** | Build vllm-omni wheel only | When iterating on omni code with an existing vLLM wheel |
| **docker-image** | Build container image only | When iterating on Dockerfiles with existing wheels |

### Tag-Based Triggers

Push a tag matching `omni-*` and the full chain runs automatically:

```bash
git tag omni-v0.21.0-rc1
git push origin omni-v0.21.0-rc1
```

The vLLM wheel is resolved from `midstream/vllm-wheels.yml` — no manual run ID needed.

### The `CICD_OMNI_PAT` Secret

An org-level fine-grained GitHub PAT with:
- **Actions:** read/write on `neuralmagic/nm-cicd` (to trigger workflows)
- **Contents:** read on `neuralmagic/nm-cicd` (to poll run status)

This is the only credential that bridges the two repos. It's scoped narrowly — it can dispatch workflows and read their status, nothing more.

## nm-cicd: What's on `feature/vllm-omni`

The feature branch adds omni-specific actions and scripts that plug into the existing `build-whl.yml` and `build-image.yml` shared workflows. The additions are selected dynamically based on `CLIENT_REPO`.

### Wheel Build Files

```
.github/actions/build-nm-vllm-omni-ent/action.yml          # Build action entry point
.github/actions/env-build-nm-vllm-omni-ent/action.yml      # Env setup (device, python, GCS paths)
.github/scripts/build_vllm_omni.sh                         # Main build script
.github/scripts/setup_env_build_vllm_omni.sh               # Env var setup
```

### Image Build Files

```
.github/actions/prepare-payload-nm-vllm-omni-ent/action.yml    # Payload prep action
.github/actions/prepare-payload-nm-vllm-omni-ent/prepare.sh    # Downloads both wheels, generates run.sh
.github/actions/env-build-image-nm-vllm-omni-ent/action.yml    # Image build env vars + device validation
neuralmagic/requirements/vllm-omni-cuda.txt                    # Runtime deps (CUDA)
neuralmagic/requirements/vllm-omni-rocm.txt                    # Runtime deps (ROCm)
neuralmagic/requirements/vllm-omni-cpu.txt                     # Runtime deps (CPU)
```

### Test Workflow

```
.github/workflows/test-vllm-omni.yml     # Omni model validation test
```

Open PR: [#531](https://github.com/neuralmagic/nm-cicd/pull/531) — adds Qwen3-Omni + TTS smoke tests with configurable `validation_type` (image_gen, tts, chat).

## How to Build (Quick Reference)

### Option A: From nm-vllm-omni-ent (recommended)

Go to [Actions → Midstream Build](https://github.com/neuralmagic/nm-vllm-omni-ent/actions/workflows/midstream-build.yml), pick a build step, and run. The workflow resolves the vLLM wheel automatically from `midstream/vllm-wheels.yml`.

For a full build + test, just push a tag:

```bash
git tag omni-v0.21.0-demo
git push origin omni-v0.21.0-demo
```

### Option B: Direct nm-cicd CLI (requires nm-cicd access)

Three commands, run in sequence. Each depends on the run ID from the previous step.

**Step 1 — vLLM wheel** (skip if reusing an existing wheel):
```bash
gh workflow run build-whl.yml --repo neuralmagic/nm-cicd \
  --ref feature/vllm-omni \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=main \
  -f target_device=cuda \
  -f python=3.12.5 \
  -f build_label=k8s-a100-build-13-0 \
  -f timeout=120 \
  -f partitions_file=neuralmagic/configs/partitions/minimal.yml
```

**Step 2 — vllm-omni wheel:**
```bash
gh workflow run build-whl.yml --repo neuralmagic/nm-cicd \
  --ref feature/vllm-omni \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=main \
  -f target_device=cuda \
  -f python=3.12.5 \
  -f build_label=k8s-a100-build-13-0 \
  -f timeout=120 \
  -f vllm_run_id=<VLLM_RUN_ID> \
  -f partitions_file=neuralmagic/configs/partitions/minimal.yml
```

**Step 3 — Docker image:**
```bash
gh workflow run build-image.yml --repo neuralmagic/nm-cicd \
  --ref feature/vllm-omni \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=main \
  -f target_device=cuda \
  -f build_label=ibm-wdc-k8s-h100-dind \
  -f run_id=<OMNI_RUN_ID> \
  -f vllm_run_id=<VLLM_RUN_ID>
```

**Tips:**
- You can reuse any existing vLLM wheel — just pass its run ID. Check GCS: `gcloud storage ls gs://nm-gha-cache/neuralmagic/nm-vllm-ent/assets/<run_id>/cuda/`
- To build from a PR: `-f branch=refs/pull/<N>/head`
- Image builds can be retriggered without rebuilding wheels — just pass the same run IDs

## Image Variants

The build produces two CUDA image variants:

| Variant | Description | Use Case |
|---------|-------------|----------|
| **`cuda`** (default) | Pre-built `flashinfer-cubin` + jit-cache, no CUDA devel headers | Most models. Smaller image, faster startup |
| **`cuda-develsdk`** | CUDA devel packages included, FlashInfer JIT fallback at runtime | Models that hit uncommon kernel configs (e.g. Qwen3-TTS) where cubin doesn't have pre-built kernels |

Both variants are pushed to `quay.io/vllm/automation-vllm-omni` with distinct tags. The develsdk variant exists because Nick Cao hit FlashInfer JIT failures running Qwen3-TTS — the pre-built cubin package doesn't cover every kernel configuration.

## Testing Infrastructure

### What's Wired Up Today

- **Build smoke test** — runs automatically at the end of a `full-chain` build. Serves Z-Image-Turbo, hits `/v1/images/generations`, validates a response comes back
- **`test-vllm-omni.yml`** on nm-cicd — configurable model validation workflow with `validation_type` parameter (image_gen, tts, chat)
- **Upstream pytest in the image** — the container image includes vllm-omni's `tests/` directory so upstream tests can be run inside the image without cloning

### Models Validated

| Model | Type | Status |
|-------|------|--------|
| Tongyi-MAI/Z-Image-Turbo | Image gen | Automated smoke test (default) |
| Qwen/Qwen3-TTS-12Hz-0.6B-Base | TTS | PR #531 (in progress) |
| Qwen/Qwen3-Omni-30B-A3B-Instruct | Chat (omni) | Needs multi-GPU runner (INFERENG-7571) |

### API Endpoints

vLLM-Omni serves OpenAI-compatible endpoints:

- `POST /v1/images/generations` — DALL-E compatible image gen
- `POST /v1/images/edits` — image editing
- `POST /v1/audio/speech` — TTS
- `POST /v1/chat/completions` — multimodal chat (text + image/audio/video)
- `GET /health`, `GET /v1/models`

## Prototype Philosophy

This work lives on feature branches intentionally. The approach:

1. **Build a working prototype on `feature/vllm-omni`** — get end-to-end builds and tests passing, learn what breaks, iterate fast
2. **Use it to inform design** — the feature branch is a reference implementation, not necessarily the final form. Some of it may merge directly, some may be rewritten with lessons learned
3. **Developer preview first** — get something in people's hands (OCTO ET, model teams) so we discover real-world issues early
4. **nm-cicd changes stay with Midstream** — the macro pattern on nm-vllm-omni-ent lets external teams trigger builds and see results without needing to understand or modify nm-cicd internals

The feature branch has been the vehicle for solving real problems: ffmpeg dependency management on UBI 9, FlashInfer JIT fallback, CUDA variant selection, cross-repo workflow dispatch, automated smoke testing. These lessons carry forward regardless of what the final integration looks like.

## Runner Labels

| Task | Label | Notes |
|------|-------|-------|
| Wheel build | `k8s-a100-build-13-0` | A100 with CUDA 13.0 build toolchain |
| Docker image | `ibm-wdc-k8s-h100-dind` | H100 with Docker-in-Docker (A100 dind unreliable since mid-May 2026) |
| Smoke test | `k8s-a100-solo` | Single A100 GPU |
| Multi-GPU test | TBD | Need `k8s-a100-duo` or similar for Qwen3-Omni-30B |

## JIRA Tracking

### Epic: [INFERENG-6288](https://issues.redhat.com/browse/INFERENG-6288) — vLLM-Omni Midstream: Initial Build

| Issue | Title | Status |
|-------|-------|--------|
| [INFERENG-6279](https://issues.redhat.com/browse/INFERENG-6279) | nm-cicd: First pass build pipeline | Closed |
| [INFERENG-6961](https://issues.redhat.com/browse/INFERENG-6961) | Midstream vLLM v0.20.0 wheel build | Closed |
| [INFERENG-7117](https://issues.redhat.com/browse/INFERENG-7117) | Midstream vLLM v0.21.0 wheel build | Closed |
| [INFERENG-7152](https://issues.redhat.com/browse/INFERENG-7152) | Image: SyntaxError, CUDA mismatch, missing deps | Closed |
| [INFERENG-7158](https://issues.redhat.com/browse/INFERENG-7158) | nm-cicd: Merge vLLM-Omni GHA stubs to main | Closed |
| [INFERENG-7162](https://issues.redhat.com/browse/INFERENG-7162) | Initial build validation smoke test | Closed |
| [INFERENG-7166](https://issues.redhat.com/browse/INFERENG-7166) | CUDA build runners broken (GHA runner deprecation) | Closed |
| [INFERENG-7302](https://issues.redhat.com/browse/INFERENG-7302) | Circular import (cv2/typing/numpy) blocks tests | Closed |
| [INFERENG-7303](https://issues.redhat.com/browse/INFERENG-7303) | Fix default wheel selection + build from synced main | Closed |
| [INFERENG-7304](https://issues.redhat.com/browse/INFERENG-7304) | Include upstream tests in container image | Closed |
| [INFERENG-7368](https://issues.redhat.com/browse/INFERENG-7368) | Add CUDA devel packages for FlashInfer JIT fallback | Closed |
| [INFERENG-7507](https://issues.redhat.com/browse/INFERENG-7507) | Qwen3-Omni e2e test against both CUDA variants | In Progress |
| [INFERENG-7571](https://issues.redhat.com/browse/INFERENG-7571) | Qwen3-Omni-30B e2e test (multi-GPU) | New |
| [INFERENG-7123](https://issues.redhat.com/browse/INFERENG-7123) | First pass model validation testing (Tarun) | New |
| [INFERENG-6964](https://issues.redhat.com/browse/INFERENG-6964) | GHA tooling for nm-vllm-omni-ent | Closed |

### Strategy / Cross-Team

| Issue | Title | Notes |
|-------|-------|-------|
| [RHAISTRAT-1266](https://issues.redhat.com/browse/RHAISTRAT-1266) | vLLM-Omni: Standalone Multimodal Inference Engine | Top-level strategy ticket |
| [RHAISTRAT-1242](https://issues.redhat.com/browse/RHAISTRAT-1242) | vLLM-Omni Tracker | Cross-team initiative tracker (Eng + OCTO) |
| [RHOAIENG-55052](https://issues.redhat.com/browse/RHOAIENG-55052) | Tech Preview - ODH and RHOAI Integrations | Downstream productization |
| [AIPCC-12521](https://issues.redhat.com/browse/AIPCC-12521) | AIPCC vllm-omni productization | Compatibility track |

## Key PRs (Breadcrumb Trail)

### nm-vllm-omni-ent

| PR | Title | Status |
|----|-------|--------|
| [#2](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/2) | Rebase midstream to v0.20.0 (Clodagh Walsh) | Merged |
| [#3](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/3) | Add midstream build tooling — GHA workflow | Merged |
| [#4](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/4) | Fix image deps: espeak-ng, ffmpeg, env vars | Merged |
| [#5](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/5) | Add smoke test gate to full-chain builds | Merged |
| [#6](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/6) | Bump v0.21.0 wheel mapping | Merged |
| [#7](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/7) | Add test stage to Dockerfile.ubi for upstream pytest | Merged |
| [#8](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/8) | Two CUDA image variants (jit-cache + develsdk) | Merged |
| [#9](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/9) | Expand smoke test to both CUDA variants + TTS | Open |
| [#10](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/10) | Two CUDA variants carry commit to main | Merged |

### nm-cicd

| PR | Title | Status |
|----|-------|--------|
| [#518](https://github.com/neuralmagic/nm-cicd/pull/518) | Add stub workflows for vllm-omni build and test | Merged |
| [#522](https://github.com/neuralmagic/nm-cicd/pull/522) | Add OCP-based smoke test and model validation | Merged |
| [#523](https://github.com/neuralmagic/nm-cicd/pull/523) | Add vLLM-Omni smoke test (stub + RBAC) | Merged |
| [#527](https://github.com/neuralmagic/nm-cicd/pull/527) | Add upstream pytest runner for test images | Merged |
| [#528](https://github.com/neuralmagic/nm-cicd/pull/528) | Two CUDA image variants — develsdk pipeline support | Merged |
| [#531](https://github.com/neuralmagic/nm-cicd/pull/531) | Qwen3-Omni + TTS smoke tests with configurable validation | Open |

## Known Pain Points

- **Podman / rootless containers** — needs `--security-opt=label=disable`, `--userns=keep-id`, tmpfs mounts for writable dirs. HF cache mounts require specific env var overrides. See the vLLM-Omni cheat sheet for working `podman run` invocations
- **ffmpeg on UBI 9** — ffmpeg is not in EPEL 9 or standard UBI repos. Solved by using `imageio-ffmpeg` (pip package with a bundled static binary) + a shim script that symlinks it into PATH
- **Runner label drift** — build labels change as runner pools rotate. If builds hang, check recent successful runs to see which pool is healthy
- **vLLM version string** — nm-vllm-ent computes versions via `git describe`, so the wheel metadata may say `0.18.1.dev1273+rhaiv.2` even when the code content is v0.21.0. This is cosmetic — the code is correct

## What's Next

- **Qwen3-Omni e2e testing** — needs multi-GPU runners (INFERENG-7571)
- **Model validation framework** — Tarun Kumar driving (INFERENG-7123)
- **Upstream sync cadence** — define branch naming, merge cadence, carry commit management for nm-vllm-omni-ent
- **Pipeline graduation** — decide what moves from `feature/vllm-omni` to nm-cicd main
- **Release flow integration** — wire into accept-sync, nightly, and release pipelines
