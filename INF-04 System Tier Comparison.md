# INF-04 System Tier Comparison

> Part of [[README]] | See also: [[INF-03 Infrastructure Analysis]], [[ADR-01 Alternative Configurations]], [[INF-05 Hardware Scaling Guide]]

Side-by-side comparison across three tiers: minimal (single machine floor), current (my homelab), and ideal (production regulated environment).

---

## Tier Definitions

**Minimal:** Smallest hardware that runs the full stack. Some components at reduced capacity. Test profiles only.
**Current:** My homelab -- HP Omen + Beelink as in [[INF-01 Infrastructure Baseline]].
**Ideal:** Production-grade regulated environment. Three-node HA control plane, dedicated compute and storage.

---

## Node Topology

| Dimension | Minimal | Current | Ideal |
|---|---|---|---|
| Control plane nodes | 1 (untainted) | 1 (untainted) | 3 (HA etcd) |
| Worker nodes | 0 | 1 (Beelink) | 4+ |
| Storage nodes | Shared | 1 (Beelink HDD) | 3 (dedicated) |
| Monitoring node | Shared | Shared | 1 (dedicated) |
| HA status | None | None | Full HA |
| Single point of failure | Everything | CP node + storage node | None |

---

## Compute Resources

| Dimension | Minimal | Current | Ideal |
|---|---|---|---|
| Total CPU | 4 cores | 12 cores | 96+ cores |
| Allocatable CPU | ~3.5 cores | ~12 cores | ~90 cores |
| Available for pipelines | ~1 core | ~6-8 cores | ~60+ cores |
| Parallel pipeline tasks | 1 (sequential only) | 2-4 concurrent | 20+ concurrent |
| STAR aligner full genome | Not possible | Not possible | Yes (128GB RAM workers) |

---

## Memory Resources

| Dimension | Minimal | Current | Ideal |
|---|---|---|---|
| Total RAM | 8GB | 23.5GB | 384GB+ |
| Available for pipelines | ~2GB | ~10-12GB | ~200GB |
| nf-core/rnaseq test profile | Borderline | Comfortable | Comfortable |
| nf-core/rnaseq full genome | No | No | Yes |
| Concurrent pipeline runs | 1 | 1-2 | 10+ |

---

## Storage

| Dimension | Minimal | Current | Ideal |
|---|---|---|---|
| Total persistent storage | 100GB (single SSD) | 320GB HDD | 48TB+ (distributed) |
| Replication | None | None | 3x minimum |
| Single point of failure | Yes | Yes | No |
| Storage throughput | ~100 MB/s | ~100 MB/s (HDD) | 2+ GB/s (NVMe RAID) |
| MinIO mode | Standalone | Standalone | Distributed (4+ nodes) |
| Max pipeline output | ~50GB | ~200GB | Multi-TB |

---

## Networking

| Dimension | Minimal | Current | Ideal |
|---|---|---|---|
| Node interconnect | 1GbE | 1GbE | 10GbE / 25GbE |
| CNI | Flannel | Flannel (Cilium post-CKA) | Cilium from day 1 |
| NetworkPolicy enforcement | No | No (Cilium pending) | Yes -- full L3/L4/L7 |
| External access | CGNAT + Cloudflare Tunnel | CGNAT + Cloudflare Tunnel | Direct public IP |
| mTLS between services | No | No | Yes -- Cilium or Linkerd |

---

## GxP Compliance Posture

| Requirement | Minimal | Current | Ideal |
|---|---|---|---|
| Audit trail | Falco + Loki (basic rules) | Falco + Loki (custom GxP rules) | Falco + Loki + external SIEM |
| Access control | RBAC only | RBAC + Authentik SSO | RBAC + Authentik + hardware MFA |
| Change management | ArgoCD + GitHub | ArgoCD + Forgejo (PRs + webhooks) | ArgoCD + Forgejo + mandatory approval gates |
| Image provenance | None | Gatekeeper require-image-digest | Gatekeeper + Cosign signing + Harbor registry |
| NetworkPolicy isolation | None | None (Cilium pending) | Full enforcement |
| Secret management | Sealed Secrets | Sealed Secrets | Vault with dynamic secrets |
| Backup and recovery | Manual etcd snapshot | etcd snapshot + MinIO backup SOP | Automated + tested recovery |
| Annex 11 completeness | Partial | Substantial | Full |

---

## Pipeline Performance -- nf-core/rnaseq Test Profile

| Metric | Minimal | Current | Ideal |
|---|---|---|---|
| Total wall time | 30-60 min | 15-25 min | 5-10 min |
| Peak CPU | ~3.5 cores (100%) | ~8 cores (~66%) | ~30 cores (~30%) |
| Peak RAM | 3.5-4GB (~70%) | 6-8GB (~35%) | 8-10GB (<5%) |
| Concurrent runs possible | 0 (at limit) | 1-2 | 20+ |
| OOM risk | High | Low | None |

---

## Pipeline Performance -- Full Human Genome

| Metric | Minimal | Current | Ideal |
|---|---|---|---|
| Feasibility | Not possible | Not possible | Yes |
| RAM required (STAR) | 32GB (cluster total < 8GB) | 32GB (cluster total ~22GB) | Available on 128GB workers |
| Total wall time | N/A | N/A | 4-12 hours per sample |
| Concurrent samples | N/A | N/A | 4-8 |

---

## Platform Steady-State Resource Consumption

| Component | Minimal | Current | Ideal |
|---|---|---|---|
| ArgoCD | 200m / 300Mi | 300m / 600Mi | 500m / 1GB (HA) |
| Ingress-NGINX | 50m / 64Mi | 100m / 128Mi | 200m / 256Mi |
| Cert-Manager | 30m / 48Mi | 50m / 64Mi | 100m / 128Mi |
| Sealed Secrets | 20m / 32Mi | 50m / 64Mi | 50m / 64Mi |
| Prometheus + Alertmanager | 200m / 512Mi | 300m / 800Mi | 500m / 2GB |
| Grafana | 50m / 128Mi | 100m / 256Mi | 200m / 512Mi |
| Loki | 100m / 256Mi | 200m / 512Mi | 500m / 1GB |
| Forgejo | 100m / 256Mi | 200m / 512Mi | 300m / 768Mi |
| Authentik (all components) | 300m / 512Mi | 700m / 1,312Mi | 1,500m / 3GB (HA) |
| MinIO | 200m / 512Mi | 250m / 512Mi | 1,000m / 2GB (distributed) |
| OPA Gatekeeper | 100m / 128Mi | 200m / 256Mi (x2) | 400m / 512Mi (x3) |
| Falco (per node) | 100m / 128Mi | 200m / 256Mi | 200m / 256Mi |
| Falcosidekick | 20m / 32Mi | 50m / 64Mi | 100m / 128Mi |
| **Total platform overhead** | **~1,470m / ~2,912Mi** | **~2,700m / ~5,376Mi** | **~6,050m / ~12GB** |

---

## Failure Scenarios

| Failure | Minimal | Current | Ideal |
|---|---|---|---|
| Control plane node down | Cluster dead | Cluster dead | Cluster continues (2/3 CP quorum) |
| Worker node down | N/A | All persistent storage unavailable | Workloads reschedule, replication maintains access |
| HDD failure | Data loss | Total data loss | No data loss (replication factor 3) |
| etcd corruption | Manual restore from snapshot | Manual restore from snapshot | Automated restore from peer |
| Pipeline pod OOM | Run fails, data may be partial | Run fails gracefully, MinIO partial output retained | Orchestration retries failed task |

---

## Cost

| Tier | Hardware Cost | Power Draw | Annual Electricity (~15c/kWh) |
|---|---|---|---|
| Minimal (1 mini PC) | ~$300-500 used | ~25W | ~$33 |
| Current (Omen + Beelink) | Already owned | ~120W combined | ~$158 |
| Ideal (8+ nodes) | $15,000-40,000 | 1,500-3,000W | $2,000-4,000 |
| Cloud equivalent (AWS EKS) | N/A | N/A | $2,000-8,000/month |

My current setup achieves approximately 60-70% of the GxP compliance posture of an ideal deployment at zero ongoing cost. The remaining gap is network isolation, storage replication, and HA -- all architectural, not functional.
