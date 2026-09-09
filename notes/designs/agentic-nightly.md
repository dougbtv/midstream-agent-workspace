# Agentic Nightly Build

Brainstorm and design doc for an autonomous agent-driven nightly build system for nm-vllm-ent midstream.

## The Big Idea

Replace the human in the human-agent loop from `skills/vllm-midstream-sync/SKILL.md` with an autonomous agent running [aicheat](https://git.decapod.one/brethil/aicheat), powered by a vLLM-served model on our existing cluster (rhaiis-midstream-snippets Helm deployment). A GHA workflow in nm-cicd kicks it off nightly (or on-demand), and the agent drives the entire merge -> build -> deploy -> smoke-test cycle solo.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Actions (nm-cicd)                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │  agentic-nightly.yml                              │  │
│  │  trigger: cron (nightly) or workflow_dispatch      │  │
│  │  inputs: upstream_ref, model_id, target_device...  │  │
│  └──────────────┬────────────────────────────────────┘  │
│                 │ dispatches k8s job                     │
│                 ▼                                        │
│  ┌──────────────────────────────────┐                   │
│  │  aicheat container (k8s pod)    │                   │
│  │  - aicheat + tools (gh, git,    │                   │
│  │    ssh, curl, uv)               │                   │
│  │  - skill file baked in          │                   │
│  │  - connects to vLLM API ──────────► vLLM inference  │
│  │    on same cluster              │    server (helm)   │
│  │  - runs the build skill         │                   │
│  │    autonomously                 │                   │
│  └──────────────┬────────────────────────────────────┘  │
│                 │ outputs                                │
│                 ▼                                        │
│  results: GH gist, job summary, slack (optional)        │
└─────────────────────────────────────────────────────────┘
```

## Components

### 1. aicheat container image

- Base: python 3.12 + uv
- Install: `aicheat` (from https://git.decapod.one/brethil/aicheat)
- Bundle: `gh` CLI, `git`, `ssh-client`, `curl`
- Bake in: the autonomous skill file (adapted from `skills/vllm-midstream-sync/SKILL.md`)
- Secrets: mounted at runtime via k8s secrets (GH PAT, SSH keys, HF token)
- **WORKDIR must be under `$HOME` (`/root`)** — aicheat calls `Path.cwd().relative_to(Path.home())` at import time; a WORKDIR outside `/root` causes a `ValueError`
- Entrypoint: `aicheat --no-confirm --host $AICHEAT_HOST "Run the agentic nightly build with upstream_ref=$UPSTREAM_REF ..."`

### 2. GHA workflow: `agentic-nightly.yml`

- Trigger: `schedule` (cron, same as existing nightly) + `workflow_dispatch` for one-offs
- Inputs:
  - `upstream_ref` (default: `main`, but accept any branch/tag/commitish)
  - `midstream_branch` (default: auto-generated like `nightly/YYYY-MM-DD`)
  - `model_id` (which model to smoke test with the built image)
  - `target_device` (cuda/rocm/cpu)
  - `agent_model` (which LLM powers the agent, e.g. `Qwen/Qwen3-235B-A22B`)

### 3. Autonomous skill (adapted from existing)

Same steps as the current `skills/vllm-midstream-sync/SKILL.md`, but the "human provides" section becomes workflow inputs. No interactive prompts — everything from env vars / config.

Agent autonomously:
1. Merges upstream ref into nm-vllm-ent
2. Pushes branches (nm-vllm-ent + matching nm-cicd branch)
3. Dispatches build workflow (wheel + image)
4. Monitors build via `gh run view` polling
5. On success: deploys to cluster (helm upgrade) or dev box (podman)
6. Runs smoke tests (health, chat completions, completions endpoint)
7. Reports results (GH gist + job summary)
8. Cleans up (GPU release, container stop, etc.)

### 4. Inference server (the agent's brain)

- Existing vLLM deployment from `rhaiis-midstream-snippets/charts`
- Needs a model capable of tool use and multi-step reasoning
- aicheat connects via `AICHEAT_HOST` (see cluster access below)

**Active cluster:** `https://api.modelsibm.ibmmodel.rh-ods.com:6443` (IBM MI300X)

Credentials from GCP Secret Manager `ocp_a100_cluster` (project `it-cloud-gcp-nm-automation`).
Use a separate kubeconfig to avoid overwriting local cluster config:

```bash
gcloud secrets versions access latest --secret=ocp_a100_cluster --project=it-cloud-gcp-nm-automation
KUBECONFIG=/tmp/kubeconfig-ocp-mi300x oc login https://api.modelsibm.ibmmodel.rh-ods.com:6443 \
  -u "htpasswd-cluster-admin-user" -p '<password>' --insecure-skip-tls-verify
```

**Available vLLM endpoints (namespace: `dtrifiro`):**

| Model | Service (in-cluster) | Route (external) | Tool calling |
|-------|---------------------|-------------------|--------------|
| Qwen/Qwen3.5-397B-A17B-FP8 | `qwen3-5-397b-vllm-svc:8080` | `qwen3-5-397b.apps.modelsibm.ibmmodel.rh-ods.com` | Yes (qwen3_coder parser) |
| mistral-large-3 | `mistral-large-3-vllm-svc:8080` | `mistral-large-3.apps.modelsibm.ibmmodel.rh-ods.com` | Unknown |

**Primary agent model:** `Qwen/Qwen3.5-397B-A17B-FP8` — 397B MoE with tool-call parsing, speculative decoding, 262K context.

**IMPORTANT:** `AICHEAT_HOST` must NOT include `/v1` — the OpenAI SDK appends it automatically. Including it causes a double `/v1/v1/` error.

For local testing (external route):
```bash
AICHEAT_HOST=https://qwen3-5-397b.apps.modelsibm.ibmmodel.rh-ods.com
AICHEAT_MODEL=Qwen/Qwen3.5-397B-A17B-FP8
OPENAI_API_KEY=<api-key-from-configmap>
```

For cluster Jobs (in-cluster service, namespace `dtrifiro`):
```bash
AICHEAT_HOST=http://qwen3-5-397b-vllm-svc.dtrifiro.svc.cluster.local:8080
```

**PVC:** `modelcache-pvc` is in namespace `arc-runners` (6Ti, RWX).

### 5. GitHub PAT (agent-secrets)

The agent needs a GitHub PAT to push branches and trigger workflows. Use a **Fine-Grained Personal Access Token** scoped as tightly as possible.

**Resource access:** Only these repos:
- `neuralmagic/nm-vllm-ent`
- `neuralmagic/nm-cicd`

**Permissions:**

| Permission | Level | Why |
|---|---|---|
| Contents | Read & Write | Push `nightly/` branches to both repos |
| Actions | Read & Write | Trigger `build-whl-image.yml`, read run status/logs |
| Metadata | Read | Auto-granted by GitHub |

Nothing else — no PR, issue, admin, or deployment permissions. The skill's push guardrail (only `nightly/` branches allowed) is a software-level safety net; consider also adding branch protection rules on `main` in both repos as a second layer.

**Expiration:** 90 days, rotate on expiry. Stored in k8s secret `agent-secrets` (namespace `dtrifiro`) under key `gh-token`.

**Current PAT owner:** @dougbtv

## What This Adds Over the Existing Nightly

The existing `nightly-build-whl-image.yml` does merge + build but stops there. The agentic version adds:

- **Autonomous troubleshooting**: if the build fails, the agent investigates logs, adjusts, retries
- **Full deploy + test**: not just "did it build" but "does it actually serve a model"
- **Flexibility**: easy to point at a PR branch, a tag, a specific commit for one-off validation
- **Self-documenting**: agent produces a gist/report with full context, gotchas, and results

## Design Decisions

| Question | Options | Leaning |
|----------|---------|---------|
| Where does the agent run? | (a) k8s pod on same cluster as vLLM, (b) directly on GHA runner | (a) — co-located with the API, simpler networking |
| How does the agent deploy the built image for testing? | (a) helm upgrade on same cluster, (b) SSH to dev box + podman | Configurable — (a) for cluster, (b) for dev box GPU access |
| How does the agent auth to GitHub? | k8s secret with CICD_GITHUB_PAT, mounted as env var | Standard |
| What model powers the agent? | Whatever's deployed — needs tool-calling support | Qwen3 or equivalent |
| How do results get reported? | GH gist, GHA job summary, optionally Slack | All three |
| Timeout / blast radius? | Max wall-clock for the agent run (e.g. 3h), automatic cleanup on failure | Needs a watchdog |
| Dockerfile for aicheat container | In nm-cicd (our infra) or in aicheat repo? | nm-cicd |

## GHA Workflows Needed

Two workflows will need stubs merged to main before `workflow_dispatch` works. Plan is to nail down the interface/inputs, get stubs merged once, then iterate on a feature branch with `gh workflow run <workflow> --ref <branch>`.

### 1. `build-agent-image.yml` — Build Agent Container Image

Generic/configurable so we can swap harnesses in the future (aicheat today, something else tomorrow).

- Trigger: `workflow_dispatch` (manual), callable from other workflows
- Inputs:
  - `harness` (default: `aicheat`, future: could be `claude-code`, `aider`, etc.)
  - `harness_version` (git ref or version tag for the harness repo)
  - `image_tag` (tag for the built image, default: `latest`)
  - `skills` (comma-separated list of skill files to bake in)
- Builds the Dockerfile from `agentic/`, pushes to quay
- Reusable — any workflow that needs an agent container calls this first (or uses a cached image)

### 2. `agentic-nightly.yml` — Agentic Nightly Run

- Trigger: `schedule` (cron) + `workflow_dispatch` for one-offs
- Inputs:
  - `upstream_ref` (default: `main`, but accept any branch/tag/commitish)
  - `midstream_branch` (default: auto-generated like `nightly/YYYY-MM-DD`)
  - `model_id` (which model to smoke test with the built image)
  - `target_device` (cuda/rocm/cpu)
  - `agent_model` (which LLM powers the agent, e.g. `Qwen/Qwen3-235B-A22B`)
  - `agent_image` (default: latest from `build-agent-image`, or override for testing)
- Launches the agent container on k8s, pointed at the vLLM API
- Collects results, posts artifacts

## Suggested File Layout in nm-cicd

```
nm-cicd/
├── .github/
│   ├── workflows/
│   │   ├── build-agent-image.yml        # generic agent container build
│   │   └── agentic-nightly.yml          # nightly agent run
│   └── actions/
│       └── agentic-build/               # (optional) composite action
│           └── action.yml
├── agentic/                             # new directory
│   ├── Dockerfile                       # agent container (WORKDIR: /root/workspace)
│   ├── build.sh                         # local build+push helper (podman)
│   ├── entrypoint.sh                    # headless runner (validates env, runs aicheat < /dev/null)
│   └── skills/
│       └── nightly-build.SKILL.md       # autonomous version of the midstream build skill
```

## Iteration Strategy

### The GHA chicken-and-egg problem

`workflow_dispatch` only works once the workflow file exists on the default branch (main). New actions require a merge to main before they're exercisable. We don't want to beg for multiple merges while iterating, so:

1. **Phase 1 doesn't need GHA at all.** We iterate on the container + skill by launching pods directly on the cluster.
2. Once the agent container is proven, we present the two workflow stubs to the team, get them merged to main in one shot.
3. After that, iterate on a feature branch using `gh workflow run <workflow> --ref <feature-branch>`.

This means phase 1 has zero blockers — no main merge needed, no workflow stubs, just `kubectl run` and iterate.

## Phased Rollout

### Phase 1 — Container + Skill (no GHA, no blockers) — SMOKE TESTS PASSING

Iterate locally and on the cluster, no GHA integration needed.

**Status (2026-05-15):** Smoke tests passing in both environments. Container builds with podman, connects to Qwen3.5-397B, executes tool calls, exits cleanly. Cluster Job runs in `dtrifiro` namespace with `agent-secrets` and `vllm-anyuid-sa`.

**Feature branches:**
- `dougbtv/agentic-nightly` on nm-cicd
- `doug/agentic-nightly` on rhaiis-midstream-snippets

**What's built:**

| File | Repo | Purpose |
|------|------|---------|
| `agentic/Dockerfile` | nm-cicd | Agent container (python 3.12 + aicheat + gh/git/ssh/curl/rg) |
| `agentic/entrypoint.sh` | nm-cicd | Headless runner — validates env, configures gh auth, runs aicheat with stdin closed |
| `agentic/build.sh` | nm-cicd | Local build+push helper (podman by default) |
| `agentic/skills/nightly-build.SKILL.md` | nm-cicd | Autonomous build skill (all inputs via env vars) |
| `charts/jobs/agentic-nightly.yml` | rhaiis-midstream-snippets | K8s Job manifest for cluster runs |
| `scripts/launch-agent.sh` | rhaiis-midstream-snippets | CLI wrapper for launching Jobs with configurable params |

**Container image:** `quay.io/dosmith/aicheat-agent` (public on quay.io)

**Cluster resources (namespace `dtrifiro`):**
- Service account: `vllm-anyuid-sa`
- Secret: `agent-secrets` (keys: `vllm-api-key`, `gh-token`)
- vLLM service: `qwen3-5-397b-vllm-svc:8080`

**How to iterate:**

```bash
# Build locally with podman
cd nm-cicd
./agentic/build.sh

# Test locally (if you can reach the vLLM API — no /v1 in the URL!)
podman run --rm \
  -e AICHEAT_HOST=https://qwen3-5-397b.apps.modelsibm.ibmmodel.rh-ods.com \
  -e AICHEAT_MODEL=Qwen/Qwen3.5-397B-A17B-FP8 \
  -e OPENAI_API_KEY=<api-key> \
  -e GH_TOKEN=$GH_TOKEN \
  -e DRY_RUN=true \
  quay.io/dosmith/aicheat-agent:latest

# Quick smoke test (no skill, just a tool call)
podman run --rm \
  -e AICHEAT_HOST=https://qwen3-5-397b.apps.modelsibm.ibmmodel.rh-ods.com \
  -e AICHEAT_MODEL=Qwen/Qwen3.5-397B-A17B-FP8 \
  -e OPENAI_API_KEY=<api-key> \
  -e AGENT_PROMPT="Use the execute_shell_command tool to run: echo HEALTH_CHECK_OK" \
  -e AGENT_SKILL="" \
  quay.io/dosmith/aicheat-agent:latest

# Push to quay for cluster testing
PUSH=true ./agentic/build.sh

# Launch on cluster (uses template defaults — Qwen3.5-397B in dtrifiro)
cd rhaiis-midstream-snippets
KUBECONFIG=/tmp/kubeconfig-ocp-mi300x ./scripts/launch-agent.sh --dry-run

# Launch with overrides
KUBECONFIG=/tmp/kubeconfig-ocp-mi300x ./scripts/launch-agent.sh \
  --host http://other-vllm-svc:8080 \
  --model other/model-id \
  --namespace dtrifiro

# Watch logs
KUBECONFIG=/tmp/kubeconfig-ocp-mi300x kubectl logs -f -n dtrifiro -l app=agentic-nightly --tail=100
```

**Exit criteria:** agent container can autonomously merge upstream, kick off a build, and report results — all from a manually launched pod or local podman run.

### Phase 2 — GHA Integration (one merge to main)

Once phase 1 is proven, present both workflow stubs to the team for a single merge to main.

1. Merge stubs for `build-agent-image.yml` and `agentic-nightly.yml` to main
2. Create feature branch, iterate on workflow inputs/logic using `gh workflow run --ref <branch>`
3. Wire up `agentic-nightly.yml` to launch the proven container
4. Add `workflow_dispatch` inputs for flexible targeting (upstream_ref, model_id, etc.)
5. Manual dispatch only at first — validate via real runs

**Exit criteria:** `gh workflow run agentic-nightly.yml` successfully launches the agent and produces results.

### Phase 3 — Nightly + Polish

- Add cron schedule for nightly runs
- Slack notifications on success/failure
- Full deploy + smoke test loop (the complete skill)
- Agent result artifacts stored as GHA artifacts

### Phase 4 — Advanced

- Multi-model smoke testing (agent tests several models per nightly)
- Integration with model-validation-configs for accuracy testing
- Agent-driven PR validation (point it at an upstream PR, get a full report)
- Historical trend tracking (build times, test pass rates, model compatibility)

## Gotchas / Lessons Learned

| Gotcha | Detail |
|--------|--------|
| `AICHEAT_HOST` no `/v1` | The OpenAI SDK appends `/v1` automatically. Including it in the URL causes `404` from double `/v1/v1/`. |
| WORKDIR must be under `/root` | aicheat calls `Path.cwd().relative_to(Path.home())` at import time. WORKDIR outside `$HOME` → `ValueError`. |
| Headless execution | Pass prompt as CLI arg + pipe `< /dev/null`. aicheat processes prompt + tool calls, then hits `EOFError` on next `get_input()` → clean exit. |
| `--no-confirm` not `--no-confirmation` | The flag for yolo mode (skip tool-call confirmations) is `--no-confirm`. |
| Quay repo must be public | Cluster nodes can't pull from private quay repos without an `imagePullSecret`. Made `quay.io/dosmith/aicheat-agent` public. |
| Separate kubeconfig for OCP | Local kubeconfig is at `/tmp/kubeconfig`. Always use `KUBECONFIG=/tmp/kubeconfig-ocp-mi300x` for the MI300X cluster. |
| Skill changes need image rebuild | The skill file is baked into the container at build time (`COPY skills/`). Any edit to `nightly-build.SKILL.md` requires `PUSH=true ./agentic/build.sh` to take effect on the cluster. |

## References

- aicheat: https://git.decapod.one/brethil/aicheat (cloned locally at `./aicheat/`)
- Existing nightly: `nm-cicd/.github/workflows/nightly-build-whl-image.yml`
- Build skill: `skills/vllm-midstream-sync/SKILL.md`
- Cluster deployments: `./rhaiis-midstream-snippets/charts/`
- JIRA: INFERENG-6206
