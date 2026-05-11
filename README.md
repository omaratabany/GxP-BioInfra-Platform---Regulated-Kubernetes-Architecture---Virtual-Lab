# GxP BioInfra Platform

A regulated Kubernetes platform for running nf-core genomics pipelines on a minimal bare-metal Talos cluster. The platform is built around auditability, policy enforcement, controlled change, and validation evidence aligned to EU GMP Annex 11, FDA 21 CFR Part 11, GAMP 5, and ISO/IEC 27001:2022.

Start here:

- [docs/Platform Primer.md](docs/Platform%20Primer.md) explains the platform in plain English.
- [plans/roadmap/PRJ-02 Phase Map and Schedule.md](plans/roadmap/PRJ-02%20Phase%20Map%20and%20Schedule.md) shows the current phase plan.
- [records/OPS-03 Implementation Log.md](records/OPS-03%20Implementation%20Log.md) records what has actually been built.

## Repository Structure

| Path | Purpose |
|---|---|
| `docs/` | Stable platform documentation and reference material |
| `plans/` | Phase plans, roadmap, project strategy, and portfolio material |
| `records/` | Build log, issue tracker, evidence registry, and checksums |
| `k8s/` | Deployable Kubernetes manifests and verification manifests |
| `.gitignore` | Excludes local OS files and operational snapshots |

## Documentation Areas

| Path | Contents |
|---|---|
| `docs/architecture/` | Infrastructure baseline, system architecture, scaling, observability, network topology, prerequisites |
| `docs/architecture/decisions/` | Architecture decision records and alternative configurations |
| `docs/architecture/modularity/` | Component interfaces, dependency maps, configuration standards, resource standards |
| `docs/operations/` | Build guide, commands, runbooks, DR, OQ scripts, baselines, onboarding, patch management |
| `docs/security/` | Security architecture, hardening, incident response, secrets, network security |
| `docs/regulatory/` | Compliance matrix, ISO 27001 mapping, risk register, supplier register, SoA, glossary |

## Planning Areas

| Path | Contents |
|---|---|
| `plans/phases/` | PH-00 through PH-07 implementation plans and phase status |
| `plans/roadmap/` | Phase map, schedule, and future roadmap |
| `plans/strategy/` | Strategic rationale, project significance, CKA alignment, public portfolio draft |

## Current Status

| Phase | Name | Status |
|---|---|---|
| PH-00 | Cluster Preparation | COMPLETE |
| PH-01 | Forgejo | IN PROGRESS |
| PH-02 | Authentik SSO | NOT STARTED |
| PH-03 | MinIO Object Storage | NOT STARTED |
| PH-04 | OPA Gatekeeper | NOT STARTED |
| PH-05 | Falco Runtime Security | NOT STARTED |
| PH-06 | Nextflow and nf-core | NOT STARTED |
| PH-07 | GxP Validation Documentation | NOT STARTED |

## Cluster at a Glance

```text
Talos v1.12.6 / Kubernetes v1.35.2
2 nodes: HP Omen control plane plus workloads, Beelink worker plus storage
Storage: local-hdd on Beelink /var/mnt/hdd
CNI: Flannel, with Cilium planned later
GitOps: ArgoCD
Ingress: Ingress-NGINX with NodePort access working
Known gap: MetalLB VIP 192.168.0.200 is not reachable from the Mac
```

## Build Dependency Chain

```text
PH-00 storage class and node labels
  PH-01 Forgejo change control
    PH-02 Authentik SSO
      PH-03 MinIO object storage
        PH-04 OPA Gatekeeper admission policy
          PH-05 Falco runtime security
            PH-06 Nextflow and nf-core pipeline run
              PH-07 validation documentation
```

## Working Rules

- Keep plans, docs, records, and manifests separated by directory.
- Keep operational snapshots outside Git.
- Commit only sealed secrets, never plaintext credentials.
- Keep deployment manifests under `k8s/apps/`.
- Keep temporary validation manifests under `k8s/tests/`.
