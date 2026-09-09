# Midstream vLLM-Omni Build from Upstream PR

Runbook for building a midstream vllm-omni image from an upstream PR branch. Use when you need to test an unmerged upstream vllm-omni fix in a midstream container image.

## Prerequisites

- `gh` CLI authenticated with access to `neuralmagic/nm-cicd` and `neuralmagic/nm-vllm-omni-ent`
- A local clone of `vllm-project/vllm-omni` with remotes:
  - `upstream` -> `vllm-project/vllm-omni`
  - `midstream` -> `neuralmagic/nm-vllm-omni-ent`
- A local clone of `neuralmagic/nm-cicd`
- An existing vLLM base wheel run ID (from a prior midstream nm-vllm-ent build)

## Overview

The two-wheel build strategy means three things need to line up:

1. **nm-vllm-omni-ent branch** with the upstream PR merged into midstream
2. **nm-cicd branch** with flashinfer/dependency pins matching the target vLLM version
3. **vLLM base wheel** already built from nm-vllm-ent at the matching version

## Step 1: Find Your vLLM Base Wheel

Check JIRA for the latest midstream vLLM build story (e.g. INFERENG-9743 for v0.27.0). The comments contain run IDs for successful wheel builds.

```bash
jira issue view INFERENG-XXXX --comments 5
```

Note the `run_id` from a successful `build-whl-image.yml` or `build-whl.yml` run. This becomes `vllm_run_id` later.

## Step 2: Create the Experimental nm-vllm-omni-ent Branch

From your vllm-omni clone:

```bash
cd vllm-omni

# Make sure remotes are fresh
git fetch midstream
git fetch upstream

# Create branch from midstream main
git checkout -b doug/<descriptive-name> midstream/main

# Merge the upstream PR branch with -X theirs
# This brings upstream to current level + the PR's changes
# -X theirs auto-resolves conflicts in favor of the incoming code
git merge -X theirs <pr-branch-name> --no-edit
```

### Verify after merge

```bash
# Midstream Dockerfiles should be untouched (they're midstream-only files)
git diff midstream/main..HEAD -- Dockerfile.ubi docker-bake.hcl

# Midstream .github/workflows should be intact
ls .github/workflows/

# Check the merge looks right
git log --oneline -5
```

### Push to nm-vllm-omni-ent

```bash
git push midstream doug/<descriptive-name>
```

## Step 3: Create the nm-cicd Branch with Updated Pins

The nm-cicd branch needs flashinfer pins that match the target vLLM version. Base your branch on `feature/vllm-omni-tp` (the active omni infrastructure branch with `accept-sync` support).

```bash
cd nm-cicd

git fetch origin feature/vllm-omni-tp
git checkout -b doug/<matching-name> origin/feature/vllm-omni-tp
```

### Check what the target vLLM version expects

```bash
# Look at the standard vLLM cuda.txt on the matching sync branch
gh api repos/neuralmagic/nm-cicd/contents/neuralmagic/requirements/cuda.txt?ref=sync-v0.27.0 \
  | python3 -c "import sys,json,base64; d=json.load(sys.stdin); print(base64.b64decode(d['content']).decode())" \
  | grep flashinfer
```

### Update flashinfer pins

Edit these files to match:

| File | Purpose |
|------|---------|
| `neuralmagic/requirements/vllm-omni-cuda.txt` | Default CUDA variant (jit-cache) |
| `neuralmagic/requirements/vllm-omni-cuda-develsdk.txt` | DevSDK variant (JIT at runtime) |

**Package rename gotcha:** `flashinfer-cubin` was renamed to `flashinfer-python` after version 0.6.13. If targeting vLLM v0.27.0+, use `flashinfer-python`, not `flashinfer-cubin`. The `flashinfer-jit-cache` name is unchanged.

Example for v0.27.0:
```
# vllm-omni-cuda.txt
--extra-index-url https://flashinfer.ai/whl/cu130
flashinfer-python==0.6.16.post3
flashinfer-jit-cache==0.6.16.post3

# vllm-omni-cuda-develsdk.txt
flashinfer-python==0.6.16.post3
```

### Update constraints if needed

Check `neuralmagic/constraints/vllm-omni.txt` against upstream's `pyproject.toml` requirements:

```bash
# From the vllm-omni clone, check what upstream expects
grep -i "transformers\|fastapi" pyproject.toml
```

If upstream has relaxed a constraint (e.g. transformers), update the constraints file to match.

### Commit and push

```bash
git add neuralmagic/requirements/vllm-omni-cuda.txt \
        neuralmagic/requirements/vllm-omni-cuda-develsdk.txt \
        neuralmagic/constraints/vllm-omni.txt
git commit -m "Bump flashinfer and dependency pins for vllm-omni v0.XX.0"
git push origin doug/<matching-name>
```

## Step 4: Dispatch the Build

### Option A: Full accept-sync (wheel + image + tests)

```bash
gh workflow run accept-sync.yml --repo neuralmagic/nm-cicd \
  --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=<nm-vllm-omni-ent-branch> \
  -f python=3.12 \
  -f target_device=cuda \
  -f wf_category=DEBUG \
  -f vllm_run_id=<VLLM_RUN_ID> \
  -f build_image=true
```

### Option B: Image-only rebuild (reuse existing omni wheel)

If the wheel already built successfully but the image failed (e.g. dependency issue):

```bash
gh workflow run build-image.yml --repo neuralmagic/nm-cicd \
  --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=<nm-vllm-omni-ent-branch> \
  -f target_device=cuda \
  -f build_label=k8s-a100-util \
  -f run_id=<OMNI_WHEEL_RUN_ID> \
  -f vllm_run_id=<VLLM_RUN_ID>
```

### Option C: Manual three-step (fine-grained control)

```bash
# Step 1: Skip — reuse existing vLLM wheel

# Step 2: Build omni wheel
gh workflow run build-whl.yml --repo neuralmagic/nm-cicd \
  --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=<nm-vllm-omni-ent-branch> \
  -f target_device=cuda \
  -f python=3.12 \
  -f build_label=k8s-a100-build-13-0 \
  -f timeout=120 \
  -f vllm_run_id=<VLLM_RUN_ID> \
  -f partitions_file=neuralmagic/configs/partitions/minimal.yml

# Step 3: Build image (after step 2 completes)
gh workflow run build-image.yml --repo neuralmagic/nm-cicd \
  --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-omni-ent \
  -f branch=<nm-vllm-omni-ent-branch> \
  -f target_device=cuda \
  -f build_label=k8s-a100-util \
  -f run_id=<OMNI_WHEEL_RUN_ID> \
  -f vllm_run_id=<VLLM_RUN_ID>
```

## Step 5: Monitor

```bash
# Check status
gh run view <RUN_ID> --repo neuralmagic/nm-cicd --json status,conclusion,jobs \
  --jq '{status: .status, conclusion: .conclusion, jobs: [.jobs[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

Typical timings:
- Omni wheel build: ~10-15 min
- Image bake: ~25-30 min
- Full accept-sync with tests: ~45-75 min

## Step 6: Get the Image Tags

```bash
# Find the image build job ID
gh api repos/neuralmagic/nm-cicd/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[] | select(.name | contains("image")) | .id'

# Extract tags from logs
gh api repos/neuralmagic/nm-cicd/actions/jobs/<JOB_ID>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' \
  | grep -i "quay.io/vllm/automation-vllm-omni" \
  | grep "pushing manifest\|image.name"
```

Image tags follow the pattern:
- `quay.io/vllm/automation-vllm-omni:cuda-<run-id>` (easiest to use)
- `quay.io/vllm/automation-vllm-omni:<version-tag>` (version from setuptools_scm)
- `quay.io/vllm/automation-vllm-omni:cuda-<commit-sha>` (commit tag)

## Troubleshooting

### Image build fails: `No solution found for flashinfer-cubin`

`flashinfer-cubin` was renamed to `flashinfer-python` after v0.6.13. Update `vllm-omni-cuda.txt` and `vllm-omni-cuda-develsdk.txt` to use `flashinfer-python`. Then retrigger image-only with Option B above.

### Image build fails: dependency version conflict

Check that `neuralmagic/constraints/vllm-omni.txt` doesn't over-constrain a package that upstream has relaxed. Common culprits: `transformers`, `fastapi`.

### Wheel build fails: vLLM import error

The omni wheel build needs the vLLM wheel installed first (vllm-omni's `setup.py` imports from vLLM at build time). Verify `vllm_run_id` points to a successful vLLM wheel build and the GCS asset exists.

### Merge conflicts on upstream merge

If the PR branch is far behind upstream main and you get many conflicts, try the "merge PR branch directly" strategy from the midstream-build-from-upstream skill: `-X theirs` resolves conflicts in favor of the incoming branch. For an experimental build, this is almost always fine.

### Tests fail but image built

Test failures don't block the image. The image tag is still available. Check the test failures separately — they may be pre-existing or unrelated to your changes.

## Example: INFERENG-10008 PII Fix Build (Aug 2026)

```
nm-vllm-omni-ent branch: doug/pii-fix-experimental
nm-cicd branch:          doug/omni-pii-fix-v2 (from feature/vllm-omni-tp)
vLLM base wheel:         31490750632 (v0.27.1 from INFERENG-9743)
Omni wheel run:          32172755731
Image build run:         32176131705
Image tag:               quay.io/vllm/automation-vllm-omni:cuda-32176131705
```

Key lesson: `flashinfer-cubin` -> `flashinfer-python` rename caught us on the image build. Wheel build was fine. Fixed the pin and did an image-only rebuild (Option B), reusing both existing wheels.

## Running and Debugging the Image

### Quick sanity check

The image entrypoint is the vllm server, not a shell. Use `--entrypoint` to get a bash session:

```bash
podman run --rm -it --entrypoint bash \
  --device nvidia.com/gpu=0 --security-opt=label=disable \
  -e CUDA_VISIBLE_DEVICES=0 \
  quay.io/vllm/automation-vllm-omni:cuda-<run-id>
```

Inside the container, the omni package lives at `/opt/vllm/lib64/python3.12/site-packages/vllm_omni/`.

### Running tests against the image

Tests aren't shipped in the wheel, so you need a local clone of the repo. Clone (or pull) the branch you're testing, then bind-mount the tests dir into the container:

```bash
# On your dev box
git clone https://github.com/<fork>/vllm-omni.git ~/vllm-omni-test
cd ~/vllm-omni-test && git checkout <your-branch>

# Run tests
podman run --rm --entrypoint bash \
  --device nvidia.com/gpu=0 --security-opt=label=disable \
  -e CUDA_VISIBLE_DEVICES=0 \
  -v ~/vllm-omni-test/tests:/tests:Z \
  quay.io/vllm/automation-vllm-omni:cuda-<run-id> \
  -c "pip install pytest pytest-mock pytest-asyncio -q && \
      python -m pytest /tests/entrypoints/openai_api/test_serving_speech.py -x -q --tb=short"
```

`pytest`, `pytest-mock`, and `pytest-asyncio` aren't in the production image — install them at runtime as shown above.

### Hot-patching source files into the container

If you have local fixes that aren't baked into the image yet (e.g. you pushed a commit after the image was built), mount the fixed source files directly over the installed package. This lets you test code changes without rebuilding the entire image:

```bash
podman run --rm --entrypoint bash \
  --device nvidia.com/gpu=0 --security-opt=label=disable \
  -e CUDA_VISIBLE_DEVICES=0 \
  -v ~/vllm-omni-test/tests:/tests:Z \
  -v ~/vllm-omni-test/vllm_omni/entrypoints/openai/serving_speech.py:/opt/vllm/lib64/python3.12/site-packages/vllm_omni/entrypoints/openai/serving_speech.py:Z \
  -v ~/vllm-omni-test/vllm_omni/entrypoints/openai/serving_chat.py:/opt/vllm/lib64/python3.12/site-packages/vllm_omni/entrypoints/openai/serving_chat.py:Z \
  quay.io/vllm/automation-vllm-omni:cuda-<run-id> \
  -c "pip install pytest pytest-mock pytest-asyncio -q && \
      python -m pytest /tests/entrypoints/openai_api/test_serving_speech.py -x -q --tb=short"
```

The pattern is `-v <local-file>:<installed-package-path>:Z` for each file you want to override. This is great for iterating on fixes without waiting 30+ minutes for a full image rebuild.

### GPU reservation on shared dev boxes

If you're on a shared box with `canhazgpu`, reserve a GPU before running:

```bash
canhazgpu   # shows available GPUs and reservations
# Then use --device nvidia.com/gpu=<N> and -e CUDA_VISIBLE_DEVICES=<N>
```

### Common gotchas

- **Version mismatch warnings**: If the image's vLLM and vLLM-Omni versions diverge (e.g. omni 0.26.x vs vllm 0.27.x), you'll see a `RuntimeWarning` about mismatched versions. Tests still run, but watch for import errors.
- **SELinux denials on mounts**: Use `:Z` on all `-v` mounts when running on RHEL/Fedora hosts, or `--security-opt=label=disable` on the container.
- **MagicMock attribute access**: Test fixtures using `mocker.MagicMock()` for `request_logger` will return MagicMock objects for any attribute (not None). Use `isinstance()` checks instead of truthiness checks when accessing mock attributes like `max_log_len`.
