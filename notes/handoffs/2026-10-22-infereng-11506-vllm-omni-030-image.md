# INFERENG-11506 — vLLM-Omni 0.30 image delivery handoff

## Mission

Produce a working RHAIIS 3.6 vLLM-Omni CUDA image containing this exact pair:

```text
vllm-omni[watermarking]==0.30.0+rhaiv.2
vllm[audio,tensorizer,cgraph-cuda13]==0.30.0+rhaiv.0
```

The immediate milestone is a successful hermetic Konflux candidate build from
containers MR `!1490`, with an image reference and digest recorded in
INFERENG-11506. The later release milestone is a production-index build from
the merged `3.6` branch and its normal component-scoped release tag.

Do not substitute upstream `0.30.0`, advance either package independently, or
move Omni to vLLM 0.31.

## Start here

From the Midstream workspace:

```bash
cd /home/dev/.openclaw/workspace/codebase/midstream-agent-workspace
sed -n '1,240p' AGENTS.md
bin/status
sed -n '1,360p' runbooks/aipcc/vllm-omni.md
```

Before Jira work, read both of these completely:

```bash
sed -n '1,240p' skills/jira-board/SKILL.md
sed -n '1,360p' skills/jira-board/references/procedure.md
```

The attached containers repository is independent:

```text
repos/aipcc/containers
```

Expected working branch:

```text
INFERENG-11506-vllm-omni-030-image
```

Always fetch and re-check live state. Everything below is a dated handoff, not
permission to assume an MR or pipeline is still in the same state.

## Primary records

- Jira: [INFERENG-11506](https://redhat.atlassian.net/browse/INFERENG-11506)
- Omni containers candidate:
  [containers !1490](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1490)
- PMT 3.6 source of truth:
  [product-management-configs !824](https://gitlab.com/redhat/rhel-ai/ci-cd/aipcc-product-management-configs/-/merge_requests/824)
- PMT-generated 3.6 PipelineRuns:
  [containers !1501](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1501)
- 3.6 test-channel base/index policy:
  [containers !1502](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1502)
- Equivalent `main` policy change, useful for comparison only:
  [containers !1500](https://gitlab.com/redhat/rhel-ai/rhaiis/containers/-/merge_requests/1500)
- Base-image RPM change:
  [fondue !1570](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/merge_requests/1570)
- Omni `.rhaiv.2` wheel collection:
  [fondue !1540](https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/merge_requests/1540)
- Test-first release guidance:
  [snippets #163](https://github.com/neuralmagic/rhaiis-midstream-snippets/pull/163) and
  [snippets #165](https://github.com/neuralmagic/rhaiis-midstream-snippets/pull/165)

## State verified on 2026-10-22

### MRs and infrastructure

- Containers `!1502` is merged into `3.6`.
- PMT `!824` is approved and its pipeline is green, but the MR was still open.
- Generated containers `!1501` is approved but was still open.
- Konflux application `rhaiis-3-6` exists in `ai-tenant`.
- Konflux component `rhaiis-vllm-omni-cuda-ubi9-3-6` exists and tracks the
  containers repository's `3.6` branch.
- `!1501` successfully built all ten 3.6 components, including the legacy Omni
  image. Its Omni label test passed; its optional Omni Podman integration test
  failed. That proves the component and service account work, but it does not
  prove the new Omni 0.30 hermetic image works.
- `!1490` is still Draft at commit
  `d963a1645cbd9e370340317d583b4f34d90951d7`. Its recorded Konflux failure is
  stale: that run happened before the 3.6 component/service account was
  provisioned.

### Wheel and base-image state

- The CUDA 13 / Torch 2.13 test index contains Omni `.rhaiv.0` and `.rhaiv.2`.
- The corresponding production index still contains only `.rhaiv.0`.
- Fondue `!1570` added `espeak-ng` and its `pcaudiolib` dependency to the
  CUDA 13 base RPM lock. The new base path also supplies the other runtime
  libraries needed by Omni. Do not restore an online `dnf install`.
- The 3.6 development policy is now to use `-test` channel base images and let
  the base image's `com.redhat.aiplatform.index_url` label select the test
  Python index. The branch flips to production channel images and regenerated
  locks only before the final RC.
- The current 3.6 CUDA test base selected by `!1502` is:

  ```text
  quay.io/aipcc/base-images/cuda13.0-torch2.13-el9.8:cuda13.0-torch2.13-1792598771-test
  ```

### What `!1490` already implements

- Exact Omni/base-vLLM pins shown at the top of this document.
- The `watermarking` extra, which brings the VideoSeal/AudioSeal dependencies.
- A generated, hash-pinned, multi-architecture
  `context/vllm-omni/requirements.txt`.
- Cachi2 prefetch and offline package installation.
- Removal of the legacy wheel-release installer and online RPM installation.
- 3.6 application/component/service-account names in both Omni PipelineRuns.
- Local hermetic-tooling tests and lock verification were green before the
  latest 3.6 branch changes.

`!1490` still needs rebasing and cleanup against `!1501`/`!1502`.

## Important ownership boundary

The files below say they are PMT-generated and must not be maintained only in
the containers MR:

```text
.tekton/rhaiis-vllm-omni-cuda-ubi9-on-pull-request.yaml
.tekton/rhaiis-vllm-omni-cuda-ubi9-on-push.yaml
```

Their durable source is `rhaiis/3.6/config.yaml` in PMT `!824`. The current PMT
Omni entry has the correct 3.6 identity but still describes the legacy build.
Add the hermetic inputs to PMT, then use its generated containers change. Do not
leave a hand-edited generated PipelineRun as the only copy of the configuration.

The Omni entry needs to match the converted CUDA/model-opt pattern:

```yaml
cel_path_changed:
  - "context/vllm-omni/***"
  - "context/scripts/install-variant-packages.sh"
  - "context/scripts/install-python-packages.sh"
params:
  - name: hermetic
    value: "true"
  - name: prefetch-input
    value: '[{"type": "pip", "path": ".", "requirements_files": ["context/vllm-omni/requirements.txt"], "binary": {"os": "linux", "arch": "x86_64,aarch64", "py_version": 312, "py_impl": "cp"}}]'
task_run_specs:
  - task_name: build-container
    memory: 8Gi
  - task_name: prefetch-dependencies
    step_specs:
      - name: prefetch-dependencies
        compute_resources:
          limits:
            cpu: "1"
            memory: 8Gi
          requests:
            cpu: "1"
            memory: 4Gi
```

Coordinate that change with the PMT/AIPCC owners. If `!824` merges before it
can be amended, use a small follow-up PMT MR and record the dependency in
`!1490` and INFERENG-11506.

## Execution plan

### 1. Refresh the dependency state

```bash
for spec in \
  "redhat/rhel-ai/ci-cd/aipcc-product-management-configs 824" \
  "redhat/rhel-ai/rhaiis/containers 1490" \
  "redhat/rhel-ai/rhaiis/containers 1501" \
  "redhat/rhel-ai/rhaiis/containers 1502"
do
  repo="${spec% *}"
  iid="${spec##* }"
  enc=$(printf '%s' "$repo" | sed 's|/|%2F|g')
  glab api --hostname gitlab.com "projects/${enc}/merge_requests/${iid}" |
    jq '{iid,title,state,draft,merge_status,detailed_merge_status,merged_at,updated_at,web_url,head_pipeline}'
done
```

Check approvals separately; `approved` is not reliably present on the normal MR
response:

```bash
glab api --hostname gitlab.com \
  projects/redhat%2Frhel-ai%2Frhaiis%2Fcontainers/merge_requests/1501/approvals |
  jq '{approved,approvals_left,approved_by}'
```

Check live Konflux resources:

```bash
oc project ai-tenant
oc get applications rhaiis-3-6
oc get components rhaiis-vllm-omni-cuda-ubi9-3-6 -o yaml
```

Direct ServiceAccount reads may be forbidden. A successful Omni Konflux build
is sufficient proof that the build service account exists and is usable.

### 2. Get the shared 3.6 wiring landed

The preferred landing order is:

1. PMT `!824` supplies/provisions the 3.6 productization resources.
2. Generated containers `!1501` lands the release-wide PipelineRuns.
3. Omni `!1490` rebases onto the resulting `3.6` branch and carries only the
   Omni hermetic/image delta not owned by the release-wide generation.

Both `!824` and `!1501` were already approved. If still open, inspect their
current failing/pending statuses and ask their owners to finish them; do not
silently merge another team's productization MR without authority.

The live resources already exist, so a test run of `!1490` is technically
possible before both MRs merge. Landing `!1490` first is still wrong because
`!1501` owns the generated release-wide PipelineRun files.

### 3. Rebase and reconcile `!1490`

Do this after `!1501` and `!1502` are present on `origin/3.6`:

```bash
cd /home/dev/.openclaw/workspace/codebase/midstream-agent-workspace/repos/aipcc/containers
git status --short --branch
git fetch --prune origin
git branch backup/INFERENG-11506-before-3.6-rebase
git rebase origin/3.6
```

During conflicts:

- Keep PMT-generated 3.6 application, component, output repository, service
  account, release-branch predicate, tag predicates, labels, and pipeline
  resolver from `!1501`.
- Add the hermetic/Cachi2 inputs through PMT as described above.
- Keep `!1502`'s channel-base Renovate manager. Remove the obsolete legacy Omni
  `WHEEL_RELEASE_*` Renovate managers after the Omni argfile is converted.
- Do not discard unrelated lock refreshes that landed through `!1502`.

After review, push rewritten history only with an explicit lease:

```bash
git push --force-with-lease origin INFERENG-11506-vllm-omni-030-image
```

### 4. Align the candidate with test-first policy

The converted Omni argfile should be:

```ini
# Renovate updates BASE_IMAGE. The lock generator resolves the package index
# from BASE_IMAGE's com.redhat.aiplatform.index_url label.
BASE_IMAGE=quay.io/aipcc/base-images/cuda13.0-torch2.13-el9.8:cuda13.0-torch2.13-1792598771-test
RHAIIS_VERSION=3.6
INDEX_VERSION=3.6
```

Remove the temporary Omni-only `VERIFY_INDEX_URL` support and assertion from:

```text
.gitlab-ci.yml
context/scripts/test_hermetic_tooling.py
```

The base-image label is the single source of truth for the index. An explicit
test-index override can hide a base/index mismatch and should no longer be
needed after selecting the `-test` channel base.

Regenerate the lock using repository tooling, then prove it is reproducible:

```bash
python3.12 context/scripts/gen-lock.py --project context/vllm-omni
python3.12 context/scripts/gen-lock.py --project context/vllm-omni --check
```

If generation says no version of `vllm-omni==0.30.0+rhaiv.2` exists, first
confirm which index the selected base advertises. Do not loosen the version or
remove `+rhaiv.2` to make resolution pass.

### 5. Run repository validation

At minimum:

```bash
python3.12 -m unittest context/scripts/test_hermetic_tooling.py
python3.12 context/scripts/gen-lock.py --project context/vllm-omni --check
```

Also run any lint/YAML checks documented by the containers repository and
inspect the complete diff against the current target:

```bash
git diff --check origin/3.6...HEAD
git diff --stat origin/3.6...HEAD
git diff origin/3.6...HEAD -- \
  Containerfile.vllm-omni.cuda-ubi9 \
  build-args.vllm-omni/cuda-ubi9.conf \
  context/vllm-omni \
  renovate.json \
  .gitlab-ci.yml \
  .tekton/rhaiis-vllm-omni-cuda-ubi9-on-pull-request.yaml \
  .tekton/rhaiis-vllm-omni-cuda-ubi9-on-push.yaml
```

Before pushing, verify:

- the target branch is `3.6`, not `main`;
- the MR remains Draft until PMT ownership is durable and the candidate gates
  pass;
- both architecture hashes exist in the generated lock;
- the Containerfile performs offline installation and contains no `dnf install`;
- both PipelineRuns contain `hermetic: "true"` and the Omni Cachi2 prefetch;
- Renovate manages the channel base and no longer manages legacy wheel-release
  variables;
- commit and MR title begin with `INFERENG-11506:`;
- commit and MR description contain a `Signed-off-by:` line.

### 6. Observe the candidate build

GitLab's ordinary `build-image-mr-*` matrix does not build Omni. The real Omni
image status is an external Konflux commit status.

```bash
PROJECT='redhat%2Frhel-ai%2Frhaiis%2Fcontainers'
MR=1490
SHA=$(glab api --hostname gitlab.com "projects/${PROJECT}/merge_requests/${MR}" | jq -r .sha)

glab api --hostname gitlab.com \
  "projects/${PROJECT}/repository/commits/${SHA}/statuses?per_page=100" |
  jq -r '.[] | [.status,.name,.description,.target_url] | @tsv' |
  sort -k2,2
```

Require these to pass:

- `linter-central`
- `test-shared-hermetic-tooling`
- `verify-vllm-omni-lock`
- `Konflux Production Internal / rhaiis-vllm-omni-cuda-ubi9-3-6-on-pull-request`
- `rhaiis-3-6-check-labels-tech-preview`
- `rhaiis-3-6-test-vllm-omni-podman-cuda-x86-64`, or a documented diagnosis
  showing that an optional test failure is unrelated to the image change

Use the external status's `target_url` for the PipelineRun. With sufficient
RBAC, PipelineRuns and TaskRuns are in `ai-tenant-build`, even though the UI URL
contains `ai-tenant`. If CLI reads are forbidden, use the Konflux UI and attach
downloaded YAML/log evidence rather than guessing.

A failure before `prefetch-dependencies` is usually productization wiring or
service-account infrastructure. A resolver failure during prefetch is an
index/lock problem. A failure during the image build is a Containerfile or
offline-install problem. Classify the phase before editing anything.

### 7. Record the actual candidate image

On a successful pull-request build, capture:

- MR SHA;
- PipelineRun URL and result;
- Snapshot name;
- temporary `quay.io/redhat-user-workloads/...` image reference;
- immutable digest;
- label-test and Omni Podman-test results.

Do not call a Quay push an RC. It is a candidate image build until the release
train declares an RC.

Add one concise, non-duplicate Jira comment with the evidence and the next
gate. The latest Jira comment currently describes missing Konflux resources;
that statement became stale after provisioning and should be corrected when
the new candidate run exists.

### 8. Move from candidate to release image

Only after the candidate passes and the MR receives review:

1. Make the PMT configuration durable and ensure generated PipelineRuns match.
2. Obtain authorization to mark `!1490` ready and merge it into `3.6`.
3. Verify the exact merged `3.6` SHA.
4. Obtain authorization for a protected, component-scoped tag using the
   established form `vllm-omni-cuda-vYYYYMMDDNN` at that exact SHA.
5. Follow the on-push PipelineRun through Snapshot and Release.
6. Record the immutable `quay.io/aipcc/rhaiis-vllm-omni/cuda-ubi9` image tag and
   digest. Ignore `-source`/`.src` artifacts.

Do not create the release tag merely to test the MR. The on-pull build is the
candidate gate.

### 9. Production-index flip before final RC

The development image should use the test channel. The final RC must not.

Before the final RC:

1. Confirm AIPCC/fondue has promoted Omni `.rhaiv.2` and all matching
   dependencies into the unsuffixed production CUDA 13 / Torch 2.13 index.
2. Confirm publication from the simple-index API; a merged Fondue MR is not
   publication evidence.
3. Use the production-channel base selected by the release/productization
   team. Verify its `com.redhat.aiplatform.index_url` label resolves to the
   production index.
4. Regenerate `context/vllm-omni/requirements.txt` against that base.
5. Re-run all lock, Konflux, label, and Omni Podman gates.
6. Record the production-index image separately from the test-index candidate.

## Failure guide

| Symptom | Likely layer | Action |
|---|---|---|
| Component or ServiceAccount missing | PMT/Konflux provisioning | Check `!824`, generated resources, and live tenant state; do not edit package pins. |
| `verify-vllm-omni-lock` says the lock differs | Base/index or stale export | Confirm the base label's index, regenerate, and review the diff. |
| Resolver cannot find `.rhaiv.2` | Wrong/prod index or publication lag | For the Draft candidate use the approved `-test` channel base; do not weaken the pin. |
| Cachi2 prefetch OOM | Pipeline resource specification | Ensure PMT generates the 8 GiB prefetch step resources. |
| Container tries to reach PyPI/Pulp | Hermetic installer wiring | Check Cachi2 output and offline install scripts; do not add network access. |
| `espeak-ng` missing | Wrong/stale base image | Use the approved RPM-bearing CUDA channel base; do not restore online DNF. |
| Ordinary GitLab image jobs pass but no Omni image exists | Looking at the wrong pipeline | Inspect the external Konflux Omni status on the MR SHA. |
| Generated `.tekton` changes disappear | PMT source was not updated | Fix `rhaiis/3.6/config.yaml` and regenerate. |
| Omni image builds but Podman test fails | Runtime/image integration | Read the integration PipelineRun logs; compare with the legacy `!1501` failure before changing dependencies. |

## Guardrails

- Do not retarget `!1490` to `main`.
- Do not use `3.6-fast1` or `3.6-fast2` artifacts for this GA image.
- Do not use the legacy `vllm-omni-wheels` release installer.
- Do not add an online RPM transaction to the Containerfile.
- Do not manually point lock verification at an index different from the base
  image label.
- Do not drop the `watermarking` extra.
- Do not remove local-version suffixes (`+rhaiv.N`).
- Do not treat a Fondue merge as proof that wheels reached an index.
- Do not treat a successful build as a released or promoted image.
- Do not edit or merge unrelated user changes in any attached repository.

## Definition of done

The immediate goal is complete when all of the following are recorded in
INFERENG-11506:

- exact source pair `.rhaiv.2` / `.rhaiv.0`;
- PMT-owned hermetic PipelineRun configuration;
- green reproducible Omni lock check;
- successful dual-architecture Konflux candidate build;
- Snapshot and immutable candidate image digest;
- label validation result;
- Omni Podman integration result or accepted, evidenced disposition;
- explicit next step for merge/release or production-index promotion.

The release goal is complete only after the merged `3.6` commit produces the
authorized component-tag build and its immutable RHAIIS vLLM-Omni image is
recorded. The final-RC goal additionally requires the production-index flip and
fresh validation.
