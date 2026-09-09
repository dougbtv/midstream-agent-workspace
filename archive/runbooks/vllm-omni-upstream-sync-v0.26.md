# vLLM-Omni v0.26.0 Upstream Sync Runbook

**JIRA:** [INFERENG-9553](https://redhat.atlassian.net/browse/INFERENG-9553) — vLLM-Omni TP: Upstream sync to vllm-omni v0.26.0
**Parent:** [INFERENG-9448](https://redhat.atlassian.net/browse/INFERENG-9448) — v0.26.0 sync coordination
**Date started:** 2026-08-03

## Overview

Sync nm-vllm-omni-ent from its current base (v0.25.z, PR #29 merged 2026-07-28) to upstream vllm-omni v0.26.0. Then build, validate, and prepare for the Aug 13 midstream tag deadline (RHOAI 3.6 EA1 tech preview).

## Repos and Branches

| Repo | Branch | Purpose |
|------|--------|---------|
| `neuralmagic/nm-vllm-omni-ent` | `sync-v0.26.0` (new) | Sync branch — merges upstream v0.26.0 + reapplied carries |
| `neuralmagic/nm-vllm-omni-ent` | `main` | PR target |
| `neuralmagic/nm-cicd` | `feature/vllm-omni-tp` | TP build pipeline (6 commits ahead of main) |
| `vllm-project/vllm-omni` | tag `v0.26.0` | Upstream release |
| `neuralmagic/nm-vllm-ent` | `main` | vLLM base wheel source (v0.26.0 wheel via INFERENG-9284) |

## Current State

- **midstream/main** is on v0.25.z (PR #29 merged 2026-07-28)
- **Upstream v0.26.0** is ~165 commits ahead of midstream/main's merge base
- **Carry commits** on midstream/main (20 commits with `[carry]` prefix) — these must be preserved

### Carry Commits to Reapply

These are the midstream-specific patches currently on `midstream/main`. They need to be cherry-picked or rebased onto the v0.26.0 sync branch. Some may apply cleanly, some may need conflict resolution, and some may have been upstreamed in v0.26.0 (check before reapplying).

```
198746a7 [carry] Relax dependency on av
9dc67c31 [carry] Drop fa3-fwd from requirements
5b482bee [carry] Re-apply downstream carries after upstream merge
e0dab07c [carry] Remove unregister_vllm_metrics monkey-patch warning log
0d426169 [carry] Extract shared serving_chat test helpers into tests/helpers/
8088e9af [carry] Add unit tests for non-streaming text/audio choice merging
2f1fed38 [carry] Simplify audio/text choice merge logic
66751cd2 [carry] Merge text and audio into single choice for non-streaming chat completions
a346daf7 [carry] Switch default image build runner from dind to util
acb68502 [carry] fix: return proper HTTP status codes for audio voice endpoints (#3969)
807f75f0 [carry] switch endpoint restrict fail to warn
610d4ad3 [carry] hack for single stage q3omni models (pr #3760)
5059a32a [carry] endpoint rejection (#4762)
cd1e2695 [carry] Update test imports for vLLM protocol refactoring
87318a70 [carry] Centralize audio format constants, fix diffusion path, add tests, update docs
cebf245f [carry] Preserve single-stage diffusion text path, add regression tests
db85f25f [carry] [Bugfix] Validate TTS speech, batch, and voice upload requests (#3649)
cfee3a42 [carry] [BugFix] Add modalities and logprobs validation for /v1/chat/completions
e4e926b3 [carry] [BugFix] Add input, audio_length, guidance_scale, and num_inference_steps validators for /v1/audio/generate
```

**Also on main but not `[carry]`-prefixed:**
```
2fe512ef INFREENG-8018: vLLM-Omni: create a single CUDA image
532f8181 Merge pull request #32 from neuralmagic/remove-test
```

### Non-carry midstream commits

The single-CUDA-image Dockerfile change (`2fe512ef`) and the test removal merge (`532f8181`) also need to land on the sync branch. These are midstream infrastructure, not carries against upstream code.

---

## Phase 1: Create Sync Branch and PR

### Step 1.1 — Merge upstream v0.26.0

**Strategy:** Do NOT use `-X theirs` blindly — we have carries to preserve. Instead:

1. Create `sync-v0.26.0` from `midstream/main`
2. Merge upstream `v0.26.0` tag **without** `-X theirs` (regular merge)
3. Resolve conflicts manually, preferring upstream for non-carry code and preserving carry intent
4. Remove `.github/` if upstream content leaked in

```bash
cd vllm-omni
git fetch upstream --tags
git checkout -b sync-v0.26.0 midstream/main
git merge v0.26.0 --no-edit
# Resolve conflicts...
git rm -rf .github/ 2>/dev/null
git commit  # if .github/ was removed
```

**Conflict resolution guidance:**
- Files touched ONLY by upstream → accept upstream
- Files touched ONLY by carries → keep our version
- Files touched by BOTH → manual merge, preserving carry intent on top of upstream changes
- If a carry was upstreamed in v0.26.0 (check the upstream changelog / PRs), drop the carry

### Step 1.2 — Verify carries survived

After merging, check that each carry's functionality is still present:

```bash
# Quick check: are the carry-modified files still carrying our changes?
git diff v0.26.0..sync-v0.26.0 --stat
# Should show our carry files as modified relative to upstream
```

Key files to verify:
- Dependency overrides (av relaxation, fa3-fwd removal)
- Endpoint restriction / validation code
- Audio/text choice merge logic
- Dockerfile changes (single CUDA image)
- Test helpers

### Step 1.3 — Update midstream metadata

```bash
# Update vllm-version if midstream/vllm-version exists
echo "0.26.0" > midstream/vllm-version

# Update vllm-wheels.yml with the v0.26.0 wheel run ID (from INFERENG-9284)
# TBD: need the actual run ID
```

### Step 1.4 — Push and PR

```bash
git push midstream sync-v0.26.0
# Create PR: sync-v0.26.0 → main
gh pr create --repo neuralmagic/nm-vllm-omni-ent \
  --base main --head sync-v0.26.0 \
  --title "Upstream sync: vllm-omni v0.26.0" \
  --body "..."
```

---

## Phase 2: Build

### Step 2.0 — Resolve vLLM v0.26.0 base wheel

INFERENG-9284 tracks the vLLM v0.26.0 midstream wheel build. Check its status:
- If a successful wheel run ID exists → reuse it
- If not → build one first via nm-cicd

```bash
# Check INFERENG-9284 status
jira issue view INFERENG-9284 --plain 2>&1 | head -20
```

### Step 2.1 — Trigger full-chain build

Use `midstream-build.yml` on nm-vllm-omni-ent, pointing nm-cicd at `feature/vllm-omni-tp`:

```bash
gh workflow run midstream-build.yml \
  --repo neuralmagic/nm-vllm-omni-ent \
  --ref sync-v0.26.0 \
  -f build_step=full-chain \
  -f nm_cicd_ref=feature/vllm-omni-tp \
  -f build_label_image=k8s-a100-util \
  -f vllm_run_id=<VLLM_V026_RUN_ID>
```

**Notes:**
- `--ref sync-v0.26.0` — build from the sync branch (or `main` after PR merges)
- `nm_cicd_ref=feature/vllm-omni-tp` — use the TP build pipeline
- Default wheel build label is `k8s-a100-build-13-0` (CUDA 13 for v0.26.0)
- Default image build label is `k8s-a100-util`

### Step 2.2 — Monitor build

```bash
# Get run ID
gh run list --repo neuralmagic/nm-vllm-omni-ent --workflow midstream-build.yml --limit 3

# Watch
gh run view <RUN_ID> --repo neuralmagic/nm-vllm-omni-ent --json status,conclusion,jobs \
  --jq '{status: .status, conclusion: .conclusion, jobs: [.jobs[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

---

## Phase 3: Smoke Test Matrix

The `midstream-build.yml` full-chain triggers smoke tests automatically via `test-vllm-omni.yml` (or `ocp-test.yml` if we pick up PR #31's migration).

### Model Matrix (7 models)

| Model | Type | GPUs | Label |
|-------|------|------|-------|
| Tongyi-MAI/Z-Image-Turbo | image_gen | 1 | img-cuda |
| Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice | tts | 1 | tts-cuda |
| black-forest-labs/FLUX.1-schnell | image_gen | 1 | flux1-cuda |
| black-forest-labs/FLUX.2-klein-4B | image_gen | 1 | flux2klein-cuda |
| black-forest-labs/FLUX.2-dev | image_gen (tp=2) | 2 | flux2dev-cuda |
| mistralai/Voxtral-4B-TTS-2603 | tts | 1 | voxtral-cuda |
| Qwen/Qwen3-Omni-30B-A3B-Instruct | omni | 2 | qwen3omni-cuda |

**PR #31 note:** PR #31 (`doug/infereng-9522-migrate-smoke-to-ocp-test`) migrates smoke tests from `test-vllm-omni.yml` to `ocp-test.yml` with updated field names (`validation_type` → `model_capabilities`, adds `deployment_type=vllm-omni`). We should pick this change into `sync-v0.26.0` or merge it to main first.

---

## Phase 4: Accept-Sync

Run vllm-omni accept-sync validation after build + smoke tests pass.

**TBD:** Document the accept-sync process for omni. This may be a manual validation step or an automated workflow — need to check what exists on `feature/vllm-omni-tp`.

---

## Phase 5: OCP Model Validation

Run the full OCP model validation suite against the built image.

**Target cluster:** `https://api.modelsibm.ibmmodel.rh-ods.com:6443` (from PR #31)

```bash
# Per-model dispatch (if not triggered automatically by midstream-build.yml)
gh workflow run ocp-test.yml \
  --repo neuralmagic/nm-cicd \
  --ref feature/vllm-omni-tp \
  -f vllm_image="quay.io/vllm/automation-vllm-omni:cuda-<RUN_ID>" \
  -f model="<MODEL>" \
  -f target_device=cuda \
  -f model_capabilities=<CAPABILITY> \
  -f deployment_type=vllm-omni \
  -f test_type=smoke \
  -f ocp_host_url=https://api.modelsibm.ibmmodel.rh-ods.com:6443 \
  -f label=ibm-wdc-k8s-h100-util \
  -f timeout=60
```

---

## Phase 6: Tag for AIPCC

After validation passes, tag per the RHAI convention:

```bash
# Convention: vX.Y.Z+rhaiv.N
git tag v0.26.0+rhaiv.1
git push midstream v0.26.0+rhaiv.1
```

**Note:** Check with Ricardo on the exact version convention. Previous tag was `v0.24.0+rhaiv.1`.

---

## Dependencies and Blockers

| Item | Status | Notes |
|------|--------|-------|
| vLLM v0.26.0 base wheel (INFERENG-9284) | Done | run_id=30396358991 |
| nm-cicd `feature/vllm-omni-tp` ready | Yes | 6 commits ahead of main |
| PR #31 (ocp-test migration) | Open | Pick into sync branch or merge first |
| FlashInfer version pins | TBD | Check if v0.26.0 needs a pin bump in nm-cicd |
| Constraints file (fastapi, transformers) | TBD | Verify current constraints still valid for v0.26.0 |
| Upstream carries upstreamed? | TBD | Check v0.26.0 changelog for any carries that landed |

## nm-cicd Dependency Checks for v0.26.0

Before building, verify these on `feature/vllm-omni-tp`:

1. **FlashInfer pins** — `neuralmagic/requirements/vllm-omni-cuda.txt` and `vllm-omni-cuda-develsdk.txt`. Current: 0.6.12. Check what v0.26.0 vLLM expects.
2. **Constraints** — `neuralmagic/constraints/vllm-omni.txt`. Current: `fastapi < 0.137.0`, `transformers < 5.13.0`. Verify still needed.
3. **Runner labels** — wheel: `k8s-a100-build-13-0`, image: `k8s-a100-util`. Should be fine.

---

## Execution Log

_Updated as we execute each phase._

| Phase | Step | Status | Run ID / PR | Notes |
|-------|------|--------|-------------|-------|
| 1 | Create sync-v0.26.0 branch | Done | — | From midstream/main |
| 1 | Merge upstream v0.26.0 | Done | — | 3 conflicts resolved manually |
| 1 | Verify carries | Done | — | All 20 carry commits preserved |
| 1 | Push + PR | Done | [PR #33](https://github.com/neuralmagic/nm-vllm-omni-ent/pull/33) | sync-v0.26.0 → main |
| 2 | Resolve vLLM base wheel | Done | 30396358991 | INFERENG-9284 |
| 2 | Trigger full-chain build (1st) | Failed | [30844594526](https://github.com/neuralmagic/nm-vllm-omni-ent/actions/runs/30844594526) | Wheels+image OK; smoke tests failed — test-vllm-omni.yml removed from feature/vllm-omni-tp |
| 2 | Fix: migrate smoke to ocp-test | Done | commit 7e3725dd | Switched dispatch to ocp-test.yml, updated matrix fields, default nm_cicd_ref |
| 2 | Trigger full-chain build (2nd) | Done | [30847383483](https://github.com/neuralmagic/nm-vllm-omni-ent/actions/runs/30847383483) | Reusing wheels from 1st run |
| 3 | Smoke tests (7 models) | **7/7 PASSED** | image: `cuda-30847639332` | img, tts, flux1, flux2klein, flux2dev, voxtral, qwen3omni all green |
| 4 | Accept-sync (1st, Tarun) | Partial fail | [30917549974](https://github.com/neuralmagic/nm-cicd/actions/runs/30917549974) | 21 passed, 2 failed (missing imports in test_stream_finish_reason.py), 2 infra (cosmos3 network) |
| 4 | Fix: add missing imports | Done | commit d0a04302 | OmniRequestOutput, RequestOutput, CompletionOutput etc. not imported after merge |
| 4 | Accept-sync (2nd) | Running | [30922122189](https://github.com/neuralmagic/nm-cicd/actions/runs/30922122189) | Retriggered with import fix |
| 5 | OCP model validation | Not started | — | — |
| 6 | Tag for AIPCC | Skipped | — | Blocked by other work |

---

## Lessons Learned (v0.26.0 Sync)

### Merge strategy: regular merge, not `-X theirs`

For vllm-omni syncs, do NOT use `-X theirs`. Unlike nm-vllm-ent (where carries are minimal and upstream is always right), nm-vllm-omni-ent has ~20 carry commits touching real production code — endpoint validation, audio/text merging, dependency overrides. A regular `git merge` with manual conflict resolution is the right call. In this sync (~165 upstream commits), only 3 files conflicted and all were straightforward to resolve.

### Conflict patterns to expect

1. **Test mock attributes** — upstream refactors rename or add attributes on mocked objects. Our carry test helpers need updating to match the new API surface. Look at what upstream changed in the class under test and adjust mock setup accordingly.
2. **Carry logic + upstream error handling** — upstream may add error checking (e.g. `isinstance(result, ErrorResponse)`) around methods our carries also modified. Combine both: keep the error check AND our carry's downstream logic (e.g. `audio_choices.extend(choices_data)`).
3. **Test resilience wrappers** — upstream sometimes wraps test assertions in `try/except (AttributeError, TypeError)` to tolerate API churn. Take those — they're strictly more robust.

### Merge conflicts can silently drop imports

When a test file has both upstream-added code and carry-added code, conflict resolution may keep both sets of *functions* but only one set of *imports*. In this sync, `test_stream_finish_reason.py` had upstream's local helper functions (`_make_text_omni_output`, `_make_audio_omni_output`) that reference `OmniRequestOutput`, `RequestOutput`, `CompletionOutput`, etc. — but the conflict resolution kept our carry's import block without adding upstream's needed imports. The error only surfaces at test *collection* time (not at merge time), so it's easy to miss.

**Post-merge checklist:** After resolving conflicts in any test file, run `python -c "import ast; ast.parse(open('<file>').read())"` to catch syntax issues, then grep for every type annotation and class reference in the file to verify it's imported.

### nm-cicd branch alignment is critical

The `midstream-build.yml` on nm-vllm-omni-ent dispatches workflows on nm-cicd. If the nm-cicd branch you're targeting has removed or renamed workflows, every smoke test will fail with HTTP 422. **Before triggering a build, verify the dispatch target workflow exists on the nm-cicd branch:**

```bash
gh api repos/neuralmagic/nm-cicd/contents/.github/workflows/test-vllm-omni.yml?ref=feature/vllm-omni-tp --jq '.name' 2>&1
# If this 404s, the workflow was removed — update midstream-build.yml to use the replacement
```

In this sync, `feature/vllm-omni-tp` had removed `test-vllm-omni.yml` in favor of `ocp-test.yml`. Cost us one full image rebuild cycle (~30 min) to discover and fix.

### Reuse wheels aggressively on retrigger

When smoke tests fail but wheels+image succeeded, pass both `vllm_run_id` AND `omni_run_id` on retrigger to skip wheel builds entirely. The image still rebuilds (needed if `midstream-build.yml` changed), but you save ~20 min of wheel compilation:

```bash
gh workflow run midstream-build.yml \
  --repo neuralmagic/nm-vllm-omni-ent \
  --ref sync-v0.26.0 \
  -f build_step=full-chain \
  -f vllm_run_id=<VLLM_RUN_ID> \
  -f omni_run_id=<OMNI_RUN_ID> \
  -f nm_cicd_ref=feature/vllm-omni-tp
```

Get the run IDs from the failed build's logs: `grep "Found run:"` in the omni-wheel and docker-image job outputs.

### Check for upstreamed carries before reapplying

Before each sync, scan the upstream release notes and diff for any carries that may have landed upstream. If a carry was upstreamed, drop it from midstream to avoid divergence. Quick check:

```bash
# Diff our carries against upstream — if a carry file shows no diff, it was likely upstreamed
git diff v0.26.0..sync-v0.26.0 -- <carry-file>
```

In this sync, none of our carries had been upstreamed yet, but this will become more likely as the upstream project matures.

### Smoke test field mapping (test-vllm-omni → ocp-test)

If migrating smoke tests from `test-vllm-omni.yml` to `ocp-test.yml`:

| Old field (test-vllm-omni) | New field (ocp-test) |
|---------------------------|---------------------|
| `validation_type: image_gen` | `model_capabilities: image_generation` |
| `validation_type: tts` | `model_capabilities: speech_generation` |
| `validation_type: omni` | `model_capabilities: omni_chat` |
| `tts_voice: vivian` | (handled by model config) |
| — | `deployment_type: vllm-omni` (new, required) |
| — | `test_type: smoke` (new, required) |
| — | `config_ref: main` (new) |
| — | `ocp_host_url: https://api.modelsibm.ibmmodel.rh-ods.com:6443` (new) |
| — | `wf_category: DEBUG` (new) |

### Timeline for a clean sync

Assuming no blockers and existing vLLM base wheel:

| Step | Wall clock |
|------|-----------|
| Merge + conflict resolution | ~15 min |
| Push + PR | ~2 min |
| Omni wheel build | ~15 min |
| Docker image bake | ~25-30 min |
| Smoke tests (7 models, parallel) | ~20-30 min |
| **Total** | **~75-90 min** |

Add ~20 min if you need to build a fresh vLLM base wheel. Add ~30 min per retrigger cycle if something fails.
