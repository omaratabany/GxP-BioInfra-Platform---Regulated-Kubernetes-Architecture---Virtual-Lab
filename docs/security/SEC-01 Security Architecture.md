# SEC-01 Security Architecture

> Part of [[README]] | See also: [[SEC-02 Component Hardening Guide]], [[SEC-03 Incident Response Playbook]], [[SEC-04 Secrets and Key Management]], [[SEC-05 Network Security Policy]], [[REG-02 ISO 27001 Alignment]]

This file describes the security model of the platform -- how threats are identified, what controls exist at each layer, and how those controls relate to each other. Every security decision traces back here.

---

## Threat Model

### Assets Being Protected

| Asset | Classification | Location | Risk if Compromised |
|---|---|---|---|
| Pipeline output data | Sensitive research data | MinIO pipeline-output bucket | Regulatory submission invalidated |
| Audit trail logs | Compliance evidence | Loki via MinIO loki-chunks | Annex 11 audit trail destroyed |
| Infrastructure credentials | Critical | Sealed Secrets in Git, cluster controller | Full cluster compromise |
| etcd state | Critical | Omen node, ephemeral disk | Cluster unrecoverable without snapshot |
| Container images | High | Pulled from external registries | Malicious code injected into pipeline |
| User identities | High | Authentik PostgreSQL | Unauthorised access to all platform tools |
| Forgejo repo contents | High | Forgejo PVC on Beelink HDD | Infrastructure manifests tampered |
| Pipeline input data | Moderate | MinIO pipeline-input bucket | Incorrect analysis results |

### Threat Actors

| Threat Actor | Likelihood | Capability | Primary Concern |
|---|---|---|---|
| Malicious container image | Medium | Code execution in pipeline pod | Supply chain attack -- image substitution |
| Compromised pipeline job | Medium | File system access within pod | Data exfiltration or audit trail tampering |
| Unauthorised internal user | Low | Cluster API access | Privilege escalation, data access |
| External attacker (internet) | Low (CGNAT protects) | Network scanning | Limited -- no inbound ports, Cloudflare Tunnel only |
| Misconfigured workload | High | Unintended resource consumption | Cluster instability affecting pipeline results |
| Drive failure | Medium | Physical | Permanent data loss |

### Attack Vectors

```
External Network
  Cloudflare Tunnel (only ingress path)
    Ingress-NGINX
      Internal services (TLS terminated)

Internal Network (LAN)
  MetalLB (192.168.0.200)
    Ingress-NGINX
      Internal services

Cluster Internal
  Pod-to-pod (Flannel -- no NetworkPolicy until Cilium)
  API server (RBAC enforced)
  etcd (Talos-managed, not externally exposed)

Supply Chain
  Container image pull (Gatekeeper approved-registries + require-image-digest)
  Helm chart pull (ArgoCD watches pinned chart versions)
  Infrastructure manifest (Forgejo PR + ArgoCD -- all changes tracked)
```

---

## Defense Layers

I operate a layered defense model. Each layer independently limits the blast radius of a breach in the layer above it.

### Layer 1 -- Perimeter

| Control | Implementation | What It Stops |
|---|---|---|
| CGNAT | DU Telecom network | No inbound connections without Cloudflare Tunnel |
| Cloudflare Tunnel | Configured in homelab | Eliminates exposed port scanning surface |
| TLS everywhere | Cert-Manager, self-signed CA | Traffic interception between browser and services |
| UniFi VLAN | LAN segmentation | Lateral movement from non-cluster devices |

### Layer 2 -- Cluster Admission

| Control | Implementation | What It Stops |
|---|---|---|
| Image digest pinning | Gatekeeper require-image-digest | Silent image updates, tag mutation attacks |
| Approved registries | Gatekeeper approved-registries | Pulling images from untrusted sources |
| Resource limits | Gatekeeper require-resource-limits | Runaway pods starving cluster resources |
| No privileged containers | Gatekeeper disallow-privileged | Container breakout via privileged mode |
| Required labels | Gatekeeper require-labels | Untracked workloads with no audit identity |

### Layer 3 -- Identity and Access

| Control | Implementation | What It Stops |
|---|---|---|
| Centralised OIDC | Authentik | Password sprawl, shared credentials |
| RBAC | K8s Roles + Authentik group sync | Lateral movement between namespaces |
| No local passwords | Enforced post-PH-02 | Credential bypass of SSO |
| Scoped ServiceAccounts | nextflow-runner Role (not ClusterRole) | Pipeline pods accessing other namespaces |
| Sealed Secrets | Controller-only decryption | Credential exposure in public Git |

### Layer 4 -- Runtime

| Control | Implementation | What It Stops |
|---|---|---|
| Falco eBPF rules | DaemonSet on all nodes | Undetected exec, file writes, privilege escalation |
| Falco CRITICAL alerts | Privilege escalation rule | Immediate detection of container breakout attempts |
| Loki tamper-evident log | MinIO backend, kernel-level capture | Post-incident log manipulation |
| Prometheus alerting | kube-prometheus-stack | Resource anomalies indicating compromise |

### Layer 5 -- Data

| Control | Implementation | What It Stops |
|---|---|---|
| MinIO IAM policies | Per-bucket access control | Pipeline pods reading other pipelines' data |
| PVC reclaim policy Retain | StorageClass reclaimPolicy | Accidental data deletion on PVC release |
| etcd snapshots | Pre-phase-transition | Cluster state loss from etcd corruption |
| MinIO bucket mirroring | mc mirror to external target | Single-drive failure losing all pipeline data |

---

## Security Control Ownership

Each control has a phase where it becomes active. Controls from earlier phases remain active throughout.

| Phase | Controls Activated |
|---|---|
| PH-00 | Node labels enforce workload placement. Talos immutable OS baseline. |
| PH-01 | Sealed Secrets for all credentials. Forgejo PR-based change control. |
| PH-02 | Authentik OIDC. No local passwords. Group-based RBAC. |
| PH-03 | MinIO IAM per bucket. Loki migrated to persistent MinIO backend. |
| PH-04 | Gatekeeper admission control. All five constraints in deny mode. |
| PH-05 | Falco eBPF runtime monitoring. Tamper-evident audit trail live. |
| Post-CKA | Cilium CNI. NetworkPolicy enforcement. Hubble observability. |

---

## Security Posture Statement

My current posture is strong at the admission and identity layers and adequate at the runtime layer. The primary gaps are:

1. **NetworkPolicy not enforced** (Flannel limitation) -- pods can reach any other pod. Mitigated by RBAC at the API level. Fixed post-CKA with Cilium.
2. **Single-node control plane** -- Omen compromise = cluster compromise. Mitigated by etcd snapshots and Talos's locked-down surface.
3. **No image signing** -- digest pinning prevents silent updates but does not verify provenance. Future: Cosign + Sigstore.
4. **No SIEM** -- Falco events go to Loki but Loki is on the same cluster. For production, events should ship to an external, immutable SIEM.
5. **Single storage node** -- Beelink HDD failure loses all data. Mitigated by MinIO mirroring SOP.

Each gap is documented in [[REG-03 Risk Register]] with likelihood, impact, and residual risk rating.


---

## G-19 Correction -- Prometheus Alerting Control (SEC-01 Layer 4)

SEC-01 Layer 4 lists "Prometheus alerting -- Resource anomalies indicating compromise" as an active control. This was previously hollow -- no alert rules were defined. As of May 2026, the alert rules are now fully specified in [[INF-06 Observability and Alerting]].

The Layer 4 Prometheus alerting control now covers: NodeNotReady (critical), NodeCPUSaturation (warning), NodeMemoryPressure (critical), BeelinkHDDUsageCritical (warning + critical), PodCrashLooping (warning), PodOOMKilled (warning), GatekeeperWebhookDown (critical), PVCNearFull (warning), FalcoDown (critical), GatekeeperViolationsIncreasing (warning), CertificateExpirySoon (warning), SealedSecretsControllerDown (critical), PipelinePodPending (warning), and PipelineResourceQuotaNearLimit (warning).

These rules are defined as a PrometheusRule object deployed via `apps/monitoring/gxp-alert-rules.yaml`. The Layer 4 control is now fully implemented.

---

## G-19 Correction -- Prometheus Alerting Control (Layer 4)

The Layer 4 defense table lists "Prometheus alerting -- Resource anomalies indicating compromise" as an active control. This is accurate in intent but was hollow when written -- no alert rules existed. Alert rules are now fully defined in [[INF-06 Observability and Alerting]] covering platform health, security events, resource consumption, and GxP-specific failure modes (Falco down, Gatekeeper down, Loki ingestion stopped, MinIO down). The Layer 4 control is now backed by concrete PromQL rules loaded via a PrometheusRule CRD. See INF-06 for the full rule set.
