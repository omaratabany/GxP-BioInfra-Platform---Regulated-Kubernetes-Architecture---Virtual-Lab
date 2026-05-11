# MOD-04 Kubernetes Resource Standards

> Part of [[README]] | See also: [[MOD-03 Configuration Management Standard]], [[MOD-01 Component Interface Specifications]]

Naming conventions, mandatory labels, annotation standards, and resource tier definitions for all Kubernetes objects deployed on this platform. Consistency across the whole cluster is what makes Prometheus aggregation correct, Gatekeeper's require-labels constraint meaningful, and kubectl output readable.

---

## Naming Conventions

### Namespaces

Pattern: `<app-name>` (single word, lowercase, matches the primary application)

| Namespace | Application | Notes |
|---|---|---|
| `argocd` | ArgoCD | Standard chart default |
| `monitoring` | Prometheus, Grafana, Loki, Falco, Promtail | All observability in one namespace |
| `cert-manager` | Cert-Manager | Standard chart default |
| `sealed-secrets` | Sealed Secrets controller | Standard chart default |
| `metallb-system` | MetalLB | Standard chart default |
| `ingress-nginx` | Ingress-NGINX | Standard chart default |
| `gatekeeper-system` | OPA Gatekeeper | Standard chart default |
| `forgejo` | Forgejo Git server | Matches app name |
| `authentik` | Authentik SSO | Matches app name |
| `minio` | MinIO object storage | Matches app name |
| `local-path-storage` | local-path-provisioner | Standard chart default |
| `pipelines` | Nextflow executor and nf-core jobs | Plural -- reflects that multiple pipelines run here |

**Rule:** Never use a namespace name that differs from the primary application name unless it is an upstream chart default that cannot be changed without breaking the chart.

### Deployments, StatefulSets, DaemonSets

Pattern: `<app-name>` or `<app-name>-<component>` when an application has multiple components

| Good | Bad | Why |
|---|---|---|
| `forgejo` | `gitserver` or `vcs` | Use the actual app name |
| `authentik-server` | `auth-web` | Use the component subname from the Helm chart |
| `authentik-worker` | `authentik-bg` | Same principle |
| `minio` | `object-storage` | Use the app name |

### Services

Pattern: Same as the Deployment or StatefulSet it fronts.

Internal services (ClusterIP) use the same name as the Deployment.
External services (LoadBalancer) are typically created by Helm and keep the chart defaults.

```bash
# Expected service names
kubectl get svc -n forgejo
# NAME      TYPE         CLUSTER-IP     PORT(S)
# forgejo   ClusterIP    10.96.x.x      3000/TCP

kubectl get svc -n minio
# NAME    TYPE         CLUSTER-IP     PORT(S)
# minio   ClusterIP    10.96.x.x      9000/TCP,9001/TCP
```

### PersistentVolumeClaims

Pattern: `<app-name>-<purpose>-pvc` or just `<app-name>` (for single-PVC apps where Helm controls the name)

| Name | What it stores |
|---|---|
| `forgejo` (Helm-managed) | Forgejo repository data |
| `minio` (Helm-managed) | MinIO bucket data on Beelink HDD |
| `prometheus-db` | Prometheus TSDB |
| `pipeline-work-pvc` | Nextflow work directory (transient pipeline data) |

### Secrets and SealedSecrets

Pattern: `<app-name>-<purpose>-secret`

| Secret name | Contains |
|---|---|
| `forgejo-admin-secret` | Forgejo admin username and password |
| `minio-credentials` | MinIO root user and password |
| `minio-nextflow-sa-secret` | nextflow-sa access key and secret key |
| `minio-loki-sa-secret` | loki-sa access key and secret key |
| `authentik-secret-key-secret` | Authentik secretKey, postgresPassword, redisPassword |
| `argocd-oidc-secret` | OIDC client secret for ArgoCD |

### Ingress Objects

Pattern: `<app-name>-ingress`

| Ingress name | Hostname |
|---|---|
| `forgejo-ingress` | forgejo.homelab |
| `authentik-ingress` | auth.homelab |
| `argocd-ingress` | argocd.homelab |
| `grafana-ingress` | grafana.homelab |
| `minio-api-ingress` | minio.homelab |
| `minio-console-ingress` | minio-console.homelab |

### ConfigMaps

Pattern: `<app-name>-<purpose>-config` for custom ConfigMaps. Helm-managed ConfigMaps keep their chart-default names.

| ConfigMap | Contains |
|---|---|
| `falco-custom-rules` | GxP audit rules for Falco |
| `nextflow-config` | Nextflow K8s executor config |
| `minio-policies` | MinIO IAM policy definitions (if stored as ConfigMap) |

---

## Mandatory Labels

These labels must exist on every Pod, Deployment, StatefulSet, DaemonSet, and Job deployed on this platform. Gatekeeper's `require-labels` constraint enforces this.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: <app-name>         # The application name (e.g., forgejo, minio)
    app.kubernetes.io/version: "<version>"     # Quoted string -- app version, not chart version
    app.kubernetes.io/component: <component>   # See component values below
    app.kubernetes.io/managed-by: argocd       # Always argocd for this platform
    env: <environment>                         # dev (this is a homelab -- always dev)
```

**Allowed values for `app.kubernetes.io/component`:**

| Value | Used for |
|---|---|
| `platform` | ArgoCD, Cert-Manager, Sealed Secrets, MetalLB, Ingress-NGINX |
| `observability` | Prometheus, Grafana, Loki, Promtail, Alertmanager |
| `security` | Falco, Falcosidekick, Gatekeeper |
| `identity` | Authentik, Authentik worker, Authentik PostgreSQL |
| `storage` | MinIO |
| `scm` | Forgejo |
| `pipeline` | Nextflow runner, nf-core job pods |

**Gatekeeper require-labels constraint targets these exact keys:**
```rego
# The constraint checks that every pod has all four required labels
required_labels = {
  "app.kubernetes.io/name",
  "app.kubernetes.io/version",
  "app.kubernetes.io/component",
  "env"
}
```

---

## Recommended Annotations

Not enforced by Gatekeeper but expected on all Ingress objects and long-running Deployments.

```yaml
metadata:
  annotations:
    # For Ingress objects (Cert-Manager)
    cert-manager.io/cluster-issuer: homelab-ca-issuer

    # For ArgoCD-managed apps (set automatically by ArgoCD)
    argocd.argoproj.io/managed-by: argocd

    # For workloads where the chart version matters for compliance
    app.kubernetes.io/chart-version: "<chart-version>"

    # For anything with a known Forgejo PR that deployed it
    gxp.homelab/change-ref: "PR-<number>"
```

---

## Resource Tier Definitions

Every container must have resource requests and limits set (enforced by Gatekeeper). Use the tiers below as starting points, then adjust based on measured usage in [[OPS-07 Performance Baselines]].

### Tier 1 -- Minimal (sidecar containers, simple daemons)

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

Use for: Promtail, Falcosidekick, simple init containers.

### Tier 2 -- Small (lightweight services)

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

Use for: Cert-Manager, Sealed Secrets controller, MetalLB speaker.

### Tier 3 -- Medium (standard platform services)

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

Use for: ArgoCD repo server, Ingress-NGINX, Gatekeeper controller, Falco.

### Tier 4 -- Large (stateful services with significant workload)

```yaml
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 1000m
    memory: 2Gi
```

Use for: Forgejo, Authentik server, Loki, Grafana.

### Tier 5 -- XLarge (database-class and high-memory services)

```yaml
resources:
  requests:
    cpu: 500m
    memory: 2Gi
  limits:
    cpu: 2000m
    memory: 4Gi
```

Use for: MinIO, Prometheus, Authentik PostgreSQL.

### Pipeline Pods -- Variable

Pipeline pods (nf-core tasks) have resource requirements that vary by pipeline step. Set at the Nextflow process level using `cpus` and `memory` directives. The `pipelines` namespace ResourceQuota acts as the aggregate ceiling.

```groovy
// nextflow.config
process {
  withName: 'STAR_ALIGN' {
    cpus   = 4
    memory = 8.GB
  }
  withName: 'FASTQC' {
    cpus   = 2
    memory = 1.GB
  }
}
```

---

## Namespace Resource Quotas

Hard limits on resource consumption per namespace. Prevents any single namespace from starving the cluster.

```yaml
# apps/nextflow/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pipelines-quota
  namespace: pipelines
spec:
  hard:
    requests.cpu: "6"
    requests.memory: 10Gi
    limits.cpu: "8"
    limits.memory: 14Gi
    pods: "20"
    persistentvolumeclaims: "5"
```

No other namespace currently has a ResourceQuota. Add quotas to new namespaces when their workloads are resource-intensive or when platform stability requires protection of shared resources.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why bad | Correct approach |
|---|---|---|
| Using generic names (`app`, `service`, `deployment`) | Unreadable output, Prometheus cannot aggregate meaningfully | Use the actual app name |
| Omitting `app.kubernetes.io/version` | Makes it impossible to correlate a running pod to a specific software version in the IQ | Always set the version label |
| Using `:latest` image tag | Gatekeeper blocks this and it is not reproducible | Always pin to a specific version and digest |
| Setting limits without requests | Scheduler cannot make accurate placement decisions | Always set both requests and limits |
| Setting limits far above requests | Leads to overcommit and OOMKilled events | Limits should be at most 2x requests for most services |
| Single large ConfigMap for everything | Hard to track changes, hard to rotate | One ConfigMap per application per purpose |
