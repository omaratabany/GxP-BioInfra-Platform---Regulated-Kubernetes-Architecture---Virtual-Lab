# REG-04 Supplier Assessment Register

> Part of [[README]] | See also: [[REG-01 Compliance Matrix]], [[REG-02 ISO 27001 Alignment]], [[SEC-01 Security Architecture]]

Formal register of all third-party software suppliers used by this platform. Required by EU GMP Annex 11 Clause 5 (Supplier and Service Providers) and ISO/IEC 27001:2022 control 5.19 (Information security in supplier relationships).

---

## Scope and Methodology

This register covers all external software components that:
- Execute on the cluster nodes or in pipeline containers
- Handle, store, or route data classified as Sensitive or above
- Form part of the change control or audit trail chain

**Assessment approach:** Each supplier is assessed against four criteria:
- **Integrity verification** -- can we confirm what we receive is what the supplier intended?
- **Change visibility** -- do we know when the supplier changes the software?
- **Incident response** -- does the supplier publish security advisories?
- **Contractual standing** -- what is the licensing and support model?

**Overall supplier risk is rated:** Low / Medium / High
**Assessment frequency:** Reviewed as part of MO-04 monthly dependency review in [[OPS-04 Operational Runbook]].

---

## SUP-01 -- Sidero Labs (Talos OS)

| Field | Detail |
|---|---|
| **Software** | Talos OS v1.12.6 |
| **Role** | Operating system for all cluster nodes -- foundational |
| **Criticality** | Critical -- cluster cannot operate without this |
| **Source** | https://github.com/siderolabs/talos |
| **License** | Mozilla Public License 2.0 |
| **Support model** | Open source, enterprise support available |
| **Integrity verification** | GitHub releases include SHA512 checksums and cosign signatures. Talos installer images are pulled by digest via `talosctl upgrade --image ghcr.io/siderolabs/installer:<version>@sha256:<digest>`. |
| **Change visibility** | GitHub releases page; CHANGELOG.md maintained per release. Security advisories via GitHub Security tab. |
| **Security advisories** | Published at https://github.com/siderolabs/talos/security/advisories |
| **Version pinning** | Talos version pinned in cluster config; upgrades are manual via `talosctl upgrade` |
| **Last assessed** | May 2026 |
| **Risk rating** | Low -- open source, high transparency, cosign signatures available |
| **Notes** | Talos's immutable architecture means a compromised node requires a full reinstall, not a patch. The attack surface is intentionally minimal. |

---

## SUP-02 -- Rancher (local-path-provisioner)

| Field | Detail |
|---|---|
| **Software** | rancher.io/local-path-provisioner |
| **Role** | Dynamic PV provisioner from Beelink HDD |
| **Criticality** | High -- all persistent storage depends on this |
| **Source** | https://github.com/rancher/local-path-provisioner |
| **License** | Apache 2.0 |
| **Integrity verification** | Container image pulled by digest. Helm chart version pinned in `apps/local-path-provisioner/helm-release.yaml`. |
| **Change visibility** | GitHub releases. Helm chart changelog via `helm show changelog`. |
| **Security advisories** | GitHub Security tab |
| **Risk rating** | Low |

---

## SUP-03 -- ArgoCD (Argo Project / CNCF)

| Field | Detail |
|---|---|
| **Software** | ArgoCD (version TBD at install) |
| **Role** | GitOps controller -- all cluster changes flow through this |
| **Criticality** | Critical -- change control SOP depends on ArgoCD |
| **Source** | https://github.com/argoproj/argo-cd |
| **License** | Apache 2.0 |
| **Integrity verification** | Helm chart version pinned. Image digest pinned via Gatekeeper require-image-digest after PH-04. ArgoCD publishes signed container images via Cosign. |
| **Change visibility** | GitHub releases; CHANGELOG maintained. Security advisories active. |
| **Security advisories** | https://github.com/argoproj/argo-cd/security/advisories |
| **Risk rating** | Low -- CNCF graduated project, widely deployed, active security programme |
| **Notes** | ArgoCD CVEs have a history of severity -- patch within 7 days of any HIGH or CRITICAL advisory. |

---

## SUP-04 -- Cert-Manager (cert-manager.io / CNCF)

| Field | Detail |
|---|---|
| **Software** | cert-manager |
| **Role** | TLS certificate provisioning for all platform services |
| **Criticality** | High -- all HTTPS depends on this |
| **Source** | https://github.com/cert-manager/cert-manager |
| **License** | Apache 2.0 |
| **Integrity verification** | Helm chart version pinned. Image digest pinned post-PH-04. |
| **Security advisories** | https://github.com/cert-manager/cert-manager/security/advisories |
| **Risk rating** | Low |

---

## SUP-05 -- Bitnami / VMware Tanzu (Sealed Secrets)

| Field | Detail |
|---|---|
| **Software** | Sealed Secrets controller |
| **Role** | Encrypts secrets for safe Git storage. Controller holds the private decryption key. |
| **Criticality** | Critical -- all credentials in the cluster are encrypted by this key |
| **Source** | https://github.com/bitnami-labs/sealed-secrets |
| **License** | Apache 2.0 |
| **Integrity verification** | Helm chart version pinned. Controller image pinned by digest. The RSA private key generated by the controller is backed up per [[SEC-04 Secrets and Key Management]]. |
| **Security advisories** | GitHub Security tab |
| **Risk rating** | Medium -- criticality is very high (holds decryption key), but attack surface is narrow (ClusterIP only, no external exposure) |
| **Notes** | Any compromise of the Sealed Secrets controller means all SealedSecrets must be considered exposed and all credentials must be rotated. |

---

## SUP-06 -- Grafana Labs (Loki, Grafana, Promtail)

| Field | Detail |
|---|---|
| **Software** | Loki, Grafana, Promtail (via kube-prometheus-stack and loki-stack Helm charts) |
| **Role** | Log aggregation (Loki + Promtail), observability and audit trail query interface (Grafana) |
| **Criticality** | High -- Annex 11 audit trail query depends on Loki and Grafana |
| **Source** | https://github.com/grafana |
| **License** | AGPLv3 (Grafana), Apache 2.0 (Loki) |
| **Integrity verification** | Helm chart versions pinned. Image digests pinned post-PH-04. |
| **Security advisories** | https://github.com/grafana/grafana/security/advisories |
| **Risk rating** | Low |
| **Notes** | Grafana has a history of authentication bypass CVEs. Patch within 7 days of any HIGH or CRITICAL advisory. Grafana is behind Authentik OIDC so direct exploit requires bypassing SSO first. |

---

## SUP-07 -- Prometheus (CNCF)

| Field | Detail |
|---|---|
| **Software** | Prometheus (via kube-prometheus-stack) |
| **Role** | Metrics collection, alerting, PQ evidence capture |
| **Criticality** | Medium -- no direct data handling, alert dependency |
| **Source** | https://github.com/prometheus/prometheus |
| **License** | Apache 2.0 |
| **Risk rating** | Low |

---

## SUP-08 -- Falco (CNCF / Sysdig)

| Field | Detail |
|---|---|
| **Software** | Falco + Falcosidekick |
| **Role** | Runtime security monitoring, Annex 11 audit trail generation |
| **Criticality** | Critical -- Falco is the audit trail mechanism. If Falco is not running, no runtime audit exists. |
| **Source** | https://github.com/falcosecurity/falco |
| **License** | Apache 2.0 |
| **Integrity verification** | Helm chart version pinned. eBPF driver verified at startup (`falco --version` shows Driver: bpf). Image digest pinned post-PH-04. |
| **Security advisories** | https://falco.org/blog/category/security/ |
| **Risk rating** | Low -- CNCF incubating project, widely deployed in regulated industries |
| **Notes** | Falco uses eBPF hooks at the kernel level. Any kernel upgrade on Talos may require a Falco update to maintain eBPF probe compatibility. Test Falco after every Talos upgrade. |

---

## SUP-09 -- OPA / Styra (Gatekeeper)

| Field | Detail |
|---|---|
| **Software** | OPA Gatekeeper |
| **Role** | Admission control, GxP policy enforcement layer |
| **Criticality** | High -- blocks non-compliant workloads from running |
| **Source** | https://github.com/open-policy-agent/gatekeeper |
| **License** | Apache 2.0 |
| **Integrity verification** | Helm chart version pinned. ConstraintTemplates are Rego code stored in Forgejo -- versioned and PR-reviewed. |
| **Security advisories** | GitHub Security tab |
| **Risk rating** | Low |
| **Notes** | Gatekeeper failurePolicy is set to Fail in production -- if Gatekeeper is unavailable, no new pods are admitted. This is intentional and aligned with the compliance posture. |

---

## SUP-10 -- nf-core Community

| Field | Detail |
|---|---|
| **Software** | nf-core/rnaseq pipeline (community-maintained Nextflow pipeline) |
| **Role** | The bioinformatics pipeline being validated. GAMP 5 Category 5 item. |
| **Criticality** | High -- the primary output of the platform depends on this pipeline |
| **Source** | https://github.com/nf-core/rnaseq |
| **License** | MIT |
| **Integrity verification** | Pipeline version pinned via `-r <version>` in Nextflow run command. Container images specified in the pipeline config are pulled by digest via Gatekeeper require-image-digest. Test profile used for OQ and PQ -- no patient data. |
| **Change visibility** | GitHub releases and CHANGELOG. nf-core publishes a release announcement for each version. |
| **Validation approach** | GAMP 5 Category 5: full IQ + OQ + PQ required. Reproducibility of results verified by MD5 checksum of MultiQC report across two independent runs. |
| **Risk rating** | Medium -- community-maintained (not a commercial vendor), but nf-core has a strong peer-review culture and the pipeline is widely used in regulated environments |
| **Notes** | When upgrading nf-core/rnaseq to a new major version, the full PQ must be re-run to verify reproducibility is maintained at the new version. Minor version updates require OQ re-run. |

---

## SUP-11 -- Cloudflare (Tunnel)

| Field | Detail |
|---|---|
| **Software** | Cloudflare Tunnel (cloudflared daemon) |
| **Role** | External access proxy -- the only inbound path through CGNAT |
| **Criticality** | Low for data integrity (no data passes through it permanently), Medium for availability (external access depends on it) |
| **Source** | https://github.com/cloudflare/cloudflared |
| **License** | Apache 2.0 |
| **Service SLA** | Cloudflare publishes a 99.99% uptime SLA for Tunnel |
| **Data handling** | TLS terminated at Ingress-NGINX inside the cluster -- Cloudflare proxies encrypted traffic, does not decrypt it (in this configuration) |
| **Risk rating** | Low -- availability dependency only, no data integrity risk |
| **Notes** | If Cloudflare Tunnel goes down, external access is lost. Internal LAN access remains fully functional. Documented as Risk R-08 in [[REG-03 Risk Register]]. |

---

## SUP-12 -- Container Registries (quay.io, ghcr.io, docker.io, registry.k8s.io)

| Field | Detail |
|---|---|
| **Software** | Public container registries |
| **Role** | Source of all container images used by platform components and pipeline containers |
| **Criticality** | High -- if registries are unavailable, no new pods can be created |
| **Integrity verification** | Gatekeeper approved-registries restricts pulls to only approved sources. Gatekeeper require-image-digest pins every image to a specific digest -- a registry cannot serve a different image for the same tag without breaking the digest check. |
| **Risk rating** | Medium -- availability risk if registries go down during a cluster event. Digest pinning mitigates supply chain risk. |
| **Notes** | All images are cached on the nodes after first pull. Short-term registry unavailability does not affect running pods or pod restarts on the same node. New pod scheduling on a new node would fail if the registry is unreachable. |

---

## Supplier Assessment Summary

| ID | Supplier | Criticality | Risk Rating | Last Assessed | Next Review |
|---|---|---|---|---|---|
| SUP-01 | Sidero Labs (Talos OS) | Critical | Low | May 2026 | Nov 2026 |
| SUP-02 | Rancher (local-path-provisioner) | High | Low | May 2026 | Nov 2026 |
| SUP-03 | ArgoCD / CNCF | Critical | Low | May 2026 | Nov 2026 |
| SUP-04 | cert-manager / CNCF | High | Low | May 2026 | Nov 2026 |
| SUP-05 | Bitnami (Sealed Secrets) | Critical | Medium | May 2026 | Nov 2026 |
| SUP-06 | Grafana Labs | High | Low | May 2026 | Nov 2026 |
| SUP-07 | Prometheus / CNCF | Medium | Low | May 2026 | Nov 2026 |
| SUP-08 | Falco / CNCF | Critical | Low | May 2026 | Nov 2026 |
| SUP-09 | OPA Gatekeeper / CNCF | High | Low | May 2026 | Nov 2026 |
| SUP-10 | nf-core community | High | Medium | May 2026 | Per pipeline upgrade |
| SUP-11 | Cloudflare | Low | Low | May 2026 | Nov 2026 |
| SUP-12 | Container registries | High | Medium | May 2026 | Nov 2026 |
