# vLLM-Omni Full Test Suite — Handoff Doc

Status: **Draft** — 2026-06-27
Tracked by: [INFERENG-8000](https://redhat.atlassian.net/browse/INFERENG-8000), [INFERENG-7813](https://redhat.atlassian.net/browse/INFERENG-7813)

## Prerequisites

1. **A built omni image** — run the full chain build first (see `MIDSTREAM_OMNI_BUILD.md`).
   Current validated image:
   ```
   quay.io/vllm/automation-vllm-omni:cuda-28259320336
   ```

2. **nm-cicd branch** with OCP test framework + build pipeline. Use either:
   - `doug/INFERENG-8000-omni-rebase` (reconciled branch, PR #587)
   - `main` (has all OCP test plumbing from Tarun's PR #566 + #519 refactor)

3. **HF access** to omni models (gated models need token with access granted).

## How It Works

The OCP test framework (`ocp-test.yml`) deploys a vLLM-Omni pod on k8s, waits for health, then runs pytest smoke tests against the API. The key parameters:

| Parameter | What it does |
|-----------|-------------|
| `deployment_type` | `vllm-omni` adds `--omni` to the `vllm serve` command |
| `model_capabilities` | Selects which `test_smoke_*` functions run (comma-separated) |
| `model` | HuggingFace model name to serve |
| `label` | Runner label — must be a `-util` runner for k8s pod deployment |

## The Full Test Matrix

### Image Generation

```bash
# Z-Image-Turbo — primary image gen target, single GPU
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-8000-omni-rebase \
  -f config_ref=main \
  -f label=k8s-a100-util \
  -f timeout=90 \
  -f model=Tongyi-MAI/Z-Image-Turbo \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-28259320336 \
  -f target_device=cuda \
  -f model_capabilities=image_generation \
  -f deployment_type=vllm-omni \
  -f test_type=smoke \
  -f ocp_host_url="" \
  -f collect_image_info=false
```

### Text-to-Speech (TTS)

```bash
# Qwen3-TTS — omni TTS model
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-8000-omni-rebase \
  -f config_ref=main \
  -f label=k8s-a100-util \
  -f timeout=90 \
  -f model=Qwen/Qwen3-TTS \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-28259320336 \
  -f target_device=cuda \
  -f model_capabilities=speech_generation \
  -f deployment_type=vllm-omni \
  -f test_type=smoke \
  -f ocp_host_url="" \
  -f collect_image_info=false
```

### Multimodal Chat (Qwen3-Omni)

```bash
# Qwen3-Omni 30B — multi-GPU (needs TP=2, so duo runner)
# This is the flagship omni model: text + audio + image in, text + audio out
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-8000-omni-rebase \
  -f config_ref=main \
  -f label=k8s-a100-duo-util \
  -f timeout=120 \
  -f model=Qwen/Qwen3-Omni \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-28259320336 \
  -f target_device=cuda \
  -f model_capabilities=text \
  -f deployment_type=vllm-omni \
  -f test_type=smoke \
  -f ocp_host_url="" \
  -f collect_image_info=false
```

> **Note:** Qwen3-Omni is a MoE model (30B). The OCP framework needs a server config in `model-validation-configs` specifying `tensor_parallel_size: 2` for this model. If there's no config yet, the deployment may fail or fall back to defaults. Check if `model-validation-configs/Qwen/Qwen3-Omni/` exists first. If not, a config needs to be added there (separate PR to model-validation-configs).

### Standard Text (Regression Check)

```bash
# Run a standard text model to verify no regression on non-omni deployments
# This uses deployment_type=vllm (default), NOT vllm-omni
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-8000-omni-rebase \
  -f config_ref=main \
  -f label=k8s-a100-util \
  -f timeout=60 \
  -f model=meta-llama/Llama-3.2-1B-Instruct \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-28259320336 \
  -f target_device=cuda \
  -f model_capabilities=text \
  -f deployment_type=vllm \
  -f test_type=smoke \
  -f ocp_host_url="" \
  -f collect_image_info=false
```

> **Note:** This tests the omni *image* with a standard vLLM deployment (no `--omni` flag). Validates that the image works for regular LLM serving too.

## Fire-and-Forget: Run All Tests

Copy-paste block to fire everything in parallel:

```bash
IMAGE="quay.io/vllm/automation-vllm-omni:cuda-28259320336"
REF="doug/INFERENG-8000-omni-rebase"

# 1. Image generation — Z-Image-Turbo
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd --ref $REF \
  -f config_ref=main -f label=k8s-a100-util -f timeout=90 \
  -f model=Tongyi-MAI/Z-Image-Turbo -f vllm_image=$IMAGE \
  -f target_device=cuda -f model_capabilities=image_generation \
  -f deployment_type=vllm-omni -f test_type=smoke \
  -f ocp_host_url="" -f collect_image_info=false

# 2. TTS — Qwen3-TTS
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd --ref $REF \
  -f config_ref=main -f label=k8s-a100-util -f timeout=90 \
  -f model=Qwen/Qwen3-TTS -f vllm_image=$IMAGE \
  -f target_device=cuda -f model_capabilities=speech_generation \
  -f deployment_type=vllm-omni -f test_type=smoke \
  -f ocp_host_url="" -f collect_image_info=false

# 3. Multimodal chat — Qwen3-Omni (multi-GPU)
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd --ref $REF \
  -f config_ref=main -f label=k8s-a100-duo-util -f timeout=120 \
  -f model=Qwen/Qwen3-Omni -f vllm_image=$IMAGE \
  -f target_device=cuda -f model_capabilities=text \
  -f deployment_type=vllm-omni -f test_type=smoke \
  -f ocp_host_url="" -f collect_image_info=false

# 4. Regression — standard text model (no --omni)
gh workflow run ocp-test.yml --repo neuralmagic/nm-cicd --ref $REF \
  -f config_ref=main -f label=k8s-a100-util -f timeout=60 \
  -f model=meta-llama/Llama-3.2-1B-Instruct -f vllm_image=$IMAGE \
  -f target_device=cuda -f model_capabilities=text \
  -f deployment_type=vllm -f test_type=smoke \
  -f ocp_host_url="" -f collect_image_info=false
```

## Monitoring

After triggering, watch all runs:

```bash
# List recent ocp-test runs (yours will be the latest)
gh run list --repo neuralmagic/nm-cicd --workflow ocp-test.yml --limit 10 \
  --json databaseId,status,conclusion,displayTitle \
  --jq '.[] | "\(.databaseId) | \(.conclusion // .status) | \(.displayTitle)"'
```

Watch a specific run:

```bash
gh run watch <RUN_ID> --repo neuralmagic/nm-cicd
```

## Diagnosing Failures

```bash
# Get job IDs and status
gh run view <RUN_ID> --repo neuralmagic/nm-cicd \
  --json jobs --jq '.jobs[] | "\(.databaseId) \(.name) \(.conclusion)"'

# Tail logs for the failed job
gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 | tail -100
```

### Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `DeploymentTimeoutError` | Model too large for runner, or image pull slow | Use bigger runner (duo/quad), increase timeout |
| `FileNotFoundError: config` | No model-validation-configs entry | Add server.yml + tasks.yml in model-validation-configs repo |
| `ModuleNotFoundError: vllm_omni` | Image doesn't have vllm-omni installed | Rebuild image, check wheel install order in prepare.sh |
| `ValueError: requires vllm-omni` | `deployment_type` not set to `vllm-omni` | Add `-f deployment_type=vllm-omni` |
| `Connection refused` on API | vLLM crashed during model load | Check pod logs for OOM, CUDA errors, missing deps |
| `pytest: no tests ran` | model_capabilities doesn't match any test marker | Check that capability string matches a `ModelTypes` enum value |
| Transient CDN/registry timeout | Flaky infrastructure | Re-trigger with same params |

## What Each Test Validates

| Test | Endpoint | Validation |
|------|----------|------------|
| `test_smoke_image_generation` | `POST /v1/images/generations` | Sends prompt, decodes b64_json response, validates raster image format (PNG/JPEG/WebP/GIF/BMP magic bytes), checks min decoded size (64 bytes) |
| `test_smoke_speech_generation` | `POST /v1/audio/speech` | Sends text + voice, validates RIFF/WAVE header, checks min WAV file size (512 bytes) |
| `test_smoke_text` | `POST /v1/chat/completions` | Sends chat messages, validates non-empty text response, checks streaming + non-streaming |
| `test_smoke_multimodal` | `POST /v1/chat/completions` | Sends multimodal input (image/audio/video URL), validates text response |

All omni-specific tests use the `require_vllm_omni_deployment` fixture which skips unless `deployment_type == "vllm-omni"`.

## Next Steps (Beyond Smoke)

Once smoke tests pass, the following are needed for full RHOAI 3.5 EA2 validation (INFERENG-7813):

1. **Add omni models to `model_registry.yml`** — with `deployment_type: vllm-omni` and appropriate capabilities. This lets `model-validation-ocp.yml` pick them up automatically in the matrix.

2. **Add model-validation-configs entries** — server.yml for each omni model specifying tensor_parallel_size, max_model_len, etc.

3. **Upstream pytest integration** — run vllm-omni's own test suite (tests/entrypoints/openai_api/) inside the container. This was previously done via the bash scripts we dropped; needs to be reimplemented via the OCP framework or a dedicated workflow.

4. **Multi-runner coverage** — run on H100 in addition to A100 (image gen and TTS models may behave differently).

5. **Accuracy/benchmark tests** — currently smoke-only. Performance benchmarks and accuracy evals need model-validation-configs entries with ground truth values.

## Reference

- **Build doc:** `MIDSTREAM_OMNI_BUILD.md` (worktree root)
- **PR:** [nm-cicd #587](https://github.com/neuralmagic/nm-cicd/pull/587)
- **Epic:** [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288)
- **Test suite ticket:** [INFERENG-7813](https://redhat.atlassian.net/browse/INFERENG-7813)
- **OCP test framework:** `neuralmagic/ocp_model_deployment/` in nm-cicd
- **Test functions:** `tests/model_deployment/test_smoke.py`
