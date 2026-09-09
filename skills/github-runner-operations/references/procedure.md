---
name: gh-runner-sysadmin
description: Diagnose and fix GitHub Actions self-hosted runner issues on nm-cicd infrastructure. Use when runners aren't picking up jobs, builds are stuck in "queued" state, runner pods are crash-looping, ARC (Actions Runner Controller) listener or controller pods are unhealthy, runner images need rebuilding, or runner versions need bumping. Also use when the user mentions runner labels like k8s-a100-*, ibm-wdc-k8s-h100-*, ibm-wdc-os-*, os-mi300x-*, k8s-gaudi-*, rdu4-k8s-cpu, gcp-k8s-*, build runners, CUDA build runners, runner scale sets, arc-runners namespace, or references stratus runner deployments. Covers all clusters documented in nm-cicd/docs/ (A100 Tarun, H100 IBM, WDC OpenShift, AMD-02, Gaudi RDU4, RDU4 OpenShift, GCP L4/TPU).
---

# GitHub Actions Runner Sysadmin

Operational playbook for diagnosing and fixing self-hosted GitHub Actions runners managed by ARC (Actions Runner Controller) across the neuralmagic org's Kubernetes clusters.

## Canonical Docs (nm-cicd)

**PR #901** (merged 2026-08-12) centralized runner/infra docs into `nm-cicd/docs/` on the `main` branch. These are the source of truth — use them instead of hunting through stratus and rel-eng-faq:

| File | What it covers |
|------|---------------|
| [`docs/runner-labels.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/runner-labels.md) | All runner labels with GPU/CPU/memory specs, naming conventions, per-cluster tables |
| [`docs/hardware-summary.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/hardware-summary.md) | 9 GPU clusters: specs, API URLs, access methods (SSH/OCP/GKE), RBAC notes |
| [`docs/how-to.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/how-to.md) | Where to find info across repos (stratus deployments, rel-eng-faq secrets, model_registry.yml) |

Key things from the PR comment and docs:
- **Tarun A100 and AMD-02 (MI300X) clusters require RBAC grants** — after first `oc login --web` (Google SSO), contact @dtrifiro or @takumar for namespace permissions
- **RDU4 OpenShift login**: `oc login https://api.openshift.nm.rdu4.corp.redhat.com:6443 --web` (needs VPN + FreeIPA DNS or `/etc/hosts` entries)
- **`ibm-wdc-k8s-h100-*` labels are misnamed** — they're on `nma-h100-solo-0-preserve` in IBM Cloud Frankfurt, not WDC
- **`config_ref` was renamed to `model_validation_configs_ref`** in AGENTS.md (matching the actual workflow input)
- **Cluster Access Policy** added to AGENTS.md: any action that modifies cluster state (oc apply, helm install, etc.) MUST be performed manually by a human — agents output commands, don't execute them
- **WDC OpenShift** is a new cluster (A100s + H100s + B300) with bastion access at `169.62.21.215`
- Runner label definitions live in stratus: `misc/<cluster>/deployments/*.yml` and `gcp/infra/*/deployments/*.yml` — grep for `runnerScaleSetName:`

## Architecture Overview

Runners are ephemeral pods managed by ARC on multiple clusters. See `docs/hardware-summary.md` for the full cluster list. The two original clusters:

| Cluster | Type | Runner prefix | Runner group | Access method |
|---------|------|---------------|--------------|---------------|
| A100 (Tarun's OpenShift) | OpenShift | `k8s-a100-*` | `runner_group_a` | `oc login --web` (Google SSO) |
| H100 (IBM Frankfurt) | MicroK8s (single-node) | `ibm-wdc-k8s-h100-*` | `runner_group_b` | SSH to `nma-h100-solo` (see below) |

Additional clusters documented in `docs/hardware-summary.md`: WDC OpenShift (`ibm-wdc-os-a100-*`, `ibm-wdc-os-h100-*`), AMD-02 Azure (`os-mi300x-*`), Gaudi RDU4 (`k8s-gaudi-*`), RDU4 OpenShift, GCP L4 (`gcp-k8s-l4-*`), GCP TPU (`gcp-k8s-tpu-*`).

Each runner label (e.g. `k8s-a100-build-13-0`) is a Helm-managed ARC runner scale set with:
- A **listener pod** in `arc-system` namespace (watches GitHub for queued jobs)
- Ephemeral **runner pods** in `arc-runners` namespace (spin up per-job, die after)
- A **controller** in `arc-system` that orchestrates the lifecycle

## Repo Layout

Runner configs live in `stratus` (see `nm-cicd/docs/how-to.md` for the full map of where things live across repos):

```
misc/a100-tarun-os/deployments/     # A100 runner scale set values
misc/a100-wdc/deployments/          # WDC A100 runner scale set values
misc/h100-on-demand/deployments/    # H100 runner scale set values (IBM Frankfurt)
misc/h100-wdc/deployments/          # WDC H100 runner scale set values
misc/openshift-mi300x/deployments/  # MI300X runner scale set values (Azure)
misc/gaudi-rdu4/deployments/        # Gaudi runner scale set values (RDU4 on-prem)
misc/cpu-on-demand/deployments/     # CPU runner scale set values (RDU4 microk8s)
gcp/infra/*/deployments/            # GCP runner scale set values (L4, TPU)
misc/scripts/runner-scale-set        # Helm install/upgrade script
misc/scripts/runner-scale-set-controller  # Controller install script
misc/scripts/create-github-app-secret     # GitHub App secret creator
quay/automation-k8s-ubuntu-build/    # Runner image Dockerfiles + docker-bake.hcl
.github/workflows/build-image.yml   # Workflow to rebuild runner images
```

Runner image repos on quay.io:
- `quay.io/vllm/automation-k8s-build-ubuntu2004` — CUDA 12.8, 12.9
- `quay.io/vllm/automation-k8s-build-ubuntu2204` — CUDA 13.0 (requires Ubuntu 22.04)

## Cluster Access

Full access details for all 9 clusters are in [`docs/hardware-summary.md`](https://github.com/neuralmagic/nm-cicd/blob/main/docs/hardware-summary.md). Summary:

- **OpenShift clusters** (Tarun A100, AMD-02, WDC, RDU4): `oc login --web` with Google SSO (or GitHub SSO for RDU4). New accounts need RBAC grants from @dtrifiro or @takumar.
- **SSH tunnel clusters** (H100 IBM, H100 RDU4, Gaudi): SSH with `autobot-keys` from GCP secret, then `microk8s kubectl` or standard kubectl.
- **GKE clusters** (L4, TPU): `gcloud container clusters get-credentials` with terraform outputs.
- **Tip**: use separate `KUBECONFIG` files per cluster to avoid token conflicts.

### A100 OpenShift Cluster (Tarun)

```bash
oc login --web  # Google SSO — token expires periodically
# Or use legacy method with GCP secret:
CREDS=$(gcloud secrets versions access latest --secret=ocp_a100_cluster --project=it-cloud-gcp-nm-automation)
oc login --username=<username> --password='<password>' --server=<api_server> --insecure-skip-tls-verify
```

### RDU4 OpenShift Cluster

Requires VPN + DNS resolution (FreeIPA at `10.14.217.1` or `/etc/hosts` entries — see `docs/hardware-summary.md`):

```bash
oc login https://api.openshift.nm.rdu4.corp.redhat.com:6443 --web  # GitHub SSO
```

### WDC OpenShift Cluster

Requires VPN + bastion access (`ssh <github-nick>@169.62.21.215`) + htpasswd user from @Yacine or @bwhorton.

### H100 IBM Cluster (MicroK8s, Frankfurt)

The H100 cluster is a **single-node MicroK8s** cluster running on `nma-h100-0-solo-preserve` in IBM Cloud Frankfurt (`eu-de`, resource group `nm-automation`). It is NOT a managed IKS/ROKS cluster.

**SSH access** (preferred method):

```bash
# SSH config entry is set up in ~/.ssh/config as "nma-h100-solo"
ssh nma-h100-solo
# Then use: microk8s kubectl <command>
```

Connection details:
- Host: `158.176.12.97` (floating IP `fip-h100-3`)
- User: `runner`
- Key: GCP secret `autobot-keys` (saved locally at `~/.ssh/autobot-keys.key`)
- kubectl: use `microk8s kubectl` (there's a shell alias but it doesn't expand in non-interactive SSH)

**Other H100 VMs in the same VPC** (`nm-automation-frk`):

| VM | Floating IP | SSH host | Profile | Notes |
|----|-------------|----------|---------|-------|
| `nma-h100-0-solo-preserve` | `158.176.12.97` | `nma-h100-solo` | `gx3d-160x1792x8h100` | MicroK8s node, 8x H100 |
| `nm-automation-h100-standalone-0-preserve` | `158.176.13.155` | `h100-standalone0` | `gx3d-160x1792x8h100` | Bare GPU node, user `dougbtv` |
| `nm-automation-h100-standalone-1-preserve` | `161.156.86.11` | `h100-standalone1` | `gx3d-160x1792x8h100` | Bare GPU node, user `dougbtv` |

The standalone VMs are NOT part of the MicroK8s cluster — they are separate GPU boxes used for dev/testing.

**Stale secrets (do NOT use for H100 cluster access):**
- `upstream-oc-admin` — points to `c114-e.eu-de.containers.cloud.ibm.com:31673`, a defunct endpoint
- `frankfurt-oc-admin` — points to `c113-e.eu-de.containers.cloud.ibm.com:31758`, also defunct

Always switch back to local context when done: `kubectl config use-context kubernetes-admin@kubernetes`

> **Tip:** For the full list of clusters and access methods, see `nm-cicd/docs/hardware-summary.md` (section "Accessing Clusters").

## Diagnosis Playbook

When runners aren't picking up jobs, work through this sequence:

### 1. Check the listener pods

Listeners live in `arc-system`. Every runner label has one.

```bash
# A100 (OpenShift)
oc get pods -n arc-system

# H100 (MicroK8s — run via SSH)
ssh nma-h100-solo "microk8s kubectl get pods -n arc-system"
```

If a listener is missing or CrashLoopBackOff, the runner label is completely dead. Fix the listener first (usually a Helm redeploy).

### 2. Check the listener logs

```bash
oc logs <listener-pod> -n arc-system --tail=30
```

**Healthy idle listener:**
```
"assigned job": 0, "decision": 0, "currentRunnerCount": 0
```

**Listener trying to scale but runners are dying:**
```
"assigned job": 0, "decision": 1, "currentRunnerCount": 1
```
This means there's a queued job, pods spin up but fail, and the listener keeps retrying every ~50 seconds. Look at runner pod logs next.

### 3. Check runner pods

```bash
oc get pods -n arc-runners
```

If pods are cycling rapidly (new names every minute, short AGE values), they're crash-looping. Catch one and grab logs:

```bash
# Watch for a running pod and grab its logs before it dies
while true; do
  POD=$(oc get pods -n arc-runners --no-headers | grep "<runner-label>" | grep "Running" | awk '{print $1}')
  if [ -n "$POD" ]; then
    oc logs "$POD" -n arc-runners -f &
    LOGPID=$!
    sleep 20
    kill $LOGPID 2>/dev/null
    break
  fi
  sleep 1
done
```

### 4. Check events

```bash
oc get events -n arc-runners --sort-by='.lastTimestamp' | grep "<runner-label>" | tail -20
```

### 5. Check Helm releases

```bash
helm list -n arc-runners
```

Every runner label should have a `deployed` Helm release. If missing, the scale set was never installed or was deleted.

### 6. Check the controller

```bash
oc get pods -n arc-system | grep controller
oc logs <controller-pod> -n arc-system --tail=50
```

## Common Failure Modes

### Runner version deprecated

**Symptom:** Runner pod starts, connects, immediately exits with:
```
Runner version v2.XXX.X is deprecated and cannot receive messages.
```

**Fix:** Bump `RUNNER_VERSION` in `quay/automation-k8s-ubuntu-build/docker-bake.hcl`, rebuild images. Check the latest version at https://github.com/actions/runner/releases.

This affects ALL runner images built from the same docker-bake.hcl, not just the one showing symptoms. Runners without queued jobs won't show the error until a job is queued.

### Runner image doesn't exist

**Symptom:** Events show `ImagePullBackOff` or `ErrImagePull`.

**Fix:** Check the image tag exists with `skopeo inspect docker://<image>:<tag>`. If missing, trigger the build-image workflow.

### GitHub App secret expired

**Symptom:** Listener pod CrashLoopBackOff, logs show auth errors against GitHub API.

**Fix:** Recreate the secret using `misc/scripts/create-github-app-secret -s gcp-k8s`. The GitHub App ID is `888712`, installation ID is `50227859`, private key comes from GCP secret `k8s-github-app`.

### Scale set not deployed

**Symptom:** `helm list -n arc-runners` shows no release for the runner label.

**Fix:** Deploy using the script from the appropriate cluster directory:
```bash
cd misc/a100-tarun-os  # or misc/h100-on-demand
../scripts/runner-scale-set -f deployments/<config>.yml
```

### Stuck/phantom runners

**Symptom:** Listener keeps scaling to replicas=1 even with no jobs, `currentReplicas: 0` in status.

**Fix:** Delete the EphemeralRunnerSet or restart the listener:
```bash
oc delete ephemeralrunners -n arc-runners -l runner-deployment-name=<scale-set-name>
oc delete pod <listener-pod> -n arc-system
```

## Rebuilding Runner Images

### Bump the runner version

Edit `quay/automation-k8s-ubuntu-build/docker-bake.hcl`:
```hcl
RUNNER_VERSION = "2.334.0"  # check https://github.com/actions/runner/releases for latest
```

### Trigger builds

Build targets are defined in the docker-bake.hcl `default` group:
- `k8s-build-ubuntu-cuda-13-0` — pushes to `quay.io/vllm/automation-k8s-build-ubuntu2204:latest-cuda-13.0`
- `k8s-build-ubuntu-cuda-12-9` — pushes to `quay.io/vllm/automation-k8s-build-ubuntu2004:latest-cuda-12.9`
- `k8s-build-ubuntu-cuda-12-8` — pushes to `quay.io/vllm/automation-k8s-build-ubuntu2004:latest-cuda-12.8`

```bash
# Build a specific target (push=true to push to quay)
gh workflow run "build image" \
  --repo neuralmagic/stratus \
  --ref <branch-with-changes> \
  -f context=quay/automation-k8s-ubuntu-build \
  -f push=true \
  -f targets=k8s-build-ubuntu-cuda-13-0 \
  -f clean_disk_space=true
```

All runner deployment configs have `imagePullPolicy: Always`, so new pods automatically pull the latest image — no cluster-side redeploy needed.

### Monitor builds

```bash
gh run list --repo neuralmagic/stratus --workflow="build image" --limit 5
```

## Cleaning Up After a Fix

After new images are pushed, clean up any leftover crash-looping pods so fresh ones pull the new image:

```bash
# Delete all runner pods for a specific label (they'll be recreated by ARC)
oc delete pods -n arc-runners -l runner-deployment-name=<scale-set-name>
```

The listener will automatically spin up new pods with the updated image when the next job is queued.

## Verifying a Fix

After rebuilding images, confirm the fix by checking a runner pod's logs:

```bash
oc logs <runner-pod> -n arc-runners
```

A healthy runner shows:
```
Connected to GitHub
Current runner version: '2.334.0'
Listening for Jobs
Running job: <job-name>
```

A still-broken runner shows the deprecation error or other failure before "Listening for Jobs".

## Key Secrets Reference

See `nm-cicd/docs/how-to.md` → "Access Information" and `rel-eng-faq/secrets/README.md` for the full list. Core secrets in GCP Secret Manager, project `it-cloud-gcp-nm-automation`:

| Secret | Purpose |
|--------|---------|
| `ocp_a100_cluster` | A100 OpenShift login (username/password/api_server) |
| `upstream-oc-admin` | H100 IBM cluster login (token/server) |
| `k8s-github-app` | GitHub App private key for ARC auth |
| `runner-gpu9-priv` | SSH key for gpu9 (A100 on-prem) |
| `runner-gpu21-priv` | SSH key for gpu21 (H100 on-prem) |
| `runner-gpus-pw` | Password for `runner` user on on-prem nodes |

## Triggering a Wheel Build (Post-Fix)

Once runners are healthy, retrigger builds via nm-cicd:

```bash
gh workflow run build-whl-image.yml \
  --repo neuralmagic/nm-cicd \
  --ref <branch> \
  -f repo=neuralmagic/nm-vllm-ent \
  -f branch=<branch> \
  -f build_label=k8s-a100-build-13-0 \
  -f build_timeout=120 \
  -f image_label=ibm-wdc-k8s-h100-util \
  -f python=3.12 \
  -f release_image=false \
  -f target_device=cuda
```
