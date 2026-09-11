# Final vLLM-Omni RC validation handoff — 2026-09-11

## Candidate

- Image: `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1789070722`
- Digest: `sha256:4d17c21a588539f6429fd1d24e81b8a3f36448f2f4e21f1f241089dfa5568a42`
- Image contains `v0.26.0+rhaiv.7`.
- nm-cicd PR: [#645](https://github.com/neuralmagic/nm-cicd/pull/645), branch `feature/vllm-omni-dev-preview`.
- OCP H100 runner mapping was pushed in commit `17452be7fec0330311fcc7ed8f0ac70b6105f3ba`.

## Validation result

[nm-cicd run 34606096112](https://github.com/neuralmagic/nm-cicd/actions/runs/34606096112) ran the seven-model OCP smoke matrix on `ibm-wdc-os-h100-util`.

- Substantive passes (6/7): FLUX.2-klein, FLUX.2-dev, FLUX.1-schnell, Z-Image-Turbo, Voxtral TTS, and Qwen3-TTS CustomVoice.
- Each passing model reached `/health` HTTP 200 and passed its two functional smoke tests.
- Qwen3-Omni-30B failed before readiness: TP=4 was requested, but only one GPU was visible in the serving pod; ranks 1-3 exited as out of bounds.
- This matches the known multi-GPU/stage-override deployment regression in [INFERENG-9672](https://redhat.atlassian.net/browse/INFERENG-9672), not evidence of an image defect.
- All wrapper jobs were red because the WDC runner service account cannot delete per-run PVCs. This is a known runner configuration defect; the six successful workloads remain valid.

## Jira state

- [INFERENG-9841](https://redhat.atlassian.net/browse/INFERENG-9841): validation and image details posted; final-RC promotion recommended; moved to **Review**.
- [INFERENG-9672](https://redhat.atlassian.net/browse/INFERENG-9672): exact Qwen3-Omni regression evidence posted; remains **Review**.
- [INFERENG-10578](https://redhat.atlassian.net/browse/INFERENG-10578): completion comment posted with references to 9841 and 9672; moved to **Closed**.

## Next

- Promote the candidate as the final RC through the coordination tracked in INFERENG-9841.
- Fix the Qwen3-Omni stage/device mapping under INFERENG-9672.
- Runner owners are separately standardizing the WDC PVC configuration; do not use cleanup `403`s to judge image validity.
