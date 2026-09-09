# Gemma 4 31B: Build, Run, and Smoke Test Guide

## Overview

This document covers how to build, deploy, and test the `google/gemma-4-31B-it` model using nm-vllm-ent (based on upstream vLLM v0.19.1) on NVIDIA A100 GPUs.

## Build Information

| Field | Value |
|-------|-------|
| Model | `google/gemma-4-31B-it` |
| Container Image | `quay.io/vllm/automation-vllm:cuda-24587997386` |
| Version Tag | `quay.io/vllm/automation-vllm:0.18.1.dev517_rhaiv.2.g9a9df285f` |
| Commit Tag | `quay.io/vllm/automation-vllm:cuda-2070b35ba8f93698374299c63b588f55494209b7` |
| nm-vllm-ent Branch | `doug/v0.19.1` (upstream `releases/v0.19.1` merged into `origin/main`) |
| nm-cicd Branch | `doug/v0.19.1` |
| GH Actions Run ID | `24587997386` |
| Target Device | CUDA |
| Python Version | 3.12 |

## Build Commands

### Full build (wheel + image)

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/v0.19.1 \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=doug/v0.19.1 \
  -f build_label=k8s-a100-build-12-9 \
  -f build_timeout=120 \
  -f image_label=ibm-wdc-k8s-h100-dind \
  -f python=3.12 \
  -f release_image=false \
  -f target_device=cuda
```

### Image only (reuse existing wheel)

```bash
gh workflow run build-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/v0.19.1 \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=doug/v0.19.1 \
  -f build_label=ibm-wdc-k8s-h100-dind \
  -f release_image=false \
  -f run_id=24587997386 \
  -f target_device=cuda
```

## Prerequisites

- Two A100 80GB GPUs (or two H100 80GB GPUs) with tensor parallelism of 2
- Model files downloaded locally (not NFS -- rootless podman UID remapping breaks NFS)
- Podman with NVIDIA CDI support
- HuggingFace token with access to `google/gemma-4-31B-it`

## Deployment Steps

### 1. Reserve GPUs

```bash
chg reserve -G 0,1 -d 4h
```

### 2. Download the model

```bash
HF_HOME=/home/$USER/.cache/huggingface \
HF_TOKEN=$HF_TOKEN \
uv tool run --from huggingface_hub hf download google/gemma-4-31B-it
```

### 3. Pull the image

```bash
podman pull quay.io/vllm/automation-vllm:cuda-24587997386
```

### 4. Start the server

```bash
podman run -d \
  --name vllm-gemma4 \
  --device nvidia.com/gpu=0 \
  --device nvidia.com/gpu=1 \
  --security-opt=label=disable \
  --shm-size=10g \
  -p 8000:8000 \
  -v /home/$USER/.cache/huggingface:/hf:Z \
  -e HF_HUB_OFFLINE=1 \
  -e FLASHINFER_DISABLE_VERSION_CHECK=1 \
  -e HF_HOME=/hf \
  -e CUDA_VISIBLE_DEVICES=0,1 \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  quay.io/vllm/automation-vllm:cuda-24587997386 \
    --model google/gemma-4-31B-it \
    --tensor-parallel-size 2 \
    --max-model-len 4096 \
    --gpu-memory-utilization 0.85 \
    --enforce-eager \
    --trust-remote-code \
    --host 0.0.0.0 --port 8000
```

### 5. Watch logs for startup

```bash
podman logs -f vllm-gemma4
```

Wait for `Application startup complete.` (takes ~2-3 minutes on A100s).

## Smoke Tests

### Health check

```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/health
# Expected: 200
```

### Chat completion

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-31B-it",
    "messages": [{"role": "user", "content": "Hello, briefly introduce yourself."}],
    "max_tokens": 128
  }'
```

### Text completion

```bash
curl -s http://127.0.0.1:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-31B-it",
    "prompt": "The capital of France is",
    "max_tokens": 32
  }'
```

## Cleanup

```bash
podman stop vllm-gemma4 && podman rm vllm-gemma4
chg release -G 0,1
```

## Notes and Gotchas

- **`--enforce-eager` is required on A100s.** Without it, CUDA graph profiling OOMs during the `determine_available_memory` phase, even at `--max-model-len 4096` and `--gpu-memory-utilization 0.85`. The model takes ~30 GiB across 2 GPUs, leaving very little headroom for CUDA graph capture. H100s with their larger memory may not need this flag.
- **Start with `--max-model-len 4096` for smoke testing.** The model supports up to 131072, but large context lengths will OOM on 2x A100s. Scale up once you've confirmed the model loads.
- **Always use local disk for the HF cache**, not NFS. Rootless podman UID remapping causes permission denied on NFS.
- **Use `127.0.0.1`, not `localhost`** for curl. Some hosts try IPv6 first and fail.
- **Don't use `--rm`** on the container if you need to debug crashes -- you lose the logs.
- **HF_HUB_OFFLINE=1** is baked into the automation-vllm images. The model must be fully downloaded before starting the container.
