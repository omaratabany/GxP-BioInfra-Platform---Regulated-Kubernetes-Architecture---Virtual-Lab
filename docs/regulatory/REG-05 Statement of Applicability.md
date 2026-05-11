# REG-05 Statement of Applicability

> Part of [[README]] | See also: [[REG-02 ISO 27001 Alignment]], [[REG-01 Compliance Matrix]], [[SEC-01 Security Architecture]]

Formal Statement of Applicability (SoA) for ISO/IEC 27001:2022 Annex A controls as applied to the GxP BioInfra Platform. This document is a mandatory component of any ISO 27001 conformance claim. It lists every Annex A control, states whether it is applicable to this platform, the justification for inclusion or exclusion, and where the implementation evidence lives.

---

## Document Information

| Field | Value |
|---|---|
| Document title | Statement of Applicability -- GxP BioInfra Platform |
| Standard | ISO/IEC 27001:2022 Annex A |
| Scope | GxP BioInfra Platform homelab cluster and associated management infrastructure |
| Version | 1.0 |
| Date | May 2026 |
| Author | Platform Operator |
| Review cycle | Every 6 months or after a significant platform change |

---

## How to Read This Document

**Applicable (Yes/No):** Whether the control applies to this platform and scope.
**Justification:** Why the control is included or excluded.
**Implementation:** What implements the control.
**Evidence:** Where to find proof of implementation.
**Status:** Implemented / Partial / Planned / Not applicable.

Exclusions are justified, not ignored. An excluded control has a documented reason.

---

## Organisational Controls (5.x)

| Control | Title | Applicable | Justification | Implementation | Evidence | Status |
|---|---|---|---|---|---|---|
| 5.1 | Policies for information security | Yes | Platform requires a security policy baseline | SEC-01 through SEC-05 form the policy set | This vault | Implemented |
| 5.2 | Information security roles and responsibilities | Yes | Operator role defined for solo project | Operator role: all responsibilities. Documented in onboarding guide. | [[OPS-08 Developer and Operator Onboarding]] | Implemented |
| 5.3 | Segregation of duties | Partial | Solo project -- full segregation not possible. Compensating control: GitOps enforces PR-based change review even for the same operator. | MOD-03 Configuration Management Standard -- self-review minimum | [[MOD-03 Configuration Management Standard]] | Partial |
| 5.4 | Management responsibilities | Yes | Operator is responsible for ensuring controls are maintained | OPS-04 monthly tasks and periodic review | [[OPS-04 Operational Runbook]] | Implemented |
| 5.5 | Contact with authorities | No | Homelab scope -- no regulatory filing obligations. Not applicable at this tier. | N/A | N/A | Not applicable |
| 5.6 | Contact with special interest groups | Yes | nf-core community membership; Falco community advisories; CNCF security channels | Supplier assessment tracks advisory channels | [[REG-04 Supplier Assessment Register]] | Implemented |
| 5.7 | Threat intelligence | Yes | Supplier security advisories consumed and actioned | Patch management procedure and supplier register | [[OPS-09 Patch Management]], [[REG-04 Supplier Assessment Register]] | Implemented |
| 5.8 | Information security in project management | Yes | Security controls built into every phase from the start | Phase-by-phase security control activation in SEC-01 | [[SEC-01 Security Architecture]] | Implemented |
| 5.9 | Inventory of information and other associated assets | Yes | Asset register exists | Asset table in SEC-01 threat model | [[SEC-01 Security Architecture]] | Implemented |
| 5.10 | Acceptable use of information and other associated assets | Yes | Pipeline data use restricted to research/validation purposes | MinIO IAM policies, RBAC | [[SEC-02 Component Hardening Guide]] | Implemented |
| 5.11 | Return of assets | No | No external users hold platform assets -- N/A at this scope | N/A | N/A | Not applicable |
| 5.12 | Classification of information | Yes | Asset classification defined (Critical, High, Moderate, Sensitive) | SEC-01 threat model asset table | [[SEC-01 Security Architecture]] | Partial -- no formal data classification policy document |
| 5.13 | Labelling of information | Partial | Kubernetes labels applied to all workloads via Gatekeeper require-labels | Labels include app, version, env -- not full data classification labels | [[PH-04 OPA Gatekeeper]] | Partial |
| 5.14 | Information transfer | Yes | All internal transfers over TLS. External access via Cloudflare Tunnel. | Cert-Manager, Ingress-NGINX TLS, Cloudflare Tunnel | [[SEC-05 Network Security Policy]] | Implemented |
| 5.15 | Access control | Yes | Core compliance requirement | Authentik OIDC, K8s RBAC, MinIO IAM | [[PH-02 Authentik SSO]], [[PH-03 MinIO Object Storage]] | Implemented |
| 5.16 | Identity management | Yes | All identities managed in Authentik | Authentik as single IdP -- no local accounts post-PH-02 | [[PH-02 Authentik SSO]] | Implemented |
| 5.17 | Authentication information | Yes | Password standards, MFA for admins, rotation schedule | SEC-04, Authentik MFA policy | [[SEC-04 Secrets and Key Management]] | Implemented |
| 5.18 | Access rights | Yes | Least privilege via namespace-scoped RBAC, MinIO per-bucket IAM | RBAC definitions, MinIO IAM policies | [[PH-06 Nextflow and nf-core]], [[SEC-02 Component Hardening Guide]] | Implemented |
| 5.19 | Information security in supplier relationships | Yes | All external software suppliers assessed | Supplier assessment register | [[REG-04 Supplier Assessment Register]] | Implemented |
| 5.20 | Addressing information security within supplier agreements | Partial | Open source -- no formal contracts. Mitigated by image digest pinning and advisory monitoring. | Gatekeeper image pinning, supplier advisory tracking | [[REG-04 Supplier Assessment Register]] | Partial |
| 5.21 | Managing information security in the ICT supply chain | Yes | Container image supply chain controlled via digest pinning and approved registries | Gatekeeper approved-registries + require-image-digest | [[PH-04 OPA Gatekeeper]] | Implemented |
| 5.22 | Monitoring, review and change management of supplier services | Yes | Monthly version review, supplier advisory monitoring | MO-04 monthly dependency review | [[OPS-04 Operational Runbook]] | Implemented |
| 5.23 | Information security for use of cloud services | Yes | Cloudflare Tunnel is the only cloud service used | Cloudflare Tunnel assessment in SUP-11 | [[REG-04 Supplier Assessment Register]] | Implemented |
| 5.24 | Information security incident management planning and preparation | Yes | Incident response playbook with severity classification | SEC-03 | [[SEC-03 Incident Response Playbook]] | Implemented |
| 5.25 | Assessment and decision on information security events | Yes | Severity classification P1-P4, triage procedures per scenario | SEC-03 | [[SEC-03 Incident Response Playbook]] | Implemented |
| 5.26 | Response to information security incidents | Yes | Per-scenario containment and recovery procedures | SEC-03 | [[SEC-03 Incident Response Playbook]] | Implemented |
| 5.27 | Learning from information security incidents | Yes | Post-incident review logged in OPS-03, preventive actions tracked | Incident log template in SEC-03 | [[OPS-03 Implementation Log]] | Implemented |
| 5.28 | Collection of evidence | Yes | Falco kernel-level capture, MinIO GOVERNANCE lock, ArgoCD history | OPS-06 OQ test scripts validate evidence collection | [[OPS-06 OQ Test Scripts]] | Implemented |
| 5.29 | Information security during disruption | Yes | DR plan covers all major disruption scenarios | OPS-05 | [[OPS-05 Disaster Recovery Plan]] | Implemented |
| 5.30 | ICT readiness for business continuity | Yes | etcd snapshots, MinIO mirroring, documented recovery procedures | OPS-04, OPS-05 | [[OPS-05 Disaster Recovery Plan]] | Implemented |
| 5.31 | Legal, statutory, regulatory and contractual requirements | Yes | EU GMP Annex 11, FDA 21 CFR Part 11 | REG-01 compliance matrix | [[REG-01 Compliance Matrix]] | Implemented |
| 5.32 | Intellectual property rights | Yes | All software used is under open source licenses (Apache 2.0, MIT, MPL) -- no proprietary software | Supplier register license column | [[REG-04 Supplier Assessment Register]] | Implemented |
| 5.33 | Protection of records | Yes | MinIO GOVERNANCE lock, etcd snapshots, Loki retention | SEC-02 MinIO hardening | [[SEC-02 Component Hardening Guide]] | Implemented |
| 5.34 | Privacy and protection of PII | Yes | No patient data processed. Test profile uses synthetic data only. nf-core test datasets are publicly available. | PH-06 test dataset note | [[PH-06 Nextflow and nf-core]] | Implemented |
| 5.35 | Independent review of information security | No | Solo project -- independent review not possible at this tier. Planned: external security review before any production use. | N/A | N/A | Not applicable |
| 5.36 | Compliance with policies, rules and standards | Yes | ArgoCD drift detection enforces all configuration policies | MOD-03, ArgoCD selfHeal | [[MOD-03 Configuration Management Standard]] | Implemented |
| 5.37 | Documented operating procedures | Yes | OPS-01 through OPS-09 | OPS- file series | All OPS- files | Implemented |

---

## People Controls (6.x)

| Control | Title | Applicable | Justification | Implementation | Evidence | Status |
|---|---|---|---|---|---|---|
| 6.1 | Screening | No | Solo project -- N/A | N/A | N/A | Not applicable |
| 6.2 | Terms and conditions of employment | No | Solo project -- N/A | N/A | N/A | Not applicable |
| 6.3 | Information security awareness, education and training | Yes | Operator must understand both K8s and GxP requirements | Platform Primer, this vault, OPS-08 onboarding guide | [[Platform Primer]], [[OPS-08 Developer and Operator Onboarding]] | Implemented |
| 6.4 | Disciplinary process | No | Solo project -- N/A | N/A | N/A | Not applicable |
| 6.5 | Responsibilities after termination or change of employment | Yes | Offboarding procedure disables access immediately | PROC-02 in OPS-04 | [[OPS-04 Operational Runbook]] | Implemented |
| 6.6 | Confidentiality or non-disclosure agreements | No | Solo project -- N/A | N/A | N/A | Not applicable |
| 6.7 | Remote working | Yes | Mac is the remote management station -- access only via kubeconfig and talosconfig | kubeconfig and talosconfig on Mac only, not synced to cloud | [[OPS-08 Developer and Operator Onboarding]] | Implemented |
| 6.8 | Information security event reporting | Yes | Falco alerts and incident log | SEC-03, OPS-03 incident log section | [[SEC-03 Incident Response Playbook]] | Implemented |

---

## Physical Controls (7.x)

| Control | Title | Applicable | Justification | Implementation | Evidence | Status |
|---|---|---|---|---|---|---|
| 7.1 | Physical security perimeters | Yes | Nodes physically located in a controlled home environment | Physical access restricted to operator | INF-01 | Implemented |
| 7.2 | Physical entry | Yes | No public access to the physical location of nodes | Physical access restricted | INF-01 | Implemented |
| 7.3 | Securing offices, rooms and facilities | Yes | As above | Physical access restricted | INF-01 | Implemented |
| 7.4 | Physical security monitoring | No | Homelab scope -- no physical security monitoring system | N/A | N/A | Not applicable |
| 7.5 | Protecting against physical and environmental threats | Yes | Nodes on UPS-protected power (if applicable) and in temperature-controlled environment | Physical setup | INF-01 | Partial |
| 7.6 | Working in secure areas | Yes | All cluster management done via kubeconfig from the Mac -- no direct physical console access required | Talos design: no console management interface | SEC-02 | Implemented |
| 7.7 | Clear desk and clear screen | No | Homelab scope -- N/A | N/A | N/A | Not applicable |
| 7.8 | Equipment siting and protection | Yes | Nodes are not co-located with untrusted equipment | Physical setup | INF-01 | Implemented |
| 7.9 | Security of assets off-premises | Yes | Mac with kubeconfig and talosconfig is the off-premises asset | Mac disk encryption required; kubeconfig not synced to cloud | [[OPS-08 Developer and Operator Onboarding]] | Implemented |
| 7.10 | Storage media | Yes | Beelink HDD contains all persistent data | SEC-04 key backup, OPS-05 DR plan | [[SEC-04 Secrets and Key Management]], [[OPS-05 Disaster Recovery Plan]] | Implemented |
| 7.11 | Supporting utilities | Partial | Power protection depends on physical setup | N/A documented | INF-01 | Partial |
| 7.12 | Cabling security | No | Homelab scope -- N/A | N/A | N/A | Not applicable |
| 7.13 | Equipment maintenance | Yes | Talos upgrades are the maintenance mechanism for nodes | MOD-02 upgrade procedures | [[MOD-02 Modularity and Dependency Map]] | Implemented |
| 7.14 | Secure disposal or reuse of equipment | Yes | `talosctl reset` wipes all data partitions before decommissioning | Talosctl reset procedure | [[OPS-05 Disaster Recovery Plan]] | Implemented |

---

## Technological Controls (8.x)

| Control | Title | Applicable | Justification | Implementation | Evidence | Status |
|---|---|---|---|---|---|---|
| 8.1 | User endpoint devices | Yes | Mac M5 is the management endpoint | Mac disk encryption, no cloud sync of credentials | [[OPS-08 Developer and Operator Onboarding]] | Implemented |
| 8.2 | Privileged access rights | Yes | No cluster-admin service accounts; nextflow-runner is namespace-scoped Role | RBAC definitions | [[PH-06 Nextflow and nf-core]] | Implemented |
| 8.3 | Information access restriction | Yes | MinIO IAM per bucket, K8s RBAC per namespace | IAM policies, RBAC | [[SEC-02 Component Hardening Guide]] | Implemented |
| 8.4 | Access to source code | Yes | Forgejo with branch protection and PR review | Forgejo configuration | [[PH-01 Forgejo]] | Implemented |
| 8.5 | Secure authentication | Yes | OIDC via Authentik, MFA for admins, no local passwords post-PH-02 | Authentik OIDC, MFA policy | [[SEC-02 Component Hardening Guide]] | Implemented |
| 8.6 | Capacity management | Yes | ResourceQuota on pipelines namespace, Gatekeeper resource limits, Prometheus alerting | ResourceQuota, Gatekeeper, OPS-07 baselines | [[OPS-07 Performance Baselines]] | Implemented |
| 8.7 | Protection against malware | Yes | Gatekeeper image policy + Falco runtime detection | Image pinning + runtime rules | [[PH-04 OPA Gatekeeper]], [[PH-05 Falco Runtime Security]] | Implemented |
| 8.8 | Management of technical vulnerabilities | Yes | Patch management procedure, supplier advisory monitoring | OPS-09, REG-04 | [[OPS-09 Patch Management]] | Implemented |
| 8.9 | Configuration management | Yes | All configuration in Git, ArgoCD drift detection | MOD-03 | [[MOD-03 Configuration Management Standard]] | Implemented |
| 8.10 | Information deletion | Yes | MinIO lifecycle policy on pipeline-work (30-day expiry), PVC Retain policy | MinIO ILM, StorageClass | [[PH-03 MinIO Object Storage]] | Implemented |
| 8.11 | Data masking | No | No PII or patient data processed. Test profile uses synthetic data only. | N/A | N/A | Not applicable |
| 8.12 | Data leakage prevention | Yes | Falco outbound connection rule, MinIO IAM, Cilium NetworkPolicy (planned) | Falco rules, IAM | [[PH-05 Falco Runtime Security]], [[SEC-05 Network Security Policy]] | Partial (Cilium pending) |
| 8.13 | Information backup | Yes | etcd snapshots weekly, MinIO bucket mirroring weekly | OPS-04 WK-01 and WK-02 | [[OPS-04 Operational Runbook]] | Implemented |
| 8.14 | Redundancy of information processing facilities | Partial | Single CP node is a known limitation. Mitigated by etcd snapshots and documented recovery. | R-02 in REG-03 | [[REG-03 Risk Register]] | Partial |
| 8.15 | Logging | Yes | Falco eBPF + Loki + Promtail -- all nodes covered | Falco DaemonSet, Loki backend on MinIO | [[PH-05 Falco Runtime Security]] | Implemented |
| 8.16 | Monitoring activities | Yes | Prometheus + Grafana, alerting rules | INF-06 | [[INF-06 Observability and Alerting]] | Implemented |
| 8.17 | Clock synchronisation | Yes | Talos NTP enabled by default | `talosctl time` verification | [[OPS-06 OQ Test Scripts]] | Implemented |
| 8.18 | Use of privileged utility programs | Yes | Gatekeeper disallow-privileged prevents privileged containers | Gatekeeper constraint | [[PH-04 OPA Gatekeeper]] | Implemented |
| 8.19 | Installation of software on operational systems | Yes | No software can be installed on Talos nodes -- immutable OS. Container images are the only installation mechanism and are controlled by Gatekeeper. | Talos immutability + Gatekeeper | [[SEC-01 Security Architecture]] | Implemented |
| 8.20 | Networks security | Yes | CGNAT + Cloudflare Tunnel + MetalLB ingress control | SEC-05 | [[SEC-05 Network Security Policy]] | Implemented |
| 8.21 | Security of network services | Yes | All services behind Ingress-NGINX with TLS termination | Ingress-NGINX, Cert-Manager | [[INF-07 Network Topology]] | Implemented |
| 8.22 | Segregation of networks | Partial | Namespace isolation via RBAC today. Full NetworkPolicy post-Cilium. | SEC-05 Cilium target state | [[SEC-05 Network Security Policy]] | Partial |
| 8.23 | Web filtering | No | Homelab scope -- no web filtering implemented | N/A | N/A | Not applicable |
| 8.24 | Use of cryptography | Yes | TLS everywhere (Cert-Manager), Sealed Secrets RSA encryption, etcd encryption at rest (Talos default) | SEC-04, SEC-05 | [[SEC-04 Secrets and Key Management]] | Implemented |
| 8.25 | Secure development lifecycle | Yes | GitOps PR workflow, Gatekeeper admission, OQ testing | MOD-03, OPS-06 | [[MOD-03 Configuration Management Standard]] | Implemented |
| 8.26 | Application security requirements | Yes | Gatekeeper constraints define security requirements for all admitted workloads | Gatekeeper constraints | [[PH-04 OPA Gatekeeper]] | Implemented |
| 8.27 | Secure system architecture and engineering principles | Yes | Defense-in-depth layered model, least privilege, immutable infrastructure | SEC-01 | [[SEC-01 Security Architecture]] | Implemented |
| 8.28 | Secure coding | Yes | All infrastructure-as-code in Forgejo with PR review | MOD-03 | [[MOD-03 Configuration Management Standard]] | Implemented |
| 8.29 | Security testing in development and acceptance | Yes | OQ test scripts validated before each phase closes | OPS-06 | [[OPS-06 OQ Test Scripts]] | Implemented |
| 8.30 | Outsourced development | No | No outsourced development -- all code is operator-written or open source | N/A | N/A | Not applicable |
| 8.31 | Separation of development, test and production environments | Partial | Single environment (homelab). Pipeline test profile uses synthetic data to separate test from production data. Namespace separation provides partial isolation. | pipelines namespace, test profile | [[PH-06 Nextflow and nf-core]] | Partial |
| 8.32 | Change management | Yes | Forgejo PR + ArgoCD GitOps is the change management system | MOD-03, Change Control SOP | [[MOD-03 Configuration Management Standard]] | Implemented |
| 8.33 | Test information | Yes | Test profile uses only publicly available synthetic data. No production data in test runs. | nf-core test profile | [[PH-06 Nextflow and nf-core]] | Implemented |
| 8.34 | Protection of information systems during audit testing | Yes | OQ tests run in the pipelines namespace with no impact on platform services | OPS-06 | [[OPS-06 OQ Test Scripts]] | Implemented |

---

## SoA Summary

| Category | Total Controls | Applicable | Implemented | Partial | Not Applicable |
|---|---|---|---|---|---|
| Organisational (5.x) | 37 | 30 | 26 | 4 | 7 |
| People (6.x) | 8 | 5 | 5 | 0 | 3 |
| Physical (7.x) | 14 | 10 | 8 | 2 | 4 |
| Technological (8.x) | 34 | 29 | 25 | 4 | 5 |
| **Total** | **93** | **74** | **64** | **10** | **19** |

**Controls fully implemented:** 64 of 74 applicable (86%)
**Controls partially implemented:** 10 of 74 applicable (14%)
**Primary partial control themes:** NetworkPolicy enforcement (pending Cilium), data classification policy, segregation of duties (solo project constraint), single CP node redundancy
