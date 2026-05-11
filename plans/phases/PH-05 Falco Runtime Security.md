# PH-05 Falco Runtime Security

> Part of [[README]] | Previous: [[PH-04 OPA Gatekeeper]] | Next: [[PH-06 Nextflow and nf-core]]
> CKA domains: DaemonSets, security contexts, host-level access patterns

**Status: NOT STARTED**
**Depends on:** [[PH-03 MinIO Object Storage]] for MinIO-backed Loki (audit events must persist), [[PH-04 OPA Gatekeeper]] active so Falco DaemonSet passes admission

---

## Goal

Tamper-evident audit events for all activity in the `pipelines` namespace. Events ship to Loki via Falcosidekick. This satisfies EU GMP Annex 11 Clause 9 (audit trail) and Clause 12.1 (security) at the runtime level.

---

## Architecture

```
Falco DaemonSet (one pod per node, eBPF driver)
  Custom GxP rules evaluate each syscall event
    Matching events forwarded to Falcosidekick
      Falcosidekick routes to Loki (http://loki.monitoring.svc:3100)
        Events queryable in Grafana with timestamp, pod, user, syscall
```

---

## Helm Values (targets)

```yaml
driver:
  kind: ebpf   # Required for Talos -- kernel module cannot load on immutable OS

falcosidekick:
  enabled: true
  config:
    loki:
      hostport: http://loki.monitoring.svc:3100
      customHeaders: "X-Scope-OrgID:1"
    customfields: "cluster=homelab,env=dev"

resources:
  limits:
    cpu: 500m
    memory: 256Mi

tolerations:
  - effect: NoSchedule
    operator: Exists
```

---

## Custom Rules (summary)

| Rule | Priority | Trigger |
|---|---|---|
| Exec Into Pipeline Pod | AUDIT | Any shell exec in `pipelines` namespace |
| File Write Outside Tmp in Pipeline | AUDIT | File written outside /tmp in pipeline pod |
| Privilege Escalation Attempt | CRITICAL | sudo, su, or newgrp in any container |
| Unexpected Outbound Connection | WARNING | Pipeline pod connecting to unexpected destination |
| Pipeline Pod Reading Sensitive Paths | WARNING | Pipeline pod reading /etc or /var/run |

Full rule YAML is in the custom-rules ConfigMap at `apps/falco/custom-rules-configmap.yaml`.

---

## Validation Steps

```bash
# Verify DaemonSet -- one pod per node
kubectl get pods -n monitoring -l app.kubernetes.io/name=falco -o wide

# Test audit event
kubectl run audit-test --image=busybox:1.36 -n pipelines -it --rm -- /bin/sh
# Exit, then Grafana -> Loki: {job="falco"} |= "exec into pipeline pod"
# Must appear within 10 seconds
```

---

## Exit Criteria

- Falco DaemonSet has one Running pod per node
- Falcosidekick routes events to Loki -- events visible in Grafana
- Exec into any `pipelines` pod generates a Loki entry within 10 seconds
- CRITICAL rule fires on `sudo` attempt inside a pipeline pod

---

## CKA Coverage

DaemonSets (one pod per node), security contexts, tolerations for tainted nodes, ConfigMap volume mounts.


---

## G-21 Correction -- Falcosidekick Deployment Model

Falcosidekick is deployed as a subchart of the Falco Helm release, not as a separate ArgoCD application. The `falcosidekick.enabled: true` value activates it within the same ArgoCD sync. There is no separate `apps/falcosidekick/` directory. One ArgoCD Application manages both. All Falcosidekick config lives under the `falcosidekick:` key in `apps/falco/helm-release.yaml`. Adding a second output destination (e.g., Grafana Cloud) only requires editing that key and pushing a PR.
