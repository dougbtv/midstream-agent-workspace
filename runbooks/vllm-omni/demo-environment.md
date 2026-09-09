# Qwen3-Omni demo environment for Yuchen

Validated on August 31, 2026 for [INFERENG-10280](https://redhat.atlassian.net/browse/INFERENG-10280).

## Environment

- Host: `h100-03.nemg-001.lab.rdu2.dc.redhat.com` (`10.14.217.25`)
- API: `http://10.14.217.25:8091/v1`
- Model: `Qwen/Qwen3-Omni-30B-A3B-Instruct`
- Image: `quay.io/vllm/automation-vllm-omni:cuda-33069069707`
- Image digest: `sha256:eb174313c757d45841cd4732bf2c9954f823cb489d7d2962f0defa8de0729267`
- Hardware: two H100 80 GB GPUs
- Container name: `yuchen-qwen3-omni-demo`
- Model cache: `/home/dougbtv/hub_cache/qwen3-omni-demo`

The bundled production deploy configuration places the thinker on the first GPU and the talker plus code-to-wave stages on the second GPU.

## One-time host setup

Enable lingering so a rootless Podman container survives the SSH session that launched it:

```bash
loginctl enable-linger "$USER"
```

Rootless NVIDIA CDI requires disabled SELinux labeling for the container process on this host. Keep `--security-opt=label=disable` in the Podman command; without it, PyTorch reports zero CUDA devices.

The host has 384 CPU cores. Limit OpenMP/BLAS thread pools or the third model stage fails with `libgomp: Thread creation failed: Resource temporarily unavailable`.

## Reserve GPUs

Check availability and reserve two GPUs for the demo window:

```bash
canhazgpu status
canhazgpu reserve \
  --gpu-ids 6,7 \
  --duration 3h \
  --nonblock \
  --note "INFERENG-10280 Yuchen Omni demo"
```

Adjust both the reservation and the two Podman `--device` values together if different GPUs are used. CDI remaps the selected physical GPUs to logical devices `0` and `1` inside the container, matching the bundled deploy configuration.

## Start the server

```bash
mkdir -p /home/dougbtv/hub_cache/qwen3-omni-demo

podman pull quay.io/vllm/automation-vllm-omni:cuda-33069069707

podman run -d \
  --name yuchen-qwen3-omni-demo \
  --pull=never \
  --security-opt=label=disable \
  --pids-limit=4096 \
  --device nvidia.com/gpu=6 \
  --device nvidia.com/gpu=7 \
  --ipc=host \
  -p 8091:8091 \
  -e HF_HUB_OFFLINE=0 \
  -e HF_HOME=/home/vllm/.cache/huggingface \
  -e OMP_NUM_THREADS=8 \
  -e MKL_NUM_THREADS=8 \
  -e OPENBLAS_NUM_THREADS=8 \
  -e NUMEXPR_NUM_THREADS=8 \
  -v /home/dougbtv/hub_cache/qwen3-omni-demo:/home/vllm/.cache/huggingface \
  quay.io/vllm/automation-vllm-omni:cuda-33069069707 \
  --model Qwen/Qwen3-Omni-30B-A3B-Instruct \
  --host 0.0.0.0 \
  --port 8091 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

Important details:

- The image sets `HF_HUB_OFFLINE=1`; override it to `0` for the initial model download.
- This image starts the raw vLLM-Omni API module. Use `--model MODEL`, not a positional model argument. A positional argument silently leaves the default `Qwen/Qwen3-0.6B` selected.
- The initial model download is about 70.5 GB. The validated download took about 90 seconds on this host.
- First startup takes several minutes while all three stages load and capture CUDA graphs.
- The `0.24.1...spyre` version text printed at startup is stale `setuptools_scm` metadata, tracked by INFERENG-8004. The image contains and successfully runs the CUDA extensions.

Watch startup:

```bash
podman logs -f yuchen-qwen3-omni-demo
```

Readiness is complete after the log shows all three stages initialized and `Application startup complete`.

## Open the VPN-facing port

The server listens on all interfaces, but firewalld blocks port 8091 by default. Run this manually with sudo:

```bash
sudo firewall-cmd --add-port=8091/tcp
```

That is a runtime-only rule and disappears after reboot or firewalld reload. Make it persistent only if this becomes a longer-lived demo:

```bash
sudo firewall-cmd --permanent --add-port=8091/tcp
sudo firewall-cmd --reload
```

The API has no authentication configured. Expose it only on the trusted Red Hat lab/VPN network.

## Validate

On the host:

```bash
curl -fsS http://127.0.0.1:8091/health
curl -fsS http://127.0.0.1:8091/v1/models | jq -r '.data[].id'
```

From another VPN-connected machine, replace `127.0.0.1` with `10.14.217.25`.

Small end-to-end text and audio generation check:

```bash
curl -fsS http://10.14.217.25:8091/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "Qwen/Qwen3-Omni-30B-A3B-Instruct",
    "messages": [
      {"role": "user", "content": "Reply with exactly: demo ready"}
    ],
    "max_tokens": 16,
    "temperature": 0
  }' \
  | jq '{
      text: .choices[0].message.content,
      has_audio: (.choices[0].message.audio.data | length > 0)
    }'
```

Validated result:

```json
{
  "text": "demo ready",
  "has_audio": true
}
```

## Operations and cleanup

```bash
# Status and logs
podman ps --filter name=yuchen-qwen3-omni-demo
podman logs --tail 100 yuchen-qwen3-omni-demo

# Stop/start without re-downloading the model
podman stop yuchen-qwen3-omni-demo
podman start yuchen-qwen3-omni-demo

# Remove the container after stopping it; the model cache remains
podman rm yuchen-qwen3-omni-demo

# Release Doug's manual GPU reservations when the demo is finished
canhazgpu release
```

Do not delete `/home/dougbtv/hub_cache/qwen3-omni-demo` unless the roughly 70.5 GB model cache is no longer needed.
