# INF-07 Network Topology

> Part of [[README]] | See also: [[INF-01 Infrastructure Baseline]], [[SEC-05 Network Security Policy]], [[OPS-08 Developer and Operator Onboarding]]

Full network topology documentation for the GxP BioInfra Platform. Required as part of the IQ (Installation Qualification) network topology evidence. Records every network boundary, subnet, VLAN, DNS zone, and port that is relevant to platform operation and security.

---

## Physical Network Overview

```
ISP (DU Telecom, Dubai)
  CGNAT -- no inbound ports forwarded, no public IPv4
  Connection type: Fibre
  IPv4: CGNAT address (shared, not routable)
  IPv6: Available but not used for cluster management

  UniFi Security Gateway / Router
    LAN: 192.168.0.0/24
    Management VLAN: 192.168.0.0/24 (flat -- homelab, no strict VLAN segmentation for cluster nodes)
    WiFi: 192.168.0.0/24 (same broadcast domain as wired)

    Connected devices:
      HP Omen (talos-asj-72z)    192.168.0.134  (static DHCP reservation)
      Beelink (talos-v3h-4m1)    192.168.0.202  (static DHCP reservation)
      MacBook Air M5             DHCP           (management workstation)
      MetalLB VIP                192.168.0.200  (virtual -- MetalLB L2 mode)
      Minisforum MS-01 (Unraid)  DHCP           (OFF LIMITS -- not cluster-connected)
      CachyOS build              192.168.0.169  (study VMs only -- isolated)
```

---

## IP Address Allocation

| Address | Assignment | Notes |
|---|---|---|
| 192.168.0.134 | HP Omen (talos-asj-72z) | Static DHCP reservation. K8s API server (port 6443). Talos API (port 50000). |
| 192.168.0.202 | Beelink (talos-v3h-4m1) | Static DHCP reservation. Worker node. Storage. |
| 192.168.0.200 | MetalLB VIP | L2 mode virtual IP. Assigned to Ingress-NGINX. All platform services reachable here. |
| 192.168.0.169 | CachyOS build | CKA study VMs only. Not used for this project. |
| 192.168.0.0/24 | LAN subnet | Full broadcast domain. All cluster nodes are layer-2 adjacent. |

**MetalLB IP Pool:**
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.200/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool
```

---

## External Access Architecture

```
Public Internet
  |
  Cloudflare CDN / Tunnel endpoint
    |  (encrypted tunnel, outbound from homelab only)
    |
  cloudflared daemon (runs on Omen as a pod or systemd service)
    |
  MetalLB VIP 192.168.0.200
    |
  Ingress-NGINX (TLS termination)
    |
  Internal services (*.homelab)
```

**Key properties:**
- No inbound ports are open on the router. CGNAT blocks all inbound connections anyway.
- Cloudflare Tunnel initiates an outbound connection from cloudflared to Cloudflare's edge. All traffic flows through this tunnel.
- Cloudflare does NOT decrypt HTTPS traffic in this configuration (tunnel proxies to Ingress-NGINX which terminates TLS with our own Cert-Manager cert)
- If Cloudflare Tunnel goes down, all external access is lost. Internal LAN access via 192.168.0.134:6443 (K8s API) and 192.168.0.200 (Ingress) remains fully functional.

---

## Internal DNS Configuration

All platform services resolve via the `*.homelab` local DNS zone. Resolution must work on the Mac for kubectl and browser access, and within the cluster via CoreDNS for pod-to-pod communication.

### Mac DNS Setup

Add all service entries to `/etc/hosts` on the MacBook Air M5:

```
# GxP BioInfra Platform -- homelab services
192.168.0.200  forgejo.homelab
192.168.0.200  auth.homelab
192.168.0.200  argocd.homelab
192.168.0.200  grafana.homelab
192.168.0.200  minio.homelab
192.168.0.200  minio-console.homelab
192.168.0.200  prometheus.homelab
192.168.0.200  alertmanager.homelab
```

All entries point to the MetalLB VIP (192.168.0.200). Ingress-NGINX uses the Host header to route to the correct backend service.

**Alternative (recommended for development):** Use dnsmasq on the Mac to wildcard resolve *.homelab to 192.168.0.200:

```bash
# Install dnsmasq
brew install dnsmasq

# Create config
echo "address=/.homelab/192.168.0.200" >> /usr/local/etc/dnsmasq.conf

# Start dnsmasq
sudo brew services start dnsmasq

# Configure macOS to use dnsmasq for .homelab TLD only
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/homelab
```

With dnsmasq, any new `*.homelab` service automatically resolves without editing /etc/hosts.

### Cluster Internal DNS (CoreDNS)

Pod-to-pod communication uses K8s DNS via CoreDNS. Standard K8s DNS patterns apply:

```
<service-name>.<namespace>.svc.cluster.local
<service-name>.<namespace>.svc
<service-name>.<namespace>
```

Key internal service addresses used in Helm values and configs:

| Service | Internal DNS | Port |
|---|---|---|
| MinIO API | minio.minio.svc | 9000 |
| Loki | loki.monitoring.svc | 3100 |
| Prometheus | prometheus-operated.monitoring.svc | 9090 |
| ArgoCD server | argocd-server.argocd.svc | 80 |
| Authentik | authentik.authentik.svc | 9000 |
| Forgejo | forgejo.forgejo.svc | 3000 |

---

## Ingress Routing Table

All requests reach 192.168.0.200:443 at Ingress-NGINX. The Host header determines routing.

| Hostname | Backend Service | Backend Port | TLS | Notes |
|---|---|---|---|---|
| forgejo.homelab | forgejo.forgejo.svc | 3000 | Cert-Manager | SSH on NodePort 2222 separately |
| auth.homelab | authentik.authentik.svc | 9000 | Cert-Manager | OIDC endpoint for all apps |
| argocd.homelab | argocd-server.argocd.svc | 80 | Cert-Manager | ArgoCD uses grpc -- needs special annotations |
| grafana.homelab | prometheus-grafana.monitoring.svc | 80 | Cert-Manager | |
| minio.homelab | minio.minio.svc | 9000 | Cert-Manager | S3 API endpoint |
| minio-console.homelab | minio.minio.svc | 9001 | Cert-Manager | MinIO web UI |
| prometheus.homelab | prometheus-operated.monitoring.svc | 9090 | Cert-Manager | Internal use only |
| alertmanager.homelab | alertmanager-operated.monitoring.svc | 9093 | Cert-Manager | Internal use only |

---

## Cert-Manager Configuration

All certificates are issued by a self-signed ClusterIssuer. No Let's Encrypt is used because `*.homelab` is not a publicly resolvable domain and cannot complete ACME HTTP-01 or DNS-01 challenges.

```yaml
# apps/cert-manager/cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: homelab-ca-issuer
spec:
  ca:
    secretName: homelab-ca-secret
```

```yaml
# apps/cert-manager/ca-secret.yaml (sealed before committing)
# Contains the CA private key and certificate
# Generated once with:
# openssl req -x509 -nodes -newkey rsa:4096 -keyout ca.key -out ca.crt \
#   -days 3650 -subj "/CN=homelab-ca/O=GxP BioInfra"
```

Each service's Ingress object references the ClusterIssuer:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
spec:
  tls:
  - hosts:
    - forgejo.homelab
    secretName: forgejo-tls
```

**Mac trust setup:** The homelab CA certificate must be trusted in the Mac keychain. See [[OPS-08 Developer and Operator Onboarding]] for the one-time setup command.

---

## Port Reference

### Exposed at MetalLB VIP (192.168.0.200)

| Port | Protocol | Service | Description |
|---|---|---|---|
| 80 | TCP | Ingress-NGINX | HTTP -- redirects to 443 |
| 443 | TCP | Ingress-NGINX | HTTPS -- all web services |
| 2222 | TCP | Forgejo SSH | Git clone over SSH |

### Exposed Directly on Node IPs (LAN only)

| Node | Port | Service | Description |
|---|---|---|---|
| 192.168.0.134 | 6443 | K8s API server | kubectl and ArgoCD cluster access |
| 192.168.0.134 | 50000 | Talos API | talosctl access |
| 192.168.0.202 | 50000 | Talos API | talosctl access |
| 192.168.0.134 | 2379/2380 | etcd | Internal only -- not accessible from LAN |

### Internal Cluster Ports (ClusterIP only -- not accessible from LAN)

| Service | Port | Description |
|---|---|---|
| MinIO S3 API | 9000 | Internal S3 access from pods |
| MinIO Console | 9001 | Ingress-routed |
| Loki | 3100 | Promtail and Falcosidekick push endpoint |
| Prometheus | 9090 | Internal PromQL queries |
| Gatekeeper webhook | 443 | K8s API server calls this on admission |
| Authentik | 9000, 9300 | OIDC and metrics |
| ArgoCD repo server | 8081 | Internal ArgoCD component |
| Falco gRPC | 5060 | Falcosidekick connection |

---

## Network Change Log

All network changes (new Ingress, new MetalLB IP, new DNS entry) must be recorded here and reflected in a Forgejo PR.

| Date | Change | PR Reference | Operator |
|---|---|---|---|
| May 2026 | Initial topology documented | N/A (pre-Forgejo) | Operator |
| | | | |
