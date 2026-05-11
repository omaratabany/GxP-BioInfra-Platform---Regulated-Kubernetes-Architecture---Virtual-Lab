# PRJ-01 Strategic Rationale

> Part of [[README]] | See also: [[INF-02 Architecture and Components]], [[PH-07 GxP Validation Documentation]], [[PRJ-03 Project Significance]]

---

## The Problem I'm Solving

Pharma and biotech companies in Basel and Zurich have a specific hiring gap: DevOps engineers who understand GxP compliance. Their platform teams know Kubernetes but not regulatory requirements. Their QA teams know the regulations but cannot build the infrastructure. The person who bridges both is rare and expensive to develop internally.

A generic DevOps profile with K8s certs competes with hundreds of applicants. My profile includes "designed and implemented a GxP-compliant Kubernetes platform with IQ/OQ/PQ documentation per EU GMP Annex 11" -- which gets forwarded immediately because it solves a real open problem these companies have budget for.

No medical degree needed. This is infrastructure work with regulatory context layered on top. That is exactly what I do.

---

## Regulatory Framework I'm Working To

| Standard | What It Is | Why It's Relevant |
|---|---|---|
| EU GMP Annex 11 | EU regulation for computerised systems in pharma | Applies to any system that processes or stores data used in drug development |
| FDA 21 CFR Part 11 | US equivalent -- electronic records and signatures | Required for US market pharma companies |
| GAMP 5 | Industry validation methodology | Framework used to categorize and validate software systems |

My platform satisfies the core Annex 11 requirements:

- **Audit trail** -- Falco + Loki, see [[PH-05 Falco Runtime Security]]
- **Access control** -- Authentik + RBAC, see [[PH-02 Authentik SSO]]
- **Change management** -- Forgejo PRs + ArgoCD GitOps, see [[PH-01 Forgejo]]
- **Data integrity** -- MinIO + image digest pinning, see [[PH-03 MinIO Object Storage]] and [[PH-04 OPA Gatekeeper]]
- **System validation evidence** -- IQ/OQ/PQ, see [[PH-07 GxP Validation Documentation]]

---

## Technical Skills I'm Building

| Skill | Phase | Market Value |
|---|---|---|
| Policy-as-code (Gatekeeper) | PH-04 | Used in every enterprise K8s environment |
| Runtime security (Falco) | PH-05 | Standard in regulated industries |
| GitOps change control | PH-01 | Required for any auditability claim |
| Bioinformatics tooling (Nextflow, nf-core) | PH-06 | Niche but high value in pharma |
| Validation documentation (IQ/OQ/PQ) | PH-07 | Almost no DevOps candidates have this |

---

## CKA Study Alignment

| Week | CKA Domain | Project Phase |
|---|---|---|
| 1-2 | Cluster architecture, etcd | [[PH-00 Cluster Preparation]] |
| 3 | Workloads | [[PH-01 Forgejo]] |
| 4 | Scheduling, affinity | [[PH-02 Authentik SSO]] |
| 5 | Networking | [[PH-03 MinIO Object Storage]] |
| 6 | Storage | [[PH-04 OPA Gatekeeper]] |
| 7 | RBAC, troubleshooting | [[PH-05 Falco Runtime Security]] + [[PH-06 Nextflow and nf-core]] |
| 8 | Exam sim | [[PH-06 Nextflow and nf-core]] pipeline run |
| Post-CKA | -- | [[PH-07 GxP Validation Documentation]] + public README |

Full phase schedule: [[PRJ-02 Phase Map and Schedule]]
