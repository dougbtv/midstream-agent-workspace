---
name: vllm-midstream-zero-day
description: "Prepare a vLLM midstream zero-day preview for a newly released model, from intake and builds through validation and handoff."
---

# vLLM Midstream Zero-Day

Read [the detailed procedure](references/procedure.md) completely before carrying
out this workflow. It preserves the operational context, commands, and examples
migrated from the former workspace.

For the merge, build, CI-monitoring, and smoke-test loop, use
`vllm-midstream-build-from-upstream` as the canonical companion procedure.

Verify current repositories, branches, remote state, CLI behavior, tickets, and
services before acting. Treat pinned versions, identifiers, hosts, and historical
results as examples unless live evidence confirms them. Do not report completion
from setup, dispatch, or login alone; verify the requested outcome.
