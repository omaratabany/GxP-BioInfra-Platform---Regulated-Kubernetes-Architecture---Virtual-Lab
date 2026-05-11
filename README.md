# GxP BioInfra Platform

A GxP-compliant Kubernetes platform for running nf-core genomics pipelines on a bare-metal homelab cluster. Full audit trail, policy enforcement, and validation documentation aligned to EU GMP Annex 11, FDA 21 CFR Part 11, and ISO/IEC 27001:2022. Built in parallel with CKA certification study -- each phase maps to the CKA domain being studied that week.

> New to this project? Start with [[Platform Primer]] -- plain English explanation of every tool, every page, and what this builds.

---

## Navigation

### Infrastructure and Architecture
- [[INF-01 Infrastructure Baseline]] -- hardware, cluster state, deployed stack, network topology
- [[INF-02 Architecture and Components]] -- final state diagram, component registry, data flow, namespace plan, node placement rules
- [[INF-03 Infrastructure Analysis]] -- utilization targets, hardware limits, misjudgments identified, what ideal looks like
- [[INF-04 System Tier Comparison]] -- tables comparing minimal / current / ideal across all dimensions
- [[INF-05 Hardware Scaling Guide]] -- how this platform adapts to any hardware tier, dynamic configuration, migration paths
- [[INF-06 Observability and Alerting]] -- Prometheus alert rules, Grafana dashboard specifications, alert routing, GxP-specific failure alerts
- [[INF-07 Network Topology]] -- full network topology, IP allocation, DNS configuration, Ingress routing table, port reference, Cert-Manager and MetalLB config

### Project Strategy
- [[PRJ-01 Strategic Rationale]] -- the hiring problem this solves, regulatory framework, CKA alignment
- [[PRJ-02 Phase Map and Schedule]] -- week-by-week schedule, phase dependencies, status overview
- [[PRJ-03 Project Significance]] -- why this was built, who needs it, what industry problem it addresses
- [[PRJ-04 CKA Study Alignment]] -- per-domain study guide tied to each project phase, key kubectl commands, practice exercises
- [[PRJ-05 Public Portfolio README]] -- draft of the GitHub-facing public README, ready to publish after Phase 7
- [[PRJ-06 Future Roadmap]] -- near, medium, and long-term improvements with risk register impact

### Architecture Decisions
- [[ADR-00 Decision Log]] -- every architectural and tooling decision with full reasoning and alternatives
- [[ADR-01 Alternative Configurations]] -- all secondary options for every component in the stack

### Security
- [[SEC-01 Security Architecture]] -- threat model, attack vectors, defense layers, current security posture
- [[SEC-02 Component Hardening Guide]] -- per-component hardening steps and verification commands
- [[SEC-03 Incident Response Playbook]] -- detection, containment, and recovery for each incident scenario
- [[SEC-04 Secrets and Key Management]] -- secret lifecycle, creation, rotation, retirement, audit procedure
- [[SEC-05 Network Security Policy]] -- zone model, traffic matrix, Flannel limitations, Cilium target state with NetworkPolicy YAML

### Regulatory and Compliance
- [[REG-01 Compliance Matrix]] -- master cross-reference: every platform capability mapped to Annex 11, 21 CFR Part 11, GAMP 5, and ISO 27001
- [[REG-02 ISO 27001 Alignment]] -- detailed ISO/IEC 27001:2022 Annex A control mapping to specific platform components
- [[REG-03 Risk Register]] -- formal risk register with likelihood/impact matrix, mitigations, residual ratings
- [[REG-04 Supplier Assessment Register]] -- 12 suppliers assessed against Annex 11 Clause 5 and ISO 27001 5.19
- [[REG-05 Statement of Applicability]] -- mandatory ISO 27001 SoA: all 93 controls assessed, 74 applicable, 64 implemented
- [[REG-06 Glossary and Terminology]] -- GxP, bioinformatics, and Kubernetes terms and acronyms for cross-functional readers

### Modularity
- [[MOD-01 Component Interface Specifications]] -- interface contracts for all 8 inter-component connections (S3, OIDC, Loki, Prometheus, Git, admission webhook, K8s API, Falco events)
- [[MOD-02 Modularity and Dependency Map]] -- dependency graph, minimum viable subsets, upgrade and swap procedures, version matrix
- [[MOD-03 Configuration Management Standard]] -- repository structure, Helm values standard, ArgoCD manifest standard, drift detection
- [[MOD-04 Kubernetes Resource Standards]] -- naming conventions, mandatory labels, resource tier definitions, anti-patterns

### Build and Operations
- [[OPS-01 Build Instructions]] -- step-by-step implementation guide, failure playbooks per phase
- [[OPS-02 Reference Commands]] -- all kubectl, talosctl, mc, ArgoCD, Helm, and Nextflow commands
- [[OPS-03 Implementation Log]] -- live build log, issue tracker, phase completion checklist, component version registry
- [[OPS-04 Operational Runbook]] -- daily checks, weekly tasks, monthly tasks, 7 named procedures including PROC-07 Homepage/Dashboard removal
- [[OPS-05 Disaster Recovery Plan]] -- step-by-step recovery from 5 scenarios: etcd corruption, HDD failure, node failure, key loss, ArgoCD state loss
- [[OPS-06 OQ Test Scripts]] -- executable scripts for OQ-01 through OQ-10 with expected output and results sheet
- [[OPS-07 Performance Baselines]] -- idle baseline measurement tables, pipeline run metric capture, PQ acceptance criteria
- [[OPS-08 Developer and Operator Onboarding]] -- Mac tool install, kubeconfig setup, local DNS, CA trust, MinIO client, Forgejo SSH, full stack verification
- [[OPS-09 Patch Management]] -- advisory sources, response times per severity, patch procedure, emergency procedure, Talos upgrade procedure

### Implementation Phases
- [[PH-00 Cluster Preparation]] -- IN PROGRESS
- [[PH-01 Forgejo]] -- NOT STARTED
- [[PH-02 Authentik SSO]] -- NOT STARTED
- [[PH-03 MinIO Object Storage]] -- NOT STARTED (includes Prometheus TSDB migration)
- [[PH-04 OPA Gatekeeper]] -- NOT STARTED (includes full Rego ConstraintTemplate definitions)
- [[PH-05 Falco Runtime Security]] -- NOT STARTED
- [[PH-06 Nextflow and nf-core]] -- NOT STARTED
- [[PH-07 GxP Validation Documentation]] -- NOT STARTED

### Reference
- [[Platform Primer]] -- plain English explanation of every tool and every file in this vault

---

## Current Status

| Phase | Name | Status |
|---|---|---|
| 0 | Cluster Preparation | IN PROGRESS |
| 1 | Forgejo | NOT STARTED |
| 2 | Authentik SSO | NOT STARTED |
| 3 | MinIO Object Storage | NOT STARTED |
| 4 | OPA Gatekeeper | NOT STARTED |
| 5 | Falco Runtime Security | NOT STARTED |
| 6 | Nextflow and nf-core/rnaseq | NOT STARTED |
| 7 | GxP Validation Documentation | NOT STARTED |

---

## Cluster at a Glance

```
Talos v1.12.6 / Kubernetes v1.35.2
2 nodes: HP Omen (CP + workloads) + Beelink (storage)
Total usable: ~12 cores / ~20GB RAM / 320GB HDD
CNI: Flannel (Cilium post-CKA)
GitOps: ArgoCD watching GitHub (Forgejo in Phase 1)
External access: Cloudflare Tunnel (CGNAT -- no inbound ports)
MetalLB VIP: 192.168.0.200
```

---

## Dependency Chain

```
PH-00 (storage class, node labels)
  PH-01 (Forgejo needs PVC on local-hdd)
    PH-02 (Authentik needs Forgejo for OIDC config)
      PH-03 (MinIO + Prometheus TSDB migration)
        PH-04 (Gatekeeper validates all subsequent workloads)
          PH-05 (Falco DaemonSet, routes to Loki)
            PH-06 (Nextflow needs MinIO + RBAC + Falco active)
              PH-07 (docs reference evidence from all phases)
```

PH-07 drafting starts in parallel from PH-04.

---

## Regulatory Standards

| Standard | Body | Scope |
|---|---|---|
| EU GMP Annex 11 | EMA | Computerised systems in pharma |
| FDA 21 CFR Part 11 | FDA | Electronic records and signatures |
| GAMP 5 | ISPE | Software validation methodology |
| ISO/IEC 27001:2022 | ISO/IEC | Information security management |

---

## Risk Summary (Current Residual)

| Risk | Residual | Primary Mitigation |
|---|---|---|
| R-01 Beelink HDD failure | Medium | Weekly MinIO mirror to external target |
| R-02 Single control plane | Medium | Weekly etcd snapshots + DR plan |
| R-03 Supply chain image attack | Low | Gatekeeper digest pinning + Falco |
| R-04 No NetworkPolicy enforcement | Medium | Eliminated post-Cilium migration |
| R-05 Audit trail on-cluster only | Medium | GOVERNANCE bucket lock + mirror |
| R-06 Sealed Secrets key loss | Low | Encrypted key backup outside cluster |
| R-07 Pipeline resource exhaustion | Low | ResourceQuota + Gatekeeper limits |
| R-08 Cloudflare Tunnel dependency | Low | Accepted -- LAN access unaffected |

Full detail: [[REG-03 Risk Register]]

---

## File Index

| Prefix | Count | Purpose |
|---|---|---|
| README, Platform Primer | 2 | Hub and plain English guide |
| PRJ- | 6 | Project strategy, CKA study, roadmap, public README |
| INF- | 7 | Infrastructure, architecture, observability, network topology |
| ADR- | 2 | Architecture decision records |
| SEC- | 5 | Security model, hardening, incident response, secrets, network |
| REG- | 6 | Compliance, ISO 27001, risk, suppliers, SoA, glossary |
| MOD- | 4 | Modularity, dependencies, config management, K8s standards |
| OPS- | 9 | Build, commands, log, runbook, DR, OQ scripts, baselines, onboarding, patch management |
| PH- | 8 | Implementation phases |
| **Total** | **49** | |

---

*Phase 0 in progress -- May 2026*
*Next: local-path-provisioner deployed, StorageClass confirmed, etcd snapshot, move to Phase 1*
