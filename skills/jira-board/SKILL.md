---
name: jira-board
description: "Inspect and manage Doug's INFERENG Midstream Jira board when board filtering, issue creation, comments, or lifecycle transitions are requested. Also covers SME label triage for the ms-* workstreams (ms-upstream, ms-omni, and the rest) — SME Backlog Prep, label cleanup, staffing queues, and epic categorization for the Midstream team's upstream and omni work."
---

# Doug's Jira Board

Read [the detailed procedure](references/procedure.md) completely before carrying
out this workflow. It preserves the operational context, commands, and examples
for Doug's board, built on the Atlassian CLI (`acli`).

The generic `acli` command reference is the team skill
`~/.openclaw/workspace/skills/atlassian-jira/` (auto-updated weekly from
`rhaiis-midstream-snippets`; do not edit it locally). This skill is the
INFERENG Midstream board overlay on top of it: component filters, status
names, custom fields, and gotchas specific to Doug's board.

Verify current repositories, branches, remote state, CLI behavior, tickets, and
services before acting. Treat pinned versions, identifiers, hosts, and historical
results as examples unless live evidence confirms them. Do not report completion
from setup, dispatch, or login alone; verify the requested outcome.
