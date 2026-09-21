# vLLM-Omni AIPCC How-To

Working notes for interacting with the AIPCC GitLab repos for vllm-omni
containerization. Based on the INFERENG-9465 work (July 2026), with the
release-branch coordination workflow last verified during the 3.6-fast2 carry
forward (September 2026).

## Repos and What They Do

| Repo | Purpose | Local Clone |
|------|---------|-------------|
| `redhat/rhel-ai/rhaiis/containers` | Containerfiles, build-args, renovate config | `repos/aipcc/containers/` |
| `redhat/rhel-ai/rhaiis/pipeline` | Wheel building pipeline (fromager-based) | (no local clone, use `glab` API) |
| `redhat/rhel-ai/core/infrastructure` | GitLab registry/index definitions | (no local clone, use `glab` API) |

## AIPCC Commit Conventions

Every commit and MR title **must** follow this format or the linter will fail:

```
INFERENG-XXXX: Short description

Longer description.

Signed-off-by: Your Name <email@redhat.com>
```

- Title must start with a JIRA ticket: `(RHELAI|RHOAIENG|AIPCC|INFERENG|RHAIENG)-XXXX:`
- Must include `Signed-off-by:` (use `git commit --signoff`)
- MR description also needs `Signed-off-by:` at the bottom
- MR description must be plain text, under 2500 chars (GitLab truncates for CI linters)

## Checking GitLab CI Pipelines

GitLab CI still runs repository checks such as the central linter. The
vllm-omni container build itself is Konflux/Tekton-only; it appears on the
commit as an external status rather than as a GitLab CI build job.

```bash
# List recent pipelines on a repo
glab ci list --repo redhat/rhel-ai/rhaiis/containers

# Get jobs for a specific pipeline
glab api "projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/pipelines/<PIPELINE_ID>/jobs" \
  | python3 -c "import json,sys; jobs=json.load(sys.stdin); [print(f\"{j['status']:12s} {j['name']}\") for j in jobs]"

# Get job logs (tail)
glab api "projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/jobs/<JOB_ID>/trace" | tail -60
```

## Containers Repo: File Layout for a New Collection

vllm-omni follows the `model-opt` pattern with dot-separated names:

```
containers/
  Containerfile.vllm-omni.cuda-ubi9       # the Containerfile
  build-args.vllm-omni/
    cuda-ubi9.conf                         # build args (wheel releases, base image)
  renovate.json                            # auto-update config for wheel releases
  .tekton/
    rhaiis-vllm-omni-cuda-ubi9-on-pull-request.yaml
    rhaiis-vllm-omni-cuda-ubi9-on-push.yaml
```

### build-args conf file

```ini
BASE_IMAGE=quay.io/aipcc/base-images/cuda-13.0-el9.8:3.6.0-ea.1-1787782448
RHAIIS_VERSION=3.6-fast1
WHEEL_RELEASE_PROJECT_ID=68845358
WHEEL_RELEASE_PACKAGE=vllm-omni-wheels
WHEEL_RELEASE_AARCH64=3.6-fast1.3898+vllm-omni-cuda13.0-ubi9-aarch64
WHEEL_RELEASE_X86_64=3.6-fast1.3898+vllm-omni-cuda13.0-ubi9-x86_64
```

### Containerfile key patterns

- `ARG WHEEL_RELEASE_AARCH64` and `ARG WHEEL_RELEASE_X86_64` declared at top
- Both referenced in the `com.redhat.aiplatform.wheel_release` label
- Entrypoint for omni: `ENTRYPOINT ["python3", "-m", "vllm_omni.entrypoints.openai.api_server", "--omni"]`
- No chat templates (that's standard vllm, not omni)
- No tiktoken pre-download (that's gpt-oss, not omni)
- `espeak-ng` is installed with `dnf`; the Konflux build service account has the
  subscription access needed for this step

## Release-Branch Productization Changes and MR Ordering

AIPCC Productization may open a broad release-branch MR that updates the
`.tekton/` PipelineRuns for every component at once. That work can include more
than renaming the application and component for the new release: it may also
change the shared pipeline resolver, service accounts, output repositories, and
branch predicates. Treat that MR as the owner of release-wide PipelineRun
wiring.

Before opening or updating a component MR, inspect all open MRs targeting the
release branch and compare changed paths:

```bash
PROJECT='redhat%2Frhel-ai%2Frhaiis%2Fcontainers'
BRANCH='3.6-fast2'

glab api "projects/${PROJECT}/merge_requests?target_branch=${BRANCH}&state=opened&per_page=100" \
  | jq -r '.[] | [.iid, .title, .source_branch, .web_url] | @tsv'

for MR in <AIPCC_MR> <COMPONENT_MR>; do
  glab api "projects/${PROJECT}/merge_requests/${MR}/changes" \
    | jq -r --arg mr "${MR}" '.changes[] | [$mr, .new_path] | @tsv'
done
```

If the AIPCC MR and the vLLM-Omni MR touch either Omni PipelineRun file:

1. Do not merge the Omni MR first.
2. Remove the overlapping `.tekton/` changes from the Omni MR. Keep its diff
   limited to component-owned content such as
   `build-args.vllm-omni/cuda-ubi9.conf` or the Containerfile.
3. Record the dependency in both MRs and the owning Jira issue.
4. Wait for the AIPCC MR to merge, then rebase the Omni MR onto the updated
   release branch.
5. Verify the inherited Omni on-pull and on-push configuration: application,
   component, output image, service account, release-branch predicate, tag
   predicate, and pipeline resolver.
6. Rerun GitLab lint, the Konflux image build, label checks, and the Podman
   integration scenario. A rebase or follow-up push can reset approval, so get
   fresh approval after the final validated diff.

Do not assume that copied PipelineRuns are correct merely because the new
release branch exists. A copied file may still target the prior release in its
CEL predicate, application/component labels, output image, or service account.

### Choosing the release-branch base image

Use the base image already merged for the corresponding component on the live
target branch. Do not select a newer final-looking image merely because
Renovate has proposed it in an open MR.

For the 3.6-fast2 carry-forward, the merged CUDA base update in containers MR
`!1072` selected:

```text
quay.io/aipcc/base-images/cuda-13.0-el9.8:3.6.0-ea.2-1789502554
```

An open Renovate MR (`!1077`) proposed a later `3.6.0-*` image, but it was not
approved or merged. The Omni component therefore followed the merged EA2 CUDA
baseline. Merged CPU (`!1082`) and Gaudi (`!1080`) Fast2 delivery MRs used the
same EA2-family convention.

For an approved exact artifact carry-forward, preserve the existing immutable
wheel release refs. Update only the target release's base image and product
identity unless release policy explicitly requires newly named wheel artifacts.
Do not add a wheel-pipeline MR or mint a source tag simply to make the artifact
look new.

### 3.6-fast2 example

- AIPCC release-wide PipelineRun MR: containers `!1083`
- vLLM-Omni component MR: containers `!1084`
- Overlap found: both initially changed the two Omni `.tekton/` files
- Resolution: `!1084` dropped the redundant PipelineRun edits and retained only
  the Omni build-args carry-forward; it queued behind `!1083` for rebase and
  final validation

This ordering preserves the AIPCC Productization team's shared pipeline changes
without losing the component-specific wheel and base-image decision.

## Konflux Container Workflow

This workflow was exercised end-to-end for `3.6-fast1` by containers MR !1001
and tag `vllm-omni-cuda-v2026090101`.

### 1. Test a container change with an MR

The on-pull PipelineRun is path-scoped. Changing the vllm-omni Containerfile,
its build-args file, or the relevant CI/Tekton configuration triggers Konflux.
The GitLab pipeline and Konflux run are parallel checks:

- GitLab CI runs the central linter.
- Konflux builds both architectures, adds the SBOM and attestations, pushes a
  temporary PR image, and creates a snapshot on success.
- Snapshot integration scenarios report back as additional commit statuses.

Find the current MR commit and all of its statuses:

```bash
PROJECT='redhat%2Frhel-ai%2Frhaiis%2Fcontainers'
MR=1001
SHA=$(glab api "projects/${PROJECT}/merge_requests/${MR}" | jq -r '.sha')

glab api "projects/${PROJECT}/repository/commits/${SHA}/statuses?per_page=100" \
  | jq -r '.[] | [.name, .status, .target_url, .description] | @tsv'
```

The `Konflux Production Internal / ...-on-pull-request` status links directly to
the PipelineRun. A PR build publishes a temporary SHA-scoped image under
`quay.io/redhat-user-workloads/`; for example:

```
quay.io/redhat-user-workloads/ai-tenant/<application>/<component>:on-pr-<sha>-linux-x86-64
```

Do not treat the temporary image as the release artifact.

### 2. Read build logs

The simplest path is the Konflux link in the GitLab commit status. Select the
failed task and container in the UI to read its log.

The actual PipelineRun, TaskRun, and pod resources live in
`ai-tenant-build`, even though the UI URL uses `ns/ai-tenant`. With RBAC:

```bash
oc get pipelineruns -n ai-tenant-build | rg 'vllm-omni'
oc get taskruns -n ai-tenant-build \
  -l tekton.dev/pipelineRun=<PIPELINERUN_NAME>
oc get pods -n ai-tenant-build \
  -l tekton.dev/pipelineRun=<PIPELINERUN_NAME>
oc logs -n ai-tenant-build <POD_NAME> --all-containers
```

Access to the `ai-tenant` project does not imply read access to
`ai-tenant-build`. If these commands are forbidden, use the Konflux UI or ask
the AIPCC/Konflux team for build-namespace access. This is still a
self-sufficiency gap for Midstream.

When debugging, identify the failing phase before changing the Containerfile.
For example, an RPM/library scan failure is part of the image content, while a
failure during SBOM or attestation handling is post-build infrastructure.

### 3. Check snapshots and integration tests

After the image build succeeds, Konflux creates a snapshot. Its name and the
integration result appear in the GitLab status description. For vllm-omni the
current scenarios are:

- `rhaiis-3-6-fast1-check-labels-tech-preview`
- `rhaiis-3-6-fast1-test-vllm-omni-podman-cuda-x86-64`

The same commit-status command above is the quickest way to check whether each
scenario is pending, running, successful, or failed and to get its PipelineRun
URL. The Podman check is a basic container test; broader model/runtime coverage
belongs in ITS.

### 4. Create a release image

Merge the tested change into the intended release branch first. Do not tag
`main` when the requested release is on `3.6-fast1`.

Use a component-scoped tag so unrelated image pipelines do not fan out. The
vllm-omni CUDA on-push trigger accepts:

- `vllm-omni-cuda-v*` (preferred)
- `vllm-omni-v*`
- broad `v*` tags

The established date/counter format is `vllm-omni-cuda-vYYYYMMDDNN`. Resolve
and verify the release-branch tip, then point the tag at that exact commit:

```bash
PROJECT='redhat%2Frhel-ai%2Frhaiis%2Fcontainers'
BRANCH='3.6-fast1'
TAG='vllm-omni-cuda-v2026090101'
SHA=$(glab api "projects/${PROJECT}/repository/branches/${BRANCH}" \
  | jq -r '.commit.id')

# Review the commit before creating the protected tag.
glab api "projects/${PROJECT}/repository/commits/${SHA}" \
  | jq '{id, title, committed_date}'

glab api -X POST "projects/${PROJECT}/repository/tags" \
  -f tag_name="${TAG}" \
  -f ref="${SHA}" \
  -f message='[RHAIIS 3.6-fast1] vLLM-Omni CUDA release image build'
```

These tags are protected; the caller needs Maintainer-level tag permission.
The tag triggers the `...-on-push` PipelineRun. That run publishes the
tagged/multi-architecture Konflux image and produces the release snapshot used
by the component release and ITS workflows.

Monitor the release commit with the same status query, filtering for
`vllm-omni-cuda-ubi9-3-6-fast1-on-push`. A successful on-push status confirms
the image build and push. Runtime validation and ITS remain separate gates.

### 5. Find the published AIPCC image

The on-push PipelineRun publishes an internal Konflux image and creates a
Snapshot. A successful `Release` for that Snapshot contains the final
`quay.io/aipcc/` image names. The GitLab commit-status description provides the
Snapshot name; use it to find the corresponding Release:

```bash
SNAPSHOT='rhaiis-3-6-fast1-20260902-144634-000-3h'

RELEASE=$(oc get releases -n ai-tenant -o json \
  | jq -r --arg snapshot "${SNAPSHOT}" \
    '.items[] | select(.spec.snapshot == $snapshot) | .metadata.name')

echo "${RELEASE}"

# Confirm promotion and show the managed/final release PipelineRuns.
oc get release "${RELEASE}" -n ai-tenant -o json \
  | jq '{
      conditions: .status.conditions,
      managed_pipeline: .status.managedProcessing.pipelineRun,
      final_pipeline: .status.finalProcessing.pipelineRun
    }'

# List the published vLLM-Omni image URLs and digest.
oc get release "${RELEASE}" -n ai-tenant -o json \
  | jq -r '.status.artifacts.images[]
    | select(.name | contains("vllm-omni"))
    | "digest: \(.shasum)", .urls[]'
```

The URL list contains several related artifacts:

- `:<version>-<timestamp>` is the immutable runtime image to use for validation
  and the Productization handoff.
- `:<version>` is a moving release alias.
- Tags ending in `-source` and `.src` are source artifacts, not runtime images.

Do not infer the AIPCC tag from the Snapshot timestamp. Read it from
`.status.artifacts.images[].urls`; the release service generates the final tag.

### 6. Proven corrected `3.6-fast1` example

- Containers MR: `redhat/rhel-ai/rhaiis/containers!1008`
- Merge commit: `f137c63a23265127f4b76c66acd500e3acb1af6e`
- Release tag: `vllm-omni-cuda-v2026090201`
- On-push PipelineRun: `rhaiis-vllm-omni-cuda-ubi9-3-6-fast1-on-push-hmwmt`
- Snapshot: `rhaiis-3-6-fast1-20260902-144634-000-3h`
- Release: `rhaiis-3-6-fast1-20260902-144634-000-3h-f137c63-n8zv2`
- Immutable image: `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1788360443`
- Digest: `sha256:39dd7f54f3d3dd81f1ddd7a8399ad997c3dabf1c83f162776d468c224bc0e67d`
- Outcome: managed and final release pipelines succeeded; formal OCP model
  validation was then dispatched against the immutable AIPCC image.

## Pipeline Repo: Wheel Builds

The pipeline repo builds wheels using fromager. Jobs follow this sequence:

```
bootstrap-and-onboard -> build-wheels -> publish-wheels -> release-tarball
```

- `bootstrap-and-onboard`: resolves deps, takes ~1 hour
- `build-wheels`: compiles everything, takes 4-6+ hours
- `publish-wheels`: uploads to GitLab PyPI registry
- `release-tarball`: creates the release tag (this is what we reference in `WHEEL_RELEASE_*`)

### Checking wheel build status

```bash
# List jobs, filter for omni
glab api "projects/redhat%2Frhel-ai%2Frhaiis%2Fpipeline/pipelines/<ID>/jobs?per_page=100&scope=running" \
  | python3 -c "import json,sys; jobs=json.load(sys.stdin); [print(f\"{j['status']:12s} {j['name']}\") for j in sorted(jobs, key=lambda x: x['name']) if 'omni' in j['name'].lower()]"
```

### Known issues

- **GitLab upload timeouts**: `flashinfer-jit-cache` is a large wheel that can timeout during upload. Retry usually works.
- **Pipeline cancellation on merge**: merging any MR to main cancels the running pipeline and starts a new one. Coordinate with the team to avoid interrupting long builds.
- Wheel cache is preserved across retries -- previously built wheels don't need rebuilding.

## Infrastructure Repo: Creating Indexes

Before the pipeline can build wheels for a new collection, the GitLab registry indexes must exist. These are defined in `data/products/rhaiis.yml` in the infrastructure repo.

```yaml
# Example: adding vllm-omni to a version
- name: vllm-omni
  variants:
    - name: cuda13.0-ubi9
      arch: x86_64
    - name: cuda13.0-ubi9
      arch: aarch64
```

Without these indexes, the pipeline hits a 404 on `get-project-id`.

## Cutting a Midstream Tag (Tag-to-Wheels Flow)

A midstream tag on the enterprise fork signals "this code is validated and ready to ship." Pushing it kicks off an automated chain that ends with container images.

### The chain

```
GitHub tag pushed (e.g. v0.26.0+rhaiv.0 on nm-vllm-omni-ent)
  → GitLab mirror syncs tag (usually within 30-60 min)
    → Renovate detects new tag, opens MR on rhaiis/pipeline to bump requirements.txt
      → Pipeline MR merged → wheel collection built + released
        → Renovate detects new wheel release, opens MR on rhaiis/containers to bump build-args
          → Containers MR merged into the release branch
            → component-scoped containers tag pushed → Konflux builds the release image
```

### How to cut the tag

```bash
# 1. Verify main HEAD is what you want to tag
gh api repos/neuralmagic/nm-vllm-omni-ent/commits?sha=main\&per_page=3 \
  --jq '.[] | "\(.sha[:8])  \(.commit.message | split("\n")[0])"'

# 2. Get the full SHA
FULL_SHA=$(gh api repos/neuralmagic/nm-vllm-omni-ent/commits?sha=main\&per_page=1 --jq '.[0].sha')

# 3. Create the tag
gh api repos/neuralmagic/nm-vllm-omni-ent/git/refs \
  -X POST \
  -f ref="refs/tags/v0.26.0+rhaiv.0" \
  -f sha="$FULL_SHA"
```

For nm-vllm-ent (standard vLLM), same pattern but different repo:
```bash
gh api repos/neuralmagic/nm-vllm-ent/git/refs \
  -X POST \
  -f ref="refs/tags/v0.26.0+rhaiv.0" \
  -f sha="$FULL_SHA"
```

### Monitoring propagation

After pushing the tag, check each step of the chain:

```bash
# Step 1: Has the GitLab mirror picked up the tag?
glab api "projects/redhat%2Frhel-ai%2Fcore%2Fmirrors%2Fgithub%2Fneuralmagic%2Fnm-vllm-omni-ent/repository/tags?search=v0.26" \
  | python3 -c "import json,sys; tags=json.load(sys.stdin); [print(t['name']) for t in tags]"

# Step 2: Has Renovate opened a pipeline MR?
glab mr list -R redhat/rhel-ai/rhaiis/pipeline | grep -i omni

# Step 3: Once pipeline MR is merged, check for new wheel releases
glab api "projects/redhat%2Frhel-ai%2Frhaiis%2Fpipeline/releases?per_page=10" \
  | python3 -c "import json,sys; [print(r['tag_name'],'—',r['created_at'][:10]) for r in json.load(sys.stdin) if 'omni' in r['tag_name']]"

# Step 4: Has Renovate opened a containers MR to bump build-args?
glab mr list -R redhat/rhel-ai/rhaiis/containers | grep -i omni
```

Mirror sync typically takes 30-60 minutes. Renovate runs on a schedule (not instant) — could be a few hours before it opens MRs.

### What Renovate watches (RHAIIS pipeline repo)

The `renovate.json` in `rhaiis/pipeline` has two custom regex managers relevant to omni:

| Manager | Watches | Updates |
|---------|---------|---------|
| "Update vLLM versions for CUDA collection" | `nm-vllm-ent` GitLab tags | `vllm==` in `collections/rhaiis/cuda13.0-ubi9/requirements.txt` AND `collections/vllm-omni/cuda13.0-ubi9/requirements.txt` |
| "Update vllm-omni versions for CUDA 13 vllm-omni collection" | `nm-vllm-omni-ent` GitLab tags | `vllm-omni==` in `collections/vllm-omni/cuda13.0-ubi9/requirements.txt` |

The `3.5` and `main` branches have **no `allowedVersions` constraint** — Renovate will bump to the latest tag. EA branches (e.g. `3.5-EA2`) have per-variant version pinning (e.g. cuda locked to `0.21.x`).

**Gotcha:** The vllm-updates MR on `3.5`/`main` bumps ALL variants (cuda, neuron, tpu, spyre, rocm) to the same tag. Neuron/TPU/spyre are intentionally on older versions. These MRs need manual review — don't blindly merge.

### Two repos, two tags

The vllm-omni collection requires paired versions of both packages:

```
vllm-omni==0.26.0+rhaiv.0    ← tag on nm-vllm-omni-ent
vllm==0.26.0+rhaiv.0          ← tag on nm-vllm-ent
```

Renovate tracks them independently via separate custom managers. You may need to coordinate — if only one tag exists, Renovate will open an MR that bumps one but not the other.

## Renovate Config

The `renovate.json` in the containers repo auto-updates wheel release versions. For a new collection, add three entries per arch:

1. `WHEEL_RELEASE_AARCH64` regex matcher pointing at `build-args.vllm-omni/cuda-ubi9.conf`
2. `WHEEL_RELEASE_X86_64` regex matcher
3. `RHAIIS_VERSION` extractor

Also add the conf file to the `cuda` packageRules group so updates are grouped together.

## Cherry-Picking from Forks

The AIPCC CI bot can't access forks. If someone submits an MR from a fork:

```bash
# Fetch the MR head ref
git fetch origin refs/merge-requests/<MR_NUMBER>/head:refs/remotes/origin/mr/<MR_NUMBER>

# Cherry-pick preserving authorship
git cherry-pick <SHA>

# Amend to add AIPCC compliance (JIRA prefix + Signed-off-by)
git commit --amend --author="Original Author <email>" -m "INFERENG-XXXX: ..."
```

## Useful glab Commands

```bash
# View MR
glab mr view <NUMBER> --repo redhat/rhel-ai/rhaiis/containers

# Get MR diff
glab mr diff <NUMBER> --repo redhat/rhel-ai/rhaiis/containers

# Create MR
glab mr create --repo redhat/rhel-ai/rhaiis/containers \
  --title "INFERENG-XXXX: Title" \
  --description "..." \
  --source-branch my-branch --target-branch main

# Update MR title/description
glab mr update <NUMBER> --repo redhat/rhel-ai/rhaiis/containers \
  --title "New title" --description "New desc"

# Approve MR
glab mr approve <NUMBER> --repo redhat/rhel-ai/rhaiis/containers

# Comment on MR
glab mr comment <NUMBER> --repo redhat/rhel-ai/rhaiis/containers -m "comment text"

# Check pipeline releases
glab api "projects/redhat%2Frhel-ai%2Frhaiis%2Fpipeline/releases/<TAG>"
```

## Team / Ownership

- `/pipeline` and `/containers` repos are **Midstream domain** (our team)
- AIPCC index/builder repos are **AIPCC's domain**
- OCTO contributions welcome but we own landing changes
- Andre Lustosa is a key contact for AIPCC infra/indexes
- Tarun (takumar1) is an SME for container review
