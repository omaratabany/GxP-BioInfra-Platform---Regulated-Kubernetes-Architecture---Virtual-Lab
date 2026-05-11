# PH-03 MinIO Object Storage

> Part of [[README]] | Previous: [[PH-02 Authentik SSO]] | Next: [[PH-04 OPA Gatekeeper]]
> CKA domains: ConfigMaps and Secrets as env vars, resource requests/limits, liveness/readiness probes

**Status: NOT STARTED**
**Depends on:** [[PH-00 Cluster Preparation]] -- Beelink HDD and `local-hdd` StorageClass required

---

## Goal

S3-compatible object storage for pipeline data and Loki log chunks running on the Beelink HDD. I also migrate Loki off its local PVC dependency onto MinIO as the backend -- removing the last locally-bound storage component from the monitoring stack.

---

## Buckets

| Bucket | Access | Purpose |
|---|---|---|
| `pipeline-input` | Read-only for pipeline pods | Input data for nf-core runs |
| `pipeline-output` | Write for pipeline pods | nf-core/rnaseq results |
| `pipeline-work` | Read/write, Nextflow SA only | Nextflow work directory -- 30-day lifecycle policy |
| `loki-chunks` | Internal only | Loki log storage, replaces local PVC |

---

## Key Config Targets

- MinIO API at `minio.homelab`
- MinIO Console at `minio-console.homelab`
- Data on Beelink HDD via PVC (`local-hdd` StorageClass)
- Credentials as SealedSecrets
- Loki reconfigured to use `loki-chunks` after MinIO is live

---

## Helm Values (targets)

```yaml
mode: standalone

persistence:
  enabled: true
  storageClass: local-hdd
  size: 200Gi

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 500m

nodeSelector:
  node-role: storage

existingSecret: minio-credentials
```

---

## Loki Migration Config

```yaml
loki:
  storage:
    type: s3
    s3:
      endpoint: http://minio.minio.svc:9000
      bucketnames: loki-chunks
      region: us-east-1
      s3forcepathstyle: true
      insecure: true
```

`s3forcepathstyle: true` is required -- MinIO standalone does not support virtual-hosted style bucket addressing.

---

## Lifecycle Policy

```bash
mc ilm add --expiry-days 30 homelab/pipeline-work
```

Nextflow work directories accumulate fast. 30-day auto-delete keeps the HDD from filling.

---

## Exit Criteria

- `mc ls homelab/pipeline-input` works from the Mac
- Loki writing to `loki-chunks` bucket confirmed in Grafana
- MinIO Console accessible at `minio-console.homelab`
- PVC bound to Beelink HDD

---

## CKA Coverage

ConfigMaps and Secrets injected as environment variables, resource requests and limits, liveness and readiness probes, PVC referencing StorageClass.

---

## G-11 Addition -- Prometheus TSDB Migration to Beelink HDD

INF-03 Misjudgment 3 identifies this as required before Phase 5. By default Prometheus stores its TSDB on ephemeral node storage (Omen's OS disk). If Omen restarts, all Prometheus history is lost. The PQ baseline in [[OPS-07 Performance Baselines]] requires historical metrics to be intact across the full pipeline run window.

This migration must be done in Phase 3 while MinIO is being deployed, because both changes touch the monitoring stack in the same Helm release update.

### Migration Steps

**Step 1 -- Add storage spec to kube-prometheus-stack Helm values**

```yaml
# In apps/monitoring/helm-release.yaml, add under prometheus.prometheusSpec:
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-hdd
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 20Gi
    nodeSelector:
      node-role: storage
    retention: 30d
    retentionSize: 18GB
```

**Step 2 -- Commit the values change to Forgejo via PR**

The PR description should reference this migration as a data persistence improvement and note that Prometheus history will be lost once (the migration creates a new empty PVC -- existing in-memory TSDB data is discarded).

**Step 3 -- ArgoCD syncs the updated Helm values**

The Prometheus StatefulSet will be restarted with the new storage spec. It will provision a PVC from `local-hdd` StorageClass. This binds to the Beelink HDD at `/var/mnt/hdd`.

```bash
# Monitor the PVC binding
kubectl get pvc -n monitoring -w
# Expected: prometheus-db PVC transitions from Pending to Bound on Beelink node

# Monitor Prometheus pod restart
kubectl rollout status statefulset/prometheus-monitoring-kube-prometheus-prometheus -n monitoring
```

**Step 4 -- Verify TSDB is on Beelink**

```bash
kubectl get pvc -n monitoring
# Should show: Bound, storageClass: local-hdd

kubectl get pv | grep prometheus
kubectl describe pv <pv-name> | grep nodeAffinity
# Should show: talos-v3h-4m1 (Beelink)

# Verify data is accumulating
kubectl exec -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -- \
  du -sh /prometheus
# Should grow over time as metrics are scraped
```

**Step 5 -- Verify Prometheus is still scraping correctly after migration**

```bash
# Check targets in Prometheus UI (if accessible via port-forward)
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Open http://localhost:9090/targets
# All targets should show State=UP

# Or check via Grafana
# Grafana -> Explore -> Prometheus -> query: up == 0
# Expected: no results (all scrape targets up)
```

**Data loss note:** The migration discards the in-memory Prometheus TSDB accumulated on Omen's ephemeral storage since initial install. This is acceptable -- the baseline measurement in [[OPS-07 Performance Baselines]] is taken after the migration, so the baseline always references data from the persistent store. There is no compliance issue with losing pre-migration metrics.
