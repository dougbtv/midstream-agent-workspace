# GLM-5.x Deployment Notes

## Current State: GLM-5.2-FP8 on MI300X

**Deployment YAML:** `rhaiis-midstream-snippets/charts/deployments/glm-5-2-fp8-mi300x.yml`
**Model cache setup:** `rhaiis-midstream-snippets/charts/deployments/glm-5-2-setup-model-cache.sh`

### Current config

| Field | Value |
|-------|-------|
| Image | `quay.io/vllm/vllm-rocm:0.28.0_rhaiv.5.rocm714` |
| Model | `zai-org/GLM-5.2-FP8` |
| Hardware | 8x MI300X (192GB each), tensor-parallel-size 8 |
| Model cache PVC | 800Gi (`lvms-vg1`) |
| gpu-memory-utilization | 0.80 |
| kv-cache-dtype | fp8_e4m3 |
| max-model-len | 524288 |
| max-num-seqs | 32 |
| speculative-config | `{"method":"mtp","num_speculative_tokens":5}` |
| tool-call-parser | glm47 |
| reasoning-parser | glm45 |
| linear-backend | aiter |
| moe-backend | aiter |
| prefix-caching | enabled |
| load-format | fastsafetensors |
| trust-remote-code | true |

### Current env vars

- `OMP_NUM_THREADS=4`
- `TVM_FFI_CACHE_DIR=/tmp/tvm`
- `VLLM_ROCM_USE_AITER_FUSION_SHARED_EXPERTS=1`

### Startup probe

`initialDelaySeconds: 180` (3 min, with failureThreshold adding ~15m → up to ~18m total)

### Resource limits

- CPU: 32 limit / 10 request
- Memory: 256Gi
- `amd.com/gpu: 8`

### Strategy

`Recreate` — full node deployment, no RollingUpdate surge.

---

## GLM-5.3 Upgrade Plan

Reference: https://recipes.vllm.ai/zai-org/GLM-5.3

### Model overview

- **Model:** `zai-org/GLM-5.3` — ~743B-param MoE (39B active), native FP8
- **BF16 weights:** `zai-org/GLM-5.3-BF16` (multi-node only)
- **NVFP4 variant:** `Inferact/GLM-5.3-NVFP4` (Blackwell only)
- **DSpark draft:** `RedHatAI/GLM-5.3-speculator.dspark` (8 speculative tokens)

### Prerequisites

- vLLM build containing native GLM5Next support ([PR #62424](https://github.com/vllm-project/vllm/pull/62424)), including matching Rust frontend binary.
- Transformers 5.15.0+.
- GPU: 8xH200/H20 (141GB) for single-node FP8; 8xB200 for full 1M context. AMD MI300X/MI325X/MI350X/MI355X supported.

### Key differences from 5.2 → 5.3

| Field | GLM-5.2 (current) | GLM-5.3 (target) |
|-------|-------------------|-------------------|
| model | `zai-org/GLM-5.2-FP8` | `zai-org/GLM-5.3` |
| reasoning-parser | `glm45` | `glm47` |
| tokenizer-mode | _(not set)_ | `glm5_next` |
| VLLM_USE_RUST_FRONTEND | _(not set)_ | `1` |
| VLLM_ROCM_USE_AITER | _(not set)_ | `1` |
| tool-call-parser | `glm47` | `glm47` (same) |
| kv-cache-dtype | `fp8_e4m3` | `fp8_e4m3` (same) |
| speculative-config | mtp, 5 tokens | mtp, 5 tokens (same) |
| gpu-memory-utilization | 0.80 | 0.80 (same) |
| max-model-len | 524288 | 524288 (same) |
| max-num-seqs | 32 | 32 (same) |
| linear-backend | aiter | aiter (same) |
| moe-backend | aiter | aiter (same) |

### AMD MI300X serve command (from recipe)

```
VLLM_ROCM_USE_AITER=1 \
VLLM_ROCM_USE_AITER_FUSION_SHARED_EXPERTS=1 \
VLLM_USE_RUST_FRONTEND=1 \
vllm serve zai-org/GLM-5.3 \
  --tokenizer-mode glm5_next \
  --kv-cache-dtype fp8_e4m3 \
  --tensor-parallel-size 8 \
  --speculative-config.method mtp \
  --speculative-config.num_speculative_tokens 5 \
  --tool-call-parser glm47 \
  --enable-auto-tool-choice \
  --reasoning-parser glm47 \
  --gpu-memory-utilization 0.80 \
  --max-model-len 524288 \
  --max-num-seqs 32 \
  --linear-backend aiter \
  --moe-backend aiter
```

### Reasoning modes

GLM-5.3 thinking is always on (opens `<think>` block unconditionally). Three effort levels via `reasoning_effort`:

| Mode | How to request | Use case |
|------|----------------|----------|
| Think Max (default) | omit or `"max"` | Hard math, multi-step planning, agentic |
| Think High | `"high"` | Balanced depth and latency |
| Think Low | `"low"` | Simple Q&A, lowest latency |

Pass via `chat_template_kwargs.reasoning_effort` or top-level OpenAI `reasoning_effort`.

### Validation data from recipe

- **8x MI325X** (v0.29.0): 1,477,168-token KV cache, 2.82x max concurrency, MTP mean acceptance 2.77 (5 tokens drafted).
- **8x MI350X** (v0.29.0): 2,013,072-token KV cache, 3.84x max concurrency, MTP mean acceptance 3.19.

### Phase plan

#### Phase 1: Try on current image (0.28.0_rhaiv.5.rocm714)

The current image predates PR #62424. Fallback path for builds without `glm5_next`:

- Replace `--tokenizer-mode glm5_next` with `--tokenizer-mode auto`
- Omit `VLLM_USE_RUST_FRONTEND=1` (Rust binary likely absent)
- Update model → `zai-org/GLM-5.3`
- Update reasoning-parser → `glm47`
- Add `VLLM_ROCM_USE_AITER=1` env var
- Keep everything else identical

If it loads and serves: great, config-only change. If the architecture isn't recognized, we move to Phase 2.

#### Phase 2: Jump to v0.31.0_rhaiv.5

Available on Quay: `v0.31.0_rhaiv.5` (Oct 27) and `v0.31.0_rhaiv.5.rubin` (Oct 30).

This should include PR #62424 and the Rust frontend binary. Use the full recipe config with `--tokenizer-mode glm5_next` and `VLLM_USE_RUST_FRONTEND=1`.

Verify the image has the ROCm 7.14 stack or check what ROCm version it bundles (the tag dropped the `.rocm714` suffix).

#### Phase 3: DSpark speculative (optional)

If MTP acceptance is underwhelming, try DSpark with 8 speculative tokens using `RedHatAI/GLM-5.3-speculator.dspark`:

```
--speculative-config '{"method":"dspark","model":"RedHatAI/GLM-5.3-speculator.dspark","num_speculative_tokens":8}'
```

### Available Quay images (as of 2026-11-05)

| Tag | Date | Notes |
|-----|------|-------|
| `0.28.0_rhaiv.5.rocm714` | Oct 7 | Current — no glm5_next support |
| `v0.31.0_rhaiv.5` | Oct 27 | Target for Phase 2 |
| `v0.31.0_rhaiv.5.rubin` | Oct 30 | Unknown "rubin" suffix |
| `0.26.0_rhaiv.9.rocm714` | Oct 6 | Older baseline |

No 0.28.0 build beyond rhaiv.5 exists. No 0.29.0 or 0.30.0 ROCm images in the repo.

### GPU sizing

- **Current:** 8x MI300X (192GB each) = 1,536GB total VRAM
- **GLM-5.3 FP8 weights:** ~743GB
- **Remaining for KV cache + activations:** ~793GB
- **Recipe floor:** 8xH200 (141GB each) = 1,128GB total — MI300X has 408GB more headroom
- **Same GPU count and type works.** No additional hardware needed.

### Troubleshooting

- FP8 performance: DeepGEMM required (`install_deepgemm.sh`) — likely not relevant on ROCm/aiter path.
- For benchmarking: add `--no-enable-prefix-caching` for clean measurement. MTP acceptance is low in synthetic runs.
- Benchmarking command:
  ```
  vllm bench serve \
    --model zai-org/GLM-5.3 \
    --dataset-name random \
    --random-input-len 8000 \
    --random-output-len 1024 \
    --request-rate 10 \
    --num-prompts 32 \
    --ignore-eos
  ```
