---
name: rocm-validation
description: >
  ROCm validation responsibility for RHAIIS releases. Use when assigned to the ROCm
  rotation on the team's release spreadsheet. Covers the full lifecycle: midstream
  build & test on MI300X/MI355X hardware, iterating on failures (upstream cherry-picks,
  dependency fixes, arch target gaps), getting changes tagged in rhai/* branches of
  nm-vllm-ent, and final validation against downstream AIPCC-built RC images.
  Includes parameterized GH Actions commands, OCP cluster deployment, failure triage,
  and acceptance criteria tracking.
---

# ROCm Validation Responsibility

## What This Is

You've been assigned the **ROCm responsibility** for a RHAIIS release. This means you own the validation of vLLM on AMD ROCm GPUs (MI300X, MI355X, future hardware) for this release cycle. Your name is on the team rotation spreadsheet next to "ROCm" for this version.

Your job is to make sure that when the release ships, vLLM works correctly on ROCm hardware — models serve, tests pass, performance is acceptable, and no regressions slipped in.

## The Critical Path

Think of it as a loop with a clear exit condition:

```
1. Build & test midstream (accept-sync on MI300X/MI355X)
2. Triage failures → fix or cherry-pick → re-test
3. Repeat step 2 until green
4. Get changes into rhai/* branch, cut a tag
5. Wait for AIPCC to build downstream RC image
6. Validate downstream RC with same test suite
7. Report results → done (or back to step 2 if RC has issues)
```

**You're done when:** accept-sync is green on all target hardware, OCP model validation passes, the midstream tag exists, and the downstream RC image passes smoke tests.

## Mental Model

Think of your job as three layers:

| Layer | Question | How you validate |
|-------|----------|-----------------|
| **Environment sanity** | Does ROCm + this GPU hardware even work? | Debug workflow, hardware smoke tests |
| **Platform validation** | Does the RHAIIS image serve models correctly? | accept-sync, lm-eval, guidellm, OCP model validation |
| **Regression detection** | Did we break anything? Are known bugs fixed? | Test partitions, comparison with previous release |

---

## Phase 1: Orientation

Before you touch anything, get the lay of the land.

### What to find out

- **What release?** Which RHAIIS version (e.g., 3.4.0, 3.5EA1)? What's the upstream vLLM version (e.g., 0.18.0)?
- **What's the release branch?** The midstream release branch in `nm-vllm-ent` follows the pattern `rhai/X.Y.Z` (e.g., `rhai/0.18.0`).
- **What changed?** Is this a ROCm version upgrade (e.g., 6.4 → 7.1)? New hardware (e.g., MI355X)? Or just standard validation on existing stack?
- **Who had it before?** Check the rotation spreadsheet. Look at their branches, JIRA trail, and any open PRs. The previous person's `sync-*` or validation branch in nm-cicd is your starting point.
- **What's the epic?** There should be an epic in INFERENG for the release (e.g., "Midstream Build & Validation for RHAIIS X.Y.Z"). Your ROCm work lives under it.

### Where to look

- **Rotation spreadsheet** — ask your lead if you don't have the link
- **JIRA** — search `project = INFERENG AND summary ~ "ROCm" AND fixVersion = <release>` for prior work
- **nm-vllm-ent branches** — `git branch -r | grep rhai` to find the release branch
- **nm-cicd branches** — look for `sync-*` or `*-rocm*` branches from the previous owner
- **Slack** — `#forum-rhaiis-release` for release coordination, `#forum-aipcc-wheels` for downstream pipeline

### Key repos

| Repo | What it is | Your role |
|------|-----------|-----------|
| `neuralmagic/nm-vllm-ent` | Midstream vLLM fork. Release branch is `rhai/X.Y.Z`. | Cherry-pick upstream fixes, remove/update deps, cut tags |
| `neuralmagic/nm-cicd` | CI/CD workflows, test configs, validation scripts. | Fix build scripts, update arch targets, add runner configs, fix test deps |
| `neuralmagic/stratus` | Runner infrastructure, provisioning scripts. | Only if you need to set up new hardware runners |
| `neuralmagic/nm-actions` | Reusable GH Actions (install deps, detect GPU, etc.). | Only if runner actions break on new OS/hardware |
| AIPCC pipeline (`gitlab.com/redhat/rhel-ai/rhai/pipeline`) | Downstream build pipeline. Builds the RC images. | You don't modify this — you validate what it produces |

---

## Phase 2: Midstream Build & Test

This is where you spend most of your time. The goal is to get a fully green accept-sync run on all target hardware.

### 2.1 Catalog build artifacts

Before running anything, verify you know what's being built:

- **Custom wheels** — ROCm builds require pre-compiled wheels for `flash_attn`, `amd_aiter`, and `fastsafetensors`. These live in GCS (`gs://nm-public-pypi/dist/`). Check which versions are pinned in `neuralmagic/requirements/rocm.txt` on the release branch.
- **PyTorch version** — ROCm uses a specific torch build from `https://download.pytorch.org/whl/rocmX.Y`. Check `rocm.txt` for the exact pin.
- **GPU architecture targets** — This is a common source of bugs. Check that `PYTORCH_ROCM_ARCH` includes all target architectures in these files:
  - `nm-cicd/.github/scripts/setup_env_build_vllm.sh`
  - `nm-cicd/.github/actions/build-aiter/build_aiter.sh`
  - `nm-cicd/.github/actions/build-flash_attn/build_rocm_flashattention.sh`
  - `nm-vllm-ent/docker/Dockerfile.rocm_base` (usually already correct)
- **Runner labels** — Check `nm-cicd/.github/scripts/process_partitions.py` for the mapping of GPU type → runner label. Current ROCm labels follow the pattern `os-<gpu>-<size>-rocm<ver>` (e.g., `os-mi300x-solo-rocm71`).

### 2.2 Run accept-sync

This is the primary validation workflow. It builds a wheel, builds an image, and runs the full test suite (lm-eval, guidellm, test partitions).

```bash
gh workflow run accept-sync.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f wf_category=DEBUG \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<release-branch> \
  -f python=3.12 \
  -f target_device=rocm
```

Replace:
- `<your-nm-cicd-branch>` — your working branch in nm-cicd (e.g., `doug/rocm7-validation`). Use this instead of `main` so you can iterate on CI configs without merging.
- `<release-branch>` — the nm-vllm-ent release branch (e.g., `rhai/0.18.0`)

**Check results:**

```bash
# List recent ROCm accept-sync runs
gh run list --repo neuralmagic/nm-cicd --workflow accept-sync.yml --limit 10 \
  --json databaseId,headBranch,conclusion,displayTitle,createdAt \
  | jq '[.[] | select(.displayTitle | test("rocm"; "i"))]'

# Job-level results for a specific run
gh run view <run_id> --repo neuralmagic/nm-cicd --json jobs \
  --jq '.jobs[] | "\(.conclusion) \(.name)"' | sort
```

### 2.3 Run individual workflows (for faster iteration)

When you're fixing specific failures, you don't need to re-run the full accept-sync. Dispatch individual workflows and reuse the wheel/image from a previous build:

**LM-Eval (accuracy):**
```bash
gh workflow run lm-eval.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f label=<runner-label> \
  -f source_id=<build-run-id> \
  -f lm_eval_configs=neuralmagic/configs/lm_evals/accept_rocm.yml
```

**GuideLLM (performance):**
```bash
gh workflow run guidellm.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f label=<runner-label> \
  -f source_id=<build-run-id> \
  -f guidellm_configs=neuralmagic/configs/guidellm/accept_rocm.yml
```

**Debug (SSH access for investigation):**
```bash
gh workflow run debug.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f label=<runner-label> \
  -f timeout=60 \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<release-branch> \
  -f python=3.12
```

### 2.4 OCP model validation

This deploys models on an OpenShift cluster with ROCm GPUs and validates serving + inference end-to-end. This is distinct from unit tests — it validates the full deployment path including container image, Helm chart, health checks, and actual model responses.

```bash
gh workflow run model-validation-ocp.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f config_ref=main \
  -f label=ubuntu-latest \
  -f timeout=120 \
  -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_rocm.yml \
  -f vllm_image=<image-from-accept-sync> \
  -f source_ref=<release-branch> \
  -f run_accuracy_check=false
```

**MI300X OCP cluster details:**
- The ROCm OCP cluster is Azure-hosted (`amd-01`). Credentials are stored as a GH Actions secret (`OPENSHIFT_PASSWORD`).
- Model configs live in `nm-cicd/neuralmagic/ocp_model_deployment/configs/`.
- If you need to test a single model first, create a minimal config (e.g., `model_validation_rocm_single.yml`) targeting one model.

**Known gotchas:**
- OCP routes take time to propagate — if health checks fail with 503, increase `MAX_RETRIES` in `constants.py` (12 works well with exponential backoff).
- Non-root containers need `HOME` set explicitly. The ROCm UBI Dockerfile should have `ENV HOME=/home/vllm` — if AITER crashes with `PermissionError: //.aiter`, that's the missing env var.
- The `ocp_a100_cluster` GCP secret works for the `amd-01` MI300X cluster. The `openshift-mi300x-user-cred` secret may be stale.

### 2.5 Config files driving the ROCm test matrix

These are the files that define what gets tested. You'll modify these when adding new hardware or adjusting test scope:

| File (in nm-cicd) | What it configures |
|-------------------|-------------------|
| `neuralmagic/configs/partitions/accept_rocm.yml` | Unit test partitions (8 partitions on MI300X) |
| `neuralmagic/configs/partitions/accept_rocm_mi355x.yml` | Unit test partitions for MI355X (if applicable) |
| `neuralmagic/configs/lm_evals/accept_rocm.yml` | LM-eval model configs (solo/duo runners) |
| `neuralmagic/configs/guidellm/accept_rocm.yml` | GuideLLM benchmark models |
| `neuralmagic/ocp_model_deployment/configs/model_validation_rocm.yml` | OCP deployment models (9-model matrix) |

---

## Phase 3: Iterate

This is the hard part. Tests will fail. Your job is to classify each failure and take the right action.

### Failure classification

When a test fails, ask these questions in order:

```
1. Is it a missing dependency? (ray, soundfile, ffmpeg, etc.)
   → Fix in nm-cicd test requirements or setup scripts.

2. Is it an infrastructure issue? (runner label wrong, disk full, stale creds)
   → Fix the infra, re-run.

3. Is it an upstream bug with a known fix?
   → Cherry-pick the upstream PR into the rhai/* branch.

4. Is it a build configuration issue? (wrong arch target, missing env var)
   → Fix in nm-cicd build scripts, rebuild.

5. Is it a new regression in this release?
   → Investigate, fix in nm-vllm-ent, get PR merged into rhai/*.

6. Is it an upstream/vendor bug with no fix yet?
   → File upstream issue, add -k skip filter in test config, document in JIRA.
   → Determine if it's a release blocker or acceptable known issue.
```

### Cherry-picking upstream fixes

This is the most common iteration pattern. You find a fix in upstream `vllm-project/vllm` and need it in midstream:

1. **Identify the upstream PR** — find the fix commit in `vllm-project/vllm`
2. **Cherry-pick into a branch** — create a branch off `rhai/X.Y.Z`, cherry-pick the commit(s)
3. **Build and validate** — run accept-sync against your cherry-pick branch
4. **Open a PR** — PR into `rhai/X.Y.Z` in `nm-vllm-ent`. Also PR into `main` if applicable.
5. **Get it merged** — review, approve, merge. Then request a new tag (e.g., `v0.18.0+rhaiv.5`).

**Finding the retrigger command** (to re-run a specific job from a previous build without rebuilding):
```bash
gh api repos/neuralmagic/nm-cicd/actions/jobs/<job_id>/logs 2>&1 \
  | sed 's/\x1b\[[0-9;]*m//g' \
  | grep -A 20 "Retrigger this job" \
  | grep -E "gh workflow|-f |--repo|--ref" \
  | sed 's/^[^ ]*Z //'
```

### Dependency assessment (CVEs, version conflicts)

Occasionally you'll need to evaluate whether a dependency can be safely removed or upgraded. The pattern:

1. **Check if it's a hard dependency** — look at imports in vLLM source. Are they inside `try/except ImportError`? Behind a lazy-load gate? Only triggered by specific model architectures?
2. **Run validation without it** — create a branch with the dep removed, run accept-sync. Check if your validation models still pass.
3. **Coordinate with AIPCC** — if you change deps in `requirements/rocm.txt`, AIPCC needs a new midstream tag before they can update their pipeline. They can't merge their removal MRs until the vLLM side stops pulling in the offending package.
4. **File upstream** — if the root cause is an overly restrictive version pin in a vendor package, file an issue/PR on the vendor repo.

### When infra changes are needed

Sometimes validation requires new runner labels, build script updates, or even provisioning new hardware. Common scenarios:

- **New GPU architecture** — add the arch to `PYTORCH_ROCM_ARCH` in build scripts, add runner label mappings to `process_partitions.py`, create new partition/lm-eval/guidellm configs
- **New hardware** — coordinate runner setup with the infra team. Runner provisioning scripts and runbooks live in the `stratus` repo under `misc/<hardware>/`
- **Bare metal runners** — may need NUMA balancing disabled (`echo 0 > /proc/sys/kernel/numa_balancing`), env vars sourced from `env-rocm*.sh`, and GPU visibility set via `ROCR_VISIBLE_DEVICES`

These are significant efforts — file separate JIRA stories for them rather than bundling with your validation work.

---

## Phase 4: Tag & Merge

Once accept-sync is green on all target hardware:

1. **Get your nm-vllm-ent changes merged into `rhai/X.Y.Z`** — all cherry-picks, dep changes, etc. should be individual PRs into the release branch.
2. **Get your nm-cicd changes merged into `main`** — build script fixes, test configs, runner labels. Open a PR, get review, merge.
3. **Request a new tag** — after PRs merge, request a re-tag of `rhai/X.Y.Z` (e.g., `v0.18.0+rhaiv.5`). This is typically done by your lead or the release coordinator.
4. **Run a final accept-sync against the tagged commit** — this is your "clean" validation run from the merged state, not your working branch.

---

## Phase 5: Downstream Validation

After the midstream tag is cut, the downstream pipeline takes over:

1. **Renovate picks up the tag** into the AIPCC `/pipeline` repo
2. **AIPCC builds the RC image** — wheel release and container images are built via Tekton
3. **RC image appears** on `quay.io/aipcc/rhaiis/rocm-ubi9:<tag>`

When the RC image is ready (your lead or release coordinator will announce it):

```bash
# Run OCP model validation against the RC image
gh workflow run model-validation-ocp.yml \
  --repo neuralmagic/nm-cicd \
  --ref <your-nm-cicd-branch> \
  -f config_ref=main \
  -f label=ubuntu-latest \
  -f timeout=120 \
  -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_rocm.yml \
  -f vllm_image=quay.io/aipcc/rhaiis/rocm-ubi9:<rc-tag> \
  -f source_ref=<release-branch> \
  -f run_accuracy_check=false
```

**What to watch for in downstream validation:**

The downstream image is built differently from midstream (different base image, different torch build, different packaging). Things that pass in midstream can fail downstream. Common causes:

- **Missing GPU architecture targets** in the downstream torch build — compare `libtorch_hip.so` file size between midstream and downstream images. If downstream is significantly smaller, arch targets may be missing.
- **Different attention backend behavior** — downstream may route through different codepaths (SDPA vs CK vs Triton flash attention). If a model crashes in downstream but not midstream, check which attention backend was selected.
- **Dependency version conflicts** — downstream has additional packages (terratorch, terrakit, etc.) that may conflict with vLLM's deps.

If downstream fails, the fix is usually in the AIPCC pipeline, not your midstream code. File a bug, provide your root-cause analysis, and coordinate with the AIPCC team.

**Report results** to your lead and in `#forum-rhaiis-release`. Include: which models passed, which failed, run IDs, and whether failures are midstream issues or downstream-only.

---

## ROCm Environment Variables

ROCm requires specific environment variables. These are set in `nm-cicd/.github/scripts/setup_env_test_vllm.sh` for test environments and should also be present in runner startup scripts:

| Variable | Value | Purpose |
|----------|-------|---------|
| `HSA_NO_SCRATCH_RECLAIM` | `1` | Required for RCCL on ROCm 7.x |
| `HIP_FORCE_DEV_KERNARG` | `1` | ROCm kernel argument handling fix |
| `NCCL_MIN_NCHANNELS` | `112` | ROCm multi-GPU performance tuning |
| `TORCH_BLAS_PREFER_HIPBLASLT` | `1` | Prefer hipBLASLt for BLAS operations |
| `VLLM_WORKER_MULTIPROC_METHOD` | `spawn` | Ray multiprocessing workaround |
| `RAY_EXPERIMENTAL_NOSET_HIP_VISIBLE_DEVICES` | `1` | Ray >= 2.45 compatibility |

These change between ROCm versions. When upgrading ROCm, check upstream vLLM and AMD documentation for new required variables.

---

## JIRA Tracking

### Story structure

Your ROCm work should be tracked under the release epic. Create stories for distinct work items:

- One story for the overall ROCm validation (maps to acceptance criteria below)
- Separate stories for significant sub-efforts (runner setup, cherry-pick validation, CVE assessment, etc.)
- Bugs filed as Bug type with `[ROCm]` prefix in summary

### Acceptance criteria (standard template)

Every ROCm validation story should track these criteria:

| Criterion | What it means |
|-----------|--------------|
| Initial accept-sync is green | All test partitions, lm-eval, and guidellm pass on all target hardware |
| OCP model validation passes | Models deploy and serve correctly on the ROCm OCP cluster |
| Tag is created in midstream | `rhai/X.Y.Z` has a tag with all your changes (e.g., `vX.Y.Z+rhaiv.N`) |
| Renovate picks up tag into /pipeline | Downstream pipeline detects the new midstream tag |
| Wheel release updated in /container | AIPCC updates the wheel reference |
| Tag created in /containers | AIPCC tags their container build |
| RC image available in quay.io/aipcc | The final RC image is published for validation |

The first three are in your control. The last four are downstream (AIPCC) — you validate, they build.

### Commenting on JIRA issues

When updating JIRA issues, include:
- **Run IDs** with links (e.g., `[run 24673307322](https://github.com/neuralmagic/nm-cicd/actions/runs/24673307322)`)
- **Pass/fail summary** as a table when possible
- **What changed** since last update (commits, PRs, config changes)
- **What's next** — always end with next steps so anyone reading the ticket knows the plan

---

## Key Workflows Reference

### nm-cicd workflows

| Workflow | Purpose | When to use |
|----------|---------|-------------|
| `accept-sync.yml` | Full pipeline: build → image → test fan-out | Primary validation entry point |
| `build-whl-test.yml` | Orchestrates build → image → tests | Called by accept-sync, rarely dispatched directly |
| `lm-eval.yml` | Accuracy evaluation against reference models | Individual re-runs after fixing specific failures |
| `guidellm.yml` | Performance benchmarks | Individual re-runs |
| `model-validation-ocp.yml` | OCP deployment validation | Validates serving on OpenShift cluster |
| `debug.yml` | SSH debug session on a runner | Interactive investigation on hardware |
| `nightly-build-whl-image.yml` | Nightly builds (matrix includes ROCm) | Reference only — nightlies run automatically |

### nm-cicd environment setup

| File | Purpose |
|------|---------|
| `.github/scripts/setup_env_build_vllm.sh` | Build-time env: `PYTORCH_ROCM_ARCH`, `HIPFLAGS`, PyTorch index URL |
| `.github/scripts/setup_env_test_vllm.sh` | Test-time env: Ray compat flags, NCCL tuning, ROCm env vars |
| `.github/scripts/process_partitions.py` | Maps (gpu_type, gpu_count) → runner label |
| `.github/scripts/get_acceptsync_options.py` | Maps `target_device=rocm` → build label, config files |
| `.github/actions/determine-gpu-type/` | Auto-detects GPU via `amd-smi` / `nvidia-smi` |

---

## Debugging Techniques

These are patterns that proved useful during ROCm 7 validation (3.4.0). They're generally applicable to future releases:

### Interactive debugging on GPU hardware

Use the debug workflow to get SSH access:
```bash
gh workflow run debug.yml \
  --repo neuralmagic/nm-cicd \
  --ref <branch> \
  -f label=<runner-label> \
  -f timeout=60
```

Once connected, useful checks:
```bash
# Verify GPU detection
python3 -c "import torch; print(f'GPUs: {torch.cuda.device_count()}, Arch: {torch.cuda.get_device_capability()}')"

# Check ROCm version
cat /opt/rocm/.info/version

# Minimal attention smoke test
python3 -c "
import torch
q = torch.randn(2, 4, 64, 64, device='cuda', dtype=torch.float16)
k = torch.randn(2, 4, 64, 64, device='cuda', dtype=torch.float16)
v = torch.randn(2, 4, 64, 64, device='cuda', dtype=torch.float16)
out = torch.nn.functional.scaled_dot_product_attention(q, k, v, is_causal=True)
print(f'OK: shape={out.shape}, nan={torch.isnan(out).any()}')
"

# For better HIP error localization
AMD_SERIALIZE_KERNEL=3 python3 -c "..."
```

### Checking if a test skip filter is working

When you add `-k "not some_test"` filters to partition configs, verify they're preserved through the dispatch chain. JSON quoting bugs can silently drop the filter. Check the actual pytest command in the job logs.

---

## Lessons Learned (from past releases)

These are real gotchas that have bitten ROCm validation in the past. They're not exhaustive, but they save time:

- **Missing `gfx950` (or future arch) in build scripts** causes a General Protection Fault in `libamdhip64.so` at runtime. The HIP ISA compatibility fallback path has bugs. Always verify `PYTORCH_ROCM_ARCH` includes all target architectures.
- **Deprecated env vars** can expose latent bugs. Example: `VLLM_ROCM_USE_AITER=1` was deprecated (AITER auto-enables when its wheel is installed), but explicitly setting it triggered a dict-iteration ordering bug in config defaults. Don't carry forward env vars from previous releases without checking if they're still valid.
- **Test deps differ between CUDA and ROCm**. ROCm runners may be missing `ray`, `soundfile`, `ffmpeg`, `av`, or other packages that CUDA runners have. Check the test error — `ModuleNotFoundError` or `NoBackendError` usually means a missing dep, not a real test failure.
- **Triton compiler bugs on new architectures** are common. The pattern is: tests crash during JIT compilation, not during model serving. Add a `-k` skip filter in the partition config, file an upstream issue on `triton-lang/triton` or `ROCm/triton`, and remove the filter when the next triton release ships with the fix.
- **OCP route propagation** takes time. If health checks fail immediately after deployment with 503, it's not a vLLM bug — the OpenShift router needs a few seconds to register the route.
- **Non-root container + AITER** requires `HOME` to be set in the Dockerfile. Without it, AITER tries to create its JIT cache at `//.aiter` and crashes with a PermissionError.
- **Downstream torch builds** may be missing GPU arch targets. Compare `libtorch_hip.so` sizes between midstream and downstream images — if downstream is much smaller, architectures were dropped during the build.
- **Don't dispatch builds with `branch=main` on nm-cicd** — it clutters the main view for other teams, and Buildkite build records can't be deleted. Use your own working branch.

---

## People & Channels

| Channel | Purpose |
|---------|---------|
| `#forum-rhaiis-release` | Release coordination, RC announcements, status updates |
| `#forum-aipcc-wheels` | Downstream wheel/pipeline issues |
| `#amd-mi355x-accelerator-cloud-sharing` | MI355X hardware access (or equivalent for future hardware) |

---

## Quick Reference: What "Done" Looks Like

- [ ] Accept-sync GREEN on MI300X (all test partitions + lm-eval + guidellm)
- [ ] Accept-sync GREEN on MI355X (if MI355X is a target for this release)
- [ ] OCP model validation PASSED
- [ ] All nm-vllm-ent changes merged into `rhai/X.Y.Z`
- [ ] All nm-cicd changes merged into `main`
- [ ] Midstream tag exists (e.g., `vX.Y.Z+rhaiv.N`)
- [ ] Downstream RC image validated (smoke test with OCP model validation)
- [ ] Results reported to lead and `#forum-rhaiis-release`
- [ ] JIRA stories updated and closed
