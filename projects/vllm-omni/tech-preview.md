# vLLM-Omni Tech Preview — Working Doc

Status: **3.6-fast1 image released and validated; vLLM-Omni infrastructure merged to nm-cicd `main`; Tech Preview delivery is ready for closeout after residual children are closed or moved to follow-up epics** — updated 2026-09-15

## Context

Dev preview (INFERENG-6288) proved we can build vLLM-Omni through the nm-cicd pipeline: two-wheel build, UBI9 container image, 14/14 smoke test matrix. Shipped as a custom ServingRuntime on RHOAI 3.5 EA2 — no product changes required.

Tech preview was the product-commitment phase: move the build pipeline to main, validate the AIPCC downstream build end-to-end, and ship a released product image. Those defining outcomes are complete; remaining work is validation hardening, known-issue ownership, and later-release planning.

**Target release:** RHOAI 3.6 EA1
**Midstream tag deadline:** Aug 13, 2026
**Confirmed by:** Taneem Ibrahim (July 20, 2026)

Detailed build pipeline docs, architecture, and dev preview history: [MIDSTREAM_OMNI_BUILD.md](./MIDSTREAM_OMNI_BUILD.md)

### What "midstream tag" means

A midstream tag is a git tag on the midstream fork (e.g. `v0.26.0+rhaiv.0` on `nm-vllm-omni-ent`) that signals "this code is validated and ready to ship." It kicks off the downstream productization pipeline:

**Tag created** → AIPCC mirrors to GitLab → Renovate opens MRs in `rhaiis/pipeline` → wheels build → Renovate updates `rhaiis/containers` → Konflux builds RC images → model validation → ship to `registry.redhat.io`

Before cutting the tag: upstream sync, build & validate (wheel + image + accept-sync), CVE fixes (Snyk), model validation (smoke + accuracy + perf), and release prep (package diff, release branches, infra config, Renovate config).

## Epics

### [INFERENG-9000 — Tech Preview Readiness](https://redhat.atlassian.net/browse/INFERENG-9000) (Critical Path)

Critical-path tracker for the completed 3.6-fast1 downstream validation/release cycle and nm-cicd mainline integration. Keep non-gating stabilization in TP.next and 3.6 GA release deltas in INFERENG-10710.

| Key | Story | Status | Owner | Notes |
|-----|-------|--------|-------|-------|
| [INFERENG-8996](https://redhat.atlassian.net/browse/INFERENG-8996) | Merge feature/vllm-omni to main | Closed | Doug | [nm-cicd PR #645](https://github.com/neuralmagic/nm-cicd/pull/645) merged to `main` on Sep 15 at `a96eff0db144bd4f0fe31ec4812ae3c131d70a70`; required approval and PR checks passed. |
| [INFERENG-9500](https://redhat.atlassian.net/browse/INFERENG-9500) | CUDA accept-sync pipeline for main (PR #757) | Closed | Tarun | Merged July 31 |
| [INFERENG-9501](https://redhat.atlassian.net/browse/INFERENG-9501) | Rebase nm-cicd omni branches after PR #757 | Closed | Doug | Branches rebased |
| [INFERENG-9448](https://redhat.atlassian.net/browse/INFERENG-9448) | v0.26.0 sync coordination | Closed | Doug | v0.26 sync complete; matched AIPCC wheel releases produced. |
| [INFERENG-9553](https://redhat.atlassian.net/browse/INFERENG-9553) | v0.26.0 rebase (nm-vllm-omni-ent PR #33) | Closed | Doug | Merged. Midstream carry regression (jpg format) found and fixed. |
| [INFERENG-9522](https://redhat.atlassian.net/browse/INFERENG-9522) | Migrate smoke-test matrix to ocp-test | Closed | Doug | PR #31 merged |
| [INFERENG-9465](https://redhat.atlassian.net/browse/INFERENG-9465) | RHAIIS containers MR !795 | Closed | Doug | Merged to main Aug 25. Release-branch follow-up completed by !1001 on Sep 1. |
| [INFERENG-9323](https://redhat.atlassian.net/browse/INFERENG-9323) | AIPCC container coordination | Closed | Doug | Corrected RC1 reached stage and passed RHOAI validation. ITS and runtime-handoff questions are follow-up scope. |
| [INFERENG-9507](https://redhat.atlassian.net/browse/INFERENG-9507) | Add espeak-ng RPM to container image | Closed | Doug | Resolved via Konflux subscription key + `dnf install` in Containerfile. EA2 base image tracker: INFERENG-10063. |
| [INFERENG-9437](https://redhat.atlassian.net/browse/INFERENG-9437) | CI GPU visibility mismatch | Closed | Doug | GPU count fix via PR #748/#749 |
| [INFERENG-9614](https://redhat.atlassian.net/browse/INFERENG-9614) | Contract test pod deletion race | Closed | Doug | Cleanup selector too broad (SMOKE vs CONTRACT) |
| [INFERENG-9318](https://redhat.atlassian.net/browse/INFERENG-9318) | Align midstream Dockerfile with AIPCC patterns | New | Tarun | |
| [INFERENG-9483](https://redhat.atlassian.net/browse/INFERENG-9483) | SME review of feature/vllm-omni branch | New | Tarun | |
| [INFERENG-9003](https://redhat.atlassian.net/browse/INFERENG-9003) | First formal validation cycle | Closed | Doug | Corrected RC1 reached stage and passed RHOAI validation; final-RC validation moved to INFERENG-9841/10578. |
| [INFERENG-8018](https://redhat.atlassian.net/browse/INFERENG-8018) | Consolidate to single CUDA image | Closed | Tarun | PR #32 merged |
| [INFERENG-9441](https://redhat.atlassian.net/browse/INFERENG-9441) | Early K8s resource deletion race | Closed | Doug | PR #751 merged |
| [INFERENG-9497](https://redhat.atlassian.net/browse/INFERENG-9497) | FLUX.2-dev xet hash issue | Closed | Doug | HF_HUB_DISABLE_XET=1 workaround |
| [INFERENG-9615](https://redhat.atlassian.net/browse/INFERENG-9615) | Numba/NumPy version mismatch | Closed | Tarun | PR #832 merged |
| [INFERENG-9647](https://redhat.atlassian.net/browse/INFERENG-9647) | Remove Z-Image-Turbo stub from main | Closed | Tarun | PR #837 merged. Unblocked CUDA release. |
| [INFERENG-9660](https://redhat.atlassian.net/browse/INFERENG-9660) | Fork sync review tooling (PR #34) | Closed | Doug | Alex Brooks' diff-fork-sync.sh |
| [INFERENG-9672](https://redhat.atlassian.net/browse/INFERENG-9672) | Multi-GPU stage-override support for omni | Closed | Tarun | Stage-override support landed through PR #645. The final-RC WDC run still exposed only one GPU for a TP=4 deployment; any remaining runner/configuration defect needs separate follow-up rather than reopening the mainline-support deliverable by implication. |
| [INFERENG-9751](https://redhat.atlassian.net/browse/INFERENG-9751) | Cut v0.26.0 midstream tag | Closed | Doug | Tags cut, validated build confirmed. |
| [INFERENG-9744](https://redhat.atlassian.net/browse/INFERENG-9744) | Contract test 500 error — cache_salt validation | Closed | Tarun | Upstream vllm#51654 → nm-vllm-ent#655 → omni PR #36 |
| [INFERENG-9761](https://redhat.atlassian.net/browse/INFERENG-9761) | Qwen3-Omni-30B regression on v0.26 | In Progress | Tarun | Known error. Upstream fix pending (vllm#50548). |
| [INFERENG-9762](https://redhat.atlassian.net/browse/INFERENG-9762) | Renovate config missing enabled:true | Closed | Doug | Fixed via MR !806 |
| [INFERENG-9913](https://redhat.atlassian.net/browse/INFERENG-9913) | Remove PyNvVideoCodec from CUDA requirements | Closed | Doug | nm-vllm-ent PR #666 merged |
| [INFERENG-9931](https://redhat.atlassian.net/browse/INFERENG-9931) | espeak-ng in CI runner Dockerfile | Closed | Tarun | stratus #1205 + nm-cicd #950 merged |
| [INFERENG-10025](https://redhat.atlassian.net/browse/INFERENG-10025) | Backport TTS PII log fix | Closed | Doug | Upstream fix cherry-picked to nm-vllm-omni-ent |
| [INFERENG-10026](https://redhat.atlassian.net/browse/INFERENG-10026) | Validate omni-release.yml end-to-end | Closed | Doug | New workflow chain validated. |
| [INFERENG-10065](https://redhat.atlassian.net/browse/INFERENG-10065) | Document validated model list for EA1 TP | Closed | Doug | Model list posted to RHAISTRAT-1928. |
| [INFERENG-10069](https://redhat.atlassian.net/browse/INFERENG-10069) | Document Konflux build/trigger workflow | Closed | Doug | Proven workflow documented in `OMNI_AIPCC_HOW_TO.md`. |
| [INFERENG-10090](https://redhat.atlassian.net/browse/INFERENG-10090) | TTS pipeline produces wrong output language | New | Tarun | Model/test follow-up; not blocking the accepted RC1. |
| [INFERENG-10120](https://redhat.atlassian.net/browse/INFERENG-10120) | Review librosa removal / upstream-first fix | Closed | Doug | Review complete; follow-up belongs in upstream/carry-reduction work. |
| [INFERENG-10159](https://redhat.atlassian.net/browse/INFERENG-10159) | Track v0.26 AIPCC wheels through working image | Closed | Doug | Corrected wheel reached a working RC1 and the image was promoted to stage. |
| [INFERENG-10211](https://redhat.atlassian.net/browse/INFERENG-10211) | Resolve post-merge tokenizers conflict | Closed | Doug | Dependency conflict resolved. |
| [INFERENG-10371](https://redhat.atlassian.net/browse/INFERENG-10371) | Published wheel omitted runtime assets | In Progress | Doug | Immediate release defect mitigated downstream; durable upstream packaging fix and artifact-level wheel check moved to TP.next. |
| [INFERENG-10403](https://redhat.atlassian.net/browse/INFERENG-10403) | Loft validation fixes and revalidate midstream image | In Progress | Doug | Immediate post-merge task: prove the merged `main` workflow against a Midstream image with an unambiguous green result and cleanup evidence. Not a 3.6-fast1 image blocker. |
| [INFERENG-10401](https://redhat.atlassian.net/browse/INFERENG-10401) | Review multimodal model-serving draft docs | Closed | Doug | Review completed. |
| [INFERENG-9841](https://redhat.atlassian.net/browse/INFERENG-9841) | Final RC validation and CVE remediation | Review | Doug | Final candidate `3.6.0-fast.1-1789070722` validated 6/7; promotion recommended. |
| [INFERENG-10578](https://redhat.atlassian.net/browse/INFERENG-10578) | Track final v0.26 image through AIPCC validation | Closed | Doug | Final candidate and matrix evidence recorded; see the [Sep 11 handoff](../../notes/handoffs/2026-09-11-infereng-10578-ocp-validation.md). |

### [INFERENG-9273 — Post-TP Enhancements (TP.next)](https://redhat.atlassian.net/browse/INFERENG-9273)

Real work descoped from TP for timeline. This is now the active stabilization and road-to-GA planning bucket, not a release gate for 3.6-fast1.

| Key | Story | Status | Notes |
|-----|-------|--------|-------|
| [INFERENG-9005](https://redhat.atlassian.net/browse/INFERENG-9005) | Nightly build and test pipeline | New | Needs merge to main first |
| [INFERENG-9004](https://redhat.atlassian.net/browse/INFERENG-9004) | Upstream sync process — tracker for agentic sync | New | Links to [INFERENG-8930](https://redhat.atlassian.net/browse/INFERENG-8930) (AI Agent Initiative) |
| [INFERENG-9052](https://redhat.atlassian.net/browse/INFERENG-9052) | Omni-specific model validation registry | New | Daniele's input — extend or create for omni model types |
| [INFERENG-9048](https://redhat.atlassian.net/browse/INFERENG-9048) | RHOAI validation test integration — scope and constraints | New | Two paths: additive tests or accept-sync (preferred) |
| [INFERENG-9094](https://redhat.atlassian.net/browse/INFERENG-9094) | Upstream input validation fixes (vllm-omni#3649) | New | ~30+ skipped negative tests upstream |
| [INFERENG-9233](https://redhat.atlassian.net/browse/INFERENG-9233) | Upstream Docker build caching optimization | New | Alex Brooks flagged slow builds |
| [INFERENG-9257](https://redhat.atlassian.net/browse/INFERENG-9257) | Long-running dogfooding instance on OpenShift | New | STT/TTS endpoint via rhaiis-snippets |
| [INFERENG-9263](https://redhat.atlassian.net/browse/INFERENG-9263) | GPU test agents image pull waste | New | CI optimization |
| [INFERENG-8825](https://redhat.atlassian.net/browse/INFERENG-8825) | Evaluate PyPI hosting for internal wheels | New | Process improvement |
| [INFERENG-9335](https://redhat.atlassian.net/browse/INFERENG-9335) | Reconcile nm-cicd constraints with AIPCC builder | New | Critical Jira priority; first hurdle resolved, but the durable dependency-policy reconciliation remains. |
| [INFERENG-9523](https://redhat.atlassian.net/browse/INFERENG-9523) | Evaluate making nm-vllm-omni-ent repo private | New | Match nm-vllm-ent; tension with public upstream visibility |
| [INFERENG-9608](https://redhat.atlassian.net/browse/INFERENG-9608) | Redesign cross-repo GitHub Actions orchestration (Ent → nm-cicd) | Closed | `omni-release.yml` / `omni-pipeline.yml` chain landed and was validated. |
| [INFERENG-9645](https://redhat.atlassian.net/browse/INFERENG-9645) | Upstream midstream carries from nm-vllm-omni-ent | New | Eliminate divergence; Nick is upstream maintainer now |
| [INFERENG-8004](https://redhat.atlassian.net/browse/INFERENG-8004) | Midstream images report stale vLLM version (setuptools_scm tag miss) | New | pip shows `0.24.1.dev992+rhaiv.3...` instead of clean versions |
| [INFERENG-10063](https://redhat.atlassian.net/browse/INFERENG-10063) | EA2: Get espeak-ng added to AIPCC base image | New | Proper fix for EA1 Containerfile workaround. Yuchen requested tracker. |
| [INFERENG-9914](https://redhat.atlassian.net/browse/INFERENG-9914) | Eliminate carried requirements patches | New | Upstream fixes or replace source carries with build-time filtering. |
| [INFERENG-10172](https://redhat.atlassian.net/browse/INFERENG-10172) | Eliminate midstream carries | New | Audit, upstream, and remove wholesale on the next sync where possible. |
| [INFERENG-10184](https://redhat.atlassian.net/browse/INFERENG-10184) | Dedicated GPU infrastructure for vLLM-Omni CI | New | Needed for reliable multi-GPU validation and less shared-runner ambiguity. |
| [INFERENG-10371](https://redhat.atlassian.net/browse/INFERENG-10371) | Durable wheel package-data fix and artifact check | In Progress | Downstream release is mitigated; upstream issue #4932 and an artifact-level wheel assertion remain. |
| [INFERENG-10709](https://redhat.atlassian.net/browse/INFERENG-10709) | Finalize 3.6 model list and update release spreadsheet | New | Confirm fast2 disposition and cutoff, agree the initial 3.6 GA list, update the shared sheet, and reconcile CI coverage. |
| [INFERENG-10711](https://redhat.atlassian.net/browse/INFERENG-10711) | Integrate GuideLLM Omni benchmarking into Midstream CI | New | Verify the existing vLLM integration, add explicit Omni coverage, and prove it against a Midstream image. |
| [INFERENG-10712](https://redhat.atlassian.net/browse/INFERENG-10712) | Add GuideLLM benchmarking for vLLM-Omni upstream CI | New | Related to INFERENG-10711; run repeatable benchmarks in upstream perf-eval and publish them on the CI dashboard. |

### TP.next planning checkpoint (Sep 15)

The epic has enough inventory; the missing piece is sequencing and a definition of done. A useful first refinement pass is:

1. **Prove the new mainline foundation** — PR #645 and INFERENG-8996 are complete. Run INFERENG-10403 from merged `main` and record the Midstream-image result rather than continuing to treat a feature branch as the validation implementation.
2. **Make validation trustworthy** — stage-override support in INFERENG-9672 is closed; isolate the remaining WDC Qwen3-Omni GPU-visibility failure, complete INFERENG-10403, secure dedicated capacity through INFERENG-10184, and resolve the WDC PVC cleanup/RBAC defect.
3. **Reduce fork and packaging debt** — INFERENG-10371, INFERENG-10172, INFERENG-9914, INFERENG-9645, INFERENG-9761, and INFERENG-9004. Require built-wheel assertions so source-checkout tests cannot mask missing runtime assets again. INFERENG-9761 should move from TP to TP.next; its proposed vLLM core fix remains open and the durable ownership is unresolved.
4. **Stabilize the downstream contract** — INFERENG-9335, INFERENG-10063, and INFERENG-8004: dependency ownership, base-image dependencies, and trustworthy version identity.
5. **Expand continuous product evidence** — INFERENG-9005, INFERENG-9052, INFERENG-9048, and INFERENG-9257: nightly coverage, a maintained Omni model registry, RHOAI integration tests, and a long-running dogfood deployment.

Items added during refinement should fit one of those outcomes or state why they belong elsewhere. Keep release-specific fixes out of TP.next once their durable follow-up is captured.

## TP Scope — What We're Doing

Keep DP tooling, don't build new things, validate the AIPCC path.

1. **Merge feature/vllm-omni to main** — build pipeline on main for a real release
2. **AIPCC touchpoint validation** — verify midstream tags produce working AIPCC builds. Container files + package lists. Ricardo/OCTO did initial setup; we validate alignment.
3. **First validation cycle** — midstream build → tag → AIPCC builds → validate their images
4. **Image consolidation** — ship one CUDA image instead of cuda + cuda-develsdk
5. **TTS model cherry-picks only** — per Taneem, limit scope this cycle

## What We're NOT Doing for TP

- Redesigning any midstream tooling
- Building new test frameworks
- Nightly CI wiring (TP.next)
- Promising anything about GA timeline (Taneem: "TP in two releases leading to GA in 3.6")
- KServe/ODH integration (downstream team)
- Konflux pipeline setup (AIPCC side)

## AIPCC Architecture (from Daniele, July 17)

Four layers — midstream owns the container repo, AIPCC owns the rest:

1. **builder** — recipes/pipelines to build python packages from source, full dependency tree
2. **\*-pipeline repos** — build wheels per arch/accelerator, publish to gitlab index
3. **\*-containers repos** — Containerfiles that install wheel collections + add env vars/entrypoints. **This is our layer.**
4. **base-images** — minimal deps for installing builder-built wheels (CUDA SDK, etc.)

**Decided: Standalone product** (July 22) — separate image, not a variant of RHAIIS. Confirmed by Andre Lustosa (AIPCC), Doug, Daniele (+1). Srija setting up infrastructure via AIPCC-21244.

**Ownership:** AIPCC handles builder/pipeline/base-image onboarding. Midstream handles the container repository.

### Dependency management — two-layer model

Learned the hard way (July 24 — test tag broke AIPCC builds):

- **Source repo requirements** (nm-vllm-omni-ent `requirements/*.txt`) = what AIPCC sees, what gets baked into wheel `Requires-Dist` metadata. Dependency *removals* (fa3-fwd) and version *widenings* (av) **must** go here — constraints files cannot do these.
- **nm-cicd constraints** (`neuralmagic/constraints/vllm-omni.txt`) = additional *restrictions* at Docker image build time (fastapi ceiling, transformers ceiling). Useful for midstream builds but invisible to AIPCC.

Key rule: AIPCC builds only see the midstream fork. nm-cicd is not in the picture for downstream.

### Fondue (AIPCC monorepo — live July 25)

AIPCC consolidated builder + RHAI pipeline into a monorepo: [fondue](https://gitlab.com/redhat/rhel-ai/wheels/fondue). vllm-omni wheel is already building there (0.24.0+rhaiv.1), publishing to the same index. The RHAIIS pipeline repo is NOT yet folded in ("next target"). Per dhellmann: when fondue migration completes, wheel build updates happen automatically — we don't have to worry about it.

### RHAIIS pipeline collection

[MR !772](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/772) adds a `vllm-omni` collection (Nick Cao). Doug approved as CODEOWNER. Andre confirmed this is still the correct repo (not fondue) for RHAIIS collections. The dhellmann-confirmed path:

1. ~~Add collection to rhaiis pipeline repo (MR !772)~~ **Done** (approved July 28)
2. ~~Add Containerfile to containers repo~~ **Done** — [MR !795](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/795) merged to main; [MR !1001](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1001) landed the real container and `.3898` wheel pins on `3.6-fast1`.
3. ~~Add and enable Konflux/Tekton pipelines~~ **Done** — on-pull build succeeded; component-scoped tag `vllm-omni-cuda-v2026090101` triggered the first successful on-push release pipeline.

### Version coupling risk

AIPCC builder constrains vLLM version tightly per CUDA (e.g. `vllm>=0.24.0,<0.25.0`). vllm-omni upstream lags behind vllm releases. Andre says they can lag builder releases or keep a wider range in future cycles. This could bite us if omni needs a different vLLM version than what AIPCC is building.

### Product classification

Per dhellmann: this is part of **RHAII** (not RHOAI). Standalone image. When maturity level is right, work with RHOAI team to add to the operator bundle as a runtime image.

## Resources

- **Midstream lead:** Doug Smith
- **Shadow resource:** Tarun Kumar (active — PR #757, PR #32, PR #832, reviewing syncs)
- **OCTO contributors:** Nick Cao, Vance Raiti, Clodagh Walsh, Shaun Walsh (Benny covering while Shaun on vacation) — PRs under midstream supervision
- **AIPCC coordination:** Srija (Tekton/Pyxis), Andre Lustosa (fondue/wheels), Klara (container build process)

## Timeline

| Milestone | 3.6 EA1 (3.6-fast1) | 3.6 EA2 | 3.6.0 GA |
|-----------|---------------------|---------|----------|
| Planning Freeze | Jul 29 | Aug 26 | Sep 18 |
| Midstream Tag | **Aug 13** | Sep 9 | Oct 13 |
| Code Freeze | **Aug 31** (pushed from ~Aug 25) | — | — |
| Final RC | Sep 2 | Oct 6 | Nov 4 |
| Release | Sep 17 | Oct 15 | Nov 19 |

**Note:** Release is officially "3.6-fast1" (not "3.6 EA1"). The containers repo has a `3.6-fast1` release branch, vllm-omni Konflux triggers are enabled, and the first component-scoped release tag has completed successfully.

### Assessment (Sep 15 — released and mainlined)

**Midstream tag: done.** Tags `v0.26.0+rhaiv.0` and `v0.26.0+rhaiv.1` on nm-vllm-omni-ent (commit `5d15b057`), synced to GitLab mirror. INFERENG-9751 closed. Validated build: `quay.io/vllm/automation-vllm-omni:cuda-31483640232` — full smoke + contract test matrix passing.

**RHOAI smoke validation passed.** Syed Ali deployed Voxtral-TTS via midstream image, inferencing successful. RHOAI PRs ready to merge before Friday code freeze (Imran pushing).

**Contract test 500 bug: fixed.** Upstream vllm#51654 → nm-vllm-ent#655 → omni PR #36. INFERENG-9744 closed.

**TTS PII log fix: done.** Upstream fix merged, backported to nm-vllm-omni-ent. INFERENG-10025 closed. EA1 image won't leak user input in INFO logs.

**PyNvVideoCodec: removed.** Carried patch to strip prebuilt-only NVIDIA dep from CUDA requirements. INFERENG-9913 closed. Aligns with fondue's approach.

**CI orchestration redesign: validated.** Tarun's `omni-release.yml` + `omni-pipeline.yml` PRs merged and the new workflow chain was validated. INFERENG-10026 is closed.

**Known issue: Qwen3-Omni-30B regression on v0.26** (INFERENG-9761). Upstream vLLM says it's an omni-side threading design issue (njhill: "not intended to be called from multiple threads"). May need omni upstream fix or midstream carry. Not blocking EA1.

**AIPCC wheel-to-image path: proven.** Builder v44 and the constraints fixes produced matched `3.6-fast1.3898` vLLM/vLLM-Omni wheel releases for aarch64 and x86_64. Containers MR [!1001](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1001) replaced the release-branch placeholders, pinned those wheels, selected the EL9.8 CUDA base, and enabled the on-pull/on-push triggers. After the packaging defect was found, pipeline MR !897 produced corrected wheel edition `3.6-fast1.3948`; containers MR !1008 consumed it for the replacement release.

**First vLLM-Omni component release pipeline: succeeded.** Tag [`vllm-omni-cuda-v2026090101`](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/tags/vllm-omni-cuda-v2026090101) from the merged `3.6-fast1` commit triggered [Konflux on-push run `qkq9v`](https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/ai-tenant/pipelinerun/rhaiis-vllm-omni-cuda-ubi9-3-6-fast1-on-push-qkq9v). The build and Tech Preview label check passed; release snapshot `rhaiis-3-6-fast1-20260901-154733-000-qj` was created.

**First RC image was defective and must not be used:**

`quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1788277735`

Its published vLLM-Omni wheel omitted deploy YAMLs and other non-Python runtime assets. [INFERENG-10371](https://redhat.atlassian.net/browse/INFERENG-10371) tracks the packaging defect. The defect was corrected and a replacement image was built.

**Corrected RC1 was promoted to stage on September 3:**

`quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1788360443`

AIPCC Productization promoted this corrected image to stage on September 3. It passed RHOAI EA1 single- and multi-GPU validation on September 7; INFERENG-9840 is closed.

It was produced from containers tag `vllm-omni-cuda-v2026090201`; both the managed and final release pipelines succeeded. Immutable digest: `sha256:39dd7f54f3d3dd81f1ddd7a8399ad997c3dabf1c83f162776d468c224bc0e67d`.

**Focused formal validation passed.** [nm-cicd run `33662749582`](https://github.com/neuralmagic/nm-cicd/actions/runs/33662749582) deployed Qwen3-TTS successfully: the deployment became Ready, `/health` returned HTTP 200, both selected tests passed, real `/v1/audio/speech` inference generated audio, and cleanup completed. The run used nm-cicd commit [`0b194adbf`](https://github.com/neuralmagic/nm-cicd/commit/0b194adbf81b4b447f179054fcefc9fa06e5b21b), which corrects the readiness endpoint and Omni pytest fixtures.

**The overall Actions run is misleadingly red.** The unused PERFORMANCE path receives `benchmarks=null`; the image-test job itself passed. [INFERENG-10403](https://redhat.atlassian.net/browse/INFERENG-10403) tracks lofting the validation fixes to mainline, fixing the false-red workflow result, repeating validation against a midstream-built image, and recording final green-run and cleanup evidence.

**Final RC candidate released and validated:**

`quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1789070722`

Immutable digest: `sha256:4d17c21a588539f6429fd1d24e81b8a3f36448f2f4e21f1f241089dfa5568a42`. The image contains `v0.26.0+rhaiv.7`.

[nm-cicd run `34606096112`](https://github.com/neuralmagic/nm-cicd/actions/runs/34606096112) ran the seven-model OCP smoke matrix on the WDC H100 runner. Six workloads reached `/health` HTTP 200 and passed both functional smoke tests: FLUX.2-klein, FLUX.2-dev, FLUX.1-schnell, Z-Image-Turbo, Voxtral TTS, and Qwen3-TTS CustomVoice.

Qwen3-Omni-30B failed before readiness because TP=4 was requested while only one GPU was visible in the serving pod. This matches INFERENG-9672 and is deployment/test plumbing, not evidence of an image defect. The six passing wrapper jobs were marked red only because the runner service account cannot delete per-run PVCs; their readiness and inference evidence remain valid.

The Konflux on-push, managed, and final release pipelines succeeded on September 11. Release `rhaiis-3-6-fast1-20260910-200419-000-6k-0a7cf4e-cp48z` published the immutable image above; the moving `3.6.0-fast.1` alias was also updated. The Tech Preview label check passed.

**nm-cicd mainline integration complete.** [PR #645](https://github.com/neuralmagic/nm-cicd/pull/645) merged on September 15 at `a96eff0db144bd4f0fe31ec4812ae3c131d70a70` after the required approval and green PR checks. The merge commit's observed Snyk and pip-graph follow-up checks also passed. INFERENG-8996 is closed; INFERENG-10403 owns post-merge functional proof from `main`.

**Konflux integration status:** the downstream chain produced and released the final candidate. Dedicated ITS coverage, RHOAI runtime-template/handoff questions, and CI cleanup remain follow-ups, not evidence against the released image.

**Release chain:** ~~builder/constraints~~ → ~~matched wheels~~ → ~~release-branch container~~ → ~~corrected RC1~~ → ~~stage promotion~~ → ~~final-candidate OCP validation~~ → ~~final release~~. Mainline CI proof, the remaining Qwen3-Omni WDC deployment defect, and PVC cleanup are follow-up stabilization work.

**PM coordination:** Model list collected and posted to RHAISTRAT-1928. INFERENG-10065 closed.

**Code freeze pushed to Aug 31** (from ~Aug 25). Final RC and release dates unchanged (Sept 2, Sept 17).

**Overall: the Tech Preview delivery phase is complete.** The final image was released and six representative workloads passed; the remaining model failure is isolated deployment plumbing, and cleanup failures are runner RBAC noise. With PR #645 on `main`, residual INFERENG-9000 children should now be closed as obsolete/completed or moved to TP.next/INFERENG-10710. None should be misreported as an unfinished 3.6-fast1 release path.

### Constraints
- Europeans + summer — reduced availability
- Ricardo: mostly OOO in August, Wed/Thu check-ins
- Shaun on vacation; Benny + Clodagh covering

## Related Epics and Trackers

| Issue | What | Owner |
|-------|------|-------|
| [INFERENG-9840](https://redhat.atlassian.net/browse/INFERENG-9840) | **3.6-fast-1 Omni — Midstream validation and stage RC** (team-wide tracker, due Aug 19) | Doug Smith |
| [INFERENG-10710](https://redhat.atlassian.net/browse/INFERENG-10710) | **vLLM-Omni Midstream: RHOAI 3.6 GA release planning** (Omni remains Tech Preview) | Doug Smith |
| [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288) | Dev Preview epic (predecessor) | Doug Smith |
| [INFERENG-8929](https://redhat.atlassian.net/browse/INFERENG-8929) | AI Agent Initiative — agentic sync lives here | Doug Smith |
| [RHAISTRAT-1266](https://redhat.atlassian.net/browse/RHAISTRAT-1266) | Strategy ticket | — |
| [RHAISTRAT-1928](https://redhat.atlassian.net/browse/RHAISTRAT-1928) | RHAII TP strat (our strat — no exception needed) | Selbi Siddiqi |
| [RHAISTRAT-2493](https://redhat.atlassian.net/browse/RHAISTRAT-2493) | RHOAI serving strat (RHOAI side — needs planning exception) | Imran Khalidi |
| [RHAISTRAT-1926](https://redhat.atlassian.net/browse/RHAISTRAT-1926) | PM acceptance criteria for TP (full scope incl. RHOAI/playground/KEDA) | Selbi Siddiqi |
| [AIPCC-12521](https://redhat.atlassian.net/browse/AIPCC-12521) | AIPCC productization track | — |
| [RHOAIENG-55052](https://redhat.atlassian.net/browse/RHOAIENG-55052) | ODH/RHOAI integrations (downstream) | — |

## Open Questions

- Architecture council review — is it required, and what's the process?
- Should INFERENG-9483 be closed as satisfied/obsolete now that the pre-merge review and required approval are complete?
- What separate tracker owns the remaining WDC Qwen3-Omni GPU-visibility/configuration defect now that INFERENG-9672 is closed?
- What is the minimum dedicated ITS coverage Srija will land for vllm-omni, and when should it gate promotion?
- Will RHOAI provide an out-of-the-box vLLM-Omni KServe runtime template, or document a custom ServingRuntime deployment?
- When will the validation harness fixes land on mainline and produce a fully green midstream-image revalidation run?
- Who owns the WDC runner PVC delete permission/configuration follow-up?

## Decision Log

| Date | Decision | Context |
|------|----------|---------|
| 2026-09-15 | Treat Tech Preview delivery as complete and begin closeout | Final image `3.6.0-fast.1-1789070722` was released Sep 11; nm-cicd PR #645 merged to `main` at `a96eff0d`. Remaining epic children are closure/rehome or post-release stabilization work. |
| 2026-09-11 | Final RC candidate recommended for promotion | `3.6.0-fast.1-1789070722` passed 6/7 substantive OCP model validations; Qwen3-Omni reproduced INFERENG-9672, and wrapper cleanup failures were isolated to WDC PVC permissions. |
| 2026-09-07 | Corrected RC1 passed RHOAI validation | RHOAI EA1 single- and multi-GPU validation passed; the team-wide INFERENG-9840 tracker was closed Sep 8. |
| 2026-09-03 | Corrected RC1 promoted to stage | Stage promotion of `3.6.0-fast.1-1788360443` confirmed. |
| 2026-09-02 | Corrected RC1 accepted for stage promotion | Productization accepted `3.6.0-fast.1-1788360443` after focused Qwen3-TTS validation; promotion planned for Sep 3. |
| 2026-09-02 | Focused AIPCC-image validation passed | nm-cicd run `33662749582`: Ready deployment, `/health` 200, two tests passed, real audio inference, and cleanup. Overall red result is a PERFORMANCE `benchmarks=null` false negative tracked by INFERENG-10403. |
| 2026-09-02 | Earlier RC rejected | `3.6.0-fast.1-1788277735` omitted packaged runtime assets and must not be used; packaging tracked by INFERENG-10371. |
| 2026-09-01 | First vLLM-Omni component release pipeline succeeded | Containers !1001 merged to `3.6-fast1`; tag `vllm-omni-cuda-v2026090101` produced snapshot `rhaiis-3-6-fast1-20260901-154733-000-qj`. |
| 2026-09-01 | RC image handed to stage-promotion thread | `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9:3.6.0-fast.1-1788277735`; promotion completion not yet verified. |
| 2026-09-01 | First formal AIPCC-image revalidation attempted | nm-cicd run `33540039249` stopped before OCP deployment because the CI robot lacks access to the new Quay repository; AIPCC-31163 filed. |
| 2026-07-20 | TP confirmed for RHOAI 3.6 EA1, Aug 13 midstream tag | Taneem confirmed after consultation with Doug and Selbi |
| 2026-07-20 | Scope locked: keep DP tooling, validate AIPCC, TTS cherry-picks only | Taneem: "limit cherry pick to TTS models only" |
| 2026-07-20 | No GA wick burning | Taneem: "having TP in two releases leading to a GA in 3.6 is not a bad idea" |
| 2026-07-20 | Created TP.next epic (INFERENG-9273), moved 9 non-gating stories | Keep TP epic focused on critical path |
| 2026-07-17 | AIPCC process mapped by Daniele/Tarun | Four-layer architecture, midstream owns container repo, AIPCC owns builder/pipeline/base-images |
| 2026-07-16 | Performance benchmarking adopted as standard TP requirement | Multimodal AI sync call decision |
| 2026-07-16 | Feature refinement doc selected as central source of truth | Multimodal AI sync call decision |
| 2026-07-14 | RHAISTRAT-1926 surfaced — PM acceptance criteria for TP | AC 1.1 confirms: TP = certified image on registry.redhat.io |
| 2026-07-10 | Created TP epic (INFERENG-9000) with initial story breakdown | Planning session — scaffold phase |
| 2026-06-26 | Midstream not signing off on AIPCC process for DP | Communicated in tech alignment agreement doc, visible to leadership |
| 2026-07-28 | Fondue live (AIPCC monorepo); vllm-omni wheel building there | builder + RHAI pipeline consolidated. RHAIIS pipeline not yet folded in. |
| 2026-08-24 | `3.6-fast1` naming adopted, indexes created | Tarun advised fast naming over EA1. Andre confirmed indexes are just labeling, zero fallout. Infrastructure !885 merged. Both `3.6-EA1` and `3.6-fast1` indexes coexist. |
| 2026-08-24 | Fondue constraint lifted, builder v43.3.0, pipeline !847 merged | vllm 0.26 wheels actively building. Both packages at v0.26 in vllm-omni collection. |
| 2026-08-24 | INFERENG-9507 closed — espeak-ng resolved via Konflux | Konflux has activation keys, `dnf install` works. GitLab CI bypassed. PM approved. EA2 base image: INFERENG-10063. |
| 2026-07-28 | dhellmann: omni is part of RHAII, standalone image, follow existing RHAII patterns | Path: collection → Containerfile → Konflux pipelines. RHOAI integration later at maturity. |
| 2026-07-28 | RHAIIS pipeline MR !772 open — adds vllm-omni collection | Nick Cao; Doug to review as CODEOWNER |
| 2026-07-28 | Tarun confirmed PR #30 dep removals are correct | fa3-fwd is just FA renaming, not a real dep. |
| 2026-07-28 | v0.25.z rebase landed (PR #29 merged), INFERENG-9317 closed | PR #28 superseded by #29; v0.26.0 sync may follow |
| 2026-07-28 | Dependency management two-layer model clarified | Source repo requirements for removals/widenings (AIPCC-visible); nm-cicd constraints for restrictions only (midstream-only) |
| 2026-07-24 | Test tag v0.24.0+rhaiv.1 broke AIPCC builds | fa3-fwd and av>=14.0.0 in requirements; PR #30 fixed it. AIPCC only sees midstream fork. |
| 2026-07-22 | Standalone image confirmed (not RHAIIS variant) | Andre Lustosa, Doug, Daniele (+1). Srija setting up AIPCC infra. |
| 2026-06-12 | Dev preview scoped as custom ServingRuntime only | Confirmed by Imran Khalidi, Sherard Griffin, Taneem, Daniele Zonca |
| 2026-07-30 | Registry paths confirmed by Srija | TP: `registry.redhat.io/rhaii-early-access/vllm-omni-cuda-rhel9`, GA: `registry.redhat.io/rhaii/vllm-omni-cuda-rhel9` |
| 2026-07-30 | `.gitlab-ci.yml` entries added to containers MR !795 | Pavan flagged missing CI job defs; added variant + build job following rubin MR !718 pattern |
| 2026-07-31 | Tarun's single CUDA image PR #32 approved and merged | INFERENG-8018 closed |
| 2026-07-31 | RHOAI strat (RHAISTRAT-2493) refined by Imran, 8/8 quality | RHOAI side needs planning exception; RHAII side (1928) is fine |
| 2026-08-04 | Midstream carries causing regression | Hard pydantic format allowlist rejected `jpg`; upstream has no such validation. Carry from PR #22. |
| 2026-08-05 | Tekton pipelines merged on AIPCC side | Srija confirmed infra is ready. Pyxis registration done. |
| 2026-08-05 | Fondue does NOT have vLLM 0.26 support yet | Andre: significant effort, targeting AIPCC code freeze date. Pipeline repo (!772) still correct for RHAIIS. |
| 2026-08-05 | v0.26.0 rebase passing smoke tests | PR #33 on nm-vllm-omni-ent. Midstream side ready. |
| 2026-08-05 | Z-Image-Turbo stub removed from main | PR #837. Unblocked Willy's CUDA release pipeline. |
| 2026-08-06 | Multi-GPU stage-overrides bug found | TP=2 on 2 GPUs invalid for multi-stage omni layout. Tarun fixing — derive gpu_count from TP. |
| 2026-08-06 | Fork sync review tooling PR #34 | Alex Brooks' diff-fork-sync.sh — reduces sync PR review from 32 files to ~10 that need careful review. |
| 2026-08-10 | v0.26.0 rebase merged (PR #33) | INFERENG-9553 closed. |
| 2026-08-10 | FlashInfer cubin 0.6.14 not on PyPI | Only on flashinfer.ai/whl starting with 0.6.14. Tarun fixed in PR #873. |
| 2026-08-10 | Three AIPCC blockers flagged | cudnn-frontend 404, espeak-ng RPM path, fondue v0.26 wheels. Pinged Srija/Andre/Benny. |
| 2026-08-11 | Midstream tags cut on nm-vllm-omni-ent | `v0.26.0+rhaiv.0` and `v0.26.0+rhaiv.1` pushed, GitLab mirror synced. INFERENG-9751. |
| 2026-08-11 | Renovate config bug found and fixed | `nm-vllm-omni-ent` silently disabled by catch-all `enabled: false`. Fixed in [MR !806](https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline/-/merge_requests/806). INFERENG-9762. |
| 2026-08-11 | Renovate MR !808 auto-opened for vllm-omni bump | `vllm-omni==0.26.0+rhaiv.1` on main. Pipeline green. Closed !807 (3.5) — targeting 3.6. |
| 2026-08-11 | v0.26 wheels blocker clarified | RHAIIS pipeline CAN build v0.26 (MR pipelines green). Real blocker: !699 (vllm bump on main) not merged — shotgun bumps all variants. No release tarballs until it merges. |
| 2026-08-13 | espeak-ng RPM path clarified | Root cause: AIPCC strips `/etc/yum.repos.d/`, uses `/etc/rhaipcc/repos.d`. EA1: `dnf install` in Containerfile via activation keys (productization to wire up). EA2: base image. Watch-and-wait — net-new plumbing for containers repo. |
| 2026-08-13 | Srija: Konflux test build playbook | Tekton pipelines merged but disabled (CEL `&& false`). Remove guard, open MR (on-pull) or tag branch (on-push) to get test image at `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9`. |
| 2026-08-13 | Midstream tag validated and closed | Tags on commit `5d15b057`, validated build at `quay.io/vllm/automation-vllm-omni:cuda-31483640232`. Full smoke + contract test passing. INFERENG-9751 closed. |
| 2026-08-13 | Contract test 500 fix landed end-to-end | Upstream vllm#51654 → nm-vllm-ent#655 → nm-vllm-omni-ent PR #36 (wheel bump). INFERENG-9744 closed. |
| 2026-08-13 | Qwen3-Omni-30B regression: known error for tag | Upstream vLLM 0.26 bug (vllm#50398). Not cherry-picking — wait for upstream merge. INFERENG-9761. |
| 2026-08-18 | v0.26 wheels pipeline unblocked | Surgical MRs !829 (vllm) and !808 (vllm-omni) merged on rhaiis/pipeline main. Bypasses shotgun Renovate MR !699. Tarballs building. |
| 2026-08-18 | RHOAI smoke validation passed | Syed Ali deployed Voxtral-TTS via midstream image, inferencing successful. RHOAI PRs ready to merge. |
| 2026-08-18 | PyNvVideoCodec removed from CUDA requirements | nm-vllm-ent PR #666. Aligns with fondue. INFERENG-9913 closed. |
| 2026-08-18 | CI orchestration redesign PRs merged | omni-release.yml (PR #38) + omni-pipeline.yml (PR #945). End-to-end validation in progress (INFERENG-10026). |
| 2026-08-19 | TTS PII log fix backported | Upstream fix merged, cherry-picked to nm-vllm-omni-ent. INFERENG-10025 closed. |
| 2026-08-19 | espeak-ng base image path reopened for EA1 | Robby (base-image team) investigating. Ryan Petrello: Friday Aug 22 deadline — if not confident by then, too late for EA1. |
| 2026-08-19 | espeak-ng base image confirmed dead for EA1 | Sarah Berry: can't make it. Back to activation key + Containerfile `dnf install`. Nick driving AIPCC side (subscription keys on runners + Konflux). Doug pinged Yuchen (PM). Midstream ready — all remaining work is AIPCC infra. |
| 2026-08-19 | Qwen3-Omni-30B regression: upstream says omni-side fix | njhill: "not intended to be called from multiple threads" — threading design issue for vllm-omni, not vLLM core. |
| 2026-08-19 | Team-wide tracker surfaced | INFERENG-9840 "3.6-fast-1 Omni — Midstream validation and stage RC" (due Aug 19, Selbi). Linked to TP epic. |
| 2026-08-19 | espeak-ng resolved via Konflux workaround | Nick/Srija moved MR !795 to Konflux-only builds (subscription key access). Konfux build confirmed: espeak-ng installed, scanlibs passed. GitLab CI image build dropped. EA2 base image tracker: INFERENG-10063. |
| 2026-08-19 | vLLM 0.26 wheels blocked on fondue pytorch rebuild | Current wheels ship vllm==0.24.0. Fondue caps `<0.25.0` due to pytorch + UCX RPM rebuild. Pavan's fondue MR !413 in dev. Same blocker affects CUDA and Gaudi. Pipeline MR !829 can't merge until constraint lifts. |
| 2026-08-19 | MR !795 moved to Konflux-only, GitLab CI build dropped | Nick/Srija: Tekton enabled, ffmpeg from base image (no shim), scanlibs ignore for flashinfer JIT. Entrypoint aligned: `python3 -m vllm_omni.entrypoints.openai.api_server --omni`. |
| 2026-08-19 | Selbi requested validated model list for EA1 | Post to RHAISTRAT-1928 as comment. Dev preview set is the baseline. RHAISTRAT-2487 (FLUX.2) and 2488 (Qwen3-TTS) may overlap; 2490 (OmniVoice/k2-fsa) almost certainly not EA1. INFERENG-10065 tracks. |
| 2026-08-19 | RHAISTRAT-1928 linked to TP epic | PM strat ticket for RHAII TP. Selbi is PM owner. |
| 2026-08-20 | Code freeze pushed to Aug 31 | Was ~Aug 25. Final RC (Sept 2) and release (Sept 17) unchanged. rnoriega announced. |
| 2026-08-20 | Release officially named 3.6-fast1 | Branch `3.6-fast1` created in containers repo, product_version set. Konflux setup done (jrusz) but builds disabled for vllm-omni (kdreyer) — !795 not on main yet. |
| 2026-08-20 | Fork sync tooling PR #34 merged | Alex Brooks' diff-fork-sync.sh. INFERENG-9660 closed. |
| 2026-08-21 | Fondue MR !413 merged — pytorch constraint lifted | vLLM `<0.25.0` cap removed. Pavan cutting builder release. v0.26 vllm wheels imminent. Nick ready to bump vllm-omni pin. Flashinfer 0.6.15.post1 (Kimi-K3) doesn't affect us. |
| 2026-08-21 | PR #47 flagged — librosa removal needs upstream-first | Nick's midstream-only code change to avoid undeclared dep. Bad form — should fix upstream first. INFERENG-10120 tracks. |
| 2026-08-21 | Pipeline MR !847 approved, supersedes !829 | Bumps fondue builder to v43.3.0, pins `vllm==0.26.0+rhaiv.1` + `vllm-omni==0.26.0+rhaiv.2`. Pipeline #3809 running. |
| 2026-08-25 | Containers MR !795 merged to main | INFERENG-9465 closed. vllm-omni Containerfile in RHAIIS containers repo. Cherry-pick to `3.6-fast1` branch and Konflux re-enable next. |
