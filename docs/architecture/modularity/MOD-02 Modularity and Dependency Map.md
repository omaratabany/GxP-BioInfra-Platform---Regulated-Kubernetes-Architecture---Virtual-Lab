# MOD-02 Modularity and Dependency Map

> Part of [[README]] | See also: [[MOD-01 Component Interface Specifications]], [[MOD-03 Configuration Management Standard]], [[INF-05 Hardware Scaling Guide]]

Which components depend on which, what the minimum viable subset looks like for different use cases, and how to add or remove components without breaking the platform.

---

## Dependency Graph

```
[Talos OS] (everything runs on this -- foundational, cannot be swapped without cluster rebuild)
    |
  [Kubernetes API server + etcd + scheduler]
    |
    [MetalLB] ---- provides IP 192.168.0.200
    |
    [Ingress-NGINX] ---- routes external traffic (depends on MetalLB)
    |
    [Cert-Manager] ---- provides TLS (depends on Ingress-NGINX for HTTP-01 challenges)
    |
    [Sealed Secrets] ---- decrypts secrets (depends on nothing -- standalone)
    |
    [ArgoCD] ---- watches Git, syncs cluster (depends on: Sealed Secrets for repo creds)
    |
    [Forgejo PH-01] ---- Git server (depends on: ArgoCD, local-hdd PVC)
    |
    [Authentik PH-02] ---- SSO (depends on: ArgoCD, Ingress-NGINX, Cert-Manager, Forgejo for OIDC config)
    |
    [MinIO PH-03] ---- Object storage (depends on: ArgoCD, local-hdd PVC, Sealed Secrets for creds)
    |
    [Loki-MinIO migration] ---- Loki backend (depends on: MinIO)
    |
    [OPA Gatekeeper PH-04] ---- Admission control (depends on: ArgoCD; enforces rules on everything after)
    |
      [Falco PH-05] ---- Runtime security (depends on: ArgoCD, MinIO-backed Loki for event persistence)
    |
    [Nextflow PH-06] ---- Pipeline orchestration (depends on: MinIO, Gatekeeper, Falco, Forgejo)
```

---

## Minimum Viable Subsets

Different use cases require different subsets of components. The table below shows which components are required for each scenario.

| Component | Core K8s Lab | GxP Compliant (min) | Full Platform | Notes |
|---|---|---|---|---|
| Talos OS | Required | Required | Required | Foundational |
| MetalLB | Required | Required | Required | Needed for any LAN-reachable service |
| Ingress-NGINX | Required | Required | Required | All services need routing |
| Cert-Manager | Recommended | Required | Required | GxP requires encrypted communication |
| Sealed Secrets | Optional | Required | Required | Required for GitOps secret management |
| ArgoCD | Optional | Required | Required | GitOps is required for Annex 11 change control |
| local-path-provisioner | Required | Required | Required | All stateful services need storage |
| Loki + Promtail | Optional | Required | Required | GxP requires log aggregation |
| Prometheus + Grafana | Recommended | Required | Required | Required for PQ evidence |
| Forgejo | Skip (use GitHub) | Required | Required | Required for change control SOP |
| Authentik | Skip (basic auth) | Required | Required | Required for Annex 11 access control |
| MinIO | Skip | Required | Required | Required for audit trail persistence and pipeline data |
| OPA Gatekeeper | Skip | Required | Required | Required for Annex 11 policy enforcement |
| Falco | Skip | Required | Required | Required for Annex 11 audit trail |
| Nextflow | Skip | Optional | Required | Required only if running pipelines |
| Cilium (post-CKA) | Skip | Planned | Required | Required for full NetworkPolicy enforcement |

---

## Component Removal Procedures

### Removing Homepage and Kubernetes Dashboard

These were in the original stack and are candidates for removal to free RAM. They have no dependencies from other components.

```bash
# Via ArgoCD
argocd app delete homepage
argocd app delete kubernetes-dashboard

# Or scale to zero to preserve the app definition
kubectl scale deployment homepage -n homepage --replicas=0
kubectl scale deployment kubernetes-dashboard -n kubernetes-dashboard --replicas=0
```

RAM recovered: ~200-300Mi. No impact on any other component.

### Removing a Phase Component

If a component needs to be removed (e.g., replacing Authentik with Keycloak):

1. Ensure the replacement is deployed and tested first (see [[ADR-01 Alternative Configurations]])
2. Update all OIDC client configurations to point at the new provider
3. Validate logins for all integrated apps
4. Delete the old application via ArgoCD
5. Remove the old manifests from the Forgejo repo via PR
6. Update the IQ document to reflect the change
7. Re-run the relevant OQ test cases (OQ-04, OQ-05 for identity changes)

---

## Adding New Components

Any new component must satisfy these requirements before being added to the platform:

### Pre-addition Checklist

- [ ] Decision recorded in [[ADR-00 Decision Log]] with reasoning
- [ ] Alternative options documented in [[ADR-01 Alternative Configurations]]
- [ ] Interface contract documented in [[MOD-01 Component Interface Specifications]] (if the component exposes or consumes a shared interface)
- [ ] Risk assessment added to [[REG-03 Risk Register]]
- [ ] Compliance mapping added to [[REG-01 Compliance Matrix]] (if the component contributes to GxP controls)
- [ ] Hardening steps documented in [[SEC-02 Component Hardening Guide]]
- [ ] Resource requests and limits defined
- [ ] SealedSecret created for any credentials
- [ ] nodeSelector applied (storage workloads to Beelink, compute to Omen)
- [ ] Helm values pinned to a specific chart version (no `latest`)
- [ ] Image pinned to a digest (not just a tag)
- [ ] ArgoCD application manifest created
- [ ] Forgejo PR opened for all of the above
- [ ] IQ entry added for the new component
- [ ] OQ test case written for the new component's core function

---

## Upgrade Procedures

### Upgrading a Helm Chart

```bash
# 1. Check what the current version is
helm list -n <namespace>

# 2. Check what the latest version is
helm repo update
helm search repo <chart-name> --versions | head -5

# 3. Diff the changes (requires helm-diff plugin)
helm diff upgrade <release-name> <repo>/<chart> \
  --version <new-version> \
  --namespace <namespace> \
  -f apps/<app>/helm-values.yaml

# 4. If the diff is acceptable, update the chart version in helm-release.yaml
# Open a PR in Forgejo with the version bump
# PR description must include: chart changelog link, diff summary, test plan

# 5. ArgoCD syncs after PR merge
# 6. Verify the upgrade succeeded
kubectl rollout status deployment/<name> -n <namespace>

# 7. Re-run relevant OQ test cases
# 8. Update the IQ document with the new chart version
```

### Upgrading Talos OS

```bash
# Talos upgrades are rolling -- one node at a time
# Take an etcd snapshot first
talosctl etcd snapshot ~/Kuber/snapshots/pre-upgrade-$(date +%Y%m%d).snapshot \
  --nodes 192.168.0.134 --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Upgrade Omen (control plane -- drains workloads automatically)
talosctl upgrade \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --image ghcr.io/siderolabs/installer:<new-version>

# Wait for Omen to come back and cluster to stabilise
kubectl get nodes -w

# Upgrade Beelink (worker)
talosctl upgrade \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --image ghcr.io/siderolabs/installer:<new-version>

# Verify
kubectl get nodes
talosctl -n 192.168.0.134 version
talosctl -n 192.168.0.202 version
```

---

## Platform Version Matrix

Track the versions of all major components here. Updated in [[OPS-03 Implementation Log]] as upgrades occur.

| Component | Current Version | Chart Version | Last Updated | Next Review |
|---|---|---|---|---|
| Talos OS | v1.12.6 | N/A | May 2026 | At each minor release |
| Kubernetes | v1.35.2 | N/A | May 2026 | With Talos upgrade |
| ArgoCD | TBD | TBD | - | Quarterly |
| Ingress-NGINX | TBD | TBD | - | Quarterly |
| Cert-Manager | TBD | TBD | - | Quarterly |
| Sealed Secrets | TBD | TBD | - | Quarterly |
| kube-prometheus-stack | TBD | TBD | - | Quarterly |
| Loki | TBD | TBD | - | Quarterly |
| Forgejo | TBD | TBD | - | Post PH-01 |
| Authentik | TBD | TBD | - | Post PH-02 |
| MinIO | TBD | TBD | - | Post PH-03 |
| OPA Gatekeeper | TBD | TBD | - | Post PH-04 |
| Falco | TBD | TBD | - | Post PH-05 |
| Nextflow | TBD | TBD | - | Post PH-06 |

---

## G-22 -- Version Matrix Pre-Population

The table above shows TBD for all components that are not yet deployed. Below is the pre-population of known values from INF-01 (already deployed stack). Fill in chart versions by running `helm list -A` once all phases are complete.

| Component | Current Version | Chart Version | Status |
|---|---|---|---|
| Talos OS | v1.12.6 | N/A | Deployed -- PH-00 |
| Kubernetes | v1.35.2 | N/A | Deployed -- PH-00 |
| local-path-provisioner | TBD | TBD | Deploying -- PH-00 |
| MetalLB | TBD | TBD | Deployed (pre-project) |
| Ingress-NGINX | TBD | TBD | Deployed (pre-project) |
| Cert-Manager | TBD | TBD | Deployed (pre-project) |
| Sealed Secrets | TBD | TBD | Deployed (pre-project) |
| ArgoCD | TBD | TBD | Deployed (pre-project) |
| kube-prometheus-stack | TBD | TBD | Deployed (pre-project) |
| Loki + Promtail | TBD | TBD | Deployed (pre-project) |
| Forgejo | TBD | TBD | PH-01 -- not started |
| Authentik | TBD | TBD | PH-02 -- not started |
| MinIO | TBD | TBD | PH-03 -- not started |
| OPA Gatekeeper | TBD | TBD | PH-04 -- not started |
| Falco + Falcosidekick | TBD | TBD | PH-05 -- not started |
| Nextflow | TBD | TBD | PH-06 -- not started |
| nf-core/rnaseq | TBD | N/A | PH-06 -- not started |

Run this to capture chart versions for the already-deployed stack:

```bash
helm list -A -o json | jq -r '.[] | [.name, .chart, .app_version, .namespace] | @tsv' | sort
```

Update the table above after each phase completes.
