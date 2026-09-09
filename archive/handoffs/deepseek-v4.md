# DeepSeek V4 + vLLM Build Handoff

**Purpose:** hand this to an agent that will build and test a vLLM image for DeepSeek V4.

**Snapshot time:** 2026-04-24. DeepSeek V4 dropped today; vLLM support is moving fast. Re-check all linked PRs before building.

---

## TL;DR

vLLM has **initial DeepSeek V4 support** already announced by the vLLM team. There is a dedicated vLLM blog post, a special Docker image tag, recipes for Pro and Flash, and a main implementation PR:

- vLLM blog: <https://vllm.ai/blog/deepseek-v4>
- Implementation PR: <https://github.com/vllm-project/vllm/pull/40760>
- Docker tags: `vllm/vllm-openai:deepseekv4-cu130` and `vllm/vllm-openai:deepseekv4-cu129`
- Pro recipe: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Pro>
- Flash recipe: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash>

The support should be treated as **fresh / initial / likely to churn**. The vLLM post explicitly says further optimizations are underway, especially:

- **DeepGEMM MegaMoE kernel**
- **Paged prefill kernel**

The build agent should start from the `deepseekv4-cu130` image or build directly from the commit/branch backing PR `#40760`, then check whether follow-up fixes like `#40772` have merged.

---

## Model artifacts and model cards

### Hugging Face collection

- Collection: <https://huggingface.co/collections/deepseek-ai/deepseek-v4>

### Main models

- DeepSeek-V4-Pro: <https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro>
- DeepSeek-V4-Flash: <https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash>

### Model variants from the model card

| Model | Total params | Activated params | Context | Precision |
|---|---:|---:|---:|---|
| `DeepSeek-V4-Flash-Base` | 284B | 13B | 1M | FP8 mixed |
| `DeepSeek-V4-Flash` | 284B | 13B | 1M | FP4 + FP8 mixed |
| `DeepSeek-V4-Pro-Base` | 1.6T | 49B | 1M | FP8 mixed |
| `DeepSeek-V4-Pro` | 1.6T | 49B | 1M | FP4 + FP8 mixed |

Notes:

- FP4 + FP8 mixed means MoE expert parameters use FP4; most non-expert parameters use FP8.
- Both Pro and Flash support up to a **1M token context**.
- Pro has 1.6T total params / 49B active.
- Flash has 284B total params / 13B active.
- License: MIT.
- The model card says there is **no Jinja-format chat template** in this release. Instead, DeepSeek provides an `encoding` folder with Python scripts and tests for OpenAI-compatible message encoding and output parsing.
- Think Max mode should use at least `--max-model-len >= 393216` / 384K to avoid truncation.

Source: model card at <https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro>

---

## DeepSeek V4 architecture notes relevant to vLLM

DeepSeek V4 is not just “DeepSeek V3 with bigger weights.” It has enough architectural delta to require real runtime support.

Key changes:

1. **Hybrid attention**
   - Combines **Compressed Sparse Attention (CSA)** and **Heavily Compressed Attention (HCA)**.
   - vLLM’s blog describes two compressed attention modes:
     - `c4a`: roughly 1/4 KV compression. One compressed token is a weighted sum of 8 uncompressed tokens with stride 4.
     - `c128a`: roughly 1/128 KV compression. One compressed token is a weighted sum of 128 uncompressed tokens with stride 128.
   - Uses a short sliding window of size 128 to preserve locality before compression boundaries.
   - Uses inverse RoPE because key and value are shared.

2. **mHC / Manifold-Constrained Hyper-Connections**
   - Architecture change around residual connections.
   - vLLM blog says this is simpler to adapt than the attention work, but still part of the model delta.

3. **Native FP4 MoE expert weights**
   - vLLM blog explicitly calls this out as requiring special handling.
   - Expect loader / quantization path sensitivity.

4. **MoE module changes**
   - Still DeepSeekMoE lineage, but changed enough that serving code has dedicated DeepSeek V4 paths.

5. **Million-token context**
   - The main challenge is not just model loading; it is correct + efficient KV/cache state handling under prefill, decode, prefix caching, disaggregated prefill, CUDA graphs, and MTP.

---

## vLLM support state

### Official vLLM blog

Source: <https://vllm.ai/blog/deepseek-v4>

The blog says:

- vLLM now supports both:
  - `deepseek-ai/DeepSeek-V4-Pro`
  - `deepseek-ai/DeepSeek-V4-Flash`
- This is the **initial release of model support**.
- Further optimizations are actively underway.
- The implementation targets NVIDIA GPUs, especially Hopper and Blackwell.
- Hardware plugins can independently support the model; vLLM explicitly points to:
  - `vllm-ascend`: <https://github.com/vllm-project/vllm-ascend>
  - `vllm-mlu`: <https://github.com/Cambricon/vllm-mlu>

### Main implementation PR

- PR: <https://github.com/vllm-project/vllm/pull/40760>
- Referenced directly by the vLLM blog as “the full implementation.”
- The agent should inspect this PR first and determine:
  - Is it merged?
  - What commit SHA landed?
  - Which image tag contains it?
  - Are there follow-up fix PRs needed?
  - Does it touch model definition, tokenizer mode, tool parser, reasoning parser, attention backend, cache kernels, MTP, FP4/FP8 handling, or Docker build bits?

### Known follow-up PRs / issues

#### PR #40772 — Fix IMA in DSA + MTP

- PR: <https://github.com/vllm-project/vllm/pull/40772>
- Title: `[Bugfix] Fix IMA in DSA + MTP`
- Status observed: open at time of research, labeled `bug` and `ready`.
- Author notes: “This PR fixes the IMA bug when using DSA with MTP. The bug was introduced in #40654.”
- Bot review summary says it modifies `csrc/cache_kernels.cu`, specifically improving robustness of `cp_gather_indexer_k_quant_cache_kernel`.
- Why it matters: DeepSeek V4 uses DSA-ish compressed attention and optional MTP. If using MTP, track this PR before validating correctness/perf.

#### PR #40654 — Avoid seq_lens_cpu GPU->CPU sync

- PR: <https://github.com/vllm-project/vllm/pull/40654>
- Title: `[Core] Avoid seq_lens_cpu GPU->CPU sync`
- Status observed: merged Apr 24, 2026.
- Motivation in comments: eliminating prefill CPU sync for DS3.2 and generalizing CPU-side upper-bound sequence length handling.
- Why it matters: not DeepSeek V4-specific, but directly adjacent to DeepSeek-style MLA/DSA prefill performance and apparently introduced the bug fixed by #40772.

#### Issue #40778 — DeepSeek V4 support request

- Issue: <https://github.com/vllm-project/vllm/issues/40778>
- Title: `[Feature]: deepseek v4 support`
- Opened Apr 24, 2026.
- Body only notes release links to Pro and Flash.
- GitHub page showed “No branches or pull requests” linked in the issue metadata at the time of viewing. This likely predates/does not reference the official support PR.

#### Issue #40790 — dpsk v4 on 8x H20-96G

- Issue: <https://github.com/vllm-project/vllm/issues/40790>
- Title: `[Bug]: dpsk v4 on 8* H20-96G`
- Opened Apr 24, 2026.
- Use this as a warning that H20 deployments may have fresh issues. Inspect logs/details before assuming H20 works.

#### vLLM Ascend Issue #8655 — download docs for Flash w8a8 MTP

- Issue: <https://github.com/vllm-project/vllm-ascend/issues/8655>
- Title: `[Bug]: how to download DeepSeek-V4-Flash-w8a8-mtp`
- Opened Apr 24, 2026.
- This is documentation friction around the Ascend path, not necessarily core vLLM CUDA.

#### vLLM Ascend DeepSeek V4 docs

- Docs: <https://docs.vllm.ai/projects/ascend/en/v0.13.0/tutorials/DeepSeek-V4.html>
- Notes from docs:
  - vllm-ascend temporarily only supports `DeepSeek-V4-Flash`.
  - `DeepSeek-V4-Flash-w8a8-mtp` requires one Atlas 800 A3 128G x 8 node or one Atlas 800 A2 64G x 8 node.
  - Integrated in `quay.io/ascend/vllm-ascend:v0.13.0rc3` / `v0.13.0rc3-a3`.

---

## Docker / image state

Docker Hub showed fresh DeepSeek V4 tags pushed recently:

- `vllm/vllm-openai:deepseekv4-cu130`
- `vllm/vllm-openai:deepseekv4-cu129`
- arch-specific variants:
  - `deepseekv4-x86_64-cu130`
  - `deepseekv4-x86_64-cu129`
  - `deepseekv4-arm64-cu130`
  - `deepseekv4-arm64-cu129`

Docker Hub tag page: <https://hub.docker.com/r/vllm/vllm-openai/tags>

Recommendation:

- For fastest initial validation, start with `vllm/vllm-openai:deepseekv4-cu130`.
- For reproducible work, inspect the image labels/digest and map it back to a vLLM commit.
- If building custom, use the commit from PR `#40760` plus any merged follow-ups after Apr 24, 2026.
- If PR `#40772` is not merged and MTP is enabled, consider cherry-picking it or disabling MTP for initial correctness validation.

---

## Official vLLM quickstart commands

### DeepSeek-V4-Pro

The vLLM blog says this command is runnable on **8xB200 or 8xB300**:

```bash
docker run --gpus all   --ipc=host -p 8000:8000   -v ~/.cache/huggingface:/root/.cache/huggingface   vllm/vllm-openai:deepseekv4-cu130 deepseek-ai/DeepSeek-V4-Pro   --trust-remote-code   --kv-cache-dtype fp8   --block-size 256   --enable-expert-parallel   --data-parallel-size 8   --compilation-config '{"cudagraph_mode":"FULL_AND_PIECEWISE", "custom_ops":["all"]}'   --attention_config.use_fp4_indexer_cache=True   --tokenizer-mode deepseek_v4   --tool-call-parser deepseek_v4   --enable-auto-tool-choice   --reasoning-parser deepseek_v4
```

### DeepSeek-V4-Flash

The vLLM blog says this command is runnable on **4xB200 or 4xB300**:

```bash
docker run --gpus all   --ipc=host -p 8000:8000   -v ~/.cache/huggingface:/root/.cache/huggingface   vllm/vllm-openai:deepseekv4-cu130 deepseek-ai/DeepSeek-V4-Flash   --trust-remote-code   --kv-cache-dtype fp8   --block-size 256   --enable-expert-parallel   --data-parallel-size 4   --compilation-config '{"cudagraph_mode":"FULL_AND_PIECEWISE", "custom_ops":["all"]}'   --attention_config.use_fp4_indexer_cache=True   --tokenizer-mode deepseek_v4   --tool-call-parser deepseek_v4   --enable-auto-tool-choice   --reasoning-parser deepseek_v4
```

---

## vLLM recipe notes

### Pro recipe

- URL: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Pro>
- Says vLLM version: `vLLM 0.20.1+`
- Recommends:
  - B300 8x GPU: single-node DP + EP with `--data-parallel-size 8`
  - H200 8x GPU: DP + EP with `--data-parallel-size 8`
  - H200 context capped at `--max-model-len 800000` to leave KV headroom with dense params replicated across ranks
  - GB200 NVL4: ~960 GB mixed-precision checkpoint does not fit on one 4-GPU tray; use 2 trays / 8 GPUs total

### Flash recipe

- URL: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash>
- Use for smaller first pass if hardware is constrained.
- Still needs very serious hardware for meaningful context/concurrency.

---

## Build strategy

### Option A: use official DeepSeek V4 image

Use this if the task is validation rather than vLLM development.

```bash
docker pull vllm/vllm-openai:deepseekv4-cu130
docker image inspect vllm/vllm-openai:deepseekv4-cu130
```

Capture:

- image digest
- labels
- vLLM version
- CUDA version
- Python version
- torch version
- commit SHA if present

Then run Flash first unless the target hardware is definitely B200/B300/H200 enough for Pro.

### Option B: build custom vLLM from source

Use this if the task is to patch, instrument, or cherry-pick.

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm

# Inspect current state
git fetch origin pull/40760/head:deepseek-v4-40760
git fetch origin pull/40772/head:deepseek-v4-40772

git log --oneline --decorate -20 origin/main
git branch --contains deepseek-v4-40760
git branch --contains deepseek-v4-40772
```

Decision tree:

1. If `#40760` is merged to `main`, build from `main` or the exact merged commit.
2. If `#40760` is not merged, build from the PR branch.
3. If `#40772` is merged, use a commit after it.
4. If `#40772` is not merged and MTP is enabled, cherry-pick it or disable MTP for the first pass.
5. Check whether any later PRs mention `DeepSeek V4`, `deepseek_v4`, `DSA`, `MTP`, `fp4 indexer`, `c4a`, `c128a`, or `MegaMoE`.

Suggested GitHub searches:

```text
repo:vllm-project/vllm deepseek_v4
repo:vllm-project/vllm "DeepSeek V4"
repo:vllm-project/vllm "c4a"
repo:vllm-project/vllm "c128a"
repo:vllm-project/vllm "fp4_indexer"
repo:vllm-project/vllm "DeepGEMM MegaMoE"
repo:vllm-project/vllm "paged prefill"
repo:vllm-project/vllm "DSA" "MTP"
```

### Build concerns

Likely build-risk areas:

- CUDA 13 / CUDA 12.9 compatibility
- Blackwell-specific paths
- FP4/FP8 kernels
- FlashInfer / FlashMLA / TRTLLM-GEN versions
- CUTLASS / CuTeDSL paths
- MTP speculative config
- Expert parallel + data parallel setup
- Docker image includes the parser modes:
  - `--tokenizer-mode deepseek_v4`
  - `--tool-call-parser deepseek_v4`
  - `--reasoning-parser deepseek_v4`
- Trust remote code / custom model card encoding scripts

---

## Hardware expectations

### NVIDIA

Official vLLM blog quickstarts:

- Pro: 8xB200 or 8xB300
- Flash: 4xB200 or 4xB300

vLLM recipe:

- Pro on 8xB300: single-node DP + EP
- Pro on 8xH200: DP + EP, cap context to 800K
- Pro on GB200 NVL4: use two 4-GPU trays / 8 GPUs total because ~960 GB checkpoint does not fit on one tray

### Ascend

vllm-ascend docs:

- only Flash currently
- `DeepSeek-V4-Flash-w8a8-mtp`
- one Atlas 800 A3 128G x 8 node or one Atlas 800 A2 64G x 8 node

### H20 warning

There is a fresh issue for “dpsk v4 on 8* H20-96G”:

- <https://github.com/vllm-project/vllm/issues/40790>

Do not assume H20 works without testing.

### Consumer GPUs

Not a serious target for Pro. Flash might be experimentally possible only with aggressive quantization and tiny contexts/concurrency, but this is outside the official vLLM quickstart.

---

## Runtime challenges to validate

### 1. Model loading

Check:

- safetensor load
- FP4 + FP8 mixed checkpoint handling
- native FP4 MoE expert weights
- dense params replicated as expected
- expert parallel placement
- data parallel rank behavior

### 2. Tokenization / chat encoding

The model card says no Jinja chat template is included. vLLM adds DeepSeek V4-specific modes:

- `--tokenizer-mode deepseek_v4`
- `--tool-call-parser deepseek_v4`
- `--reasoning-parser deepseek_v4`

Validate:

- normal chat
- reasoning mode
- tool calls
- parsing of `<think>...</think>`
- non-think / high / max behavior via `chat_template_kwargs`
- OpenAI-compatible `/v1/chat/completions`

### 3. Attention correctness

Validate at small context first, then scale.

Areas to watch:

- `c4a` compressed attention
- `c128a` compressed attention
- short sliding window
- inverse RoPE correctness
- DSA top-k behavior
- prefix cache with compressed KV
- disaggregated prefill / decode if used
- chunked prefill

### 4. MTP

Optional but performance-relevant.

Watch PR #40772. If not merged, MTP + DSA may be broken.

### 5. Long context

Do not jump to 1M first. Suggested progression:

1. 4K
2. 32K
3. 128K
4. 384K for Think Max
5. 800K if H200 recipe target
6. 1M only after memory/cache behavior is understood

Validate KV usage, OOM behavior, prefix cache hits, and prefill time.

### 6. Performance

Track:

- TTFT
- ITL / TPOT
- throughput at batch sizes 1, 2, 4, 8, 16
- GPU memory
- NCCL/all-to-all overhead
- expert parallel imbalance
- c4a/c128a kernel timings
- MTP gain/loss
- FP8 KV vs BF16 KV behavior

---

## Minimal validation script

After the server starts:

```bash
curl http://localhost:8000/v1/models | jq .
```

Basic chat:

```bash
curl http://localhost:8000/v1/chat/completions   -H 'Content-Type: application/json'   -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [
      {"role": "user", "content": "What is 17*19? Return only the final integer."}
    ],
    "temperature": 1.0,
    "top_p": 1.0,
    "max_tokens": 64
  }' | jq .
```

Think high:

```bash
curl http://localhost:8000/v1/chat/completions   -H 'Content-Type: application/json'   -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [
      {"role": "user", "content": "Solve this carefully: if a cluster has 8 GPUs and each GPU has 192GB, how much total VRAM is available?"}
    ],
    "temperature": 1.0,
    "top_p": 1.0,
    "max_tokens": 512,
    "chat_template_kwargs": {
      "thinking": true,
      "reasoning_effort": "high"
    }
  }' | jq .
```

Tool calling smoke test:

```bash
curl http://localhost:8000/v1/chat/completions   -H 'Content-Type: application/json'   -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [
      {"role": "user", "content": "What is the weather in Burlington, Vermont? Use the tool."}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get weather for a location",
          "parameters": {
            "type": "object",
            "properties": {
              "location": {"type": "string"}
            },
            "required": ["location"]
          }
        }
      }
    ],
    "tool_choice": "auto",
    "max_tokens": 512
  }' | jq .
```

---

## Agent checklist

Before build:

- [ ] Open vLLM PR `#40760`; record merge status and commit SHA.
- [ ] Open vLLM PR `#40772`; record merge status and decide whether to cherry-pick.
- [ ] Search vLLM PRs/issues for `deepseek_v4`, `DeepSeek V4`, `c4a`, `c128a`, `DSA`, `MTP`, `fp4 indexer`, `MegaMoE`, `paged prefill`.
- [ ] Inspect Docker tag `vllm/vllm-openai:deepseekv4-cu130`.
- [ ] Confirm target hardware and driver/CUDA compatibility.
- [ ] Decide model: Flash first unless Pro hardware is known-good.

During build:

- [ ] Capture source commit SHA.
- [ ] Capture torch/CUDA versions.
- [ ] Capture installed vLLM version.
- [ ] Capture `pip freeze` or environment manifest.
- [ ] Confirm DeepSeek V4 parser/tokenizer/reasoning modes are present.
- [ ] Confirm model architecture registration includes DeepSeek V4.

During run:

- [ ] Start with small context and no MTP.
- [ ] Validate chat response.
- [ ] Validate Think High.
- [ ] Validate tool-call parsing.
- [ ] Enable MTP only after checking PR #40772.
- [ ] Increase context progressively.
- [ ] Capture memory usage and failure modes.
- [ ] Log exact command line.

Deliverables:

- [ ] Dockerfile or build script.
- [ ] Exact source commit SHA.
- [ ] Runtime command for Flash.
- [ ] Runtime command for Pro, if hardware allows.
- [ ] Known issues / workaround list.
- [ ] Smoke-test results.
- [ ] Performance notes.
- [ ] Recommendation: use official image, custom build, or wait for upstream release.

---

## Source links

Primary:

- vLLM blog: <https://vllm.ai/blog/deepseek-v4>
- vLLM blog markdown source: <https://raw.githubusercontent.com/vllm-project/vllm-project.github.io/main/_posts/2026-04-24-deepseek-v4.md>
- vLLM PR #40760: <https://github.com/vllm-project/vllm/pull/40760>
- vLLM PR #40772: <https://github.com/vllm-project/vllm/pull/40772>
- vLLM PR #40654: <https://github.com/vllm-project/vllm/pull/40654>
- vLLM issue #40778: <https://github.com/vllm-project/vllm/issues/40778>
- vLLM issue #40790: <https://github.com/vllm-project/vllm/issues/40790>
- Docker Hub tags: <https://hub.docker.com/r/vllm/vllm-openai/tags>

Model:

- HF collection: <https://huggingface.co/collections/deepseek-ai/deepseek-v4>
- Pro model card: <https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro>
- Flash model card: <https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash>
- Technical report PDF link from model card: <https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf>

Recipes / hardware:

- Pro recipe: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Pro>
- Flash recipe: <https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash>
- vllm-ascend DeepSeek V4 docs: <https://docs.vllm.ai/projects/ascend/en/v0.13.0/tutorials/DeepSeek-V4.html>
- vllm-ascend issue #8655: <https://github.com/vllm-project/vllm-ascend/issues/8655>

Context:

- Reuters: Huawei Ascend supernode support: <https://www.reuters.com/business/media-telecom/huawei-ascend-supernode-support-deepseek-v4-2026-04-24/>
- Reuters: DeepSeek V4 early hardware access / Huawei: <https://www.reuters.com/world/china/deepseek-withholds-latest-ai-model-us-chipmakers-including-nvidia-sources-say-2026-02-25/>
