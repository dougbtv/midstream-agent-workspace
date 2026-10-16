# Fast2 vLLM-Omni RC validation handoff — 2026-09-23

## Candidate

- Image: `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.2-1790103942`
- Digest: `sha256:d1de5abf8f8cd69e8f7f901b5b7cd1d28f4d3afad17fa91c1c18d5690803497b`
- AIPCC Release: `rhaiis-3-6-fast2-20260922-190448-000-5fe9050-99gkg`
- Component tag: `vllm-omni-cuda-v2026092201` at `5fe905032dcabd90b58c01aef971994406b419db`

## Validation dispatched

- nm-cicd workflow: [`model-validation-ocp.yml`](https://github.com/neuralmagic/nm-cicd/actions/workflows/model-validation-ocp.yml)
- Run: [35920031042](https://github.com/neuralmagic/nm-cicd/actions/runs/35920031042)
- State at dispatch: queued.
- Workflow ref: `main`
- Inputs:
  - `config_ref=main`
  - `source_ref=main`
  - `target_device=cuda`
  - `testing_options_flags=--image-test-type smoke`
  - `timeout=180`
  - `run_accuracy_check=false`
  - `wf_category=RELEASE`

This repeats the Fast1 formal validation shape (seven-model OCP smoke matrix)
against the immutable Fast2 AIPCC image. The workflow identifies the image as
vLLM-Omni from its name and uses the Omni model registry entries and
`deployment_type: vllm-omni`.

## Results

Pending. Record each substantive model's readiness and functional smoke-test
result here, then distinguish test/deployment infrastructure failures from
image failures. In particular, compare any Qwen3-Omni multi-GPU failure with
the known stage/device mapping issue in INFERENG-9672, and do not treat a
wrapper-only cleanup failure as a failed workload without its underlying logs.

## Follow-up

1. Monitor [run 35920031042](https://github.com/neuralmagic/nm-cicd/actions/runs/35920031042) to completion.
2. Capture the generated test matrix and per-model outcomes.
3. Update INFERENG-10568 with the final validation evidence and classify any
   failures using logs.
