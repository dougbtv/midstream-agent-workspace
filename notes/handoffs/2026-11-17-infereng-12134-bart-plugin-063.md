# INFERENG-12134 — 3.5.4 CUDA vllm-bart-plugin 0.6.3 bump

Date: 2026-11-17

## Release impact

The RHAIIS 3.5.4 z-stream GA CUDA image must ship `vllm-bart-plugin` 0.6.3
(management request). The wheel release currently feeding the image ships
**0.6.2**, so a patch is required and is now posted for review.
`vllm-beam-search-plugin` already ships the requested **0.1.5** — no change
needed there.

Tracker: [INFERENG-12134](https://redhat.atlassian.net/browse/INFERENG-12134)
(status: **Review**, assigned Doug, parent epic INFERENG-10471, Related-linked
to the CUDA lane story [INFERENG-10506](https://redhat.atlassian.net/browse/INFERENG-10506)).

## Immutable evidence

- Current image input: `containers/3.5` `build-args/cuda-ubi9.conf` pins wheel
  release `3.5.4781+rhaiis-cuda13.0-ubi9-{x86_64,aarch64}` (pinned 2026-10-30,
  INFERENG-11830).
- Verified contents of that release (GitLab `rhaiis-wheels`, project 68845358,
  package id 74478264, `build-order.json`):
  - `vllm-bart-plugin==0.6.2` (built from AIPCC mirror of `vllm-project/bart-plugin` v0.6.2)
  - `vllm-beam-search-plugin==0.1.5`
- Patch MR: [pipeline !1053](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/1053)
  (`doug/INFERENG-12134-bart-063` → `3.5`, head `08e24d3584`), one-line bump in
  `collections/rhaiis/cuda13.0-ubi9/requirements.txt`. MR pipeline:
  [3079274277](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/pipelines/3079274277)
  (running at handoff time).
- Upstream `vllm-project/bart-plugin` v0.6.3: released 2026-11-09, tag
  `77673541bd7ebdb67a83fa72fd8f72c6aafea17c`. The AIPCC mirror
  `redhat/rhel-ai/core/mirrors/github/vllm-project/bart-plugin` already carries
  the tag at the same commit — **no mirror-sync needed**.

## Upstream v0.6.3 behavior notes

- Packed varlen encoder forward: up to ~1.8x serving throughput under load
  (bart-large + beam-search plugin, TP=1). Outputs identical in fp32;
  fp16/bf16 differ only at rounding level.
- **Behavior change:** encoder inputs longer than `max_position_embeddings`
  (1024 for BART) now return HTTP 400 (`VLLMValidationError`) instead of
  crashing the engine or silently returning garbage at TP>1.

## Remaining chain (in order)

1. Merge pipeline !1053 once its pipeline is green and it has review approval.
2. Build a new CUDA wheel release from the updated collection — both
   `x86_64` and `aarch64` (the containers conf pins both).
3. Update `containers/3.5` `build-args/cuda-ubi9.conf` `WHEEL_RELEASE_X86_64`
   and `WHEEL_RELEASE_AARCH64` to the new release. The conf header says
   Renovate-driven from `main`; a direct bump also has precedent
   (INFERENG-11830 commit `5d7bcd5`).
4. Re-tag the CUDA image with a new immutable tag (never move an existing
   tag) and let Konflux build it.
5. Run the standard z-stream gates: Snyk + `model-validation-ocp` CUDA lane
   validation on the exact rebuilt image.
6. Release notes candidates: the HTTP 400 over-long-input behavior change and
   the fp16/bf16 rounding-level output differences.
7. Record final evidence on INFERENG-12134 and close it (or move it back to
   In Progress while the chain runs — it is in Review now awaiting the MR).

## Timing risk

The 3.5.4 final image is due **2026-11-17** and production push is
**2026-11-18**. The patch itself is trivial; the wheel-release → containers →
image → validation chain is the schedule risk. If the chain cannot complete in
time, the release owner needs a disposition call (ship 0.6.2 + follow-up vs.
slip the image).

## Verification gotchas

- Wheel-release contents: query the GitLab package via
  `glab api "projects/68845358/packages/<id>/package_files"` and fetch
  `build-order.json` through `glab api` (the generic-package download URL
  401s with the token parsed from `~/.config/glab-cli/config.yml`).
- Jira REST needs `bash -c 'source ~/.bashrc; curl ...'` (Gateway env token
  is stale); `acli` is authenticated separately.
- Full audit and working state live in
  [projects/vllm/3.5.4-z-stream.md](../../projects/vllm/3.5.4-z-stream.md)
  ("Confirmed scope item: INFERENG-12134" section).
