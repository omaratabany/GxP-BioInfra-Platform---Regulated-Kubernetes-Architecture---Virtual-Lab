# PRJ-02 Phase Map and Schedule

> Part of [[README]] | See also: [[PRJ-01 Strategic Rationale]], [[INF-02 Architecture and Components]]

---

## Schedule

| Week | CKA Domain | Project Phase | Status |
|---|---|---|---|
| 1-2 | Cluster architecture, etcd | [[PH-00 Cluster Preparation]] | COMPLETE |
| 3 | Workloads | [[PH-01 Forgejo]] | NOT STARTED |
| 4 | Scheduling, affinity | [[PH-02 Authentik SSO]] | NOT STARTED |
| 5 | Networking | [[PH-03 MinIO Object Storage]] | NOT STARTED |
| 6 | Storage | [[PH-04 OPA Gatekeeper]] | NOT STARTED |
| 7 | RBAC, troubleshooting | [[PH-05 Falco Runtime Security]] + [[PH-06 Nextflow and nf-core]] | NOT STARTED |
| 8 | Exam sim | [[PH-06 Nextflow and nf-core]] -- pipeline run | NOT STARTED |
| Post-CKA | -- | [[PH-07 GxP Validation Documentation]] + public README | NOT STARTED |

---

## Phase Summary

### [[PH-00 Cluster Preparation]] -- COMPLETE
My foundation is locked. Node labels are applied, the Omen is untainted, the Beelink HDD is mounted and verified, `local-hdd` is the default StorageClass, and a real PVC test proved provisioning onto the Beelink HDD. The PH-00 etcd snapshot is saved outside the Git repo at `../snapshots/phase0-complete.snapshot`.

### [[PH-01 Forgejo]] -- NOT STARTED
My self-hosted Git goes live on the cluster. Once it is up, Forgejo is my primary SCM and GitHub drops to a public mirror. ArgoCD switches to watching Forgejo on push via webhook.

### [[PH-02 Authentik SSO]] -- NOT STARTED
OIDC and SSO across the whole platform. After this phase, I have a single account across Forgejo, ArgoCD, and Grafana. No local user passwords remain anywhere on the cluster.

### [[PH-03 MinIO Object Storage]] -- NOT STARTED
S3-compatible object storage on the Beelink HDD. I create the pipeline data buckets and migrate Loki off its local PVC onto the MinIO backend.

### [[PH-04 OPA Gatekeeper]] -- NOT STARTED
Policy-as-code admission control. My first hard GxP enforcement layer -- image digest pinning, resource limits, and approved registry constraints enforced at the API before anything runs.

### [[PH-05 Falco Runtime Security]] -- NOT STARTED
Runtime security and audit trail running as a DaemonSet across both nodes. Events route to Loki through Falcosidekick. This is how I satisfy the Annex 11 audit trail requirement for runtime behaviour.

### [[PH-06 Nextflow and nf-core]] -- NOT STARTED
nf-core/rnaseq end-to-end on the cluster with a full Falco audit trail captured and output in MinIO. This is the integration test for everything built in Phases 0-5.

### [[PH-07 GxP Validation Documentation]] -- NOT STARTED (drafting starts at PH-04)
IQ, OQ, and PQ documents plus supporting SOPs. I write these to the standard a pharma QA auditor would accept, not as an afterthought.

---

## Dependency Chain

```
PH-00 (storage class, node labels)
  PH-01 (Forgejo needs PVC on local-hdd)
    PH-02 (Authentik needs Forgejo for OIDC config)
      PH-03 (MinIO needs sealed secrets, ingress)
        PH-04 (Gatekeeper validates all subsequent workloads)
          PH-05 (Falco DaemonSet, routes to Loki)
            PH-06 (Nextflow needs MinIO + RBAC + Falco active)
              PH-07 (docs reference evidence from all phases)
```

I start drafting PH-07 in parallel from PH-04 -- the structure is known before PH-06 is done.
