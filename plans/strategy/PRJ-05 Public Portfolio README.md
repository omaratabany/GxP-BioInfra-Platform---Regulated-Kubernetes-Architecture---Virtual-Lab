# PRJ-05 Public Portfolio README

> Part of [[README]] | See also: [[PRJ-01 Strategic Rationale]], [[PRJ-03 Project Significance]]

Draft of the GitHub-facing public README. This is what a recruiter, hiring manager, or pharma platform engineer sees when they open the repo. Written for an external audience -- no Obsidian links, no internal references, no assumption of prior context.

Publish this file to the GitHub repo root as README.md when PH-07 is complete and the pipeline run evidence exists.

---

## Draft Content

---

# GxP-Compliant Kubernetes Bioinformatics Platform

A production-grade Kubernetes platform for running nf-core genomics pipelines with full GxP compliance -- audit trail, policy enforcement, and validation documentation aligned to EU GMP Annex 11, FDA 21 CFR Part 11, and ISO/IEC 27001:2022.

Built on a bare-metal homelab cluster running Talos OS. Every component is deployed via GitOps (ArgoCD watching this repo) with no manual cluster changes.

---

## What This Demonstrates

| Capability | Implementation |
|---|---|
| GxP validation | IQ/OQ/PQ documentation per GAMP 5 Category 4/5 methodology |
| Audit trail | Falco eBPF kernel-level event capture -> Loki -> tamper-evident storage |
| Policy enforcement | OPA Gatekeeper -- 5 constraints covering image pinning, resource limits, privilege escalation, and registry control |
| Identity and access | Authentik OIDC -- single IdP for Forgejo, ArgoCD, and Grafana. No local passwords. |
| Change control | All infrastructure changes via Forgejo PR -- ArgoCD enforces GitOps, no manual cluster changes survive |
| Secret management | Sealed Secrets -- all credentials encrypted at rest in Git |
| Object storage | MinIO S3-compatible -- pipeline I/O, Loki log persistence, MinIO bucket lock for audit trail retention |
| Bioinformatics pipeline | nf-core/rnaseq running end-to-end with Nextflow K8s executor |
| Observability | Prometheus + Grafana + Loki -- custom alert rules for GxP-specific failure modes |
| ISO 27001 alignment | 74 applicable controls assessed, 64 fully implemented, formal Statement of Applicability produced |

---

## Architecture

```
MacBook Air M5 (management)
  kubectl / talosctl / argocd CLI
        |
        | LAN 192.168.0.0/24
        |
  +-----+----------------------------------+
  |         Talos K8s Cluster              |
  |                                        |
  |  HP Omen (talos-asj-72z)              |
  |  192.168.0.134 -- Control Plane       |
  |  8 cores / 16GB RAM                   |
  |                                        |
  |    ArgoCD (GitOps controller)          |
  |    Authentik (OIDC / SSO)             |
  |    OPA Gatekeeper (policy admission)  |
  |    Falco DaemonSet (runtime audit)    |
  |    Nextflow + nf-core pods            |
  |                                        |
  |  Beelink Mini PC (talos-v3h-4m1)     |
  |  192.168.0.202 -- Storage Node       |
  |  4 cores / 7.5GB RAM / 320GB HDD     |
  |                                        |
  |    MinIO (S3 -- pipeline data)        |
  |    Loki data (GOVERNANCE lock)        |
  |    Prometheus TSDB                    |
  |    Forgejo repositories               |
  |                                        |
  |  Shared: Ingress-NGINX, Cert-Manager  |
  |  MetalLB L2 (192.168.0.200 VIP)     |
  +----------------------------------------+
        |
        | Cloudflare Tunnel (outbound only -- CGNAT)
        |
  External access at *.homelab
```

---

## GxP Compliance Coverage

### EU GMP Annex 11 (2011) -- Key Clauses

| Clause | Requirement | Implementation |
|---|---|---|
| 1 | Risk management | Formal risk register with 8 identified risks, likelihood/impact/residual ratings |
| 4.2 | Change control | Forgejo PR workflow -- all changes tracked, ArgoCD enforces no out-of-band changes |
| 4.4 | Resource management | OPA Gatekeeper denies any pod without CPU/memory limits |
| 7.1 | Data integrity | MinIO object versioning on pipeline-output, Retain PVC reclaim policy |
| 9 | Audit trail | Falco eBPF captures at kernel level -- cannot be disabled from within a container. Events stored in Loki on MinIO with 30-day GOVERNANCE lock. |
| 10 | Change management | Gatekeeper requires image digests (no mutable tags), approved registry list enforced |
| 12.1 | Access control | Authentik OIDC -- no local passwords, group-based RBAC synced to K8s |
| 17 | Archival and backup | Weekly etcd snapshots, weekly MinIO bucket mirroring |

### FDA 21 CFR Part 11 -- Key Sections

| Section | Requirement | Implementation |
|---|---|---|
| 11.10(c) | Record protection | MinIO bucket lock (GOVERNANCE mode) on audit log storage |
| 11.10(d) | Access limitation | Authentik SSO + K8s RBAC -- no shared accounts |
| 11.10(e) | Audit trails | Kernel-level Falco capture -- tamper-evident by design |
| 11.10(j) | Password management | 32-char random service credentials, 90-day rotation, Sealed Secrets in Git |

---

## Validation Evidence

| Document | Contents |
|---|---|
| IQ -- Installation Qualification | Component versions, config checksums, installation verification, network topology |
| OQ -- Operational Qualification | 10 scripted test cases with pass/fail results, timestamps, operator records |
| PQ -- Performance Qualification | Two independent nf-core/rnaseq runs, reproducibility verified by MultiQC MD5 checksum |
| Change Control SOP | Forgejo PR procedure -- in active use from Phase 1 |
| Audit Trail SOP | How Falco + Loki satisfy Annex 11 Clause 9 |
| Access Control SOP | Authentik group management, onboarding/offboarding procedure |
| Backup and Recovery SOP | etcd snapshot procedure, MinIO mirror SOP, recovery tested |

All validation documents are stored in `validation-docs/` in this repository.

---

## Platform Stack

| Component | Version | Purpose |
|---|---|---|
| Talos OS | v1.12.6 | Immutable Kubernetes OS -- no SSH, no shell, minimal attack surface |
| Kubernetes | v1.35.2 | Orchestration |
| ArgoCD | TBD | GitOps -- all cluster changes flow through this |
| Forgejo | TBD | Self-hosted Git -- primary SCM, PR-based change control |
| Authentik | TBD | OIDC / SSO -- single IdP, no local passwords |
| MinIO | TBD | S3-compatible object storage -- pipeline data and audit log persistence |
| OPA Gatekeeper | TBD | Policy-as-code admission control |
| Falco | TBD | Runtime security + GxP audit trail (eBPF, kernel-level) |
| Nextflow | TBD | Pipeline orchestration -- Kubernetes executor |
| nf-core/rnaseq | TBD | RNA-seq pipeline -- validated per GAMP 5 Category 5 |
| Prometheus + Grafana | TBD | Metrics, alerting, PQ evidence capture |
| Loki + Promtail | TBD | Log aggregation -- Falco audit events via Falcosidekick |
| Cert-Manager | TBD | TLS certificate provisioning |
| Sealed Secrets | TBD | Encrypted secrets for safe Git storage |
| MetalLB | TBD | Bare-metal load balancer |
| local-path-provisioner | TBD | Dynamic PV provisioning from HDD |

---

## Repository Structure

```
apps/
  <app-name>/
    namespace.yaml          -- Namespace definition
    argocd-application.yaml -- ArgoCD Application manifest
    helm-release.yaml       -- Chart version and values
    sealed-secret.yaml      -- Encrypted credentials
    ingress.yaml            -- Ingress routing
    pvc.yaml                -- Persistent storage definitions
    policies/               -- Gatekeeper Rego, MinIO IAM policies
patches/
  beelink-hdd.yaml          -- Talos machineconfig patches
validation-docs/
  IQ-Installation-Qualification.md
  OQ-Operational-Qualification.md
  PQ-Performance-Qualification.md
  SOPs/
    change-control.md
    audit-trail.md
    backup-recovery.md
    access-control.md
README.md                   -- This file
```

---

## Regulatory Alignment

| Standard | Scope | Coverage |
|---|---|---|
| EU GMP Annex 11 (EMA, 2011) | Computerised systems in pharma | Substantial -- all applicable clauses |
| FDA 21 CFR Part 11 | Electronic records and signatures | Substantial -- all applicable sections |
| GAMP 5 (ISPE, 2022) | Software validation methodology | Full -- Category 4 and 5 items validated |
| ISO/IEC 27001:2022 | Information security | 86% of applicable controls implemented, formal SoA produced |

---

## Why This Project Exists

Pharma and biotech companies in Basel and Zurich (Roche, Novartis, Lonza, and CROs) have a specific engineering gap: they have platform teams that know Kubernetes and teams that know GxP, but almost no one who knows both. This project is the practical demonstration that someone can build the infrastructure AND produce the regulatory documentation it requires.

This is not a theoretical exercise. Every component is running on real hardware, every validation test has a real pass/fail result, and every SOP is actively used during the build. The validation documents are not templates -- they reference specific component versions, real command outputs, and actual configuration checksums.

---

## Build Status

- [x] Phase 0: Cluster preparation -- storage class, node labels, etcd snapshots
- [x] Phase 1: Forgejo -- self-hosted Git, primary SCM
- [x] Phase 2: Authentik -- OIDC SSO, no local passwords
- [x] Phase 3: MinIO -- S3 storage, Loki migration
- [x] Phase 4: OPA Gatekeeper -- policy enforcement active
- [x] Phase 5: Falco -- runtime audit trail live
- [x] Phase 6: Nextflow -- nf-core/rnaseq pipeline run complete
- [x] Phase 7: GxP validation docs -- IQ/OQ/PQ complete

---

## Contact

Built by Omar Atabany. Platform Engineering and DevOps targeting pharmaceutical and biomedical companies.

GitHub: https://github.com/omaratabany
LinkedIn: [add when publishing]

---

*End of draft. Update build status checkboxes and fill in component versions from OPS-03 Implementation Log before publishing.*
