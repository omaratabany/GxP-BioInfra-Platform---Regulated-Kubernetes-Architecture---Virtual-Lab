# ADR-00 Decision Log

> Part of [[README]] | See also: [[ADR-01 Alternative Configurations]], [[INF-03 Infrastructure Analysis]]

ADR stands for Architecture Decision Record -- the industry standard format for documenting why a technical decision was made, not just what was decided. Every significant choice in this project is recorded here with full context, reasoning, and rejected alternatives.

---

## D-01 -- Talos OS

**Decision:** Talos OS on both nodes.

**Context:** Already running Talos before this project started.

**Why:** Immutable OS -- no SSH, no package manager, no shell. Minimal attack surface, directly supports Annex 11 security requirements. API-driven config means every machine change goes through `talosctl`, is version-controlled as a YAML patch, and leaves an audit trail. Production-grade and aligned with CKS exam topics.

**Rejected:** Ubuntu + kubeadm introduces SSH, apt, and mutable OS -- all add attack surface. Proxmox + K3s adds VM management overhead and K3s strips admission controller features needed for the GxP layer. RKE2 is philosophically similar but heavier for homelab management.

**References:** [[INF-01 Infrastructure Baseline]], [[PH-00 Cluster Preparation]]

---

## D-02 -- Flannel CNI with Planned Cilium Migration

**Decision:** Keep Flannel now. Migrate to Cilium post-CKA.

**Context:** Flannel was already in place. CNI replacement on a live 2-node cluster without HA risks downtime.

**Why deferred:** CKA week 5 covers networking -- migrating CNI during that same week is high-risk timing. Flannel is acceptable for PH-00 through PH-03 since NetworkPolicy enforcement is not yet relied upon.

**Why Cilium is the correct long-term answer:** Flannel ignores NetworkPolicy objects -- that is a GxP gap. Cilium adds NetworkPolicy enforcement, Hubble observability, and lower eBPF-based overhead at scale.

**Trade-off accepted:** Network-level namespace isolation is not enforced until post-CKA. Documented in the OQ as a known gap.

**References:** [[INF-01 Infrastructure Baseline]], [[PH-04 OPA Gatekeeper]], [[ADR-01 Alternative Configurations]] A-02

---

## D-03 -- HP Omen Untainted

**Decision:** Remove the control-plane taint from the Omen.

**Context:** Omen started as a dedicated control plane. With only 2 nodes, keeping it tainted leaves 13GB of RAM idle.

**Why:** 13.5GB of RAM sitting idle while Beelink is the only scheduling target is wasteful. In a 2-node cluster without HA, losing the control plane kills the cluster regardless of taint -- the taint provides no meaningful separation in this topology.

**Risk accepted:** etcd competes with application workloads for Omen RAM. Documented in [[INF-03 Infrastructure Analysis]].

**References:** [[INF-01 Infrastructure Baseline]], [[PH-00 Cluster Preparation]]

---

## D-04 -- Beelink as Dedicated Storage Node

**Decision:** Label Beelink as storage node, pin all PVC-dependent workloads to it.

**Why:** The 320GB HDD is physically on Beelink. Using it for storage is the most direct mapping. Beelink's 4 cores and 7.5GB RAM are not enough for memory-intensive pipeline compute -- that goes to the Omen. Separated compute and storage means MinIO disk IO does not compete with CPU-bound pipeline tasks.

**Misjudgment acknowledged:** The original plan did not prevent Nextflow worker pods from landing on Beelink. Fixed with explicit `nodeSelector: node-role: infra` on all pipeline pods. See [[INF-03 Infrastructure Analysis]] Misjudgment 1.

**References:** [[INF-01 Infrastructure Baseline]], [[PH-00 Cluster Preparation]], [[PH-03 MinIO Object Storage]]

---

## D-05 -- GitHub as Temporary GitOps Source

**Decision:** Use GitHub as the ArgoCD source until Forgejo is deployed.

**Why:** Forgejo is PH-01. ArgoCD needs a source repo from day one. Bootstrapping Forgejo from Forgejo is a chicken-and-egg problem. GitHub's public repo avoids authentication complexity during initial setup.

**Transition:** Once Forgejo is live, ArgoCD's source is updated to point at Forgejo. GitHub becomes a public mirror. The changeover is a single commit to the ArgoCD ApplicationSet.

**References:** [[PH-01 Forgejo]], [[INF-01 Infrastructure Baseline]]

---

## D-06 -- Forgejo over GitLab CE or Gitea

**Decision:** Forgejo as my self-hosted Git platform.

**Why:** Hard fork of Gitea maintained by a community-governed organisation -- no single entity can change the license or abandon it. GitLab CE requires 4GB+ RAM minimum, which is 55% of Beelink's total RAM for one service. Forgejo is API-compatible with GitHub -- ArgoCD and every tool that works with GitHub works with Forgejo without modification.

**References:** [[PH-01 Forgejo]], [[ADR-01 Alternative Configurations]] A-06

---

## D-07 -- Authentik over Keycloak or Dex

**Decision:** Authentik as my OIDC/SSO provider.

**Why:** Keycloak's minimum RAM footprint is 1.5-2GB -- that is 20-28% of Beelink's available RAM for one service. Dex is lightweight but is a proxy/connector, not a full identity provider. Authentik is a full IdP at 400-600MB, has a clean admin UI, supports OIDC, SAML, LDAP outbound, and hardware MFA. Its group-to-RBAC mapping is well-documented for ArgoCD and Grafana specifically.

**References:** [[PH-02 Authentik SSO]], [[ADR-01 Alternative Configurations]] A-07

---

## D-08 -- MinIO Standalone over Rook-Ceph or Longhorn

**Decision:** MinIO in standalone mode.

**Why:** Rook-Ceph requires a minimum of 3 storage nodes -- I have one. Architecturally impossible. Longhorn provides block storage (RWO PVCs), not S3-compatible object storage -- Nextflow and Loki need the S3 API specifically. MinIO standalone runs on a single node, supports the full S3 API, and is the on-premise standard in bioinformatics deployments.

**Trade-off accepted:** No replication. One drive, one node. Drive failure loses everything. Documented in [[INF-03 Infrastructure Analysis]] and mitigated in the Backup and Recovery SOP.

**References:** [[PH-03 MinIO Object Storage]], [[ADR-01 Alternative Configurations]] A-08

---

## D-09 -- OPA Gatekeeper over Kyverno

**Decision:** OPA Gatekeeper for admission control policy enforcement.

**Why:** Gatekeeper uses the Rego policy language -- the industry standard for policy-as-code. Learning Rego has value beyond this project. Gatekeeper's audit mode lets me identify violations in existing workloads before switching to deny mode -- the safe rollout path for a live cluster. Gatekeeper is the CNCF reference implementation for K8s admission control and appears more frequently in regulated industry job descriptions than Kyverno.

**References:** [[PH-04 OPA Gatekeeper]], [[ADR-01 Alternative Configurations]] A-09

---

## D-10 -- Falco for Runtime Security

**Decision:** Falco for runtime security monitoring and audit event generation.

**Why:** CNCF-graduated runtime security standard, open source, widely used in regulated industries. Falcosidekick provides a routing layer to any destination without touching Falco itself. Tetragon is more powerful but requires Cilium CNI -- unsupported with Flannel. Post-Cilium migration, Tetragon becomes viable. Sysdig is the commercial version of Falco -- the open-source Falco covers my audit trail use case without the cost.

**References:** [[PH-05 Falco Runtime Security]], [[ADR-01 Alternative Configurations]] A-10

---

## D-11 -- Nextflow with K8s Executor

**Decision:** Nextflow with the Kubernetes executor.

**Why:** nf-core pipelines are Nextflow-native -- using any other engine means rewriting the pipeline. The K8s executor is first-class and explicitly supported by the nf-core community. Nextflow has native S3/MinIO support. Cromwell has a more complex K8s integration and a Java-heavy server component. Snakemake's K8s support is less mature and its ecosystem is more academic than regulated-environment focused.

**References:** [[PH-06 Nextflow and nf-core]], [[ADR-01 Alternative Configurations]] A-11

---

## D-12 -- Sealed Secrets over Vault or SOPS

**Decision:** Sealed Secrets for secret management.

**Why:** Lets me commit encrypted secrets to a public Git repository -- the sealed blob is decryptable only by the controller in my specific cluster. External Secrets Operator requires an external secret store (Vault, AWS Secrets Manager) which is an additional dependency I do not have yet. SOPS requires GPG key or KMS infrastructure that adds setup complexity. Sealed Secrets gives the right balance for a public-repo GitOps project: encrypted at rest in Git, no external dependency, works with ArgoCD out of the box.

**References:** [[PH-01 Forgejo]], [[INF-01 Infrastructure Baseline]]
