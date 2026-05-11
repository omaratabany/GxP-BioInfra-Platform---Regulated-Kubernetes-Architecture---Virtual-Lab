# INF-01 Infrastructure Baseline

> Part of [[README]] | See also: [[INF-02 Architecture and Components]], [[OPS-02 Reference Commands]]

---

## Network

```
ISP:         DU Telecom, Dubai
Topology:    CGNAT -- no inbound port forwarding possible
Remote:      Cloudflare Tunnel for external access
LAN:         UniFi with VLAN segmentation
DNS:         Local wildcard *.homelab resolved via hosts file / local DNS
```

---

## My Machines

| Machine | Role | CPU | RAM | Storage | IP |
|---|---|---|---|---|---|
| MacBook Air M5 | Primary dev, kubectl/talosctl control | Apple M5 | 16GB | - | DHCP |
| HP Omen (talos-asj-72z) | Talos control plane + workloads (untainted) | 8 core | 16GB | ~950GB | 192.168.0.134 |
| Beelink mini PC (talos-v3h-4m1) | Talos worker + storage node | 4 core / 3950m allocatable | 7.5GB / 7.2GB allocatable | 113GB SSD + 320GB HDD | 192.168.0.202 |
| CachyOS build | CKA study VMs only -- NOT used for this project | i9-13900KF | 64GB | NVMe | 192.168.0.169 |
| Minisforum MS-01 | Unraid home server -- OFF LIMITS | i9-12900H | 64GB | ~62TB DAS | DHCP |

CachyOS and MS-01 are explicitly excluded. CachyOS is my CKA study environment. MS-01 is off-limits.

---

## Live Cluster State

**Cluster:** Talos v1.12.6 / Kubernetes v1.35.2

### HP Omen -- talos-asj-72z (Control Plane)

```
IP:           192.168.0.134
CPU:          8 cores / 7950m allocatable
RAM:          16GB / 15.2GB allocatable
Disk:         ~950GB ephemeral
Taint:        REMOVED -- untainted, schedules workloads
Current use:  660m CPU / 1512Mi RAM
Free:         ~13.5GB RAM available
```

### Beelink -- talos-v3h-4m1 (Worker / Storage)

```
IP:           192.168.0.202
CPU:          4 cores / 3950m allocatable
RAM:          7.5GB / 7.2GB allocatable
Disk:         113GB SSD (Talos OS + ephemeral) + 320GB HDD (/var/mnt/hdd, XFS)
Current use:  100m CPU / 50Mi RAM (~5 pods only)
Free:         ~7.1GB RAM available
```

Confirmed disk layout:
```
sda    128GB   GPT   Talos OS (EFI, META, STATE, EPHEMERAL)
sdb    320GB   GPT   HDD -- sdb1 XFS, mounted at /var/mnt/hdd
```

---

## Currently Deployed Stack

| App | Namespace | How Deployed | Notes |
|---|---|---|---|
| ArgoCD | argocd | Helm via ApplicationSet | GitOps controller |
| MetalLB | metallb-system | Helm | L2 mode, 192.168.0.200 pool |
| Ingress-NGINX | ingress-nginx | Helm | Assigned 192.168.0.200 |
| Cert-Manager | cert-manager | Helm | TLS for *.homelab |
| Sealed Secrets | sealed-secrets | Helm | Secret encryption at rest |
| kube-prometheus-stack | monitoring | Helm | Prometheus + Grafana + alertmanager |
| Loki + promtail | monitoring | Helm | Log aggregation |
| Homepage | homepage | Helm | Dashboard -- candidate for removal |
| Kubernetes Dashboard | kubernetes-dashboard | Helm | UI -- candidate for removal |
| Flannel | kube-system | Built-in | CNI -- no NetworkPolicy enforcement |

**Known CNI gap:** Flannel does not enforce NetworkPolicy objects. Cilium migration planned post-CKA. See [[ADR-00 Decision Log]] D-02.

---

## Code & Config Locations

```
Cluster manifests:    github.com/omaratabany/Home-Lab-Infra-as-code
Mac kubeconfig:       ~/Kuber/kubeconfig
Mac talosconfig:      ~/Kuber/talos-init/talosconfig
ArgoCD source:        GitHub repo above (Forgejo takes over in PH-01)
```
