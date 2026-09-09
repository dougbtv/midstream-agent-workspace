# Midstream Release Blocker Bug Fix

How to investigate, fix, validate, and ship a blocker bug in nm-vllm-ent midstream — from JIRA triage through upstream cherry-pick, control/fix testing on GPU hardware, OCP smoke test, and PR creation. Follows the same workflow used for INFERENG-6229 and INFERENG-6236.

## Human-Agent Flow

This skill is a collaborative workflow for fixing bugs that block a midstream release tag (e.g. `v0.18.0+rhaiv.5`). The human provides the bug context and hardware access; the agent drives investigation, fix, build, validation, JIRA updates, and PRs.

### What the human provides

1. **JIRA issue** — the blocker bug ticket (e.g. `INFERENG-6229`)
2. **Release branch** — the nm-vllm-ent branch to fix (e.g. `rhai/0.18.0`)
3. **Upstream references** — related upstream issues or PRs in `vllm-project/vllm`
4. **Dev box access** — hostname, username, GPU availability (e.g. `dougbtv@a100-07`)
5. **Model to reproduce with** — the model and serve flags from the bug report

### What the agent does

0. **Triages** — reads the JIRA issue, related tickets, upstream issues, and the validation tracking doc to understand the full context
1. **Investigates** — traces the bug through the codebase, identifies root cause, finds upstream fix candidates
2. **Creates the fix branch** — cherry-picks the upstream fix onto a new branch based on latest release branch
3. **Kicks off the build** — dispatches `build-whl-image.yml` via nm-cicd, sets up monitoring crons
4. **Runs the control test** — deploys the pre-fix image on GPU hardware, reproduces the bug, logs the result
5. **Runs the fix test** — deploys the fix image, verifies the bug is resolved, logs the result
6. **Dispatches OCP smoke test** — runs `model-validation-ocp.yml` against the fix image to check for regressions
7. **Updates JIRA** — comments at each milestone (investigation, build, control test, fix verified, PRs opened)
8. **Creates PRs** — one targeting the release branch, one targeting `main`, both with validation details
9. **Comments on upstream** — adds reproduction details and validation results to the upstream PR
10. **Monitors OCP results** — hourly cron watches the smoke test, posts final JIRA update when done
11. **Cleans up** — stops containers, releases GPUs, deletes crons

### Expected deliverables

- Bug reproduced with control test (pre-fix image, logged 400/crash/error)
- Fix verified with fix test (post-fix image, logged successful response)
- OCP smoke test passing (no regressions)
- Two PRs in nm-vllm-ent (release branch + main) with validation details
- JIRA issue fully documented with investigation, reproduction, fix, and PR links
- Upstream PR commented with downstream validation results
- GPUs released, containers stopped, crons cleaned up

### Terminology

- **upstream** = `vllm-project/vllm` (the open source vLLM repo)
- **midstream** = `neuralmagic/nm-vllm-ent` (our fork, builds, and images)
- **control test** = deploying the pre-fix image to reproduce the bug (proves the bug exists)
- **fix test** = deploying the post-fix image to verify the fix works
- **OCP smoke test** = `model-validation-ocp.yml` — deploys models on OpenShift and runs inference checks

## Phase 1: Investigation

### Triage the JIRA landscape

Use the `jira` CLI to read the bug, its parent story, sibling bugs, and any linked upstream work items. Understand what's already been tried and who's involved.

```bash
jira issue view INFERENG-XXXX
jira issue view INFERENG-XXXX --comments 5
```

### Search upstream for existing fixes

```bash
# Search for related PRs
gh search prs --repo vllm-project/vllm "<keyword>" --limit 10 --json title,number,state,url

# Check a specific upstream issue for comments pointing to fixes
gh issue view <number> --repo vllm-project/vllm --json title,state,body,comments \
  --jq '{title: .title, state: .state, comment_count: (.comments | length), latest_comments: [.comments[-3:][] | {author: .author.login, body: (.body | .[0:300])}]}'

# Check a candidate fix PR
gh pr view <number> --repo vllm-project/vllm --json title,state,mergedAt
gh pr diff <number> --repo vllm-project/vllm
```

### Trace the bug in the codebase

```bash
# Search for the error message or symbol in the release branch
git fetch origin rhai/0.18.0
git grep -n "reasoning_effort" origin/rhai/0.18.0 -- '*.py'

# Read the relevant code
git show origin/rhai/0.18.0:vllm/tokenizers/mistral.py | sed -n '430,460p'
```

### Post investigation comment on JIRA

Write the comment body to a temp file, then use the jira CLI. Never use inline heredocs with `jira issue comment add` — they hang. Use plain markdown (not Jira wiki markup).

```bash
cat > /tmp/jira-comment.txt << 'EOF'
### Investigation update — root cause identified

<details of what you found, code references, upstream PR candidates>
EOF

jira issue comment add INFERENG-XXXX --template /tmp/jira-comment.txt --no-input
```

## Phase 2: Create the Fix Branch

### Cherry-pick the upstream fix

Always cherry-pick — never manually re-implement the fix. Both the release branch and main PRs should contain the exact same commit.

```bash
# Create branch from latest release branch
git fetch origin rhai/0.18.0
git checkout -b doug/INFERENG-XXXX-short-description origin/rhai/0.18.0

# If the upstream PR is merged, get its merge commit
gh pr view <number> --repo vllm-project/vllm --json mergeCommit --jq '.mergeCommit.oid'

# If the fix is small and the upstream PR is unmerged, apply the diff directly
gh pr diff <number> --repo vllm-project/vllm | git apply
git add -A && git commit -m "$(cat <<'COMMIT'
Fix <short description>

Cherry-pick of vllm-project/vllm#NNNNN.
Fixes: INFERENG-XXXX

Co-authored-by: Claude Opus 4.6 <noreply@anthropic.com>
COMMIT
)"

# Push the fix branch (NEVER push directly to rhai/0.18.0 or main)
git push -u origin doug/INFERENG-XXXX-short-description
```

### Create the main branch cherry-pick

```bash
git fetch origin main
git checkout -b doug/INFERENG-XXXX-short-description-main origin/main
git cherry-pick <fix-commit-sha>
git push -u origin doug/INFERENG-XXXX-short-description-main
```

## Phase 3: Build

### Create matching nm-cicd branch

The `--ref` flag in the build workflow dispatch refers to the nm-cicd branch. It must exist on the remote.

```bash
# Create nm-cicd branch off main via GitHub API
gh api repos/neuralmagic/nm-cicd/git/refs \
  -f ref="refs/heads/doug/INFERENG-XXXX-short-description" \
  -f sha="$(gh api repos/neuralmagic/nm-cicd/branches/main --jq '.commit.sha')"
```

### Dispatch the build

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-XXXX-short-description \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=doug/INFERENG-XXXX-short-description \
  -f build_label=k8s-a100-build-12-9 \
  -f build_timeout=120 \
  -f image_label=ibm-wdc-k8s-h100-dind \
  -f python=3.12 \
  -f release_image=false \
  -f target_device=cuda
```

### Get the run ID

```bash
sleep 8 && gh run list --repo neuralmagic/nm-cicd --workflow build-whl-image.yml --limit 3 \
  --json databaseId,headBranch,status,conclusion,displayTitle
```

### Monitor the build with a self-deleting cron

Set up a cron (every 20 min) that checks build status and extracts image tags on success:

```bash
# Check status
gh run view <RUN_ID> --repo neuralmagic/nm-cicd --json status,conclusion,jobs \
  --jq '{status: .status, conclusion: .conclusion, jobs: [.jobs[] | {name: .name, status: .status, conclusion: .conclusion}]}'

# On success, extract image tags
gh api repos/neuralmagic/nm-cicd/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[] | select(.name | contains("image")) | .id'

gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' | grep "naming to quay"
```

Image tags follow the pattern:
- `quay.io/vllm/automation-vllm:<version>` (version tag)
- `quay.io/vllm/automation-vllm:cuda-<run-id>` (run ID tag — use this one for testing)

### Post build comment on JIRA

Include the build run link, the dispatch command, the fix commit, and the validation plan.

## Phase 4: Control Test (Reproduce the Bug)

The control test proves the bug exists in the pre-fix image. This is critical evidence for the JIRA trail.

### Reserve GPUs

```bash
ssh <user>@<host> "chg reserve -G 0,1,2,3 -d 4h"
```

### Download the model

Start in the background while the build runs:

```bash
ssh <user>@<host> "HF_HOME=/mnt/nvme-data/<user>/hub_cache \
  nohup uv tool run --from huggingface_hub hf download <model_id> \
  > /tmp/hf-download.log 2>&1 &"
```

### Pull and deploy the pre-fix image

```bash
ssh <user>@<host> "podman pull <pre-fix-image>"

ssh <user>@<host> "podman run -d \
  --name vllm-<model>-control \
  --user 0:0 \
  --device nvidia.com/gpu=0 --device nvidia.com/gpu=1 \
  --device nvidia.com/gpu=2 --device nvidia.com/gpu=3 \
  --security-opt=label=disable \
  --shm-size=10g \
  -p 8000:8000 \
  -v /mnt/nvme-data/<user>/hub_cache:/hf \
  -e HF_HUB_OFFLINE=1 -e FLASHINFER_DISABLE_VERSION_CHECK=1 \
  -e HF_HOME=/hf -e CUDA_VISIBLE_DEVICES=0,1,2,3 \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --entrypoint /bin/bash \
  <pre-fix-image> \
  -c 'rm -rf /opt/vllm/lib64/python3.12/site-packages/flashinfer_jit_cache/jit_cache && exec python3 -m vllm.entrypoints.openai.api_server \
    --model <model_id> <serve-flags>'"
```

### Fire the test request

```bash
# Wait for health
ssh <user>@<host> "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/health"

# Fire the request that triggers the bug
ssh <user>@<host> 'curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '"'"'{"model": "<model_id>", "messages": [{"role": "user", "content": "Hello"}], "max_tokens": 128}'"'"''
```

Log the full error response. Stop and remove the container.

```bash
ssh <user>@<host> "podman stop vllm-<model>-control && podman rm vllm-<model>-control"
```

### Post control test JIRA comment

Include the image tag, serve command, request, and the full error response.

## Phase 5: Fix Test (Verify the Fix)

Same process as the control test but with the fix image. Expect a successful response.

```bash
ssh <user>@<host> "podman pull <fix-image>"

# Deploy with same flags as control test, just different image
ssh <user>@<host> "podman run -d --name vllm-<model>-fix ..."

# Fire the same request — expect 200 with model output
ssh <user>@<host> 'curl -s http://127.0.0.1:8000/v1/chat/completions ...'

# Clean up
ssh <user>@<host> "podman stop vllm-<model>-fix && podman rm vllm-<model>-fix"
ssh <user>@<host> "chg release -G 0,1,2,3"
```

### Post fix verification JIRA comment

Include before/after comparison showing the control error and the fix success.

## Phase 6: OCP Smoke Test

Dispatch the OCP model validation workflow to verify the fix image doesn't introduce regressions.

```bash
gh workflow run model-validation-ocp.yml \
  --repo neuralmagic/nm-cicd \
  --ref doug/INFERENG-XXXX-short-description \
  -f config_ref=main \
  -f label=ubuntu-latest \
  -f timeout=120 \
  -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_cuda_minimal.yml \
  -f vllm_image=quay.io/vllm/automation-vllm:cuda-<RUN_ID> \
  -f source_ref=doug/INFERENG-XXXX-short-description \
  -f run_accuracy_check=false
```

Set up an hourly cron to monitor the result:

```bash
gh run view <OCP_RUN_ID> --repo neuralmagic/nm-cicd --json status,conclusion,jobs \
  --jq '{status: .status, conclusion: .conclusion, jobs: [.jobs[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

Post a JIRA comment when the OCP test completes — pass or fail.

## Phase 7: PRs and Upstream

### Create PRs

Create two PRs in nm-vllm-ent — one targeting the release branch, one targeting `main`. Both should contain the exact same cherry-picked commit.

PR body should include:
- Summary of the bug and root cause
- Link to upstream PR being cherry-picked
- Link to the JIRA issue
- Validation results (control test, fix test, OCP smoke test)
- Note that merge should wait for OCP results if still in progress

```bash
gh pr create --repo neuralmagic/nm-vllm-ent \
  --head doug/INFERENG-XXXX-short-description \
  --base rhai/0.18.0 \
  --title "[rhai/0.18.0] Fix <short description>" \
  --body "$(cat <<'EOF'
## Summary
...
## Validation
...
## Test plan
...
EOF
)"
```

### Comment on upstream PR

Add a comment to the upstream PR with reproduction conditions and validation results. This helps the upstream maintainers see real-world impact and may accelerate the merge.

### Post PR links on JIRA

Final JIRA comment with links to both PRs and the OCP smoke test run.

## Gotchas and Lessons Learned

### Rootless podman UID remapping

The container runs as `vllm` (UID 2000) which gets remapped in rootless podman. Host files become unreadable. Fix: use `--user 0:0` (maps to host user's UID in rootless mode) with `--security-opt=label=disable`. Do NOT use `:Z` volume flag — it triggers slow SELinux relabeling and doesn't fix the UID issue.

### FlashInfer stale JIT cache

The `flashinfer_jit_cache/jit_cache/` directory inside the image may contain `.so` files compiled against a different CUDA version (e.g. `libcudart.so.13`). This causes `Failed to load dynamic shared library` crashes on startup. Fix: clear the stale cache at container start:

```bash
--entrypoint /bin/bash \
<image> \
-c 'rm -rf /opt/vllm/lib64/python3.12/site-packages/flashinfer_jit_cache/jit_cache && exec python3 -m vllm.entrypoints.openai.api_server ...'
```

Do NOT mount an empty dir over the entire `flashinfer_jit_cache` package — that wipes the Python module and causes `AttributeError: module 'flashinfer_jit_cache' has no attribute '__version__'`.

### JIRA CLI formatting

- Write comment body to a temp file, then `jira issue comment add INFERENG-XXXX --template /tmp/file.txt --no-input`
- Use plain markdown (not Jira wiki markup) — the CLI converts to ADF automatically
- Never use inline heredocs with `jira issue comment add` — they hang
- Never use `$'...\n...'` for body content — `\n` renders as literal text

### Health endpoint

vLLM's health endpoint returns an empty body with HTTP 200. Use:
```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/health
```

### Model download parallelization

Start the model download on the dev box as soon as you have access — don't wait for the build. The build takes 45-75 min; the download takes 1-5 min. Use `nohup` and background it:
```bash
ssh <user>@<host> "HF_HOME=<path> nohup uv tool run --from huggingface_hub hf download <model> > /tmp/download.log 2>&1 &"
```

Note: the `huggingface_hub` CLI binary is `hf` (not `huggingface-cli`) in recent versions.

### Never push directly to release branches

Always create feature branches (`doug/INFERENG-XXXX-...`) and open PRs. Never push directly to `rhai/0.18.0`, `main`, or any shared branch.

### Never use branch=main for Buildkite builds

This clutters the main CI view, annoys maintainers, and the record can't be deleted.

## Cron Strategy

Use self-deleting crons to automate the monitoring pipeline. Consolidate where possible — one cron per logical phase, not one per sub-step.

| Phase | Interval | Purpose |
|-------|----------|---------|
| Build monitor | 20 min | Watch `build-whl-image.yml` run, extract image tag on success |
| Fix verification | 5 min | Unified: image pull → container deploy → health check → test → cleanup |
| OCP monitor | hourly | Watch `model-validation-ocp.yml`, post JIRA update on completion |

Each cron self-deletes when its phase completes. If a step fails, the cron investigates and retries rather than silently deleting.

## End-to-End Checklist

1. [ ] Read JIRA issue and related tickets
2. [ ] Search upstream for existing fixes
3. [ ] Trace root cause in codebase
4. [ ] Post investigation comment on JIRA
5. [ ] Cherry-pick upstream fix onto release branch
6. [ ] Create nm-cicd branch
7. [ ] Dispatch build, set up build monitor cron
8. [ ] Download model on dev box (parallel with build)
9. [ ] Pull pre-fix image, run control test, log error
10. [ ] Post control test comment on JIRA
11. [ ] Pull fix image, run fix test, log success
12. [ ] Post fix verification comment on JIRA
13. [ ] Dispatch OCP smoke test, set up hourly monitor cron
14. [ ] Post OCP dispatch comment on JIRA
15. [ ] Create PR targeting release branch
16. [ ] Cherry-pick onto main, create PR targeting main
17. [ ] Comment on upstream PR with validation results
18. [ ] Post PR links comment on JIRA
19. [ ] OCP smoke test passes — post final JIRA update
20. [ ] Clean up: containers stopped, GPUs released, crons deleted

Typical wall-clock time: ~2-3 hours (investigation 15-30 min, build 45-75 min, control+fix tests 15-30 min, OCP 1-2 hours).

## Reference: INFERENG-6229 (Mistral 4 reasoning_effort)

This skill was developed from the INFERENG-6229 workflow. Key details for reference:

- **Bug:** `reasoning_effort` kwarg unconditionally passed to `MistralCommonTokenizer.apply_chat_template()` even when `None`, breaking all Mistral 4 chat completions
- **Root cause:** `vllm/tokenizers/mistral.py` line 450 — `version_kwargs["reasoning_effort"] = kwargs.get("reasoning_effort")` inside `if self.version >= 15`
- **Fix:** Only add to `version_kwargs` when not `None` (3-line change)
- **Upstream PR:** [vllm-project/vllm#38448](https://github.com/vllm-project/vllm/pull/38448)
- **Sibling bug pattern:** INFERENG-6236 (Mistral-7B tool calling JSONDecodeError) used the identical workflow — upstream PR [#40531](https://github.com/vllm-project/vllm/pull/40531), cherry-picks to both branches

---
name: midstream-release-blocker-bug
description: Fix a blocker bug in nm-vllm-ent midstream release. Use when investigating, fixing, and validating bugs that block a release tag — includes upstream cherry-pick, control/fix testing on GPU hardware, OCP smoke test, JIRA updates, and PR creation. Trigger when: user mentions a blocker bug in INFERENG, needs to cherry-pick an upstream vLLM fix, or wants to validate a bugfix on GPU hardware before merging.
---
