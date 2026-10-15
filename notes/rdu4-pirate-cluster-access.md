# RDU4 Pirate OpenShift access

Verified 2026-10-15 for Doug (`dougbtv`). This note covers the RDU4
bare-metal OpenShift cluster named **`pirate`**, which has the B200 workers.
`pirate2` is a distinct cluster; do not assume its documented access or
hardware details describe `pirate`.

## Cluster and hardware

- API: `https://api.openshift.nm.rdu4.corp.redhat.com:6443`
- Console: `https://console-openshift-console.apps.openshift.nm.rdu4.corp.redhat.com`
- Authentication: web SSO (`oc login --web`)
- Live B200 workers on 2026-10-15:
  - `b200-02`
  - `dgx-b200-1.accl-001.lab.rdu2.dc.redhat.com`
- Both advertise 8 allocatable `nvidia.com/gpu` devices and 63 allocatable
  `rdma/rdma_shared_device_ib` devices.

## DNS is not provided by the usual VPN resolver

RDU4 cluster names did not resolve through the normal VPN DNS path. Add these
entries to `/etc/hosts` on **both** the machine running `oc` and the machine
running the browser for SSO:

```text
10.14.217.201 api.openshift.nm.rdu4.corp.redhat.com
10.14.217.200 oauth-openshift.apps.openshift.nm.rdu4.corp.redhat.com
10.14.217.202 api.pirate2.nm.rdu4.corp.redhat.com
10.14.217.203 oauth-openshift.apps.pirate2.nm.rdu4.corp.redhat.com
```

The browser only strictly needs the OAuth mapping for the selected cluster;
keeping the four documented entries together makes switching clusters less
surprising.

## Using the nm-cicd helper

The merged helper is `scripts/setup-kubeconfig.sh` in `neuralmagic/nm-cicd`.
In a checkout containing it:

```bash
./scripts/setup-kubeconfig.sh setup
./scripts/setup-kubeconfig.sh login pirate
```

It configures the `pirate` context and starts `oc login --web`. The RDU4 API
currently uses a self-signed/private certificate, so `oc` asks to accept the
insecure TLS connection.

### Headless VM login

The helper expects a graphical browser. On a headless VM it fails to invoke
`xdg-open`; use `oc` directly and tunnel the OAuth callback instead.

On the VM, leave this running:

```bash
oc login --web \
  --auto-open-browser=false \
  --callback-port=18080 \
  --insecure-skip-tls-verify=true \
  --server=https://api.openshift.nm.rdu4.corp.redhat.com:6443
```

It prints an OAuth URL and waits for the callback. On the laptop, in a second
terminal, keep this tunnel open:

```bash
ssh -N -L 18080:127.0.0.1:18080 dev@192.168.121.115
```

Open the VM-provided OAuth URL on the laptop, complete SSO, and the redirect to
`127.0.0.1:18080` traverses the SSH tunnel back to the waiting `oc` process.

Verify the session:

```bash
oc whoami
oc get nodes
```

## Personal projects and B200 smoke testing

The first SSO login does not grant cluster-wide pod or namespace visibility.
`dougbtv` could self-provision a project:

```bash
oc new-project dougbtv-dev
```

A one-GPU CUDA Job in that project scheduled immediately on
`dgx-b200-1.accl-001.lab.rdu2.dc.redhat.com` and reported:

```text
NVIDIA B200, driver 595.71.05, 183359 MiB
```

The live B200 nodes had no taints at validation time. Target them with:

```yaml
nodeSelector:
  nvidia.com/gpu.product: NVIDIA-B200
resources:
  limits:
    nvidia.com/gpu: "1"
```

The static Stratus MachineSet still documents a `nvidia.com/gpu-test=b200`
`NoSchedule` taint, so check live node taints before relying on this:

```bash
oc get nodes -l nvidia.com/gpu.product=NVIDIA-B200 \
  -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

For a real vLLM test, start with one GPU and a small model. The existing
`nm-cicd` OCP model-deployment harness is CI-oriented; a personal project is
appropriate for user-driven experiments.

## OpenShift-native vLLM deployment pattern

For user-driven work, replace a direct `podman run` recipe with persistent
OpenShift resources in `dougbtv-dev`:

```text
Hugging Face token Secret
          |
          v
init container downloads model --> PersistentVolumeClaim (model/HF cache)
                                      |
                                      v
                         vLLM Deployment (one replica on B200)
                                      |
                                      v
                       ClusterIP Service --> oc port-forward (development)
                                          \-> Route (only for shared access)
```

The mappings from direct-container flags to OpenShift are:

| Direct host pattern | OpenShift resource/pattern |
| --- | --- |
| `podman run` | `Deployment` with one replica |
| `--device nvidia.com/gpu=all` | `resources.requests/limits.nvidia.com/gpu: N` |
| bind-mounted Hugging Face cache | PVC mounted at the model/cache path |
| `--ipc=host` | memory-backed `emptyDir` mounted at `/dev/shm` |
| host port `8000` | ClusterIP Service and `oc port-forward` |
| `--rm` | delete the Deployment and PVC when the experiment ends |

Store the Hugging Face token in a Kubernetes Secret, never inline in a
manifest. An init container should populate the PVC before vLLM starts; mount
the completed model cache read-only in the server container when feasible. A
PVC avoids repeated model downloads on Pod restart. A single-node `ReadWriteOnce`
PVC is sufficient for the initial experiment; shared/multi-node caches need
deliberate storage design rather than assuming one claim can be mounted
everywhere.

Start with one B200 and a small model, accessed through:

```bash
oc port-forward service/<vllm-service> 8000:8000
curl -fsS http://127.0.0.1:8000/health
```

Then scale the GPU request and vLLM `--tensor-parallel-size` together (for
example, 2, 4, or 8) according to model memory requirements. Single-node TP on
one B200 DGX uses NVLink/NVSwitch; it does not need an RDMA resource request.
Treat multi-node inference/RDMA as a separate design step.

The `nm-cicd` `neuralmagic/ocp_model_deployment` harness provides an existing
reference implementation for the PVC-or-`emptyDir`, `/dev/shm`, GPU request,
Deployment, Service/Route, and health-check pieces. It is designed for
ephemeral CI validation, so a user-facing deployment should retain those
primitives but omit CI-specific automatic namespace/resource cleanup.
