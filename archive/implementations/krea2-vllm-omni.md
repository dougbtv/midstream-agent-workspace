# Krea 2 Support in vLLM-Omni — Implementation Report

**PR:** [vllm-project/vllm-omni#4687](https://github.com/vllm-project/vllm-omni/pull/4687)
**Branch:** `doug/krea2-support` on `dougbtv/vllm-omni`
**Jira:** INFERENG-8135
**Head commit:** `c1568475`
**Date:** 2026-06-23
**Status:** Both checkpoints verified on A100 GPU — clean, artifact-free output

---

## What shipped

Text-to-image support for Krea 2 flow-matching diffusion models. Both base
(28-step, guidance_scale=4.5) and distilled/TDM (8-step, guidance_scale=0)
checkpoints are handled. Creativity levels, style references, and moodboard
conditioning are explicitly out of scope — they aren't in diffusers yet either.

Note: Krea 2 weights are released under the **Krea 2 Community License**.
See the model card at https://huggingface.co/krea/Krea-2-Raw for details.

## Files (8 files, ~1,450 lines)

| File | Lines | Purpose |
|------|------:|---------|
| `vllm_omni/diffusion/models/krea2/krea2_transformer.py` | 438 | Full transformer port — 10 classes |
| `vllm_omni/diffusion/models/krea2/pipeline_krea2.py` | 496 | Pipeline: text encoding, CFG, denoising loop, VAE decode |
| `vllm_omni/diffusion/models/krea2/preprocess_krea2.py` | 75 | Latent packing, position IDs, VAE denormalization |
| `vllm_omni/diffusion/models/krea2/__init__.py` | 15 | Module exports |
| `vllm_omni/diffusion/registry.py` | +6 | Registry entries for model + post-process |
| `tests/diffusion/models/krea2/test_krea2_structure.py` | 294 | 15 structural/param-name tests |
| `examples/.../krea2_generate.py` | 123 | CLI example with auto distilled detection |
| `tests/diffusion/models/krea2/__init__.py` | 2 | Test package init |

## Architecture at a glance

```
Prompt
  |
  v
Qwen3VLModel (text encoder)
  |  extract hidden states from 12 decoder layers
  |  (indices 2,5,8,11,14,17,20,23,26,29,32,35)
  v
Krea2TextFusion (3-stage)
  |  1. layerwise attention blocks (2x) across layers per token
  |  2. linear projector collapses layer axis
  |  3. refiner attention blocks (2x) across token sequence
  v
Krea2Transformer2DModel (28 MMDiT blocks)
  |  - GQA: 48 Q heads, 12 KV heads, head_dim=128
  |  - 3-axis RoPE (t,h,w) dims (32,48,48), theta=1000
  |  - AdaLN with 6-channel modulation per block
  |  - SwiGLU feed-forward, sigmoid-gated attention output
  |  - Zero-centered RMSNorm (weight + 1.0)
  v
AutoencoderKLQwenImage (VAE decode)
  |  per-channel denormalization: latents * std + mean
  v
Image
```

## Key design decisions

### Separate Q/K/V linears (not fused)

Krea 2 uses individual `to_q`, `to_k`, `to_v`, `to_gate` projections — unlike
Flux which fuses into `QKVParallelLinear`. This means no `stacked_params_mapping`
in `load_weights`. Simpler weight loading, and checkpoint names match 1:1.

### RoPE compatibility

Diffusers Krea 2 produces full-dim cos/sin via `use_real=True` +
`repeat_interleave_real=True`. vLLM-Omni's `RotaryEmbedding` expects half-dim.
Solved by using `use_real=False` (half-dim cos/sin) paired with
`RotaryEmbedding(is_neox_style=False)` — mathematically equivalent, matches the
Flux pattern already in the codebase.

### Non-standard CFG formula

Standard CFG: `uncond + scale * (cond - uncond)`
Krea 2 CFG: `cond + scale * (cond - uncond)`

Handled by overriding `combine_cfg_noise` in the pipeline. This is algebraically
`(1 + scale) * cond - scale * uncond`, so guidance_scale=4.5 effectively
amplifies the conditional signal by 5.5x.

### Dynamic timestep shifting

Base checkpoint: `mu` interpolated linearly from 0.5 (at 256 image tokens) to
1.15 (at 6400 tokens) based on resolution. Distilled checkpoint: fixed
`mu=1.15`. Controlled via `is_distilled` in `model_config`.

### Distilled detection

The pipeline auto-detects distilled checkpoints in two ways:
1. Explicit `model_config.is_distilled` flag (preferred)
2. Name-based fallback: checks for `turbo`, `tdm`, or `distill` in the model name

This means `krea/Krea-2-Turbo` and `krea-ai/krea-2-medium-tdm` are both
auto-detected without needing explicit config.

### Prompt template

```
<|im_start|>system\nDescribe the image by...<|im_end|>\n<|im_start|>user\n{PROMPT}<|im_end|>\n<|im_start|>assistant\n
```

The 34-token system prefix is dropped after encoding (it conditions the
encoder's attention but isn't fed to the transformer). Max effective prompt
length is 512 tokens.

### VAE denormalization

Unlike most diffusion models that use a single `scaling_factor`, Krea 2's
Qwen-Image VAE uses per-channel `latents_mean` and `latents_std` arrays from
the VAE config. The correct formula is:

```python
z_denorm = latents * std + mean
```

**Gotcha:** the diffusers reference is easy to misread — it precomputes
`latents_std = 1.0 / config.latents_std` then *divides* by it
(`latents / latents_std`), which equals `latents * config.latents_std`.
Our initial implementation multiplied by the precomputed reciprocal instead
of dividing by it, producing a woven crosshatch grain artifact in all outputs.
See "Bugs found" section below.

## Test coverage

The structural test suite (`test_krea2_structure.py`) covers 15 tests:

- `Krea2RMSNorm` zero-centered weight initialization and forward shape
- `Krea2Attention` separate Q/K/V/gate params, no bias, correct weight names
- `Krea2SwiGLU` gate/up/down structure, no bias
- `Krea2TextFusionBlock` norm + attn + ff structure
- `Krea2TextFusion` forward shape and 3-stage param names
- `Krea2TransformerBlock` scale_shift_table and sub-module params
- `Krea2FinalLayer` bias on output linear, scale_shift_table
- Full model param names match diffusers checkpoint keys exactly
- `pack_latents` / `unpack_latents` roundtrip shapes
- `prepare_position_ids` shape and text-at-origin invariant
- Registry entries present for both model and post-process
- Import smoke test (`Krea2Pipeline`, `Krea2Transformer2DModel`, `get_krea2_post_process_func`)
- Distilled name detection (6 parametrized cases covering turbo/tdm/distill/plain names)
- `denormalize_latents` formula correctness — would have caught the `1/std` bug

## Example usage

```bash
# Base model (28 steps, guidance 4.5)
python examples/offline_inference/text_to_image/krea2_generate.py \
    --model krea/Krea-2-Raw \
    --prompt "a serene Vermont mountain lake at dawn"

# Distilled / Turbo (auto-detects from model name, adjusts steps to 8, guidance to 0)
python examples/offline_inference/text_to_image/krea2_generate.py \
    --model krea/Krea-2-Turbo

# Custom resolution
python examples/offline_inference/text_to_image/krea2_generate.py \
    --model krea/Krea-2-Raw \
    --prompt "a cup of coffee on the table" \
    --height 768 --width 1360
```

## GPU verification

Verified on an A100-80GB (a100-07 dev box) using the midstream container image
built via the nm-cicd pipeline.

### Build pipeline

Three-step build reusing the existing vLLM v0.23.0 wheel:

**Initial build (pre-denorm fix):**

| Step | Run ID | Result |
|------|--------|--------|
| vLLM wheel (reused) | `27562765781` | passed |
| vllm-omni wheel | `28063253418` | passed |
| Docker image | `28063604713` | passed |

Image: `quay.io/vllm/automation-vllm-omni:cuda-28063604713`

**Rebuild (with denorm fix + review feedback):**

| Step | Run ID | Result |
|------|--------|--------|
| vLLM wheel (reused) | `27562765781` | passed |
| vllm-omni wheel | `28126751686` | passed |
| Docker image | `28127674792` | passed |

Image: `quay.io/vllm/automation-vllm-omni:cuda-28127674792` ← current

### Container launch

```bash
nohup podman run --name krea2-api --network host \
  --device nvidia.com/gpu=7 --security-opt=label=disable \
  -e NVIDIA_VISIBLE_DEVICES=7 -e CUDA_VISIBLE_DEVICES=0 \
  -e HF_TOKEN="${HF_TOKEN}" -e HF_HOME=/hf \
  --mount type=tmpfs,target=/home/vllm/.cache \
  -v /mnt/nvme-data/dougbtv/hub_cache:/hf \
  quay.io/vllm/automation-vllm-omni:cuda-28127674792 \
  bash -c "mkdir -p /home/vllm/.cache/vllm-omni/speakers && \
           exec vllm serve krea/Krea-2-Turbo --omni --port 8000" \
  > /tmp/krea2-server.log 2>&1 &
```

Key notes:
- Model loads 713 weights (32.4 GiB) in ~12 seconds
- `torch.compile` warmup + dummy run takes ~14s additional
- Total startup: ~52 seconds to API ready
- Requires `--mount type=tmpfs,target=/home/vllm/.cache` for speaker storage init
- Dev box has `Linger=no` so `nohup` is required (detached `-d` containers get killed)

### Verification results

Both checkpoints fully verified with the rebuilt image `quay.io/vllm/automation-vllm-omni:cuda-28127674792`:

| Checkpoint | Prompt | Result |
|------------|--------|--------|
| Krea-2-Turbo (8-step distilled) | apres ski polaroid | Clean, vibrant neon jackets, no artifacts |
| Krea-2-Raw (28-step base) | Vermont mountain lake | Sharp mountain detail, smooth mist, natural reflections |

The VAE denorm fix (`latents * std + mean` instead of `latents * (1/std) + mean`)
completely eliminated the woven crosshatch grain. Both images are clean and
photorealistic.

### Test API call

```bash
curl -s http://localhost:8000/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{"model":"krea/Krea-2-Turbo",
       "prompt":"polaroid 1980s apres ski, neon jackets, steaming hot chocolate, powder day glow, retro vibes \\m/",
       "n":1,"size":"1024x1024"}'
```

### Bugs found and fixed during verification

Five issues were discovered and fixed during GPU testing:

1. **Timestep device mismatch** (`pipeline_krea2.py:342`) — scheduler timesteps
   stayed on CPU; fixed with `.to(dtype=latents.dtype, device=latents.device)`.

2. **4D attention masks** (`krea2_transformer.py:413`) — masks were expanded to
   4D with `[:, None, None, :]` but the flash attention backend asserts 2D.
   Fixed by keeping masks as `(batch_size, seq_len)` throughout.

3. **Tuple noise prediction** (`pipeline_krea2.py:371`) — `CFGParallelMixin`
   wraps transformer output in a tuple; `scheduler.step()` expects a plain
   tensor. Fixed with `if isinstance(noise_pred, tuple): noise_pred = noise_pred[0]`.

4. **Speaker storage permissions** (container config) — `serving_speech.py`
   tries to create `/home/vllm/.cache/vllm-omni/speakers` even for diffusion
   models. Fixed with tmpfs mount + `mkdir -p` in the entrypoint.

5. **VAE latent denormalization** (`preprocess_krea2.py:74`) — the formula
   `latents * (1/std) + mean` was incorrect. The diffusers reference confusingly
   precomputes `latents_std = 1.0 / config.latents_std` then divides by it
   (`latents / latents_std`), which equals `latents * config.latents_std`.
   Our code multiplied by the precomputed reciprocal instead of dividing by it.
   Fixed: `std = latents_std.view(...)` (removed `1.0 /`). Symptom was a woven
   crosshatch grain artifact visible in all generated images.

## Commit history

```
c1568475 fix: address review feedback on Krea 2 PR
13e4f413 fix: correct VAE latent denormalization formula
56b2c04b style: apply ruff formatting to krea2 files
291c6a19 [feat] Add Krea 2 text-to-image diffusion model support
```

## What's next

1. **Buildkite CI** — PR needs a maintainer label to trigger the full GPU test
   suite (GitHub Actions CI only runs pre-commit / DCO / build checks)
2. **Tensor parallelism validation** — the transformer uses plain `nn.Linear`
   everywhere; TP via `ColumnParallelLinear` / `RowParallelLinear` is a
   follow-up if perf requires it
3. **Future features** — creativity levels, style reference conditioning, and
   moodboard support once diffusers adds them upstream
