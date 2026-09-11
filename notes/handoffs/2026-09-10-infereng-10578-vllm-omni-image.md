# INFERENG-10578 vLLM-Omni image handoff

## Completed

- Midstream tag: `v0.26.0+rhaiv.7` from `neuralmagic/nm-vllm-omni-ent#54`.
- AIPCC wheel releases: both architectures at `.3997`.
- Containers MR `redhat/rhel-ai/rhaiis/containers!1033` merged into `3.6-fast1`.
- Exact merge commit: `0a7cf4edbfbcf7fc1867f8b31df6b1b38c921b66`.
- Protected component tag: `vllm-omni-cuda-v2026091001`; verified at the exact merge commit.
- INFERENG-10578 has comments covering the `.7` tag, pipeline MR, wheel pins, containers MR, merge, and component tag.

## State at end of day

- Konflux on-push PipelineRun:
  `rhaiis-vllm-omni-cuda-ubi9-3-6-fast1-on-push-vhsqf`
- GitLab commit status was still `running` at the final check on 2026-09-10.
- No Snapshot or successful Release had been observed yet.
- The 30-minute monitor automation was explicitly removed at quitting time. Do not assume monitoring is still active.

## Continue in the morning

1. Read `AGENTS.md`, run `bin/status`, and read `runbooks/aipcc/vllm-omni.md`.
2. Verify `vllm-omni-cuda-v2026091001` still resolves to
   `0a7cf4edbfbcf7fc1867f8b31df6b1b38c921b66`.
3. Query commit statuses for that SHA and inspect the vLLM-Omni `3-6-fast1`
   on-push status/PipelineRun.
4. Once successful, obtain the Snapshot name from live status or Konflux evidence.
5. In `ai-tenant`, find the `Release` whose `spec.snapshot` matches. Require both
   managed and final release processing to be successful.
6. From the successful Release, capture every vLLM-Omni runtime image URL/alias
   and digest. Exclude `-source` and `.src` artifacts; do not infer image tags.
7. Before commenting, read `skills/jira-board/SKILL.md` and its linked procedure.
   Add one non-duplicate INFERENG-10578 comment containing the component tag,
   PipelineRun, Snapshot, Release, runtime image URLs/aliases, and digest.

Do not deploy, start model validation, merge another MR, or create another tag
as part of this handoff.

## Completed 2026-09-11

- RDU VPN and a refreshed kubeconfig provided access to `ai-tenant` as `dosmith`.
- On-push build and Release succeeded.
- Release: `rhaiis-3-6-fast1-20260910-200419-000-6k-0a7cf4e-cp48z`.
- Immutable runtime image: `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1789070722`.
- Digest: `sha256:4d17c21a588539f6429fd1d24e81b8a3f36448f2f4e21f1f241089dfa5568a42`.
- Final release evidence was added to INFERENG-10578.
