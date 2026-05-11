# OPS-07 Performance Baselines

> Part of [[README]] | See also: [[PH-07 GxP Validation Documentation]], [[OPS-06 OQ Test Scripts]], [[INF-03 Infrastructure Analysis]]

Measured baseline values for both nodes at idle and under pipeline load. PQ acceptance criteria are compared against these baselines, not against theoretical limits. Record actual measured values here as the platform is built -- do not use estimates.

---

## How to Use This File

This file has two sections:

1. **Idle Baseline** -- measured with all platform services running, no pipeline executing. Captured once after PH-06 completes and all services are stable. This is the "floor" -- the resource cost of running the platform itself.

2. **Pipeline Run Metrics** -- measured during the nf-core/rnaseq test profile run. Compared against the idle baseline to determine the pipeline's incremental resource cost. Run twice -- both sets of values must be within 10% of each other to satisfy the PQ reproducibility requirement.

---

## How to Measure

All measurements come from Grafana backed by Prometheus. Use the time ranges specified. Export exact values -- do not round.

```bash
# Open Grafana at https://grafana.homelab
# Datasource: Prometheus
# Dashboard: Kubernetes / Compute Resources / Cluster (or node-specific)

# For storage:
kubectl exec -n minio <minio-pod> -- df -h /export
talosctl -n 192.168.0.202 read /proc/mounts | grep hdd

# For etcd size:
talosctl -n 192.168.0.134 etcd members
# etcd size is visible in the Talos dashboard or via:
kubectl exec -n kube-system etcd-talos-asj-72z -- \
  etcdctl endpoint status --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Section 1 -- Idle Baseline

**Measurement conditions:**
- All platform services running and healthy (PH-00 through PH-06 complete)
- No pipeline pods in any namespace
- System has been stable for at least 30 minutes
- Measurement window: 10 minutes average from Prometheus

**Date measured:** _______________
**Operator:** _______________

### HP Omen (talos-asj-72z) -- Idle

| Metric | Measured Value | Measurement Method |
|---|---|---|
| CPU usage (average, 10 min) | | Grafana: node_cpu_seconds_total rate |
| CPU usage (peak, 10 min) | | Grafana: node_cpu_seconds_total rate |
| RAM used (average, 10 min) | | Grafana: node_memory_MemUsed_bytes |
| RAM used (peak, 10 min) | | Grafana: node_memory_MemUsed_bytes |
| RAM available | | `kubectl describe node talos-asj-72z` |
| Disk IO read (average) | | Grafana: node_disk_read_bytes_total rate |
| Disk IO write (average) | | Grafana: node_disk_written_bytes_total rate |
| Network RX (average) | | Grafana: node_network_receive_bytes_total rate |
| Network TX (average) | | Grafana: node_network_transmit_bytes_total rate |
| Pod count | | `kubectl get pods -A --field-selector=spec.nodeName=talos-asj-72z | wc -l` |

### Beelink (talos-v3h-4m1) -- Idle

| Metric | Measured Value | Measurement Method |
|---|---|---|
| CPU usage (average, 10 min) | | Grafana: node_cpu_seconds_total rate |
| RAM used (average, 10 min) | | Grafana: node_memory_MemUsed_bytes |
| RAM available | | `kubectl describe node talos-v3h-4m1` |
| HDD used (MinIO + Loki + Forgejo) | | `kubectl exec -n minio <pod> -- df -h /export` |
| HDD free | | Same as above |
| MinIO total objects | | `mc ls --recursive homelab | wc -l` |
| Loki ingestion rate (logs/min) | | Grafana: Loki dashboard |
| Disk IO write (average) | | Grafana: node_disk_written_bytes_total rate |
| Pod count | | `kubectl get pods -A --field-selector=spec.nodeName=talos-v3h-4m1 | wc -l` |

### Cluster-Wide -- Idle

| Metric | Measured Value | Measurement Method |
|---|---|---|
| etcd database size | | etcdctl endpoint status |
| Total SealedSecrets | | `kubectl get sealedsecrets -A | wc -l` |
| Total PVCs | | `kubectl get pvc -A | grep Bound | wc -l` |
| Total PV storage allocated | | `kubectl get pv | grep Bound` |
| Prometheus TSDB size | | Grafana: Prometheus stats dashboard |
| ArgoCD application count | | `argocd app list | wc -l` |
| Gatekeeper constraint violations | | `kubectl get constraints -A -o json | jq '[.items[].status.totalViolations] | add'` |

### Per-Component RAM Reference (Idle)

| Component | Namespace | Measured RAM Usage | Notes |
|---|---|---|---|
| ArgoCD (server + repo + app controller) | argocd | | |
| Authentik (server + worker) | authentik | | |
| Authentik PostgreSQL | authentik | | |
| MinIO | minio | | |
| Forgejo | forgejo | | |
| Loki | monitoring | | |
| Prometheus | monitoring | | |
| Grafana | monitoring | | |
| Falco (per node) | monitoring | | |
| OPA Gatekeeper | gatekeeper-system | | |
| Ingress-NGINX | ingress-nginx | | |
| Cert-Manager | cert-manager | | |
| Sealed Secrets | sealed-secrets | | |

---

## Section 2 -- Pipeline Run Metrics

**Run conditions:**
- nf-core/rnaseq at pinned version, test profile
- nextflow run nf-core/rnaseq -profile test,k8s -r <version>
- Work directory: s3://pipeline-work
- Output directory: s3://pipeline-output/pq-run-<N>

### Run 1

**Date:** _______________
**Operator:** _______________
**Nextflow version:** _______________
**nf-core/rnaseq version:** _______________
**Command:**
```
nextflow run nf-core/rnaseq -profile test,k8s -r <version> \
  --outdir s3://pipeline-output/pq-run-1-$(date +%Y%m%d)
```

| Metric | Measured Value | Source |
|---|---|---|
| Pipeline start time | | Nextflow log |
| Pipeline end time | | Nextflow log |
| Total wall time | | Nextflow execution summary |
| Total tasks executed | | Nextflow execution summary |
| Tasks succeeded | | Nextflow execution summary |
| Tasks failed | | Nextflow execution summary |
| Omen peak CPU (during run) | | Grafana: 1-minute interval |
| Omen peak RAM (during run) | | Grafana: 1-minute interval |
| Omen average CPU (run duration) | | Grafana |
| Beelink peak disk IO write | | Grafana |
| Beelink HDD consumed by output | | `mc du homelab/pipeline-output/pq-run-1-*` |
| Beelink HDD consumed by work dir | | `mc du homelab/pipeline-work/` |
| Falco events during run | | Grafana Loki: `{job="falco"} | namespace="pipelines"` count |
| MultiQC report MD5 | | `md5sum <output>/multiqc/multiqc_report.html` |
| MultiQC report location | | MinIO path |

### Run 2

**Date:** _______________
**Operator:** _______________

| Metric | Measured Value | Matches Run 1? |
|---|---|---|
| Total wall time | | Within 10%? |
| Tasks succeeded | | Exact match? |
| Omen peak CPU | | Within 20%? |
| Omen peak RAM | | Within 20%? |
| MultiQC report MD5 | | Exact match with Run 1? |
| Falco event count | | Within 10% of Run 1? |

---

## PQ Acceptance Criteria

Evaluated after both runs are complete. All criteria must pass.

| Criterion | Threshold | Run 1 Value | Run 2 Value | Pass? |
|---|---|---|---|---|
| Pipeline completion | 100% tasks succeed | | | |
| Wall time | Under 60 minutes | | | |
| Omen peak RAM during run | Under 14GB | | | |
| Beelink peak RAM during run | Under 6.5GB | | | |
| MultiQC reproducibility | MD5 checksums match exactly | | | |
| Falco coverage | Events present for 100% of run duration (no gaps > 60s) | | | |
| No CRITICAL Falco events | Zero CRITICAL events during normal run | | | |
| Gatekeeper violations during run | Zero new violations | | | |

**PQ result:** PASS / FAIL
**Date of determination:** _______________
**Operator:** _______________

---

## Anomaly Detection Reference

After the baseline is recorded, these thresholds trigger investigation if exceeded during normal operations (no pipeline running).

| Metric | Idle Baseline | Alert Threshold | Action |
|---|---|---|---|
| Omen CPU | Record above | Idle baseline + 30% sustained for > 5 min | Investigate unexpected workload |
| Omen RAM | Record above | > 14GB | Check for memory leak or unexpected pod |
| Beelink disk IO write | Record above | 10x idle baseline sustained for > 10 min | Check MinIO or Loki for runaway write |
| Beelink HDD free | Record above | < 20% free | Trigger MinIO mirror and cleanup |
| Falco events per hour | Record above | 10x idle rate | Security incident investigation |
| Loki ingestion rate | Record above | 5x idle rate | Log explosion investigation |

These thresholds feed the Prometheus alerting rules defined in [[INF-06 Observability and Alerting]].
