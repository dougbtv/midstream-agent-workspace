# GLM-5.x Deployment Notes

## GLM-5.3 cutover progress (2026-11-06)

The reversible cutover is underway on the verified IBM MI300X cluster.

| Item | Current state |
|------|---------------|
| Preserved fallback | `glm-5-2-vllm` is scaled to **0**; `glm-5-2` Helm release revision 17 and both 5.2 PVCs remain intact. |
| New release | `glm-5-3`, Helm revision 3, scheduled on `sndgpuasw-mi300x-2` with 8 AMD GPUs. |
| Image | `quay.io/vllm/automation-vllm:rocm-40752177506`, proven by the 5.2 runtime as vLLM `0.31.1.dev1+rhaiv.5`. |
| Storage | Preloaded `glm-5-3-fp8-pvc` (800Gi) and new `glm-5-3-vllm-cache-pvc` (10Gi), both `lvms-vg1`. |
| Model cache | Offline cache snapshot `ba32f8c19c8738e2c57c71798c601d477a7bf844` is being used; no weight download during serving startup. |
| Current startup | Pod is Running with one historical restart from the initial KV-cache sizing failure; the retry is loading the MTP drafter. It is **not ready yet**. |

### Applied deployment fixes

1. The image lacks the optional `vllm-rs` binary, so `VLLM_USE_RUST_FRONTEND=1`
   failed with `FileNotFoundError`. The deployed config uses `tokenizer-mode: auto`
   and omits that env var.
2. At `gpu-memory-utilization: 0.80`, a `max-model-len` of 524288 required
   23.64GiB KV cache while only 23.56GiB was available. The retry uses
   `max-model-len: 522000` rather than raising utilization.

Source changes are on [PR #184](https://github.com/neuralmagic/rhaiis-midstream-snippets/pull/184):
`f8001b4` (proven image), `9b0f744` (tokenizer fallback), and `1505b3a`
(MI300X KV-cache sizing).

## Current State: GLM-5.2-FP8 on MI300X

**Deployment YAML:** `rhaiis-midstream-snippets/charts/deployments/glm-5-2-fp8-mi300x.yml`
**Model cache setup:** `rhaiis-midstream-snippets/charts/deployments/glm-5-2-setup-model-cache.sh`

### Verified live inventory (2026-11-06)

The actual serving cluster is the IBM Cloud MI300X OpenShift cluster, not the
retired Azure AMD-02 cluster documented in older `nm-cicd` material.

| Field | Verified value |
|-------|----------------|
| API server | `https://api.sndgpuasw.cat.mvbp-cloud.com:6443` |
| Namespace | `rhaiis-midstream` |
| Helm release | `glm-5-2`, revision 17 (`deployed`) |
| Deployment | `glm-5-2-vllm`, `Recreate`, 1 replica |
| Runtime image | `quay.io/vllm/automation-vllm:rocm-40752177506` |
| Current pod/node | `glm-5-2-vllm-f58b7775-sl8c7` on `sndgpuasw-mi300x-2` |
| GPU allocation | 8 × `amd.com/gpu` (all GPUs on node 2) |
| Model PVC | `glm-5-2-fp8-pvc`, 800Gi, `lvms-vg1`, Bound |
| vLLM cache PVC | `glm-5-2-vllm-cache-pvc`, 10Gi, `lvms-vg1`, Bound |
| Route | `https://glm-5-2.apps.sndgpuasw.cat.mvbp-cloud.com` |
| Service account | `vllm-anyuid-sa` |

The other MI300X node is not spare: `qwen3-5-397b-vllm` uses 4 of 8 GPUs on
`sndgpuasw-mi300x-1`. Therefore GLM-5.3 cannot run in parallel with GLM-5.2;
the cutover must free node 2 first.

### Reversible GLM-5.2 → GLM-5.3 cutover

Keep the `glm-5-2` Helm release and both of its PVCs. Do **not** uninstall it:
that preserves the known-good deployment definition, release history, and model
cache for rollback.

1. Scale `glm-5-2-vllm` to zero. This releases all eight GPUs on
   `sndgpuasw-mi300x-2` without deleting the deployment or PVCs.
2. Install GLM-5.3 as a distinct Helm release with its own route and
   `glm-5-3-fp8-pvc`. With node 1 retaining only four free GPUs, its 8-GPU pod
   can schedule only on node 2.
3. Validate the GLM-5.3 route, `/health`, `/v1/models`, and chat completion.
4. If the test fails, scale GLM-5.3 to zero and restore GLM-5.2 to one replica.

The core rollback commands are:

```bash
oc scale deployment/glm-5-2-vllm -n rhaiis-midstream --replicas=0
# deploy and validate the separate GLM-5.3 release
# on failure: scale the GLM-5.3 deployment to zero first
oc scale deployment/glm-5-2-vllm -n rhaiis-midstream --replicas=1
```

The checked-in GLM-5.2 values file is useful historical configuration, but its
image tag is not the live runtime image above; use the live inventory as the
restore source of truth.

### Current config

| Field | Value |
|-------|-------|
| Image | See verified runtime image above; the historical values file lists `quay.io/vllm/vllm-rocm:0.28.0_rhaiv.5.rocm714` |
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

### Decision: start on v0.31.0_rhaiv.5

v0.31.0 was released Oct 5. PR #62424 (native GLM5Next tokenizer) merged Oct 19.
Assume it is **not** cherry-picked into rhaiv.5. The fallback `--tokenizer-mode auto`
works without #62424 — it uses the HuggingFace/MiniJinja renderer instead of the
fast Rust path. Slower prompt processing, but the model loads and serves.

v0.31.0 release notes confirm GLM-5.3 model architecture support is in
(GLM-5.3-Flash Quark MXFP4 #56176, GLM-5.2-MXFP4 ROCm #51915). The missing piece
is just the native `glm5_next` renderer, which is an optimization, not a gate.

Skip 0.28.0 — no point testing on an image that predates the architecture support.

`v0.31.0_rhaiv.5.rubin` is likely a next-gen NVIDIA (Rubin) variant — skip it.

### Phase plan

#### Phase 1: Deploy on v0.31.0_rhaiv.5 (current target)

Image: `quay.io/vllm/vllm-rocm:v0.31.0_rhaiv.5`

Try `--tokenizer-mode glm5_next` + `VLLM_USE_RUST_FRONTEND=1` first. If the image
rejects `glm5_next` (renderer not present without #62424 cherry-pick), fall back to:
- `--tokenizer-mode auto`
- Omit `VLLM_USE_RUST_FRONTEND=1`
- Everything else stays the same

#### Phase 2: DSpark speculative (optional)

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
