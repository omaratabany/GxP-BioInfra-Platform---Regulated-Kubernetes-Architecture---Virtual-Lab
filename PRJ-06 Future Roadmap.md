# PRJ-06 Future Roadmap

> Part of [[README]] | See also: [[PRJ-01 Strategic Rationale]], [[ADR-00 Decision Log]], [[REG-03 Risk Register]]

Planned improvements after Phase 7 completes. Organized into three tiers: near-term (directly closes known gaps or risks), medium-term (enhances capabilities), and long-term (production-grade expansion).

---

## Near-Term -- Close Existing Gaps (Post-CKA, Pre-Job)

These directly address residual risks or compliance gaps identified in the current platform.

### NT-01 -- Cilium CNI Migration

**What:** Replace Flannel with Cilium as the cluster CNI.
**Why:** Flannel does not enforce NetworkPolicy objects. Risk R-04 in [[REG-03 Risk Register]] is a Medium residual risk specifically because NetworkPolicy is unenforced. Cilium closes this completely.
**What it unlocks:**
- Full NetworkPolicy enforcement across all namespaces (SEC-05 policies become active)
- Hubble for network flow observability (feeds ISO 27001 8.16 monitoring control)
- Layer 7 policy support (HTTP-aware policies for MinIO and Authentik)
- Removes R-04 from the risk register entirely
**Pre-flight checklist:** Exists in [[SEC-05 Network Security Policy]]. All NetworkPolicy YAML is already written.
**Complexity:** Medium -- needs a drain/upgrade cycle. etcd snapshot before starting.

### NT-02 -- Cosign Image Signing

**What:** Sign all platform container images with Cosign (Sigstore) and add a Gatekeeper constraint that verifies signatures before admission.
**Why:** REG-01 marks Annex 11 Clause 5 as Partial because image digest pinning proves integrity but not provenance. Cosign adds provenance verification -- you can confirm the image was built by the expected pipeline, not just that it has the expected hash.
**What it unlocks:**
- Annex 11 Clause 5 becomes Full (not Partial)
- ISO 27001 8.21 supply chain control becomes stronger
- REG-03 Risk R-03 residual likelihood drops further
**Complexity:** Medium -- requires a signing key, a Cosign Gatekeeper policy, and a decision on which images to sign (at minimum: all Helm chart default images).

### NT-03 -- Trivy CVE Scanning in ArgoCD

**What:** Add Trivy image scanning as an ArgoCD admission plugin or as a Forgejo CI job that runs before ArgoCD syncs.
**Why:** OPS-09 patch management relies on advisory subscriptions. Trivy adds proactive scanning -- it catches vulnerabilities in images already deployed before an advisory is published.
**What it unlocks:**
- ISO 27001 8.8 becomes Implemented (not Partial)
- REG-01 CVE scanning gap is closed
- The Prometheus/Grafana patch history of HIGH CVEs becomes a managed metric, not a manual check
**Complexity:** Low -- Trivy is a single binary, Forgejo CI integration is straightforward.

---

## Medium-Term -- Enhanced Capabilities

### MT-01 -- Harbor Registry

**What:** Deploy Harbor as a private container registry, use it to store copies of all images used on the platform, and configure Gatekeeper to pull only from Harbor (not external registries).
**Why:** Currently the approved-registries constraint allows pulling from external registries (docker.io, quay.io, ghcr.io) at runtime. This means if a registry is unavailable, new pods cannot start. Harbor as a pull-through cache eliminates this dependency. Combined with Cosign, it becomes a fully controlled image distribution layer.
**Dependencies:** Requires additional storage (Harbor uses significant disk), realistically needs more than 320GB HDD. Good trigger for upgrading Beelink's storage.

### MT-02 -- HashiCorp Vault or External Secrets Operator

**What:** Replace Sealed Secrets with either Vault (for dynamic secrets) or the External Secrets Operator (ESO) pointing at a Vault backend.
**Why:** Sealed Secrets requires manually rotating all credentials on a schedule. Vault supports dynamic credentials -- MinIO service accounts that expire after 24 hours, not 90 days. This dramatically reduces the blast radius of any credential exposure.
**Complexity:** High -- Vault is a significant operational dependency. Consider using ESO with a managed Vault (HCP Vault or a simple Vault on the Minisforum if it becomes available).

### MT-03 -- External SIEM for Falco Events

**What:** Route Falco events to an external, immutable log target outside the cluster. Options: Grafana Cloud free tier (Loki-compatible), a small VPS running Loki, or a managed SIEM.
**Why:** REG-03 Risk R-05 is Medium residual because the audit trail lives on the same cluster it monitors. If the cluster is compromised, the audit trail could be compromised too. GOVERNANCE lock mitigates this but an external copy eliminates the risk entirely.
**Complexity:** Low to Medium -- Falcosidekick supports 50+ outputs. Adding a second Loki target is a config change, not an architectural change.

### MT-04 -- Thanos for Prometheus Long-Term Storage

**What:** Add Thanos as a sidecar to Prometheus, archiving metrics to MinIO for long-term retention beyond the 30-day TSDB retention.
**Why:** PQ evidence (resource utilization during pipeline runs) should be retained beyond the default Prometheus retention. Thanos stores Prometheus data as object blobs in MinIO with indefinite retention.
**Complexity:** Medium -- Thanos is a multi-component project but the sidecar pattern is well-documented.

---

## Long-Term -- Production-Grade Expansion

### LT-01 -- Three-Node Highly Available Control Plane

**What:** Add a third machine to the cluster as a second control plane node. This gives a 3-node etcd cluster (quorum requires at least 2 of 3 to be healthy).
**Why:** Risk R-02 (single CP node) cannot be fully closed without additional hardware. An HA control plane means Omen can be rebooted without the cluster going offline.
**Hardware requirements:** Any mini PC with 4+ cores and 8GB+ RAM. A used Minisforum or Beelink is sufficient.
**Impact:** Removes R-02 from the risk register. Unlocks the CKS (Certified Kubernetes Security Specialist) study track.

### LT-02 -- MinIO Distributed Mode

**What:** Run MinIO in distributed mode across two or more nodes instead of standalone mode.
**Why:** Risk R-01 (HDD failure) currently requires manual recovery and has a data loss window. MinIO distributed with erasure coding provides data redundancy across nodes -- a single node failure does not lose data.
**Requirements:** At minimum 2 nodes with equivalent storage. A second storage node with a 300GB+ drive is needed.

### LT-03 -- Multi-Environment Promotion

**What:** Implement the environment promotion model described in [[MOD-03 Configuration Management Standard]]: dev -> stg -> prd branches, separate ArgoCD Applications per environment, separate credentials per environment.
**Why:** Demonstrates the full enterprise GitOps workflow to prospective employers. Pharma companies run production pipelines in isolated environments with strict promotion gates.

### LT-04 -- Tetragon (eBPF Security Enforcement)

**What:** Add Tetragon alongside Falco. Where Falco observes and alerts, Tetragon can enforce -- it can kill a process or terminate network connections in response to a policy violation at kernel speed.
**Why:** Falco detects a privilege escalation in seconds. Tetragon prevents it in microseconds. Post-Cilium (which is a prerequisite since Tetragon integrates with Cilium), this becomes the most capable runtime security layer possible.

---

## Roadmap Timeline

```
Post-CKA (Month 1)
  NT-01: Cilium migration (highest priority)
  NT-02: Cosign image signing
  NT-03: Trivy CVE scanning

Months 2-3
  MT-01: Harbor registry (with storage upgrade)
  MT-03: External SIEM for Falco events

Months 4-6
  MT-02: Vault / External Secrets
  MT-04: Thanos long-term metrics

6+ Months (hardware investment required)
  LT-01: HA control plane (new hardware)
  LT-02: MinIO distributed (new hardware)
  LT-03: Multi-environment GitOps
  LT-04: Tetragon (post-Cilium)
```

---

## Risk Register Impact of Roadmap Completion

After NT-01 through NT-03 and MT-03 are complete, the risk register looks like this:

| Risk | Current Residual | After Roadmap Item | New Residual |
|---|---|---|---|
| R-03 Supply chain attack | Low | NT-02 (Cosign) + NT-03 (Trivy) | Very Low |
| R-04 No NetworkPolicy | Medium | NT-01 (Cilium) | Eliminated |
| R-05 On-cluster audit trail | Medium | MT-03 (External SIEM) | Low |

R-01 (HDD failure) and R-02 (single CP node) require hardware investment and are in the LT tier.
