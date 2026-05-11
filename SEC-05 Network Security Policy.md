# SEC-05 Network Security Policy

> Part of [[README]] | See also: [[SEC-01 Security Architecture]], [[INF-02 Architecture and Components]], [[ADR-00 Decision Log]] D-02

Network security model for the platform -- current state with Flannel, target state with Cilium, allowed traffic flows, and the zone model I am working toward.

---

## Network Zone Model

The platform is divided into logical security zones. Physical enforcement is partial today (Flannel cannot enforce it). Full enforcement arrives with Cilium.

```
Zone 1: External
  Cloudflare Tunnel endpoint (only ingress path)
  No direct external IP (CGNAT)

Zone 2: Ingress (192.168.0.200)
  MetalLB L2 VIP
  Ingress-NGINX (terminates TLS)

Zone 3: Platform Services
  Namespace: argocd, cert-manager, sealed-secrets, monitoring
  Contains: ArgoCD, Prometheus, Grafana, Loki, Falco

Zone 4: Identity
  Namespace: authentik
  Contains: Authentik server, worker, PostgreSQL, Redis
  Restriction: should only receive traffic from Zone 2 (Ingress) and Zone 3 (monitoring scrape)

Zone 5: Storage
  Namespace: minio
  Node: Beelink (node-role=storage)
  Contains: MinIO object storage
  Restriction: should only receive traffic from Zone 3 (Loki), Zone 6 (pipelines), Zone 7 (Forgejo PVC -- separate)

Zone 6: Pipeline Execution
  Namespace: pipelines
  Contains: Nextflow runner pods, nf-core job pods
  Restriction: should only egress to MinIO (S3 API). No ingress. No access to Zone 3 or Zone 4.

Zone 7: Source Control
  Namespace: forgejo
  Contains: Forgejo Git server
  Restriction: ingress from Zone 2 only. Egress to Zone 3 (ArgoCD webhook) only.
```

---

## Current State -- Flannel (Partial Enforcement)

With Flannel, NetworkPolicy objects exist in the cluster but are **not enforced**. All pods can reach all other pods on any port. The zone model above is aspirational -- enforced only at the RBAC and application layer, not the network layer.

**What IS enforced today:**
- Talos firewall: nodes only expose ports they explicitly need (6443 for API server, 50000 for Talos API, 2380/2379 for etcd internally)
- Ingress-NGINX routing: external traffic only reaches services with an Ingress object
- MinIO IAM: even if a pod can reach MinIO's port, it cannot access buckets without valid credentials
- RBAC: even if a pod can reach the K8s API server, it cannot perform actions its ServiceAccount is not authorised for

**What is NOT enforced today:**
- Pod-to-pod direct communication is unrestricted
- A pipeline pod could attempt to connect directly to Authentik's PostgreSQL port
- A compromised pod could scan the internal network

This gap is documented in [[REG-03 Risk Register]] as Risk R-04.

---

## Target State -- Cilium (Full Enforcement)

After Cilium migration, these NetworkPolicy objects will be applied and enforced.

### Pipeline Namespace -- Strict Egress Control

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pipelines-egress-policy
  namespace: pipelines
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []   # No ingress to pipeline pods from within the cluster
  egress:
    # Allow DNS resolution
    - ports:
        - protocol: UDP
          port: 53
    # Allow MinIO S3 API only
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: minio
      ports:
        - protocol: TCP
          port: 9000
    # Allow K8s API server (Nextflow needs to create/delete pods)
    - to:
        - ipBlock:
            cidr: 192.168.0.134/32   # Omen node IP (API server)
      ports:
        - protocol: TCP
          port: 6443
```

### Authentik Namespace -- Identity Traffic Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: authentik-ingress-policy
  namespace: authentik
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from Ingress-NGINX only (user logins)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 9000
    # Allow from ArgoCD (OIDC token validation)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
    # Allow from Prometheus (metrics scrape)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9300
```

### MinIO Namespace -- Storage Access Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: minio-ingress-policy
  namespace: minio
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from pipelines namespace (Nextflow data access)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: pipelines
      ports:
        - protocol: TCP
          port: 9000
    # Allow from monitoring namespace (Loki log writes)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9000
    # Allow from Ingress-NGINX (MinIO console access)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 9001
```

---

## Allowed Traffic Matrix

This is the intended state after Cilium migration. Currently aspirational.

| Source | Destination | Port | Protocol | Allowed | Enforced Today |
|---|---|---|---|---|---|
| Internet | Ingress-NGINX | 443 | HTTPS | Yes | Yes (CGNAT + Cloudflare Tunnel) |
| Ingress-NGINX | Forgejo | 3000 | HTTP | Yes | Routing only |
| Ingress-NGINX | Authentik | 9000 | HTTP | Yes | Routing only |
| Ingress-NGINX | ArgoCD | 8080 | HTTP | Yes | Routing only |
| Ingress-NGINX | Grafana | 3000 | HTTP | Yes | Routing only |
| Ingress-NGINX | MinIO Console | 9001 | HTTP | Yes | Routing only |
| Pipelines | MinIO | 9000 | S3/HTTP | Yes | IAM only (no NetworkPolicy) |
| Pipelines | K8s API | 6443 | HTTPS | Yes | RBAC only |
| Pipelines | Any other pod | Any | Any | No | Not enforced until Cilium |
| Authentik | Forgejo | Any | Any | No | Not enforced until Cilium |
| ArgoCD | Forgejo | 3000 | HTTP | Yes | Routing only |
| Monitoring (Loki) | MinIO | 9000 | S3/HTTP | Yes | IAM only |
| Monitoring (Prometheus) | All pods | 9090/metrics | HTTP | Yes | Labels only |
| Forgejo | ArgoCD webhook | 8080 | HTTP | Yes | Service discovery |

---

## Port Exposure Reference

What is actually exposed at the cluster boundary (MetalLB IP 192.168.0.200):

| Port | Protocol | Service | Notes |
|---|---|---|---|
| 80 | HTTP | Ingress-NGINX | Redirects to 443 |
| 443 | HTTPS | Ingress-NGINX | All user-facing services |
| 2222 | TCP | Forgejo SSH | Git clone over SSH |

Everything else is ClusterIP only. The Talos API (port 50000) and K8s API (port 6443) are accessible from the LAN directly on the node IPs -- not through MetalLB. This is intentional: cluster management is LAN-only, never through the public ingress path.

---

## Cilium Migration Pre-flight Checklist

Before migrating, verify all these are in place:

- [ ] All NetworkPolicy YAML files are written and reviewed (see above)
- [ ] Namespace labels are set: `kubectl label namespace pipelines kubernetes.io/metadata.name=pipelines`
- [ ] All namespace labels set for the policy selectors to work
- [ ] Falco rules for unexpected connections have been tested and are generating alerts
- [ ] MinIO IAM policies are in place (application-layer fallback if NetworkPolicy is misconfigured)
- [ ] An etcd snapshot has been taken immediately before the migration
- [ ] The migration is documented as a Forgejo PR (change control SOP)
