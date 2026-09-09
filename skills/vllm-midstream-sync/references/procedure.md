# Upstream Merge Skill for nm-vllm-ent

How to merge upstream vllm-project/vllm work (PRs, branches, main) into the nm-vllm-ent fork ("midstream"), build images, deploy them, and verify they serve models.

## Human-Agent Flow

This skill is designed as a collaborative workflow. The human provides inputs and the agent drives the build/deploy/test cycle autonomously, reporting back with results. Here's how it goes:

### What the human provides

1. **Branch names** for nm-vllm-ent and nm-cicd (e.g. `<your-name>/v0.19.1`)
2. **Upstream resources** to bring into midstream — a release branch (e.g. `releases/v0.19.1`), a PR, or upstream/main
3. **A model to test** with any relevant docs or recipes
4. **Dev box access** — hostname, username, and GPU availability

### What the agent does

0. **Researches** upstream vLLM as needed — uses `gh` CLI to inspect PRs, issues, and code in `vllm-project/vllm`, and reads the upstream remote in the nm-vllm-ent clone to understand model configs, supported flags, Dockerfile changes, or known issues. This research informs merge decisions, serve parameters, and troubleshooting. The agent can also make experimental fixes directly on the nm-vllm-ent branch when upstream code needs adjustment for midstream.
1. **Merges** the upstream resources into nm-vllm-ent, removes `.github/`, pushes
2. **Creates** matching nm-cicd branch, pushes
3. **Dispatches** the build workflow (wheel + image)
4. **Downloads the model** on the dev box in the background while the build runs
5. **Monitors** the build with self-deleting crons (every 5 min), reports when done
6. **Reserves GPUs**, pulls the image, starts the container
7. **Monitors startup** with self-deleting crons (every 2 min)
8. **Runs smoke tests** — health check, chat completions, completions endpoint
9. **If startup or tests fail**: investigates logs, adjusts flags (e.g. `--enforce-eager`, lower `--max-model-len`), and retries. This fix-and-rebuild cycle repeats until the model is serving or a blocking issue is identified
10. **Reports results** to the human: image name/tags, test output, any interesting findings
11. **Creates a GitHub gist** with a usage doc covering build info, deploy steps, smoke test commands, and gotchas

### Expected deliverables

- A working container image tag (e.g. `quay.io/vllm/automation-vllm:cuda-<run-id>`)
- Passing smoke tests (health 200, coherent chat response)
- A published GitHub gist with the full usage guide — see [this example](https://gist.github.com/dougbtv/475c085537016a715348daf50aa241d0) for the expected format and level of detail
- GPUs released, containers stopped, crons cleaned up

### Terminology

- **upstream** = `vllm-project/vllm` (the open source vLLM repo)
- **midstream** = `neuralmagic/nm-vllm-ent` (our fork, and the builds/images we produce from it)

## Repo Layout

- **origin**: `git@github.com:neuralmagic/nm-vllm-ent.git` (our fork)
- **upstream**: `git@github.com:vllm-project/vllm.git` (upstream vllm)
- nm-vllm-ent has **no `.github/` directory** -- upstream does. Always exclude it.
- The two repos share a common ancestor but diverge significantly (500+ commits each direction is typical).

## Strategy: Merging an Upstream PR Branch

When cherry-picking an open (unmerged) upstream PR into nm-vllm-ent:

### What works

1. **Find the PR's base commit** (the merge-base between the PR branch and upstream/main):
   ```bash
   git fetch https://github.com/<author>/<repo>.git <branch>
   PR_BASE=$(git merge-base FETCH_HEAD upstream/main)
   ```

2. **Create a clean branch from that base and merge the PR onto it** (should fast-forward or merge cleanly since the PR was developed against that base):
   ```bash
   git checkout -b pr-NNNNN-clean $PR_BASE
   git merge FETCH_HEAD --no-edit
   ```

3. **Switch to your target branch and merge with `-X theirs`**:
   ```bash
   git checkout <your-name>/my-feature-branch   # based on origin/main
   git merge -X theirs pr-NNNNN-clean --no-edit
   ```

4. **Clean up**:
   ```bash
   git branch -D pr-NNNNN-clean
   ```

### What does NOT work well

- **Cherry-picking PR commits onto latest upstream/main**: If the PR is hundreds of commits behind upstream/main, you get conflicts in every file both sides touched. Manual resolution is error-prone because you're trying to apply the PR's intent onto code that has changed underneath it.

- **Applying the PR as a patch (`git diff base..PR | git apply --3way`)**: Same conflict set, no real advantage over merge.

- **Merging the PR branch directly after first merging upstream/main**: You still get the same conflicts because the common ancestor between the PR branch and your now-upstream-synced branch is old.

The winning move is to **skip the upstream/main sync entirely** when you only need the PR's work. Merge the PR branch directly into your origin/main-based branch. The common ancestor between origin/main and the PR branch is close enough that git resolves everything cleanly with `-X theirs`.

## Strategy: Merging an Upstream Release Branch

When merging a specific upstream release (e.g. `releases/v0.19.1`) into nm-vllm-ent:

```bash
git fetch upstream releases/v0.19.1
git checkout -b <your-name>/v0.19.1 origin/main
git merge -X theirs upstream/releases/v0.19.1 --no-edit
```

Conflicts will almost always be limited to `.github/` files (modify/delete). Resolve and commit per the `.github/` handling section below, then push both the nm-vllm-ent branch and a matching nm-cicd branch before triggering the build.

### Branch naming convention: `sync-v<version>`

When syncing a new upstream release, the team convention is to use **`sync-v<version>`** as the branch name (e.g. `sync-v0.25.0`) in both nm-vllm-ent and nm-cicd. This makes it easy for others to continue the work across repos and releases.

- Use `sync-v<version>` for the official branch that the team builds on
- It's fine to develop on a personal branch first (e.g. `doug/v0.25.0`) and push the `sync-` branch once validated
- If nm-cicd needs changes for the new release (dependency bumps, build fixes), create the `sync-v<version>` branch there too with those changes
- Prior examples: `sync-v0.24.0`, `sync-v0.23.0-spyre`

## Strategy: Full Upstream Sync (origin/main <- upstream/main)

When syncing nm-vllm-ent with upstream/main:

```bash
git checkout -b sync-upstream origin/main
git fetch upstream main
git merge -X theirs upstream/main --no-edit
```

### Handling `.github/`

nm-vllm-ent does not use upstream's `.github/`. After any upstream merge:

```bash
# Remove anything that came in from upstream
git rm -rf .github/ 2>/dev/null
git commit -m "Remove upstream .github directory"
```

Or if the merge isn't committed yet, just `git rm -rf .github/` before committing.

### Handling modify/delete conflicts

These show up as `DU` (deleted by us, updated by them) or `UD` (updated by us, deleted by them) in `git status`:

- **DU (we deleted, they updated)**: If it's `.github/` stuff, `git rm` it. Otherwise decide case-by-case.
- **UD (we updated, they deleted)**: Usually prefer upstream's deletion: `git rm <file>`.

## Strategy: Upstream PR + Upstream Sync Combined

If you need BOTH an unmerged upstream PR AND a full upstream sync:

1. Do the upstream sync first (`-X theirs upstream/main`)
2. Then merge the PR branch on top
3. The PR merge will have conflicts where upstream changed the same code the PR touches -- these require manual resolution

Alternatively, if you only need the PR's functionality (not a full sync), just merge the PR branch alone. Much simpler.

## Useful Commands

### Divergence and merge prep

```bash
# Check divergence between origin and upstream
git merge-base origin/main upstream/main
git rev-list --count <merge-base>..upstream/main   # upstream ahead
git rev-list --count <merge-base>..origin/main     # origin ahead

# See what files a PR touches
gh pr view <number> --repo vllm-project/vllm --json files --jq '.files[].path'

# Check if a PR has been superseded
gh pr view <number> --repo vllm-project/vllm --comments | grep -i "stale\|supersed\|favor"

# Get PR metadata
gh pr view <number> --repo vllm-project/vllm --json title,state,mergeCommit,headRefName,headRepositoryOwner

# Check which PR files also changed on upstream since the PR was branched
PR_BASE=$(git merge-base FETCH_HEAD upstream/main)
git diff --name-only $PR_BASE..upstream/main -- <file1> <file2> ...
```

### Researching upstream for model support, fixes, and serve parameters

The nm-vllm-ent clone has an `upstream` remote pointing at `vllm-project/vllm`. Use it to read source code, check model configs, and understand serve flags without needing a separate checkout. Combine this with `gh` CLI queries to find relevant PRs and issues.

```bash
# Search upstream issues/PRs for a model or feature
gh search issues --repo vllm-project/vllm "gemma-4" --limit 10
gh search prs --repo vllm-project/vllm "gemma4" --state all --limit 10

# Read model config from upstream to understand serve parameters
git show upstream/main:vllm/model_executor/models/<model_file>.py | head -100

# Check what models are registered
git show upstream/main:vllm/model_executor/models/registry.py | grep -i <model_name>

# Look at a specific upstream release branch
git log --oneline upstream/releases/v0.19.1 -- vllm/model_executor/models/ | head -20

# Read Dockerfiles to understand image defaults
git show upstream/main:docker/Dockerfile | grep -i "ENV\|ARG\|HF_"

# Find upstream fixes for a specific error message
gh search issues --repo vllm-project/vllm "num_gpu_blocks" --limit 5
gh search prs --repo vllm-project/vllm "enforce-eager" --state merged --limit 5

# Check if upstream has a recipe or example for a model
gh api repos/vllm-project/vllm/contents/examples --jq '.[].name' | grep -i <model>
```

This research is useful at multiple stages: before merging (to pick the right branch/PRs), during deployment (to figure out the right flags), and during troubleshooting (to find known fixes for errors). When a fix is needed that doesn't exist upstream, make the change directly on the nm-vllm-ent branch, commit, push, and rebuild.

## Lessons Learned

- Always check if an upstream PR has been superseded before merging it. Use `gh pr view --comments` to look for author notes about newer PRs.
- PR #37081 ("Add Mistral Guidance") was superseded by #38150 (merged) + #39217 (open). The original PR still worked fine for our purposes but the successor PRs are the canonical path upstream.
- The `-X theirs` strategy auto-resolves conflicts in favor of "theirs" (the branch being merged in). This is safe when you trust the incoming code and just want it applied.
- Merging upstream release branches (e.g. `releases/v0.19.1`) works cleanly with `-X theirs` — conflicts are typically limited to `.github/` modify/delete issues.
- Always start with `--enforce-eager` and a small `--max-model-len` (4096) for initial smoke tests. Once the model loads and serves successfully, you can try removing `--enforce-eager` and increasing context length.
- Download the model in the background while the build runs — the build takes 45-75 min and the download takes 1-5 min, so there's no reason to wait.
- The health endpoint returns an empty body with HTTP 200. Use `curl -s -o /dev/null -w '%{http_code}'` to verify it.

## Post-Merge: FlashInfer Dependency Alignment

**This is the single most common build failure after an upstream merge.** Every upstream sync that bumps vLLM's flashinfer dependency will break the CUDA image build unless `cuda.txt` is updated to match. This has caused recurring nightly failures across v0.23.0, v0.25.0, and v0.27.0 cycles.

### The dependency chain

1. The **vLLM wheel** (built from nm-vllm-ent) declares `flashinfer-python` and `flashinfer-cubin` as pip dependencies — version comes from upstream's `pyproject.toml`
2. **`neuralmagic/requirements/cuda.txt`** in **nm-cicd** separately pins `flashinfer-python` and `flashinfer-cubin` for the container image build
3. After `git merge -X theirs upstream/...`, the wheel's metadata updates to the new flashinfer version, but **`cuda.txt` in nm-cicd keeps the old pins** because it's our file, not upstream's
4. At image build time, `uv` tries to install the wheel (which requires flashinfer==X.Y.Z) alongside cuda.txt (which pins flashinfer==A.B.C) — the conflict is unresolvable and the build fails

### Package rename: `flashinfer-jit-cache` → `flashinfer-cubin`

Upstream renamed `flashinfer-jit-cache` to `flashinfer-cubin`. If `cuda.txt` still references `flashinfer-jit-cache`, update the package name to `flashinfer-cubin`.

### After every upstream merge, before dispatching a build

```bash
# 1. In nm-vllm-ent: check what flashinfer version the merged vLLM code expects
grep -i flashinfer pyproject.toml setup.py requirements*.txt 2>/dev/null
grep FLASHINFER docker/Dockerfile 2>/dev/null

# 2. In nm-cicd: check what cuda.txt currently pins
grep -i flashinfer neuralmagic/requirements/cuda.txt

# 3. If they don't match, update cuda.txt in nm-cicd to align with the wheel's requirement
#    Edit neuralmagic/requirements/cuda.txt — bump flashinfer-python and flashinfer-cubin
#    to match what the wheel expects
#    Also rename flashinfer-jit-cache to flashinfer-cubin if still using the old name

# 4. In nm-cicd: commit the pin bump
git add neuralmagic/requirements/cuda.txt
git commit -m "Bump flashinfer pins in cuda.txt to match upstream"
```

### Diagnosing flashinfer build failures

If the CUDA image build fails with a `uv` resolution error mentioning `flashinfer`, this is almost certainly a pin mismatch in `cuda.txt`. Check the build logs for lines like:
- `error: cannot install flashinfer-python==X.Y.Z and flashinfer-python==A.B.C`
- `conflict: flashinfer-cubin` version requirements

Fix: update `cuda.txt` pins in nm-cicd, commit, push, and rebuild (image-only via `build-image.yml` if the wheel already succeeded).

### Runtime version mismatch (serve time)

Even with aligned build-time pins, a version mismatch between `flashinfer` and `flashinfer-cubin` (or the old `flashinfer-jit-cache`) at runtime will crash the vLLM process. The `FLASHINFER_DISABLE_VERSION_CHECK=1` env var (already in the deploy command template) bypasses this check but means baked JIT cubins won't be used — flashinfer falls back to slower runtime JIT compilation on first request.

---
name: midstream-build-from-upstream
description: Build, deploy, and smoke test upstream vLLM model images. Use when the user wants to build a wheel and/or container image from nm-vllm-ent, deploy it to a dev box, and verify it serves a model. Handles the full lifecycle from GH Actions dispatch through image pull, model download, container startup, and inference testing. Also use when the user wants to merge an upstream vLLM release branch (e.g. releases/v0.19.1) into nm-vllm-ent. Make sure to use this skill whenever the user mentions building vLLM images, deploying models to dev boxes, smoke testing inference, or merging upstream releases.
---

# Upstream Build, Deploy, and Smoke Test

End-to-end skill for building nm-vllm-ent wheels/images via GitHub Actions and validating them on a dev box with GPU hardware.

## Prerequisites

- `gh` CLI authenticated with access to `neuralmagic/nm-cicd`
- SSH access to a dev box with GPUs (e.g., h100-02 at 10.14.217.24)
- A branch pushed to both `neuralmagic/nm-cicd` and `neuralmagic/nm-vllm-ent`

> **Infrastructure reference:** For cluster access methods, hardware specs, and SSH targets, see [`nm-cicd/docs/hardware-summary.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/hardware-summary.md). For finding runner configs in stratus, see [`nm-cicd/docs/how-to.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/how-to.md).

## Step 1: Kick Off the Build

### Full build (wheel + image)

Use `build-whl-image.yml` when you need both a fresh wheel and a container image.

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref <NM_CICD_BRANCH> \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<NM_VLLM_ENT_BRANCH> \
  -f build_label=<BUILD_LABEL> \
  -f build_timeout=120 \
  -f image_label=<IMAGE_LABEL> \
  -f python=3.12 \
  -f release_image=false \
  -f target_device=cuda
```

### Image only (reuse existing wheel)

Use `build-image.yml` when you already have a successful wheel build and only need a new image (e.g., after a Dockerfile fix).

```bash
gh workflow run build-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref <NM_CICD_BRANCH> \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<NM_VLLM_ENT_BRANCH> \
  -f build_label=<IMAGE_LABEL> \
  -f release_image=false \
  -f run_id=<PREVIOUS_RUN_ID> \
  -f target_device=cuda
```

The `run_id` is the `databaseId` from a previous successful `build-whl-image` or `build-whl` run. This avoids rebuilding the wheel from scratch.

**Prefer `*-util` runners for `build_label`:** Instead of the legacy `-dind` runners, use a `*-util` runner label (e.g. `k8s-a100-util`, `os-mi300x-util`, `k8s-gaudi-util`). These use the Kubernetes Buildx driver and support multi-arch builds. Match the runner to the target accelerator cluster. See the runner labels table below for the full list.

### Runner Labels

> **Canonical source:** [`nm-cicd/docs/runner-labels.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/runner-labels.md) has the full list of all runner labels with GPU/CPU/memory specs across all clusters. The table below covers the labels most relevant to build workflows.

| Purpose | Label | CUDA | Status |
|---------|-------|------|--------|
| Wheel build (cuda 12.9) | `k8s-a100-build-12-9` | 12.9 | Reliable, default for accept-sync |
| Wheel build (cuda 12.9, alt) | `k8s-a100-solo-build-12-9` | 12.9 | Works, separate pool |
| Wheel build (cuda 13.0) | `k8s-a100-build-13-0` | 13.0 | Use for upstream v0.22.0+ (defaults to CUDA 13) |
| Image build (cuda, legacy) | `ibm-wdc-k8s-h100-dind` | — | Still works, but prefer `-util` runners |
| Image build (cuda, legacy alt) | `k8s-a100-dind` | — | Works for cpu/rocm, has failed on cuda bake |
| Image build (util) | `k8s-a100-util` | — | Kubernetes Buildx driver, multi-arch capable |
| Image build (util, H100) | `ibm-wdc-k8s-h100-util` | — | Kubernetes Buildx driver, multi-arch capable |
| Image build (util, MI300X) | `os-mi300x-util` | — | Kubernetes Buildx driver, multi-arch capable |
| Image build (util, Gaudi) | `k8s-gaudi-util` | — | Kubernetes Buildx driver, multi-arch capable |
| Wheel build (cpu) | `rdu4-k8s-cpu` | — | Reliable |
| Wheel build (rocm) | `os-mi300x-build-ubi-rocm71` | — | Reliable |
| Wheel build (gaudi) | `k8s-gaudi-build` | — | Reliable |

**`*-util` runners for image builds (preferred):** As of [nm-cicd PR #537](https://github.com/neuralmagic/nm-cicd/pull/537), the `build-image.yml` workflow supports any `*-util` runner via the Kubernetes Buildx driver. This removes the dependency on `-dind` runners and enables multi-arch builds. Match the util runner to the target cluster — e.g. `--image-label os-mi300x-util` for MI300X, `--image-label k8s-gaudi-util` for Gaudi. OpenShift clusters get automatic handling for privileged service accounts and seLinux context.

**IMPORTANT: Match the wheel build runner CUDA version to upstream's default.** Starting with v0.22.0, upstream vLLM defaults to CUDA 13.0 (`ARG CUDA_VERSION=13.0.2` in `docker/Dockerfile`). Building a CUDA 13 wheel on a CUDA 12.9 runner produces a wheel tagged `cu129` that crashes at runtime with `ImportError: libcudart.so.12`. Check upstream's Dockerfile for the default CUDA version before dispatching.

To check what accept-sync uses by default:
```bash
python .github/scripts/get_acceptsync_options.py cuda build_label
```

### Important: nm-cicd branch must exist

The `--ref` flag refers to the nm-cicd branch. If it doesn't exist on the remote, the dispatch will fail with `HTTP 422: No ref found`. Create it off main and push:
```bash
cd <nm-cicd-repo>
git fetch origin main
git checkout -b <branch-name> origin/main
git push -u origin <branch-name>
```

### Parallelizing prep work

While the build runs (typically 45-75 min for wheel + image), start the model download on the target dev box in the background. This saves significant time since large models (30+ GiB) take several minutes to download:
```bash
ssh <user>@<host> "HF_HOME=/home/<user>/.cache/huggingface \
  HF_TOKEN=<token> \
  nohup uv tool run --from huggingface_hub hf download <model_id> \
  > /tmp/hf-download.log 2>&1 &"
```

## Step 2: Monitor the Build

### Get the run ID

```bash
sleep 5 && gh run list --repo neuralmagic/nm-cicd --workflow build-whl-image.yml --limit 3 \
  --json databaseId,headBranch,status,conclusion,displayTitle
```

### Watch progress

```bash
gh run view <RUN_ID> --repo neuralmagic/nm-cicd --json status,conclusion,jobs \
  --jq '{status: .status, conclusion: .conclusion, jobs: [.jobs[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

Set up a CronCreate job checking every 5 minutes to track progress automatically. Delete the cron when the build reaches a terminal state.

### On failure: investigate

```bash
# Find the failed job ID
gh api repos/neuralmagic/nm-cicd/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[] | select(.conclusion == "failure") | {id: .id, name: .name}'

# Get the logs
gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' | grep -i "error\|##\[error" | head -20
```

If the wheel build succeeded but the image build failed, reuse the wheel via `build-image.yml` with `run_id` set to the current run.

### On success: get the image tags

```bash
# Get the image build job ID
gh api repos/neuralmagic/nm-cicd/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[] | select(.name | contains("image")) | .id'

# Extract image tags from logs
gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' | grep "naming to quay"
```

Image tags follow the pattern:
- `quay.io/vllm/automation-vllm:<version>` (version tag)
- `quay.io/vllm/automation-vllm:cuda-<commit-sha>` (commit tag)
- `quay.io/vllm/automation-vllm:cuda-<run-id>` (run ID tag)

## Step 3: Deploy to Dev Box

### Reserve GPUs

```bash
ssh <user>@<host> "chg reserve -G <gpu_ids> -d 4h"
```

Note: the flag is lowercase `-d` (not `-D`). Check available GPUs first:
```bash
ssh <user>@<host> "chg status"
```

### Download the model

```bash
ssh <user>@<host> "HF_HOME=/home/<user>/.cache/huggingface \
  HF_TOKEN=<token> \
  uv tool run --from huggingface_hub hf download <model_id>"
```

### Pull the image

The image pull happens automatically on `podman run`, but you can pre-pull to save time:
```bash
ssh <user>@<host> "podman pull <image>"
```

### Start the server

```bash
ssh <user>@<host> "podman run -d \
  --name vllm-<model-short-name> \
  --device nvidia.com/gpu=<gpu0> \
  --device nvidia.com/gpu=<gpu1> \
  --security-opt=label=disable \
  --shm-size=10g \
  -p 8000:8000 \
  -v /home/<user>/.cache/huggingface:/hf:Z \
  -e HF_HUB_OFFLINE=1 \
  -e FLASHINFER_DISABLE_VERSION_CHECK=1 \
  -e HF_HOME=/hf \
  -e CUDA_VISIBLE_DEVICES=<gpu_list> \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  <image> \
    --model <model_id> \
    --tensor-parallel-size <tp> \
    --max-model-len <ctx_len> \
    --gpu-memory-utilization 0.9 \
    --enforce-eager \
    --trust-remote-code \
    --host 0.0.0.0 --port 8000"
```

Set up a CronCreate job checking every 2-3 minutes that:
1. Checks if the container is still running (`podman ps`)
2. Tails logs looking for `Application startup complete.`
3. Fires the smoke test when ready
4. Reports crash logs if the container exits
5. Self-deletes when done

### Watch logs

```bash
ssh <user>@<host> "podman logs -f vllm-<model-short-name>"
```

## Step 4: Smoke Test

### Health check

```bash
ssh <user>@<host> "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/health"
# Expected: 200 (body is empty)
```

### Completion request

```bash
ssh <user>@<host> 'curl -s http://127.0.0.1:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '"'"'{"model": "<model_id>", "prompt": "Hello, world", "max_tokens": 128}'"'"''
```

### Chat request

```bash
ssh <user>@<host> 'curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '"'"'{"model": "<model_id>", "messages": [{"role": "user", "content": "Hello"}], "max_tokens": 128}'"'"''
```

## Gotchas and Lessons Learned

### FlashInfer version mismatch = image build failure

The most common build failure after an upstream merge. If `neuralmagic/requirements/cuda.txt` pins a different flashinfer version than what the vLLM wheel expects, `uv` will fail to resolve dependencies during the CUDA image build. See the "Post-Merge: FlashInfer Dependency Alignment" section above for the full dependency chain and fix procedure. Also watch for the `flashinfer-jit-cache` → `flashinfer-cubin` package rename — stale package names in `cuda.txt` will also break the build.

### NFS + rootless podman = permission denied

Rootless podman remaps UIDs. The container's vllm user (UID 2000) gets mapped to an unmapped UID that NFS rejects. **Always use local disk** for the HF cache mount (`/home/<user>/.cache/huggingface`), not NFS paths.

Symptom: `Permission denied` when the container tries to read the model, even with `--security-opt=label=disable`.

### HF_HUB_OFFLINE=1 is baked into the image

The automation-vllm images set `HF_HUB_OFFLINE=1` in the Dockerfile. The model must be fully downloaded and cached before the container starts. You cannot rely on the container downloading it at runtime.

### Use 127.0.0.1, not localhost

On some hosts, `curl http://localhost:8000` tries IPv6 first (::1) and fails with exit code 56 even though the server is up. Always use `http://127.0.0.1:8000`.

### OOM with large context lengths

Model cards often recommend large `--max-model-len` values (e.g., 262144) that won't fit on 2 GPUs. Start with a smaller value (4096) for smoke testing and scale up once you've confirmed the model loads. Even 8192 can be too much on A100s with larger models.

### CUDA graph profiling OOM — use --enforce-eager

On A100 80GB GPUs, models that consume ~30+ GiB (like gemma-4-31B-it with TP=2) leave almost no headroom after loading. The CUDA graph profiling phase (`determine_available_memory`) will report `num_gpu_blocks=0` and crash with `RuntimeError: cancelled` during KV cache initialization. The symptom is the container exiting shortly after `Profiling CUDA graph memory` appears in logs.

Fix: add `--enforce-eager` to skip CUDA graph capture entirely. This trades some inference throughput for reliable startup. H100s with more memory may not need this flag. Reducing `--gpu-memory-utilization` alone does not fix it — the profiling step itself is what OOMs.

### Container disappears on crash with --rm

If you use `--rm` and the container crashes, there are no logs to inspect. For debugging, omit `--rm` so you can run `podman logs <name>` after a crash. Clean up manually with `podman rm <name>`.

### SELinux :Z relabel on large mounts

The `:Z` flag on volume mounts triggers SELinux relabeling. On large model directories this can be slow. If you hit issues, try without `:Z` and use `--security-opt=label=disable` instead.

### Retrigger commands in job summaries

Every workflow writes a retrigger command to the GitHub job summary. To extract it:
```bash
gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' \
  | grep -A 20 "Retrigger this job" \
  | grep -E "gh workflow|-f |--repo|--ref" \
  | sed 's/^[^ ]*Z //'
```

### Check .workspace.yaml

If a `.workspace.yaml` exists in the repo root, use the branches and defaults specified there instead of requiring manual input. See the CLAUDE.md for the full spec. Note: the workflow input for model-validation-configs branch is `model_validation_configs_ref` (not the old `config_ref`).

## Cleanup

Always release GPUs when done — reservations are shared resources.

```bash
ssh <user>@<host> "podman stop vllm-<name> && podman rm vllm-<name>"
ssh <user>@<host> "chg release -G <gpu_ids>"
```

## End-to-End Checklist

When running the full lifecycle (merge + build + deploy + test), this is the sequencing:

1. **Merge** upstream release/PR into nm-vllm-ent, remove `.github/`, push
2. **Check FlashInfer alignment** in `neuralmagic/requirements/cuda.txt` — bump pins to match what the wheel expects (see "Post-Merge: FlashInfer Dependency Alignment" section)
3. **Create matching nm-cicd branch** off main, push
4. **Dispatch build** via `build-whl-image.yml`
5. **Start model download** on dev box in background (parallelizes with build)
6. **Monitor build** with CronCreate every 5 min, delete cron on terminal state
7. **On build success**: extract image tags from job logs (`grep "naming to quay"`)
8. **Reserve GPUs**, pull image, start container
9. **Monitor container startup** with CronCreate every 2 min
10. **Smoke test**: health (200), chat completions (coherent response), completions endpoint
11. **Write usage doc** in the style of existing gists (build info table, deploy steps, smoke tests, gotchas)
12. **Cleanup**: stop container, release GPUs, delete crons

Typical wall-clock time: ~75-90 min (45-75 min build, 5-10 min image pull, 2-3 min startup).
