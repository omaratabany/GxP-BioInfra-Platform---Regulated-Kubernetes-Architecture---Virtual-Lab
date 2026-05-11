# INF-03 Infrastructure Analysis

> Part of [[README]] | See also: [[INF-01 Infrastructure Baseline]], [[INF-04 System Tier Comparison]], [[ADR-00 Decision Log]], [[INF-05 Hardware Scaling Guide]]

An honest analysis of what my hardware can actually sustain, where the real limits are, what I got wrong in the initial design, and what this platform looks like when the hardware ceiling is removed.

---

## What I Am Actually Running

```
Control plane: HP Omen (talos-asj-72z)
  8 cores / 7950m allocatable
  16GB RAM / 15.2GB allocatable
  ~950GB ephemeral SSD
  IP: 192.168.0.134

Worker: Beelink mini PC (talos-v3h-4m1)
  4 cores / 3950m allocatable
  7.5GB RAM / 7.2GB allocatable
  113GB SSD + 320GB HDD (/var/mnt/hdd, XFS)
  IP: 192.168.0.202

Total allocatable: ~12 cores / ~22.4GB RAM / 320GB HDD persistent
```

---

## Utilization Targets Per Node

### HP Omen (infra node)

| Workload | CPU Request | RAM Request |
|---|---|---|
| ArgoCD (3 components) | 300m | 600Mi |
| Ingress-NGINX | 100m | 128Mi |
| Cert-Manager | 50m | 64Mi |
| Sealed Secrets | 50m | 64Mi |
| Prometheus + Alertmanager | 300m | 800Mi |
| Grafana | 100m | 256Mi |
| Forgejo (PH-01) | 200m | 512Mi |
| Authentik server + worker + PG + Redis (PH-02) | 700m | 1,312Mi |
| OPA Gatekeeper x2 replicas (PH-04) | 100m | 256Mi |
| Falco on Omen (PH-05) | 200m | 256Mi |
| Nextflow head pod (PH-06) | 100m | 256Mi |
| **Total estimated** | **~2,200m** | **~4,500Mi** |

Leaves ~5,750m CPU and ~10GB RAM headroom for pipeline worker pods. The Omen has the capacity. The risk is stateful workloads writing to ephemeral disk instead of PVCs -- data gone on reboot.

### Beelink (storage node)

| Workload | CPU Request | RAM Request | Storage |
|---|---|---|---|
| Loki | 200m | 512Mi | HDD via MinIO |
| Promtail | 50m | 64Mi | - |
| MinIO (PH-03) | 250m | 512Mi | 200GB+ PVC on HDD |
| Falco on Beelink (PH-05) | 200m | 256Mi | - |
| **Total estimated** | **~700m** | **~1,350Mi** | |

Leaves ~3,250m CPU and ~5.8GB RAM. Nextflow worker pods should NOT schedule here -- see Misjudgment 1 below.

---

## Real Limits I Am Working Within

### Limit 1 -- Single Control Plane Node
One etcd member. Omen goes down, the entire cluster goes with it. No scheduling, no reconciliation. Manual recovery from etcd snapshot is the only path. Acceptable for a portfolio project. Disqualifying for production.

### Limit 2 -- Beelink RAM for Pipeline Compute
4 cores and 7.5GB RAM handles MinIO and Loki fine. It cannot run memory-intensive bioinformatics tasks. STAR aligner (used in nf-core/rnaseq) needs 32GB for a full human genome. The test profile runs within 4-6GB -- that is the ceiling for Beelink.

My fix: route all pipeline compute pods to the Omen via `nodeSelector: node-role: infra`. Beelink handles storage IO only.

### Limit 3 -- Flannel Has No NetworkPolicy Enforcement
Flannel ignores NetworkPolicy objects. Any pod can reach any other pod on any port. This is a GxP gap -- no network-level namespace isolation. I document this in the OQ and fix it post-CKA with Cilium.

### Limit 4 -- All Persistent Data on One HDD, No Replication
MinIO, Loki chunks, Forgejo repos, Prometheus TSDB -- all on one 320GB HDD with no RAID. Drive failure loses everything. Mitigation: etcd snapshots and MinIO mirroring to an external target per the Backup and Recovery SOP in [[PH-07 GxP Validation Documentation]].

### Limit 5 -- CGNAT, No Direct Inbound Connections
DU Telecom uses CGNAT. No public IP, no port forwarding. Cloudflare Tunnel is my only external access path. Internal `*.homelab` access is unaffected and does not depend on Cloudflare.

### Limit 6 -- Single MetalLB IP for All Ingress
Everything routes through `192.168.0.200`. One IP, one Ingress controller. No separation between admin and pipeline-facing services at the network edge. Low priority at current scale.

---

## Misjudgments I Have Identified

### Misjudgment 1 -- Beelink as Storage AND Compute
I originally left default scheduling in place. Nextflow worker pods could land on Beelink and compete with MinIO and Loki for 4 cores and 7.5GB RAM. The fix is explicit `nodeSelector: node-role: infra` on all pipeline pods in the Nextflow executor config. Storage stays on Beelink. Compute goes to the Omen.

### Misjudgment 2 -- No Dedicated etcd Node
etcd is latency-sensitive and shares the Omen's 16GB RAM with application workloads. Under heavy scheduling load, etcd read/write latency increases. Accepted for current hardware -- documented in the IQ.

### Misjudgment 3 -- Prometheus TSDB on Ephemeral Storage
If kube-prometheus-stack is writing TSDB data to the Omen's ephemeral disk, every Omen reboot loses metric history. I need to migrate Prometheus TSDB to a PVC on `local-hdd` before PH-05, because Falco alert history in Prometheus must persist.

### Misjudgment 4 -- Homepage and Kubernetes Dashboard Running Permanently
Both consume RAM that is more useful for pipeline workloads. Neither does anything `kubectl` cannot. Removing or scaling both to zero recovers 200-300Mi RAM.

### Misjudgment 5 -- No Resource Requests or Limits on Current Stack
Everything currently deployed has no enforced limits. A misbehaving pod can starve the scheduler with no signal until something OOMs. Before PH-04 enforces this via Gatekeeper, I need to manually add resource requests and limits to all current Helm releases.

---

## What Ideal Looks Like Without Hardware Constraints

### Node Layout

| Role | Count | Spec | Purpose |
|---|---|---|---|
| Control plane | 3 | 16 cores / 64GB RAM / 500GB NVMe | HA etcd, API server, scheduler |
| Compute workers | 4+ | 32 cores / 128GB RAM / 1TB NVMe | Pipeline execution, Nextflow workers |
| Storage nodes | 3 | 8 cores / 32GB RAM / 4x 4TB NVMe | Ceph or MinIO distributed |
| Monitoring node | 1 | 8 cores / 32GB RAM / 2TB NVMe | Prometheus, Grafana, Loki -- isolated |

### Network
10GbE between nodes, Cilium from day one, VLAN segmentation per service tier, direct public IP or datacenter hosting, mTLS between services via Cilium or Linkerd.

### Storage
Ceph distributed across 3 storage nodes with replication factor 3, MinIO in distributed mode with erasure coding, separate RWX StorageClass for shared pipeline working directories, off-cluster replication to a cloud target on a 15-minute cadence.

---

## Priority Order for Working Within Current Hardware

1. Pin pipeline compute pods to Omen via nodeSelector
2. Pin storage services to Beelink HDD PVCs
3. Remove Homepage and K8s Dashboard -- recover 200-300Mi RAM
4. Add resource requests and limits to all current Helm releases before enabling Gatekeeper deny mode
5. Migrate Prometheus TSDB to a PVC on `local-hdd` before PH-05
6. Use nf-core test profile exclusively -- do not attempt full genome data on this cluster
7. Take etcd snapshots before every phase transition
