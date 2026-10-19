# Muse Spark Open B200 testing handoff

**Status:** validated preview image is serving on one B200 node; a second node
has a complete, independent local model cache and is the preferred target for
additional testing.

## Proven artifact

- Preview tag: `quay.io/vllm/rhaiis-early-access:muse-spark`
- Immutable amd64 manifest:
  `sha256:22c8dbacef74716c9c82372e711e18593ec4dec52500735dee66538b500a4439`
- Build: [nm-cicd run 39295371751](https://github.com/neuralmagic/nm-cicd/actions/runs/39295371751)
- Source: `neuralmagic/nm-vllm-ent` branch `doug/muse-spark-open`, commit
  `0b221dba28`
- Public deployment/smoke recipe:
  [Muse Spark Open vLLM B200 smoke test](https://gist.github.com/dougbtv/1572696bb2dbaf5714b281b7e231223e)
- Checkpoint:
  `meta-models/Muse-Spark-Open-816B-A42B-NVFP4`

The image was validated with `/health`, `/v1/models`, and an OpenAI
`/v1/completions` request on 4x B200 (TP=4, expert parallel, 4K context).

## Current cluster state (2026-10-19)

| Node | Allocatable GPUs | Our active GPU load | Model cache |
|---|---:|---:|---|
| `b200-02` | 8 | 4 | `muse-spark-open-cache` (600 Gi RWO local PVC) |
| `dgx-b200-1.accl-001.lab.rdu2.dc.redhat.com` | 8 | 0 | `muse-spark-open-cache-b200-1` (600 Gi RWO local PVC) |

The B200 cluster is project-RBAC scoped, so the live allocation from other
namespaces is not visible. “Our active GPU load” is not a guarantee of free
capacity; make the scheduler prove a 4- or 8-GPU request before committing to a
test.

## Existing validated server: do not disturb

`dougbtv-dev/muse-spark-open` is currently Ready on `b200-02` and holds four
GPUs. Its model is mounted from:

```text
/models/Muse-Spark-Open-816B-A42B-NVFP4
```

Use it for API-level tests if its current TP4/EP, 4K-context configuration is
enough:

```bash
oc -n dougbtv-dev port-forward service/muse-spark-open 8000:8000
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/v1/models | jq .
```

Do not scale, replace, or delete this Deployment/PVC unless the current smoke
service is explicitly being retired.

## Recommended target for new work: second B200 node

Use `dgx-b200-1.accl-001.lab.rdu2.dc.redhat.com`. Its download Job completed
and no active workload in `dougbtv-dev` requests GPUs there. The checkpoint is
already materialized under the same path:

```text
PVC:       muse-spark-open-cache-b200-1
mount path: /models
model path: /models/Muse-Spark-Open-816B-A42B-NVFP4
```

Start from the OCP manifest in the public gist. Make these substitutions:

```yaml
metadata:
  name: muse-spark-open-<your-test>
spec:
  nodeSelector:
    kubernetes.io/hostname: dgx-b200-1.accl-001.lab.rdu2.dc.redhat.com
  # PVC volume claimName:
  # muse-spark-open-cache-b200-1
```

Keep `strategy: Recreate`: a rolling Deployment surge can deadlock when an old
4-GPU pod holds capacity while the replacement waits for the same GPUs.

## Known preview runtime requirements

- Use `quay.io/vllm/rhaiis-early-access:muse-spark`, preferably resolve and pin
  its immutable manifest before deployment.
- Set `VLLM_USE_RUST_FRONTEND=0`; this preview image does not package
  `vllm-rs`.
- Keep the empty-directory mount masking
  `/opt/vllm/lib64/python3.12/site-packages/cv2`; it avoids an unused optional
  TerraTorch/OpenCV path whose native dependencies are absent in this preview.
- The image includes `libcublas-devel` and `libcurand-devel`, which FlashInfer
  needs for the first-use SM100 JIT. Do not trim those CUDA development headers
  from a derivative runtime image.
- Cold start is long: 127 checkpoint shards, then FlashInfer/CUTLASS JIT and
  FP4 MoE autotuning. The observed 22-candidate tuner took about 40 minutes.
  Wait for `Application startup complete` before treating it as a failure.

## Access and cleanup

Follow [RDU4 Pirate cluster access](../rdu4-pirate-cluster-access.md) for the
current login/context procedure. A valid context was observed as `dougbtv` with
namespace `dougbtv-dev`.

For an isolated test, delete only the new Deployment/Service when finished.
Preserve both model PVCs and their completed download Jobs: each represents a
roughly 533 GB staged checkpoint and avoids another lengthy download.
