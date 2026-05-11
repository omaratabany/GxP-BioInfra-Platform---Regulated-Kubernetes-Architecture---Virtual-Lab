# MOD-03 Configuration Management Standard

> Part of [[README]] | See also: [[MOD-01 Component Interface Specifications]], [[MOD-02 Modularity and Dependency Map]], [[ADR-00 Decision Log]]

How configuration is structured, validated, versioned, and promoted across this platform. This standard ensures that every configuration change is traceable, reversible, and consistent regardless of which component is being changed.

---

## Configuration Philosophy

Everything is configuration. There is no manually-applied setting that exists only on a running cluster. If it is not in Git, it does not exist. This is enforced by ArgoCD's drift detection -- any manual change to the cluster is detected and reverted on the next sync.

The three sources of configuration truth:
1. **Talos machineconfig YAML patches** -- machine-level configuration (disk mounts, kernel parameters)
2. **Helm values files** -- application configuration for Helm-managed releases
3. **Raw Kubernetes manifests** -- RBAC, namespace definitions, StorageClass, NetworkPolicy, ResourceQuota

All three live in the Forgejo repo, watched by ArgoCD.

---

## Repository Structure Standard

```
Home-Lab-Infra-as-code/
  apps/
    <app-name>/
      namespace.yaml          -- Namespace definition
      argocd-application.yaml -- ArgoCD Application manifest
      helm-release.yaml       -- HelmRelease (chart name, version, values)
      sealed-secret.yaml      -- SealedSecret for credentials
      ingress.yaml            -- Ingress object (if needed)
      pvc.yaml                -- PVC definitions (if needed)
      policies/               -- MinIO IAM policies, Gatekeeper Rego, etc.
        <policy-name>.json
  patches/
    <node-name>-<purpose>.yaml -- Talos machineconfig patches
  validation-docs/             -- IQ/OQ/PQ documents
    IQ-Installation-Qualification.md
    OQ-Operational-Qualification.md
    PQ-Performance-Qualification.md
    SOPs/
      change-control.md
      audit-trail.md
      backup-recovery.md
      access-control.md
```

---

## Helm Values File Standard

Every Helm release is configured through a `helm-release.yaml` file. This is not an ArgoCD HelmRelease CRD -- it is a plain Helm values file committed to the repo, referenced by the ArgoCD Application manifest.

**Required fields in every helm-release.yaml:**

```yaml
# META BLOCK -- required at the top of every helm-release.yaml
# chart: <repo>/<chart-name>
# version: <exact-version> -- never use "latest" or a range
# last-reviewed: <date>
# reviewed-by: <operator>

# RESOURCE REQUESTS AND LIMITS -- required for every container
resources:
  requests:
    cpu: <value>
    memory: <value>
  limits:
    cpu: <value>
    memory: <value>

# NODE SELECTOR -- required for all stateful workloads
nodeSelector:
  node-role: storage   # or infra

# IMAGE TAG -- must be pinned to a specific version, not latest
image:
  tag: "<version>"
  # When Gatekeeper require-image-digest is in deny mode, also pin:
  digest: "sha256:<hash>"
```

---

## Configuration Validation Before Commit

Before any Helm values change is committed to Forgejo:

```bash
# 1. Lint the Helm chart with the new values
helm lint <repo>/<chart> --version <version> -f apps/<app>/helm-release.yaml

# 2. Template the chart to verify output is valid YAML
helm template <release-name> <repo>/<chart> \
  --version <version> \
  --namespace <namespace> \
  -f apps/<app>/helm-release.yaml | kubectl apply --dry-run=client -f -

# 3. Diff against the running release (requires helm-diff plugin)
helm diff upgrade <release-name> <repo>/<chart> \
  --version <version> \
  --namespace <namespace> \
  -f apps/<app>/helm-release.yaml

# 4. If the diff touches anything security-relevant (RBAC, ingress, secrets),
#    require explicit sign-off in the PR before merging
```

---

## ArgoCD Application Manifest Standard

Every application deployed to the cluster has an ArgoCD Application manifest. This is the single object that tells ArgoCD where the source is and where to deploy it.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  labels:
    app.kubernetes.io/name: <app-name>
    app.kubernetes.io/component: <category>  # platform, storage, security, pipeline
    app.kubernetes.io/managed-by: argocd
spec:
  project: default
  source:
    repoURL: https://forgejo.homelab/<org>/Home-Lab-Infra-as-code.git
    targetRevision: main
    path: apps/<app-name>
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true      # Remove resources not in Git
      selfHeal: true   # Revert manual changes to match Git
    syncOptions:
      - CreateNamespace=true
```

`prune: true` and `selfHeal: true` together enforce GitOps -- no manual cluster change survives a sync.

---

## Talos Machineconfig Patch Standard

Talos patches are applied with `talosctl patch machineconfig`. Every patch:
- Has a descriptive filename: `<node-name>-<purpose>.yaml`
- Contains only the specific change being made (not a full machineconfig)
- Is committed to `patches/` in the repo before being applied
- Has a corresponding commit message explaining why the patch was needed

```yaml
# patches/beelink-hdd.yaml
# Purpose: Mount 320GB HDD at /var/mnt/hdd for persistent storage
# Node: talos-v3h-4m1 (Beelink)
# Applied: <date>
# Reference: PH-00 Cluster Preparation

machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/hdd
  kubelet:
    extraMounts:
      - destination: /var/mnt/hdd
        type: bind
        source: /var/mnt/hdd
        options:
          - bind
          - rshared
          - rw
```

---

## Configuration Drift Detection

ArgoCD detects and reports when the running cluster state differs from what is defined in Git. Two types of drift:

**Type 1 -- Unintended drift** (someone applied a change directly to the cluster without a PR)
- ArgoCD marks the application as OutOfSync
- With `selfHeal: true`, ArgoCD automatically reverts the change on the next sync cycle (every 3 minutes by default)
- The drift is logged in ArgoCD's operation history -- reviewable in the UI or via `argocd app history <app>`

**Type 2 -- Expected drift** (a change was made intentionally but not yet committed to Git)
- This should never happen after PH-01 -- all changes go through Forgejo PRs
- If it does happen, it constitutes a Change Control SOP violation and must be documented as an incident in [[OPS-03 Implementation Log]]

```bash
# Check for drift across all applications
argocd app list | grep OutOfSync

# Get the specific diff for a drifted application
argocd app diff <app-name>
```

---

## Environment Promotion (Future State)

Currently I run a single environment (homelab). In a production context, configurations promote through environments:

```
Development (dev) --> Staging (stg) --> Production (prd)
```

Each environment has its own:
- Namespace suffix (e.g., `forgejo-dev`, `forgejo-stg`, `forgejo-prd`)
- ArgoCD Application (pointing at the same repo, different path or branch)
- Helm values overlay (resource limits differ per environment)
- SealedSecrets (different credentials per environment)

Promotion is a PR from the dev branch to the stg branch, reviewed, then from stg to main (prd). ArgoCD watches each branch.

For current homelab: this is noted as the correct model but only one environment exists. The groundwork is laid by having all configuration in Helm values files that can be overridden per environment.

---

## Configuration Change Authorisation Matrix

| Change Type | Who Can Approve | Review Required | ArgoCD Sync |
|---|---|---|---|
| Helm values update | Operator | Self-review minimum | Auto on merge |
| New application added | Operator | Self-review + ADR entry | Auto on merge |
| Secret rotation | Operator | Implicit (follows SEC-04 procedure) | Auto on merge |
| Talos machineconfig patch | Operator | Self-review | Manual (`talosctl patch`) |
| Gatekeeper constraint change | Operator | Self-review + 48hr monitor period | Auto on merge |
| Cilium NetworkPolicy | Operator | Self-review + pre-flight checklist | Auto on merge |
| etcd configuration | Operator | Requires etcd snapshot first | Manual (`talosctl`) |
