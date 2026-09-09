# Upstream CI Perf & Eval

## Goal

Stand up a comprehensive performance benchmark and model accuracy evaluation pipeline for vllm upstream CI. Make key perf/eval workloads **gating** on nightly (and eventually per-commit), so regressions are caught before they ship.

This is a Q2 2026 SIG CI roadmap item:
- "Performance benchmark suite for nightly & release"
- "Model accuracy eval suite for nightly & release"

Rob Shaw (vllm leadership) flagged this as the **top priority** for Red Hat's upstream CI contribution — ahead of time-to-signal optimization.

## Context: Doug/Rob 1:1 (May 2026)

- Big takeaway: model eval should be the first step
- Focus areas from the meeting:
  1. **Perf & Eval** — make a proposal for XYZ models, push to make this gating
  2. **BFCL** — for tool calling eval specifically
  3. Per-commit DevX / time-to-signal (secondary)
  4. General "greenness" of CI (secondary)
  5. Release process (model releases, regular releases)
  6. Infra maintenance / donations
- TODOs from meeting:
  - Doug to get JIRAs going
  - Rob to make the list on perf & eval (doc to cover it)
- 1:1 notes: https://docs.google.com/document/d/15OsllGGTkeK9TxmIhmyFmhD0JD3HNYEZYlUe1YhzdRU/edit?tab=t.0#heading=h.90k0zz51ia8y

## Touchpoints & Data Flow

### End-to-end pipeline

```
workloads/*.yaml (YAML recipe in perf-eval repo)
    │
    ▼
Buildkite bootstrap (.buildkite/generate_pipeline.py)
  - globs workloads/*.yaml, selects nightly:true or WORKLOADS env var
  - generates one Buildkite step per workload, routed to GPU-specific queue
    │
    ▼
lib/run.sh (per-workload orchestrator)
  ├─ parse_workload.py → validates YAML, resolves vLLM image, emits shell vars
  ├─ server.sh → starts vLLM server (docker on H200, native on B200)
  ├─ vllm_bench runs first (if vllm_bench: block present)
  │   ├─ run_vllm_bench.sh → `vllm bench serve`
  │   ├─ results → results/<name>/bench-<config>.json
  │   └─ ingest_perf.py → POST to Cloud Run ingestion endpoint
  └─ lm_eval tasks next (if lm_eval: block present)
      ├─ run_lm_eval.sh → `lm_eval` CLI (one invocation per task)
      ├─ results → results/<name>/<task>/results_*.json + samples_*.jsonl
      └─ ingest.py → POST to Cloud Run ingestion endpoint
              │
              ▼
Cloud Run endpoint (vllm-perf-data-ingest-*.us-central1.run.app)
              │
              ▼
Databricks tables:
  - vllm_data_warehouse.default.vllm_perf_data_ingest (perf)
  - vllm_data_warehouse.default.vllm_eval_data_ingest (eval)
              │
              ▼
Dashboard (ci.vllm.ai) pulls from Databricks on-demand
  - /eval: filters, time-series, leaderboard, sample browser
  - /perf: throughput-latency pareto charts
  - /compare: baseline vs candidate delta analysis
```

### What repos to touch for each type of work

| Task | Repo | What to change | Access needed |
|------|------|----------------|---------------|
| **Add a new model** | `vllm-project/perf-eval` | Write a YAML file in `workloads/`, open PR | Repo write access |
| **Add new lm_eval task** (gsm8k, gpqa, etc.) | `vllm-project/perf-eval` | Add task to recipe's `lm_eval.tasks` list | Repo write access |
| **Add new hardware** | `vllm-project/perf-eval` | Add GPU profile in `lib/gpu_profiles.yaml`, then write workload recipes | Repo write + BK queue access |
| **Add BFCL eval** | `vllm-project/perf-eval` + maybe `vllm-dashboard` | See BFCL section below | Repo write + possibly dashboard |
| **Trigger a build** | Buildkite UI or MCP tools | Trigger on `vllm/perf-eval` pipeline | Buildkite org access |
| **Modify ingestion** | Cloud Run service | Change ingest endpoint logic | DevOps/infra access |
| **Modify dashboard** | `vllm-project/vllm-dashboard` | TypeScript/Next.js changes | Dashboard repo write |

### Key: adding models is just YAML

This is the good news for delegation. Adding a model is:
1. Copy an existing recipe (e.g. `workloads/qwen3_5_h200.yaml`)
2. Edit model name, serve args, eval tasks
3. Set `nightly: true`
4. Open PR

No Buildkite config changes, no dashboard changes. The pipeline auto-discovers new `workloads/*.yaml` files, and the dashboard auto-discovers new models/tasks from Databricks.

### BFCL: the question mark

BFCL is a separate eval harness from lm_eval. The dashboard auto-discovers any eval data that matches the lm_eval JSON schema (`results → task → metric,filter`). Two paths:

1. **If BFCL can output lm_eval-compatible JSON** — zero dashboard changes. Just need a new runner in perf-eval (like `run_bfcl.sh` alongside `run_lm_eval.sh`) and a new recipe block type.
2. **If BFCL has its own schema** — need a data adapter either in the ingest step (transform to lm_eval schema before POSTing) or in the dashboard (`src/lib/eval-data.ts`).

Investigation needed before implementation.

### Dashboard internals (for reference)

- **Stack**: Next.js App Router on Vercel, Databricks SQL Warehouse, Postgres (Supabase for operational data)
- **Workspace attachment**: `repos/vllm/vllm-dashboard`
- **Eval page** (`src/app/eval/page.tsx`): auto-populates filter widgets from available data, shows time-series with σ significance, sample browser
- **Perf page** (`src/app/perf/page.tsx`): pareto frontier charts (throughput vs latency), filters by model/device/TP/concurrency. Chart configs are hardcoded — adding new chart types requires code changes.
- **Compare page** (`src/app/compare/`): baseline vs candidate image comparison with regression detection (2% threshold for perf, 2σ for eval)
- **No dashboard changes needed for new models or lm_eval tasks** — all auto-discovered from Databricks

### perf-eval repo internals (for reference)

- **Workspace attachment**: `repos/vllm/perf-eval`
- **Key files**:
  - `.buildkite/generate_pipeline.py` — step generator, auto-discovers `workloads/*.yaml`
  - `lib/run.sh` — main orchestrator
  - `lib/parse_workload.py` — YAML parser, image resolution, task validation
  - `lib/server.sh` — vLLM server lifecycle (docker vs native runtime)
  - `lib/run_lm_eval.sh` — lm_eval wrapper
  - `lib/run_vllm_bench.sh` — vllm bench wrapper
  - `lib/ingest.py` — eval results → Databricks
  - `lib/ingest_perf.py` — perf results → Databricks (transforms units, divides throughput by TP)
  - `lib/gpu_profiles.yaml` — GPU queue/env/cache definitions
  - `CLAUDE.md` — agent conventions, Buildkite workflow details
- **Image resolution**: `VLLM_IMAGE` > `VLLM_COMMIT` > workload `vllm.image` > `vllm/vllm-openai:latest`
- **Ingestion is best-effort**: failures log but don't abort the pipeline
- **vllm_bench runs first**: surfaces perf bugs quickly before waiting on full eval

### Triggering Buildkite builds from fork PRs

We don't have push access to `vllm-project/perf-eval`, but we **can** trigger builds against fork PR commits via the Buildkite MCP tools. The trick: once a PR is open, GitHub makes the fork's commits accessible from the upstream repo. Buildkite resolves them fine.

```
mcp__buildkite__create_build(
  org_slug="vllm",
  pipeline_slug="perf-eval",
  commit="<full 40-char SHA>",          # MUST be full SHA — short SHAs fail from forks
  branch="dougbtv/<branch-name>",       # fork branch name, NEVER main
  message="<description>",
  environment=[
    {"key": "VLLM_IMAGE", "value": "vllm/vllm-openai:nightly"},
    {"key": "VLLM_COMMIT", "value": "<vllm main SHA>"},
    {"key": "WORKLOADS", "value": "<workload stem>"}   # scope to specific recipes
  ]
)
```

**Key rules:**
- **Always use full 40-char SHAs** — abbreviated SHAs fail to resolve from fork PRs (learned the hard way on build #114)
- **Branch must be the fork branch** (e.g. `dougbtv/add-bfcl-eval`), never `main` — builds on main clutter the maintainers' view
- **VLLM_IMAGE and VLLM_COMMIT are required** — they set which vLLM build to test against
- **WORKLOADS** scopes the run to specific workloads (omit to run all `nightly: true`)
- To cancel: `mcp__buildkite__cancel_build(org_slug="vllm", pipeline_slug="perf-eval", build_number="N")`
- To check status: `mcp__buildkite__get_build(...)` with `job_state="failed,broken"`, then `tail_logs` for diagnosis

## Existing Infrastructure

### perf-eval repo (brand new, Apr 27 2026)

- Repo: https://github.com/vllm-project/perf-eval
- YAML recipe-driven: one file per (model, hardware) combo in `workloads/`
- Buildkite pipeline: https://buildkite.com/vllm/perf-eval
- Supports two evaluation modes per recipe:
  - **lm_eval** — accuracy via lm-evaluation-harness (gsm8k, aime25, gpqa, etc.)
  - **vllm_bench** — perf via `vllm bench serve` (throughput, latency at various concurrency)
- GPU profiles defined in `lib/gpu_profiles.yaml` — currently H200 and B200

### Current workloads (6 recipes, all H200)

| Recipe | Model | Nightly | Added |
|--------|-------|---------|-------|
| deepseek_v3_2_h200.yaml | DeepSeek v3.2 | yes | launch |
| deepseek_v4_pro_5_h200.yaml | DeepSeek V4 Pro | yes | May 15 (PR #4, NickLucche) — bugfix PR #6 open, hasn't passed pipeline yet |
| glm_5_1_h200.yaml | GLM 5.1 | yes | launch |
| gpt_oss_120b_h200.yaml | GPT-OSS 120B | yes | May 26 (PR #9, Doug) — TP=4, gsm8k + BFCL + 8k/1k bench. Build #128 passed. |
| kimi_k2_5_h200.yaml | Kimi K2.5 | yes | launch |
| minimax_m2_5_h200.yaml | Minimax 2.5 | yes | launch |
| qwen3_5_h200.yaml | Qwen3.5-397B-A17B-FP8 | yes | launch |

### In-flight PRs (as of May 27)

| PR | Author | Status | What |
|----|--------|--------|------|
| [#5](https://github.com/vllm-project/perf-eval/pull/5) | NickLucche | Open | Nemotron Super + MTP recipe |
| [#6](https://github.com/vllm-project/perf-eval/pull/6) | NickLucche | Open | DSv4 Pro bugfix (hasn't passed pipeline) |
| [#9](https://github.com/vllm-project/perf-eval/pull/9) | Doug | Open | GPT-OSS 120B recipe (TP=4, gsm8k + BFCL + bench) — build #128 passed |

### Dashboards

- Eval dashboard: https://vllm-ci-dashboard.vercel.app/eval
- Perf dashboard: https://vllm-ci-dashboard.vercel.app/perf
- Compare (release baselines): https://vllm-ci-dashboard.vercel.app/compare
- **Nightly page** (new, merged May 11): https://vllm-ci-dashboard.vercel.app/nightly — full CI tracker with perf/eval deltas
- Dashboard codebase: https://github.com/vllm-project/vllm-dashboard (TypeScript/Next.js on Vercel, Databricks warehouse backend)

### Slack

- Channel: #sig-perf-eval
- Kevin shared initial setup, nightly job runs all 5 workloads
- Open items from Kevin: add DeepSeek v4?, migrate nightly LM eval tests, support B200
- Rob: nightly lm eval tests need pruning, swap to latest models + recipes

## Model & Hardware Targets

### Models discussed across SIG CI meetings

**Currently in perf-eval nightly:**
- DeepSeek v3.2, GLM 5.1, Kimi K2.5, Minimax 2.5, Qwen3.5

**Discussed as targets (from Charlotte's proposal, Dec 2025):**
- deepseek r1, kimi k2 thinking, qwen vl, gpt-oss, llama4, another dense (qwen or llama3)
- After initial: add linear attention models

**From Rob's meeting:**
- BFCL for tool calling eval (new category)
- DeepSeek v4 (Kevin mentioned as TODO — landed via NickLucche PR #4, bugfix in progress)

**Commercially critical / regression-motivated:**
- **gpt-oss-120b** — active perf regression reported (v0.16 → v0.20), upstream issue [vllm#40838](https://github.com/vllm-project/vllm/issues/40838), Red Hat internal [PSAP-2442](https://redhat.atlassian.net/browse/PSAP-2442). Rob Greenberg: "this is a critical commercial model for us." **Now in perf-eval** — PR [#9](https://github.com/vllm-project/perf-eval/pull/9), 8xH200 TP=4, build #128 passed. Pending merge.

**From May 19 SIG CI meeting:**
- **Gemma 4** — new model target
- **DeepSeek V4 MTP** — MTP/speculative decoding variant, distinct from the DSv4 Pro recipe that already landed
- Kevin confirmed dashboard changes needed for BFCL metrics display

### Hardware matrix

- H200 (current, all 5 recipes)
- B200 (GPU profile exists, no workloads yet — Kevin flagged as TODO)
- H100 (Red Hat contributing — Apr 7 meeting, has been stuck for months)
- MI300 (AMD, 72 new nodes coming — Apr 28 meeting)
- A100 (discussed as target in Charlotte's proposal)

### Eval benchmarks in use

- **GSM8k** — math reasoning
- **AIME25** — competition math
- **GPQA diamond** — graduate-level science QA
- **BFCL** — tool calling (Rob wants this added — needs investigation)

### Perf benchmarks in use

- `vllm bench serve` with various input/output/concurrency configs
- Typical config: 8k-in/1k-out at concurrency 128-256
- `speed_bench` dataset for throughput measurement

## Release validation (related)

The v0.20.0 release validation (Apr 21) included:
- GSM8k, AIME25, GPQA diamond for accuracy on Qwen, GLM5, DSv32
- Perf benchmark 8k/1k
- H200 & GB200
- Nightly basis

This establishes the pattern: the same perf-eval pipeline should feed release decisions.

## Key People

- **Rob Shaw** (Red Hat) — vllm leadership, driving perf-eval priority
- **Kevin Luu** (Anyscale) — set up perf-eval repo/pipeline/dashboard, SIG CI lead
- **Charlotte Qi** (Meta) — test coverage and model selection proposals
- **Andrey Talman** (Meta) — PyTorch release coordination, nightly signals
- **Huamin Li** (Meta) — trunk health, model onboarding (Qwen3, gpt-oss)
- **Doug Smith** (Red Hat) — coordination, JIRA, team delegation

## Strategic Notes

Doug's perspective: Rob's framing is "jam this out fast" but our approach needs to be:
- **Team-oriented** — break work out so multiple people can contribute in parallel
- **Sustainable** — integrate into the SIG CI structure, not a one-off sprint
- **Complementary** — this feeds into the time-to-signal work (INFERENG-6820) since perf/eval jobs are part of overall CI time
- Balance Rob's urgency with existing obligations and team capacity

## Epic: INFERENG-6864

https://redhat.atlassian.net/browse/INFERENG-6864 — "Upstream Perf & Eval: Model Coverage & Pipeline Expansion"

### Stories

| Key | Title | Assignee | Status |
|-----|-------|----------|--------|
| [INFERENG-6865](https://redhat.atlassian.net/browse/INFERENG-6865) | Coordinate perf & eval effort | Doug | In Progress |
| [INFERENG-6866](https://redhat.atlassian.net/browse/INFERENG-6866) | Extend model coverage in perf-eval | Unassigned | New |
| [INFERENG-6867](https://redhat.atlassian.net/browse/INFERENG-6867) | Add BFCL tool calling evaluation | Unassigned | New |
| [INFERENG-7116](https://redhat.atlassian.net/browse/INFERENG-7116) | Add gpt-oss-120b to upstream perf-eval pipeline | Doug | Review |
| [INFERENG-7138](https://redhat.atlassian.net/browse/INFERENG-7138) | Add Gemma 4 to upstream perf-eval pipeline | Unassigned | New |
| [INFERENG-7139](https://redhat.atlassian.net/browse/INFERENG-7139) | Add DeepSeek V4 MTP to upstream perf-eval pipeline | Unassigned | New |
| [INFERENG-7140](https://redhat.atlassian.net/browse/INFERENG-7140) | BFCL dashboard updates for tool calling metrics | Doug | **Closed** |
| [INFERENG-7141](https://redhat.atlassian.net/browse/INFERENG-7141) | [spike] Long context eval: GLM 5 garbage output investigation | Unassigned | New |
| [INFERENG-6868](https://redhat.atlassian.net/browse/INFERENG-6868) | Expand hardware coverage for perf-eval | Unassigned | New |

### Future follow-on (not in this epic)

- Release gating — once coverage is mature, define pass/fail criteria and integrate into release workflow
- Dashboard enhancements — concurrency sweep, regression alerting
