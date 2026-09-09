---
name: upstream-pr
description: Manage upstream vLLM pull requests end-to-end — cherry-picking fixes onto nm-vllm-ent branches, running before/after validation on dev GPU boxes, triggering Buildkite CI and OCP smoke tests, and handling PR review comments via the GitHub CLI. Use this skill whenever the user mentions upstream PRs, cherry-picking commits, before/after testing, Buildkite builds for upstream branches, OCP smoke tests, or responding to PR review feedback on vllm-project/vllm.
---

# Upstream PR Workflow

End-to-end skill for managing upstream vLLM PRs: from opening and iterating on review feedback, to cherry-picking onto nm-vllm-ent release branches, running before/after validation, and driving CI.

## Prerequisites

- `gh` CLI authenticated with access to `vllm-project/vllm` and `neuralmagic/nm-cicd`
- `bk` CLI (Buildkite) authenticated for the `vllm` org
- `jira` CLI configured for Red Hat Atlassian
- SSH access to a dev GPU box (e.g., `dougbtv@a100-07`)
- The `dougbtv/vllm` fork as a remote in nm-vllm-ent (`dougbtv` remote)

## Commit and Push Requirements

All commits must be signed off:
```bash
git commit -s -m "Your message

Signed-off-by: dougbtv <dosmith@redhat.com>
Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

Always run pre-commit before committing:
```bash
pre-commit run --files <changed-files>
# Or for all files:
pre-commit run --all-files
```

## Part 1: Opening and Managing Upstream PRs

### Creating the PR

Work on a branch in the `dougbtv/vllm` fork, push to the `dougbtv` remote:
```bash
git push dougbtv <branch-name>
```

Create the PR against `vllm-project/vllm`:
```bash
gh pr create --repo vllm-project/vllm \
  --title "[Bugfix][Parser] Fix whatever" \
  --body "description here"
```

### Checking PR status

```bash
# Overview
gh pr view <number> --repo vllm-project/vllm

# Structured status
gh pr view <number> --repo vllm-project/vllm \
  --json state,mergeable,mergeStateStatus,reviews,reviewRequests

# CI checks
gh pr checks <number> --repo vllm-project/vllm
```

### Responding to review comments

Finding review comment IDs:
```bash
# List inline review comments
gh api repos/vllm-project/vllm/pulls/<number>/comments \
  --jq '.[] | "\(.id) \(.user.login) L\(.line // "?") \(.path): \(.body[:100])"'

# List top-level PR comments
gh api repos/vllm-project/vllm/issues/<number>/comments \
  --jq '.[] | "\(.id) \(.user.login): \(.body[:100])"'
```

Replying to an inline review comment (use `in_reply_to`):
```bash
gh api repos/vllm-project/vllm/pulls/<number>/comments \
  -X POST \
  -F in_reply_to=<comment_id> \
  -f body='Your reply here'
```

Posting a top-level PR comment:
```bash
gh pr comment <number> --repo vllm-project/vllm --body "Your comment"
```

Note: for `in_reply_to` replies, single quotes inside the body need shell escaping (`'"'"'`).

### Posting before/after results as a PR comment

Use a heredoc for multi-line markdown with tables:
```bash
gh pr comment <number> --repo vllm-project/vllm --body "$(cat <<'EOF'
**Before/after validation (Model-Name, GPU-Type)**

- **Before**: `<nightly-image-tag>`
- **After**: `<fixed-image-tag>`

| | Nightly | Fixed |
|---|---|---|
| Successful | **0** | **26** |
| Errors | 13 | 0 |

All unit tests pass in CI (build #XXXXX).
EOF
)"
```

## Part 2: Triggering Buildkite CI

Use the BK API to trigger a build on the upstream `ci` pipeline for your fork branch. Never use `--branch=main` — it clutters the main view and can't be deleted.

**Important:** Use the full 40-char commit SHA (not `HEAD`) and set `BUILDKITE_PULL_REQUEST` to the PR number — without these, BK's bootstrap can't fetch the fork branch.

```bash
bk api --method POST /pipelines/ci/builds --data '{
  "commit": "<full 40-char SHA>",
  "branch": "dougbtv:<branch-name>",
  "ignore_pipeline_branch_filters": true,
  "message": "<commit message>",
  "env": {
    "BUILDKITE_PULL_REQUEST": "<PR_NUMBER>"
  }
}'
```

### Monitoring BK builds

```bash
# Check build state and jobs
bk api '/pipelines/ci/builds/<number>' | python3 -c "
import json, sys
b = json.load(sys.stdin)
print(f'State: {b[\"state\"]}')
jobs = b.get('jobs', [])
for j in jobs:
    name = j.get('name','') or j.get('label','')
    state = j.get('state','')
    if state not in ('blocked', 'waiting') and j.get('type') == 'script':
        print(f'  [{state:8s}] {name}')"

# Check specific job logs (e.g., tool parser tests in CPU step)
bk api '/pipelines/ci/builds/<number>/jobs/<job-id>/log' | python3 -c "
import json, sys
data = json.load(sys.stdin)
for line in data.get('content','').split('\n'):
    if 'PASSED' in line or 'FAILED' in line:
        print(line[:200])"
```

### Retrying failed BK jobs

```bash
bk api --method PUT '/pipelines/ci/builds/<number>/jobs/<job-id>/retry'
```

Each job can only be retried once. If a second retry is needed, trigger a whole new build.

### Saving resources: cancel after target job completes

A full CI build runs 200+ jobs across CPU, GPU, and AMD queues. When you only need to validate a specific test (e.g., tokenizer changes), let the relevant job finish and then cancel the rest:

1. Identify which job runs your test. For `tests/tokenizers_/`, it's **"Async Engine, Inputs, Utils, Worker, Config (CPU)"** (command 8/12: `pytest -v -s tokenizers_`). CPU jobs start immediately — no Docker image build needed.
2. Monitor that job until it passes/fails.
3. Cancel the build to free up the remaining blocked/waiting jobs:
   ```bash
   bk api --method PUT '/pipelines/ci/builds/<number>/cancel'
   ```

You can't cancel individual jobs via the API — it's all or nothing. So wait for your job to finish first.

### Key CI steps to watch

- **:docker: Build image** — produces the test image at `public.ecr.aws/q9t5s3a7/vllm-ci-test-repo:<commit-sha>`
- **Async Engine, Inputs, Utils, Worker, Config (CPU)** — runs tokenizer, tool parser, reasoning parser, config unit tests (CPU-only, starts immediately)

## Part 3: Cherry-Picking onto nm-vllm-ent

When cherry-picking specific commits (not full branch merges) onto an nm-vllm-ent release branch:

```bash
# In the nm-vllm-ent repo
git fetch dougbtv <upstream-branch>
git checkout <target-branch>  # e.g., doug/rhai-0.18.0-mistral-fix based on rhai/0.18.0

# Cherry-pick specific commits (NOT the whole branch)
git cherry-pick <commit-sha-1> <commit-sha-2> ...
```

Important: do NOT `git merge` an entire upstream branch — that drags in the full upstream diff. Cherry-pick individual commits only.

### Resolving cherry-pick conflicts

Cherry-pick conflicts are common when the nm-vllm-ent base differs from upstream. Common resolutions:
- Missing imports: add them manually (check what the target branch already has)
- Missing constants/fixtures: add them inline
- Logger format differences: take whichever version matches the target branch style

After resolving:
```bash
git add <resolved-files>
git cherry-pick --continue
```

### Building the cherry-picked branch

The nm-cicd branch must match the nm-vllm-ent branch name. See the midstream-build-from-upstream skill for full build-whl-image.yml dispatch details.

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<nm-vllm-ent-branch> \
  -f build_label=k8s-a100-build-12-9 \
  -f build_timeout=120 \
  -f image_label=ibm-wdc-k8s-h100-dind \
  -f python=3.12 \
  -f release_image=false \
  -f target_device=cuda
```

Get the resulting image tags:
```bash
JOB_ID=$(gh api repos/neuralmagic/nm-cicd/actions/runs/<run-id>/jobs \
  --jq '.jobs[] | select(.name | contains("image")) | .id')
gh api repos/neuralmagic/nm-cicd/actions/jobs/$JOB_ID/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' | grep "naming to quay"
```

## Part 4: Before/After Testing on Dev GPU Boxes

### SSH access

Use `dougbtv@<hostname>` (not `doug@`). If SSH fails with "Too many authentication failures", the SSH agent has too many keys loaded — specify the key explicitly or reduce agent keys.

### GPU reservation

```bash
ssh dougbtv@<host> 'chg status'           # Check availability
ssh dougbtv@<host> 'chg reserve -G 0 -d 4h'  # Reserve GPU 0 for 4 hours
ssh dougbtv@<host> 'chg release -G 0'     # Release when done
```

### Rootless podman + detached containers

Rootless podman containers die when the SSH session that started them ends (Linger=no). The conmon process gets SIGKILL (exit 137).

The fix: run the entire test as a single SSH session using a script. Do NOT start a detached container in one SSH call and test it in another.

### Before/after test script pattern

Write a script to the remote box that runs both tests in one session:

```bash
ssh dougbtv@<host> 'cat > /tmp/before-after-test.sh << '"'"'SCRIPT'"'"'
#!/bin/bash
BEFORE_IMAGE="<nightly-image>"
AFTER_IMAGE="<fixed-image>"
COMMON_ARGS="--device nvidia.com/gpu=0 --security-opt=label=disable --ipc=host -p 8000:8000 -v /mnt/nvme-data/dougbtv/hub_cache:/root/.cache/huggingface -e CUDA_VISIBLE_DEVICES=0 -e HF_HUB_OFFLINE=1"
SERVER_CMD="python3 -m vllm.entrypoints.openai.api_server --model <model-id> --max-model-len 4096 --enforce-eager --host 0.0.0.0 --port 8000"

wait_for_server() {
    local container=$1 max_wait=120 waited=0
    while [ $waited -lt $max_wait ]; do
        if podman logs "$container" 2>&1 | grep -q "Application startup complete"; then
            echo "Server ready after ${waited}s"; sleep 2; return 0
        fi
        if ! podman ps --format "{{.Names}}" 2>/dev/null | grep -q "$container"; then
            echo "ERROR: Container $container exited!"; return 1
        fi
        sleep 5; waited=$((waited + 5))
    done
    echo "ERROR: timeout"; return 1
}

run_test() {
    local label=$1 image=$2 container=$3 results=$4
    podman rm -f "$container" 2>/dev/null || true
    podman run -d --name "$container" $COMMON_ARGS "$image" $SERVER_CMD
    if wait_for_server "$container"; then
        python3 /tmp/tool-test.py 2>&1 | tee "$results"
    else
        echo "$label failed to start" | tee "$results"
    fi
    podman stop "$container" 2>/dev/null; podman rm -f "$container" 2>/dev/null; sleep 3
}

run_test "BEFORE" "$BEFORE_IMAGE" "vllm-before" "/tmp/before-results.txt"
run_test "AFTER" "$AFTER_IMAGE" "vllm-after" "/tmp/after-results.txt"
echo "=== BEFORE ==="; cat /tmp/before-results.txt
echo "=== AFTER ==="; cat /tmp/after-results.txt
SCRIPT
chmod +x /tmp/before-after-test.sh'

# Run in a single SSH session (keeps conmon alive)
ssh dougbtv@<host> 'bash /tmp/before-after-test.sh'
```

### Key details for upstream CI images

- Upstream CI images (`public.ecr.aws/q9t5s3a7/vllm-ci-test-repo:*`) have NO entrypoint — use `python3 -m vllm.entrypoints.openai.api_server` explicitly
- Add `-e HF_HUB_OFFLINE=1` when the model is pre-cached
- Use `--ipc=host` instead of `--shm-size=10g` for reliability
- Model cache on a100-07: `/mnt/nvme-data/dougbtv/hub_cache` (NOT home dir — space issues)
- Detect server startup via `podman logs` grep for "Application startup complete" (not HTTP health check, which may return unexpected formats)

### Nightly "before" images

Nightly postmerge images are at `public.ecr.aws/q9t5s3a7/vllm-ci-postmerge-repo:<commit-sha>`. Find recent ones:
```bash
# Check recent nightly builds in BK
bk api '/pipelines/ci/builds?branch=main&per_page=5' | python3 -c "
import json, sys
for b in json.load(sys.stdin):
    print(f'{b[\"number\"]}: {b[\"commit\"][:12]} {b[\"state\"]}')"
```

## Part 5: OCP Smoke Test

Trigger the OCP model validation workflow with the Quay image from a build-whl-image run:

```bash
gh workflow run model-validation-ocp.yml \
  --repo neuralmagic/nm-cicd \
  --ref main \
  -f config_ref=main \
  -f label=ubuntu-latest \
  -f timeout=480 \
  -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_cuda.yml \
  -f vllm_image=quay.io/vllm/automation-vllm:cuda-<commit-or-run-id> \
  -f source_ref=<nm-vllm-ent-branch> \
  -f run_accuracy_check=true \
  -f platform=linux/amd64
```

Monitor:
```bash
gh run view <run-id> --repo neuralmagic/nm-cicd --json jobs \
  --jq '[.jobs[] | select(.status == "completed")] | group_by(.conclusion) | .[] | {conclusion: .[0].conclusion, count: length}'
```

The SNYK_IMAGE job often fails due to Quay pull issues — this is infra, not code. Focus on the model validation jobs.

## Part 6: JIRA Updates

Use temp files for comments (avoids quoting issues):
```bash
cat > /tmp/jira-update.txt << 'EOF'
## Your Update Title

Use *regular* MARKDOWN (not jira wiki format)
EOF

jira issue comment add <ISSUE-KEY> --template /tmp/jira-update.txt --no-input
```

Update at each milestone: PR opened, CI results, before/after results, review feedback addressed, OCP smoke test results.

## Part 7: Monitoring with Crons

Set up crons to track long-running builds and PR reviews:

```bash
# Build monitoring (every 5-10 min)
CronCreate: check build status, report when terminal

# PR review monitoring (every 25-30 min)
CronCreate: check for new reviews/comments, report changes

# Before/after test (one-shot after build completes)
CronCreate: pull image, run test, post results to JIRA
```

Always clean up crons when their purpose is fulfilled. Use session-only crons (not durable) for build monitoring.

### Cron lifecycle

Start frequent, scale down as urgency drops, kill when done:
- **Active work** (build running, awaiting review): every 5-30 min
- **Waiting** (PR approved, waiting for merge): every 2 hours
- **Done** (PR merged, build finished): delete immediately

Don't let crons run as no-ops — if a cron fires 2-3 times with "no new activity", kill it. A merged upstream PR has nothing left to watch.

## Lessons Learned

- Cherry-pick individual commits, never merge whole upstream branches onto release branches
- Rootless podman containers die with SSH session disconnect (exit 137) — run everything in a single SSH call
- Upstream CI images have no entrypoint — always prefix with `python3 -m vllm.entrypoints.openai.api_server`
- The `in_reply_to` field is required for threaded replies on PR review comments (not the `/replies` endpoint)
- Model cache on dev boxes should use local NVMe storage, not home directories
- BOT token errors (multiple `[TOOL_CALLS]`) are model behavior, not parser bugs
- Always run `pre-commit run --files <files>` before committing — CI will catch it anyway but saves a round trip
- BK builds: use `ignore_pipeline_branch_filters: true` and `branch=dougbtv:<branch>` format, never `branch=main`
