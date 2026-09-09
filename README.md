# Midstream agent workspace

This repository is the portable, version-controlled workspace around Doug's
vLLM and vLLM-Omni engineering work. It keeps the knowledge surrounding the
work close to independently managed source repositories without absorbing their
Git histories.

## Layout

```text
.
├── AGENTS.md          Agent orientation and safety rules
├── workspace.toml     Declarative source and attachment inventory
├── bin/               Workspace bootstrap and status commands
├── skills/            Reusable agent procedures
├── projects/          Durable project context and current state
├── runbooks/          Repeatable operational procedures
├── notes/             Reference material and future ideas
├── archive/           Historical material kept out of normal context
├── scratch/           Ignored disposable work
└── repos/             Ignored independent Git repositories/worktrees
```

Repository groups describe the work, not company history: `vllm`,
`vllm-omni`, `aipcc`, `docs`, `infra`, and `tools`. Actual checkout names retain
literal upstream names such as `nm-cicd` where that precision matters.

## Bootstrap

Bootstrap requires Git and Python 3.11 or newer.

On a new machine:

```bash
git clone git@github.com:dougbtv/midstream-agent-workspace.git
cd midstream-agent-workspace
bin/init --dry-run
bin/init
bin/status
```

`bin/init` reads `workspace.toml`, creates shared backing clones below the
ignored `repos/.sources/` directory, and creates detached worktrees at the
declared attachment paths. Detached bootstrap checkouts deliberately avoid
claiming that one branch is canonical across machines. Select or create the
appropriate branch inside each repository afterward.

The command is safe to rerun: valid existing repositories are preserved. It
refuses to overwrite non-repository paths. Pass `--source-root DIR` to reuse
compatible existing backing clones from a known directory instead of cloning
them again:

```bash
bin/init --source-root /home/doug/codebase
```

It does not recursively search your home directory or silently adopt arbitrary
repositories.

Some attachments require Red Hat or private GitHub access. Bootstrap continues
through independent failures and reports them at the end.

## Workspace status

`bin/status` summarizes every declared attachment without fetching or changing
it. It reports missing repositories, branch or detached HEAD, dirty-file count,
and ahead/behind state when the current branch has an upstream.

Use ordinary Git commands from the exact repository path shown by the status
output. The workspace root itself has a separate Git history.

## Safety

Everything under `repos/` and most of `scratch/` is ignored. Those ignored paths
can contain valuable repositories and uncommitted changes. Never use
`git clean -x` or `git clean -xfd` at the workspace root.

Do not put secrets, credentials, tokens, private keys, or secret-bearing logs in
this repository. The old workspace's `secrets` file was deliberately not
migrated.

## Future ideas

- Add lightweight manifest profiles if the full attachment set becomes too
  expensive for small VMs.
- Add a non-mutating check for missing expected remotes when real usage shows it
  would prevent mistakes.
