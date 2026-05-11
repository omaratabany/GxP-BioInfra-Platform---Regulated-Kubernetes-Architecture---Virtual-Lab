# ADR-01 Alternative Configurations

> Part of [[README]] | See also: [[ADR-00 Decision Log]], [[INF-04 System Tier Comparison]]

Secondary options for every component in my stack. Each entry references the decision that selected the primary choice and shows the scenarios where the alternative would be the better pick.

---

## A-01 -- Cluster OS

**My choice:** Talos OS -- see [[ADR-00 Decision Log]] D-01

| Alternative | Pros | Cons | When to use instead |
|---|---|---|---|
| Ubuntu 22.04 + kubeadm | Standard enterprise approach, huge docs base, full shell access | SSH + shell = attack surface, mutable OS | First K8s lab where troubleshooting ease matters more than security posture |
| Proxmox + VMs + K3s | VM-level snapshots, VLAN isolation per VM | VMs add management layer, K3s strips some admission controller features | Team comfortable with hypervisor management wanting VM-level snapshot/restore |
| RKE2 | Government-hardened, FIPS mode, CIS benchmark compliant | Heavier than Talos, complex upgrade path | Actual US government or DoD environments where FIPS is required |
| MicroK8s | Single-node simplicity, snap-based | Snap conflicts with regulated environments, not production-grade multi-node | Single-node test environment only, not for this use case |

---

## A-02 -- CNI

**My interim choice:** Flannel. **My target:** Cilium. See [[ADR-00 Decision Log]] D-02

| Alternative | NetworkPolicy | Observability | When to use instead |
|---|---|---|---|
| Cilium (my target) | Full enforcement + extended policies | Hubble L3/L4/L7 | Correct for any regulated environment from day one |
| Calico | Full enforcement | Felix metrics | When Cilium's eBPF dependency is a problem on older kernels |
| Canal (Calico policies + Flannel overlay) | Full enforcement | Basic | When replacing Flannel overlay is not feasible but Calico policies are needed |
| Weave Net | Full enforcement | Basic | Legacy -- deprioritised by the community, avoid for new deployments |

---

## A-03 -- GitOps Controller

**My choice:** ArgoCD (already deployed)

| Alternative | UI | Multi-cluster | When to use instead |
|---|---|---|---|
| Flux v2 | No UI (CLI + Grafana) | Yes | CLI-first workflows, tighter Git integration without a UI |
| Rancher Fleet | Yes | Native multi-cluster | Managing many clusters centrally -- over-engineered for a 2-node homelab |

---

## A-04 -- Ingress Controller

**My choice:** Ingress-NGINX (already deployed)

| Alternative | When to use instead |
|---|---|
| Traefik | Simpler setups -- auto-discovers services, native Let's Encrypt, harder to integrate with existing Cert-Manager |
| Contour | Large clusters with many ingress rules -- HTTPProxy CRD is more expressive |
| Istio Gateway | Only if also running Istio for mTLS -- adds 3GB+ RAM overhead, not justified here |

---

## A-05 -- Secret Management

**My choice:** Sealed Secrets -- see [[ADR-00 Decision Log]] D-12

| Alternative | External Dependency | Rotation | When to use instead |
|---|---|---|---|
| External Secrets Operator + HashiCorp Vault | Yes -- requires Vault | Yes -- automatic | Production: Vault provides dynamic secrets and rotation |
| External Secrets Operator + AWS Secrets Manager | Yes -- requires AWS | Yes | AWS-native deployments |
| SOPS + Age | No (key management only) | Manual | When you control Age keys and prefer encrypted files in Git |

---

## A-06 -- Self-hosted Git

**My choice:** Forgejo -- see [[ADR-00 Decision Log]] D-06

| Alternative | RAM Footprint | When to use instead |
|---|---|---|
| Gitea | ~200-400MB | Use Forgejo instead -- identical capability, no single-entity license risk |
| GitLab CE | 4GB+ minimum | When built-in CI/CD runners and a container registry are also needed |
| Gogs | ~100MB | Severely RAM-constrained environments -- loses some ArgoCD webhook features |

---

## A-07 -- Identity Provider

**My choice:** Authentik -- see [[ADR-00 Decision Log]] D-07

| Alternative | RAM Footprint | When to use instead |
|---|---|---|
| Keycloak | 1.5-2GB | Enterprise deployments with existing LDAP or Active Directory integration |
| Dex | ~50MB | When an upstream identity source already exists and only an OIDC adapter is needed |
| Zitadel | ~500MB | Competitive with Authentik -- better native audit events, less mature community |
| LLDAP + Dex | ~100MB combined | When Authentik's footprint is too high for available hardware |

---

## A-08 -- Object Storage

**My choice:** MinIO standalone -- see [[ADR-00 Decision Log]] D-08

| Alternative | Min Nodes | S3 Compatible | Replication | When to use instead |
|---|---|---|---|---|
| MinIO distributed | 4 nodes minimum | Yes | Yes -- erasure coding | Production with 4+ storage nodes |
| Rook-Ceph | 3 nodes minimum | Via RADOS Gateway | Yes -- factor 3 | Production clusters with 3+ storage nodes, need block + object + filesystem |
| Longhorn | 1 node minimum | No -- block only (RWO) | Yes -- configurable | When block storage PVCs are needed but S3 is not required |
| SeaweedFS | 1 node minimum | Yes | Yes | MinIO alternative with lower RAM footprint -- less mature ecosystem |

---

## A-09 -- Policy Enforcement

**My choice:** OPA Gatekeeper -- see [[ADR-00 Decision Log]] D-09

| Alternative | Language | Mutation | When to use instead |
|---|---|---|---|
| Kyverno | YAML native | Yes -- powerful | Teams where Rego is a barrier -- equivalent enforcement capability |
| Kubewarden | WASM modules | Yes | Policies compiled to WebAssembly -- good for Go or Rust policy authors |
| Polaris | YAML | No | Audit-only use cases where a compliance score dashboard is more useful than enforcement |

---

## A-10 -- Runtime Security

**My choice:** Falco -- see [[ADR-00 Decision Log]] D-10

| Alternative | When to use instead |
|---|---|
| Tetragon (Cilium) | Post-Cilium migration -- deeper eBPF visibility, lower overhead than Falco |
| Sysdig (commercial) | When budget is available and managed rules + threat intelligence are needed |
| KubeArmor | When policy-based enforcement (blocking syscalls) is needed in addition to detection |

---

## A-11 -- Workflow Engine

**My choice:** Nextflow -- see [[ADR-00 Decision Log]] D-11

| Alternative | Language | nf-core Compatible | When to use instead |
|---|---|---|---|
| Cromwell (Broad Institute) | WDL | No | Pipelines written in WDL or integrating with the Terra/Google ecosystem |
| Snakemake | Python rules | No | Academic/research environments where Python familiarity outweighs other factors |
| Argo Workflows | YAML DAGs | No | K8s-native workflow engine with visual DAG UI needed without an external executor |
| Airflow | Python DAGs | No | Data engineering pipelines -- different problem domain |

---

## A-12 -- Monitoring Stack

**My choice:** kube-prometheus-stack + Loki + Grafana (already deployed)

| Alternative | When to use instead |
|---|---|
| VictoriaMetrics + Grafana | Prometheus cardinality or memory consumption is a problem at scale |
| Datadog | Budget available and a managed SaaS observability platform is preferable |
| Elastic Stack (ELK) | Not appropriate for this hardware -- Elasticsearch needs 4GB+ RAM minimum |
| OpenTelemetry + Tempo | When distributed tracing is required -- not a current need for this use case |
