# PRJ-03 Project Significance

> Part of [[README]] | See also: [[PRJ-01 Strategic Rationale]], [[ADR-00 Decision Log]]

---

## The Problem This Platform Addresses

Drug development generates large volumes of genomic, proteomic, and clinical data. That data gets processed through computational pipelines -- tools that trim reads, align sequences, call variants, and generate reports. These pipelines influence which molecules get studied, which patient populations are targeted, and what ends up in a regulatory submission to health authorities.

When a pipeline produces a wrong result and nobody can prove what version of the software ran, on what data, with what parameters -- the regulatory submission is compromised. In the worst case, the wrong compound gets developed. In the more common case, years of work get invalidated because the audit trail is missing or incomplete.

The platforms running these pipelines at most pharma and biotech companies are one of three things: cloud-based but loosely governed with no audit trail attached to the run itself, on-premise HPC clusters running uncontainerised software with no reproducibility guarantees, or Kubernetes deployments built by teams who understand containers but not the regulatory requirements surrounding the data.

This project builds the third category correctly. A Kubernetes platform where the infrastructure enforces regulatory requirements at the admission layer, captures a tamper-evident runtime audit trail, runs pipelines through a reproducible executor, and produces documentation that a QA auditor can validate against.

---

## Who Needs This

The demographic is not a single job title -- it is an intersection of industries and organisational functions with a specific, well-funded gap.

### Industry

Any company that uses computational data as part of a regulated workflow:
- Pharmaceutical manufacturers involved in drug development, formulation analysis, or biomarker discovery
- Biotech companies running genomics platforms, developing diagnostics, or operating CDMO-scale biomanufacturing informatics
- Clinical research organisations running computational analyses on behalf of sponsors
- Medical device companies with software components that process patient-derived data

Geography matters. The EU GMP Annex 11 standard is the primary framework in Europe, and the highest concentration of pharma and biotech outside the US east coast biotech corridor sits in Basel and Zurich. These companies file with multiple health authorities simultaneously, which means they need platforms compliant with both Annex 11 and FDA 21 CFR Part 11 at the same time.

### Organisational Role

The people who operate or contract for this kind of platform:
- Platform engineering teams at mid-to-large pharma -- they know Kubernetes but not the regulatory layer
- QA and IT compliance teams who understand the regulations but cannot build the infrastructure
- Managed service providers and consultancies delivering regulated cloud infrastructure to pharma clients
- Smaller biotech companies that need a GxP-compliant compute environment but cannot afford to hire a full specialised team

The person who bridges platform engineering and regulatory compliance is rare. Large pharma companies typically keep one or two on staff as expensive contractors. Smaller biotechs either under-build (no compliance) or over-spend on commercial validated platforms that cost six figures a year and offer zero flexibility.

This project demonstrates that the capability can be built from open-source components with the right architecture -- which is the argument I am making to those hiring teams.

---

## Why Kubernetes Specifically

Kubernetes is the dominant container orchestration platform in enterprise pharma IT for practical reasons. Container-based pipelines are reproducible in a way HPC jobs are not -- the same image, the same configuration, produces the same result. The admission controller pattern (which Gatekeeper uses) gives a programmable policy layer that applies before any workload runs. The RBAC model maps cleanly to the role-based access concepts in Annex 11 and 21 CFR Part 11. Helm plus GitOps provides an auditable change management mechanism -- every change is a Git commit with a timestamp, author, and diff.

Commercial alternatives exist -- AWS Batch, Azure ML, Nextflow Tower -- but they introduce vendor lock-in and make it harder to prove the audit trail under regulatory scrutiny. An open-source stack where every component is auditable and configurable is more defensible in a regulatory submission context.

---

## Why Nextflow and nf-core

Nextflow is the dominant workflow engine in bioinformatics. nf-core is the community-maintained collection of validated pipelines built on it, designed specifically with reproducibility and traceability in mind.

nf-core/rnaseq is my reference choice because it is widely used in drug discovery and biomarker research, it has a test profile that runs on minimal data, it is a real pipeline that runs in pharma production environments, and its Kubernetes executor configuration maps directly to the RBAC and resource management requirements of the platform. Running it successfully proves something meaningful.

---

## What This Is Not

This is not a clinical system that handles patient data directly. No patient data runs through this platform. It is not a replacement for commercial validated platforms like Veeva or Medidata for document management. It is not a production-grade HA deployment -- I have a 2-node homelab cluster with one control plane node.

It is a demonstration that a GxP-compliant bioinformatics platform can be built from open-source components with the right architecture. The regulatory compliance layer is real. The audit trail is real. The validation documentation format is correct. The hardware is a homelab.
