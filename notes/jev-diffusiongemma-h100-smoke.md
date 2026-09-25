# Jev-style DiffusionGemma smoke test on H100

This is the first-run sketch for [INFERENG-11113](https://redhat.atlassian.net/browse/INFERENG-11113): run the preview image built from `doug/diffusion-jev`, verify the OpenAI server, then exercise the upstream example Jev-compatible decision endpoint.

## Proven inputs

- Build image: `quay.io/vllm/automation-vllm:cuda-36142158631`
- Early-access preview image: `quay.io/vllm/rhaiis-early-access:diffusiongemma-jev`
- Published manifest: `sha256:3f653d8da80d118fd133d49a4dc5fbff97c9bcb1f33053da866cc7f5fe080afa`
- Model: `RedHatAI/diffusiongemma-26B-A4B-it-FP8-dynamic`
- Model repository cache: `/mnt/nfs-preprod-1/dougbtv/hub_cache/hub/models--RedHatAI--diffusiongemma-26B-A4B-it-FP8-dynamic`
- Model snapshot: `3b3dae4697494da5a290e9c0461954449e76c4f5`
- Host: one reserved NVIDIA H100 80 GB GPU, exposed to rootless Podman through NVIDIA CDI.

The 26 GB FP8 model is already cached. The image has already been pulled. The cache mount was preflighted from the image as its runtime UID (`2000`), so this is not relying on a download during startup.

## Start the engine

GPU 0 is currently held by a **manual** `canhazgpu` reservation. That means the detached server may be launched directly. Do not wrap a detached `podman run -d` in `canhazgpu run`: the reservation wrapper exits as soon as Podman detaches and would release its lease.

```bash
podman run -d --replace \
  --name jev-dgemma-smoke \
  --device nvidia.com/gpu=0 \
  --security-opt=label=disable \
  --ipc=host \
  -p 127.0.0.1:8000:8000 \
  -p 127.0.0.1:8011:8011 \
  -v /mnt/nfs-preprod-1/dougbtv/hub_cache/hub/models--RedHatAI--diffusiongemma-26B-A4B-it-FP8-dynamic:/model-repo:ro \
  -e CUDA_VISIBLE_DEVICES=0 \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  quay.io/vllm/automation-vllm:cuda-36142158631 \
  --model /model-repo/snapshots/3b3dae4697494da5a290e9c0461954449e76c4f5 \
  --served-model-name dgemma \
  --diffusion-config '{"canvas_length":64}' \
  --max-logprobs 32 \
  --enable-prefix-caching \
  --max-num-seqs 4 \
  --gpu-memory-utilization 0.85 \
  --enforce-eager \
  --host 0.0.0.0 --port 8000
```

`--security-opt=label=disable` is deliberate: the model cache is on NFS. The service is bound only to loopback; use SSH forwarding if it needs to be reached off-host. Mount the model repository rather than just its `snapshots/` directory: snapshot files are symlinks into the repository's `blobs/` directory. Serving the snapshot path directly also avoids an offline Hugging Face cache lookup that this preview image did not resolve.

## Verify the engine

```bash
podman logs -f jev-dgemma-smoke
curl -fsS http://127.0.0.1:8000/health
```

Wait for `Application startup complete` in the logs before proceeding.

## Start the example decision server

The `/v1/systemone` API is an upstream example adapter, not a stable vLLM API. The diffusion read machinery itself is in vLLM core. This preview image does not include vLLM's `examples/` tree, so copy the matching branch's adapter script into the running container first.

```bash
podman cp \
  /path/to/nm-vllm-ent/examples/features/structured_diffusion/structured_server.py \
  jev-dgemma-smoke:/tmp/jev-structured-server.py

podman exec jev-dgemma-smoke sh -lc 'nohup python3 /tmp/jev-structured-server.py --upstream http://127.0.0.1:8000 --tokenizer /model-repo/snapshots/3b3dae4697494da5a290e9c0461954449e76c4f5 --canvas 64 >/tmp/structured-server.log 2>&1 &'

until curl -fsS http://127.0.0.1:8011/health >/dev/null; do sleep 2; done
```

## Demo: probabilistic incident-triage gate

This is deliberately a decision workload, not an ersatz chatbot. The same small schema is reusable across events, making prefix caching useful. Each `noul` answer returns a probability rather than fabricated certainty; a caller can use thresholds to page, route to a queue, or request a human review.

```bash
curl -sS http://127.0.0.1:8011/v1/systemone \
  -H 'content-type: application/json' \
  -d '{
    "model": "jev-latest",
    "state": {
      "service": "checkout-api",
      "environment": "production",
      "symptom": "HTTP 503 rate rose from 0.1% to 23% after deploy 2026.09.25.4; payment completions are failing; rollback is available.",
      "recent_change": "enabled a new payment-provider retry path"
    },
    "questions": {
      "urgent": {
        "type": "noul",
        "instructions": "Does this need an on-call response within 15 minutes?"
      },
      "customer_impact": {
        "type": "noul",
        "instructions": "Are customers currently unable to complete purchases?"
      },
      "needs_human": {
        "type": "noul",
        "instructions": "Should a human incident commander review this before an automated action?"
      }
    }
  }' | jq .
```

For a real integration, keep the action policy separate and auditable. An illustrative policy is: page on-call only when `urgent >= 0.85`; open an incident room when both `urgent` and `customer_impact` cross their thresholds; otherwise create a review item. Do not let this preview endpoint autonomously roll back production.

### Validated result (2026-09-25)

On one H100 80 GB, the request above returned HTTP 200 in 15.88 seconds after initial kernel warmup:

- `urgent`: 0.9859 yes
- `customer_impact`: 0.9983 yes
- `needs_human`: 0.6283 yes

The normal OpenAI chat endpoint also returned `Paris` for the one-word capital-of-France smoke request. Do not send `temperature` with diffusion requests: this model rejects it as unsupported.

## Observed preview-image caveats

- The image logs a missing `GLIBCXX_3.4.31` for optional DeepSelect/DeepGEMM extensions. It chose the supported Cutlass FP8 MoE fallback and served both the chat and decision tests.
- NFS model loading worked for this H100 smoke, but took 174 seconds for the 25.33 GiB checkpoint. Use local disk for a repeatable performance or production validation.

## Cleanup

```bash
podman stop jev-dgemma-smoke
podman rm jev-dgemma-smoke
# Release the manual GPU reservation when the session is actually finished.
canhazgpu release
```
