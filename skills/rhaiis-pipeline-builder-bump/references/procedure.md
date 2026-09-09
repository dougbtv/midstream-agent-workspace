---
name: rhaiis-pipeline-builder-bump
description: Bump the fromager builder version in the RHAIIS pipeline repo. Use when the user mentions bumping the builder, a new builder release, updating builder version, or needs to pick up an AIPCC builder fix in the RHAIIS wheel pipeline. Also trigger when the user references builder-image-version.yml, fromager builder tags (v28.x, v29.x), or mentions AIPCC fixes that need to flow into RHAIIS wheel collections. The pipeline repo is redhat/rhel-ai/rhaiis/pipeline on GitLab.
---

# RHAIIS Pipeline Builder Bump

Bump the fromager builder version in the RHAIIS pipeline repo (`redhat/rhel-ai/rhaiis/pipeline`) to pick up fixes from the AIPCC builder team.

## When this is needed

The AIPCC team maintains the fromager builder (`redhat/rhel-ai/wheels/builder`). When they release a new builder version (e.g., fixing wheel packaging, adding arch targets, updating Rust/toolchain), the RHAIIS pipeline repo needs to be updated to consume it. Renovate handles this automatically for `main`, but **not for release branches** (e.g., `3.4`). Release branch bumps require a manual MR.

## Step 1: Identify the target builder version

Confirm with the user which branch they're targeting and which builder version line to use before proceeding.

Check the latest builder releases to find the right version:

```bash
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Fwheels%2Fbuilder/repository/tags?per_page=10&order_by=version&sort=desc" | \
  jq '.[] | {name, created: .commit.created_at}'
```

If the user mentioned a specific builder MR or AIPCC ticket, verify it's included in the target version:

```bash
glab api --hostname gitlab.com \
  "projects/redhat%2Frhel-ai%2Fwheels%2Fbuilder/merge_requests?state=merged&order_by=updated_at&sort=desc&per_page=10" | \
  jq '.[] | {iid, title, merged_at, target_branch}'
```

## Step 2: Check the current builder version

```bash
# Check what's currently pinned on the target branch
git fetch origin <TARGET_BRANCH> --quiet
git show origin/<TARGET_BRANCH>:builder-image-version.yml
git show origin/<TARGET_BRANCH>:.gitlab-ci.yml | grep -n "ref: v28\|ref: v29"
```

There are two files that reference the builder version:

| File | What it controls |
|------|-----------------|
| `builder-image-version.yml` | `BUILDER_IMAGE_VERSION` variable used by CI trigger rules |
| `.gitlab-ci.yml` | `ref:` on every `include:` block that pulls from `redhat/rhel-ai/wheels/builder` (typically 14 occurrences) |

Both must be updated to the same version.

## Step 3: Create the branch and make changes

```bash
git checkout -b bump-builder-v<NEW_VERSION> origin/<TARGET_BRANCH>
```

Update both files. Use `replace_all` to hit all 14 `ref:` lines in `.gitlab-ci.yml`:

- `.gitlab-ci.yml` — change every `ref: v<OLD>` to `ref: v<NEW>`
- `builder-image-version.yml` — change `BUILDER_IMAGE_VERSION: v<OLD>` to `BUILDER_IMAGE_VERSION: v<NEW>`

Verify the diff looks clean:

```bash
git diff --stat
git show origin/<TARGET_BRANCH>:.gitlab-ci.yml | grep "ref: v" | wc -l  # should match number of changes
```

## Step 4: Commit and push

Use the INFERENG ticket key if one exists. Include the key fix in the commit message so reviewers know why.

```bash
git add .gitlab-ci.yml builder-image-version.yml
git commit -m "<TICKET>: bump builder to v<NEW> (<short reason>)

<One-line explanation of the key fix, e.g., 'Builder vX includes AIPCC-YYYYY which fixes Z'>

Signed-off-by: <USER_NAME> <USER_EMAIL>"

git push -u origin bump-builder-v<NEW_VERSION>
```

## Step 5: Create the MR

```bash
glab mr create \
  --source-branch bump-builder-v<NEW_VERSION> \
  --target-branch <TARGET_BRANCH> \
  --title "<TICKET>: bump builder to v<NEW> (<short reason>)" \
  --description "$(cat /tmp/mr-desc.md)" \
  --assignee <USER> \
  -R redhat/rhel-ai/rhaiis/pipeline
```

MR description template:

```markdown
## Summary

Bumps the fromager builder from `v<OLD>` to `v<NEW>` on the <TARGET_BRANCH> branch.

Key fix: **AIPCC-XXXXX** — <one paragraph explaining what the builder fix does and why we need it>

### Other fixes in v<NEW>

- AIPCC-XXXXX: <title>
- AIPCC-XXXXX: <title>
- ...

cc @<relevant reviewers>

Signed-off-by: <USER_NAME> <USER_EMAIL>
```

To populate the "other fixes" list, check the builder release announcement in Slack or query merged MRs between the old and new tags.

## What happens after merge

Once the MR merges, the pipeline kicks off automatically:

1. **Wheel collections rebuild** — CI builds wheel collections for each accelerator (CUDA, ROCm, CPU, Spyre, etc.) using the new builder. This takes 2-3+ hours (torch builds from source).
2. **Renovate updates containers** — Renovate detects new wheel collection versions and creates an MR in `redhat/rhel-ai/rhaiis/containers` targeting the same release branch.
3. **Containers MR merges** — after CI passes, merge the Renovate MR (or it auto-merges if approved).
4. **Create accelerator tags** — tag the containers branch to trigger RC image builds (see the RC image release runbook for tag commands).
5. **RC images land on quay.io** — Tekton builds push images to `quay.io/aipcc/rhaiis/{cuda,rocm,cpu}-ubi9`.

## Key repos and paths

| Repo | GitLab API path (URL-encoded) | Purpose |
|------|-------------------------------|---------|
| `rhaiis/pipeline` | `redhat%2Frhel-ai%2Frhaiis%2Fpipeline` | Wheel collection builds |
| `rhaiis/containers` | `redhat%2Frhel-ai%2Frhaiis%2Fcontainers` | Container image builds |
| `wheels/builder` | `redhat%2Frhel-ai%2Fwheels%2Fbuilder` | The fromager builder itself |

## Past examples

| MR | Version bump | Reason | Branch |
|----|-------------|--------|--------|
| [!553](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/553) | v28.22.0 → v28.26.0 | pyarrow 24.0.0 hermetic build fix (xsimd) | 3.4 |
| [!560](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/560) | v28.26.0 → v28.26.2 | torch-sendnn constraints + builder upgrade | 3.4 |
| [!567](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/567) | v28.26.2 → v28.27.0 | aotriton ROCm kernel images fix (AIPCC-13815) | 3.4 |
