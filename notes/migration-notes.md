# Initial migration notes

The first workspace migration was curated from the former `worktree-nm` and
`worktree-omni` meta-workspaces. Repository names and URLs were preserved in the
manifest, while workspace-owned material was organized around vLLM,
vLLM-Omni, AIPCC, documentation, infrastructure, and tools.

## Deliberately not migrated

- the old `secrets` file and repository `.env` files
- `.claude` state, caches, virtual environments, logs, and generated artifacts
- `full_ci_sig.md`, `Notes - vLLM CI SIG.md`, and `tmp.txt`, which are overlapping
  raw meeting-note exports rather than maintained workspace context
- generated Krea images
- `vllm-dsv4-official`, an unversioned copied source tree
- duplicate source checkouts whose purpose was only a transient branch
- absolute symlinks into the external notes repository

Historical documents retained in `archive/` are evidence and context, not
current instructions. Current procedures should live in `skills/` or
`runbooks/` and be verified against authoritative repositories before use.
