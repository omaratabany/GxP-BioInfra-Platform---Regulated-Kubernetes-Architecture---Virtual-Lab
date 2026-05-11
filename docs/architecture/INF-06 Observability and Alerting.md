# INF-06 Observability and Alerting

> Part of [[README]] | See also: [[SEC-01 Security Architecture]], [[OPS-07 Performance Baselines]], [[PH-05 Falco Runtime Security]]

Defines the full observability stack: what Prometheus scrapes, what alert rules fire at what thresholds, how Grafana dashboards are structured, and where alerts route. This file is the specification -- the actual configurations live in the Helm values files in the repo.

---

## Observability Stack

```
Prometheus (pull)
  scrapes: node-exporter, kube-state-metrics, all /metrics endpoints
  stores: TSDB on Beelink HDD PVC (migrated from ephemeral in PH-03)
  evaluates: alert rules defined in this file
  routes to: Alertmanager

Alertmanager
  receives: firing alerts from Prometheus
  routes to: (currently) log-only -- future: Slack or PagerDuty

Grafana
  datasource 1: Prometheus (metrics)
  datasource 2: Loki (logs)
  dashboards: defined in this file

Loki
  receives: pod logs from Promtail, Falco events from Falcosidekick
  stores: MinIO loki-chunks bucket
  queried by: Grafana

Falco
  generates: runtime security events
  routes to: Falcosidekick -> Loki
  also feeds: custom Grafana Falco dashboard
```

---

## Prometheus Scrape Targets

| Target | Namespace | Port | Scrape Interval | Labels |
|---|---|---|---|---|
| node-exporter (Omen) | monitoring | 9100 | 15s | node=talos-asj-72z |
| node-exporter (Beelink) | monitoring | 9100 | 15s | node=talos-v3h-4m1 |
| kube-state-metrics | monitoring | 8080 | 30s | |
| kube-apiserver | kube-system | 6443 | 30s | |
| kubelet (Omen) | kube-system | 10250 | 15s | |
| kubelet (Beelink) | kube-system | 10250 | 15s | |
| ArgoCD | argocd | 8083 | 30s | |
| MinIO | minio | 9000 | 30s | |
| Falco | monitoring | 8765 | 15s | |
| Gatekeeper | gatekeeper-system | 8888 | 30s | |
| Loki | monitoring | 3100 | 30s | |

---

## Prometheus Alert Rules

All rules are written in PromQL and loaded via a PrometheusRule custom resource (part of kube-prometheus-stack). The ConfigMap lives at `apps/monitoring/prometheus-rules.yaml` in the repo.

### Platform Health Alerts

```yaml
groups:
- name: platform-health
  rules:

  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Node {{ $labels.node }} is not Ready"
      description: "Node has been NotReady for more than 2 minutes. Check talosctl health."

  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} is crash looping"
      description: "Container {{ $labels.container }} has restarted multiple times in 15 minutes."

  - alert: PodNotRunning
    expr: kube_pod_status_phase{phase!~"Running|Succeeded"} == 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} is not Running"

  - alert: PVCStorageLow
    expr: (kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes) < 0.20
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "PVC {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
      description: "Less than 20% storage remaining on PVC. Trigger MinIO mirror and cleanup."

  - alert: PVCStorageCritical
    expr: (kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes) < 0.10
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "PVC {{ $labels.persistentvolumeclaim }} is critically full"
      description: "Less than 10% storage remaining. Immediate action required."

  - alert: CertificateExpiringSoon
    expr: certmanager_certificate_expiration_timestamp_seconds - time() < 604800
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificate {{ $labels.name }} in {{ $labels.namespace }} expires in less than 7 days"
```

### Security Alerts

```yaml
- name: security
  rules:

  - alert: GatekeeperViolationDetected
    expr: gatekeeper_violations > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Gatekeeper constraint {{ $labels.constraint }} has {{ $value }} violations"
      description: "Run: kubectl get constraints -A -o json | jq to identify violations"

  - alert: FalcoCriticalEvent
    expr: increase(falco_events_total{priority="CRITICAL"}[5m]) > 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "Falco CRITICAL event detected on {{ $labels.hostname }}"
      description: "Check Grafana Loki: {job=\"falco\", priority=\"CRITICAL\"} for details. Follow SEC-03 Incident Response Playbook."

  - alert: SealedSecretsControllerDown
    expr: kube_deployment_status_replicas_available{deployment="sealed-secrets", namespace="sealed-secrets"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Sealed Secrets controller is not running"
      description: "All new secret decryption will fail. See OPS-05 Scenario 4 for recovery."

  - alert: ArgoCDOutOfSync
    expr: argocd_app_info{sync_status="OutOfSync"} == 1
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "ArgoCD application {{ $labels.name }} is OutOfSync for more than 15 minutes"
      description: "Run: argocd app diff {{ $labels.name }} to inspect. Check Forgejo for recent commits."
```

### Resource Consumption Alerts

```yaml
- name: resources
  rules:

  - alert: OmenHighMemory
    expr: (node_memory_MemTotal_bytes{instance="192.168.0.134:9100"} - node_memory_MemAvailable_bytes{instance="192.168.0.134:9100"}) / node_memory_MemTotal_bytes{instance="192.168.0.134:9100"} > 0.90
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Omen node memory usage above 90%"
      description: "Current usage: {{ $value | humanizePercentage }}. Check for runaway pods in all namespaces."

  - alert: BeelinkHighMemory
    expr: (node_memory_MemTotal_bytes{instance="192.168.0.202:9100"} - node_memory_MemAvailable_bytes{instance="192.168.0.202:9100"}) / node_memory_MemTotal_bytes{instance="192.168.0.202:9100"} > 0.85
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Beelink node memory usage above 85%"
      description: "Beelink has less RAM headroom. At 85% investigate before it impacts MinIO."

  - alert: PipelineResourceQuotaNearLimit
    expr: kube_resourcequota_used{resource="requests.cpu", namespace="pipelines"} / kube_resourcequota_hard{resource="requests.cpu", namespace="pipelines"} > 0.80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pipeline namespace CPU quota at {{ $value | humanizePercentage }}"
      description: "ResourceQuota is 80% consumed. New pipeline pods may fail to schedule."

  - alert: BeelinkDiskIOSpike
    expr: rate(node_disk_written_bytes_total{instance="192.168.0.202:9100"}[5m]) > 100000000
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Beelink disk write rate above 100MB/s sustained"
      description: "Sustained high write rate. Check MinIO for runaway data ingestion or Loki log explosion."
```

### GxP-Specific Alerts

```yaml
- name: gxp-compliance
  rules:

  - alert: LokiIngestionStopped
    expr: rate(loki_ingester_chunks_flushed_total[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Loki is not ingesting new log data"
      description: "Audit trail may have a gap. Check Loki pod status and MinIO connectivity. Annex 11 Clause 9 at risk."

  - alert: FalcoNotRunningOnNode
    expr: kube_daemonset_status_number_ready{daemonset="falco"} < kube_daemonset_status_desired_number_scheduled{daemonset="falco"}
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Falco DaemonSet has {{ $value }} pods missing"
      description: "Runtime audit trail has coverage gaps. Annex 11 Clause 9 at risk. Check Falco pod status."

  - alert: GatekeeperWebhookDown
    expr: kube_deployment_status_replicas_available{deployment=~"gatekeeper-controller-manager", namespace="gatekeeper-system"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Gatekeeper admission controller is not running"
      description: "Policy enforcement is offline. All workload admissions are uncontrolled. Annex 11 Clause 4.4 at risk."

  - alert: MinIODown
    expr: kube_deployment_status_replicas_available{deployment=~"minio.*", namespace="minio"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "MinIO is not running"
      description: "Audit trail persistence and pipeline data storage unavailable. Falco events will be lost if Loki cannot flush to MinIO."
```

---

## Grafana Dashboard Specifications

### Dashboard 1 -- Platform Overview

**Purpose:** At-a-glance cluster health for daily checks (see OPS-04 daily check procedure).

| Panel | Type | PromQL / LogQL | Description |
|---|---|---|---|
| Node Status | Stat | `kube_node_status_condition{condition="Ready",status="true"}` | Green/Red per node |
| Total Pod Count | Stat | `count(kube_pod_status_phase{phase="Running"})` | Running pods |
| Unhealthy Pods | Table | `kube_pod_status_phase{phase!~"Running|Succeeded|Pending"}` | Pods needing attention |
| Omen RAM Used | Gauge | `node_memory_MemUsed_bytes{instance="192.168.0.134:9100"}` | Current RAM |
| Beelink RAM Used | Gauge | `node_memory_MemUsed_bytes{instance="192.168.0.202:9100"}` | Current RAM |
| HDD Free % | Gauge | `kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes` for minio PVC | Storage runway |
| Active Alerts | Alert list | All firing alerts | Quick incident view |
| Gatekeeper Violations | Stat | `sum(gatekeeper_violations)` | Should always be 0 |
| ArgoCD Sync Status | Table | `argocd_app_info` | All apps, sync status |
| Cert Expiry | Table | `certmanager_certificate_expiration_timestamp_seconds - time()` | Days until expiry |

### Dashboard 2 -- Falco Audit Events

**Purpose:** Annex 11 audit trail review interface.

| Panel | Type | Query | Description |
|---|---|---|---|
| Events by Priority (24h) | Bar chart | `{job="falco"}` grouped by priority | Event volume trend |
| CRITICAL Events | Logs panel | `{job="falco"} \| json \| priority="CRITICAL"` | Requires immediate review |
| AUDIT Events (exec) | Logs panel | `{job="falco"} \| json \| rule="Exec Into Pipeline Pod"` | All exec events |
| Events by Namespace | Pie chart | `{job="falco"}` grouped by namespace | Where events originate |
| Events in Pipelines Namespace | Logs panel | `{job="falco"} \| json \| namespace="pipelines"` | Pipeline-specific audit |
| Event Rate | Time series | `rate({job="falco"}[5m])` | Anomaly detection |

### Dashboard 3 -- Pipeline Run Monitor

**Purpose:** Real-time view during nf-core/rnaseq pipeline execution. Used for PQ metric capture.

| Panel | Type | Query | Description |
|---|---|---|---|
| Omen CPU % | Time series | `rate(node_cpu_seconds_total{mode!="idle",instance="192.168.0.134:9100"}[1m])` | CPU during run |
| Omen RAM | Time series | `node_memory_MemUsed_bytes{instance="192.168.0.134:9100"}` | RAM during run |
| Pipeline Pod Count | Stat | `count(kube_pod_status_phase{namespace="pipelines",phase="Running"})` | Active task pods |
| Beelink Disk Write | Time series | `rate(node_disk_written_bytes_total{instance="192.168.0.202:9100"}[1m])` | Data being written |
| MinIO Objects Created | Time series | `rate(minio_s3_requests_total{method="PUT"}[1m])` | Output rate |
| Falco Events (Pipelines) | Logs panel | `{job="falco"} \| json \| namespace="pipelines"` | Runtime audit stream |
| ResourceQuota Consumption | Bar chart | `kube_resourcequota_used / kube_resourcequota_hard` for pipelines namespace | Quota headroom |

### Dashboard 4 -- Security Posture

**Purpose:** Monthly security review and ISO 27001 8.16 monitoring control.

| Panel | Type | Query | Description |
|---|---|---|---|
| Gatekeeper Violations by Constraint | Bar chart | `gatekeeper_violations` by constraint name | Policy compliance |
| Failed Login Attempts | Logs panel | Loki: `{job="authentik"}` with login failure events | Access control monitoring |
| ArgoCD Operations | Logs panel | Loki: `{job="argocd"}` | Change log stream |
| Certificate Status | Table | `certmanager_certificate_ready_status` | TLS health |
| Sealed Secrets Controller | Stat | `kube_deployment_status_replicas_available{deployment="sealed-secrets"}` | Key management health |

---

## Alertmanager Routing

Current routing is log-only (alerts are recorded in Alertmanager history). Future: Slack or PagerDuty.

```yaml
# apps/monitoring/alertmanager-config.yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: 'critical-receiver'
    repeat_interval: 1h

receivers:
- name: 'default-receiver'
  # Currently: no-op. Add Slack webhook URL here when available.

- name: 'critical-receiver'
  # Currently: no-op. Critical alerts visible in Grafana alert list.
  # Future: webhook to Slack or PagerDuty
```

---

## Prometheus TSDB Configuration

Prometheus TSDB must be stored on a PVC backed by the Beelink HDD, not on ephemeral node storage. If Prometheus data is lost on Omen restart, the PQ baseline comparison fails.

**Required Helm values addition in kube-prometheus-stack:**
```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-hdd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
    nodeSelector:
      node-role: storage
    retention: 30d
    retentionSize: 18GB
```

This migration must be completed in PH-03 before Prometheus accumulates baseline data. See the TSDB migration procedure in [[PH-03 MinIO Object Storage]].
