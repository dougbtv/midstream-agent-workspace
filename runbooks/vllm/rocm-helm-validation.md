# ROCm 7 Helm-based Model Validation (INFERENG-6085)

> **Date:** 2026-04-15
> **Author:** Doug Smith
> **Status:** Ready to deploy

## What and why

Deploy `RedHatAI/Qwen3-8B-FP8-dynamic` on the MI300X OCP cluster using
`quay.io/vllm/vllm-rocm:0.18.0` (ROCm 7) via the RHAIIS Helm chart.
This provides early ROCm 7 serving validation while MI355X access is blocked.

**Values file:** `rhaiis-midstream-snippets/charts/deployments/qwen3-8b-rocm7.yml`

| Setting | Value | Rationale |
|---------|-------|-----------|
| Image | `quay.io/vllm/vllm-rocm:0.18.0` | ROCm 7 image, includes updated AITER |
| Model | `RedHatAI/Qwen3-8B-FP8-dynamic` | 8B FP8, fits on 1 GPU, in OCP validation config |
| GPUs | 1x MI300X | Single GPU sufficient for 8B FP8 (~8GB weights) |
| Memory | 64Gi | Generous for an 8B model |
| Model cache | Shared `modelcache-pvc` | Same pattern as gpt-oss-120b deployment |
| AITER | Off initially | Validate baseline first, flip to `1` for second pass |

---

## Prerequisites

### CLI tools

```bash
which oc helm   # both required
```

### Cluster credentials

From the `secrets` file (sourced from GCP Secret Manager `openshift-mi300x-user-cred`):

```bash
export OCP_USER="htpasswd-cluster-admin-user"
export OCP_PASSWORD="<from secrets file>"
export HF_TOKEN="<from secrets file>"
```

---

## Step 1: Login to cluster

```bash
oc login https://api.ods-az-amd-02-pool-5smt9.azure.rh-ods.com:6443 \
  -u "$OCP_USER" \
  -p "$OCP_PASSWORD" \
  --insecure-skip-tls-verify
```

## Step 2: Find the right namespace

```bash
# Find where modelcache-pvc lives
oc get pvc --all-namespaces | grep modelcache

# Set it
export NS=<namespace-from-above>
```

## Step 3: Verify cluster readiness

```bash
# Service account exists?
oc get sa vllm-anyuid-sa -n "$NS"

# If missing:
# oc create serviceaccount vllm-anyuid-sa -n "$NS"
# oc adm policy add-scc-to-user anyuid system:serviceaccount:$NS:vllm-anyuid-sa

# GPU availability
oc describe nodes | grep -A5 "amd.com/gpu"

# cert-manager installed? (needed for route TLS)
oc get crd certificates.cert-manager.io

# PriorityClass exists? (chart uses low-priority)
oc get priorityclass low-priority
```

## Step 4: Create HF token secret

```bash
oc create secret generic hf-token \
  --from-literal=token="$HF_TOKEN" \
  -n "$NS"
```

## Step 5: Dry run

```bash
helm install qwen3-8b-rocm7 \
  rhaiis-midstream-snippets/charts/vllm/ \
  -f rhaiis-midstream-snippets/charts/deployments/qwen3-8b-rocm7.yml \
  -n "$NS" \
  --dry-run=server
```

Check the rendered output for:
- Image is `quay.io/vllm/vllm-rocm:0.18.0`
- `amd.com/gpu: 1` in resources
- All ROCm 7.1 env vars present
- `HF_TOKEN` references `hf-token` secret
- `modelcache-pvc` volume mounted at `/model-cache`

## Step 6: Install

```bash
helm install qwen3-8b-rocm7 \
  rhaiis-midstream-snippets/charts/vllm/ \
  -f rhaiis-midstream-snippets/charts/deployments/qwen3-8b-rocm7.yml \
  -n "$NS"
```

---

## Post-install: Monitor deployment

```bash
# Watch pod status
oc get pods -n "$NS" -l app.kubernetes.io/instance=qwen3-8b-rocm7 -w

# Get pod name
POD=$(oc get pods -n "$NS" -l app.kubernetes.io/instance=qwen3-8b-rocm7 \
  -o jsonpath='{.items[0].metadata.name}')

# Init container logs (model download)
oc logs "$POD" -c download-model -n "$NS" -f

# Main container logs (vLLM startup)
oc logs "$POD" -c vllm -n "$NS" -f
# Look for: "Application startup complete."

# Pod events (if stuck)
oc describe pod "$POD" -n "$NS" | tail -30
```

## Post-install: Validate inference

```bash
# Get route hostname
ROUTE=$(oc get route -n "$NS" \
  -l app.kubernetes.io/instance=qwen3-8b-rocm7 \
  -o jsonpath='{.items[0].spec.host}')
API_KEY="rocm7-validation-key"

# Health check
curl -s "https://$ROUTE/health"

# List models
curl -s "https://$ROUTE/v1/models" \
  -H "Authorization: bearer $API_KEY" | jq .

# Chat completion
curl -s "https://$ROUTE/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: bearer $API_KEY" \
  -d '{
    "model": "RedHatAI/Qwen3-8B-FP8-dynamic",
    "messages": [{"role": "user", "content": "What is 2+2? Answer briefly."}],
    "max_tokens": 50,
    "temperature": 0.0
  }' | jq .

# Streaming
curl -s "https://$ROUTE/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: bearer $API_KEY" \
  -d '{
    "model": "RedHatAI/Qwen3-8B-FP8-dynamic",
    "messages": [{"role": "user", "content": "Explain what vLLM is in one sentence."}],
    "max_tokens": 100,
    "stream": true
  }'

# Completions endpoint
curl -s "https://$ROUTE/v1/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: bearer $API_KEY" \
  -d '{
    "model": "RedHatAI/Qwen3-8B-FP8-dynamic",
    "prompt": "The capital of France is",
    "max_tokens": 20,
    "temperature": 0.0
  }' | jq .

# Helm test (wget against route)
helm test qwen3-8b-rocm7 -n "$NS"
```

## AITER validation (second pass)

Edit `qwen3-8b-rocm7.yml` and change `VLLM_ROCM_USE_AITER` from `'0'` to `'1'`, then:

```bash
helm upgrade qwen3-8b-rocm7 \
  rhaiis-midstream-snippets/charts/vllm/ \
  -f rhaiis-midstream-snippets/charts/deployments/qwen3-8b-rocm7.yml \
  -n "$NS"

# Watch pod restart
oc get pods -n "$NS" -l app.kubernetes.io/instance=qwen3-8b-rocm7 -w

# Check for AITER messages in logs
POD=$(oc get pods -n "$NS" -l app.kubernetes.io/instance=qwen3-8b-rocm7 \
  -o jsonpath='{.items[0].metadata.name}')
oc logs "$POD" -c vllm -n "$NS" | grep -i aiter

# Re-run smoke tests from above
```

## Cleanup

```bash
helm uninstall qwen3-8b-rocm7 -n "$NS"
oc delete secret hf-token -n "$NS"

# Verify
oc get all -n "$NS" -l app.kubernetes.io/instance=qwen3-8b-rocm7
```

---

## Cluster notes

| Item | Value |
|------|-------|
| Cluster URL | `https://api.ods-az-amd-01-pool-ps2mk.azure.rh-ods.com:6443` |
| Username | `htpasswd-cluster-admin-user` |
| Password source | GCP Secret Manager: `ocp_a100_cluster` (project `it-cloud-gcp-nm-automation`) — same password works across all OCP clusters |
| Node type | `Standard_ND96isr_MI300X_v5` (Azure) |
| GPU resource key | `amd.com/gpu` |
| Storage class | `lvms-vg1` |
| Service account | `vllm-anyuid-sa` (needs anyuid SCC) |
| Helm chart | `rhaiis-midstream-snippets/charts/vllm/` (v0.14.0+rhai0) |

## ROCm 7.1 environment variables (set in values file)

| Variable | Value | Why |
|----------|-------|-----|
| `HSA_NO_SCRATCH_RECLAIM` | `1` | Required for RCCL on ROCm 7.1 |
| `VLLM_USE_TRITON_FLASH_ATTN` | `0` | Triton flash attn disabled on ROCm |
| `NCCL_MIN_NCHANNELS` | `112` | ROCm perf tuning |
| `TORCH_BLAS_PREFER_HIPBLASLT` | `1` | Prefer hipBLASLt |
| `VLLM_WORKER_MULTIPROC_METHOD` | `spawn` | Ray multiprocessing workaround |
| `HIP_FORCE_DEV_KERNARG` | `1` | ROCm kernel arg fix |
| `RAY_EXPERIMENTAL_NOSET_HIP_VISIBLE_DEVICES` | `1` | Ray >= 2.45 compat |
| `VLLM_ROCM_USE_AITER` | `0` / `1` | Off for baseline, on for second pass |

## Risks

| Risk | What to check |
|------|---------------|
| ROCm driver mismatch on nodes | `oc describe node` — look for ROCm version labels. CI runners use `rocm71` labels on this cluster. |
| `modelcache-pvc` not in namespace | `oc get pvc --all-namespaces \| grep modelcache` |
| cert-manager missing | `oc get crd certificates.cert-manager.io` — if missing, route TLS won't work but service reachable via `oc port-forward` |
| Model not cached + HF auth fails | That's what the `hf-token` secret is for |

## Note on cluster URL

~~The GCP secret `openshift-mi300x-user-cred` references `amd-02`~~ **Resolved:** `amd-02` is decommissioned (NXDOMAIN). The active cluster is `amd-01`. Use the `ocp_a100_cluster` GCP secret — same password works on all OCP clusters.

---
