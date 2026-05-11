# PH-06 Nextflow and nf-core

> Part of [[README]] | Previous: [[PH-05 Falco Runtime Security]] | Next: [[PH-07 GxP Validation Documentation]]
> CKA domains: RBAC scoped to namespace, ServiceAccount token projection, pod lifecycle, resource quotas

**Status: NOT STARTED**
**Depends on:** [[PH-03 MinIO Object Storage]], [[PH-04 OPA Gatekeeper]], [[PH-05 Falco Runtime Security]] all active

---

## Goal

A real bioinformatics pipeline running end-to-end on my cluster with a full audit trail captured and output in MinIO. nf-core/rnaseq with the test profile and the K8s executor is the integration test for everything I built in PH-00 through PH-05. If the pipeline completes, the Falco audit trail is in Loki, and the output is in MinIO, the platform works as designed.

---

## RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nextflow-runner
  namespace: pipelines
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nextflow-role
  namespace: pipelines
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "watch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["create", "delete", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nextflow-binding
  namespace: pipelines
subjects:
  - kind: ServiceAccount
    name: nextflow-runner
    namespace: pipelines
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nextflow-role
```

---

## Nextflow Configuration (Mac)

```groovy
process {
  executor = 'k8s'
  container = 'quay.io/nf-core/rnaseq:3.14.0@sha256:<digest>'
  cpus = 2
  memory = '4 GB'

  withLabel: process_high {
    cpus = 4
    memory = '8 GB'
  }
}

k8s {
  namespace = 'pipelines'
  serviceAccount = 'nextflow-runner'
  storageClaimName = 'pipeline-work-pvc'
  workDir = 's3://pipeline-work'
  pullPolicy = 'IfNotPresent'

  pod = [
    [nodeSelector: 'node-role=infra']
  ]
}

aws {
  accessKey = '<minio-access-key>'
  secretKey = '<minio-secret-key>'
  client {
    endpoint = 'https://minio.homelab'
    s3PathStyleAccess = true
    signerOverride = 'AWSS3V4SignerType'
  }
}
```

`nodeSelector: node-role=infra` pins all pipeline pods to the Omen. Beelink handles MinIO storage IO only. See [[INF-03 Infrastructure Analysis]] Misjudgment 1.

---

## Run Command

```bash
nextflow run nf-core/rnaseq \
  -profile test,k8s \
  -r 3.14.0 \
  --outdir s3://pipeline-output/rnaseq-test-$(date +%Y%m%d-%H%M) \
  -resume
```

---

## Resource Quota for Pipelines Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pipelines-quota
  namespace: pipelines
spec:
  hard:
    requests.cpu: "6"
    requests.memory: "10Gi"
    limits.cpu: "8"
    limits.memory: "12Gi"
    pods: "20"
```

---

## Exit Criteria

- `nextflow run nf-core/rnaseq -profile test,k8s` completes without error
- MultiQC report in MinIO `pipeline-output` is populated and shows pass
- Falco audit trail in Loki covers the full run duration
- All pipeline pods scheduled to Omen -- Beelink shows disk IO only, not CPU load
- ResourceQuota enforced in the `pipelines` namespace

---

## CKA Coverage

RBAC scoped to a single namespace (Role vs ClusterRole), ServiceAccount token projection, pod lifecycle management, resource quotas.
