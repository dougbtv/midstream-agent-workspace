---
name: upstream-0-day-release
description: Build, validate, and tag upstream vLLM Docker images on DockerHub for model launch day ("0-day") releases. Use this skill when a new model PR lands on vllm-project/vllm and we need to produce official vllm/vllm-openai Docker images — triggering Buildkite release-v2 builds, unblocking Docker image steps, copying images from ECR to DockerHub, creating multi-arch manifests, and tagging with the model name. Covers the full lifecycle from PR identification through smoke testing to official DockerHub publication.
---

# Upstream 0-Day Release Workflow

End-to-end skill for producing official `vllm/vllm-openai` Docker images on launch day for new model support PRs. This is separate from the midstream build process (nm-cicd / nm-vllm-ent) — this targets the upstream vLLM repo and publishes to the public DockerHub.

## Prerequisites

- `bk` CLI (Buildkite) authenticated for the `vllm` org, with **build-trigger access on the `release-v2` pipeline** (request from Kevin Luu / SIG CI if needed)
- `gh` CLI authenticated with access to `vllm-project/vllm` (and any private pre-release repos)
- `jira` CLI configured for Red Hat Atlassian
- `docker` CLI with DockerHub login (creds in GCP secrets — see below)
- SSH access to a dev GPU box with NVIDIA GPUs (e.g., H100, H200)
- `skopeo` (optional, faster image copies)

### Buildkite access methods

The `bk` CLI is the primary tool for interacting with release-v2. A Buildkite MCP server is also available but its auth token can expire mid-session — if MCP returns a re-authorization error, fall back to `bk` CLI which uses a separate, longer-lived token. The `bk` CLI is reliable for both read operations (listing builds, checking status) and write operations (triggering builds, unblocking steps).

## Part 1: Identify the Upstream PR

Find or confirm the upstream PR on `vllm-project/vllm` for the new model:

```bash
# Check PR status
gh pr view <PR_NUMBER> --repo vllm-project/vllm \
  --json title,state,headRefName,headRepositoryOwner,mergeable,commits

# Get the branch and commit SHA — you'll need these for the build
gh pr view <PR_NUMBER> --repo vllm-project/vllm \
  --json headRefName,headRepositoryOwner,commits \
  --jq '{branch: (.headRepositoryOwner.login + ":" + .headRefName), commit: .commits[-1].oid}'
```

Note the **branch** (in `owner:branch-name` format for fork PRs) and **commit SHA**. These are inputs to the Buildkite build.

### Private pre-release repos do NOT work with release-v2

The release-v2 pipeline is hardwired to clone `https://github.com/vllm-project/vllm.git`. Commits from private pre-release repos (e.g., `vllm-project/vllm-new-model-03-23-2026`) are not reachable and will fail at checkout with `upload-pack: not our ref`. Tested and confirmed with DiffusionGemma build #2454 (June 2026).

You **must wait** for the code to be pushed to the public `vllm-project/vllm` repo (either merged to main or as an open PR branch) before triggering release-v2. Per typical 0-day timelines, this happens ~1h30 before the official release.

## Part 2: Trigger the Release Build

### Option A: release-v2 pipeline (preferred — produces vllm-openai shaped images)

This pipeline builds wheels + Docker images for multiple architectures and CUDA versions. It produces the correct `vllm-openai` image shape for DockerHub.

```bash
# For PRs on the public repo (fork PRs use owner:branch format)
bk api --method POST /pipelines/release-v2/builds --data '{
  "commit": "<COMMIT_SHA>",
  "branch": "<OWNER>:<BRANCH_NAME>",
  "ignore_pipeline_branch_filters": true,
  "message": "<Model Name> - upstream Docker image build from PR #<NUMBER>"
}'

# For branches on private vllm-project repos (just the branch name)
bk api --method POST /pipelines/release-v2/builds --data '{
  "commit": "<COMMIT_SHA>",
  "branch": "<BRANCH_NAME>",
  "ignore_pipeline_branch_filters": true,
  "message": "<Model Name> - upstream Docker image build from PR #<NUMBER>"
}'
```

If you get a 403, you don't have build-trigger access on `release-v2`. Ask Kevin Luu (SIG CI lead) to grant it for your Buildkite account.

### Option B: ci pipeline (fallback — for validation only)

If you can't access `release-v2`, the `ci` pipeline will produce a test image you can validate with, but it won't have the right shape for the official DockerHub tag.

```bash
bk api --method POST /pipelines/ci/builds --data '{
  "commit": "<COMMIT_SHA>",
  "branch": "<OWNER>:<BRANCH_NAME>",
  "ignore_pipeline_branch_filters": true,
  "message": "<Model Name> - upstream CI build from PR #<NUMBER>"
}'
```

The ci pipeline produces images at `public.ecr.aws/q9t5s3a7/vllm-ci-test-repo:<COMMIT_SHA>`. Good for smoke testing, not for official tagging.

> **Never use `--branch=main`** — it clutters the main build view, annoys maintainers, and the build record can't be deleted.

## Part 3: Unblock Docker Image Build (release-v2 only)

The release-v2 pipeline has manual gate steps. After bootstrap completes, the pipeline expands to show all jobs.

### List jobs and find the unblock step

```bash
bk api /pipelines/release-v2/builds/<BUILD_NUMBER> | python3 -c "
import json, sys
b = json.load(sys.stdin)
print(f'Build #{b[\"number\"]} — {b[\"state\"]}')
for j in b.get('jobs', []):
    name = j.get('name','') or j.get('label','') or j.get('type','')
    state = j.get('state','')
    jtype = j.get('type','')
    jid = j.get('id','')
    if jtype == 'manual':
        print(f'  >>> UNBLOCK >>> [{state:12s}] {name}  ({jid})')
    else:
        print(f'  [{state:12s}] [{jtype:15s}] {name}  ({jid})')
"
```

### Unblock the Docker image build

Find the job ID for **"Unblock to build release Docker images"** and unblock it:

```bash
bk api --method PUT /pipelines/release-v2/builds/<BUILD_NUMBER>/jobs/<JOB_ID>/unblock --data '{}'
```

Do NOT unblock:
- "Provide Release version here" — leave this alone (no release version needed for 0-day builds)
- "Confirm update release wheels to PyPI" — we're not publishing wheels
- CPU image unblocks — skip unless specifically needed

### Monitor the build

```bash
# Quick status check
bk api /pipelines/release-v2/builds/<BUILD_NUMBER> | python3 -c "
import json, sys
b = json.load(sys.stdin)
print(f'Build #{b[\"number\"]} — {b[\"state\"]}')
jobs = b.get('jobs', [])
running = [j for j in jobs if j.get('state') == 'running' and j.get('type') == 'script']
passed = [j for j in jobs if j.get('state') == 'passed' and j.get('type') == 'script']
failed = [j for j in jobs if j.get('state') == 'failed' and j.get('type') == 'script']
print(f'Running: {len(running)}, Passed: {len(passed)}, Failed: {len(failed)}')
for j in running:
    print(f'  [running] {j.get(\"name\",\"\")}')
for j in failed:
    print(f'  [FAILED]  {j.get(\"name\",\"\")}')
"
```

The build produces images at `public.ecr.aws/q9t5s3a7/vllm-release-repo:<COMMIT_SHA>-<SUFFIX>`.

### Expected image suffixes from release-v2

| Suffix | Description |
|--------|-------------|
| `-x86_64` | x86_64, CUDA 13.0 (default) |
| `-aarch64` | arm64, CUDA 13.0 (default) |
| `-x86_64-cu129` | x86_64, CUDA 12.9 |
| `-aarch64-cu129` | arm64, CUDA 12.9 |
| `-x86_64-ubuntu2404` | x86_64, CUDA 13.0, Ubuntu 24.04 |
| `-aarch64-ubuntu2404` | arm64, CUDA 13.0, Ubuntu 24.04 |
| `-x86_64-cu129-ubuntu2404` | x86_64, CUDA 12.9, Ubuntu 24.04 |
| `-aarch64-cu129-ubuntu2404` | arm64, CUDA 12.9, Ubuntu 24.04 |

## Part 4: Validate the Image

Before tagging on DockerHub, smoke test the image on GPU hardware.

### Pull and run

```bash
# On a dev GPU box (e.g., H100)
COMMIT=<COMMIT_SHA>
docker pull public.ecr.aws/q9t5s3a7/vllm-release-repo:${COMMIT}-x86_64

# Run with model weights (adjust paths, GPU, and model args)
docker run --rm \
  --gpus '"device=0"' \
  --ipc=host \
  -p 8000:8000 \
  -v /path/to/model/weights:/model \
  public.ecr.aws/q9t5s3a7/vllm-release-repo:${COMMIT}-x86_64 \
    --model /model \
    --tensor-parallel-size 1 \
    --enforce-eager \
    --max-model-len 4096 \
    --trust-remote-code
```

If using podman with CDI:
```bash
podman run --rm \
  --device nvidia.com/gpu=0 \
  --security-opt=label=disable \
  --ipc=host \
  -p 8000:8000 \
  -v /path/to/model/weights:/model \
  public.ecr.aws/q9t5s3a7/vllm-release-repo:${COMMIT}-x86_64 \
    --model /model \
    --tensor-parallel-size 1 \
    --enforce-eager \
    --max-model-len 4096 \
    --trust-remote-code
```

### Smoke test

```bash
# Health check (use 127.0.0.1, not localhost — avoids IPv6 issues)
curl http://127.0.0.1:8000/health

# Chat completions
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/model",
    "messages": [{"role": "user", "content": "Hello, what model are you?"}],
    "max_tokens": 128
  }'

# Completions
curl -s http://127.0.0.1:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/model",
    "prompt": "Tell me a fact about the capitol of Vermont",
    "max_tokens": 32
  }'
```

## Part 5: Tag and Push to DockerHub

### Get DockerHub credentials

Credentials are stored in Google Cloud Secrets:

```bash
gcloud secrets versions access latest \
  --secret=upstream-vllm-docker-instructions \
  --project=264456887452
```

Log in:
```bash
docker login
```

### Tag convention

Follow the pattern established by deepseekv4 on [DockerHub](https://hub.docker.com/r/vllm/vllm-openai/tags):

| Tag | Description |
|-----|-------------|
| `vllm/vllm-openai:<model>` | Top-level multi-arch, defaults to latest CUDA |
| `vllm/vllm-openai:<model>-cu130` | Multi-arch manifest, CUDA 13.0 |
| `vllm/vllm-openai:<model>-cu129` | Multi-arch manifest, CUDA 12.9 |
| `vllm/vllm-openai:<model>-x86_64-cu130` | Per-arch, CUDA 13.0 |
| `vllm/vllm-openai:<model>-arm64-cu130` | Per-arch, CUDA 13.0 |
| `vllm/vllm-openai:<model>-x86_64-cu129` | Per-arch, CUDA 12.9 |
| `vllm/vllm-openai:<model>-arm64-cu129` | Per-arch, CUDA 12.9 |

Replace `<model>` with the model's short name (e.g., `laguna`, `deepseekv4`).

### Copy per-arch images

```bash
COMMIT=<COMMIT_SHA>
SRC=public.ecr.aws/q9t5s3a7/vllm-release-repo
DST=vllm/vllm-openai
MODEL=<model-short-name>

# CUDA 13.0
docker pull ${SRC}:${COMMIT}-x86_64
docker tag  ${SRC}:${COMMIT}-x86_64  ${DST}:${MODEL}-x86_64-cu130
docker push ${DST}:${MODEL}-x86_64-cu130

docker pull ${SRC}:${COMMIT}-aarch64
docker tag  ${SRC}:${COMMIT}-aarch64  ${DST}:${MODEL}-arm64-cu130
docker push ${DST}:${MODEL}-arm64-cu130

# CUDA 12.9
docker pull ${SRC}:${COMMIT}-x86_64-cu129
docker tag  ${SRC}:${COMMIT}-x86_64-cu129  ${DST}:${MODEL}-x86_64-cu129
docker push ${DST}:${MODEL}-x86_64-cu129

docker pull ${SRC}:${COMMIT}-aarch64-cu129
docker tag  ${SRC}:${COMMIT}-aarch64-cu129  ${DST}:${MODEL}-arm64-cu129
docker push ${DST}:${MODEL}-arm64-cu129
```

> **Faster alternative with skopeo:**
> ```bash
> skopeo copy docker://${SRC}:${COMMIT}-x86_64 docker://${DST}:${MODEL}-x86_64-cu130
> ```
> Repeat for each image. Copies layers directly without extracting locally.

### Create multi-arch manifests

```bash
# CUDA 13.0
docker manifest create ${DST}:${MODEL}-cu130 \
  ${DST}:${MODEL}-x86_64-cu130 \
  ${DST}:${MODEL}-arm64-cu130
docker manifest push ${DST}:${MODEL}-cu130

# CUDA 12.9
docker manifest create ${DST}:${MODEL}-cu129 \
  ${DST}:${MODEL}-x86_64-cu129 \
  ${DST}:${MODEL}-arm64-cu129
docker manifest push ${DST}:${MODEL}-cu129

# Top-level alias (defaults to latest CUDA version)
docker manifest create ${DST}:${MODEL} \
  ${DST}:${MODEL}-x86_64-cu130 \
  ${DST}:${MODEL}-arm64-cu130
docker manifest push ${DST}:${MODEL}
```

### Verify

```bash
docker manifest inspect vllm/vllm-openai:${MODEL}
docker manifest inspect vllm/vllm-openai:${MODEL}-cu130
docker manifest inspect vllm/vllm-openai:${MODEL}-cu129
```

Check on DockerHub: `https://hub.docker.com/r/vllm/vllm-openai/tags?name=<model>`

## Part 6: JIRA Updates

Post updates to the tracking JIRA ticket at each milestone. Use temp files for comments:

```bash
cat > /tmp/jira-update.md << 'EOF'
### Status Update — <date>

Your update here in markdown.
EOF

jira issue comment add <ISSUE-KEY> --template /tmp/jira-update.md --no-input
```

Key milestones to update on:
1. Build triggered (include BK build link, branch, commit)
2. Docker step unblocked
3. Smoke test results (pass/fail, which endpoints tested, which GPU)
4. Images tagged on DockerHub (list all tags created)
5. Any blockers (access issues, build failures, hardware contention)

## Why not nm-cicd?

It's tempting to use midstream CI (nm-cicd) since we know it well, but the build architectures are fundamentally different:

- **nm-cicd** does a two-phase build (wheel on GHA runner → inject into UBI image via `payload/run.sh`). Its `docker-bake.hcl` expects targets named `cuda`, `rocm`, etc.
- **release-v2** does a single multi-stage Docker build from upstream's `docker/Dockerfile` (Debian-based). Its `docker-bake.hcl` uses targets named `openai`, `test`, etc.

nm-cicd also requires repo-specific action directories (`env-build-image-<repo>/`, `prepare-payload-<repo>/`, etc.) that only exist for `nm-vllm-ent`. Adding upstream vLLM support would require creating 4 new action dirs, remapping bake targets, and skipping the payload injection — roughly 2-4 hours of engineering. Not appropriate for a 0-day, but could be a future project to unify the build paths.

For upstream DockerHub images, always use the Buildkite release-v2 pipeline.

## Troubleshooting

### 403 on release-v2 pipeline
You need build-trigger access. Ask Kevin Luu (SIG CI lead, upstream) to grant it for your Buildkite account email. As a fallback, use the `ci` pipeline for validation builds.

### Unblock API returns "Body should be a JSON Hash"
Pass an empty JSON body: `--data '{}'`

### curl localhost returns "Connection reset by peer"
Use `127.0.0.1` instead of `localhost` — curl may resolve to IPv6 (::1) which the server doesn't handle.

### podman run -d exits with 137
Podman lock file corruption. Use `screen -dmS <name> bash -c 'podman run --rm ...'` to run in a persistent foreground session instead of `podman run -d`.

### CUDA version mismatch at runtime
Make sure the image CUDA version matches the host driver. If the image is CUDA 13 based, the host needs CUDA 13 compatible drivers. Check with `nvidia-smi`.

### Buildkite MCP token expired
The Buildkite MCP server token can expire mid-session. If you get a re-authorization error, fall back to the `bk` CLI which uses a separate, longer-lived token. All MCP operations have `bk` CLI equivalents.

### Private repo commit not found (`upload-pack: not our ref`)
release-v2 only has access to `vllm-project/vllm`. Commits from private pre-release repos (even within the same org) will fail with this error. Wait for the code to land on the public repo before triggering the build.

## People / Contacts

- **Kevin Luu** — upstream SIG CI lead, controls Buildkite release-v2 access and DockerHub credentials
- **Lucas Wilkinson** — upstream integration lead, coordinates model-specific releases
- **Robert Shaw** — has done upstream 0-day image builds before (Poolside), reference for process questions

## Reference

- Buildkite release-v2: `https://buildkite.com/vllm/release-v2`
- Buildkite ci: `https://buildkite.com/vllm/ci`
- DockerHub tags: `https://hub.docker.com/r/vllm/vllm-openai/tags`
- ECR release repo: `public.ecr.aws/q9t5s3a7/vllm-release-repo`
- GCP secrets project: `264456887452`
- DeepSeek v4 tags (reference pattern): `https://hub.docker.com/r/vllm/vllm-openai/tags?name=deep`

## Past Builds

| Model | JIRA | Build | Private Repo | Notes |
|-------|------|-------|-------------|-------|
| DiffusionGemma | INFERENG-7741 | [#2454](https://buildkite.com/vllm/release-v2/builds/2454) | `vllm-project/vllm-new-model-03-23-2026` PR #10 | FAILED — private repo commit not reachable from release-v2. Must wait for public PR. |
