# Agent Task: Investigate gpt-oss-120b H200 Performance Regression (INFERENG-6237)

## Objective

Investigate the gpt-oss-120b performance regression on NVIDIA H200 between RHAIIS 3.4-EA2 and RHAIIS 3.4-GA. Determine root cause (or narrow it down) and make a recommendation on whether this blocks the RHAIIS 3.4 RC image release.

Doug needs to make the call on whether this blocks the RC. Give him clear findings so he can decide.

## Background

- **Jira:** INFERENG-6237 (use `jira issue view INFERENG-6237 --plain` to read — MCP is down today, use the jira CLI for all Jira operations)
- **Reporter:** Harshith Umesh (PSAP perf validation team)
- **Assignee:** Daniele Trifiro (out — family obligations, Doug is covering)
- **Tracking doc:** `./FRIDAY_RC_IMAGE.md` — this bug is listed in the "Last-Minute Bugs" section

## The Regression

gpt-oss-120b on H200, comparing RHAIIS 3.4-EA2 vs 3.4-GA:
- **Throughput:** TP1 down 8-9%, TP4 down 14-17%
- **ITL (inter-token latency):** up 13-46%
- **E2E latency:** up 3-6%
- **TTFT:** stable (+3-6%) — regression is in the **decode phase**, not prefill
- **MI300X:** unaffected (within ±2%) — this is purely H200/CUDA
- **TP4 hit hardest** on throughput, **TP1 hit hardest** on ITL (8k/1k workload: +46%)

## Version Mapping

The key question is: what vLLM code changed between EA2 and GA?

- **EA2** images were built from the `3.4-EA2` branch in `rhaiis/containers` (local: `./containers/`). EA2 tags in containers: `v3.4-ea2-2026031801`, `v3.4-ea2-2026032501`. EA2 likely used an earlier rhaiv tag (rhaiv.2 or rhaiv.3 era).
- **GA** images are being built now from the `3.4` branch, using rhaiv.5 (current) soon to be rhaiv.6.

In `nm-vllm-ent` (local: `./nm-vllm-ent/`), the relevant tags are:
```
v0.18.0+rhaiv.0 through v0.18.0+rhaiv.7
```

The commits between rhaiv.3 and rhaiv.6 (the likely EA2→GA delta for CUDA) include:
- Mistral tool parser fixes (multiple PRs)
- Mistral reasoning_effort fix
- AMD Zen CPU backend (#478) — CPU only, shouldn't affect CUDA
- tcmalloc addition for CPU image — CPU only
- CUDA 13.0 reference updates (#472) — **this could matter**
- amd-quark removal — ROCm only
- CUTLASS grouped GEMM OOB fix (#460 / upstream #38571) — **this could matter for H200**
- KV cache offloading support (#37160) — **this could matter**
- PD+Offloading for hybrid models (#457)
- Various Mistral cherry-picks

## Investigation Steps

### 1. Determine exact EA2 vLLM version

Figure out which rhaiv tag EA2 was actually built from:

```bash
# Check the EA2 containers branch for the wheel reference
cd ./containers
git log --oneline remotes/origin/3.4-EA2 | head -20

# Look at the build-args for the EA2 image to find the wheel collection version
git show remotes/origin/3.4-EA2:build-args/cuda-ubi9.conf
```

Also check if the EA2 image is still on quay.io and inspect its labels:
```bash
skopeo inspect --no-tags --override-arch amd64 --override-os linux \
  docker://quay.io/aipcc/rhaiis/cuda-ubi9:3.4.0-EA2 2>&1 | \
  jq '{version: .Labels["version"], release: .Labels["release"], git_commit: .Labels["git.commit"]}'
```
(The exact tag format may differ — check `v3.4-ea2-*` tags or browse quay)

### 2. Narrow the commit delta

Once you know the EA2 rhaiv tag, get the exact diff:
```bash
cd ./nm-vllm-ent
git log --oneline 'v0.18.0+rhaiv.<EA2_VERSION>'..'v0.18.0+rhaiv.6' --
```

### 3. Identify CUDA-relevant changes

From the commit delta, filter for changes that could affect H200/CUDA decode performance. Ignore:
- Anything ROCm/AMD-only (amd-quark, aiter, rocm paths)
- Anything CPU-only (zentorch, zen backend)
- Mistral-specific fixes (tool parser, grammar factory) — these are functional, not perf

Focus on:
- CUTLASS/GEMM changes (the OOB fix in #460 could have changed kernel selection)
- CUDA version reference updates (#472)
- KV cache or scheduling changes
- Anything touching attention backends, FlashInfer, or decode kernels

### 4. Check upstream vLLM for known H200 regressions

```bash
# Search upstream issues
gh search issues "H200 regression" --repo vllm-project/vllm --limit 10
gh search issues "H200 performance" --repo vllm-project/vllm --limit 10
gh search issues "decode regression" --repo vllm-project/vllm --state open --limit 10
```

### 5. Check if INFERENG-5644 is related

There's a separate ticket for DeepSeek throughput regression (-23%) between EA1 and EA2. It's mentioned in `./ROCM7_VALIDATION.md` under INFERENG-5644. Check if it's the same root cause pattern.

```bash
jira issue view INFERENG-5644 --plain
```

### 6. Check the CUTLASS grouped GEMM fix specifically

The OOB read fix (upstream vllm#38571, cherry-picked as nm-vllm-ent #460) is suspicious. OOB fixes in GEMM kernels can change alignment or kernel dispatch paths, which could affect performance.

```bash
cd ./nm-vllm-ent
git show v0.18.0+rhaiv.4:vllm/ -- | head -1  # just to verify you can access
# Look at the actual fix
gh pr view 460 --repo neuralmagic/nm-vllm-ent --json body,title
```

### 7. Write up findings

Create a summary with:
1. **Exact version delta** (which rhaiv tags, which commits)
2. **Suspect commits** that could cause a CUDA decode perf regression
3. **Confidence level** — is this clearly caused by commit X, or is it unclear?
4. **Recommendation** — does this block the RC?

Factors for the RC decision:
- The model still *works*, this is perf not functional
- TP4 losing 15-17% throughput is significant for production
- MI300X is unaffected, so the regression is CUDA-specific
- If it's caused by a specific cherry-pick, could we revert just that?
- Selbi wants RC images for Monday testing — if we hold the RC, we delay the whole 3.4 release

### 8. Update tracking

Add a comment to INFERENG-6237 via Jira CLI with your findings:
```bash
# Write comment to temp file first, then:
jira issue comment add INFERENG-6237 --template /tmp/infereng-6237-comment.md
```

Update `./FRIDAY_RC_IMAGE.md` — in the "Last-Minute Bugs" section under INFERENG-6237, fill in the "Blocking RC?" field with your recommendation and reasoning.

## Important Notes

- Use `jira` CLI for all Jira operations (MCP is busted today). Write comment bodies to temp files first.
- The `nm-vllm-ent` repo is at `./nm-vllm-ent/` — it has all the rhaiv tags fetched.
- The `containers` repo is at `./containers/` — it has the EA2 branch history.
- Do NOT attempt to fix the regression. Just investigate and recommend.
- Time is tight — Doug has ~4 hours left on a Friday to ship RC images. A clear "ship it, investigate Monday" or "hold, here's why" is more valuable than a deep root cause analysis.
