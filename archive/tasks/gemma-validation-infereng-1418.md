# Agent Task: Gemma Validation (INFERENG-1418)

## Objective

Validate whether the `google/t5gemma-2b-2b-ul2` model deployment failure (INFERENG-1418) still reproduces on the current RHAIIS rhaiv.6 image. This bug was originally filed against vLLM v0.10.0 — it may already be fixed in 0.18.0.

## Background

- **Jira:** https://redhat.atlassian.net/browse/INFERENG-1418
- **Bug:** t5gemma-2b-2b-ul2 falls back to Transformers backend (`T5GemmaForConditionalGeneration has no vLLM implementation`) then crashes during model loading with an EngineCore error.
- **Originally reported against:** vLLM v0.10.0 / RHAIIS 3.2.1
- **We're testing against:** `quay.io/vllm/vllm-cuda:0.18.0_rhaiv.6` (current RC candidate image)
- **Tracking doc:** `./FRIDAY_RC_IMAGE.md` — update Step 0 with results

## What to Do

### 1. Dispatch the test

```bash
gh workflow run ocp-test.yml \
    --repo neuralmagic/nm-cicd \
    --ref main \
    -f config_ref=main \
    -f label=ubuntu-latest \
    -f timeout=60 \
    -f model=google/t5gemma-2b-2b-ul2 \
    -f vllm_image=quay.io/vllm/vllm-cuda:0.18.0_rhaiv.6 \
    -f target_device=cuda \
    -f model_type=language \
    -f ocp_host_url=https://api.modelsibm.ibmmodel.rh-ods.com:6443 \
    -f test_type=smoke
```

### 2. Get the run ID

```bash
# Wait a few seconds after dispatch, then:
gh run list --repo neuralmagic/nm-cicd --workflow ocp-test.yml --limit 5 \
  --json databaseId,displayTitle,status,conclusion,createdAt
```

Look for the most recent run with `t5gemma` or matching timestamp.

### 3. Monitor the run

```bash
gh run watch <RUN_ID> --repo neuralmagic/nm-cicd
```

This will block until completion. The test has a 60-minute timeout.

### 4. Analyze results

**If PASS:**
- The bug is fixed in 0.18.0. Update INFERENG-1418 via the jira CLI:
  ```bash
  jira issue comment add INFERENG-1418 --template /tmp/gemma-comment.md
  ```
  Write a comment to `/tmp/gemma-comment.md` first saying: validated against `quay.io/vllm/vllm-cuda:0.18.0_rhaiv.6` — model deploys and serves successfully. Original bug was against v0.10.0. Recommend closing. Include the GH Actions run link.

**If FAIL:**
- Pull the logs:
  ```bash
  gh run view <RUN_ID> --repo neuralmagic/nm-cicd --log 2>&1 | tail -200
  ```
- Check if the failure is the same `T5GemmaForConditionalGeneration` Transformers fallback crash, or something different.
- Do NOT attempt to fix it. Just document the failure clearly.

### 5. Update the tracking doc

Edit `./FRIDAY_RC_IMAGE.md`, Step 0 section:
- Check the checkbox
- Fill in the result (PASS/FAIL)
- Add the run ID/link
- Note whether this blocks the RC

## Important Notes

- Use the `jira` CLI for Jira operations (MCP is down today). Use temp files for comment bodies.
- Do NOT use `--ref main` with `branch=main` for Buildkite — that's a different system. This is GitHub Actions, `--ref main` is correct here.
- The OCP host URL (`api.modelsibm.ibmmodel.rh-ods.com:6443`) is a shared test cluster — don't worry about it, it's pre-configured with credentials in the repo's GH secrets.
- This test is **not on the critical path** for the RC image release — it's a side validation. The RC pipeline (Steps 1-8) can proceed in parallel.
