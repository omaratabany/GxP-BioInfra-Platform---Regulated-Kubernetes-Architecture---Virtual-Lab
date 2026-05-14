# PRJ-02 Phase Map and Schedule

> Part of [[README]] | See also: [[PRJ-01 Strategic Rationale]], [[INF-02 Architecture and Components]]

---

## Schedule

Status column reconciled against the live cluster on 2026-05-14. See [[OPS-03 Implementation Log]] for the verification entry.

| Week | CKA Domain | Project Phase | Status |
|---|---|---|---|
| 1-2 | Cluster architecture, etcd | [[PH-00 Cluster Preparation]] | COMPLETE |
| 3 | Workloads | [[PH-01 Forgejo]] | COMPLETE |
| 4 | Scheduling, affinity | [[PH-02 Authentik SSO]] | IN PROGRESS |
| 5 | Networking | [[PH-03 MinIO Object Storage]] | IN PROGRESS |
| 6 | Storage | [[PH-04 OPA Gatekeeper]] | NOT STARTED |
| 7 | RBAC, troubleshooting | [[PH-05 Falco Runtime Security]] + [[PH-06 Nextflow and nf-core]] | NOT STARTED |
| 8 | Exam sim | [[PH-06 Nextflow and nf-core]] -- pipeline run | NOT STARTED |
| Post-CKA | -- | [[PH-07 GxP Validation Documentation]] + public README | NOT STARTED |

---

## Phase Summary

### [[PH-00 Cluster Preparation]] -- COMPLETE
My foundation is locked. Node labels are applied, the Omen is untainted, the Beelink HDD is mounted and verified, `local-hdd` is the default StorageClass, and a real PVC test proved provisioning onto the Beelink HDD. The PH-00 etcd snapshot is saved outside the Git repo at `../snapshots/phase0-complete.snapshot`.

### [[PH-01 Forgejo]] -- COMPLETE
Forgejo is deployed and healthy on the Beelink storage node with a `local-hdd` PVC, SealedSecret admin credentials, TLS from the homelab CA issuer, and SSH exposed through NodePort `32222`. The MetalLB VIP is reachable on 80/443, the platform repo `gxp-admin/gxp-platform` exists in Forgejo, and the ArgoCD root Application now reads from `http://forgejo-http.forgejo.svc.cluster.local:3000/gxp-admin/gxp-platform.git`.

### [[PH-02 Authentik SSO]] -- IN PROGRESS
Authentik server, worker, and PostgreSQL are running. OIDC clients are configured for Forgejo, ArgoCD, and Grafana, with the platform groups (`platform-admin`, `developer`, `readonly`) created. Remaining: end-to-end browser logins per app, confirmation of group-claim role mapping in live tokens, and disabling local password fallback.

### [[PH-03 MinIO Object Storage]] -- IN PROGRESS
MinIO is live on the Beelink HDD with a 200Gi PVC, all four buckets (`pipeline-input`, `pipeline-output`, `pipeline-work`, `loki-chunks`) created with versioning and lifecycle as planned, and least-privilege `nextflow-sa` / `loki-sa` users provisioned. Remaining: reconfigure Loki to write to `loki-chunks` and migrate Prometheus TSDB onto `local-hdd` (G-11 addendum in [[PH-03 MinIO Object Storage]]).

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
