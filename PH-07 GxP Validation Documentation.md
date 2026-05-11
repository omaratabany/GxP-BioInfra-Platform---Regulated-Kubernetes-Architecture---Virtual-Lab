# PH-07 GxP Validation Documentation

> Part of [[README]] | Previous: [[PH-06 Nextflow and nf-core]] | Next: --
> Drafting starts in parallel from PH-04. IQ entries are filled in real time during PH-00 through PH-06 -- not reconstructed after the fact.

**Status: NOT STARTED**

---

## Goal

Written validation documentation that a pharma QA auditor would recognise and accept as evidence. Produced to the GAMP 5 Category 4/5 framework. Every entry in the IQ, OQ, and PQ references actual commands run against the actual cluster -- nothing is fabricated or templated from a generic source.

This is what distinguishes this project from every other Kubernetes portfolio. The IQ/OQ/PQ format is what regulated pharma environments require, and almost no platform engineers have written one.

---

## Document Set

All documents live in the Forgejo repository under `validation-docs/`:

```
validation-docs/
  IQ-Installation-Qualification.md
  OQ-Operational-Qualification.md
  PQ-Performance-Qualification.md
  SOPs/
    change-control.md
    audit-trail.md
    backup-recovery.md
    access-control.md
```

---

## IQ -- Installation Qualification Template

The IQ proves the system was installed correctly. Fill in each component entry as the phase completes. Do not backfill after Phase 7 -- the installation date must reflect when it actually happened.

---

```markdown
# Installation Qualification (IQ)
# GxP BioInfra Platform

| Field             | Value                                  |
|-------------------|----------------------------------------|
| Document number   | IQ-001                                 |
| Version           | 1.0                                    |
| Date created      | <date>                                 |
| Author            | <operator name>                        |
| Reviewed by       | <operator name>                        |
| Status            | DRAFT / APPROVED                       |
| Platform scope    | Talos Kubernetes cluster (2 nodes)     |
| GAMP 5 category   | Category 4 (config) + 5 (nf-core)     |

---

## 1. Purpose and Scope

This Installation Qualification documents that each software component of the GxP BioInfra Platform has been installed correctly and in accordance with its specifications. Scope covers all components deployed in PH-00 through PH-06.

Components classified as GAMP 5 Category 1 (Talos OS, Linux kernel, Kubernetes) are vendor-qualified. IQ entries are provided for completeness but full validation is not required for Category 1.

---

## 2. Infrastructure Qualification

### IQ-INF-01 -- Cluster Nodes

| Field              | Omen (CP)           | Beelink (Storage)   |
|--------------------|---------------------|---------------------|
| Hostname           | talos-asj-72z       | talos-v3h-4m1       |
| IP address         | 192.168.0.134       | 192.168.0.202       |
| CPU cores          | 8                   | 4                   |
| RAM                | 16 GB               | 7.5 GB              |
| OS                 | Talos vX.Y.Z        | Talos vX.Y.Z        |
| Role               | Control plane + infra | Storage            |
| Node label applied | node-role=infra     | node-role=storage   |
| Taint removed      | Yes (CP taint removed) | N/A              |
| Verification cmd   | `kubectl get nodes` | `kubectl get nodes` |
| Install date       |                     |                     |
| Verified by        |                     |                     |

### IQ-INF-02 -- Kubernetes Version

| Field            | Value                        |
|------------------|------------------------------|
| K8s version      |                              |
| etcd version     |                              |
| Verification cmd | `kubectl version`            |
| Verified by      |                              |
| Date             |                              |

### IQ-INF-03 -- Beelink HDD Mount

| Field             | Value                                         |
|-------------------|-----------------------------------------------|
| Device            | /dev/sdb                                      |
| Mountpoint        | /var/mnt/hdd                                  |
| Filesystem        | XFS                                           |
| Capacity          | 320 GB                                        |
| Talos patch file  | patches/beelink-hdd.yaml                      |
| Verification cmd  | `talosctl read /proc/mounts -n 192.168.0.202` |
| Verification output | (paste actual output here)                  |
| Mount confirmed   | Y / N                                         |
| Install date      |                                               |
| Verified by       |                                               |

---

## 3. Platform Component Qualification

One entry per component. Fill in immediately after the component's phase completes.

### IQ-PH00-01 -- local-path-provisioner

| Field              | Value                                              |
|--------------------|----------------------------------------------------|
| Component          | local-path-provisioner                             |
| GAMP 5 category    | 3 (non-configured COTS)                            |
| Chart name         |                                                    |
| Chart version      |                                                    |
| Image              |                                                    |
| Image digest       | sha256:                                            |
| Namespace          | local-path-storage                                 |
| Installed via      | ArgoCD                                             |
| Config file        | apps/local-path-provisioner/helm-release.yaml      |
| Config checksum    | `sha256sum apps/local-path-provisioner/helm-release.yaml` = |
| StorageClass name  | local-hdd                                          |
| Verification cmd   | `kubectl get sc local-hdd`                         |
| Verification output | (paste)                                           |
| Install date       |                                                    |
| Verified by        |                                                    |

### IQ-PH01-01 -- Forgejo

| Field              | Value                                   |
|--------------------|-----------------------------------------|
| Component          | Forgejo                                 |
| GAMP 5 category    | 4 (configured software)                 |
| Chart name         | forgejo/forgejo                         |
| Chart version      |                                         |
| Image              |                                         |
| Image digest       | sha256:                                 |
| Namespace          | forgejo                                 |
| Installed via      | ArgoCD                                  |
| Config file        | apps/forgejo/helm-release.yaml          |
| Config checksum    |                                         |
| PVC name           | forgejo-data                            |
| PVC node binding   | talos-v3h-4m1 (Beelink HDD)            |
| SSH host key fingerprint | (record here -- changes = alert)  |
| Admin account      | Created via Authentik SSO (no local pw) |
| Verification cmd   | `kubectl get pods -n forgejo`           |
| Verification output | (paste)                                |
| Install date       |                                         |
| Verified by        |                                         |

### IQ-PH02-01 -- Authentik

| Field              | Value                              |
|--------------------|------------------------------------|
| Component          | Authentik                          |
| GAMP 5 category    | 4 (configured software)            |
| Chart name         | authentik/authentik                |
| Chart version      |                                    |
| Image              |                                    |
| Image digest       | sha256:                            |
| Namespace          | authentik                          |
| Config file        | apps/authentik/helm-release.yaml   |
| Config checksum    |                                    |
| OIDC providers     | Forgejo, ArgoCD, Grafana           |
| Groups created     | platform-admin, developer, readonly |
| MFA enforced       | Y / N (platform-admin group)       |
| Verification cmd   | `kubectl get pods -n authentik`    |
| Verification output | (paste)                           |
| Local login disabled | Y / N (post-SSO)                 |
| Install date       |                                    |
| Verified by        |                                    |

### IQ-PH03-01 -- MinIO

| Field              | Value                             |
|--------------------|-----------------------------------|
| Component          | MinIO                             |
| GAMP 5 category    | 4 (configured software)           |
| Chart name         |                                   |
| Chart version      |                                   |
| Image              |                                   |
| Image digest       | sha256:                           |
| Namespace          | minio                             |
| Config file        | apps/minio/helm-release.yaml      |
| Config checksum    |                                   |
| PVC node binding   | talos-v3h-4m1 (Beelink HDD)      |
| Buckets created    | pipeline-input, pipeline-output, pipeline-work, loki-chunks |
| pipeline-work lifecycle | 30-day expiry              |
| loki-chunks lock   | GOVERNANCE 30 days                |
| pipeline-output versioning | Enabled                  |
| rootUser status    | Disabled after service accounts   |
| Service accounts   | nextflow-sa, loki-sa              |
| Verification cmd   | `mc ping homelab`                 |
| Verification output | (paste)                          |
| Install date       |                                   |
| Verified by        |                                   |

### IQ-PH04-01 -- OPA Gatekeeper

| Field              | Value                               |
|--------------------|-------------------------------------|
| Component          | OPA Gatekeeper                      |
| GAMP 5 category    | 4 (configured software)             |
| Chart name         | gatekeeper/gatekeeper               |
| Chart version      |                                     |
| Image              |                                     |
| Image digest       | sha256:                             |
| Namespace          | gatekeeper-system                   |
| Config file        | apps/gatekeeper/helm-release.yaml   |
| Config checksum    |                                     |
| failurePolicy      | Fail                                |
| Constraints deployed | require-resource-limits, require-image-digest, approved-registries, disallow-privileged, require-labels |
| All constraints in deny mode | Y / N                   |
| Verification cmd   | `kubectl get constraints -A`        |
| Verification output | (paste)                            |
| Install date       |                                     |
| Verified by        |                                     |

### IQ-PH05-01 -- Falco

| Field              | Value                             |
|--------------------|-----------------------------------|
| Component          | Falco + Falcosidekick             |
| GAMP 5 category    | 4 (configured software)           |
| Chart name         | falcosecurity/falco               |
| Chart version      |                                   |
| Image              |                                   |
| Image digest       | sha256:                           |
| Namespace          | monitoring                        |
| Driver type        | ebpf                              |
| Config file        | apps/falco/helm-release.yaml      |
| Custom rules file  | apps/falco/custom-rules-configmap.yaml |
| Rules checksum     |                                   |
| Falcosidekick target | Loki at http://loki.monitoring.svc:3100 |
| Loki connectivity  | Verified (Y / N)                  |
| DaemonSet nodes    | talos-asj-72z, talos-v3h-4m1     |
| Verification cmd   | `kubectl get pods -n monitoring -l app.kubernetes.io/name=falco -o wide` |
| Verification output | (paste)                          |
| Install date       |                                   |
| Verified by        |                                   |

### IQ-PH06-01 -- Nextflow + nf-core/rnaseq

| Field              | Value                             |
|--------------------|-----------------------------------|
| Component          | Nextflow                          |
| GAMP 5 category    | 4 (configured software)           |
| Version            |                                   |
| Executor           | k8s                               |
| Config file        | apps/nextflow/nextflow.config     |
| Config checksum    |                                   |
| Namespace          | pipelines                         |
| ServiceAccount     | nextflow-runner                   |
| ResourceQuota applied | Y / N                          |
| Verification cmd   | `nextflow -version`               |
| Verification output | (paste)                          |
| Install date       |                                   |
| Verified by        |                                   |

| Field              | Value                             |
|--------------------|-----------------------------------|
| Component          | nf-core/rnaseq                    |
| GAMP 5 category    | 5 (custom/community software)     |
| Version            |                                   |
| Source             | https://github.com/nf-core/rnaseq |
| Image registry     | quay.io/nf-core                   |
| Image digest       | Pinned in nextflow.config         |
| Config file        | apps/nextflow/nextflow.config     |
| Test profile run   | Y / N (Phase 6 validation run)    |
| Verification cmd   | `nextflow run nf-core/rnaseq -r <version> --version` |
| Verification output | (paste)                          |
| Install date       |                                   |
| Verified by        |                                   |

---

## 4. IQ Sign-Off

| Role        | Name | Signature | Date |
|-------------|------|-----------|------|
| Author      |      |           |      |
| Reviewer    |      |           |      |

---

## OQ -- Operational Qualification Template

The OQ proves the system operates as intended. All 10 test scripts are in [[OPS-06 OQ Test Scripts]]. Record actual results here.

---

```markdown
# Operational Qualification (OQ)
# GxP BioInfra Platform

| Field             | Value                                  |
|-------------------|----------------------------------------|
| Document number   | OQ-001                                 |
| Version           | 1.0                                    |
| Date executed     | <date>                                 |
| Operator          | <name>                                 |
| Platform state    | All phases PH-00 to PH-06 complete     |
| IQ reference      | IQ-001                                 |

---

## Pre-execution Confirmation

| Check | Result |
|-------|--------|
| All pods Running (no failures) | Y / N |
| All Gatekeeper constraints in deny mode | Y / N |
| Falco DaemonSet running on both nodes | Y / N |
| MinIO reachable (mc ping homelab) | Y / N |
| OPERATOR variable set | Y / N |

---

## OQ Test Results

| Test ID | Description | Annex 11 | Expected Result | Actual Result | Verdict | Time | Notes |
|---------|-------------|----------|-----------------|---------------|---------|------|-------|
| OQ-01 | Gatekeeper denies pod with no resource limits | 4.4 | Admission denied | | PASS/FAIL | | |
| OQ-02 | Gatekeeper denies pod with :latest image tag | 10 | Admission denied | | PASS/FAIL | | |
| OQ-03 | Falco AUDIT event on exec into pipeline pod | 9 | Event in Loki <10s | | PASS/FAIL | | |
| OQ-04 | Forgejo login via Authentik SSO | 12.1 | SSO succeeds, local pw rejected | | PASS/FAIL | | |
| OQ-05 | ArgoCD readonly group access | 12.1 | View only, no sync/delete | | PASS/FAIL | | |
| OQ-06 | Forgejo push triggers ArgoCD sync | 10 | Sync within 30s | | PASS/FAIL | | |
| OQ-07 | PVC from local-hdd binds to Beelink | IQ verify | Bound to talos-v3h-4m1 | | PASS/FAIL | | |
| OQ-08 | MinIO accessible from pipeline pod | Phase 3 | mc ls succeeds | | PASS/FAIL | | |
| OQ-09 | etcd snapshot creation | 7.1 | Non-zero snapshot file | | PASS/FAIL | | |
| OQ-10 | Falco CRITICAL on privilege escalation | 9 | CRITICAL in Loki <10s | | PASS/FAIL | | |

**Overall OQ Result: PASS / FAIL**

Deficiencies: ___

Corrective actions: ___

---

## Evidence Artifacts

| Test | Evidence File | Location |
|------|--------------|----------|
| OQ-01 | kubectl output screenshot | Forgejo validation-docs/evidence/ |
| OQ-03 | Loki query result | Forgejo validation-docs/evidence/ |
| OQ-06 | ArgoCD sync timestamp | ArgoCD history |
| OQ-07 | kubectl describe pvc output | Forgejo validation-docs/evidence/ |
| OQ-09 | Snapshot file path + size | ~/Kuber/snapshots/ |
| OQ-10 | Loki CRITICAL query result | Forgejo validation-docs/evidence/ |

---

## Sign-Off

| Role     | Name | Signature | Date |
|----------|------|-----------|------|
| Operator |      |           |      |
```

---

## PQ -- Performance Qualification Template

The PQ proves the system performs consistently and reproducibly. Two independent pipeline runs are required.

---

```markdown
# Performance Qualification (PQ)
# GxP BioInfra Platform -- nf-core/rnaseq

| Field             | Value                                     |
|-------------------|-------------------------------------------|
| Document number   | PQ-001                                    |
| Version           | 1.0                                       |
| Date executed     | <date>                                    |
| Operator          | <name>                                    |
| IQ reference      | IQ-001                                    |
| OQ reference      | OQ-001                                    |
| Pipeline          | nf-core/rnaseq                            |
| Pipeline version  |                                           |
| Profile           | test,k8s                                  |
| Reference genome  | Synthetic (test profile)                  |

---

## Pre-PQ Baseline

Record idle cluster metrics from [[OPS-07 Performance Baselines]] before starting either run.

| Metric | Idle Baseline (from OPS-07) |
|--------|---------------------------|
| Cluster CPU (cores) | |
| Omen RAM (GiB) | |
| Beelink RAM (GiB) | |
| Beelink HDD available | |

---

## Run 1

| Parameter | Value |
|-----------|-------|
| Start time | |
| End time | |
| Wall time (minutes) | |
| Output directory | s3://pipeline-output/rnaseq-pq-run1-<date> |
| Exit status | |

### Peak Resource Utilisation (from Grafana)

| Metric | Measured Peak | Acceptance Threshold | Pass/Fail |
|--------|--------------|---------------------|-----------|
| Omen CPU (cores) | | < 7.5 | |
| Omen RAM (GiB) | | < 14.0 | |
| Beelink RAM (GiB) | | < 6.5 | |
| Beelink HDD write (MB/s) | | < 100 | |

### Output Integrity

| Check | Value |
|-------|-------|
| All tasks completed (no failures) | Y / N |
| Number of output files | |
| MultiQC report generated | Y / N |
| MultiQC report path | |
| MultiQC MD5 checksum | `md5sum multiqc_report.html` = |
| MultiQC SHA256 checksum | `sha256sum multiqc_report.html` = |
| Falco events generated during run | Y / N |
| Audit trail gap detected | Y / N |

---

## Run 2

| Parameter | Value |
|-----------|-------|
| Start time | |
| End time | |
| Wall time (minutes) | |
| Output directory | s3://pipeline-output/rnaseq-pq-run2-<date> |
| Exit status | |

### Peak Resource Utilisation

| Metric | Measured Peak | Acceptance Threshold | Pass/Fail |
|--------|--------------|---------------------|-----------|
| Omen CPU (cores) | | < 7.5 | |
| Omen RAM (GiB) | | < 14.0 | |
| Beelink RAM (GiB) | | < 6.5 | |
| Beelink HDD write (MB/s) | | < 100 | |

### Output Integrity

| Check | Value |
|-------|-------|
| All tasks completed (no failures) | Y / N |
| MultiQC MD5 checksum | `md5sum multiqc_report.html` = |
| MultiQC SHA256 checksum | `sha256sum multiqc_report.html` = |

---

## Reproducibility Assessment

| Check | Run 1 | Run 2 | Match |
|-------|-------|-------|-------|
| Exit status | | | Y / N |
| Number of output files | | | Y / N |
| MultiQC MD5 | | | **Y / N** |
| MultiQC SHA256 | | | **Y / N** |
| Wall time within 20% | | | Y / N |

**Reproducibility verdict: PASS / FAIL**

Checksums must match for PASS. Wall time within 20% is informational.

---

## PQ Acceptance Criteria Summary

| Criterion | Threshold | Run 1 | Run 2 | Overall |
|-----------|-----------|-------|-------|---------|
| Pipeline completion | 100% tasks | | | PASS/FAIL |
| Wall time | < 60 min | | | PASS/FAIL |
| Omen peak RAM | < 14 GiB | | | PASS/FAIL |
| Beelink peak RAM | < 6.5 GiB | | | PASS/FAIL |
| Output reproducibility | MD5 match | N/A | | PASS/FAIL |
| Audit trail completeness | Falco events > 0 during run | | | PASS/FAIL |

**Overall PQ Result: PASS / FAIL**

---

## Sign-Off

| Role     | Name | Signature | Date |
|----------|------|-----------|------|
| Operator |      |           |      |
```

---

## Supporting SOPs

### Change Control SOP

**Purpose:** Ensure all infrastructure changes are reviewed, approved, and traceable.

**Trigger:** Any change to files in the Forgejo repository that will be applied to the cluster.

**Procedure:**
1. Create a branch from `main` in Forgejo
2. Make the change and commit with a descriptive message
3. Open a Pull Request with title, description of change, reason, and impact assessment
4. Reviewer (or self-review with mandatory waiting period for critical changes) approves the PR
5. Merge to `main` -- ArgoCD detects the change and syncs automatically
6. Verify the sync succeeded: `argocd app get <app> --refresh`
7. Log the change in [[OPS-03 Implementation Log]] with the PR number and merge SHA

**Change record format:**
```
Date: <date>
PR number: #<n>
Merge SHA: <sha>
Changed: <what>
Reason: <why>
Impact: <what could break>
Verified by: <operator>
```

**Emergency changes:** If a production issue requires immediate action without a PR, apply the change directly and open a retroactive PR within 24 hours documenting what was done. Log as a Change Control SOP deviation in [[OPS-03 Implementation Log]].

---

### Audit Trail SOP

**Purpose:** Ensure the audit trail is tamper-evident and accessible for regulatory review.

**Audit trail components:**
- Falco eBPF events → Falcosidekick → Loki (runtime operations)
- ArgoCD operation history (configuration changes)
- Forgejo commit and PR history (change records)

**Querying the audit trail:**
```
Grafana → Loki datasource
Query: {job="falco"} | json | namespace="pipelines"
Time range: Specify the period under review
Export: Download as CSV for regulatory submission
```

**Retention:** Loki chunks stored in MinIO with GOVERNANCE lock (30 days minimum). Longer retention via weekly mirror to external target.

**Tamper evidence:** Falco operates at eBPF level -- container processes cannot modify their own audit events. If the Falco DaemonSet is killed, that event itself is captured by the other node's Falco instance.

---

### Backup and Recovery SOP

**Purpose:** Ensure platform data can be recovered from failure.

**Backup schedule:**
- etcd: before each phase transition + weekly Sunday
- MinIO buckets: weekly Sunday mirror to external target
- Sealed Secrets key: after each 30-day rotation

**Recovery procedure reference:** [[OPS-05 Disaster Recovery Plan]]

**Recovery test:** DR procedure to be tested at least once per year. Results recorded in [[OPS-03 Implementation Log]].

---

### Access Control SOP

**Purpose:** Ensure platform access is granted, maintained, and revoked in a controlled manner.

**Granting access:** [[OPS-04 Operational Runbook]] PROC-01
**Revoking access:** [[OPS-04 Operational Runbook]] PROC-02
**Access review:** Quarterly -- list all Authentik users and verify each still requires access
**No shared accounts:** Every user has a named Authentik account
**No local passwords:** Authentik is the only identity source post-Phase 2

---

## Exit Criteria

All of the following must be true before Phase 7 is considered complete:

- [ ] IQ complete -- all component entries filled with actual values
- [ ] OQ complete -- all 10 tests PASS with recorded output
- [ ] PQ complete -- two runs, checksums match, all acceptance criteria met
- [ ] Change Control SOP active -- minimum 3 PRs merged through Forgejo
- [ ] Audit Trail SOP active -- Falco events queryable in Grafana
- [ ] Backup and Recovery SOP active -- at least one successful etcd snapshot and MinIO mirror
- [ ] Access Control SOP active -- all access via Authentik, no local passwords
- [ ] All SOPs stored in Forgejo under `validation-docs/SOPs/`
- [ ] All validation documents reviewed and signed
