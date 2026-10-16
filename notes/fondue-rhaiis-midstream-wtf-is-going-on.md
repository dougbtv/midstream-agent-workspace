# Fondue & RHAIIS Midstream: What Changed

This document explains how the consolidation of `builder`, `rhaiis-pipeline`,
and the wheel-building infrastructure into the `fondue` monorepo affects the
RHAIIS midstream team.

## Repo consolidation

The standalone `builder` and `rhaiis-pipeline` repositories have been merged
into a single monorepo called
[fondue](https://gitlab.com/redhat/rhel-ai/wheels/fondue):

| What                                     | Old repo                                            | New location in fondue                                                                                                                       |
| ---------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Fromager plugins, overrides, settings    | `builder/`                                          | [`builder/`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/blob/main/builder/)                                                           |
| RHAIIS wheel collection definitions      | `rhaiis-pipeline/collections/rhaiis/`               | [`rhai-pipeline/collections/rhaiis/`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/tree/main/rhai-pipeline/collections/rhaiis)          |
| Pipeline CI (ci-wheelhouse.yml includes) | `rhaiis-pipeline/.gitlab-ci.yml`                    | [`fondue/.gitlab-ci.yml`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/blob/main/.gitlab-ci.yml) (runs everything internally)           |
| Product version                          | `rhaiis-pipeline/product-version.yml` (`3.6-fast2`) | [`rhai-pipeline/product-version.yml`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/blob/main/rhai-pipeline/product-version.yml) (`3.6`) |

The `rhaiis-pipeline` repo is effectively superseded — its collection files,
constraints, and CI configuration now live under `rhai-pipeline/` in fondue.
Fondue's own CI runs the full bootstrap → build → publish pipeline for every
collection, including rhaiis, without external includes.

## Collection files migration

The rhaiis collection files moved from
`rhaiis-pipeline/collections/rhaiis/` to
[`rhai-pipeline/collections/rhaiis/`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/tree/main/rhai-pipeline/collections/rhaiis)
in fondue. The structure within each variant directory changed slightly:

- **`requirements.txt`** — common packages (triton, timm, bitsandbytes,
  opentelemetry, etc.), anything not linking to torch.
- **`constraints.txt`** — build constraints. Simplified compared to the old
  repo; most detailed pins moved into the torch-specific constraints file.
- **`torch/requirements-torch-<version>.txt`** — torch-linked packages
  (vLLM, FlashInfer, xformers, llm-d kernels, plugins) split out by torch
  version. In the old repo these were all in a single `requirements.txt`.
- **`torch/constraints-torch-<version>.txt`** — torch-specific build
  constraints.
- **`constraints-rules.txt`** — mostly removed. The old constraints API
  mechanism (pulling builder constraints via `constraints-rules.txt`) has
  been replaced by explicit constraint files. Only `rubin-ubi9` and
  `tpu-ubi9` still carry one.

For example, the CUDA 13.0 torch-2.13 collection is at:
[`rhai-pipeline/collections/rhaiis/cuda13.0-ubi9/torch/requirements-torch-2.13.txt`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/blob/main/rhai-pipeline/collections/rhaiis/cuda13.0-ubi9/torch/requirements-torch-2.13.txt)

## Multiple versions in one requirements file

You may notice that a single `requirements-torch-<version>.txt` file lists
the same package multiple times with different `==` pins — for example, both
`vllm==0.28.0+rhaiv.3` and `vllm==0.31.0+rhaiv.1`. This is intentional and
is how fondue builds multiple stacks in one channel.

Fondue enables **multi-version bootstrap** for the rhaiis collections. When
enabled, Fromager does not resolve each package to a single best-match
version. Instead, it builds **every** version listed in the requirements
file, plus any versions published within the last 10 days (`max_release_age`).
All built wheels land in the same Pulp channel.

This is why the file uses `requirements.txt` rather than `constraints.txt`:
a constraints file can only express one version per package, but a
requirements file can list the same package multiple times with exact pins.
Each block of pins (vLLM + FlashInfer + quack-kernels + xgrammar + ...)
forms a self-contained stack, and all stacks coexist in the channel.

Your containers' lock file then selects one specific stack by pinning to the
version it needs. For example, after fondue builds vLLM 0.28.0 and 0.31.0
side-by-side, the CUDA container's `context/cuda/requirements.txt` can pin
`vllm==0.31.0+rhaiv.1` and pull only that stack.

## Pulp channels and index URLs

Wheels are no longer published to the GitLab Generic Package Registry as
versioned release artifacts (e.g. `3.5.3697+rhaiis-cuda13.0-ubi9-x86_64`).
Instead, fondue publishes wheels to **Pulp channels** identified by
`{accel}{sdk}-torch{major.minor}-el9.8`.

The channel catalog is defined in
[`rhai-pipeline/channels.yml`](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/blob/main/rhai-pipeline/channels.yml).
For example, the CUDA 13.0 torch-2.13 channel is `cuda13.0-torch2.13-el9.8`.

Each channel has a domain:

- **`public-rhai`** — CPU, CUDA, ROCm, Spyre channels. Index is publicly
  accessible.
- **`rhai`** (private) — Gaudi, Neuron, TPU channels. Index requires
  authentication.

The pip simple index URLs follow this pattern:

- **Test index** (used by CI, Konflux builds, AutoQA):
  `https://packages.redhat.com/api/pypi/public-rhai/{channel}-test/simple/`
- **Production index** (used by released containers):
  `https://packages.redhat.com/api/pypi/public-rhai/{channel}/simple/`

For private channels, the base URL changes to
`https://private.console.redhat.com/api/pypi/rhai/{channel}[-test]/simple/`.

For example, the CUDA 13.0 torch-2.13 test index is:
`https://packages.redhat.com/api/pypi/public-rhai/cuda13.0-torch2.13-el9.8-test/simple/`

## CI pipeline triggers

Fondue's CI runs wheel build jobs on two types of events:

**Post-merge push to main** — jobs run only when the changed files match the
collection's path list. For example, the rhaiis CUDA 13.0 torch-2.13 jobs
only trigger if files under
`rhai-pipeline/collections/rhaiis/cuda13.0-ubi9/` change. If you only touch
a CPU collection, the CUDA jobs don't run.

**Scheduled nightly** (8 PM Eastern) — all jobs run unconditionally,
regardless of what changed. This is how new upstream releases get picked up:
multi-version bootstrap builds anything published within the last 10 days,
even if no fondue files changed. The nightly schedule is configured in the
[pipeline schedules](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/pipeline_schedules/).

The nightly also serves as a safety net: if a post-merge pipeline fails or is
canceled, the nightly will retry the build. This is relevant when a build
failure blocks a new version from landing on the index — the next nightly
will attempt it again.

Other schedule types (`wheel-promote`, `wheel-copy`, `channel-apply`) trigger
Pulp promotion and channel management workflows, not builds.

## Coming up: hermetic container builds (MR !1232)

[MR !1232](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1232)
in the rhaiis-containers repo will change how containers consume the wheels
that fondue builds. This is the largest change to the containers workflow.

### What changes

All variants switch from fetching wheels at build time to installing
from a **committed, hash-pinned lock file**.

**Old flow**: `install-wheel-release.sh` downloads wheels from the GitLab
Generic Package Registry during the container build, using a
secret-mounted `BOT_PAT`. Build-args hardcode wheel release tags (e.g.
`WHEEL_RELEASE_X86_64=3.5.3697+rhaiis-cuda13.0-ubi9-x86_64`). Renovate
tracks those tags and bumps them when rhaiis-pipeline publishes a new
release.

**New flow**: each variant carries a committed
`context/<variant>/requirements.txt` with hash-pinned wheels.
[Cachi2](https://cachi2.github.io/cachi2/) prefetches all wheels before the
container build. The build installs from the prefetch cache with
`--no-index --offline --require-hashes` so that it doesn't require network
access.

Build-args change from `WHEEL_RELEASE_*` to `INDEX_VERSION` and `*_INSTALL_MODE`.
Renovate now tracks fondue's channel-stamped base image tags instead of
wheel release tags.

### New lock management

The Makefile adds lock management targets:

- `make lock-<variant>` — regenerate the hash-pinned `requirements.txt`
- `make check-lock-<variant>` — validate the committed lock is up to date
- `make upgrade-lock-<variant> UPGRADE_PACKAGE=NAME[==VERSION]` — targeted
  upgrade of one package, with all other pins frozen

### vLLM-Omni deferral

vLLM-Omni is prepared but **not converted** in this MR — it still uses the
legacy `install-wheel-release.sh` path until its `espeak-ng` RPM prerequisite
has a home (fondue base image vs in-repo RPM lockfile). That decision is
still pending.
