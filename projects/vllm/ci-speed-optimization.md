# Upstream CI Speed Optimization

## Goal

Trimming down CI time.
CI on avg takes 1.5hrs-2hrs.
Our goal for the quarter is to trim it down to 30 minutes.
Amr has this tool called bkslow (you can probably search in #sig-ci).

This is a **Q2 2026 SIG CI roadmap item** — "Bring time to signal from 1.5hrs down to 30 minutes" is item #1 on the Q2 roadmap.

## Context & Research

### Q2 Roadmap (from CI SIG meeting notes, top of agenda)

The full Q2 roadmap includes:
1. **Bring time to signal from 1.5hrs down to 30 minutes** <-- this epic
2. Migrate L4 fleet on AWS to MIG slices on H200
3. Performance benchmark suite for nightly & release
4. Model accuracy eval suite for nightly & release
5. Automatic test target determination
6. Test coverage data collection
7. Automatic quarantine for flaky tests
8. Improve torch nightly coverage
9. Improve AMD coverage

### Q1 Retrospective (Mar 24, 2026)

- What went well: Reliability, nightly failure tracking, suspicious PR detection
- What didn't: **CI time is still really long**, not enough model coverage
- "Time to signal -> 30 mins" was already a Q1 goal but wasn't achieved

### Prior Art: Meta-Led Docker Build Speed Optimization (Late 2025 - Early 2026)

This is the previous traunch of optimization work, primarily driven by Meta (Amr Mahdi, Reza Barazesh, Junpu Fan) with Red Hat participation (Doug, Andy Feldman):

**Timeline:**
- **Nov/Dec 2025**: Docker builds were ~35+ minutes. Amr started restructuring Dockerfiles for cache-friendliness. Image size reduced ~8.3GB. Remote build caching introduced. CUDA kernels split into separate Docker stage.
  - PRs: vllm#28823, vllm#29270, vllm#29060, vllm#29452, ci-infra#212
  - Tracking board: https://github.com/orgs/vllm-project/projects/33/views/2
- **Dec 2025**: Caching active, saving ~15 minutes (~30% savings). Amr working on AMI with warm cache baked in daily.
- **Jan 6, 2026**: Incremental builds (Python-only) down to ~16 minutes (from 35+). EC2 IOPS/throughput increase brought it to ~12 min. Targeting ~7 min with AMI warm cache.
- **Jan 20, 2026**: Docker builds now **5-7 minutes** for most cases. Huge milestone.
- **Feb 2026**: Build time regression from 7-10 min back to 15-20 min (pipeline generator migration invalidated cache). Doug found the culprit, Amr fixed it. Also optimized AMD CI build (~20 min, looking to cut further).
- **Mar 2026**: Build time regression fixed. Amr added more PRs for cleanup including precompiled wheel usage.

**Key result**: Docker build time went from **~35 min → ~5-7 min** (an ~80% reduction). This was a massive win but only addresses one component of the full CI pipeline time.

### Current State of "Time to Signal" (as of Q2 2026)

- **Total CI time still 1.5-2 hrs** despite the build speed wins
- The remaining time is in **test execution**, not docker builds
- Key strategies being discussed/started:
  1. **Parallelizing CI jobs** → target max 20 min per job (Kevin proposed a merge rule: jobs >20 min can't merge)
     - Facebook has this rule internally
     - "Still looking for an owner for parallelizing jobs!" (Feb 2026)
  2. **Automatic test target determination** — only run tests affected by the change
     - Tarun (RH) proposed using pytest-testmon + coverage.py
     - Avinash (RH) design moving forward in shadow mode (Apr 2026)
     - `/runci` command system proposed: `runci auto`, `runci kernels`, `runci quantization test 1`
  3. **bkslow** — Amr's tool to analyze job duration and find slow tests
     - Repo: https://github.com/amrmahdi/bkslow (Go CLI, created Feb 2026)
     - Generates waterfall timeline charts from Buildkite build URLs
     - Can drill into individual jobs to see time breakdown by log section
     - Has interactive TUI mode (`--tui`), JSON output (`--json`), sorting by duration
     - Install: `go install github.com/amrmahdi/bkslow/cmd/bkslow@latest`
     - Needs BK API token with `read_builds` + `read_job_logs` (picks up `bk` CLI config automatically)
     - Kevin tried it (Feb 24 meeting) and says "it works really well! Will make PRs to trim down CI time soon"
  4. **Use more dummy weights** — reduce model download/load time in tests
  5. **Less e2e style unit tests** — lighter test patterns
  6. **L4 → H200 MIG migration** — faster hardware, ⅓ migrated as of Apr 14

### Key People

- **Amr Mahdi** (Meta) — bkslow tool author, docker build optimization lead
- **Kevin Luu** (Anyscale) — CI SIG lead, pipeline generator, overall coordination
- **Reza Barazesh** (Meta) — pipeline generator, docker caching, tree hugger
- **Zhewen Li** (Meta) — nightly failure tracking, AMD CI failures, auto-bisect
- **Avinash** (Red Hat) — automatic test target determination design
- **Tarun** (Red Hat) — pytest-testmon proposal for test selection
- **Doug Smith** (Red Hat) — docker build caching, test ownership by SIG, CI shadows
- **Andy Feldman / Andy Linfoot** (Red Hat) — ci-infra, IBM H100 cluster, auto-bisect

### Stale Epic: INFERENG-4249

"Upstream CI SIG: Q1 Alignment & Foundations" — created Jan 2026, 19 child stories.
This was a broad discovery/alignment epic. Relevant children that could be linked or moved:
- INFERENG-4252: [spike] Time-to-signal: bottleneck analysis & job decomposition
- INFERENG-4253: [spike] Automatic test targeting: feasibility & data sources
- INFERENG-4251: [spike] Flaky test handling: quarantine strategies & thresholds
- INFERENG-4169: Context aware testing: Test only what's been changed

## Epic: INFERENG-6820

https://redhat.atlassian.net/browse/INFERENG-6820 — "Upstream CI Speed Optimization: Time-to-Signal → 30 Minutes"

### Stories

| Key | Title | Assignee | Status |
|-----|-------|----------|--------|
| [INFERENG-6821](https://redhat.atlassian.net/browse/INFERENG-6821) | Coordinate CI speed optimization effort | Doug | In Progress |
| [INFERENG-6822](https://redhat.atlassian.net/browse/INFERENG-6822) | Profile CI builds with bkslow to identify slowest jobs and tests | Unassigned | New |
| [INFERENG-6823](https://redhat.atlassian.net/browse/INFERENG-6823) | Split and parallelize CI jobs exceeding 20 minutes | Unassigned | New |
| [INFERENG-6824](https://redhat.atlassian.net/browse/INFERENG-6824) | Identify and optimize slow individual tests | Unassigned | New |
| [INFERENG-6825](https://redhat.atlassian.net/browse/INFERENG-6825) | Implement flaky test quarantine to reduce rerun waste | Unassigned | New |
| [INFERENG-6826](https://redhat.atlassian.net/browse/INFERENG-6826) | Track time-to-signal metrics on CI dashboard | Unassigned | New |

### Parallel Workstream (tracked, not owned here)

- **Automatic test target determination** — Avinash's design in shadow mode. Huge potential win but has its own scope/timeline. We track awareness but don't depend on it or own it under this epic.
