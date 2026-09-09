# AIPCC and downstream product integration

AIPCC owns the builder, pipeline, and base-image infrastructure downstream of
midstream artifacts. Midstream supplies vLLM/vLLM-Omni artifacts and validation;
the product pipeline assembles and promotes the resulting images.

## Repositories

- `repos/aipcc/pipeline`: RHAIIS pipeline definitions
- `repos/aipcc/containers`: product container definitions

The detailed vLLM-Omni architecture and current-state material remains under
`projects/vllm-omni/` because it is primarily product-delivery context for that
workstream. Use `runbooks/aipcc/vllm-omni.md` for the repeatable GitLab and
Konflux workflow.

Pipeline branches, build collections, image tags, and promotion state are live
system state. Verify them before changing Jira lifecycle or reporting a release
outcome.
