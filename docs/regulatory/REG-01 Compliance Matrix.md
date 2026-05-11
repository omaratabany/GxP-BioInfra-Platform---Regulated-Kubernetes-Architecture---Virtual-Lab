# REG-01 Compliance Matrix

> Part of [[README]] | See also: [[REG-02 ISO 27001 Alignment]], [[REG-03 Risk Register]], [[PH-07 GxP Validation Documentation]]

Master cross-reference table mapping every platform capability to the specific regulatory control it satisfies. This is the document a QA auditor, ISO auditor, or compliance reviewer reads first to understand what the platform claims to cover and where the evidence lives.

---

## How to Read This Matrix

Each row is a regulatory requirement. The columns show which platform component implements it, where the technical evidence lives, and which test case in the OQ validates it.

Legend:
- **Full** -- requirement is fully satisfied by current implementation
- **Partial** -- requirement is partially satisfied; gap is noted
- **Planned** -- planned post-Cilium or post-CKA
- **Out of scope** -- explicitly not in scope for this platform tier

---

## EU GMP Annex 11 (2011) -- Computerised Systems

| Clause | Requirement | Implementation | Evidence Location | OQ Test | Status |
|---|---|---|---|---|---|
| 1 | Risk management | [[REG-03 Risk Register]] with likelihood/impact/mitigation | REG-03 | N/A | Full |
| 4.1 | Validation documentation (specifications) | IQ/OQ/PQ in Forgejo validation-docs/ | PH-07 GxP Validation Documentation | N/A | Full |
| 4.2 | Change control | Forgejo PR workflow + ArgoCD GitOps | Git history, ArgoCD app history | OQ-06 | Full |
| 4.3 | System documentation | This vault + README | All INF- files | N/A | Full |
| 4.4 | Resource management | OPA Gatekeeper require-resource-limits | Gatekeeper constraint violations report | OQ-01 | Full |
| 4.6 | Operator qualification | IQ operator field; Access Control SOP | IQ document, PH-07 SOPs | N/A | Full |
| 5 | Computer system supplier assessment | Helm chart provenance; image digest pinning | Gatekeeper approved-registries + image-digest | N/A | Partial -- no Cosign signing yet |
| 7.1 | Data integrity | MinIO object versioning; etcd snapshots | mc version info homelab/pipeline-output | OQ-09 | Full |
| 9 | Audit trail | Falco eBPF + Loki + Falcosidekick | Grafana -> Loki: {job="falco"} | OQ-03, OQ-10 | Full |
| 10 | Change management | Forgejo PR + ArgoCD; Gatekeeper image-digest + approved-registries | Git PRs, ArgoCD history | OQ-02, OQ-06 | Full |
| 11 | Electronic signatures | Not applicable (no clinical data) | N/A | N/A | Out of scope |
| 12.1 | Access control | Authentik OIDC + K8s RBAC | Authentik audit log, RBAC bindings | OQ-04, OQ-05 | Full |
| 12.3 | System control | OPA Gatekeeper disallow-privileged | Gatekeeper constraint | N/A | Full |
| 12.4 | Incident management | [[SEC-03 Incident Response Playbook]]; Falco CRITICAL alerts | Incident log in OPS-03 | OQ-10 | Full |
| 13 | Periodic review | Hardening checklist in SEC-02; quarterly secret audit | SEC-02, SEC-04 | N/A | Partial -- no automated scheduling yet |
| 17 | Archival and backup | etcd snapshot SOP + MinIO mirror SOP | OPS-05, PH-07 Backup SOP | OQ-09 | Full |

---

## FDA 21 CFR Part 11 -- Electronic Records and Signatures

| Section | Requirement | Implementation | Evidence | Status |
|---|---|---|---|---|
| 11.10(a) | Validation that systems produce accurate results | IQ/OQ/PQ; PQ reproducibility test | PH-07 GxP Validation Documentation | Full |
| 11.10(b) | Ability to generate accurate copies of records | MinIO object versioning; mc cp for record retrieval | mc version info output | Full |
| 11.10(c) | Record protection (computer-generated) | MinIO bucket lock (GOVERNANCE) on loki-chunks; Retain reclaimPolicy | SEC-02 MinIO hardening | Full |
| 11.10(d) | Limiting system access to authorised individuals | Authentik OIDC; K8s RBAC; no local passwords post-PH-02 | Authentik group bindings, RBAC | Full |
| 11.10(e) | Secure computer-generated audit trails | Falco eBPF (kernel-level, tamper-evident); Loki on MinIO | Grafana Loki queries | Full |
| 11.10(f) | Use of operational system checks | OPA Gatekeeper; ResourceQuota on pipelines namespace | Gatekeeper violations report | Full |
| 11.10(g) | Authority checks (electronic signatures) | Not applicable -- no electronic signatures in current scope | N/A | Out of scope |
| 11.10(h) | Device checks | Node labels + nodeSelector enforce workload placement | kubectl get nodes --show-labels | Partial |
| 11.10(i) | Training | Operator training: this vault + Platform Primer | Platform Primer | Partial -- no formal training records |
| 11.10(j) | Policies for password management | [[SEC-04 Secrets and Key Management]] rotation schedule | SEC-04 | Full |
| 11.30 | Controls for open systems | Cloudflare Tunnel; TLS everywhere; Sealed Secrets | SEC-01, SEC-05 | Full |

---

## GAMP 5 (ISPE, 2022) -- Software Categorisation and Validation

| Category | Definition | Platform Mapping | Validation Approach |
|---|---|---|---|
| Category 1 | Infrastructure software | Talos OS, Linux kernel, Kubernetes | Vendor qualification only |
| Category 2 | Non-configured software | Does not apply to this platform | N/A |
| Category 3 | Non-configured software (COTS) | Prometheus, Grafana, Loki, Flannel, Cert-Manager | Installation testing (IQ) only |
| Category 4 | Configured software | Helm chart configurations, OPA Rego policies, Falco rules, Nextflow config, ArgoCD ApplicationSets | IQ + OQ -- this is the primary validation target |
| Category 5 | Custom software | nf-core/rnaseq pipeline (community-developed but version-controlled and validated by us) | Full IQ + OQ + PQ |

**GAMP 5 validation scope for this platform:**
- Category 4 items (configurations) undergo IQ and OQ validation
- Category 5 items (nf-core/rnaseq) undergo full IQ + OQ + PQ
- Category 1 and 3 items undergo vendor qualification only -- no IQ/OQ from our side

---

## ISO 27001:2022 -- Information Security Controls (Summary)

Full mapping is in [[REG-02 ISO 27001 Alignment]].

| Control Domain | Key Controls | Implementation | Status |
|---|---|---|---|
| 5.1 Policies | Information security policy | This vault (SEC-01 through SEC-05) | Full |
| 5.15 Access control | Access rights management | Authentik + K8s RBAC | Full |
| 5.17 Authentication | Authentication rules | OIDC via Authentik, MFA for admins | Full |
| 6.8 Information security events | Reporting | Falco alerts + incident response playbook | Full |
| 7.14 Secure disposal | Equipment disposal | Talos disk wipe on node decommission | Full |
| 8.4 Information in use | Handling of assets | MinIO IAM per bucket, RBAC per namespace | Full |
| 8.7 Protection against malware | Malware protection | Gatekeeper image policy; Falco runtime detection | Full |
| 8.8 Vulnerability management | Technical vulnerabilities | Gatekeeper image-digest pinning; approved-registries | Partial -- no CVE scanning yet |
| 8.12 Data leakage prevention | DLP controls | Falco outbound connection rule; NetworkPolicy (planned) | Partial |
| 8.15 Logging | System logging | Falco + Loki + Promtail | Full |
| 8.16 Monitoring | Network monitoring | Prometheus + Grafana; Hubble post-Cilium | Partial |
| 8.29 Secure coding | Development lifecycle | GitOps PR workflow; Gatekeeper admission | Full |
| 8.32 Change management | Managing changes | Forgejo PR + ArgoCD; IQ/OQ for each change | Full |
| 8.34 Protection during testing | Test separation | pipelines namespace isolated; test profile uses synthetic data | Full |

---

## Compliance Gap Summary

| Gap | Standard | Severity | Planned Resolution | Timeline |
|---|---|---|---|---|
| NetworkPolicy not enforced | Annex 11 12.3, ISO 27001 8.12 | Medium | Cilium migration post-CKA | Post-CKA |
| No image signing (Cosign) | Annex 11 5, ISO 27001 8.7 | Low | Harbor registry + Cosign in ideal infra | Future |
| No CVE scanning pipeline | ISO 27001 8.8 | Low | Trivy scan in ArgoCD pipeline | Future |
| No formal training records | 21 CFR Part 11 11.10(i) | Low | Add training log to OPS-03 | PH-07 |
| No external SIEM | Annex 11 9, ISO 27001 8.15 | Medium | Loki event export to external target | Future |
| Electronic signatures | 21 CFR Part 11 11.30 | N/A | Explicitly out of scope for this tier | Out of scope |
