# Midstream agent workspace

This repository is the durable control plane for Doug's vLLM and vLLM-Omni
engineering work. It owns instructions, project context, skills, runbooks, and
notes. It does **not** own the application source repositories under `repos/`.

## Start here

1. Run `bin/status` before modifying source repositories.
2. Read the relevant project context under `projects/`.
3. Load a relevant procedure from `skills/` or `runbooks/` when one exists.
4. Enter the exact repository under `repos/` before running Git commands.

The two primary workstreams are:

- `projects/vllm/` and `repos/vllm/`
- `projects/vllm-omni/` and `repos/vllm-omni/`

Supporting repositories are grouped under `repos/aipcc/`, `repos/docs/`,
`repos/infra/`, and `repos/tools/`. Cross-repository work is normal.

## Ownership boundaries

- This workspace Git history tracks the workspace material outside `repos/`.
- Every directory below `repos/` is an independent Git repository or worktree.
- Literal names such as `nm-cicd` and `nm-vllm-ent` identify real repositories;
  `nm` is not the workspace taxonomy.
- Never move, clean, reset, or rewrite an attached repository merely to make the
  workspace tidy.
- Before rewriting source-repository history, inspect its remotes, branch stack,
  and remote tip, make a local backup, and use explicit `--force-with-lease`.

`git clean -x` and especially `git clean -xfd` are dangerous at the workspace
root: ignored repositories may contain valuable uncommitted work. Do not run
them here.

## Where durable material belongs

- `projects/`: current intent, architecture, state, and decisions for a body of work
- `skills/`: reusable agent procedures; each skill has a discoverable `SKILL.md`
- `runbooks/`: operational procedures and reference material
- `notes/`: useful context without a stronger home
- `archive/`: historical handoffs, completed investigations, and dated snapshots
- `scratch/`: disposable local material; ignored by this repository

Project context should preserve durable intent and verified outcomes, not every
transient command or log. Prefer improving a reusable skill or runbook over
rediscovering the same procedure. Treat imported procedures as starting points:
verify current repository state, CLI behavior, branches, tickets, and services
before acting.

## Repository map

`workspace.toml` is the source of truth for expected attachments. Broadly:

- `vllm`: midstream engine, build automation, upstream CI, evaluation, dashboard,
  model validation, and Gaudi integration
- `vllm-omni`: upstream/fork work, midstream engine, vLLM dependency, build
  automation, CI, and Diffusers reference
- `aipcc`: downstream RHAIIS pipeline and containers
- `docs`: release-engineering references, shared snippets, and Doug's notes
  repository
- `infra`: shared runner/cloud infrastructure and GitHub Actions
- `tools`: supporting agent utilities

Use `bin/init --dry-run` to preview attachment creation. See `README.md` for the
bootstrap model and command details.

## High-value context and procedures

For vLLM work, start with:

- `projects/vllm/README.md`
- `projects/vllm/ci-performance-and-evaluation.md`
- `skills/vllm-midstream-sync/SKILL.md`
- `skills/vllm-upstream-pr/SKILL.md`

For vLLM-Omni and its downstream image chain, start with:

- `projects/vllm-omni/README.md`
- `projects/vllm-omni/field-guide.md`
- `projects/vllm-omni/tech-preview.md`
- `projects/aipcc/README.md`
- `runbooks/vllm-omni/build-from-upstream.md`
- `runbooks/aipcc/vllm-omni.md`

For shared operations, use:

- `skills/jira-board/SKILL.md`
- `skills/github-runner-operations/SKILL.md`
- `runbooks/infrastructure/amd-accelerator-cloud.md`

Do not load `archive/` wholesale. Search it only when historical evidence or a
past implementation matters to the current task.

## Safety and secrets

Do not commit credentials, tokens, private keys, `.env` files, secret-bearing
logs, or copied source trees. Internal context may be appropriate; authentication
material is not. Large generated output and raw meeting/log dumps should remain
local unless distilled into useful durable knowledge.
