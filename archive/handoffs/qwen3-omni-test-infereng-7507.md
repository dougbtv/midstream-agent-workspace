# Agent Handoff: INFERENG-7507 — Qwen3-Omni e2e test against both CUDA image variants

## What you're doing

Add an end-to-end test for **Qwen3-Omni** (`Qwen/Qwen3-Omni-7B`) to the vLLM-Omni test workflow. The test should run against **both** CUDA image variants we ship:

1. **cuda** (default) — has flashinfer-jit-cache pre-compiled kernels, no CUDA devel packages
2. **cuda-develsdk** — has CUDA devel packages for JIT fallback, no jit-cache

This validates that Qwen3-Omni serves and responds correctly on both paths. Nick Cao originally hit a FlashInfer JIT failure on the develsdk path (INFERENG-7368, now fixed) — this test makes sure it stays fixed.

## Why this matters

We shipped an image with bugs that a reporter (Nick Cao) found first. We now have a smoke test for image gen (Z-Image-Turbo), but nothing that exercises the Qwen/Omni models or the JIT fallback path. This fills that gap.

## JIRA

- **This story:** [INFERENG-7507](https://redhat.atlassian.net/browse/INFERENG-7507)
- **Epic:** [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288) — vLLM-Omni Midstream: Initial Build
- **Related:** [INFERENG-7368](https://redhat.atlassian.net/browse/INFERENG-7368) — FlashInfer CUDA devel (closed, PRs merged)
- **Related:** [INFERENG-7162](https://redhat.atlassian.net/browse/INFERENG-7162) — Initial smoke test (closed, pattern to follow)

## Current test infrastructure

### Workflow
- **File:** `.github/workflows/test-vllm-omni.yml` (on `vllm-omni-build` and `doug/flashinfer-jit-cuda-devel` branches)
- **Stub on main** via PR #518 — feature branches override with `--ref <branch>`
- Supports both `workflow_dispatch` and `workflow_call`

### Key inputs
```
vllm_image:            # Required — the omni image to test
model:                 # Default: Tongyi-MAI/Z-Image-Turbo
label:                 # Default: k8s-a100-solo
tensor_parallel_size:  # Default: 1
run_upstream_tests:    # Default: false — run pytest from test image
test_image:            # Default: vllm_image + '-test' suffix
test_commands:         # JSON array of pytest commands
```

### How to trigger a test
```bash
# Smoke test (existing, Z-Image-Turbo)
gh workflow run test-vllm-omni.yml --repo neuralmagic/nm-cicd \
  --ref vllm-omni-build \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-26785885017 \
  -f model=Tongyi-MAI/Z-Image-Turbo \
  -f label=k8s-a100-solo

# What we want to ADD — Qwen3-Omni against both variants:
# cuda (default, jit-cache):
gh workflow run test-vllm-omni.yml --repo neuralmagic/nm-cicd \
  --ref <your-branch> \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-26785885017 \
  -f model=Qwen/Qwen3-Omni-7B \
  -f label=k8s-a100-solo

# cuda-develsdk (JIT fallback):
gh workflow run test-vllm-omni.yml --repo neuralmagic/nm-cicd \
  --ref <your-branch> \
  -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-develsdk-26785885794 \
  -f model=Qwen/Qwen3-Omni-7B \
  -f label=k8s-a100-solo
```

### Deploy script
- **File:** `.github/scripts/deploy_vllm_omni.sh`
- Deploys a k8s pod with the omni image, waits for `/health` → 200, then hits `/v1/images/generations`
- For Qwen3-Omni you'll need to hit a **different endpoint** — likely `/v1/chat/completions` with multimodal input (audio/text), not `/v1/images/generations`

### Test actions
- `.github/actions/test-vllm-omni/action.yml` — composite action for smoke test (pod deploy + API validation + cleanup)
- `.github/actions/test-vllm-omni-upstream/action.yml` — runs upstream pytest suite from a test image

### Successful smoke test run (reference)
- **Run:** [26830390549](https://github.com/neuralmagic/nm-cicd/actions/runs/26830390549) — Z-Image-Turbo on cuda default variant, PASSED
- Check this run's logs for the exact flow: deploy → health check → API call → cleanup

## Available images

| Variant | Image | Notes |
|---------|-------|-------|
| cuda (default) | `quay.io/vllm/automation-vllm-omni:cuda-26785885017` | jit-cache, no CUDA devel |
| cuda-develsdk | `quay.io/vllm/automation-vllm-omni:cuda-develsdk-26785885794` | CUDA devel, JIT fallback |
| cuda-test | `quay.io/vllm/automation-vllm-omni:cuda-26820652811-test` | Has upstream tests/ baked in |

## What needs to change

1. **Deploy script update** — `deploy_vllm_omni.sh` currently validates via `/v1/images/generations`. For Qwen3-Omni, you need to validate a different endpoint. Options:
   - `/v1/chat/completions` with a simple text prompt (Qwen3-Omni is an omni model that handles text/audio/image)
   - Just validate `/health` → 200 and server starts without errors (simpler, still catches JIT failures)

2. **Workflow update** — either:
   - Add a second job/step in `test-vllm-omni.yml` that runs Qwen3-Omni after Z-Image-Turbo
   - Or make the endpoint validation configurable (param for which endpoint to hit)
   - The workflow already accepts `model` as input, so the dispatch side is ready

3. **Run against both variants** — the test should be triggered twice (or the workflow should matrix over variants). The key validation for develsdk is that the server starts without FlashInfer JIT compilation errors.

## Repos and local paths

| Repo | Local path | Branch |
|------|-----------|--------|
| nm-cicd | `/home/hdds/480ssd/codebase/worktree-omni/nm-cicd` | `doug/flashinfer-jit-cuda-devel` (latest) |
| nm-vllm-omni-ent | `/home/hdds/480ssd/codebase/worktree-omni/vllm-omni` | remote `midstream` → neuralmagic/nm-vllm-omni-ent |
| Working doc | `/home/hdds/480ssd/codebase/worktree-omni/MIDSTREAM_OMNI_BUILD.md` (symlink) | — |
| Resolved doc path | `/home/hdds/480ssd/codebase/dougbtv-redhat-notes/Snippets/MIDSTREAM_OMNI_BUILD.md` | Use this for edits |

## Reference: Nick Cao's original repro

- [Repro gist](https://gist.github.com/dougbtv/409b465316e041b32e162dbc8df23e86)
- [nm-vllm-omni-ent PR #5 comment](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/5#issuecomment-4557342043)
- Nick was running Qwen3-TTS tests, hit missing CUDA headers during FlashInfer JIT

## JIRA CLI (MCP is busted)

```bash
# Comment on the story
cat > /tmp/jira-comment.md << 'EOF'
### Your update here
EOF
jira issue comment add INFERENG-7507 -T /tmp/jira-comment.md --no-input

# Move status
jira issue move INFERENG-7507 "In Progress"
```

## Definition of done

- [ ] Qwen3-Omni test added to test workflow (or standalone script)
- [ ] Test passes against `cuda` (default) variant
- [ ] Test passes against `cuda-develsdk` variant
- [ ] Server starts without FlashInfer JIT errors on develsdk
- [ ] JIRA updated with results
- [ ] Working doc updated if anything notable
