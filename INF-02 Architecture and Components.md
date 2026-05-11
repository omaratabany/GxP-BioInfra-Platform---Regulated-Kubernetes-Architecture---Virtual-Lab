# INF-02 Architecture and Components

> Part of [[README]] | See also: [[INF-01 Infrastructure Baseline]], [[PRJ-02 Phase Map and Schedule]]

---

## What I Built

A GxP-compliant Kubernetes platform for running nf-core genomics pipelines with a full audit trail, policy enforcement, and validation documentation aligned to EU GMP Annex 11 and FDA 21 CFR Part 11. Targeting DevOps and Platform Engineering roles at pharmaceutical and biomedical companies in Basel and Zurich. Full reasoning in [[PRJ-01 Strategic Rationale]].

---

## Final State Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Talos K8s Cluster                     │
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │   HP Omen (CP)       │  │   Beelink (Worker)       │ │
│  │                      │  │                          │ │
│  │  ArgoCD              │  │  MinIO (S3 storage)      │ │
│  │  Forgejo             │  │  Loki data               │ │
│  │  Authentik (SSO)     │  │  Prometheus TSDB         │ │
│  │  Gatekeeper (OPA)    │  │  Forgejo PVC             │ │
│  │  Falco               │  │   320GB HDD mount        │ │
│  │  Nextflow pods       │  │                          │ │
│  │  nf-core pipelines   │  │                          │ │
│  └──────────────────────┘  └──────────────────────────┘ │
│                                                         │
│  Shared: Ingress-NGINX, Cert-Manager, MetalLB, Flannel  │
│  (Cilium migration planned post-CKA)                    │
└─────────────────────────────────────────────────────────┘
         │
         │ kubectl / talosctl
         │
   MacBook Air M5
```

---

## Component Registry

| Component | Purpose | Phase |
|---|---|---|
| local-path-provisioner | Dynamic PV provisioning from Beelink HDD | [[PH-00 Cluster Preparation]] |
| Node labels + taints | Workload placement control | [[PH-00 Cluster Preparation]] |
| Forgejo | Self-hosted Git -- SCM for all project code | [[PH-01 Forgejo]] |
| Authentik | OIDC/SSO provider -- unified identity | [[PH-02 Authentik SSO]] |
| MinIO | S3-compatible object storage -- pipeline data | [[PH-03 MinIO Object Storage]] |
| OPA Gatekeeper | Policy-as-code -- GxP enforcement layer | [[PH-04 OPA Gatekeeper]] |
| Falco | Runtime security -- audit event generation | [[PH-05 Falco Runtime Security]] |
| Nextflow + K8s executor | Pipeline orchestration | [[PH-06 Nextflow and nf-core]] |
| nf-core/rnaseq | Reference pipeline -- demo + validation run | [[PH-06 Nextflow and nf-core]] |
| IQ/OQ/PQ docs | GxP validation documentation | [[PH-07 GxP Validation Documentation]] |

---

## Data Flow

```
I push code
  Forgejo (PH-01)
    ArgoCD webhook triggered
      ArgoCD syncs manifests to cluster
        Gatekeeper validates admission (PH-04)
          Pods scheduled, RBAC enforced (PH-06)
            Pipeline runs, output to MinIO (PH-03)
              Falco captures runtime events (PH-05)
                Events ship to Loki
                  Grafana dashboard
```

---

## Namespace Plan

| Namespace | Contents |
|---|---|
| argocd | ArgoCD controller |
| monitoring | Prometheus, Grafana, Loki, Falco sidekick |
| forgejo | Forgejo Git server |
| authentik | Authentik SSO |
| minio | MinIO object storage |
| gatekeeper-system | OPA Gatekeeper |
| pipelines | Nextflow executor pods, nf-core jobs |

---

## Identity and Access Flow (Post-PH-02)

```
I authenticate via Authentik (OIDC)
  Token issued
    Forgejo (OIDC client)
    ArgoCD (OIDC client)
    Grafana (OIDC client)
    K8s RBAC groups synced from Authentik groups
```

Groups: `platform-admin`, `developer`, `readonly`. Full OIDC config in [[PH-02 Authentik SSO]].

---

## Node Placement Rules

| Workload Type | Target Node | Mechanism |
|---|---|---|
| Storage services (MinIO, Loki data, Forgejo PVC) | Beelink | `nodeSelector: node-role: storage` |
| Compute workloads (Nextflow workers, pipeline pods) | Omen | `nodeSelector: node-role: infra` |
| DaemonSets (Falco, Promtail) | Both nodes | No selector -- DaemonSet runs everywhere |
| Platform services (ArgoCD, Authentik, Gatekeeper) | Omen | Default scheduling -- Omen has the RAM headroom |
