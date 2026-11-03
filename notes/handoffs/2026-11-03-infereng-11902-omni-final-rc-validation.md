# INFERENG-11902 — vLLM-Omni 3.6 final-RC validation failure

Date: 2026-11-03

## Release impact

The released vLLM-Omni 3.6 candidate is not ready for final-RC disposition.
Corrected OCP smoke validation completed **1/7 passed** one day before the
November 4 final-RC deadline. Three serving models cannot start because the
released image contains incompatible FlashInfer packages. Three diffusion
models serve inference but disagree with one smoke-harness expectation.

Tracker: [INFERENG-11902](https://redhat.atlassian.net/browse/INFERENG-11902)

## Immutable evidence

- Released image: `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-1793655518`
- Digest: `sha256:b9f77b51012e6b04e108f297fb5a050c8e1cb69629db6e4faae0b351f4855623`
- Containers source commit: `20fe60f153fcb37f84d2071d431817f0e792d5a1`
- Corrected validation: [nm-cicd run 41664893948](https://github.com/neuralmagic/nm-cicd/actions/runs/41664893948)
- Workflow/config source: `rhai/0.30.0`; model configs: `main`
- Image packages from the run SBOM:
  - `vllm-omni==0.30.0+rhaiv.3`
  - `vllm==0.30.0+rhaiv.1`
  - `flashinfer-python==0.6.18.post1`
  - `flashinfer-cubin==0.7.1`
  - `flashinfer-jit-cache==0.7.0.post1+cu130` and matching architecture caches

The run passed setup, registry authentication, image validation, image metadata
collection, SBOM generation, model-config checkout, and OCP deployment setup.
This result supersedes the earlier attempts blocked by tenant-image access and
the nonexistent model-config `rhai/0.30.0` ref.

## Matrix classification

| Model | Result | Classification | Evidence |
|-------|--------|----------------|----------|
| `black-forest-labs/FLUX.2-klein-4B` | Passed | Validated | Deployment and smoke job passed. |
| `Qwen/Qwen3-Omni-30B-A3B-Instruct` | Failed at engine start | Released-image dependency defect | `flashinfer-cubin 0.7.1` rejects `flashinfer-python 0.6.18.post1`. |
| `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` | Failed at engine start | Released-image dependency defect | `flashinfer_jit_cache` lacks `get_jit_cache_dir`. |
| `mistralai/Voxtral-4B-TTS-2603` | Failed at engine start | Released-image dependency defect | Same missing JIT-cache API during FlashInfer import. |
| `black-forest-labs/FLUX.1-schnell` | Failed one assertion | Validation-contract question | Server started and inferred, but accepted an empty prompt. |
| `black-forest-labs/FLUX.2-dev` | Failed one assertion | Validation-contract question | Server started and inferred, but accepted an empty prompt. |
| `Tongyi-MAI/Z-Image-Turbo` | Failed one assertion | Validation-contract question | Server started and inferred, but accepted an empty prompt. |

Automated aggregation created
[nm-cicd #1604](https://github.com/neuralmagic/nm-cicd/issues/1604) for the
FlashInfer startup failures and
[nm-cicd #1605](https://github.com/neuralmagic/nm-cicd/issues/1605) for the
empty-prompt expectation.

## Root cause boundary

The selected `nm-vllm-ent` `v0.30.0+rhaiv.1` source contract pins
`flashinfer-python`, `flashinfer-cubin`, and `flashinfer-jit-cache` together at
`0.6.18.post1` in `requirements/cuda.txt`.

The released containers commit instead declared all three as unversioned direct
dependencies in `context/vllm-omni/pyproject.toml`. The generated hermetic lock
resolved Python to `0.6.18.post1` through vLLM while independently choosing
newer cubin and JIT-cache packages. Lock verification proved reproducibility of
that set; it did not prove cross-package runtime compatibility. The immutable
SBOM and runtime traces confirm the mismatch reached the released image.

Do not use `FLASHINFER_DISABLE_VERSION_CHECK=1` as the release fix. It bypasses
one guard but does not restore the missing JIT-cache API or establish kernel
compatibility.

The diffusion failures are separate. Empty prompts can be meaningful for some
diffusion systems; the team must decide the supported behavior. If acceptance
is intended, narrow the harness expectation. If rejection is required, add
server-side validation and rebuild. Those failures do not justify delaying the
FlashInfer correction.

## Required recovery path

1. Pin the three FlashInfer packages to one reviewed compatible version set;
   the selected source contract is `0.6.18.post1` for all three.
2. Regenerate the x86_64/aarch64 hermetic lock from the selected production
   base/index and review the exact resolved versions.
3. Add a lock or image assertion that the FlashInfer Python, cubin, and
   JIT-cache versions match the selected vLLM contract.
4. Build a new Konflux candidate and verify its immutable digest and SBOM before
   testing.
5. Run targeted OCP startup smoke for Qwen3-Omni, Qwen3-TTS, and Voxtral first;
   then run the full seven-model matrix against the same digest.
6. Record the empty-prompt product decision and corresponding harness/runtime
   change separately.
7. Promote only after the corrected immutable image passes the agreed matrix or
   Release Engineering explicitly accepts documented exceptions.

## Current disposition

- **Proved:** released-image dependency defect affects three serving models.
- **Proved:** three diffusion servers start and infer; their failure is the
  empty-prompt expectation.
- **Proved:** one model passes the current matrix.
- **Not yet proved:** a corrected image, targeted startup success, full rerun,
  or accepted exception.
- **Recommendation:** hold final-RC promotion of this image digest.
