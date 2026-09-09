# Agent Prompt: INFERENG-6341 / INFERENG-6238 — HIP hipErrorInvalidValue Blocker

You are investigating a **Blocker-priority** HIP runtime crash in the Pixtral vision encoder on ROCm MI300X GPUs. Two Jira tickets describe the same crash class. Your goal is to **reproduce the crash, identify the root cause, and propose a fix or workaround**.

---

## Situation

Two users independently hit `hipErrorInvalidValue` during vLLM's memory profiling phase when loading Pixtral-architecture vision-language models on MI300X (gfx942) with ROCm. The crash occurs at `pixtral.py:642` — a residual connection `h = x + r` inside the vision encoder transformer layer — during `EngineCore._initialize_kv_caches → profile_run → embed_multimodal → vision_encoder.forward`.

**We could NOT reproduce this crash across 6 rounds of testing on 2026-04-24.** The bug is environment-sensitive. Your job is to figure out what's different.

---

## The Two Tickets

### INFERENG-6341 (Primary — Blocker, filed 2026-04-27 by Tarun Kumar)
- **Model:** `mistralai/Mistral-Small-3.1-24B-Instruct-2503`
- **Image:** `v0.18.0+rhaiv.7` (nm-vllm-ent tag)
- **Config:** dtype=bfloat16, NO quantization, TP=1, max_model_len=9000
- **Attention backend:** `ROCM_ATTN` was selected (NOT TRITON_ATTN)
- **GPU:** MI300X (gfx942)
- **Warning:** `VLLM_USE_TRITON_FLASH_ATTN` was flagged as "unknown config variable" in rhaiv.7
- **Crash point:** `pixtral.py:642` — `h = x + r` (residual after attention in vision transformer layer)
- **Note:** Model weights loaded successfully (44.78 GiB). Crash happens during profiling, not weight loading.
- **Encoder cache budget:** 8192 tokens, 2 image items at profiling time
- Jira: https://redhat.atlassian.net/browse/INFERENG-6341

### INFERENG-6238 (Related — filed 2026-04-21 by Harshith Umesh)
- **Model:** `mistralai/Ministral-3-14B-Instruct-2512`
- **Image:** RHAIIS-3.4-GA (production image, NOT rhaiv.7)
- **Config:** fp8 quantization, TP=1
- **Attention backend:** Reporter says TRITON_ATTN, but in our repro tests vLLM overrode it to ROCM_ATTN
- **GPU:** MI300X (gfx942)
- **Same crash path:** vision encoder profiling → hipErrorInvalidValue
- Jira: https://redhat.atlassian.net/browse/INFERENG-6238

### Critical Observation
INFERENG-6341 crashes with **ROCM_ATTN** selected. Our INFERENG-6238 hypothesis was that the bug might be TRITON_ATTN-specific. **6341 disproves that.** The crash is attention-backend-agnostic — it happens during the vision encoder forward pass, which runs its own attention (not the main vLLM decode attention path).

---

## What's Been Tried (6 Rounds of Repro, 2026-04-24, INFERENG-6238)

All tests were run on MI300X nodes in an OCP cluster using an accept-sync test image.

| Round | Setup | Result | Notes |
|-------|-------|--------|-------|
| 1-3 | Ministral-3-14B, various configs | BLOCKED | AITER JIT failed because `HOME=/` in OCP containers. Root caused to missing `ENV HOME=/home/vllm` in ROCm Dockerfile. |
| 4 | Ministral-3-14B, language model_type, ROCM_ATTN | PASS | But this didn't exercise the vision encoder |
| 5 | Ministral-3-14B, **vision** model_type, ROCM_ATTN | PASS | Vision encoder was exercised — no crash |
| 6 | Ministral-3-14B, vision, TRITON_ATTN requested | PASS | vLLM overrode TRITON_ATTN → ROCM_ATTN. Still passed. |

**Key finding from round 6:** Setting `VLLM_ATTENTION_BACKEND=TRITON_ATTN` does NOT guarantee Triton attention is used. vLLM's ROCm backend selection logic (`rocm.py`) has its own priority that prefers ROCM_ATTN. This means the reporter's claim of using TRITON_ATTN may be inaccurate.

---

## Environment Variables That Matter

Our CI test environment sets these (in `nm-cicd/.github/scripts/setup_env_test_vllm.sh`):

| Variable | Value | Purpose |
|----------|-------|---------|
| `HIP_FORCE_DEV_KERNARG` | `1` | ROCm kernel argument handling fix (line 63) |
| `HSA_NO_SCRATCH_RECLAIM` | `1` | Required for RCCL in ROCm 7.1 |
| `NUMA_BALANCING` | `0` (disabled via `echo 0 > /proc/sys/kernel/numa_balancing`) | Prevents NUMA migration issues |

**The reporter's environments may NOT have these set.** This is the #1 hypothesis for why we can't reproduce.

For debugging, also useful:
- `AMD_SERIALIZE_KERNEL=3` — serializes GPU kernel execution for precise error localization (very slow, debug only)

---

## The Crash Code (pixtral.py at v0.18.0+rhaiv.7)

```python
# pixtral.py:635-646 — TransformerBlock.forward()
def forward(
    self,
    x: torch.Tensor,
    mask: torch.Tensor,
    freqs_cis: torch.Tensor,
) -> torch.Tensor:
    r = self.attention.forward(                    # line 641
        self.attention_norm(x), mask=mask, freqs_cis=freqs_cis
    )
    h = x + r    # <--- LINE 642: CRASH HERE (hipErrorInvalidValue)
    r = self.feed_forward.forward(self.ffn_norm(h))
    out = h + r
    return out
```

**Important:** HIP errors are reported asynchronously. The real faulting kernel is likely `self.attention.forward()` on line 641 (or the RMSNorm in `self.attention_norm(x)`), NOT the elementwise add on line 642. The add is just where the runtime surfaces the earlier async error.

---

## Investigation Plan

### Phase 1: Environment Diff (No GPU needed)

1. **Get Tarun's exact environment** — Ask on INFERENG-6341 for:
   - Exact container image digest (not just tag)
   - `env | grep -i 'hip\|hsa\|rocm\|numa\|amd\|vllm'` output
   - `cat /proc/sys/kernel/numa_balancing`
   - ROCm version: `rocminfo | head -20` and `hipcc --version`
   - Is this OCP or bare-metal?
   - How is the model being served? (vLLM CLI args, K8s manifest, etc.)

2. **Diff against our test environment** — Compare Tarun's env against what nm-cicd sets in `setup_env_test_vllm.sh`. Key suspects:
   - `HIP_FORCE_DEV_KERNARG` missing
   - NUMA balancing enabled
   - Different ROCm patch level
   - Different container base image (EL9 vs Ubuntu)

3. **Check the RHAIIS-3.4-GA image vs rhaiv.7** — What attention backend does the GA image default to? If it defaults to TRITON_ATTN but rhaiv.7 defaults to ROCM_ATTN, that's relevant context but doesn't explain 6341 (which uses rhaiv.7 with ROCM_ATTN).

### Phase 2: Reproduction Attempt

4. **Reproduce with Mistral-Small-3.1-24B on rhaiv.7** — Use the exact model and config from INFERENG-6341:
   ```bash
   # On an MI300X node in OCP
   vllm serve mistralai/Mistral-Small-3.1-24B-Instruct-2503 \
     --dtype bfloat16 \
     --tensor-parallel-size 1 \
     --max-model-len 9000
   ```

5. **Try WITHOUT our env vars** — Strip `HIP_FORCE_DEV_KERNARG`, enable NUMA balancing, and see if crash reproduces:
   ```bash
   unset HIP_FORCE_DEV_KERNARG
   unset HSA_NO_SCRATCH_RECLAIM
   echo 1 > /proc/sys/kernel/numa_balancing
   vllm serve mistralai/Mistral-Small-3.1-24B-Instruct-2503 --dtype bfloat16 --tp 1 --max-model-len 9000
   ```

6. **If crash reproduces, bisect env vars** — Add them back one at a time to find which one is the fix.

7. **If crash doesn't reproduce, try AMD_SERIALIZE_KERNEL=3** — This slows execution but gives precise error localization if there's a latent race condition.

### Phase 3: Root Cause Analysis

8. **If `HIP_FORCE_DEV_KERNARG=1` is the fix** — This means the vision encoder's attention kernel has a kernel argument alignment issue on MI300X. Document it, ensure it's set in all RHAIIS images, and close both tickets.

9. **If NUMA balancing is the cause** — Document the requirement to disable it for vision encoder models on MI300X. Add to RHAIIS deployment docs.

10. **If neither env var explains it** — The crash may be model-size or config dependent. Try:
    - Varying `max_model_len` (try 4096, 16384)
    - Varying `encoder_cache_budget` (if exposed)
    - Trying with `--enforce-eager` (disables CUDA graphs)
    - Checking if the model's specific attention head configuration triggers a kernel shape that hits a HIP bug

---

## Tools Available

### Jira CLI
```bash
# Read a ticket
jira issue view INFERENG-6341 --plain

# Add a comment (use temp file for multiline)
cat > /tmp/jira_comment.md << 'EOF'
Your comment here in markdown
EOF
jira issue comment add INFERENG-6341 --body "$(cat /tmp/jira_comment.md)"

# Transition
jira issue move INFERENG-6341 "In Review"
```

### GitHub CLI
```bash
# Check nm-vllm-ent tags
gh api repos/neuralmagic/nm-vllm-ent/tags --jq '.[].name' | head -10

# Trigger a debug workflow on MI300X
gh workflow run debug.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/v0.18.0-rocm7-gpt-oss-fix \
  -f label=amd-mi300x-1gpu \
  -f timeout=60 \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=rhai/0.18.0 \
  -f python=3.12
```

**IMPORTANT: Never use `branch=main` for Buildkite or CI builds. It clutters the main view, annoys maintainers, and you can't delete the record.**

### Local Repos
- **nm-cicd:** `/home/hdds/480ssd/codebase/worktree-nm/nm-cicd/` (branch: `doug/v0.20.0-release-validation`)
  - Test env setup: `.github/scripts/setup_env_test_vllm.sh` (line 63 = HIP_FORCE_DEV_KERNARG)
  - Build env: `.github/scripts/setup_env_build_vllm.sh`
  - ROCm Dockerfile: look for `ENV HOME` fix
- **nm-vllm-ent:** `/home/hdds/480ssd/codebase/worktree-nm/nm-vllm-ent/` (branch: `doug/deepseek-v4-0day-v4`)
  - Tag `v0.18.0+rhaiv.7` is the release that crashes
  - `vllm/model_executor/models/pixtral.py` — the crash file
  - `vllm/attention/backends/rocm.py` — attention backend selection logic
- **Validation doc:** `/home/hdds/480ssd/codebase/worktree-nm/ROCM7_VALIDATION.md`
  - Section 9: Prior MI355X hipErrorInvalidValue triage (different hardware, but same error class)

### Related Jira Tickets
- **INFERENG-6111** — GPT-OSS MXFP4 Triton 3.6 crash on MI355X (RESOLVED)
- **INFERENG-6219** — CanonicalizePointers gfx950 assertion (RESOLVED)
- **INFERENG-5308** — Parent ROCm 7 validation epic

---

## Success Criteria

1. **Reproduce the crash** — or conclusively explain why we cannot (environment delta identified)
2. **Identify root cause** — which env var, kernel, or config triggers it
3. **Propose fix** — env var requirement in Dockerfile, code change, or upstream bug report
4. **Update Jira** — Comment findings on INFERENG-6341, link to 6238, recommend whether 6238 should close as duplicate
5. **Update ROCM7_VALIDATION.md** — Add findings to the Known Risks section and update the Remaining checklist

---

## What NOT To Do

- Don't push to `main` or `rhai/0.18.0` branches directly
- Don't use `branch=main` for any CI builds
- Don't close tickets without confirming with Doug
- Don't modify the FRIDAY_RC_IMAGE.md file (another agent owns that)
- Don't spend time on MI355X (gfx950) — this bug is MI300X (gfx942) specific
