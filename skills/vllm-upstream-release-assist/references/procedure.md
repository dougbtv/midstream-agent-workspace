---
name: upstream-release-assist
description: Assist with validating an upstream vLLM release from Red Hat's perspective. Use this skill when the user mentions upstream release validation, release branch triage, cherry-pick validation, accept-sync for a release, or mock downstream builds for a vLLM release. This covers both upstream CI triage and mock downstream builds through nm-cicd/nm-vllm-ent. Use it even if the user just says "help with the release" or "validate the release" — this is the process.
---

# Upstream Release Assist

End-to-end process for validating upstream vLLM releases from Red Hat's perspective. This skill coordinates two parallel workstreams that feed findings back to upstream and produce early-signal branches reusable for the official midstream sync.

## Goal

Validate upstream vLLM release branches so Red Hat can:
1. Help upstream ship with confidence (report regressions, recommend cherry-picks)
2. Get early signal on whether the release merges cleanly into nm-vllm-ent
3. Produce nm-cicd and nm-vllm-ent branches that capture any CI config changes needed for the new release — reusable when the team does the official sync

## Two-Prong Approach

### Prong 1 — Upstream CI Triage

Run the upstream Buildkite CI suite against the release branch, triage failures, and separate real regressions from noise.

**What the human provides:**
- The release branch name (e.g. `releases/v0.20.0`)
- Any context from upstream Slack or the release manager

**What the agent does:**

1. **Research the release** — check the GitHub milestone for cherry-pick requests, read the release branch commit log, review CI SIG notes for validation expectations
2. **Study prior art** — look at how the previous release was validated (which pipelines, how many iterations, what was shipped with known failures)
3. **Trigger CI** on the release branch via Buildkite MCP or CLI:
   ```
   mcp__buildkite__create_build(
     org_slug="vllm",
     pipeline_slug="ci",
     branch="releases/vX.Y.Z",
     commit="<full 40-char SHA>",
     message="vX.Y.Z release validation - Red Hat",
     environment=[
       {"key": "RUN_ALL", "value": "1"},
       {"key": "NIGHTLY", "value": "1"},
       {"key": "CONTINUE_ON_FAILURE", "value": "1"},
       {"key": "PRIORITY", "value": "HIGH"}
     ],
     ignore_branch_filters=true
   )
   ```
4. **Triage failures** when the build completes:
   - Separate hard failures from soft failures
   - Compare against recent main nightly to identify release-specific regressions vs pre-existing issues
   - Categorize: infra (agent lost), known flaky (check upstream issues), platform-specific, expected (backward compat), or real regression
5. **Report findings** — post triage to the JIRA story and upstream Slack thread
6. **Iterate if needed** — if real regressions are found:
   - Create a fork branch off the release branch on dougbtv/vllm
   - Cherry-pick fixes from main or milestone PRs
   - Open a draft PR targeting the release branch
   - Trigger CI against the draft PR to validate
   - Mark ready for review when CI is acceptable, ping the release manager

### Prong 2 — Mock Downstream Build (Accept-Sync)

Merge the release branch into nm-vllm-ent, run it through our accept-sync pipeline, and validate that it builds, serves models, and passes accuracy/perf checks.

**What the human provides:**
- The release branch name
- Target device (default: cuda)
- Any specific models or test scenarios to prioritize

**What the agent does:**

1. **Create branches** on both repos:
   ```bash
   # nm-vllm-ent: merge the release branch
   git fetch upstream releases/vX.Y.Z
   git checkout -b doug/vX.Y.Z-release-validation origin/main
   git merge -X theirs upstream/releases/vX.Y.Z --no-edit
   git rm -rf .github/   # resolve modify/delete conflicts
   git push -u origin doug/vX.Y.Z-release-validation

   # nm-cicd: create matching branch for CI config iteration
   git checkout -b doug/vX.Y.Z-release-validation origin/main
   git push -u origin doug/vX.Y.Z-release-validation
   ```

2. **Trigger accept-sync** (CUDA):
   ```bash
   gh workflow run accept-sync.yml \
     --repo neuralmagic/nm-cicd \
     --ref doug/vX.Y.Z-release-validation \
     -f wf_category=DEBUG \
     -f repo=neuralmagic/nm-vllm-ent \
     -f branch=doug/vX.Y.Z-release-validation \
     -f python=3.12 \
     -f target_device=cuda
   ```

3. **Monitor and iterate** — accept-sync runs: BUILD (wheel) -> IMAGES + TEST + LM-EVAL + GUIDELLM in parallel. Common issues on a new upstream release:
   - **Requirements paths changed** — upstream reorganizes `requirements/` between major releases. Fix in nm-cicd build scripts with fallbacks that handle both old and new layouts.
   - **Dependency version bumps** — upstream may bump pinned deps (e.g. flashinfer, deepgemm). Update pins in `neuralmagic/requirements/cuda.txt` on the nm-cicd branch.
   - **Dockerfile changes** — upstream may change base images, build args, or install steps. Fix in nm-vllm-ent if it's a Dockerfile issue, nm-cicd if it's a build script issue.
   - **Deeper code issues** — if the failure is in vLLM source code (not build/CI config), stop and discuss with the human before making changes. These may need upstream patches.

4. **Fix and re-trigger** — push fixes to the appropriate branch and re-dispatch. Expect 2-4 iterations for a major release. Each iteration:
   - Investigate failures (get job logs via `gh api`)
   - Determine if it's nm-cicd config (fix freely) or nm-vllm-ent code (discuss first)
   - Commit fix, push, re-trigger accept-sync
   - Track progress in JIRA comments

5. **Report results** — when the run converges:
   - Which jobs passed/failed
   - Whether failures are release-specific or pre-existing
   - Any nm-cicd changes needed (these become the "early signal" for the official sync)

## Iteration Strategy

Set up a monitoring cron (every 20 min) to track the accept-sync run. On each check:
- If jobs are still running: report brief status
- If a job failed: investigate immediately, fix if it's CI config, report if it's deeper
- If all jobs are done: report full summary and clean up the cron

## Deliverables

By the end of this process, the team should have:

1. **Upstream CI triage report** — categorized failures, identified regressions, cherry-pick recommendations
2. **Accept-sync results** — BUILD, IMAGES, TEST, LM-EVAL, GUIDELLM pass/fail for the release
3. **Early-signal branches**:
   - `nm-cicd: doug/vX.Y.Z-release-validation` — CI config changes needed for the new release (requirements paths, dependency pins, build script fixes)
   - `nm-vllm-ent: doug/vX.Y.Z-release-validation` — clean merge of the release branch, ready to reference when the official sync happens
4. **JIRA story** with full audit trail of iterations, fixes, and findings
5. **Upstream feedback** — findings posted to the release Slack thread, cherry-pick PRs if needed

## Key Tooling

| Tool | Purpose |
|------|---------|
| Buildkite MCP / `bk` CLI | Trigger and triage upstream CI builds |
| `gh` CLI | Query PRs/issues, dispatch GH Actions workflows, inspect job logs |
| `jira` CLI | Track progress, post updates (use temp files for comments, markdown formatting) |
| `git` | Merge release branches, cherry-pick fixes |
| CI Watch MCP | Analyze upstream CI health, test failure history |

## What NOT to Do

- Don't push to `upstream/releases/*` directly — only maintainers can. Use a fork branch + draft PR.
- Don't make deep vLLM code changes on nm-vllm-ent without discussing upstream patches first.
- Don't use `branch=main` when triggering Buildkite builds — it clutters the main view and can't be deleted.
- Don't skip investigating failures — even if a fix seems obvious, check the logs to confirm the root cause.

## Related Skills

- `midstream-build-from-upstream` — the underlying build/deploy/smoke-test mechanics for nm-vllm-ent
- `Upstream_release_validation.md` — detailed notes and build log for a specific release (v0.20.0)
