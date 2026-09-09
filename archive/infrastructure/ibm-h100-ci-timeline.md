# IBM H100 Cluster for vLLM Upstream CI — Timeline & Status Report

## Executive Summary

In August 2025, IBM donated 4x 8xH100 nodes (32 GPUs) to the vLLM upstream CI effort. Andy Linfoot was the sole owner of bringing this infrastructure online for the community. After ~10 months of repeated delays, the cluster never ran a single upstream CI job in production. In April 2026, IBM reclaimed the Frankfurt-hosted hardware for customer use. The contribution has been replaced on paper by a planned WDC deployment (8x 8xH100), but no timeline has been committed.

This report documents the pattern of overpromising and lack of follow-through so leadership can assess the risk of similar commitments going forward.

---

## Timeline

### Phase 1: Commitment (Jul–Aug 2025)

| Date | Source | Event |
|------|--------|-------|
| Jul 15, 2025 | Internal | Meta + IBM/Red Hat call on vLLM HW resource budget/allocations |
| Aug 5, 2025 | CI SIG | Eli (Meta) announces: "4x 8xH100 nodes from IBM will be added to the CI" — 32 GPUs committed |
| Aug 8, 2025 | Internal | Andy references JIRA epic INFERENG-1285: "build out a k8s cluster in IBM cloud to support GHA deployments" |
| Aug 12, 2025 | CI SIG | Andy reports nodes are up but not connected to Buildkite. ETA: that Friday |
| Aug 22, 2025 | Internal | Cluster described as "functioning correctly," working through storage configuration. Plan to "fully transition in September" |

### Phase 2: Brief Functionality Window (Sep 2025)

| Date | Source | Event |
|------|--------|-------|
| Sep 2, 2025 | CI SIG | Andy: "Up and functional. Internal validation uncovered storage issues. Reconfiguring." |
| Sep 9, 2025 | CI SIG | Andy: "GH runners on H100 IBM cluster now. This afternoon getting BK runners." — Closest the cluster ever came to serving upstream CI |
| Sep 12–26, 2025 | Internal | Ongoing sysadmin issues: NVLink configuration, missing nvcc, disk usage, quota management, driver concerns |
| Sep 30, 2025 | CI SIG | Community asks where the cluster is. Update relayed: **"The cluster had to be brought down (on IBM side), so he has to re-provision."** |

### Phase 3: Repeated "Coming Soon" (Oct–Dec 2025)

| Date | Source | Event |
|------|--------|-------|
| Oct 21, 2025 | CI SIG | Huamin (Meta) asks: "What's the status? Neither mithril-h100-pool nor IBM-H100-OpenShift can be used." |
| Oct 28, 2025 | CI SIG | Andy: "will set up the IBM clusters tomorrow" |
| Nov 4, 2025 | CI SIG | "Expect them to be up this week." → Revised to: "Expect a few more weeks" |
| Nov 11, 2025 | CI SIG | Andy offers "If IBM H100 cluster, zero cost" for migrating distributed tests — cluster still not operational |
| Nov 18, 2025 | CI SIG | "Andy sent credentials to Simon for the IBM H100 cluster. Kevin will work with Andy" |
| Dec 2, 2025 | CI SIG | IBM-H100-OpenShift mentioned as potential E2E test host, still blocked |

### Phase 4: Stalled, Minimal Visibility (Jan–Apr 2026)

| Date | Source | Event |
|------|--------|-------|
| Jan 16, 2026 | Internal | Noted internally that Andy has managed this cluster with "very little information" shared with the team |
| Jan 20, 2026 | CI SIG | "Cluster up and running — permission issues to be resolved" |
| Jan 27, 2026 | CI SIG | "Andy is setting it up. IBM wants to migrate." |
| Feb 6, 2026 | Internal / CI SIG | IBM H100 cluster confirmed not working |
| Feb 10, 2026 | CI SIG | Andy: "Talking to IBM research to move to a new cluster" |
| Feb 17, 2026 | CI SIG | "Andy is still setting this up. Andy is OOO this week." |
| Feb 24, 2026 | CI SIG | "IBM H100 cluster — Andy is OOO" |
| Mar 3, 2026 | CI SIG | Action item: Kevin & Andy to discuss using IBM H100 cluster for CUDA 13 parallel testing |
| Mar 10, 2026 | CI SIG | "IBM H100 cluster" — listed, no update |
| Mar 24, 2026 | CI SIG | "4xH100 from IBM — Andy to bring it up" — scope appears reduced from 32 GPUs to 4 |
| Apr 14, 2026 | CI SIG | "IBM H100 cluster" — listed, no substantive update |
| Apr 24, 2026 | Internal | **Frankfurt H100s going down. IBM reclaimed hardware for paying customers. Andy tearing it down.** |

---

## Key Observations

1. **No production CI jobs were ever run on this cluster.** Despite being announced in Aug 2025 and appearing as a standing agenda item in ~20 consecutive CI SIG meetings, the cluster never served its stated purpose.

2. **Single point of ownership with no accountability mechanism.** Andy was the sole owner. When he was OOO (which was frequent during critical periods), progress stopped entirely. The team and community had no visibility into blockers or status beyond what Andy chose to share.

3. **Scope quietly reduced.** The original commitment was 4x 8xH100 (32 GPUs). By March 2026, references had shifted to "4xH100 from IBM" — an 8x reduction with no documented explanation or discussion.

4. **The community was making plans against this capacity.** Multiple CI SIG discussions (distributed test migration, CUDA 13 parallel testing, E2E eval hosting) were predicated on the IBM cluster being available. These plans were repeatedly deferred.
