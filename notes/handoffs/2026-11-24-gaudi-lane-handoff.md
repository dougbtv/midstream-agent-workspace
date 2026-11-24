# Gaudi Lane Handoff — RHAIIS 3.5.4 (2026-11-24)

**For:** agent execution (Doug's request, Tuesday 2026-11-24 ~16:00 UTC)
**Card:** [INFERENG-12292](https://redhat.atlassian.net/browse/INFERENG-12292)
"[3.5.4 Gaudi] Address CVEs, build downstream image and validation" —
child of epic [INFERENG-10471](https://redhat.atlassian.net/browse/INFERENG-10471),
label `ms-release`, assigned to Doug.

## Objective

Take the 3.5.4 Gaudi lane from its current state to a released,
CVE-scanned, validated image — matching the bar the other five lanes
already cleared (see `projects/vllm/3.5.4-z-stream.md` for their
evidence patterns).

## Current state (investigated 2026-11-24)

- **No 3.5.4 Gaudi image exists** — `quay.io/aipcc/rhaiis/gaudi-ubi9`
  has no `3.5.4-*` tags.
- **The conf is stale**: `build-args/gaudi-ubi9.conf` on `3.5` was last
  touched 2026-10-30 (INFERENG-11830, 3.5.6 base). It **missed the
  3.5.7 base refresh** (85e1c28, 2026-11-11, which updated the other
  variants) and still carries 3.5.3-era wheel inputs.
- **Open MR**: containers !1764 (renovate, `renovate/3.5-gaudi` → `3.5`)
  bumps `RHAIIS_VERSION=3.5.3 → 3.5.4` only — it does NOT align the base
  image or wheel release.
- **Live risk**: [INFERENG-12175](https://redhat.atlassian.net/browse/INFERENG-12175)
  (In Progress) — "HPU midstream release workflow failing — Gaudi SDK
  404 from Habana vault". Check its state before relying on any
  Gaudi-involving pipeline.

## Procedure (following the established 3.5.4 lane patterns)

1. **Align the conf.** Review/merge !1764 for the version bump, then
   bring `gaudi-ubi9.conf` the rest of the way: 3.5.7 base-image pins
   (follow what 85e1c28 did for the other variants) and the
   `WHEEL_RELEASE` input for the Gaudi wheel release. Precedents: the
   base refresh MR !1701 and the CPU/CUDA conf bumps (!1725/!1730).
   Verify a green Konflux on-PR build before merging.
2. **Cut the release tag** at the merged `3.5` head:
   `rhaiis-gaudi-v<YYYYMMDD><NN>`. Before cutting, check
   `git ls-remote --tags origin` for same-day collisions — the daily
   sequence is a shared namespace across release lines. Read on-push
   wiring from `git show origin/3.5:.tekton/...`, not the working tree
   (it sits on a parallel branch).
3. **Konflux build + release** → image lands in
   `quay.io/aipcc/rhaiis/gaudi-ubi9` as `3.5.4-<timestamp>`. Konflux has
   had intermittent release-path issues; if builds go green but no image
   lands, check the platform status before re-triggering (never move an
   existing tag — cut a new one).
4. **CVE scans**: dispatch `quay-inspect` with `operation=cves` from the
   nm-cicd PR #1691 branch (`doug/INFERENG-12106-quay-inspect-impl`) —
   **main's `cves` operation is a stub** and returns no data. If Quay
   reports `manifest_layer_too_large`, fall back to the `trivy-scan`
   workflow (`scanners=vuln`). Append the per-variant CVE list to
   `projects/vllm/3.5.4-rc-cves.md`.
5. **Validation**: follow the Gaudi validation pattern from the 3.6/3.7
   lines (e.g. [INFERENG-11964](https://redhat.atlassian.net/browse/INFERENG-11964)
   "[3.7.0 Gaudi] Midstream validation and stage RC" as template). Check
   with the release owner (Selbi) what validation depth 3.5.4 Gaudi
   requires for a z-stream. HPU-specific: watch for the 12175 SDK-404
   failure mode.
6. **Evidence and close-out**: digest-pin the image on INFERENG-12292
   with the validation evidence and scan run links; transition to
   Review; update the z-stream map's status section; add the image to
   the epic's RC list and the `@aipcc-productization` announcement
   block.

## Key references

- Working map: `projects/vllm/3.5.4-z-stream.md` (the release notebook)
- CVE lists note: `projects/vllm/3.5.4-rc-cves.md`
- Epic: INFERENG-10471 (overall update + proposed RCs in comments)
- Workspace: `codebase/midstream-agent-workspace/` (containers repo at
  `repos/aipcc/containers`, remote `redhat/rhel-ai/rhaiis/containers`)
- Jira CLI: `acli` (authenticated); GitLab: `glab`; GitHub: `gh`
  (nm-cicd at `neuralmagic/nm-cicd`)

## Known gotchas (from this release's ledger)

- ADF tables in Jira comments bounce `INVALID_INPUT` — use bullets.
- `gh run view --log` misses artifact-borne results — CVE/trivy data
  lands in run artifacts, download with `gh run download`.
- Konflux on-push triggers are tag-based; the run-level GHA status can
  wedge at "queued" — job-level REST data is the ground truth.
- Don't re-cut or move existing release tags without Doug's word.

*Compiled 2026-11-24 by Rand. All state facts verified live the same day.*
