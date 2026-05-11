# REG-03 Risk Register

> Part of [[README]] | See also: [[REG-01 Compliance Matrix]], [[SEC-01 Security Architecture]], [[OPS-05 Disaster Recovery Plan]]

Formal risk register for the GxP BioInfra Platform. Each risk is assessed for likelihood and impact, has a mitigation strategy, and a residual risk rating. Reviewed and updated at each phase completion.

---

## Risk Scoring Methodology

**Likelihood:**
| Score | Label | Definition |
|---|---|---|
| 1 | Rare | Unlikely to occur in the life of this platform |
| 2 | Unlikely | Could occur but historical precedent is low |
| 3 | Possible | Could occur -- similar events are known |
| 4 | Likely | Will probably occur at some point |
| 5 | Almost certain | Expected to occur without intervention |

**Impact:**
| Score | Label | Definition |
|---|---|---|
| 1 | Negligible | No effect on platform function or compliance |
| 2 | Minor | Minor disruption, recoverable within 1 hour |
| 3 | Moderate | Significant disruption, recoverable within 1 day |
| 4 | Major | Loss of compliance evidence or significant data |
| 5 | Critical | Total data loss or regulatory submission invalidated |

**Risk Rating = Likelihood x Impact**

| Rating | 1-4 | 5-9 | 10-16 | 17-25 |
|---|---|---|---|---|
| Level | Low | Medium | High | Critical |

---

## Risk Register

### R-01 -- Beelink HDD Failure (Single Point of Storage Failure)

| Field | Detail |
|---|---|
| **Category** | Infrastructure |
| **Description** | The Beelink HDD at /var/mnt/hdd is the single physical disk holding all persistent data: MinIO buckets, Loki chunks, Forgejo repos, Prometheus TSDB. Drive failure loses everything with no automated recovery. |
| **Affected assets** | Pipeline output data, audit trail logs, Git repo history, infrastructure secrets (recoverable from SealedSecrets), Authentik user data |
| **Likelihood** | 3 -- Possible (HDD MTBF is measured in years; 320GB HDD is modest workload) |
| **Impact** | 5 -- Critical (all persistent data lost if no backup) |
| **Raw risk** | 15 -- High |
| **Controls** | MinIO bucket mirror to external target (daily, mc mirror). etcd snapshot (before each phase). MinIO loki-chunks bucket lock (30-day GOVERNANCE prevents deletion). |
| **Residual likelihood** | 3 (drive failure probability unchanged) |
| **Residual impact** | 2 (data recoverable from external mirror, loss window is < 24 hours) |
| **Residual risk** | 6 -- Medium |
| **Owner** | Operator |
| **Review trigger** | Drive shows SMART errors, after any storage incident |
| **References** | [[OPS-05 Disaster Recovery Plan]] Scenario 2, [[SEC-04 Secrets and Key Management]] |

---

### R-02 -- Single Control Plane Node (No HA)

| Field | Detail |
|---|---|
| **Category** | Infrastructure |
| **Description** | The HP Omen is the only control plane node. If it fails, the K8s API server goes down, etcd becomes unavailable, and no new pods can be scheduled. Running workloads continue until they need rescheduling, at which point they are also lost. |
| **Affected assets** | All cluster workloads, etcd state, API server availability |
| **Likelihood** | 2 -- Unlikely (Omen is a desktop machine, not a server; risk is real but low for short deployment windows) |
| **Impact** | 4 -- Major (cluster unavailable until Omen is restored; recovery from etcd snapshot) |
| **Raw risk** | 8 -- Medium |
| **Controls** | etcd snapshots before each phase transition (at least weekly). talosconfig and kubeconfig stored on Mac separately. Recovery documented in OPS-05. |
| **Residual likelihood** | 2 |
| **Residual impact** | 3 (recovery from snapshot is documented and tested) |
| **Residual risk** | 6 -- Medium |
| **Owner** | Operator |
| **Review trigger** | Any Omen hardware issue, before each phase transition |
| **References** | [[OPS-05 Disaster Recovery Plan]] Scenario 1, [[INF-03 Infrastructure Analysis]] Limit 1 |

---

### R-03 -- Supply Chain Attack via Container Image

| Field | Detail |
|---|---|
| **Category** | Security |
| **Description** | A pipeline pod pulls a container image that has been tampered with, either via tag mutation (image at :latest changed) or via a compromised registry. The image executes malicious code within the pipeline namespace. |
| **Affected assets** | Pipeline data, audit trail integrity, cluster compute |
| **Likelihood** | 2 -- Unlikely (nf-core images are community-maintained and widely scrutinised, but risk exists) |
| **Impact** | 4 -- Major (malicious code in a pipeline pod could exfiltrate data or corrupt results) |
| **Raw risk** | 8 -- Medium |
| **Controls** | Gatekeeper require-image-digest (digest pinning prevents tag mutation). Gatekeeper approved-registries (only quay.io, ghcr.io, docker.io, registry.k8s.io, forgejo.homelab). Falco runtime rules detect unexpected behaviour even if image is compromised. |
| **Residual likelihood** | 1 (digest pinning makes silent substitution essentially impossible) |
| **Residual impact** | 3 (Falco detects and alerts even if the image executes) |
| **Residual risk** | 3 -- Low |
| **Owner** | Operator |
| **Review trigger** | Any Gatekeeper approved-registry violation, any new pipeline image introduced |
| **References** | [[PH-04 OPA Gatekeeper]], [[PH-05 Falco Runtime Security]], [[SEC-01 Security Architecture]] |

---

### R-04 -- Flannel CNI -- No NetworkPolicy Enforcement

| Field | Detail |
|---|---|
| **Category** | Security / Compliance |
| **Description** | Flannel does not enforce NetworkPolicy objects. Any pod in the cluster can directly connect to any other pod on any port. A compromised pipeline pod could attempt to connect to Authentik's PostgreSQL, MinIO's admin API, or any other internal service. |
| **Affected assets** | All internal services; identity data in Authentik PostgreSQL; all MinIO buckets |
| **Likelihood** | 2 -- Unlikely (requires a compromised image; see R-03 mitigations) |
| **Impact** | 4 -- Major (access to Authentik DB or MinIO admin could lead to full platform compromise) |
| **Raw risk** | 8 -- Medium |
| **Controls** | Gatekeeper image controls reduce the likelihood of a compromised image. MinIO IAM requires valid credentials regardless of network access. RBAC prevents pods from accessing K8s API resources beyond their ServiceAccount. Falco detects unexpected connections. |
| **Planned mitigation** | Cilium migration post-CKA will enforce NetworkPolicy and eliminate this risk. |
| **Residual likelihood** | 2 (until Cilium) |
| **Residual impact** | 3 (MinIO IAM and RBAC are secondary controls) |
| **Residual risk** | 6 -- Medium |
| **Owner** | Operator |
| **Review trigger** | Post-CKA Cilium migration (this risk is eliminated after migration) |
| **References** | [[SEC-05 Network Security Policy]], [[ADR-00 Decision Log]] D-02 |

---

### R-05 -- Audit Trail Stored On-Cluster Only

| Field | Detail |
|---|---|
| **Category** | Compliance |
| **Description** | Falco events flow to Loki, and Loki stores data in MinIO. MinIO is on the same cluster and on the same physical node (Beelink). If the cluster is compromised or the Beelink HDD fails, the audit trail is lost or tampered. An external SIEM would remove this risk. |
| **Affected assets** | Falco audit trail (Annex 11 Clause 9 compliance) |
| **Likelihood** | 2 -- Unlikely |
| **Impact** | 4 -- Major (loss of audit trail invalidates Annex 11 compliance claim) |
| **Raw risk** | 8 -- Medium |
| **Controls** | MinIO loki-chunks bucket lock (GOVERNANCE 30 days) prevents deletion. Falco operates at kernel level (cannot be disabled from within a container). R-01 mitigations (daily mirror) protect against HDD failure. |
| **Planned mitigation** | Ship Falco events to an external Loki or SIEM instance as a future improvement. |
| **Residual likelihood** | 2 |
| **Residual impact** | 3 (loki-chunks mirror and GOVERNANCE lock provide partial protection) |
| **Residual risk** | 6 -- Medium |
| **Owner** | Operator |
| **Review trigger** | Any Loki ingestion failure, any MinIO storage alert |
| **References** | [[SEC-01 Security Architecture]], [[REG-01 Compliance Matrix]] |

---

### R-06 -- Sealed Secrets Controller Key Loss

| Field | Detail |
|---|---|
| **Category** | Security / Operations |
| **Description** | The Sealed Secrets controller holds the private key used to decrypt all SealedSecrets in the cluster. If the controller namespace is accidentally deleted and the key is not backed up, all SealedSecrets become permanently unrecoverable. All credentials must be regenerated and redeployed. |
| **Affected assets** | All service credentials (Forgejo, MinIO, Authentik, OIDC secrets) |
| **Likelihood** | 2 -- Unlikely (requires accidental deletion of the sealed-secrets namespace) |
| **Impact** | 3 -- Moderate (credentials must be regenerated; platform is unavailable until redeployment) |
| **Raw risk** | 6 -- Medium |
| **Controls** | Key backup procedure in SEC-04. Backup stored encrypted outside the cluster. Key rotation every 30 days (old keys retained for decryption). |
| **Residual likelihood** | 1 (backup exists and is tested) |
| **Residual impact** | 2 (recovery from backup is straightforward) |
| **Residual risk** | 2 -- Low |
| **Owner** | Operator |
| **Review trigger** | Before any cluster rebuild; after each 30-day key rotation |
| **References** | [[SEC-04 Secrets and Key Management]], [[SEC-02 Component Hardening Guide]] Sealed Secrets section |

---

### R-07 -- Pipeline Pod Resource Exhaustion

| Field | Detail |
|---|---|
| **Category** | Availability |
| **Description** | A runaway Nextflow process or misconfigured pipeline consumes all available CPU or RAM on the Omen node, causing existing platform services (ArgoCD, Authentik, Gatekeeper) to be evicted or crash. |
| **Affected assets** | Platform service availability; in-flight pipeline run |
| **Likelihood** | 3 -- Possible (memory-intensive bioinformatics tasks can exceed configured limits) |
| **Impact** | 3 -- Moderate (platform services disrupted; pipeline run fails) |
| **Raw risk** | 9 -- Medium |
| **Controls** | ResourceQuota on pipelines namespace (requests.cpu: 6, requests.memory: 10Gi). Gatekeeper require-resource-limits enforces limits on every pod. Platform services run with their own resource requests ensuring QoS class. |
| **Residual likelihood** | 2 (ResourceQuota hard-caps pipeline consumption) |
| **Residual impact** | 2 (platform services protected by QoS; pipeline fails gracefully) |
| **Residual risk** | 4 -- Low |
| **Owner** | Operator |
| **Review trigger** | Any OOMKilled event in the pipelines namespace |
| **References** | [[PH-06 Nextflow and nf-core]] ResourceQuota section, [[PH-04 OPA Gatekeeper]] |

---

### R-08 -- Cloudflare Tunnel Availability Dependency

| Field | Detail |
|---|---|
| **Category** | Availability |
| **Description** | External access to the platform depends on Cloudflare Tunnel, which depends on Cloudflare's availability and the internet connection. If either fails, external users cannot access Forgejo, ArgoCD, or Grafana. Internal LAN access is unaffected. |
| **Affected assets** | External platform access only; internal access unaffected |
| **Likelihood** | 3 -- Possible (internet outages and Cloudflare incidents occur) |
| **Impact** | 1 -- Negligible (all operations can continue from the LAN; no data is at risk) |
| **Raw risk** | 3 -- Low |
| **Controls** | All services accessible internally via *.homelab DNS. Tunnel is external-only. |
| **Residual risk** | 3 -- Low |
| **Owner** | Operator |
| **References** | [[INF-01 Infrastructure Baseline]] Network section |

---

## Risk Summary Dashboard

| Risk ID | Description | Raw Rating | Residual Rating | Status |
|---|---|---|---|---|
| R-01 | Beelink HDD failure | 15 High | 6 Medium | Active -- mitigated by daily mirror |
| R-02 | Single control plane node | 8 Medium | 6 Medium | Active -- mitigated by etcd snapshots |
| R-03 | Supply chain image attack | 8 Medium | 3 Low | Active -- mitigated by Gatekeeper + Falco |
| R-04 | Flannel no NetworkPolicy | 8 Medium | 6 Medium | Active -- eliminated by Cilium post-CKA |
| R-05 | Audit trail on-cluster only | 8 Medium | 6 Medium | Active -- partial mitigation by GOVERNANCE lock |
| R-06 | Sealed Secrets key loss | 6 Medium | 2 Low | Active -- mitigated by encrypted backup |
| R-07 | Pipeline resource exhaustion | 9 Medium | 4 Low | Active -- mitigated by ResourceQuota |
| R-08 | Cloudflare Tunnel dependency | 3 Low | 3 Low | Accepted |

**Accepted risks (no further mitigation planned for current tier):** R-02 (requires HA hardware), R-04 (Cilium migration planned), R-05 (external SIEM is future state), R-08 (LAN access unaffected).

---

## Corrections Log

**G-18 Fix -- Mirror Frequency Correction (May 2026)**

R-01 Controls section originally stated "daily, mc mirror." This contradicts [[OPS-04 Operational Runbook]] WK-02 which defines the mirror as weekly (Sundays). The correct frequency is **weekly**. The Controls section for R-01 should read: "MinIO bucket mirror to external target (weekly, mc mirror -- see OPS-04 WK-02)." The residual impact rating of 2 assumes a maximum data loss window of 7 days (one mirror cycle), not 24 hours as previously stated. Residual impact remains 2 (Minor -- recoverable within acceptable window for homelab context) but the loss window is up to 7 days, not 24 hours. For production, daily mirroring would be appropriate.

Updated risk summary for R-01:
- Controls: weekly MinIO mirror + GOVERNANCE bucket lock + etcd snapshots
- Maximum data loss window: up to 7 days (between mirror runs)
- Residual impact: 2 (Minor -- data recoverable from weekly mirror)
- Residual risk: 6 -- Medium (unchanged)
