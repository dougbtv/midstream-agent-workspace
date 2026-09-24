---
name: vllm-midstream-build-from-upstream
description: "Build and validate a vLLM midstream image from an upstream release or PR: merge, preflight dependencies and runners, dispatch CI, smoke test, and record evidence."
---

# vLLM Midstream Build from Upstream

Use this procedure when an upstream vLLM release or PR must become a
`neuralmagic/nm-vllm-ent` image. For a complete preview-release lifecycle,
also use `vllm-midstream-zero-day`.

## 1. Establish the build contract

Record the upstream source (release/tag/PR SHA), target device and model,
desired artifact, and whether the work is a supported sync or unsupported
preview. Inspect the current `nm-vllm-ent` and `nm-cicd` branches and remotes;
never reset or clean an attached repository to make it look tidy. Confirm both
named branches exist remotely before dispatching.

**Done when:** source, two branch names, runner pair, validation target, and
artifact are explicit.

## 2. Merge upstream deliberately

For a release, create `sync-v<version>` from the accepted midstream base and
merge the release branch:

```bash
git fetch upstream releases/v<version>
git checkout -b sync-v<version> origin/main
git merge -X theirs upstream/releases/v<version> --no-edit
```

For an unmerged PR, inspect its state and successor PRs first. Merge the PR
branch rather than blindly cherry-picking a stale stack. If a full upstream sync
is also necessary, merge upstream first and resolve the PR on top. Remove
upstream `.github/` content from `nm-vllm-ent`, resolve the remaining conflicts
deliberately, and push a matching `nm-cicd` branch.

**Done when:** both remote branches point at reviewable commits with no
unresolved paths.

## 3. Run preflight gates

### CUDA and compiler runner

Inspect the upstream Dockerfile and accept-sync configuration. Match the wheel
runner to the image CUDA version: CUDA 12.9 uses `k8s-a100-build-12-9`; CUDA
13 uses `k8s-a100-build-13-0`. Use a `*-util` runner for image builds.

CUDA matching is not compiler matching. Inspect changed native dependencies and
the actual runner toolchain. If a dependency requires a newer libstdc++ feature
than vLLM's declared compiler floor, preserve that evidence and use a separately
tagged runner-image candidate/canary; do not silently replace a shared
`latest-*` runner tag.

### Dependencies

Inspect changed requirements, Dockerfiles, and native extensions. After every
merge, check the FlashInfer chain:

```bash
grep -i 'flashinfer\|extra-index' requirements/cuda.txt
grep FLASHINFER_VERSION docker/Dockerfile
```

Keep the downstream `flashinfer-jit-cache` carry **and** its CUDA-specific
index (CUDA 13: `https://flashinfer.ai/whl/cu130`). Validate the exact wheel
exists before a full build; `flashinfer-cubin` and `flashinfer-jit-cache` are
different packages.

### Validation capacity

Identify the dev/OCP target and a fallback. Check GPU capacity, weight access,
local storage, and conservative serve flags.

**Done when:** CUDA, compiler, dependency-resolution, and validation capacity
are proven rather than assumed.

## 4. Dispatch the right build

Use `build-whl-image.yml` for a fresh wheel and image:

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd --ref <nm-cicd-branch> \
  -f repo=neuralmagic/nm-vllm-ent -f branch=<nm-vllm-ent-branch> \
  -f build_label=<cuda-matched-wheel-runner> -f build_timeout=120 \
  -f image_label=<util-image-runner> -f python=3.12 \
  -f release_image=false -f target_device=cuda
```

Use `build-image.yml` only after a successful wheel build, passing its run ID.
Record the workflow URL, branch, commit, runners, and result in the owning Jira
story.

**Done when:** the workflow run exists and its identifying evidence is recorded.

## 5. Monitor and classify failure

Monitor until terminal state. On failure, retrieve the failed job logs and
classify the first real failure: dependency resolution, compiler/ABI,
compilation, image assembly, or runtime. Rebuild only after the causal change.
If a shared runner image must change, publish an immutable candidate, canary the
target workflow, then promote only after validation.

**Done when:** either a usable image tag exists or the ticket names a specific
terminal blocker and next owner.

## 6. Validate and hand off

Stage weights on local disk while CI runs. Begin with `--enforce-eager` and
`--max-model-len 4096`; verify `/health` (HTTP 200), one completion or chat
request, and model-specific behavior (for example structured decision reads).
Record the image digest/tag, topology, checkpoint, flags, and exact results.
Release reservations and stop test containers afterward.

For ordinary syncs, attach evidence to the sync story. For previews, follow
`vllm-midstream-zero-day` for the preview label, AIPCC handoff, constraints,
and communications material.

**Final verification:** the ticket, branch commits, CI conclusion, artifact tag,
and smoke evidence all refer to the same build.
