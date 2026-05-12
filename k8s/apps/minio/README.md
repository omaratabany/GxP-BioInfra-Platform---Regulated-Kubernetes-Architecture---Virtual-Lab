# MinIO

MinIO provides S3-compatible storage for pipeline data and Loki chunks.

## Layout

| File | Purpose |
|---|---|
| `namespace.yaml` | Namespace and Pod Security labels |
| `sealed-secret.yaml` | Sealed root credential |
| `pvc.yaml` | 200Gi `local-hdd` storage claim on the Beelink HDD |
| `deployment.yaml` | Single-node MinIO server pinned to the storage node |
| `service.yaml` | ClusterIP service for API and console |
| `ingress.yaml` | TLS ingress for `minio.homelab` and `minio-console.homelab` |
| `provisioning-job.yaml` | Bucket creation, versioning, and work bucket lifecycle |
| `policies-configmap.yaml` | MinIO IAM policies for platform service users |
| `users-sealed-secret.yaml` | Sealed service-user credentials |
| `iam-job.yaml` | Service-user and policy provisioning |

## Buckets

| Bucket | Purpose |
|---|---|
| `pipeline-input` | Input data for pipeline runs |
| `pipeline-output` | Versioned pipeline outputs |
| `pipeline-work` | Nextflow work directory with 30-day expiry |
| `loki-chunks` | Versioned object-lock bucket for Loki chunks |
