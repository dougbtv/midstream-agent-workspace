---

# RHAIIS 3.4 RC Image Release — Friday 2026-04-25 Runbook

> **Owner:** Doug Smith (handed off from Daniele Trifiro, who has family obligations)
> **Requestor:** Selbi — "make sure we have the RC image so we can test on Monday and push to stage for RHOAI to consume"
> **Created:** 2026-04-24
> **Last updated:** 2026-04-28

---

## Context

Daniele walked me through the RC image release pipeline for RHAIIS 3.4. This is the downstream release flow that takes our midstream nm-vllm-ent tags and turns them into RC container images on quay.io for RHOAI consumption. The `rhaiv.7` tag includes a critical CPU/zen fix (cc Domenic).

**Related upstream work:** See [ROCM7_VALIDATION.md](./ROCM7_VALIDATION.md) for all the midstream validation work that feeds into this.

---

## Key Links

### GitLab Repos (Red Hat internal — cloned locally)

| Repo | Purpose | Local Path | GitLab API Path (URL-encoded) |
|------|---------|------------|-------------------------------|
| **rhaiis/pipeline** | Wheel collection builds (requirements per accelerator) | `./pipeline/` | `redhat%2Frhel-ai%2Frhaiis%2Fpipeline` |
| **rhaiis/containers** | Container image builds, tagging, RC image publishing | `./containers/` | `redhat%2Frhel-ai%2Frhaiis%2Fcontainers` |

### Pipelines to Monitor

| Pipeline | Link |
|----------|------|
| Pipeline (wheel collections) | https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/pipelines |
| Containers (RC images) | https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/pipelines |

### RC Image Destinations (quay.io)

| Accelerator | Quay Repo |
|-------------|-----------|
| CUDA | https://quay.io/repository/aipcc/rhaiis/cuda-ubi9?tab=tags |
| ROCm | https://quay.io/repository/aipcc/rhaiis/rocm-ubi9?tab=tags |
| CPU | https://quay.io/repository/aipcc/rhaiis/cpu-ubi9?tab=tags |

### Key MRs

| MR | Purpose | Status |
|----|---------|--------|
| [pipeline !551](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/551) | Bump vllm to rhaiv.6/7 (CPU gets rhaiv.7 for zen fix, others get rhaiv.6) | Draft — ready to un-draft and merge (mirror synced) |
| [containers !492](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/492) | Example renovate wheel collection update MR | Reference only |
| Container tags | Published at https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/tags | |

### Jira Tickets

| Ticket | Summary | Assignee | Priority |
|--------|---------|----------|----------|
| [INFERENG-1418](https://redhat.atlassian.net/browse/INFERENG-1418) | [CUDA] google/t5gemma-2b-2b-ul2 deployment fails (vLLM v0.10.0) | Doug Smith | Major |
| [INFERENG-6237](https://redhat.atlassian.net/browse/INFERENG-6237) | gpt-oss-120b perf regression on H200 (3.4-GA vs 3.4-EA2) | Daniele | Undefined |
| [INFERENG-6238](https://redhat.atlassian.net/browse/INFERENG-6238) | Ministral-3-14B-Instruct-2512 HIP error on MI300X (vision encoder) | Doug Smith | Undefined |

### Team Skills (from rhaiis-midstream-snippets)

Daniele's team has release automation skills that document the full process with exact `glab api` commands:

| Skill | Covers | Relevant to us? |
|-------|--------|-----------------|
| `rhaiis-release-branch` | Steps 2-3: Create release branches, enable repeatable builds | Not needed now (3.4 branch already exists) |
| `rhaiis-release-infra` | Step 4: Add version entry to infrastructure product config | Not needed now |
| `rhaiis-release-packages` | Step 1: Package diff between vLLM versions, create AIPCC issues | Not needed now |
| `rhaiis-release-renovate` | Steps 5-7: Update Renovate configs and product version | Useful reference for how renovate works |
| `release_images.sh` | Build release images via GH Actions `build-whl-image.yml` | Reference — this is the *midstream* image build path, not the downstream AIPCC path we're using today |

---

## Pipeline Repo Structure (what we learned)

The `pipeline/collections/rhaiis/` directory has per-accelerator subdirectories, each with a `requirements.txt`:

```
collections/rhaiis/
├── cpu-ubi9/requirements.txt        # vllm[audio,tensorizer,zen]==0.18.0+rhaiv.X
├── cuda13.0-ubi9/requirements.txt   # vllm[audio,tensorizer,cgraph-cuda13]==0.18.0+rhaiv.X + flashinfer, triton, etc.
├── gaudi-ubi9/requirements.txt
├── neuron-ubi9/requirements.txt
├── rocm7.1-ubi9/requirements.txt    # vllm[audio,tensorizer]==0.18.0+rhaiv.X + flash-attn
├── spyre-ubi9/requirements.txt
└── tpu-ubi9/requirements.txt
```

**MR !551 diff (the version bump we need to merge):**
- CPU: `rhaiv.5` → `rhaiv.7` (zen fix is CPU-only, hence different tag)
- CUDA: `rhaiv.5` → `rhaiv.6`
- ROCm: `rhaiv.5` → `rhaiv.6`
- Spyre: `rhaiv.5` → `rhaiv.6`

The MR also notes: *"Includes Mistral 4 tool calling fixes (INFERENG-6229). CPU uses `v0.18.0+rhaiv.7` because of INFERENG-5257 / nm-vllm-ent #484 (partial revert of #478)"*

## Containers Repo Structure (what we learned)

The `containers/` repo has per-accelerator Containerfiles and build-args:

```
containers/
├── Containerfile.cpu-ubi9
├── Containerfile.cuda-ubi9
├── Containerfile.rocm-ubi9
├── Containerfile.gaudi-ubi9
├── Containerfile.neuron-ubi9
├── Containerfile.spyre-ubi9
├── build-args/              # Build arg configs per accelerator
└── renovate.json            # Renovate auto-updates wheel collection refs
```

**Existing tag pattern (from containers repo):**
```
rhaiis-cuda-v2026042401    ← yesterday
rhaiis-rocm-v2026042401    ← yesterday
rhaiis-spyre-v2026042401   ← yesterday
rhaiis-neuron-v2026042401  ← yesterday
(no CPU tag since 2026041502 — waiting on the zen fix!)
```

No CPU tag since April 15 — confirms the CPU/zen fix in rhaiv.7 is the blocker.

---

## RC Image Release Runbook (from Daniele's handoff)

### Step 0: Validate gemma issue (INFERENG-1418)

- [x] Run gemma validation test: **Ran direct GPU reproducer on a100-07** (A100-SXM4-80GB, podman + single GPU)
  - Used `quay.io/vllm/automation-vllm:cuda-24851595501` (vLLM 0.18.1.dev6+rhaiv.5) because the `rhaiv.6` image has a separate `libcudart.so.13` build issue
- [x] Record result: **FAIL** — bug still reproduces on 0.18.0. Crash path evolved: architecture resolves to `TransformersMultiModalForCausalLM`, falls back to Transformers, then crashes with `TypeError: Invalid type of HuggingFace processor. Expected ProcessorMixin, found GemmaTokenizerFast`
- [x] If FAIL: **Not blocking RC** — upstream vLLM limitation with T5Gemma encoder-decoder models. Model has never worked. Jira comment added 2026-04-24. **Issue moved to Review with recommendation to close.**

**Bug context:** T5Gemma is an encoder-decoder model and vLLM dropped encoder-decoder support. There's an [open PR (vllm#32617)](https://github.com/vllm-project/vllm/pull/32617) in vLLM core but it's blocked with critical review issues. See also [vllm#31004](https://github.com/vllm-project/vllm/issues/31004), [vllm#31410](https://github.com/vllm-project/vllm/issues/31410). Jira transitioned to Review 2026-04-24.

**Update 2026-04-25 (from Tarun):** Fix path exists via [bart-plugin PR #8](https://github.com/vllm-project/bart-plugin/pull/8). Once merged, t5gemma support comes through the bart-plugin (already included in CUDA wheel collection as `vllm-bart-plugin>=0.3.3`). Not blocking RC — fix will land in a future rhaiv bump.

### Step 1: Wait for rhaiv.7 tag mirror sync

- [x] Confirm `rhaiv.7` tag is out in nm-vllm-ent midstream — **confirmed 2026-04-25**
- [x] Confirm mirror has synced to GitLab — **confirmed 2026-04-25** (both `v0.18.0+rhaiv.6` and `v0.18.0+rhaiv.7` present)

**GitLab mirror project:** `redhat/rhel-ai/core/mirrors/github/neuralmagic/nm-vllm-ent`

**How to check mirror sync:**
```bash
# Check if the tag exists in the GitLab mirror (correct project path)
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Fcore%2Fmirrors%2Fgithub%2Fneuralmagic%2Fnm-vllm-ent/repository/tags?search=rhaiv&per_page=20" | \
  jq -r '.[].name' | sort -V
```

### Step 2: Merge pipeline MR !551

- [ ] Un-draft MR !551: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/551
- [ ] Merge it
- [ ] Monitor pipeline: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/pipelines

**What happens:** When merged, CI builds wheel collections for each accelerator and publishes them to the GitLab package registry. Collection names follow `3.4.<build_number>` pattern with rhaiis + target device in the name.

**Monitor via CLI:**
```bash
glab ci status -R redhat/rhel-ai/rhaiis/pipeline
# or
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Frhaiis%2Fpipeline/pipelines?per_page=5&order_by=id&sort=desc" | \
  jq '.[] | {id, status, ref, created_at}'
```

### Step 3: Wait for wheel collections to build

- [ ] CUDA wheel collection: ______
- [ ] ROCm wheel collection: ______
- [x] **CPU wheel collection: FAILED** — zentorch version mismatch (see below)

**CPU Build Failure (INFERENG-5257):**

zentorch wheel builds as version `5.2` but the pipeline validation expects `5.2.1`:
```
ERROR zentorch-5.2-cp312-cp312-linux_x86_64.whl '5.2' does not match public version '5.2.1'
```
- Failed job: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/jobs/14083017174
- Pipeline: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/pipelines/2478218870
- Pinged Domenic (zentorch owner) at 3:50 PM for guidance

**Root cause:** The fromager builder (v28.22.0) resolves `zentorch==5.2.1` from PyPI but hardcodes `major.minor` for the git tag clone, so it checks out the `5.2` tag and builds a `5.2` wheel — which then fails post-build validation against the expected `5.2.1`.

**Fix:** Pipeline [MR !552](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/552) (targeting `3.4`) pins `zentorch==5.2` in CPU constraints to match what the builder actually produces. Once merged, retrigger the CPU wheel collection build on `3.4`. CUDA/ROCm builds from !551 are unaffected.

**ROCm / Spyre Build Failures (pyarrow hermetic build issue):**

ROCm x86_64 and all three Spyre variants also failed — same root cause: pyarrow 24.0.0 tries to download `xsimd` from github.com during its CMake build, but the builder runs in network isolation (`fromager`'s `run_network_isolation.sh`). The download fails because the sandbox is hermetic and can't reach github.com.

```
ERROR downloading 'https://github.com/xtensor-stack/xsimd/archive/14.0.0.tar.gz' failed
status_string: "Couldn't resolve host name"
```

- ROCm failed job: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/jobs/14083017173
- Retried ROCm: https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/jobs/14083788674 (pending)
- CUDA builds are still running (may have xsimd cached from prior builds)

**Root cause confirmed:** pyarrow 24.0.0 (released Apr 21) added `xsimd` as a new build-time dep. The `fromager` bootstrap didn't know to pre-fetch it, so the hermetic build stage fails trying to download from github.com. Nothing in our rhaiv.5→rhaiv.6/7 delta changed pyarrow or its deps — this is a builder issue.

**Fix:** [Pipeline MR !553](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/553) bumps the fromager builder from v28.22.0 → v28.26.0, which includes the full xsimd bootstrap fix (v28.25.1 had a `pyarrow<24` workaround; v28.26.0 is the proper fix). MR CI is green — needs approval, then merge. Once merged, retry the failed ROCm/CPU/Spyre jobs.

**Current state (updated):**
- CUDA: still building (~2hrs for big collections)
- ROCm/CPU/Spyre: blocked on !553 merge, then retry failed jobs
- !552 (zentorch pin) also needed for CPU specifically

**Sequence:** Approve + merge !553 (builder v28.26.0) + !552 (zentorch pin) → retry failed ROCm/CPU/Spyre jobs on pipeline `2478218870`.

**Check pipeline jobs:**
```bash
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Frhaiis%2Fpipeline/pipelines/<PIPELINE_ID>/jobs" | \
  jq '.[] | {name, status, stage}'
```

### Step 4: Update containers repo with new collection IDs

**Renovate should handle this automatically.** It creates a MR in `rhaiis/containers` referencing the newly built collections.

- [ ] Renovate MR appeared in https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests
- [ ] MR is targeting branch `3.4`
- [ ] Review and merge the renovate MR

See [!492](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/492) for what this looks like.

**Check for renovate MRs:**
```bash
glab mr list -R redhat/rhel-ai/rhaiis/containers --target-branch 3.4
```

### Step 5: Create accelerator tags for RC images

After the collection ID MR is merged (on branch `3.4`), create a tag for **each accelerator** to trigger RC image builds.

**Tag pattern:** `rhaiis-<accelerator>-v<YYYYMMDD><two-digit-build-number>`

- [ ] Created tag: `rhaiis-cuda-v__________`
- [ ] Created tag: `rhaiis-rocm-v__________`
- [ ] Created tag: `rhaiis-cpu-v__________`

Tags are published to: https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/tags

**Create tags via glab API (no local checkout needed):**
```bash
# Get the latest commit on 3.4 branch
COMMIT=$(glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/repository/branches/3.4" | \
  jq -r '.commit.id')

# Create tags
for accel in cuda rocm cpu; do
  glab api --hostname gitlab.com --method POST \
    "projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/repository/tags" \
    -f tag_name="rhaiis-${accel}-v2026042501" \
    -f ref="$COMMIT"
done
```

**Or via local git:**
```bash
cd containers
git fetch origin 3.4
git checkout 3.4
for accel in cuda rocm cpu; do
  git tag "rhaiis-${accel}-v2026042501"
done
git push origin --tags
```

### Step 6: Monitor container image pipelines

- [ ] Monitor: https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/pipelines
- [ ] CUDA pipeline: PASS / FAIL
- [ ] ROCm pipeline: PASS / FAIL
- [ ] CPU pipeline: PASS / FAIL

```bash
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/pipelines?per_page=10&order_by=id&sort=desc" | \
  jq '.[] | {id, status, ref, created_at}'
```

### Step 7: Verify RC images on quay.io

- [ ] CUDA image visible: https://quay.io/repository/aipcc/rhaiis/cuda-ubi9?tab=tags
- [ ] ROCm image visible: https://quay.io/repository/aipcc/rhaiis/rocm-ubi9?tab=tags
- [ ] CPU image visible: https://quay.io/repository/aipcc/rhaiis/cpu-ubi9?tab=tags

Record image tags:
```
CUDA: quay.io/aipcc/rhaiis/cuda-ubi9:__________
ROCm: quay.io/aipcc/rhaiis/rocm-ubi9:__________
CPU:  quay.io/aipcc/rhaiis/cpu-ubi9:__________
```

**Verify via skopeo:**
```bash
skopeo inspect --no-tags --override-arch amd64 --override-os linux \
  docker://quay.io/aipcc/rhaiis/cuda-ubi9:<TAG> | \
  jq '{git_url: .Labels["git.url"], git_commit: .Labels["git.commit"], release: .Labels["release"], version: .Labels["version"]}'
```

### Step 8: Share RC images

- [ ] Post RC image tags to `#forum-rhaiis-release` Slack channel
- [ ] Notify Selbi that RC images are available for Monday testing

---

## Quay Access Blocker (dosmith)

**Status: IN PROGRESS** — request filed 2026-04-27, waiting on AIPCC team to process.

### What happened

`dosmith` on quay.io cannot read `quay.io/aipcc/rhaiis/*` repos (cuda-ubi9, rocm-ubi9, cpu-ubi9). Confirmed 2026-04-27: `podman login quay.io` succeeds (creds in `~/.docker/config.json`) but `skopeo inspect` returns `unauthorized` on all `aipcc` org repos including `base-images`. The account isn't a member of the `aipcc` org on quay.io.

**Gotcha:** podman stores creds in `$XDG_RUNTIME_DIR/containers/auth.json` but may also check `~/.docker/config.json`. Skopeo uses the same files. Even if `podman login quay.io` says "Already logged in", the creds only prove identity — they don't grant org-level permissions. The real issue is org membership, not authentication.

### How to request quay.io/aipcc access

The process is documented in the [AIPCC-14666](https://redhat.atlassian.net/browse/AIPCC-14666) template ticket. Steps:

1. **Fork and MR in `app-interface`:** Fork `gitlab.cee.redhat.com/service/app-interface` (big repo — clone will hang, use the API to fork + commit instead). Create a user YAML file:

   ```yaml
   # data/teams/rhel-ai/users/<org_username>.yml
   ---
   $schema: /access/user-1.yml

   labels: {}

   name: Your Name
   org_username: your-kerberos-id
   github_username: your-github
   quay_username: your-quay-username

   roles:
   - $ref: /teams/rhel-ai/roles/consumer.yml
   ```

   The `consumer.yml` role grants: quay membership (aipcc-consumers), ci-int read-only, and Konflux contributor access to rhel-ai and ai-tenant namespaces. Read-only — no manager approval needed.

   Example MR for reference: [app-interface !162764](https://gitlab.cee.redhat.com/service/app-interface/-/merge_requests/162764/diffs)

   **Important:** You must add `devtools-bot` (App SRE bot, user ID 3889) as a **Maintainer** to your fork, otherwise the MR CI won't run. Developer (access level 30) is **not enough** — the bot needs Maintainer (access level 40) to trigger the external CI pipeline. If you get it wrong, the pipeline will show as "failed" with zero jobs. Via API:
   ```bash
   # Add devtools-bot as Maintainer (access_level=40, NOT 30)
   glab api --hostname gitlab.cee.redhat.com --method POST \
     "projects/<YOUR_FORK_PROJECT_ID>/members" \
     -f user_id=3889 -f access_level=40

   # If you already added it at the wrong level, update it:
   glab api --hostname gitlab.cee.redhat.com --method PUT \
     "projects/<YOUR_FORK_PROJECT_ID>/members/3889" \
     -f access_level=40
   ```
   After fixing permissions, push a new commit (even a no-op update to the same file) to trigger a fresh pipeline — you can't retry the zero-job failed one.

   **Also note:** app-interface is a high-traffic repo. Your MR will frequently flip to `need_rebase` between pipeline runs. Rebase via API:
   ```bash
   glab api --hostname gitlab.cee.redhat.com --method PUT \
     "projects/service%2Fapp-interface/merge_requests/<MR_IID>/rebase"
   ```

2. **Create Jira ticket in AIPCC project:** Use Task type, title format `Grant quay.io/aipcc Access for <username> to quay.io/aipcc`. Link under epic [AIPCC-10105](https://redhat.atlassian.net/browse/AIPCC-10105). Fill in: name, Red Hat email, org/team, justification, and the app-interface MR link.

   **Note:** You may not have create permission in the AIPCC Jira project. If so, create it through the Jira web UI (which may have different permissions than the API), or ask someone on the AIPCC team to create it.

3. **Ping `@aipcc-support` in `#forum-aipcc` Slack channel** with links to the Jira ticket and MR.

### What we filed (dosmith, 2026-04-27)

| Item | Link |
|------|------|
| Jira ticket | [AIPCC-14939](https://redhat.atlassian.net/browse/AIPCC-14939) (linked under epic AIPCC-10105) |
| app-interface MR | [!185422](https://gitlab.cee.redhat.com/service/app-interface/-/merge_requests/185422) — CI green, rebased, handed off to Rishup (aipcc-support) to merge |
| Slack request | [#forum-aipcc message](https://redhat-internal.slack.com/archives/C07JX0EMKCZ/p1777299768857059) |

**Short-term workaround:** Ask Daniele or Selbi (who already have access) to confirm images are on quay.

---

## Build Blockers

### INFERENG-5257: zentorch version mismatch (CPU build failure)

- **Impact:** CPU wheel collection build fails. CUDA/ROCm unaffected.
- **Error:** `zentorch-5.2-cp312-cp312-linux_x86_64.whl '5.2' does not match public version '5.2.1'`
- **Root cause:** Builder `v28.22.0`'s zentorch fromager plugin (`package_plugins/zentorch.py`) hardcodes `f"v{version.major}.{version.minor}"` for both git tag clone and `ZenTorch_BUILD_VERSION`. AMD released `zentorch 5.2.1` on PyPI (and pushed `v5.2.1` tag to their mirror), so fromager resolves `5.2.1` but the builder clones `v5.2` and builds version `5.2`. Post-build validation rejects the mismatch. Builder `main` branch has a `version_to_tag()` fix, but the pipeline is pinned to `v28.22.0`.
- **Fix:** [pipeline !552](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/552) — pins `zentorch==5.2` in CPU constraints.
- **Failed job:** https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/jobs/14083017174
- **Next steps:** Merge !552, retrigger CPU wheel collection build on `3.4`.

---

## Last-Minute Bugs (from perf validation runs)

These were flagged by Harshith just before the release:

### INFERENG-6237: gpt-oss-120b H200 perf regression

- **Impact:** H200-specific throughput regression (-8% to -17%), ITL degradation (+13% to +46%). AMD MI300X unaffected.
- **Assignee:** Daniele (Doug covering)
- **Blocking RC? No.** Regression is from the full vLLM 0.16.0→0.18.0 major version upgrade (~1250 commits), not a specific cherry-pick. MR !551 (rhaiv.5→rhaiv.6) has zero CUDA decode changes. Model works functionally. Upstream vllm#39004 shows similar B200 decode regression — may share root cause (flashinfer upgrade). Investigation continues next week. Jira comment posted 2026-04-24.

### INFERENG-6238: Ministral-3-14B vision encoder HIP error on MI300X

- **Impact:** Model fails to deploy on MI300X. `torch.AcceleratorError: HIP error: invalid argument` during Pixtral vision encoder profile run.
- **Assignee:** Doug Smith
- **Blocking RC? No (likely).** Unable to reproduce as of 2026-04-25. Sending additional tests to confirm. Not blocking RC pending further investigation.

### INFERENG-1418: t5gemma-2b-2b-ul2 deployment failure

- **Impact:** Model deployment fails — `T5GemmaForConditionalGeneration` falls back to Transformers backend, then crashes.
- **Assignee:** Doug Smith
- **Blocking RC? No.** Validated on 0.18.0 — still fails. Fix path via [bart-plugin PR #8](https://github.com/vllm-project/bart-plugin/pull/8) (from Tarun, 2026-04-25). Once merged, support comes through `vllm-bart-plugin` which is already in the CUDA wheel collection. Will land in a future rhaiv bump, not this RC.

---

## Tooling Setup (completed 2026-04-24)

### glab (GitLab CLI)

- **Installed:** `glab v1.89.0` via `dnf install glab`
- **Authenticated:** as `dosmith` on `gitlab.com`
- **Git credential helper configured:** `git config --global credential.https://gitlab.com.helper '!glab auth git-credential'`
- **SSH:** GitLab host keys added to `~/.ssh/known_hosts` (SSH clone not working yet — using HTTPS with credential helper)
- **Red Hat SSO:** https://red.ht/GitLabSSO (off VPN)

### Repos cloned (HTTPS, in worktree-nm/)

- `./pipeline/` — `redhat/rhel-ai/rhaiis/pipeline` (main branch)
- `./containers/` — `redhat/rhel-ai/rhaiis/containers` (main branch)

---

## Step 9: RC Image Testing — What It Means and How To Do It

### What "run it through tests" means

When Selbi says "run the RC images through tests", that means running the **model-validation-ocp** workflow in `neuralmagic/nm-cicd` against the published `quay.io/aipcc/rhaiis/*` images. This workflow:

1. **SETUP** — Parses a model config YAML to build a matrix of models to test
2. **COLLECT_IMAGE_INFO** — Pulls the image, extracts SBOM (vLLM version, image version) via anchore
3. **Per-model validation** — For each model in the matrix:
   - Deploys vLLM as a pod on OCP (or ARC runner k8s cluster)
   - Runs smoke tests (completions, chat, vision, transcription, embeddings, rerank — depending on model type)
   - Optionally runs accuracy checks via lm-eval
4. **SECURITY / SNYK_IMAGE** — Runs Snyk vulnerability scan on the container image

### RC image tags (from Daniele's Sunday tagging)

| Accelerator | Image | Tag timestamp |
|-------------|-------|---------------|
| CUDA | `quay.io/aipcc/rhaiis/cuda-ubi9:3.4.0-1777218914` | 2026-04-26 |
| ROCm | `quay.io/aipcc/rhaiis/rocm-ubi9:3.4.0-1777218934` | 2026-04-26 |
| CPU | `quay.io/aipcc/rhaiis/cpu-ubi9:3.4.0-*` | TBD — no CPU validation run found yet |

### Existing validation runs (Tarun, 2026-04-27)

Tarun (tarukumar) already kicked off validation on both CUDA and ROCm RC images from the `refactor/smoke-capability-driven` branch:

| Image | Run | Result | Models Tested |
|-------|-----|--------|---------------|
| `cuda-ubi9:3.4.0-1777218914` | [24977680020](https://github.com/neuralmagic/nm-cicd/actions/runs/24977680020) | All model tests PASSED, Snyk FAILED | 22 models: phi-4, gemma-3-12b, Mixtral-8x7B, gpt-oss-120b, DeepSeek-R1, Llama-3.3-70B-FP8, Qwen3-30B, whisper, vision models, embeddings, rerankers |
| `rocm-ubi9:3.4.0-1777218934` | [24983454145](https://github.com/neuralmagic/nm-cicd/actions/runs/24983454145) | All model tests PASSED, Snyk FAILED | 13 models: phi-4, Phi-3-medium, granite-4.0-h-small, granite-3.3-8b, Llama-3.1-8B-FP8, DeepSeek-R1, Llama-3.3-70B-FP8, Qwen3-8B-FP8, whisper, Llama-Guard-4 |
| `cpu-ubi9:3.4.0-*` | — | Not yet run | — |

**Verdict:** Functional smoke tests all pass. Only failure is Snyk security scan (compliance, not functional).

### How to run RC validation yourself

The `model-validation-ocp.yml` workflow is dispatched via `gh workflow run`. Auth to `quay.io/aipcc` is handled by GitHub Actions secrets (`QUAY_AIPCC_USERNAME` / `QUAY_AIPCC_SECRET`) — a robot account, so your personal quay access doesn't matter.

**CUDA (H100):**
```bash
gh workflow run model-validation-ocp.yml \
    --repo neuralmagic/nm-cicd \
    --ref refactor/smoke-capability-driven \
    -f config_ref=summit-new-model \
    -f label=ibm-wdc-k8s-h100-util \
    -f timeout=600 \
    -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_cuda_h100.yaml \
    -f vllm_image=quay.io/aipcc/rhaiis/cuda-ubi9:3.4.0-1777218914 \
    -f source_ref=v0.18.0+rhaiv.7 \
    -f run_accuracy_check=true \
    -f platform=linux/amd64
```

**ROCm (MI300X):**
```bash
gh workflow run model-validation-ocp.yml \
    --repo neuralmagic/nm-cicd \
    --ref refactor/smoke-capability-driven \
    -f config_ref=summit-new-model \
    -f label=ubuntu-latest \
    -f timeout=600 \
    -f ocp_validation_models=neuralmagic/ocp_model_deployment/configs/model_validation_rocm.yml \
    -f vllm_image=quay.io/aipcc/rhaiis/rocm-ubi9:3.4.0-1777218934 \
    -f source_ref=v0.18.0+rhaiv.7 \
    -f run_accuracy_check=true \
    -f platform=linux/amd64
```

**Key parameters explained:**

| Parameter | What it is | How to set it |
|-----------|-----------|---------------|
| `--ref` | nm-cicd branch with the workflow code | Use `refactor/smoke-capability-driven` (Tarun's latest) or `main` |
| `config_ref` | Branch of `model-validation-configs` repo for ground truth/server configs | `summit-new-model` or `main` |
| `label` | GitHub Actions runner label | `ibm-wdc-k8s-h100-util` (CUDA H100), `ubuntu-latest` (ROCm — routes to AMD OCP cluster) |
| `ocp_validation_models` | YAML file listing which models to test | See `neuralmagic/ocp_model_deployment/configs/` for per-accelerator lists |
| `vllm_image` | The RC image to test | `quay.io/aipcc/rhaiis/<accel>-ubi9:<tag>` |
| `source_ref` | The vLLM version tag used to build the image | `v0.18.0+rhaiv.7` for this RC |
| `run_accuracy_check` | Whether to run lm-eval accuracy tests | `true` for full validation |

### Model config files (what gets tested)

Located in `nm-cicd/neuralmagic/ocp_model_deployment/configs/`:

| Config | Accelerator | Models |
|--------|-------------|--------|
| `model_validation_cuda_h100.yaml` | CUDA (H100) | 22 models: language (phi-4, gemma-3, Qwen3, Mixtral, DeepSeek-R1, gpt-oss-120b, ...), vision (Qwen2.5-VL, Mistral-Small-3.1), transcription (whisper, Voxtral), embeddings (nomic, MiniLM, embeddinggemma), rerank (bge-reranker) |
| `model_validation_rocm.yml` | ROCm (MI300X) | 10 models: language only (phi-4, Phi-3-medium, granite-4.0-h, granite-3.3-8b, Llama-3.1-8B-FP8, DeepSeek-R1, Llama-3.3-70B-FP8, Qwen3-8B-FP8, Mistral-Small-3.1) |
| `model_validation_cuda.yml` | CUDA (generic) | Smaller set, different runner labels |
| `model_validation_cuda_minimal.yml` | CUDA (A10/RTX6000) | Minimal set for smaller GPUs |

### Simpler single-model smoke test (ocp-test.yml)

For quick single-model validation without the full matrix, use `ocp-test.yml`:

```bash
gh workflow run ocp-test.yml \
    --repo neuralmagic/nm-cicd \
    --ref main \
    -f config_ref=main \
    -f label=ibm-wdc-k8s-h100-util \
    -f timeout=60 \
    -f model=microsoft/phi-4 \
    -f vllm_image=quay.io/aipcc/rhaiis/cuda-ubi9:3.4.0-1777218914 \
    -f target_device=cuda \
    -f model_type=language \
    -f ocp_host_url="" \
    -f test_type=smoke
```

### Podman-based local testing (no k8s needed)

For quick local validation on a GPU box, use `scripts/run-rhaiis-image-test.sh`:

```bash
# Requires: podman logged into quay.io, a GPU available
cd nm-cicd
./scripts/run-rhaiis-image-test.sh \
    quay.io/aipcc/rhaiis/cuda-ubi9:3.4.0-1777218914 \
    microsoft/phi-4
```

This starts the container with podman, waits for health, then hits `/v1/completions` and `/v1/chat/completions`.

---

## Saturday Morning Follow-up (2026-04-25)

Overnight cron monitoring caught that CUDA container builds were failing on Renovate MR !499 with `ResolutionImpossible: vllm==0.18.0+rhaiv.6 vs vllm==0.18.0+rhaiv.7`. Spent Saturday morning tracking down the root cause:

- **Root cause:** MR !551 ("bump vllm version to rhaiv.6/7") only bumped CPU to rhaiv.7 (for INFERENG-5257). CUDA, ROCm, and Spyre `requirements.txt` were left on rhaiv.6. With both rhaiv.6 and rhaiv.7 in the mirror, fromager generated constraints for rhaiv.7 that conflict with the rhaiv.6 requirement.
- **Fix:** Created [!555](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/555) to bump all three to rhaiv.7.
- **Next steps:** Need approval on !555, merge, trigger new wheel collection build, wait for Renovate to update !499, then merge !499 and create accelerator tags.
- **Slack:** Posted to team asking for approval on !555. Aiming for images by Monday.

---

## Sunday Update (2026-04-26)

Daniele (dtrifiro-rh) picked up the baton:
- Approved + merged [!555](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/555) (rhaiv.7 requirements fix)
- Wheel collections rebuilt with correct rhaiv.7 pins
- Renovate auto-updated [!499](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/499), CI passed, Daniele merged it
- Created accelerator tags: `rhaiis-cuda-v2026042601`, `rhaiis-rocm-v2026042602`, `rhaiis-cpu-v2026042601`
- Container image builds now running via Tekton. Waiting for images on quay.io.

---

## Status Summary

| Step | Status | Notes |
|------|--------|-------|
| 0a. Gemma validation (INFERENG-1418) | [x] Done — not blocking | FAIL on 0.18.0: no upstream encoder-decoder support. Jira moved to Review w/ recommendation to close. |
| 0b. H200 perf regression (INFERENG-6237) | [x] Done — not blocking | vLLM 0.16→0.18 major version delta (~1250 commits). MR !551 is safe. Investigate next week. Jira comment posted. |
| 1. rhaiv.7 mirror sync | [x] Done | Both rhaiv.6 and rhaiv.7 confirmed in GitLab mirror (`core/mirrors/github/neuralmagic/nm-vllm-ent`) |
| 2. Merge pipeline !551 | [x] Done | Approved + merged by dosmith. Commit `0fd9c8f9`. |
| 3a. Wheel collection build (CUDA) | [x] **DONE** | Collection rebuilt with rhaiv.7 after !555. Daniele handled the rebuild cycle. |
| 3b. Wheel collection build (ROCm) | [x] **DONE** | Rebuilt with rhaiv.7 after !555. |
| 3c. Wheel collection build (CPU) | [x] **DONE** | Pipeline `2478508404`: already on rhaiv.7. |
| 3d. Wheel collection build (Spyre) | [x] **DONE** | Rebuilt with rhaiv.7 after !555. |
| 3*. Builder bump v28.22.0→v28.26.0 | [x] **MERGED** | [!553](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/553) merged by dosmith. Commit `8b53f671`. |
| 3**. Zentorch pin (CPU) | [x] **MERGED** | [!552](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/552) merged by dosmith. Commit `dcf44010`. |
| 3***. rhaiv.7 requirements fix | [x] **MERGED** | [!555](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/555) by dosmith. Bumps CUDA/ROCm/Spyre requirements from rhaiv.6→rhaiv.7. Daniele approved + merged. |
| 4. Renovate MR in containers | [x] **MERGED** | [!499](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/499) merged by dtrifiro-rh (Daniele). Merge commit `b006c3dc`. |
| 5. Create accelerator tags | [x] **DONE** | Created by Daniele: `rhaiis-cuda-v2026042601`, `rhaiis-rocm-v2026042602` (v2026042601 was taken from 3 days ago), `rhaiis-cpu-v2026042601`. All point to merge commit `b006c3dc`. |
| 6. Container pipelines | [x] **DONE** | GitLab CI pipelines on `main` and `3.4` both succeeded. Tekton builds triggered by tags. |
| 7. Verify quay images | [ ] **BLOCKED — no quay access** | quay.io/aipcc/rhaiis repos are private. `dosmith` quay account lacks read access to the `aipcc` org. Need to file AIPCC access request (see below). |
| 8. Share in Slack | [ ] Waiting on step 7 | Post to #forum-rhaiis-release for Selbi's team. |
| 9. RC testing (Selbi) | [ ] **IN PROGRESS** | Selbi (2026-04-28 3:15 AM): "Also new RHAIIS RC for CUDA, ROCm and Spyre available. @Daniele @Doug could you please run it through tests?" Tarun already running validation — see RC Testing section below. |
