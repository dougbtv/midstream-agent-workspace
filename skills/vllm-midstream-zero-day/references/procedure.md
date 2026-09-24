# Midstream 0-Day Preview Release for New Models

How to build, validate, and publish a Red Hat RHAIIS/vLLM preview container image within 24 hours of a new AI model release. These previews are published to the official Red Hat Container Registry as unsupported, early-access images — clearly tagged as preview, ephemeral, and outside Red Hat's standard product lifecycle.

This skill covers the full lifecycle: upstream coordination, JIRA tracking, merge/build/test (via the `vllm-midstream-build-from-upstream` skill), AIPCC handoff, image labeling, and comms/documentation.

## Human-Agent Flow

### What the human provides

1. **Model release info** — model name, expected release date, upstream PRs or commits, hardware requirements
2. **Upstream resources** — private repo access, pre-release branches, cherry-pick lists, or an upstream release branch that already has model support
3. **Dev box access** — hostname, username, GPU availability (type and count)
4. **Weight access** — HuggingFace model ID, HF token, or S3/presigned links for pre-release weights
5. **Squad contacts** — who is the Model SME, AIPCC contact, and BU/Marketing rep for this release
6. **Constraints** — known hardware limitations, model-specific caveats, timeline pressure

### What the agent does

1. **Creates JIRA epic** — clones from template INFERENG-1857, creates stories for each checkpoint
2. **Assembles the nm-vllm-ent branch** — merges upstream resources using the `vllm-midstream-build-from-upstream` skill's merge strategies
3. **Runs pre-flight checks** — CUDA version compatibility, dependency audit, dev box availability
4. **Drives the build/deploy/test cycle** — dispatches GH Actions, monitors builds, deploys to dev box, runs smoke tests (all per the `vllm-midstream-build-from-upstream` skill)
5. **Tracks build iterations** — maintains a running build table in JIRA comments
6. **Prepares AIPCC handoff** — pushes validated image to `quay.io/vllm/rhaiis-early-access`, provides Containerfile LABEL text
7. **Publishes usage gist** — build info, deploy steps, smoke tests, gotchas, model-specific notes
8. **Documents constraints** — hardware limitations, configuration caveats, disclaimers for marketing/comms use
9. **Reports results** — image tags, test output, constraint list, gist link, AIPCC handoff status

### Expected deliverables

- A validated midstream container image (e.g., `quay.io/vllm/automation-vllm:cuda-<run-id>`)
- An early-access image pushed to `quay.io/vllm/rhaiis-early-access:<model-name>`
- JIRA epic with stories tracking each checkpoint
- A published GitHub gist with the usage guide (see Usage Gist Template below)
- Constraint documentation suitable for marketing/comms
- AIPCC handoff complete (image + LABEL metadata provided)
- GPUs released, containers stopped, crons cleaned up

### Terminology

- **upstream** = `vllm-project/vllm` (open source vLLM)
- **midstream** = `neuralmagic/nm-vllm-ent` (our fork, builds, and images)
- **AIPCC** = AI Platform Continuous Certification — the team that processes midstream images through Konflux and publishes to the official Red Hat registry
- **Konflux** = Red Hat's build system for producing official container images
- **Day 0 Squad** = cross-functional team assembled for each 0-day release (see Squad section)
- **early access** = the Red Hat Container Registry channel for preview/unsupported images

## Process Timeline

### Fast Track vs Full Process

**Fast Track** — When model support is already in an upstream release (like Gemma 4 in v0.19.1). Single JIRA story, no epic needed, no SME consultation required. Merge the release, build, test, hand off. Can be done in a day by one person.

**Full Process** — When you're merging unmerged upstream PRs, dealing with private repos, or the model needs dependency changes (like DeepSeek V4). Multi-day, multi-story epic with squad coordination. Expect build iterations, dependency surprises, and possible dev box drama.

Choose the track at notification time based on: Is the model already supported in a released or about-to-be-released upstream version?

### T-7 to T-5: Notification & Prep

**Human-driven. Agent assists with JIRA setup and research.**

- Advance warning of model release received (critical dependency)
- Day 0 Squad notified under confidentiality
- Agent creates JIRA epic (see JIRA Workflow below)
- Agent researches upstream: relevant PRs, commits, model config, known issues
- Identify hardware requirements and dev box target
- Determine if model support is in a release (fast track) or requires PR merges (full process)

### T-5 to T-1: Pre-Build & Validation

**Agent-driven. Human provides guidance on merge strategy and model constraints.**

- Assemble nm-vllm-ent branch with commits/cherry-picks (per `vllm-midstream-build-from-upstream`)
- Run pre-flight checks (see Pre-Flight Checks below)
- Trigger build via GitHub Actions (per `vllm-midstream-build-from-upstream`)
- Download model weights on dev box in background during build
- Run smoke tests on dev box
- Identify constraints and limitations (critical dependency — must scope what works and what doesn't)
- If build/test fails: iterate (see Build Iteration Tracking below)

### T-0: Model Release Day

**Agent + Human coordination.**

- Build final preview image if needed (may reuse T-1 image if nothing changed)
- Tag image for early access: `quay.io/vllm/rhaiis-early-access:<model-name>`
- Apply Containerfile LABEL: `LABEL rhaiis.preview=<model name>`
- Execute AIPCC handoff (see AIPCC Handoff below)
- Push image to official Red Hat Container Registry (AIPCC responsibility)

### T+0 to T+1: Publishing & Comms

**Human-driven. Agent provides source material.**

- Agent publishes usage gist with build info, deploy steps, smoke tests, gotchas
- Agent documents hardware/model limitations and disclaimers
- BU/Marketing rep drafts external messaging (blog, social media)
- Usage gist serves as shareable technical reference
- Agent updates the **rosetta stone** codename tracking entry (see below)

### T+N: Sunset

- No ongoing maintenance — the preview image is ephemeral
- Model support will be included in the next officially supported RHAIIS release
- Preview image remains available but is not updated

## JIRA Workflow

### Epic creation

Clone from template epic INFERENG-1857 (or create manually matching this structure).

**Epic fields:**

| Field | Value |
|-------|-------|
| Project | INFERENG |
| Type | Epic |
| Summary | `<MODEL CODENAME> 0-day Preview Release` |
| Component | INFERENG Midstream |
| Label | msboard |

**Important:** The component is `INFERENG Midstream`, NOT "RHAIIS Midstream / Model Validation" — that component doesn't exist and the CLI will error with a permissions failure trying to create it.

**Epic summary format:** `<Codename: NAME> 0-day Preview Release` (matches Ultra Magnus, Springer, Blurr pattern).

**Story naming convention:** `CODENAME: Story Title` (e.g., `Blurr: Build Wheel & Container Image`).

**Confidentiality:** Some model releases are under NDA or embargo. When the vendor requests confidentiality, use the codename ONLY in JIRA — no model name, parameter count, vendor name, or architecture details that could identify the model. Keep those details in the Slack thread and the rosetta stone entry (Obsidian / Slack canvas) instead.

**Epic description should include:**

- Model codename (always)
- Model ID only if NOT under confidentiality (e.g., `google/gemma-4-31B-it`)
- Target release date
- Upstream PR(s) or release branch (if public)
- Hardware target (GPU type and count)
- Link to internal Slack announcement thread

### Standard stories under the epic

For the full process, create these stories linked to the epic (fast track may only need story #2):

1. **Notification & Prep** — confirm timing, identify upstream resources, assign responsibilities
2. **Build Wheel & Container Image** — merge, build, deploy, smoke test (the core technical work)
3. **Model SME Consultation** — evaluate constraints, document hardware/config limitations
4. **Image Publishing** — AIPCC handoff, Konflux build, push to Red Hat registry
5. **Documentation & Comms** — usage notes, disclaimers, blog/social material

All stories should use project INFERENG, component "INFERENG Midstream", and label "msboard".

After creating each issue (epic and all stories), set the team field via REST API — the `jira` CLI `--custom` flag doesn't work for this field:

```bash
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -u "${JIRA_USERNAME}:${JIRA_API_TOKEN}" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  -d '{"fields":{"customfield_10001":"ec74d716-af36-4b3c-950f-f79213d08f71-1602"}}'
```

Link stories to the epic with `jira epic add INFERENG-XXXX INFERENG-YYYY`.

### Build iteration tracking table

Every JIRA comment during the build phase should include a running build table:

```markdown
### Build Tracking

| Build | Branch | Run ID | Result | Issue |
|-------|--------|--------|--------|-------|
| V1 | model-0day-v1 | 24886096891 | Build fail | moe_runner syntax error |
| V2 | model-0day-v2 | 24930330345 | Runtime crash | missing imports |
| V3 | model-0day-v3 | 24955194812 | SUCCESS | -- |
```

### Comment conventions

Post status updates as JIRA comments at each milestone. Each comment should be markdown-formatted and include:

1. A `### Status Update — <date>` header
2. The update content
3. The current build tracking table (copy-paste and extend from previous comment)

Key milestones to comment on:
- Build triggered (include GH Actions run link, branch, commit)
- Build success or failure (with root cause if failed)
- Smoke test results (pass/fail, which endpoints, which GPU)
- Image tagged for early access
- AIPCC handoff complete
- Any blockers (access issues, hardware contention, dependency problems)

## Pre-Flight Checks

Run these before dispatching the first build. Lessons learned from Laguna (CUDA mismatch) and DeepSeek V4 (dependency hell, dev box failure) make these non-optional.

### 1. CUDA version compatibility

The runner CUDA version must match the image's CUDA runtime. Mismatches cause `ImportError: libcudart.so.12` or similar at container startup.

```bash
# Check what CUDA version the image build will use
# Look at docker-bake.hcl or Dockerfile.ubi for the base image CUDA version
git show origin/main:docker/docker-bake.hcl 2>/dev/null | grep -i cuda
git show origin/main:Dockerfile.ubi 2>/dev/null | grep -i "FROM\|cuda"

# Match the wheel build runner to the image CUDA version:
# CUDA 12.9 image → k8s-a100-build-12-9 runner
# CUDA 13.0 image → k8s-a100-build-13-0 runner
```

If in doubt, check what accept-sync uses:
```bash
python .github/scripts/get_acceptsync_options.py cuda build_label
```

### 2. Dependency audit

New models often need newer versions of `deep_gemm`, `flashinfer`, `tilelang`, or other GPU libraries. Check the upstream PR's requirements against what's pinned in the midstream build.

```bash
# Check upstream PR dependencies
gh pr view <PR_NUMBER> --repo vllm-project/vllm --json files --jq '.files[].path' \
  | grep -i "requirements\|setup\|pyproject\|docker"

# Compare with midstream pins
cat requirements/build.txt requirements/cuda.txt 2>/dev/null | grep -i "deep_gemm\|flashinfer\|tilelang"

# If upstream has a model-specific Docker image, use it as a reference
# (DeepSeek V4 had vllm/vllm-openai:deepseekv4-cu130 — extract its deps)
docker run --rm --entrypoint pip <upstream-image> list 2>/dev/null | grep -i "deep_gemm\|flashinfer"
```

### 3. Dev box availability

```bash
# Check GPU availability on primary target
ssh <user>@<host> "chg status"

# If primary is unavailable, check fallback
ssh <user>@<fallback-host> "chg status"
```

Always have a fallback machine identified. Dev boxes die (Frankfurt H200 kernel-panicked during DeepSeek V4), GPUs get hogged (all H200s were occupied during Laguna).

### 4. UBI packaging gaps

Check for known UBI-specific issues with the model's dependencies:

```bash
# Known gap: _xxsubinterpreters (needed by tilelang's Cython JIT)
# Fix: add python3.12-test to Dockerfile.ubi
# Known gap: flashinfer cubins permissions
# Workaround: --user root or chmod in Dockerfile
```

Maintain awareness of UBI Python packaging differences vs upstream's Debian/Ubuntu base.

## Technical Build Cycle

**The core merge/build/deploy/test loop follows `vllm-midstream-build-from-upstream`.** Use it for:

- Merge strategies (upstream release branch, upstream PR, full sync)
- Build dispatch (`build-whl-image.yml`, `build-image.yml`)
- Runner labels and selection
- Build monitoring and failure investigation
- Image tag extraction
- Dev box deployment (GPU reservation, model download, container startup)
- Smoke testing (health, chat completions, completions)
- Cleanup (stop containers, release GPUs)

### 0-day-specific overrides

**Branch naming convention**: Use `<your-name>/<model-name>-0day` or `<your-name>/<model-name>-0day-v<N>` for iteration branches. Match in both nm-vllm-ent and nm-cicd.

**The "3 strikes, clean branch" rule**: If you hit 3+ merge artifacts or build failures from accumulated patches, abandon the branch and start fresh. Create a new `-v<N+1>` branch from a known-good base (e.g., `origin/main`, a validated upstream release tag, or `upstream/main`). DeepSeek V4 went through 7 build iterations — the breakthrough came from abandoning the patch-heavy V2 branch and starting clean V3 from `upstream/main` + the rebased PR.

**Using upstream's official image as a reference**: When stuck on dependency or configuration issues, pull upstream's model-specific Docker image (if one exists) and inspect it:

```bash
# Extract dependency versions from upstream's image
docker run --rm --entrypoint pip vllm/vllm-openai:<model>-cu130 list 2>/dev/null
docker run --rm --entrypoint pip vllm/vllm-openai:<model>-cu130 show deep_gemm 2>/dev/null

# Inspect Dockerfile layers
docker history vllm/vllm-openai:<model>-cu130 --no-trunc
```

**Start conservative, scale up**: Always begin smoke tests with `--enforce-eager` and `--max-model-len 4096`. Once the model loads and serves, try removing `--enforce-eager` and increasing context length. This is especially important for 0-day models where we have no prior deployment experience.

**Parallel workstreams**: If there's also an upstream DockerHub 0-day build happening (tracked separately, see `skills/vllm-upstream-zero-day/SKILL.md`), coordinate timing but keep the JIRA tracking separate. The midstream and upstream builds are independent — different images, different registries, different audiences.

## Model Weight & Code Access

### Pre-release weights

Weights may arrive via different channels depending on the model vendor:

- **HuggingFace** (most common post-release): Standard `hf download` with token
- **S3 presigned links** (pre-release): Download immediately — links expire (Laguna used 1-hour expiry). Stage to local disk on the dev box.
- **Direct transfer** from Model SME: Download to dev box local disk, set appropriate permissions

```bash
# HuggingFace download (post-release or gated models)
ssh <user>@<host> "HF_HOME=/home/<user>/.cache/huggingface \
  HF_TOKEN=<token> \
  uv tool run --from huggingface_hub hf download <model_id>"

# S3 presigned link download (pre-release)
ssh <user>@<host> "mkdir -p /mnt/data/playground/<user>/models/<model-name> && \
  cd /mnt/data/playground/<user>/models/<model-name> && \
  wget '<presigned-url>' -O model.tar && tar xf model.tar"
```

Always use **local disk** (not NFS) for model weights. Rootless podman UID remapping breaks NFS. Large NVMe mounts (e.g., `/mnt/data/playground/`) work well for big models.

### Pre-release code

vLLM code changes for new models are often in private repositories before release.

- **Preferred**: Uses a private fork readable by `nm-cicd` GitHub secret. Merge into nm-vllm-ent from the private fork.
- **Backup**: Uses a branch on nm-vllm-ent directly with cherry-picked commits from a private staging repo (e.g., `vllm-project/vllm-new-model-feb-2026`).

Restrict access to pre-release code and weights to Day 0 Squad members only.

## Image Labeling

### Container LABEL

The midstream image must carry metadata identifying the target model:

```dockerfile
LABEL rhaiis.preview=<Model Name>
```

This LABEL is inherited when AIPCC uses the midstream image as a `FROM` base in their Konflux build.

### Image tags

```
quay.io/vllm/rhaiis-early-access:<model-name>
```

The model name tag should be lowercase, hyphenated (e.g., `gemma4`, `deepseek-v4`, `laguna-xs`).

The automation-vllm build also produces standard tags that remain useful for internal tracking:
- `quay.io/vllm/automation-vllm:cuda-<run-id>` (run ID tag)
- `quay.io/vllm/automation-vllm:cuda-<commit-sha>` (commit tag)

## AIPCC Handoff Workflow

### Step 1: Determine build path

Can AIPCC build directly from the nm-vllm-ent branch?

- **Yes** — if all dependent builds (base images, wheels) are available in AIPCC's build chain. AIPCC builds from the branch using Konflux.
- **No** — Midstream provides a ready image as a `FROM` base.

### Step 2 (if AIPCC can't build from branch): Provide FROM image

```bash
# Tag the validated midstream image for early access
podman tag quay.io/vllm/automation-vllm:cuda-<run-id> \
  quay.io/vllm/rhaiis-early-access:<model-name>

# Push to quay.io
podman push quay.io/vllm/rhaiis-early-access:<model-name>
```

### Step 3: Hand off to AIPCC

Provide to the AIPCC contact (historically Jakub Rusz):

1. **Image**: `quay.io/vllm/rhaiis-early-access:<model-name>`
2. **LABEL text**: `rhaiis.preview=<Model Name>` (for their Containerfile)
3. **nm-vllm-ent branch**: `<branch-name>` (for traceability)
4. **Constraints document**: hardware limitations, tested configurations

AIPCC processes the image through Konflux and publishes to the official Red Hat Container Registry in the early-access RHAIIS channel.

## Usage Gist Template

The final deliverable is a GitHub gist that serves as both a technical reference and source material for marketing. Structure follows the established pattern (see [Gemma 4 example](https://gist.github.com/dougbtv/475c085537016a715348daf50aa241d0)):

### Required sections

1. **Title**: `<Model Name>: Build, Run, and Smoke Test Guide (nm-vllm-ent <version>)`
2. **Overview**: One-liner on model, base version, target hardware
3. **Build Information** (table):
   - Model, Container Image (all tags), nm-vllm-ent branch, nm-cicd branch, GH Actions Run ID, Target Device, Python Version
4. **Build Commands**: Exact `gh workflow run` for full build and image-only rebuild
5. **Prerequisites**: GPU requirements, storage, tooling
6. **Deployment Steps**: Reserve GPUs, download model, pull image, start server (full `podman run`), watch logs
7. **Smoke Tests**: Health check, chat completion, text completion — all copy-pasteable with actual values
8. **Cleanup**: Stop container, release GPUs
9. **Notes and Gotchas**: Model-specific quirks discovered during the build

### Creating the gist

```bash
gh gist create <filename>.md --desc "<Model Name>: Build, Run, and Smoke Test Guide" --public
```

## Comms & Documentation

### Constraint documentation

Every 0-day release must document:

- Which models/checkpoints were tested (e.g., bf16, FP8, NVFP4)
- What hardware was validated (e.g., "tested on 2x A100 80GB with TP=2")
- What hardware was NOT tested
- Known limitations (e.g., "context length limited to 4096 on A100s", "tool calling not yet validated")
- Required serve flags (e.g., `--enforce-eager`)

### Disclaimers

All external communications must include:

- This is an unsupported preview release
- Not intended for production use
- No ongoing maintenance or updates
- Preview image is ephemeral — model support will be included in the next official RHAIIS release

### Marketing source material

The usage gist doubles as source material for blog posts and social media. The BU/Marketing rep can extract:

- Model name and capabilities
- Hardware requirements
- Quick-start commands
- Performance observations from smoke tests

## Gotchas & Lessons Learned

### CUDA 12.9 vs 13.0 mismatch (recurring)

Both Laguna and DeepSeek V4 hit this. The wheel build runner must match the image's CUDA runtime. If the image uses CUDA 13.0 (`libcudart.so.13`), a wheel built on a CUDA 12.9 runner will crash at import time. Always verify before dispatching.

### Dev box kernel panics and GPU contention

The Frankfurt H200 box kernel-panicked during DeepSeek V4. All H200 GPUs were occupied during Laguna. Always have a fallback machine and check `chg status` before committing to a build plan.

### Merge artifact accumulation

DeepSeek V4 went through 7 builds partly because patches accumulated on a broken base. When you hit syntax errors, missing imports, or double-registrations from merge artifacts — stop patching and start a clean branch. The "V3 fresh approach" (clean branch from `upstream/main` + the target PR) resolved in one build what 5 patch iterations couldn't.

### UBI Python packaging gaps

UBI's Python 3.12 excludes some CPython C extensions that upstream's Debian base includes:
- `_xxsubinterpreters` — needed by tilelang's Cython JIT. Fix: add `python3.12-test` RPM to `Dockerfile.ubi`.
- Other potential gaps may surface with new models. Compare UBI's Python modules against upstream's when debugging import errors.

### deep_gemm and dependency versioning

DeepSeek V4 needed `deep_gemm` 2.4.2 but the build pinned 2.1.1 (too old, missing `tf32_hc_prenorm_gemm` kernel). When a model needs a specific library version, check the upstream PR's environment and update the midstream build accordingly.

### FlashInfer cubins permissions

FlashInfer writes compiled cubins at runtime. In rootless podman, these writes can fail due to permissions. Workarounds:
- `--user root` on `podman run`
- `chmod` in the Dockerfile
- `tmpfs` mount for the cubins directory

### NFS + rootless podman = permission denied

Always use local disk for model weights, never NFS. See `vllm-midstream-build-from-upstream` for details.

### S3 presigned links expire fast

Laguna's pre-release weights came via S3 presigned links with 1-hour expiry. Download immediately and stage to local disk. Don't assume you can re-download later.

### No model SME available

DeepSeek V4 had zero SME help. When this happens, self-serve from:
1. The upstream PR description and code changes
2. Upstream's official Docker image (if available) — inspect deps, Dockerfile, environment
3. Upstream issues and PR comments for known deployment issues
4. The model's README/docs for serve parameters

## Fast Track Checklist

For releases where model support is already in upstream (e.g., Gemma 4 in v0.19.1):

1. Create single JIRA story (no epic needed)
2. Merge upstream release into nm-vllm-ent, remove `.github/`, push
3. Create matching nm-cicd branch, push
4. Run pre-flight checks (CUDA version, dev box availability)
5. Dispatch build, download model in parallel
6. Monitor build, extract image tags on success
7. Deploy to dev box, smoke test
8. Tag and push to `quay.io/vllm/rhaiis-early-access:<model-name>`
9. Publish usage gist
10. Hand off to AIPCC
11. Cleanup: stop container, release GPUs, delete crons

## Full Process Checklist

For releases requiring PR merges, private repos, or dependency changes:

1. Create JIRA epic + stories (clone from INFERENG-1857 template)
2. Research upstream: PRs, commits, dependencies, known issues
3. Run pre-flight checks (CUDA version, dependency audit, dev box availability, UBI gaps)
4. Assemble nm-vllm-ent branch (merge strategy per `vllm-midstream-build-from-upstream`)
5. Dispatch build, download model weights in parallel
6. Monitor build — on failure, investigate and iterate
7. Track build iterations in JIRA comment table
8. If 3+ failures from merge artifacts: clean branch, start fresh (`-v<N+1>`)
9. On success: deploy to dev box, smoke test
10. If smoke test fails: adjust flags, fix code, rebuild (the fix-and-rebuild cycle)
11. Document constraints and limitations
12. Tag and push to `quay.io/vllm/rhaiis-early-access:<model-name>`
13. Apply Containerfile LABEL: `rhaiis.preview=<model name>`
14. Publish usage gist
15. Hand off to AIPCC (image + LABEL + branch + constraints)
16. Provide constraint docs and marketing source material
17. Cleanup: stop container, release GPUs, delete crons

## Day 0 Squad

| Role | Responsibility |
|------|---------------|
| Midstream lead & builder | Assembles branch, drives builds, smoke tests, AIPCC handoff |
| Runtime / Model SME | Evaluates model constraints, provides usage instructions, hardware guidance |
| AIPCC representative | Konflux build, push to official Red Hat registry |
| BU/Marketing representative | External comms — blog posts, social media, disclaimers |

The squad is notified under confidentiality when a new model release is imminent. Members keep release details confidential until the official preview image is published.

## Rosetta Stone / Codename Tracking

Every 0-day release gets a **rosetta stone entry** — a short block that maps the internal codename to the real model, timeline, Slack thread, and JIRA epic. These are maintained in Doug's Obsidian notes and a Slack canvas shared with the team.

**Format:**

```markdown
## CODENAME

For `Model-Name-Here` (short description of what makes it notable)

> One paragraph with the key context: what the model is, when it's releasing,
> what's needed from vLLM, who the contacts are, any special considerations.

Expected: DATE
[Announcement thread](SLACK_URL) (DATE)
Tracked in: [INFERENG-XXXX](https://redhat.atlassian.net/browse/INFERENG-XXXX)
```

Create this entry at Notification & Prep time. Update the "Tracked in" link once the JIRA epic is created. The rosetta stone is the ONE place where the real model name and codename live together — JIRA may be codename-only for confidential releases.

## Reference JIRA Issues

- **INFERENG-1857** — Template epic for 0-day preview releases
- **INFERENG-6204** — Gemma 4 (fast track, single story, clean run)
- **INFERENG-4094** — Poolside Laguna (full process, CUDA mismatch, GPU contention)
- **INFERENG-6294** — DeepSeek V4 (full process, 7 build iterations, dependency hell, no SME help)
- **INFERENG-7471** — Springer / Nemotron 3 Ultra (full process, patch release cherry-pick)
- **INFERENG-7670** — Ultra Magnus / DiffusionGemma (first dLLM, upstream release images)
- **INFERENG-8137** — Blurr (full process, confidential, MoE with MLA)

---
name: midstream-0-day-release
description: Build, validate, and publish Red Hat RHAIIS/vLLM 0-day preview container images for new model releases. Use when a new AI model drops and we need to produce an early-access midstream image — merging upstream support into nm-vllm-ent, building via GH Actions, smoke testing, tagging for early access, and handing off to AIPCC for publication on the official Red Hat Container Registry. Covers the full lifecycle from upstream notification through JIRA tracking, build/test iteration, AIPCC handoff, and comms/documentation. Triggers on: 0-day release, preview release, model launch day, early access image, AIPCC handoff, rhaiis-early-access, Red Hat registry, new model build, day zero.
---
