# vLLM-Omni midstream work

This project area covers upstream and fork contribution, the enterprise
midstream fork, its base-vLLM dependency, build automation, test coverage, Tech
Preview delivery, and the AIPCC product-image path.

## Primary repositories

- `repos/vllm-omni/vllm-omni`: upstream and Doug-fork work
- `repos/vllm-omni/nm-vllm-omni-ent`: midstream vLLM-Omni engine
- `repos/vllm-omni/nm-vllm-ent`: separately branchable base-vLLM dependency
- `repos/vllm-omni/nm-cicd`: separately branchable build automation
- `repos/vllm-omni/ci-infra`: separately branchable upstream CI work
- `repos/vllm-omni/diffusers`: implementation reference

## Context

- `field-guide.md`: architecture and repository relationships
- `tech-preview.md`: active and historical Tech Preview planning state
- `aipcc-integration.md`: productization architecture and delivery state

Operational procedures are in `runbooks/vllm-omni/` and `runbooks/aipcc/`.
Version-specific sync history is archived rather than presented as a current
runbook.
