# Model-opt Lane Handoff — RHAIIS 3.5.4 (2026-11-23)

**For:** Doug (z-stream rotation owner)
**Trigger:** Selbi confirmed Model-opt is REQUIRED for the 3.5.4 release
(Tuesday 2026-11-24 target). The 10508 lane card was the neglected lane:
empty acceptance criteria, no evidence — and my earlier "no image exists"
finding was WRONG (namespace miss, corrected below).

## What Model-opt actually is

- A **separate image line** in its own Quay namespace:
  `quay.io/aipcc/rhaiis-model-opt/` — not a standard RHAIIS variant, but it
  is built from the same containers repository via
  `Containerfile.model-opt.cuda-ubi9` and
  `build-args.model-opt/cuda-ubi9.conf`.
- **CUDA-based**: `cuda-ubi9` is the only variant in the namespace. It uses
  its own `model-opt-wheels` release and the pipeline's separate
  `collections/model-opt/cuda13.0-ubi9/` dependency set. That collection
  does not contain `vllm-bart-plugin` or `vllm-beam-search-plugin`.
- **Lane owner precedent: Dan Huang.** He ran the 3.6 model-opt RC
  ([INFERENG-12010](https://redhat.atlassian.net/browse/INFERENG-12010),
  Closed). His process: final RC image + CVE analysis with rpm-level
  verification of suspected false alarms (e.g. pip/wheel CVE that the
  base image reports green on Quay).
- Related future mechanism work:
  [INFERENG-11851](https://redhat.atlassian.net/browse/INFERENG-11851)
  changes how the Model-opt lock expresses direct vLLM dependencies. It is
  not the mechanism for this 3.5.4 image build.

## Current 3.5.4 state

- **RC images exist** in `quay.io/aipcc/rhaiis-model-opt/cuda-ubi9`:
  - `3.5.4-1794485429` (latest, built ~2026-11-12 evening — same era as
    the other post-base-refresh lane images)
  - `3.5.4-1794248683` (earlier, ~2026-11-11)
  - Floating `3.5.4` tag also present
- **Disposition: no bart-driven rebuild is needed.** The image is built
  from containers commit `85e1c280e532403f943f2a9ff634adae3e27b97a`, uses
  the correct 3.5.7 CUDA base and Model-opt wheel release `3.5.4786`, and
  there are no Model-opt input changes between that commit and the current
  `origin/3.5` head. The later bart 0.6.3 change affects the separate RHAIIS
  CUDA wheel collection only.
- Candidate digest:
  `sha256:28060390da8d9287b4dc0b13408380317ba120890f7ace8bb0d02d63ca79899c`.
- [INFERENG-10508](https://redhat.atlassian.net/browse/INFERENG-10508)
  contains generic acceptance criteria, including `model-validation-ocp`,
  but the current nm-cicd workflows show no explicit Model-opt support.
  Selbi said Dan Huang can help Doug validate the image.

## Recommended path

1. Stand on `3.5.4-1794485429` at the digest above; do not rebuild for bart.
2. Work with Dan Huang, per Selbi, to run the appropriate Model-opt validation
   and establish whether the generic `model-validation-ocp` criterion should
   be replaced by a Model-opt-specific check.
3. Run and record the CVE analysis. If Quay reports the same pip/wheel issue
   seen on 3.6, use Dan's RPM-versus-base comparison to determine whether it
   is a false positive.
4. Record the image identity, validation, and CVE disposition on 10508, then
   move the lane to Review.
5. Rebuild only if validation or a real High/Critical finding identifies an
   image-content change that is actually required.

## CVE validation dispatched 2026-11-23

Both checks ran against the digest-pinned Model-opt candidate using nm-cicd
PR #1691 branch `doug/INFERENG-12106-quay-inspect-impl`:

- Quay/Clair `cves` run `45260415062` completed, but returned
  `status: unsupported`; it produced no usable CVE report and is not green
  evidence.
- Trivy `vuln` run `45260420988` completed successfully and found **zero
  Critical** but **14 fixable High package findings** across 12 unique CVEs:
  `libXrender`, `python3`/`python3-libs`, `python3-urllib3`, `tesseract`, and
  Python `setuptools`.
- The exact private 3.5.7 base comparison was attempted with the same branch,
  but both Quay (`45260976412`) and Trivy (`45260980634`) failed because the
  workflow's Quay credentials cannot pull `aipcc/base-images/cuda-13.0-el9.6`.
  Therefore inheritance from the base is strongly suggested for the RPM
  findings but not yet proved by a like-for-like scan.

**Disposition:** the candidate is not CVE-green on current evidence. Work with
Dan to classify the findings, obtain a base-image comparison or newer base,
and decide whether a Model-opt rebuild is required. Do not treat the successful
GitHub workflow conclusion as a passing security gate; the report contents are
the gate.

## Corrections to prior records

- The 2026-11-23 epic comment on INFERENG-10471 says Model-opt "has no
  image, build config, or acceptance criteria" — the image part is now
  known wrong (namespace miss). A correction comment should follow this
  handoff. The map's Friday sweep finding had the same miss.

*Compiled and corrected 2026-11-23 by Rand from live Quay/GitLab/Jira and
repository evidence. The correction replaces the earlier mistaken build-path
and bart-coupling assumptions with the actual Model-opt container definition,
wheel collection, source commit, base image, and digest.*
