# Model-opt Lane Handoff — RHAIIS 3.5.4 (2026-11-23)

**For:** Doug (z-stream rotation owner)
**Trigger:** Selbi confirmed Model-opt is REQUIRED for the 3.5.4 release
(Tuesday 2026-11-24 target). The 10508 lane card was the neglected lane:
empty acceptance criteria, no evidence — and my earlier "no image exists"
finding was WRONG (namespace miss, corrected below).

## What Model-opt actually is

- A **separate image line** in its own Quay namespace:
  `quay.io/aipcc/rhaiis-model-opt/` — NOT a containers-repo variant.
  There is no model-opt conf in `build-args/` on the `3.4` or `3.5`
  branches (confs: cpu, cuda, gaudi, neuron, rocm, spyre, tpu).
- **CUDA-based**: `cuda-ubi9` is the only variant in the namespace.
  Per Dan Huang's 3.6 analysis, the model-opt image carries the pip/wheel
  stack from its base image.
- **Lane owner precedent: Dan Huang.** He ran the 3.6 model-opt RC
  ([INFERENG-12010](https://redhat.atlassian.net/browse/INFERENG-12010),
  Closed). His process: final RC image + CVE analysis with rpm-level
  verification of suspected false alarms (e.g. pip/wheel CVE that the
  base image reports green on Quay).
- Related mechanism hint:
  [INFERENG-11851](https://redhat.atlassian.net/browse/INFERENG-11851)
  "Float vLLM rhaiv respins: model-opt variant manifest" — the model-opt
  image appears to be produced via a rhaiv respin with a variant manifest,
  not a containers-repo build.

## Current 3.5.4 state

- **RC images exist** in `quay.io/aipcc/rhaiis-model-opt/cuda-ubi9`:
  - `3.5.4-1794485429` (latest, built ~2026-11-12 evening — same era as
    the other post-base-refresh lane images)
  - `3.5.4-1794248683` (earlier, ~2026-11-11)
  - Floating `3.5.4` tag also present
- **The gap**: the latest model-opt image predates the bart 0.6.3 CUDA
  wheel bump (containers !1730 merged 2026-11-18). If Model-opt must
  carry the current wheels, it needs a **rebuild on the new CUDA base**
  (`cuda-ubi9:3.5.4-1795123583`). If the z-stream bar is CVE floors +
  the Nov-12 base, the existing image may stand.
- No digest pin, no validation evidence, no acceptance criteria on
  [INFERENG-10508](https://redhat.atlassian.net/browse/INFERENG-10508).

## Approach options

- **Option A — rebuild on current base** (if wheel currency is
  required): respin model-opt against `cuda-ubi9:3.5.4-1795123583`,
  then run Dan's 3.6-style validation.
- **Option B — stand on the Nov-12 image**: pin
  `3.5.4-1794485429`'s digest, run the CVE analysis (Dan's rpm-level
  approach), record evidence on 10508.
- Either way, **talk to Dan Huang first** — he owns the 3.6 model-opt
  process and knows the build path (and whether the respin manifest
  from 11851 is the mechanism).

## Open questions (Selbi / Dan)

1. What does "required" mean for this z-stream: image + CVE check
   (Dan's 3.6 bar), or functional validation beyond that?
2. Must the model-opt image carry the bart 0.6.3 wheels, or is the
   Nov-12 base acceptable?
3. Who triggers the 3.5.4 model-opt build/respin, and what is the
   hard cutoff vs the Tuesday target?

## Suggested immediate moves

1. Ping Dan Huang with the two image refs and the rebuild-vs-stand
   question (he closed the 3.6 equivalent a week ago — freshest
   context in the building).
2. Get acceptance criteria filled in on 10508 (it's an empty shell —
   Selbi's "required" answer should populate it).
3. Decide the path, run the validation, pin the digest + evidence on
   10508, move it to Review with the other lanes.

## Corrections to prior records

- The 2026-11-23 epic comment on INFERENG-10471 says Model-opt "has no
  image, build config, or acceptance criteria" — the image part is now
  known wrong (namespace miss). A correction comment should follow this
  handoff. The map's Friday sweep finding had the same miss.

*Compiled 2026-11-23 ~20:30 UTC by Rand from live Quay/GitLab/Jira
investigation. Evidence: rhaiis-model-opt namespace listing, Dan Huang's
comments on INFERENG-12010, build-args listings on origin/3.4 and
origin/3.5, rhaiis group project list.*
