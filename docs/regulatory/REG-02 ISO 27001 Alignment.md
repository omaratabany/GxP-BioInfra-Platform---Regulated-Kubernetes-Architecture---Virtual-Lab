# REG-02 ISO 27001 Alignment

> Part of [[README]] | See also: [[REG-01 Compliance Matrix]], [[REG-03 Risk Register]], [[SEC-01 Security Architecture]]

Detailed mapping of ISO/IEC 27001:2022 Annex A controls to specific platform components and configurations. This file is the evidence base for an ISO 27001 gap analysis or certification audit.

ISO 27001:2022 reorganised from 14 domains and 114 controls (2013 edition) to 4 themes and 93 controls. I reference the 2022 structure below with the legacy A.x reference in brackets where applicable.

---

## Theme 1 -- Organisational Controls (5.x)

### 5.1 Policies for Information Security [A.5.1]

| Requirement | Implementation | File |
|---|---|---|
| Information security policy exists and is approved | This vault constitutes the policy documentation. SEC-01 is the security architecture policy. | [[SEC-01 Security Architecture]] |
| Policy is communicated to all relevant parties | Platform Primer serves as the accessible version for any user of the platform | [[Platform Primer]] |
| Policy is reviewed at defined intervals | Review triggered by: each phase completion, each incident, or every 6 months | [[OPS-03 Implementation Log]] review entries |

### 5.15 Access Control [A.9.1, A.9.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| Access control policy | Authentik group-based access: platform-admin, developer, readonly | [[PH-02 Authentik SSO]] |
| User access rights assigned based on need-to-know | K8s RBAC scoped to namespace; nextflow-runner Role not ClusterRole | [[PH-06 Nextflow and nf-core]] RBAC section |
| Privileged access rights restricted | Gatekeeper disallow-privileged prevents any container running as privileged | [[PH-04 OPA Gatekeeper]] |
| Access rights reviewed | Quarterly audit procedure in SEC-04 | [[SEC-04 Secrets and Key Management]] |
| Access removed on user departure | Access Control SOP: disable in Authentik = immediate revocation across all apps | [[PH-07 GxP Validation Documentation]] SOPs |

### 5.17 Authentication Information [A.9.4.3]

| Requirement | Implementation | Evidence |
|---|---|---|
| Allocation and management of authentication info | All service credentials in SealedSecrets; no shared passwords | [[SEC-04 Secrets and Key Management]] |
| MFA for privileged access | Authentik enforces TOTP or WebAuthn for platform-admin group | [[SEC-02 Component Hardening Guide]] Authentik section |
| Session management | Authentik session tokens expire after 12 hours for admin accounts | [[SEC-02 Component Hardening Guide]] |
| Password strength requirements | Minimum 32-character random strings for service accounts | [[SEC-04 Secrets and Key Management]] |

### 5.23 Information Security for Cloud Services [A.15]

| Requirement | Implementation | Evidence |
|---|---|---|
| Cloud service acquisition policies | Cloudflare Tunnel is the only cloud dependency for external access | [[INF-01 Infrastructure Baseline]] |
| Risk assessment for cloud services | Cloudflare Tunnel risk: availability dependency only (not data processing) | [[REG-03 Risk Register]] |
| Exit strategy for cloud services | Cloudflare Tunnel can be replaced with a WireGuard VPN or direct IP (future) | [[ADR-01 Alternative Configurations]] |

---

## Theme 2 -- People Controls (6.x)

### 6.3 Information Security Awareness [A.7.2.2]

| Requirement | Implementation |
|---|---|
| Security training for all personnel | [[Platform Primer]] covers all tools and security model in plain language |
| Awareness of platform security policies | SEC-01 through SEC-05 are part of the platform documentation |
| Training records | To be added to [[OPS-03 Implementation Log]] as a training log section |

### 6.8 Reporting of Security Events [A.16.1.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| Mechanism to report security events | Falco -> Loki -> Grafana alert pipeline | [[PH-05 Falco Runtime Security]] |
| Incident log | [[OPS-03 Implementation Log]] incident section | [[SEC-03 Incident Response Playbook]] |
| Responsible party defined | Operator (solo project) -- see incident escalation table | [[SEC-03 Incident Response Playbook]] |

---

## Theme 3 -- Physical Controls (7.x)

### 7.6 Working in Secure Areas [A.11.1.5]

| Requirement | Implementation |
|---|---|
| Physical access to hardware | Homelab: nodes are in a physically controlled environment |
| No unauthorised physical access to nodes | Talos API is the only management interface; no USB boot, no console access without physical presence |

### 7.14 Secure Disposal [A.11.2.7]

| Requirement | Implementation |
|---|---|
| Secure disposal of hardware containing data | Talos `talosctl reset` wipes all data partitions before node decommissioning |
| Procedure documented | Add to Backup and Recovery SOP: node decommission step |

---

## Theme 4 -- Technological Controls (8.x)

### 8.2 Privileged Access Rights [A.9.2.3]

| Requirement | Implementation | Evidence |
|---|---|---|
| Privileged access is restricted and controlled | No cluster-admin binding for any non-system service account | `kubectl get clusterrolebindings` -- verify only system accounts have cluster-admin |
| Privileged access is time-limited where possible | Admin Authentik sessions expire after 12 hours | [[SEC-02 Component Hardening Guide]] |
| Privileged operations are logged | ArgoCD logs all sync operations; Falco logs all privileged syscalls | Grafana Loki + ArgoCD history |

### 8.4 Information in Use [A.9.4.1]

| Requirement | Implementation | Evidence |
|---|---|---|
| Access to information restricted per access rights | MinIO IAM policies per bucket per service account | [[SEC-02 Component Hardening Guide]] MinIO section |
| Namespace isolation | K8s RBAC scoped to namespace | [[PH-06 Nextflow and nf-core]] RBAC section |
| Data classification and handling | Pipeline data in dedicated buckets; audit logs in separate bucket with retention lock | [[PH-03 MinIO Object Storage]] |

### 8.7 Protection Against Malware [A.12.2.1]

| Requirement | Implementation | Evidence |
|---|---|---|
| Anti-malware controls | Gatekeeper approved-registries + require-image-digest prevents unknown code from running | [[PH-04 OPA Gatekeeper]] |
| Runtime threat detection | Falco eBPF rules detect malicious behaviour patterns | [[PH-05 Falco Runtime Security]] |
| Scanning of incoming content | Planned: Trivy image scanning in ArgoCD pipeline | Future |

### 8.8 Management of Technical Vulnerabilities [A.12.6.1]

| Requirement | Implementation | Evidence |
|---|---|---|
| Vulnerability identification | Gatekeeper blocks unverified images; image digest prevents silent updates | [[PH-04 OPA Gatekeeper]] |
| Vulnerability remediation | Image update via Forgejo PR + ArgoCD (controlled change) | [[PH-01 Forgejo]] |
| Patch management timeline | Images updated via PR within 30 days of security advisory | Change Control SOP |
| Vulnerability scanning | Planned: Trivy scan in CI pipeline | Future |

### 8.12 Data Leakage Prevention [A.13.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| DLP controls on egress | Falco unexpected outbound connection rule; MinIO IAM prevents unauthorised data access | [[PH-05 Falco Runtime Security]] |
| Network segmentation | Flannel today (no enforcement), Cilium post-CKA (full NetworkPolicy enforcement) | [[SEC-05 Network Security Policy]] |
| Monitoring for data exfiltration | Falco outbound rule generates WARNING event for any unexpected egress from pipelines namespace | [[PH-05 Falco Runtime Security]] custom rules |

### 8.15 Logging [A.12.4.1]

| Requirement | Implementation | Evidence |
|---|---|---|
| Logging of user activities | Authentik login events; ArgoCD operation log; Falco exec events | Grafana Loki |
| Logging of exceptions and security events | Falco AUDIT and CRITICAL events; Gatekeeper violations | Grafana Loki + Gatekeeper audit |
| Log protection | Falco captures at kernel level; Loki on MinIO with bucket lock (GOVERNANCE) | [[SEC-02 Component Hardening Guide]] |
| Log retention | MinIO loki-chunks bucket lock: 30 days minimum GOVERNANCE retention | [[SEC-02 Component Hardening Guide]] MinIO section |
| Log review | Grafana dashboards for Falco events; Prometheus alerting for anomalies | [[PH-05 Falco Runtime Security]] |
| Time synchronisation | Talos uses NTP by default; all node clocks synchronised | Verify: `talosctl -n 192.168.0.134 time` |

### 8.16 Monitoring [A.12.4.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| Network monitoring | Prometheus node metrics; Grafana dashboards | Grafana -> Kubernetes -> Nodes |
| Security event monitoring | Falco alert stream in Grafana; Prometheus alerting | Grafana -> Falco dashboard |
| Capacity monitoring | Prometheus disk, CPU, RAM metrics; ResourceQuota enforcement | Grafana -> Resource usage |
| Hubble network observability | Planned post-Cilium | [[SEC-05 Network Security Policy]] |

### 8.29 Secure Development Lifecycle [A.14.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| Secure development policy | All changes via Forgejo PR with required review | Change Control SOP |
| Security testing in development | OQ scripted test cases run before any phase is closed | [[PH-07 GxP Validation Documentation]] |
| Secure configuration management | Helm values pinned to chart versions; ArgoCD tracks all configuration | ArgoCD application history |
| Security requirements for system acquisition | Gatekeeper approved-registries + require-image-digest | [[PH-04 OPA Gatekeeper]] |

### 8.32 Change Management [A.12.1.2]

| Requirement | Implementation | Evidence |
|---|---|---|
| Change control procedure | Forgejo PR required for all changes; ArgoCD enforces GitOps | Change Control SOP |
| Impact assessment | PR description must include reason and impact | Forgejo PR history |
| Testing before production | OQ test suite re-run after significant changes | [[PH-07 GxP Validation Documentation]] OQ section |
| Emergency change procedure | Emergency changes can bypass PR review with incident documentation | [[SEC-03 Incident Response Playbook]] |
| Change record | Git commit history is the change record; ArgoCD sync history provides timestamps | Forgejo + ArgoCD |

### 8.34 Protection of Information Systems During Audit Testing [A.14.2.8]

| Requirement | Implementation | Evidence |
|---|---|---|
| Audit requirements planned | OQ test scripts documented before execution | [[PH-07 GxP Validation Documentation]] |
| Audit access controlled | OQ tests run by operator with platform-admin role only | Authentik access control |
| Audit does not disrupt production | Test profile uses synthetic data; tests run in dedicated pipelines namespace | [[PH-06 Nextflow and nf-core]] |

---

## ISO 27001 Certification Readiness Assessment

For reference when evaluating readiness for a formal ISO 27001 audit.

| Area | Current State | Gap | Priority |
|---|---|---|---|
| ISMS scope definition | Defined (this platform) | Formal scope statement needed | Medium |
| Risk assessment methodology | Qualitative (see REG-03) | Quantitative scoring not yet applied | Low |
| Statement of Applicability (SoA) | This document serves as informal SoA | Formal SoA document needed for certification | Medium |
| Internal audit procedure | OQ tests + quarterly secret audit | Scheduled audit calendar not defined | Medium |
| Management review | Solo project -- not applicable | N/A for homelab | Out of scope |
| Supplier assessment | Helm chart provenance checked; no formal supplier register | Formal supplier register needed for certification | Low |
| Incident management | SEC-03 playbook exists | No formal incident tracking system beyond OPS-03 | Low |
| Business continuity | OPS-05 DR plan | DR plan not formally tested | Medium |
