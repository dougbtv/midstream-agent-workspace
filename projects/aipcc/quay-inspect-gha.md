# Quay remote-inspect GHA (INFERENG-12106)

Plan for a dispatchable GitHub Actions workflow in `neuralmagic/nm-cicd` that
performs AIPCC Quay registry operations remotely, so neither humans nor agents
pull multi-GB CUDA images locally.

- Jira: [INFERENG-12106](https://redhat.atlassian.net/browse/INFERENG-12106)
- Status: In Progress (plan phase)
- Created: 2026-11-13

## Problem

We repeatedly need image metadata from `quay.io/aipcc/*` during z-stream
rotation (e.g. INFERENG-10471) and release work (Omni 3.6 GA): manifests,
digests, tags, labels, CVE reports. Engineers and agents generally lack direct
Quay credentials for the namespace, but nm-cicd GitHub Actions already hold
working credentials: `QUAY_AIPCC_USERNAME` / `QUAY_AIPCC_SECRET` (proven by
`trivy-scan.yml` and `snyk-image-security-scan.yml`). Pulling a 25 GiB image
locally to read one label is waste; the interaction should happen on a runner
and return results.

## Scope inventory

Surveyed rhaiis-midstream-snippets skills, workspace skills/runbooks, and
nm-cicd workflows (2026-11-13). Operations cluster in three tiers.

### Tier 1 — registry metadata, no pull (v1 core)

All via the registry protocol (skopeo/crane), seconds of runtime, plain
hosted runner:

- `inspect`: `skopeo inspect` with `--override-os linux --override-arch amd64`
  (multi-arch indexes are the norm). Returns labels (notably
  `com.redhat.aiplatform.index_url`), env, digest, created date.
- `inspect-raw`: `skopeo inspect --raw` — manifest / image index, platform
  child-digest resolution.
- `list-tags`: `skopeo list-tags` with a tag-prefix filter; supports the
  "newest epoch-suffixed tag for release prefix" convention (epoch decode
  included in output).
- `digest`: tag-to-digest pinning for release-gate scans (tags are mutable;
  every gate scan must be pinned by digest).

Consumers today: downstream-release-status, fix-cves, rhaiis-release-branch,
rhaiis-stage-rc-release, shepherd-wheel-to-image, github-runner-operations,
runbooks/aipcc/vllm-omni.md.

### Tier 2 — Quay JSON API (api/v1)

- `cves`: Clair security report per manifest digest (the sign-off source for
  the CVE gate). Port the logic of `quay-fetch-cves.sh` (fix-cves skill).
- `repo-check`: repository existence and `specificTag` query
  (release-vllm-image pattern).

Auth nuance (documented in fix-cves): the Quay **tag-listing API rejects
registry tokens** (`401 invalid_token`) while the **manifest security
endpoint accepts the same credential as HTTP Basic auth**. Consequence for
design: route all tag operations through the registry protocol via skopeo;
use the API only for the security endpoint. This sidesteps the
`invalid_token` failures seen with local `quay-fetch-cves.sh`.

Known Clair limitations to carry into output handling (from fix-cves):
index-level `status: unsupported` is inconclusive (resolve platform child
digest); `manifest_layer_too_large` means no Clair report (fall back to the
existing `trivy-scan.yml`); Clair vs trivy severity ratings differ — the CVE
gate treats the union of fixable High/Critical from both scanners as the
gate.

### Tier 3 — image contents (pull on a runner; v1.5)

- `exec-in-image`: run a command inside the image (`pip list`, `pip show`) to
  check packaged versions (deep_gemm, flashinfer); file-existence checks
  (e.g. `structured_server.py`).
- Requires self-hosted dind runners (`k8s-a100-dind`, `ibm-wdc-k8s-h100-dind`)
  — hosted runners lack disk for 25 GiB CUDA images. Prefer
  `ibm-wdc-k8s-h100-dind` (the snyk workflow has documented rootless-dockerd
  failures on `k8s-a100-dind`).

### Already covered — do not duplicate

`trivy-scan.yml`, `snyk-image-security-scan.yml` (CVE scans), `rapidast`,
`fips-compliance`, `model-validation-ocp.yml` (GPU validation), and the
build/publish workflows (`build-image`, `build-validate-publish`, `release`,
`omni-pipeline`). All already dispatch with QUAY_AIPCC credentials.

### Out of scope for v1

GPU serving validation (stays on runner hosts / model-validation-ocp) and
`skopeo copy` cross-registry mirroring (write path, different risk class).

## Recommended design

One operation-parameterized workflow — not one workflow per operation:

- File: `.github/workflows/quay-inspect.yml` in `neuralmagic/nm-cicd`
- Triggers: `workflow_dispatch` + `workflow_call`
- Inputs:
  - `image` (required): tag or `repo@sha256:...` digest reference
  - `operation`: `inspect` | `inspect-raw` | `list-tags` | `digest` | `cves`
    | `repo-check`
  - `tag-prefix` (optional): filter for `list-tags` / newest-tag resolution
  - `platform` (default `linux/amd64`)
  - `build_label` (runner selection, mirroring trivy-scan's pattern)
- Auth: `QUAY_AIPCC_USERNAME` / `QUAY_AIPCC_SECRET`; registry protocol for
  tags, API only for the security endpoint
- Output: JSON artifact per operation plus a human-readable job summary;
  agent consumption via `gh run watch` / `gh run download`
- Rationale for a single dispatch surface: an agent calling one workflow with
  an `operation` input is easier to get right than five overlapping
  workflows.

## Follow-ups (post-v1)

- Composite action in `nm-actions` embedding login + inspect for workflows
  that need it inline.
- Update rhaiis-midstream-snippets skills (downstream-release-status,
  fix-cves, release-vllm-image, shepherd-wheel-to-image,
  github-runner-operations) to dispatch the GHA instead of instructing local
  skopeo/docker pulls.
- Repo/tag listing for the aipcc namespace via the Quay API, if a credential
  that the API accepts becomes available.

## Validation / acceptance

- Workflow merged to nm-cicd `main`.
- Dispatch `inspect` and `list-tags` against a real `quay.io/aipcc/*` image:
  labels and digest returned without a local pull.
- Dispatch `cves` against a digest with a known Clair report; output matches
  the Quay security tab.
- Results downloadable via `gh` CLI from a machine with no quay credentials.
- At least one rhaiis-midstream-snippets skill updated to use it.

## First step

Smoke-test the credential before writing YAML: confirm
`QUAY_AIPCC_USERNAME`/`QUAY_AIPCC_SECRET` work for `skopeo inspect` against a
specific aipcc repo from a scratch GHA (or an existing dispatchable workflow
extended with a step). The creds are proven for docker login + pull and for
trivy; fresh skopeo inspect is expected but unproven.
