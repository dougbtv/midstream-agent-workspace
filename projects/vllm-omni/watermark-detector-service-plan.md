# INFERENG-11526 — Watermark detector service implementation and launch plan

**Status:** proposed; do not deploy until the approval gates pass.  
**Scope:** detector evidence for **text, audio, and image** in the vLLM-Omni
0.30 carry. **Video is out of scope:** do not add a video API, dependency,
capability, test, or deployment claim.

## Baseline and provenance

* Tarun Kumar's experimental FastAPI handoff is
  `scratch/INFERENG-11526/watermark-detect-api`, archive SHA-256
  `d996121c2528f344ac5fe485ecafd85dfa15d15d74f984a7b8d25a3732bddc4f`.
  It is adapted from a vLLM Apache-2.0 example, preserves SPDX attribution, and
  reportedly passed vLLM 0.30 text positive/negative controls. It is neither
  maintained source nor infrastructure.
* `app/server.py:33-120` accepts request key and arbitrary tokenizer and returns
  both; it has static `/health`, no auth/limits/readiness/admission, and a
  mutable-tag vLLM base image. Its bundled chart points at Tarun's personal Quay
  image and Route. None is a production baseline.
* Text parity must reuse `GumbelWatermarkDetector`
  (`repos/vllm/nm-vllm-ent/vllm/v1/watermarking/gumbel.py:180-205`), including
  known-vector semantics from `tests/watermarking/test_detection.py:75-90`
  (`philox`, width 4, key 42).
* Omni config keeps text at vLLM sampling time and registers non-text algorithms
  for post-processing (`nm-vllm-omni-ent/vllm_omni/config/watermarking.py:17-51`).
  Audio is CPU-loaded, mono-only AudioSeal with a `>0.5` decision
  (`watermarking/audio_seal.py:39-57,122-148`); image uses CPU-loaded
  VideoSeal image detection (`watermarking/video_seal.py:112-246`).
* The 2026-10-14 release record says PR #61 merged the carry into the 0.30
  handoff/tag `v0.30.0+rhaiv.0` at `698fe82c`. Verify the selected
  base-vLLM/Omni release refs before pinning; local checkout state is not
  release authority and must not be cleaned.
* The proposed nm-cicd PR #1401 remains an open draft at
  `doug/INFERENG-11432-omni-watermark` / `bf669402`, not main.
  `neuralmagic/ocp_model_deployment/watermark.py:36-82` needs an explicit
  URL, bearer header, and boolean `detected`; its tests define text/audio/image
  controls and redacted JSONL evidence. It is a compatibility target pending
  merge/rebase, not released behavior.
* Stratus has declarative Deployment/Service, TLS/OAuth-proxy, and readiness
  patterns in `ibm/infra/redhat-ai/nm-upstream-wdc-canary-openshift/maas-bodega/`
  (`02-deployment.yaml:7-79`, `03-service.yaml:6-18`), and its
  `.github/workflows/build-image.yml:4-19,96-118` defaults Quay push to false.

## Proven cluster and credential path

The deployment target is the active IBM A100 OpenShift cluster at
`https://api.modelsibm.ibmmodel.rh-ods.com:6443`. The supported credential is
GCP Secret Manager secret `ocp_a100_cluster` in project
`it-cloud-gcp-nm-automation`, using `htpasswd-cluster-admin-user`; do not use
the stale `openshift-mi300x-user-cred` secret. This mapping is documented in
`rel-eng-faq` branch `origin/doug/ocp-cluster-docs` at `34e2dad` and by
`stratus/misc/a100-tarun-os/README.md`.

Access was live-proved on 2026-10-14: the API reports Kubernetes `v1.33.9`,
`oc whoami` returns the documented user, namespace `takumar-rhaiis` exists, and
the credential can create Deployments, Routes, and Secrets there. The context
was merged into `/home/dev/.kube/config`; the prior Stone context was preserved
and the pre-merge config is backed up at
`/home/dev/.kube/config.pre-modelsibm-20261014T193918Z`.

Tarun's live resources confirm the handoff state: Helm-managed Deployment,
Service, and admitted Route named `watermark-detect`; the Deployment is scaled
to zero and uses mutable image `quay.io/takumar/watermark-detect:v30-api`.
The Route is plain HTTP and the application has no detector authentication.
These resources are inspection evidence only. Cluster access is no longer a
blocker; the remaining G3 decision is a new Stratus-owned namespace/topology
versus an explicitly temporary canary in `takumar-rhaiis`.

## Recommended ownership

| Concern | Source of truth | Default |
| --- | --- | --- |
| Application, image workflow, contract client, tests | `neuralmagic/nm-cicd` | Add `services/watermark-detector/` beside validation client: this is validation infrastructure. |
| Algorithm correctness | vLLM/vLLM-Omni component owners | Reuse Gumbel, AudioSeal, VideoSeal-image semantics; never reimplement. |
| Namespace, manifests, TLS, network, RBAC, operations | Stratus | Dedicated declarative component in approved environment, not the handoff chart/route. |
| Evidence | nm-cicd #1401 successor / INFERENG-11432 | Explicit runtime URL, secret-free artifacts. |

Preserve Tarun attribution in `services/watermark-detector/NOTICE`, retain copied
SPDX headers and upstream Apache-2.0 notices. Before shipping AudioSeal/VideoSeal,
record exact package versions, licenses, and model/checkpoint terms in SBOM/license
review; do not assume the Omni repository license covers them.

## Normalized contract and legacy behavior

**Canonical endpoint:** `POST /v1/detect`, TLS plus
`Authorization: Bearer <service credential>`.

```json
{"modality":"text","text":"...","profile":"text-v1"}
{"modality":"audio","artifact_b64":"<canonical WAV bytes>","profile":"audioseal-v1"}
{"modality":"image","artifact_b64":"<PNG bytes>","profile":"videoseal-image-v1"}
```

Every success returns:
```json
{"detected":true,"detector":"gumbel|audioseal|videoseal-image","version":"build-or-package-version","score":0.0,"threshold":0.0,"configuration_id":"non-secret-profile-and-model-id"}
```
`detected` is always a JSON boolean. `score`/ `threshold` may be null only
when the detector has no meaningful scalar. Never log or return request text,
artifacts, bearer/Gumbel keys, sensitive tokenizer data, or model contents.

Use stable Problem Details-style errors with `code`, safe `message`, and
`request_id`: 400 malformed/unsupported media, 401/403 auth, 413 limits, 422
semantic/profile failure, 429 admission, 503 unavailable/unready, 500 unexpected
fault.

Profiles are server-selected immutable records:
* `text-v1`: tokenizer allowlist plus server-side Gumbel key, `philox`,
  width 4, threshold 0.01 unless the recorded serving tuple differs.
* `audioseal-v1`: WAV only initially; bounded bytes/duration/samples, mono
  required, release-pinned AudioSeal detector. Reject stereo; do not downmix.
* `videoseal-image-v1`: PNG only initially; bounded bytes/pixels/dimensions
  before tensor work, release-pinned image detector. No video profile/MIME type.

Keep legacy `POST /detect` for one agreed deprecation window. It accepts historic
`text`, `tokenizer`, and `key` fields but maps only to `text-v1`. Under TLS and
service bearer authentication, the supplied tokenizer/key must exactly match
the configured profile; mismatches and PRF/width/threshold overrides return
422. The service must redact and never persist those legacy request fields.
Return canonical fields plus
`is_watermarked == detected`, `p_value`, and `num_scored_tokens`, never
`key`. New nm-cicd code calls `/v1/detect`; a legacy response adapter expires
on the agreed date. Per-request development keys may exist only in a build
incapable of Stratus deployment.

## Proven #1401 auth/key/tokenizer gap

`WATERMARK_DETECTOR_KEY` in #1401 is only the service bearer credential.
`WATERMARK_TEXT_KEY` is separately supplied to the generator to emit matching
Gumbel text. Neither authenticates or configures the other. The service loads
the same Gumbel algorithm key from an independently referenced OpenShift Secret
and loads tokenizer/profile from configuration; the canonical harness sends
neither to `/v1/detect`. Provisioning must place the matching algorithm value in
the GitHub Actions `WATERMARK_TEXT_KEY` secret and the detector's OpenShift
Gumbel-key Secret without conflating either with the bearer credential.

Before merge, prove with wiring tests: missing/wrong bearer is rejected; valid
bearer cannot select another tokenizer/key; a generated sample using the
matching configured algorithm key detects; a wrong-key sample does
not; logs/evidence contain neither secret. This is required to close the
service-auth versus detector-semantic gap.

## Design and early image feasibility gate

Implement typed ASGI routing, profile registry, detector adapters, bounded lazy
load with per-profile async lock, fixed LRU byte/entry budgets, CPU semaphore,
short queue deadline, and graceful shutdown. `/livez` does no model work;
`/readyz` is true only after required warmup/readiness. Run non-root, with
explicit resources and a bounded writable cache volume.

Do **not** assume one CPU image is viable:
1. Build pinned text-only and audio/image candidates from exact selected 0.30
   dependencies. The Omni extra is `audioseal>=0.2,<1.0` and
   `videoseal>=1,<2.0` (`pyproject.toml:36-41`). Generate lockfiles, SBOM,
   CVE, license and model-term review.
2. Measure compressed size, cold/warm start, RSS, CPU latency, model
   download/cache behavior, and offline restart under proposed OpenShift limits;
   record wheel/checkpoint digests.
3. Combine only if approved resource, pull-time, security/license, and
   reproducibility budgets pass. Otherwise deploy split text/audio/image
   services behind the same external contract/profile routing.

**Default is split images/services until this gate passes.** Text, AudioSeal, and
VideoSeal have separate models/native dependency risk; never include VideoSeal's
video detector.

## Phased implementation plan

1. **Adopt source in nm-cicd.** Add `services/watermark-detector/`:
   `app/{main,api,profiles,auth,limits}.py`,
   `app/detectors/{gumbel,audioseal,videoseal_image}.py`, `pyproject.toml`,
   lockfile, `Dockerfile`, `docker-bake.hcl`, `NOTICE`, `README.md`;
   add `tests/services/watermark_detector/`. Cite handoff checksum; do not
   copy its Helm chart.
2. **Text parity.** Use selected base-vLLM detector and explicit tokenizer
   allowlist/profile map. Add known vectors and real 0.30 positive/negative
   integration. Never permit request-driven remote tokenizer download.
3. **Adapters.** Reuse AudioSeal/VideoSeal-image CPU, mono/native-rate, safe-PNG,
   and detector semantics. Pin or pre-stage model artifacts; expose non-secret
   profile/model identity as `configuration_id`.
4. **Hardening.** Add rotation-friendly bearer references with constant-time
   compare, global body limit before base64 decode, decoded modality limits,
   request IDs/redacted logs, probes, cache/semaphore/timeout. Assert redaction.
5. **Land #1401 successor.** Rebase/update
   `neuralmagic/ocp_model_deployment/watermark.py`,
   `tests/unit/test_watermark.py`, `tests/model_deployment/test_watermark.py`,
   `.github/actions/ocp-test/action.yml`, `.github/workflows/ocp-test.yml`,
   and `model_registry.yml`. Default `/v1/detect`, fail closed without URL,
   wire bearer and generator key independently, test artifact secrecy.
6. **Build candidate.** Add `.github/workflows/watermark-detector-image.yml`
   for unit/integration/SBOM/license/image scan. First use Stratus workflow with
   `push=false`; after approval publish exactly one initial
   `quay.io/dosmith/...` candidate, resolve it to immutable `@sha256:...`,
   and deploy only the digest. Record source SHA, SBOM, scan, tag, digest.
7. **Stratus deployment.** Add
   `ibm/infra/redhat-ai/<approved-openshift-env>/watermark-detector/` with
   `01-serviceaccount-rbac.yaml`, `02-secrets-reference.yaml`,
   `03-configmap.yaml`, `04-deployment-{text,audio,image}.yaml`,
   `05-service.yaml`, `06-route-or-private-ingress.yaml`,
   `07-networkpolicy.yaml`, and `08-monitoring.yaml`. Do not modify
   maas-bodega or the experimental chart.
8. **Canary/enablement.** Render and dry-run, then private digest-pinned canary,
   run the matrix below, grant CI bearer access, and enable each modality. A
   public route is not default.

## Security and operational requirements

TLS and application bearer auth behind TLS are the default headless-CI pattern
because they fit #1401. Store bearer, Gumbel, registry/artifact, and TLS values
as independent Secret keys with least-privilege service accounts; manifests name
references only. Rotate bearer and Gumbel secrets independently. OAuth proxy may
serve humans but cannot replace compatible CI machine auth.

Default exposure is ClusterIP/private ingress, NetworkPolicy limited to approved
CI namespaces/DNS/approved artifact egress, resource requests/limits, PDB when
replicas allow, anti-affinity when available, non-root/read-only filesystem where
viable, scoped cache, and alerts for 5xx/429/readiness/latency/memory/model/auth
failure. Pin base image and artifacts by digest/version; eliminate runtime
internet downloads after staging.

Use `helm lint`/ `helm template` only to inspect the archival handoff. After
human authorization, use `oc apply --dry-run=server`; cluster mutation remains
human-run. Never send credentials as Docker build args.

## Validation matrix

| Layer | Positive control | Negative/failure control | Evidence |
| --- | --- | --- | --- |
| Unit/API | every profile returns boolean/version/config ID | malformed JSON/base64, oversized/Png bomb, stereo/corrupt WAV, auth/profile/tokenizer/key override, saturation | focused pytest, status and redaction assertions |
| Text | matching 0.30 profile generation detects true | watermark false, wrong-key fixture, empty/short defined non-detection | vectors, real client, config/image IDs |
| Audio | watermarked mono WAV true | clean WAV false; stereo/corrupt/over-limit reject | AudioSeal CPU/native-rate/model ID |
| Image | watermarked PNG true | approved clean image false; corrupt/oversize/non-PNG reject | round-trip/fixture provenance |
| INFERENG-11432 | `pytest tests/unit/test_watermark.py`; focused `pytest tests/model_deployment/test_watermark.py -m smoke` | timeout/non-JSON/missing boolean/401/403/bad URL; no secrets in artifact | JSONL expected/result, version/config, source/run/image digest/model |
| Image/deploy | digest candidate ready/authorized | missing secret/pull/cache, restart, egress denial, wrong bearer | SBOM/license/scan/digest, server dry-run, canary probes/metrics/cleanup |

The old reference command remains non-production text-only validation:
`python test_local.py --chat-url <vllm>/v1 --detect-url <detector> --model <model> --key <key>`.
It must not send a production key to the new service.

## Exit, rollback, and approval gates

**INFERENG-11526 exit:** accepted ownership; selected 0.30 refs/profiles pinned;
feasibility/security/license gates passed; approved `quay.io/dosmith` candidate
digest recorded; Stratus private TLS/authenticated digest deployment; text/audio/
image controls pass; release record includes manifest revision, digest, profile
IDs, artifacts, monitoring and rollback owner. No video surface exists.

**INFERENG-11432 exit:** harness merged/rebased to main; explicit URL fails
closed; service bearer and generator Gumbel key are independent; normalized
response/redacted evidence proven; approved text/audio/image matrix—including
human-approved image negative—passes.

Rollback: disable capability/CI input, scale affected deployment to zero or
revert declarative manifest to previous known-good immutable digest, revoke/rotate
bearer, retain non-secret evidence. Never move a tag. Ephemeral validation cleanup
removes only approved namespace-scoped workload/service/route/PVC objects; retain
digest/SBOM/evidence per policy, revoke temporary secret, and confirm no personal
Tarun image/route reference remains.

| Gate | Required decision | Approvers |
| --- | --- | --- |
| G0 ownership | nm-cicd source + Stratus deploy; named component/operator owners | Midstream lead, Stratus, component owner |
| G1 parity | exact Omni/base-vLLM refs, profiles, tokenizer/threshold/model artifacts | vLLM/vLLM-Omni SME |
| G2 feasibility/legal | split/combined result, dependencies/model terms | security/legal, platform, component owner |
| G3 topology | IBM A100 cluster is selected and access-proved; approve namespace, TLS/ingress, network/egress, CI identity, and secret rotation/OAuth | Stratus, security |
| G4 semantics | image clean negative, audio/image thresholds, legacy sunset | component, validation owner |
| G5 release | Quay repo, SBOM/scan disposition, digest, canary, monitoring/rollback owner | release, security, Stratus |

An unapproved gate is a stop sign: do not invent credentials, expose a public
route, push an image, or broaden scope into video.
