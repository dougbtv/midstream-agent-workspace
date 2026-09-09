# AMD Accelerator Cloud — MI355X Access Notes

> **Date:** 2026-04-17
> **Author:** Doug Smith
> **JIRA:** [INFERENG-6013](https://redhat.atlassian.net/browse/INFERENG-6013) (access), [INFERENG-6074](https://redhat.atlassian.net/browse/INFERENG-6074) (runner setup)
> **Parent:** [INFERENG-5685](https://redhat.atlassian.net/browse/INFERENG-5685) (ROCm v0.18.0 validation)
> **Status:** Runner deployed on gpu-33, accepting jobs

---

## Network Topology

```
Laptop
  └── ssh amd@207.148.13.85          ← jump host (bastion)
        ├── ssh mi355-gpu-33          ← RHEL 9.7,  8x MI355X
        └── ssh mi355-gpu-35          ← RHEL 9.6, 8x MI355X
```

**Bastion host:** `207.148.13.85` — AMD Accelerator Cloud (AAC)
**Auth:** SSH key (provisioned after AMD backend auth migration, confirmed working 2026-04-17)
**Docs:** https://aac.amd.com/help/bare-metal/aac-user-guide/

### GPU Nodes

| Node | OS | GPUs | VRAM | ROCm |
|------|----|------|------|------|
| `mi355-gpu-33` | RHEL 9.7 | 8x MI355X | ~1.12 TiB (8x 144 GB) | 7.1.1 |
| `mi355-gpu-35` | RHEL 9.6 | 8x MI355X | ~1.12 TiB (8x 144 GB) | 7.1.1 |

Total across both nodes: **2.25 TiB VRAM** (per aoguntayo's `amd-smi` output).

GPU arch: **gfx950**

### Quick VRAM check
```bash
amd-smi | awk 'match($0, /([0-9]+)\/([0-9]+) MB/, a) {sum += a[2]} END {printf "Total VRAM: %d MB (%.2f TiB)\n", sum, sum/1024/1024}'
```

---

## First Login Sequence

```bash
# 1. SSH to bastion
ssh -i <private-key> amd@207.148.13.85

# 2. Hop to GPU node (from bastion)
ssh mi355-gpu-33    # RHEL 9.7 (runner deployed here)
# or
ssh mi355-gpu-35    # RHEL 9.6
```

---

## Hardware Recon Checklist (first 5 minutes)

Run these on whichever GPU node you land on:

```bash
# Identity
hostname && uname -a && cat /etc/os-release | head -5

# GPU detection
amd-smi list
rocminfo | grep -E "gfx|Name|Marketing"

# ROCm version
/opt/rocm/bin/rocm_agent_enumerator
apt list --installed 2>/dev/null | grep rocm-core || rpm -qa | grep rocm-core

# Group membership (need render + video for GPU access)
groups
ls -la /dev/dri/render*
ls -la /dev/kfd

# Disk space (need ~200GB for model cache)
df -h

# Network (needed for GH runner, HF downloads, GCS wheels)
curl -s -o /dev/null -w "%{http_code}" https://api.github.com
curl -s -o /dev/null -w "%{http_code}" https://huggingface.co
curl -s -o /dev/null -w "%{http_code}" https://storage.googleapis.com

# Python / torch pre-installed?
which python3 && python3 --version
python3 -c "import torch; print(f'torch {torch.__version__}, CUDA avail: {torch.cuda.is_available()}')" 2>/dev/null

# Sudo?
sudo -n true 2>/dev/null && echo "sudo: yes" || echo "sudo: no (or needs password)"

# Other users / shared box?
who
```

---

## SDPA Sanity Check (first 10 minutes)

Minimal GPU compute test — confirms `gfx950` + ROCm 7.1 attention path works:

```python
import torch

device = torch.device("cuda")
print(f"Device: {torch.cuda.get_device_name()}")
print(f"Capability: {torch.cuda.get_device_capability()}")
print(f"GPU count: {torch.cuda.device_count()}")

B, H, S, D = 2, 4, 64, 64
q = torch.randn(B, H, S, D, device=device, dtype=torch.float16)
k = torch.randn(B, H, S, D, device=device, dtype=torch.float16)
v = torch.randn(B, H, S, D, device=device, dtype=torch.float16)

with torch.no_grad():
    out = torch.nn.functional.scaled_dot_product_attention(q, k, v, is_causal=True)
    print(f"OK: shape={out.shape}, nan={torch.isnan(out).any()}")
```

> **Note:** The AIPCC probe tests hit `hipErrorInvalidValue` on MI355X with raw SDPA (see ROCM7_VALIDATION.md section 9). vLLM uses different attention backends (aiter/rocm_attn), so this may fail without being a blocker — but good to know.

---

## Runner Setup (INFERENG-6074)

Full scripts in: [`stratus/misc/mi355x/`](https://github.com/neuralmagic/stratus/pull/1041)

| Script | Purpose |
|--------|---------|
| `setup.sh` | One-time provisioning (runner user, GH runner binary, registration, `/model-cache`) |
| `start-runner.sh` | Start runner in screen session with ROCm env vars + GPU visibility |
| `stop-runner.sh` | Stop the runner |
| `env-rocm71.sh` | Sourceable ROCm 7.1 env vars (mirrors nm-cicd `setup_env_test_vllm.sh`) |
| `verify-hw.sh` | Hardware sanity checks (gfx950, amd-smi, group membership) |
| `README.md` | Full runbook |

### Quick setup (~10 min)

```bash
# Get registration token
gh api -X POST orgs/neuralmagic/actions/runners/registration-token --jq .token

# Run setup as root
sudo ./setup.sh --token <TOKEN>

# Verify hardware
./verify-hw.sh

# Start runner (all GPUs, or limit: ./start-runner.sh 2)
./start-runner.sh

# Confirm online
# https://github.com/organizations/neuralmagic/settings/actions/runners
```

**Runner label:** `mi355x-static` (in `runner_group_b`)
**Deployed on:** `mi355-gpu-33` (2026-04-17)
**Runner binary:** v2.333.1
**Upgrade path:** → `os-mi355x-solo-rocm71` when hardware becomes permanent

### Deployment notes (2026-04-17)

- No `git` on the nodes — had to `scp` scripts via bastion (two-hop: laptop→bastion→gpu-33)
- `verify-hw.sh` had a bash bug: `((PASS++))` fails under `set -e` when PASS=0 (0 is falsy). Fixed to use `PASS=$((PASS + 1))`.
- `setup upterm session` step in `debug.yml` fails on bare metal (needs WebSocket relay). Not a real problem — checkout, python, GCP auth all pass.
- NUMA balancing must be disabled manually: `sudo sh -c 'echo 0 > /proc/sys/kernel/numa_balancing'`
- `rocminfo` shows 24 GPU agents (8 physical × 3 entries each — CPU agent + 2 GPU sub-agents per device)

### Bare metal CI gotchas (learned the hard way)

- **Runner user has no passwordless sudo.** The `runner` user created by `setup.sh` does NOT get a sudoers entry. Any CI action that does `sudo <pkg-manager> install` will fail with "a password is required". Fix: pre-install everything in `setup.sh` (which runs as root), and make actions skip install when deps are present.
- **`nm-actions/install-automation-components`** only knows `apt-get` and `microdnf`. `setup.sh` creates a microdnf shim (`/usr/local/bin/microdnf → dnf`) and pre-installs all packages so microdnf is a no-op. The shim is needed because nm-actions is an external repo we don't control (and it's effectively dead — INFERENG-6190 tracks removing it).
- **`install-guidellm-system-deps`** had the same problem but worse — its microdnf branch was literally `echo "TODO!"`. Fixed in nm-cicd to: (1) check `apt-get` → `dnf` → `microdnf`, (2) skip package install entirely if deps are already present, (3) install yq to `~/.local/bin` if `/usr/bin` isn't writable.
- **Runner `.path` file** controls PATH for all job steps (not the shell PATH). Default omits `/usr/local/bin`, breaking the microdnf shim and gh CLI. `setup.sh` now patches it.

---

## ROCm 7.1 Environment Variables

Source these or use `env-rocm71.sh`:

| Variable | Value | Why |
|----------|-------|-----|
| `HSA_NO_SCRATCH_RECLAIM` | `1` | Required for RCCL in ROCm 7.1 |
| `VLLM_USE_TRITON_FLASH_ATTN` | `0` | Triton flash attn disabled on ROCm |
| `NCCL_MIN_NCHANNELS` | `112` | ROCm perf tuning |
| `TORCH_BLAS_PREFER_HIPBLASLT` | `1` | Prefer hipBLASLt |
| `VLLM_WORKER_MULTIPROC_METHOD` | `spawn` | Ray multiprocessing workaround |
| `HIP_FORCE_DEV_KERNARG` | `1` | ROCm kernel arg fix |
| `RAY_EXPERIMENTAL_NOSET_HIP_VISIBLE_DEVICES` | `1` | Ray >= 2.45 compat |
| `VLLM_ROCM_USE_AITER` | `0` | Off by default (flip to `1` for AITER pass) |

---

## Validation Game Plan (once on the box)

### Phase 1: Recon (5 min)
- Run hardware recon checklist above
- Report: topology, GPU count, ROCm version, disk, network, sudo, what's pre-installed

### Phase 2: Smoke test (10 min)
- SDPA sanity script
- `amd-smi` GPU utilization during test

### Phase 3: Assess runner feasibility (10 min)
- Can we install a GH Actions runner? (internet, disk, sudo)
- Is there a `/model-cache` or equivalent?
- Shared box considerations

### Phase 4: Runner deployment (10 min)
- Run `setup.sh` from stratus PR #1041
- `verify-hw.sh` + `start-runner.sh`
- Confirm runner appears in GitHub org settings

### Phase 5: First accept-sync on MI355X
```bash
gh workflow run accept-sync.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/sync-v0.18.0 \
  -f wf_category=DEBUG \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=doug/v0.18.0-rocm7-gpt-oss-fix \
  -f python=3.12 \
  -f target_device=rocm \
  -f model_validation_configs_ref=main
```

### Phase 6: Promote gfx950 wheels (INFERENG-6120)
Once smoke tests pass on MI355X:
```bash
gsutil cp gs://nm-gha-cache/ROCm/aiter/assets/24473484344/*.whl gs://nm-public-pypi/dist/
gsutil cp gs://nm-gha-cache/Dao-AILab/flash-attention/assets/24472917287/*.whl gs://nm-public-pypi/dist/
```

---

## Key Unknowns — RESOLVED

| Question | Answer |
|----------|--------|
| Is this a shared box? | Yes, other team members (Tarun, aoguntayo) also have access. Coordinate usage. |
| Do we have sudo? | Yes, passwordless on both nodes |
| Is Python/torch pre-installed? | Python 3.9.25 (system). No torch — workflows install via uv venv. |
| Disk space for model cache? | 6.8 TB free on `/home`. `/model-cache` created by setup.sh. |
| Internet access for GH runner? | Yes — GitHub API and HuggingFace both return 200 |
| Is there an existing venv or conda? | No — clean system |
| Which node should we use? | Using `gpu-33` (RHEL 9.7). `gpu-35` (RHEL 9.6) available as backup. |
| No git on nodes | Scripts must be scp'd via bastion. `setup.sh` installs git as a dep. |

---

## Containerized vLLM on MI355X (Tarun's validated commands)

Tarun ran these on the MI355X nodes (INFERENG-6111, 2026-04-16). Both models started successfully with `ROCM_AITER_UNIFIED_ATTN` attention backend on gfx950.

### Podman run template

```bash
sudo podman run --rm -it \
  --device /dev/kfd --device /dev/dri \
  --security-opt=label=disable \
  --group-add keep-groups \
  --shm-size=4GB -p 8000:8000 \
  --env "HF_HOME=/cache" \
  --env "HF_HUB_OFFLINE=0" \
  -v ./model:/cache \
  -v ./rhaii-cache:/opt/app-root/src/.cache \
  <IMAGE> \
  --model <MODEL> \
  --tensor-parallel-size <N>
```

### Validated configurations

| Model | TP | Image | Memory | KV cache | Startup |
|-------|----|-------|--------|----------|---------|
| `openai/gpt-oss-20b` | 1 | `quay.io/vllm/automation-vllm:rocm-24462782547` | 15.0 GiB | 240.72 GiB (5.2M tokens) | ~77s |
| `openai/gpt-oss-120b` | 2 | `quay.io/vllm/automation-vllm:rocm-24462782547` | 36.46 GiB | 219.23 GiB (6.4M tokens) | ~114s |

### Observations from Tarun's runs

- **Attention backend:** `ROCM_AITER_UNIFIED_ATTN` selected automatically (not `TRITON_ATTN`). AITER works on gfx950.
- **NUMA balancing warning:** Both runs show `[aiter] WARNING: NUMA balancing is enabled`. Fix:
  ```bash
  sudo sh -c 'echo 0 > /proc/sys/kernel/numa_balancing'
  ```
- **HF cache permission errors:** Non-fatal `Permission denied` on `.no_exist/` and `refs/main` writes inside `/cache`. The volume mount permissions don't allow the container user to write cache metadata. Noisy but harmless — models load fine.
- **torch.compile worked:** Both runs completed `torch.compile` + CUDA graph capture without errors.
- **Hostname:** Prompt shows `amd@216` — GPU nodes resolve to Vultr IPs internally (`104.238.164.15.vultrusercontent.com` for gpu-33, `216.128.145.8.vultrusercontent.com` for gpu-35).
- **nccl version:** `2.27.7` (from TP=2 run logs)
- **Container runtime:** `sudo podman` (not docker). Requires `--device /dev/kfd --device /dev/dri` for GPU passthrough and `--security-opt=label=disable --group-add keep-groups` for SELinux/permissions.

### Original crash (before fix) — for reference

The AIPCC RC image (`quay.io/aipcc/rhaiis/rocm-ubi9:3.4.0-1776144316`) crashed on gpt-oss-20b with:
```
ImportError: cannot import name 'GFX950MXScaleLayout' from 'triton_kernels.tensor_details.layout'
```
This was the Triton 3.6 rename bug (INFERENG-6111), now fixed.

---

## People

| Person | Context |
|--------|---------|
| egallen | Shared SSH access info in Slack |
| aoguntayo | First to access, documented SSH hop sequence |
| Landon LaSmith (llasmith) | Coordinating RHEL install on `gpu-35` with AMD |
| Selbi (heyselbi) | Leading ROCm 7.1 / v0.18.0 sync |
| Tarun Kumar | Validated GPT-OSS fix on MI355X (INFERENG-6111) |

---

## Session Log

_Record what you find during each session here._

### Session 1 — 2026-04-17

**Result: Full access confirmed, zero blockers.**

Validated SSH chain: laptop → bastion (`aac14-ele-priv-ctl-9`) → both GPU nodes.

| Check | gpu-33 | gpu-35 |
|-------|--------|--------|
| SSH | OK | OK |
| OS | RHEL 9.7 (Plow) | RHEL 9.6 (Plow) |
| GPUs | 8x gfx950 | 8x gfx950 |
| ROCm | 7.1.1 (`rocm-core-7.1.1.70101-38.el8`) | 7.1.1 |
| Groups | wheel, video, render | wheel, video, render |
| Sudo | passwordless | passwordless |
| /home free | 6.8 TB | 6.8 TB |
| Podman | 5.6.0 | 5.4.0 |
| /dev/kfd | present (rw) | present |
| /dev/dri | 16+ render nodes | present |
| GitHub API | 200 | 200 |
| HuggingFace | 200 | 200 |
| Python | 3.9.25 (system) | present |
| Hostname | `104.238.164.15.vultrusercontent.com` | `216.128.145.8.vultrusercontent.com` |

**Note:** `gpu-35` was previously Ubuntu 22.04 — now RHEL 9.6 (Landon's RHEL install completed).

### Session 2 — 2026-04-17 (runner deployment)

**Result: GH Actions runner deployed and accepting jobs on gpu-33.**

1. SCP'd `stratus/misc/mi355x/` scripts to gpu-33 via bastion (no git on node)
2. `sudo ./setup.sh --token <TOKEN>` — runner v2.333.1 installed, registered as `mi355x-static` in `runner_group_b`
3. `runner` user created with video + render groups, `/model-cache` created
4. Disabled NUMA balancing (`echo 0 > /proc/sys/kernel/numa_balancing`)
5. `./start-runner.sh` — screen session `github-runner` running with all 8 GPUs visible
6. Dispatched `debug.yml` targeting `label=mi355x-static` — run [24571272043](https://github.com/neuralmagic/nm-cicd/actions/runs/24571272043)
7. Job picked up successfully. Checkout, uv, python install, GCP auth all passed. Only `upterm` step failed (expected on bare metal).

**Bug found:** `verify-hw.sh` crashes on first check — `((PASS++))` with `set -e` when PASS=0 evaluates to falsy. Fixed: `PASS=$((PASS + 1))`. Committed to stratus PR #1041.
