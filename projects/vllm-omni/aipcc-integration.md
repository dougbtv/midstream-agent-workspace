# vLLM-Omni × AIPCC — Working Doc

Status: **In Progress — constraints reconciliation** — updated 2026-07-22

Companion to [MIDSTREAM_OMNI_BUILD.md](./MIDSTREAM_OMNI_BUILD.md) (build pipeline) and [OMNI_TECH_PREVIEW.md](./OMNI_TECH_PREVIEW.md) (TP scope).

## What Is This

Documents everything we know about how vLLM-Omni flows through AIPCC's four-layer architecture: what exists, what's missing, where constraints live, and how to coordinate. This is the reference for anyone on midstream who needs to understand the AIPCC side of omni productization.

## AIPCC Four-Layer Architecture

From Daniele Trifirò (July 17, 2026):

```
Layer 1: Builder        → builds Python wheels from source (fromager)
Layer 2: Pipeline repos → build wheel collections per arch/accelerator, publish to GitLab index
Layer 3: Containers     → Containerfiles that install wheel collections + env vars/entrypoints
Layer 4: Base images    → minimal deps for installing builder-built wheels (CUDA SDK, etc.)
```

**Ownership:** AIPCC handles builder/pipeline/base-images. Midstream owns the container repository.

### Key Repos

| Repo | GitLab Path | Purpose |
|------|-------------|---------|
| builder | `redhat/rhel-ai/wheels/builder` | Fromager-based wheel builder — package plugins, overrides, collections |
| pipeline | `redhat/rhel-ai/rhaiis/pipeline` | Production wheel collections — requirements + constraints per variant |
| containers | `redhat/rhel-ai/rhaiis/containers` | Containerfiles, build-args, Tekton pipeline config |
| base-images | `redhat/rhel-ai/core/base-images/app` | Base container images (AIPCC-owned) |
| infrastructure | `redhat/rhel-ai/core/infrastructure` | Product onboarding — `data/products/rhaiis.yml` |
| nm-vllm-omni-ent mirror | `redhat/rhel-ai/core/mirrors/github/neuralmagic/nm-vllm-omni-ent` | GitLab mirror of our source repo |

## Layer 1: Builder — DONE

AIPCC Ecosystems team (Andre Lustosa, Shaun Walsh) completed full onboarding in June 2026.

### What Exists

**Package plugin** — `package_plugins/vllm_omni.py`:
```python
VLLM_OMNI_REPO = "redhat/rhel-ai/core/mirrors/github/neuralmagic/nm-vllm-omni-ent"

def get_resolver_provider(...):
    return resolver.GitLabTagProvider(
        project_path=VLLM_OMNI_REPO,
        matcher=create_midstream_matcher(),
        constraints=ctx.constraints,
    )
```

Fetches vllm-omni source from the GitLab mirror of nm-vllm-omni-ent. Uses `create_midstream_matcher()` to match our `+rhaiv.X` tag pattern. Passes build constraints through.

**Overrides** — `overrides/settings/vllm_omni.yaml`:
```yaml
annotations:
  aipcc.component: Development Platform
changelog:
  "0.21.0+rhaiv.1":
    - "2026-06-09: initial build for vllm-omni from midstream fork"
download_source:
  destination_filename: vllm-omni-${version}.tar.gz
resolver_dist:
  include_sdists: false
  include_wheels: true
  min_release_age: 0
env:
  VLLM_OMNI_VERSION_OVERRIDE: "${__version__}"
variants:
  cpu-ubi9:
    env:
      VLLM_OMNI_TARGET_DEVICE: cpu
  cuda-ubi9:
    env:
      VLLM_OMNI_TARGET_DEVICE: cuda
```

**Builder collections** — vllm-omni is listed in:
- `collections/torch-2.11.0/cpu-ubi9/requirements.txt`
- `collections/torch-2.11.0/cuda12.9-ubi9/requirements.txt`
- `collections/torch-2.11.0/cuda13.0-ubi9/requirements.txt`

All dependency packages also onboarded: cache-dit, x-transformers, janus, openai-whisper, onnxruntime, sox, resampy, imageio-ffmpeg, fa3-fwd, torchsde.

### AIPCC Tickets (All Closed)

| Ticket | What | Status |
|--------|------|--------|
| AIPCC-16724 | vllm-omni package request (parent) | Closed (June 29) |
| AIPCC-16726 | vllm-omni onboarded into builder | Closed |
| AIPCC-16727 | Probe tests created | Closed |
| AIPCC-16728 | Added to RHAI pipeline onboarding collection | Closed |
| AIPCC-16729 | QE testing passed | Closed |
| AIPCC-18030 | nm-vllm-omni-ent mirrored to GitLab | Closed |
| AIPCC-14176/14178/14181 | cache-dit onboarding | Closed |
| AIPCC-14160 | x-transformers | Closed |
| AIPCC-14161 | janus | Closed |
| AIPCC-14162 | openai-whisper | Closed |
| AIPCC-14163 | onnxruntime | Closed |
| AIPCC-14164 | sox | Closed |
| AIPCC-14165 | resampy | Closed |
| AIPCC-14166 | imageio-ffmpeg | Closed |
| AIPCC-14184 | fa3-fwd | Closed |
| AIPCC-14185 | torchsde | Closed |

## Layer 2: Pipeline — PARTIALLY DONE (The Gap)

The builder can produce vllm-omni wheels, but the **shipping pipeline collection** doesn't include vllm-omni yet.

### What Exists

The pipeline repo (`redhat/rhel-ai/rhaiis/pipeline`) has standard RHAIIS collections:

```
collections/rhaiis/cpu-ubi9/
collections/rhaiis/cuda13.0-ubi9/
collections/rhaiis/gaudi-ubi9/
collections/rhaiis/neuron-ubi9/
collections/rhaiis/rocm7.14-ubi9/
collections/rhaiis/spyre-ubi9/
collections/rhaiis/tpu-ubi9/
```

Each variant has three files:
- `requirements.txt` — direct dependencies (e.g., `vllm[audio,tensorizer,cgraph-cuda13]==0.24.0+rhaiv.2`)
- `constraints.txt` — hand-written pins for specific dep versions
- `constraints-rules.txt` — rules for pulling constraints from the builder

### What's Missing

**`requirements.txt`** — no `vllm-omni` line in any variant. The current CUDA requirements include standard vLLM, flashinfer, vllm-bart-plugin, xformers, numba, etc. — but no omni.

**`constraints.txt`** — no omni-specific constraints. Current content is generic (aiohttp CVE, boto3 pin, terrakit conflict, numba pin). Our midstream constraints (fastapi, transformers) are not here.

**Decision needed:** Does vllm-omni go into the existing `collections/rhaiis/` collection, or does it need its own collection (e.g., `collections/vllm-omni/`)? The pipeline API docs say "Wheel collections should not be reused between products" — so if omni is a standalone product, it needs its own collection.

### How Constraints Flow

```
Builder builds wheels
  → generates constraints as side-effect (version pins for everything it resolved)
        ↓
Pipeline constraints-rules.txt:
  "torch-2.11.0 *"
  → pulls ALL constraints from builder's torch-2.11.0 collection
        ↓
Pipeline constraints.txt:
  hand-written ADDITIONAL pins (CVE fixes, conflict resolution)
        ↓
Final constraints = builder-generated + hand-written
```

The constraints API is documented at `pipeline-api/README.md` in the builder repo. Rule syntax: `collection_name package_name` (wildcards supported). The pipeline currently pulls from `torch-2.11.0` for all packages.

### Active Bug: Version Mismatch (AIPCC-21515)

AIPCC's AutoQA nightly flags import failures: vllm-omni 0.21.0+rhaiv.3 tries to import `supports_xccl` from `vllm.utils.torch_utils`, which doesn't exist in vLLM 0.24.0+rhaiv.1. AIPCC's index has "latest of each" but the two packages must be version-paired. Our midstream builds manage this explicitly via the two-wheel strategy. AIPCC's test harness installs them independently — needs coordination on version pinning.

## Existing Precedent: model-opt as a Standalone Collection

The `model-opt` collection in the AIPCC pipeline is the closest existing precedent for how vllm-omni should be structured as a standalone product. It ships separately from standard RHAIIS but lives in the same pipeline and containers repos.

### Pipeline Layer — Own Collection, Own Constraints

```
collections/model-opt/cuda13.0-ubi9/
├── requirements.txt          # llmcompressor==0.12.0, speculators==0.6.0 (NOT vllm)
├── constraints.txt           # its own pins (transformers>=5.9.0,<=5.10.1, numpy, accelerate, etc.)
└── constraints-rules.txt     # same "torch-2.11.0 *" builder pull as rhaiis
```

Key observations:
- **Separate requirements** from standard RHAIIS — different packages entirely
- **Own constraints** — model-opt pins transformers to a *different* range than what rhaiis would need. This is exactly why separate collections matter: conflicting constraints can coexist.
- **Same constraints-rules mechanism** — pulls builder-generated constraints from `torch-2.11.0 *` just like rhaiis does

### Containers Layer — Own Containerfile, Own Build-Args

```
Containerfile.model-opt.cuda-ubi9           # naming pattern: Containerfile.<product>.<variant>
build-args.model-opt/cuda-ubi9.conf         # separate dir: build-args.<product>/
```

**build-args differences** (model-opt vs standard RHAIIS):

| Field | RHAIIS (standard) | model-opt |
|-------|-------------------|-----------|
| `WHEEL_RELEASE_PACKAGE` | `rhaiis-wheels` | `model-opt-wheels` |
| `WHEEL_RELEASE_*` | `3.5-ea2.3083+rhaiis-cuda13.0-ubi9-*` | `3.5-ea1.2706+model-opt-cuda13.0-ubi9-*` |
| `BASE_IMAGE` | Same base image | Same base image |

Both share the same base image and use the same `install-wheel-release.sh` mechanism. The base image provides CUDA SDK, Python, and the install tooling — what differs is which wheel collection gets installed.

**Containerfile differences:**

| Feature | RHAIIS (standard) | model-opt |
|---------|-------------------|-----------|
| Labels | `rhaiis-vllm-cuda-rhel9-container` | `rhaiis-model-opt-cuda-rhel9-container` |
| Entrypoint | `python3 -m vllm.entrypoints.openai.api_server` | None (tools image, not a server) |
| Tiktoken pre-download | Yes | No |
| NCCL config | Yes | No |
| Chat templates | Yes (`COPY template/*.jinja`) | No |
| scanlibs ignores | flashinfer, cupy, torio | torio, cupy |
| Env vars | Full server config (HF_HUB_OFFLINE, VLLM_USAGE_*, caches) | Just NVIDIA_REQUIRE_CUDA |

### What This Means for vllm-omni (Standalone Path)

Following the model-opt precedent, the standalone vllm-omni path would look like:

**Pipeline:**
```
collections/vllm-omni/cuda13.0-ubi9/
├── requirements.txt          # vllm-omni==X.Y.Z+rhaiv.N, vllm[audio,...]==A.B.C+rhaiv.M
│                             # (both pinned together — solves AIPCC-21515 version pairing)
├── constraints.txt           # fastapi[standard] >= 0.133.0, < 0.137.0
│                             # transformers >= 5.5.3, < 5.13.0
│                             # (plus fa3-fwd, av from INFERENG-9317)
└── constraints-rules.txt     # torch-2.11.0 *
```

**Containers:**
```
Containerfile.vllm-omni.cuda-ubi9
build-args.vllm-omni/cuda-ubi9.conf        # WHEEL_RELEASE_PACKAGE=vllm-omni-wheels
```

The Containerfile would be closer to standard RHAIIS than to model-opt (it's a server, needs entrypoint, NCCL config, caches), but with:
- `vllm serve --omni` or equivalent entrypoint
- omni-specific labels (`rhaiis-vllm-omni-cuda-rhel9-container`)
- Multimedia system deps (libsndfile) if not in base image
- Additional scanlibs ignores for omni-specific packages

## Existing Precedent: Neuron/TPU as Hardware Variants

The neuron and TPU collections in the AIPCC pipeline represent the **variant pattern** — they live inside the same `collections/rhaiis/` collection as standard CUDA, sharing the same product identity and infrastructure.

### Pipeline Layer — Same Collection, Per-Variant Files

```
collections/rhaiis/
├── cuda13.0-ubi9/          # standard CUDA
├── cpu-ubi9/               # CPU
├── rocm7.14-ubi9/          # ROCm
├── neuron-ubi9/            # ← variant, same collection
├── tpu-ubi9/               # ← variant, same collection
├── gaudi-ubi9/             # ← variant
└── spyre-ubi9/             # ← variant
```

Each variant has its own `requirements.txt`, `constraints.txt`, and `constraints-rules.txt` — so they can have **completely different packages and pins**. They're separate builds that just happen to live under the same collection umbrella.

**Neuron's constraints are massive** — `collections/rhaiis/neuron-ubi9/constraints.txt` (abridged):
```
vllm==0.16.0+rhaiv.12
vllm-neuron==0.5.1
transformers<5
torch==2.9.1
torch-xla==2.9.0
torch-neuronx==2.9.0.2.14.27725+e2ff0410
libneuronxla==2.2.16974.0+a550bfe0
neuronx-cc==2.25.3371.0+f524f7f8
# ... 15+ exact pins for the full Neuron SDK
```

Neuron pins vLLM to 0.16.0 while CUDA is on 0.24.0. Different transformers constraint (`<5` vs unconstrained). Completely different torch version. These constraints **don't bleed across variants** — each variant is built independently.

**TPU constraints** are minimal by comparison — `collections/rhaiis/tpu-ubi9/constraints.txt` just has a urllib3 CVE fix. Its `constraints-rules.txt` has `torch-2.9.1 [!v]*` commented out. Most TPU constraints come from the builder side.

### Containers Layer — Shared Infrastructure

| | CUDA | Neuron | TPU |
|---|---|---|---|
| `WHEEL_RELEASE_PACKAGE` | `rhaiis-wheels` | `rhaiis-wheels` | `rhaiis-wheels` |
| Containerfile | `Containerfile.cuda-ubi9` | `Containerfile.neuron-ubi9` | `Containerfile.tpu-ubi9` |
| build-args dir | `build-args/` | `build-args/` | `build-args/` |
| Label | `rhaiis-vllm-cuda-rhel9` | `rhaiis-vllm-neuron-rhel9` | similar |
| Product name | "RHAIIS for NVIDIA CUDA" | "RHAIIS for AWS Neuron" | similar |

All variants share the same `build-args/` directory (not `build-args.neuron/`). They share the same product identity — "Red Hat AI Inference Server" — just for different hardware.

### nm-cicd Side — Per-Device Constraints

Neuron has its own constraints file in nm-cicd (`neuralmagic/constraints/neuron.txt`):
```
torch-xla < 2.9
wheel >= 0.46.2
```

Applied via `--constraints` in `prepare_neuron()` (see `prepare.sh` lines 108-129). Standard CUDA has **no constraints file at all** — just requirements. TPU has `neuralmagic/constraints/tpu.txt` (currently empty).

## Variant vs Standalone — The Comparison

| Dimension | Variant (neuron/TPU) | Standalone (model-opt) |
|-----------|---------------------|----------------------|
| Pipeline location | `collections/rhaiis/{variant}/` | `collections/model-opt/{variant}/` |
| Wheel package name | `rhaiis-wheels` | `model-opt-wheels` |
| build-args dir | `build-args/` (shared) | `build-args.model-opt/` (separate) |
| Product identity | Same — "Red Hat AI Inference Server" | Different — "Red Hat Model Optimization Server" |
| Why it's separate | Different *hardware* | Different *product purpose* |
| Constraints isolation | Per-variant files, don't bleed across | Per-collection, fully independent |
| Infrastructure overhead | Lower — reuses existing Tekton, Pyxis, KRD | Higher — own Pyxis entry, KRD, Tekton pipelines |

## Where Does vllm-omni Fit?

This is the fork in the road. vllm-omni characteristics:

- **Same product purpose** as standard RHAIIS (inference serving) → leans variant
- **Same hardware** as CUDA (runs on NVIDIA GPUs) → doesn't fit the hardware-variant naming convention
- **Different software stack** (vllm + vllm-omni layered, not just vllm) → leans standalone
- **Different constraints** (fastapi, transformers pins) → both patterns handle this fine

AIPCC-21244's initial scoping (own Pyxis, own KRD, own Tekton) implies standalone. But that's AIPCC's initial approach — it could change if the variant path is simpler and makes more sense.

**Key tension:** Every existing variant differs by *hardware target*. vllm-omni would be the first variant that differs by *software capability* on the same hardware. That's either a category error or a natural extension, depending on how you squint.

### INFERENG-1004 — What It Actually Is

For reference, INFERENG-1004 (Tahmid, closed Sept 2025) was about creating per-device constraint files in nm-cicd to align with AIPCC's builder constraints. Daniele's vision: set `UV_CONSTRAINTS=neuralmagic/constraints/<device>.txt` so all pip installs in nm-cicd respect the same pins AIPCC uses.

In practice, only neuron and TPU got constraint files (`neuralmagic/constraints/neuron.txt`, `tpu.txt`). Standard CUDA never got one. The "pull from builder" was more about conceptual alignment than an automated sync mechanism. So for vllm-omni, we're actually ahead of standard vLLM by already having a constraints file (`neuralmagic/constraints/vllm-omni.txt` on `feature/vllm-omni`).

## Layer 3: Containers — NOT STARTED

Epic **AIPCC-21244** ("Downstream infrastructure for the vLLM-Omni container image for RHAII") was created July 9 by Klara Bezdekova (AIPCC Productization), priority Critical.

### Child Tasks (All Unassigned, All "New")

| Ticket | Task |
|--------|------|
| AIPCC-21246 | Create Pyxis entry for vLLM-Omni container image |
| AIPCC-21247 | Create KRD resources for vLLM-Omni component onboarding |
| AIPCC-21248 | Create Tekton pipelineruns for vLLM-Omni CI/CD |
| AIPCC-21249 | Create configuration for vLLM-Omni base image |
| AIPCC-21250 | Add vLLM-Omni support to Konflux integration tests |
| AIPCC-21515 | Bug: vllm-omni/vllm version mismatch in index |

**Contact:** Srija Ganguly (Software Engineer, AIPCC) is picking up the container infrastructure work. Klara Bezdekova (AIPCC Productization) manages the epic.

Midstream offered to share Containerfile patterns, build-args, and prototype build results to accelerate. Related midstream ticket: [INFERENG-9278](https://redhat.atlassian.net/browse/INFERENG-9278).

## Layer 4: Base Images — Unknown

No specific vllm-omni base image config exists. AIPCC-21249 covers this. Likely reuses existing CUDA base images (`quay.io/aipcc/base-images/*`) since vllm-omni's system deps (libsndfile, imageio-ffmpeg) are pip-installable.

## Constraints Reconciliation (INFERENG-9335)

### The Problem

Midstream (nm-cicd) applies dependency constraints via `neuralmagic/constraints/vllm-omni.txt` and `--constraints` flags in `prepare.sh`. AIPCC builds via Konflux/fromager from tagged repos — they never see our constraints file. AIPCC-built images could ship with broken deps.

### Current Midstream Constraints (nm-cicd, feature/vllm-omni branch)

```
fastapi[standard] >= 0.133.0, < 0.137.0
transformers >= 5.5.3, < 5.13.0
```

Pending from v0.25.z rebase (INFERENG-9317): fa3-fwd removal, av version override.

### Where Constraints Need to Go in AIPCC

**Option A — New pipeline collection** (if standalone product):
```
collections/vllm-omni/cuda13.0-ubi9/
├── requirements.txt          # vllm-omni==X.Y.Z+rhaiv.N, vllm==..., flashinfer, etc.
├── constraints.txt           # fastapi, transformers pins, fa3-fwd, av
└── constraints-rules.txt     # torch-2.11.0 *
```

**Option B — Existing rhaiis collection** (if variant):
- Add `vllm-omni==X.Y.Z+rhaiv.N` to `collections/rhaiis/*/requirements.txt`
- Add our pins to `collections/rhaiis/*/constraints.txt`
- Risk: constraints could conflict with standard vLLM's deps

**Recommendation:** Option A is cleaner and matches the pipeline API docs guidance ("collections should not be reused between products"). It also aligns with AIPCC-21244 which sets up standalone infrastructure.

### How Standard vLLM Handles This (for reference)

For standard vLLM, the pipeline's `constraints.txt` has hand-written pins. The `constraints-rules.txt` pulls builder-generated constraints from `torch-2.11.0`. There's no mechanism where nm-cicd "pulls from" AIPCC — despite what INFERENG-1004 described, the current nm-cicd code has its own constraint files (`neuralmagic/constraints/neuron.txt`, `tpu.txt`) that are hand-maintained. Standard vLLM CUDA doesn't even use a constraints file in nm-cicd.

### Reconciliation Plan

1. **Short-term (TP, Aug 13):** Keep nm-cicd's `neuralmagic/constraints/vllm-omni.txt` for midstream builds. Provide AIPCC the constraints content for their pipeline collection setup.

2. **Medium-term (post-TP):** Once AIPCC has a vllm-omni pipeline collection with its own `constraints.txt`, evaluate whether nm-cicd should pull from there (INFERENG-1004 pattern) or continue maintaining its own copy. Single source of truth is the goal.

3. **Version pairing (AIPCC-21515):** The two-wheel strategy means vllm-omni and vllm must be version-paired. AIPCC's pipeline collection needs to pin both together in `requirements.txt`. This is a constraint that belongs in requirements, not constraints.txt.

## Coordination Contacts

| Person | Role | Context |
|--------|------|---------|
| Srija Ganguly | AIPCC Software Engineer | Picking up container infrastructure (AIPCC-21244) |
| Klara Bezdekova | AIPCC Productization | Manages the container infra epic |
| Andre Lustosa | AIPCC Ecosystems | Completed builder onboarding |
| Ricardo Noriega | OCTO | Filed original package request, coordination lead |
| Daniele Trifirò | Midstream SME | Designed AIPCC architecture, knows every corner |
| Tarun Kumar | Midstream SME | AIPCC integration experience, model validation |

## Slack Channels

- `#forum-rhaii-release` — primary coordination channel for AIPCC release work
- `#beyond-autoregressive-llms` — cross-team omni sync (Eng + OCTOET)

## Decision Log

| Date | Decision | Context |
|------|----------|---------|
| 2026-07-22 | **Standalone confirmed** — Andre (AIPCC) confirmed separate image, Daniele +1'd, Srija ready to build infra | Slack, INFERENG-9335 |
| 2026-07-22 | Variant vs standalone precedent analysis complete — neuron/TPU (variant) vs model-opt (standalone) | INFERENG-9335 |
| 2026-07-22 | Constraints analysis complete — AIPCC builder has vllm-omni but pipeline collection doesn't | INFERENG-9335 |
| 2026-07-20 | Standalone product implied by AIPCC-21244 (own Pyxis, KRD, Tekton, base image) | Emilien's July 8 comment: "a new vllm image to build and ship" |
| 2026-07-20 | AIPCC layers 1-2 ~60-70% done, layers 3-4 not started | INFERENG-9002 deep dive |
| 2026-07-20 | Srija Ganguly assigned to container infra | Klara response in #forum-rhaii-release |
| 2026-06-29 | vllm-omni in production builder for cpu, cuda12.9, cuda13.0 | AIPCC-16724 closed |
| 2026-06-26 | Midstream not signing off on AIPCC process for DP | Tech alignment agreement doc |
| 2026-06-09 | Initial vllm-omni build in AIPCC builder (0.21.0+rhaiv.1) | Builder changelog |

## JIRA Cross-Reference

| Ticket | What | Owner |
|--------|------|-------|
| [INFERENG-9335](https://redhat.atlassian.net/browse/INFERENG-9335) | Constraints reconciliation (this doc's focus) | Doug Smith |
| [INFERENG-9002](https://redhat.atlassian.net/browse/INFERENG-9002) | AIPCC process alignment spike (closed) | Doug Smith |
| [INFERENG-9278](https://redhat.atlassian.net/browse/INFERENG-9278) | Container repo artifacts (Containerfile, Tekton) | Doug Smith |
| [INFERENG-9000](https://redhat.atlassian.net/browse/INFERENG-9000) | TP Readiness epic | Doug Smith |
| [AIPCC-12521](https://redhat.atlassian.net/browse/AIPCC-12521) | AIPCC vllm-omni productization (parent) | AIPCC |
| [AIPCC-21244](https://redhat.atlassian.net/browse/AIPCC-21244) | Downstream container infrastructure | Klara Bezdekova |
| [AIPCC-21515](https://redhat.atlassian.net/browse/AIPCC-21515) | Version mismatch bug | AIPCC |
| [AIPCC-16724](https://redhat.atlassian.net/browse/AIPCC-16724) | Package onboarding (closed) | Andre Lustosa |
