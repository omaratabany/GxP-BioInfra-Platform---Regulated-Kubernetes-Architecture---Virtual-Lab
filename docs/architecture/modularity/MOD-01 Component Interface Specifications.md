# MOD-01 Component Interface Specifications

> Part of [[README]] | See also: [[MOD-02 Modularity and Dependency Map]], [[INF-02 Architecture and Components]], [[ADR-01 Alternative Configurations]]

How each component in this platform connects to every other component. This file defines the interfaces -- the protocols, ports, authentication mechanisms, and data formats that components use to communicate. Any component can be swapped for an alternative as long as the replacement satisfies the same interface contract.

---

## Interface Design Principles

Every component in this platform exposes or consumes a defined interface. The platform is modular because:

1. Components communicate through standard protocols (S3, OIDC, Prometheus metrics, webhook HTTP POST)
2. No component has proprietary coupling to another -- each interface could be satisfied by a different implementation
3. Configuration for each interface is in Helm values or environment variables, not hardcoded

When an alternative component is chosen (see [[ADR-01 Alternative Configurations]]), it must satisfy the interface contract defined here.

---

## Interface Catalogue

### IF-01 -- Object Storage (S3 API)

**Protocol:** S3 (HTTP/HTTPS, path-style addressing)
**Current implementation:** MinIO standalone
**Consumers:** Nextflow (pipeline work dir, output), Loki (log chunks)
**Provider:** MinIO at `http://minio.minio.svc:9000` (internal) / `https://minio.homelab` (external)

**Contract:**
```
Endpoint format:   http[s]://<host>:<port>
Path style:        s3forcepathstyle=true (required for MinIO standalone)
Auth:              AWS Signature V4 (access key + secret key)
TLS:               Optional internal, required external
Buckets:           pipeline-input, pipeline-output, pipeline-work, loki-chunks
```

**Required for swap (e.g., to Rook-Ceph RADOS Gateway, SeaweedFS, or Cloudflare R2):**
- Must support S3 API path-style addressing
- Must support AWS Signature V4 authentication
- Must support IAM policies per bucket
- Must support object versioning (for pipeline-output)
- Must support object lock/retention (for loki-chunks)

**Config location:** `apps/minio/helm-release.yaml`, `apps/minio/policies/`

---

### IF-02 -- Identity and SSO (OIDC)

**Protocol:** OpenID Connect 1.0 (OAuth 2.0 extension)
**Current implementation:** Authentik
**Consumers:** Forgejo, ArgoCD, Grafana, (future) any web-facing service
**Provider:** Authentik at `https://auth.homelab`

**Contract:**
```
Issuer URL:          https://auth.homelab/application/o/<app-name>/
Authorization URL:   https://auth.homelab/application/o/authorize/
Token URL:           https://auth.homelab/application/o/token/
UserInfo URL:        https://auth.homelab/application/o/userinfo/
JWKS URL:            https://auth.homelab/application/o/<app-name>/jwks/
Scopes:              openid, profile, email, groups
Group claim:         groups (array of group names)
```

**Required for swap (e.g., to Keycloak, Zitadel, Dex):**
- Must expose standard OIDC discovery endpoint (/.well-known/openid-configuration)
- Must include `groups` claim in the ID token
- Must support client_credentials and authorization_code flows
- Issuer URL format must be configurable per app (Keycloak uses /realms/<name>)

**Config location:** `apps/authentik/helm-release.yaml`, each app's OIDC config block

---

### IF-03 -- Log Aggregation (Loki HTTP API)

**Protocol:** HTTP POST (Loki push API) / LogQL (query)
**Current implementation:** Loki + Promtail + Falcosidekick
**Producers:** Promtail (all pod logs), Falcosidekick (Falco audit events)
**Consumer:** Grafana (query and dashboard)
**Provider:** Loki at `http://loki.monitoring.svc:3100`

**Contract:**
```
Push endpoint:     POST /loki/api/v1/push
Query endpoint:    GET /loki/api/v1/query_range
Label format:      {job="<source>", namespace="<ns>", pod="<pod>"}
Auth:              X-Scope-OrgID header (value: "1" for single-tenant)
Retention:         Defined in Loki config (default: no limit; MinIO lifecycle handles it)
```

**Required for swap (e.g., to Grafana Cloud, self-hosted OpenSearch, ElasticSearch):**
- Must support Loki-compatible push API OR Falcosidekick must be configured for the alternative
- Grafana datasource must be updated to point at new endpoint
- Query language change: switching from Loki (LogQL) to Elastic (KQL) requires updating all Grafana panels

**Config location:** `apps/loki/helm-release.yaml`, Falcosidekick config in `apps/falco/helm-release.yaml`

---

### IF-04 -- Metrics (Prometheus Exposition Format)

**Protocol:** HTTP GET (pull-based), Prometheus exposition text format
**Current implementation:** kube-prometheus-stack (Prometheus + Alertmanager)
**Producers:** All services exposing /metrics endpoints (Falco, MinIO, ArgoCD, node-exporter, kube-state-metrics)
**Consumer:** Grafana dashboards, Alertmanager for alerts
**Provider:** Prometheus at `http://prometheus.monitoring.svc:9090`

**Contract:**
```
Scrape endpoint:   GET /metrics on each service
Format:            Prometheus text exposition format (UTF-8)
Labels:            Consistent with K8s labeling: namespace, pod, container
Alert routing:     Alertmanager -> (future: Slack, PagerDuty, email)
```

**Required for swap (e.g., to VictoriaMetrics):**
- Must support Prometheus scrape configuration (scrape_configs)
- Must expose PromQL-compatible query API for Grafana
- VictoriaMetrics is a drop-in replacement -- same config format, lower RAM

---

### IF-05 -- GitOps Source (Git HTTP/SSH)

**Protocol:** Git over SSH (port 2222) or HTTPS (port 443)
**Current implementation:** Forgejo (transitioning from GitHub)
**Consumer:** ArgoCD (watches repo for manifest changes)
**Provider:** Forgejo at `https://forgejo.homelab`

**Contract:**
```
Repo URL format:   ssh://git@forgejo.homelab:2222/<org>/<repo>.git
                   https://forgejo.homelab/<org>/<repo>.git
Webhook:           POST to http://argocd-server.argocd.svc.cluster.local/api/webhook
Webhook event:     push (all branches or main only)
Auth:              SSH key or HTTPS token for ArgoCD repo credential
```

**Required for swap (e.g., back to GitHub, to GitLab):**
- ArgoCD repo credential must be updated to new remote URL
- Webhook URL and secret remain the same (ArgoCD side)
- Branch protection rules must be recreated in the new provider
- PR review requirements must be recreated

**Config location:** ArgoCD repository configuration in `apps/argocd/`

---

### IF-06 -- Admission Control (Kubernetes Webhook)

**Protocol:** HTTPS (Kubernetes ValidatingAdmissionWebhook)
**Current implementation:** OPA Gatekeeper
**Consumer:** Kubernetes API server (calls webhook for every CREATE/UPDATE)
**Provider:** Gatekeeper webhook service at `gatekeeper-webhook-service.gatekeeper-system.svc`

**Contract:**
```
Webhook URL:       https://gatekeeper-webhook-service.gatekeeper-system.svc:443/v1/admit
TLS:               Gatekeeper manages its own TLS cert (self-signed, trusted by K8s API server)
Failure policy:    Fail (after initial stable rollout)
Timeout:           10 seconds (K8s default)
Rules evaluated:   ConstraintTemplates + Constraint objects
```

**Required for swap (e.g., to Kyverno):**
- Kyverno installs its own ValidatingWebhookConfiguration
- All Gatekeeper ConstraintTemplates must be rewritten as Kyverno ClusterPolicy objects
- Rego policies are not portable -- full rewrite required
- Audit mode equivalent in Kyverno: `validationFailureAction: audit`

---

### IF-07 -- Pipeline Orchestration (K8s Pod API)

**Protocol:** Kubernetes API (HTTPS, ServiceAccount token auth)
**Current implementation:** Nextflow with K8s executor
**Consumer:** Nextflow (creates, watches, deletes pods)
**Provider:** Kubernetes API server at `https://kubernetes.default.svc:443`

**Contract:**
```
Auth:              nextflow-runner ServiceAccount token (projected volume)
Namespace:         pipelines (scoped Role, not ClusterRole)
Operations:        pods CRUD, pods/log get, services CRUD, configmaps CRUD
Work directory:    s3://pipeline-work (via IF-01)
Output:            s3://pipeline-output (via IF-01)
```

**Required for swap (e.g., to Argo Workflows):**
- Argo Workflows uses CRD-based workflow definitions instead of Nextflow DSL
- All nf-core pipeline definitions would need to be rewritten as Argo Workflow templates
- S3 storage interface (IF-01) is compatible with Argo Workflows artifacts

---

### IF-08 -- Runtime Security Events (Falco gRPC / Falcosidekick HTTP)

**Protocol:** Falco -> Falcosidekick (gRPC or stdout), Falcosidekick -> Loki (HTTP POST)
**Current implementation:** Falco DaemonSet + Falcosidekick sidecar
**Producer:** Falco eBPF kernel hooks
**Consumer:** Loki (via Falcosidekick)

**Contract:**
```
Falco output:      Falcosidekick receives structured JSON events
Event fields:      output, priority, rule, time, hostname, source
Loki push:         HTTP POST to http://loki.monitoring.svc:3100/loki/api/v1/push
Custom labels:     cluster=homelab, env=dev
Custom fields:     Defined in Falcosidekick config
```

**Required for swap (e.g., route events to Slack or PagerDuty instead of Loki):**
- Falcosidekick supports 50+ output destinations via config -- no code change required
- Add the destination config to `apps/falco/helm-release.yaml` under `falcosidekick.config`
- Loki output can coexist with other outputs simultaneously

---

## Interface Dependency Summary

```
IF-01 (S3)          <-- Nextflow (IF-07), Loki (IF-03)
IF-02 (OIDC)        <-- Forgejo, ArgoCD, Grafana
IF-03 (Loki)        <-- Promtail, Falcosidekick (IF-08), Grafana
IF-04 (Prometheus)  <-- Grafana, Alertmanager
IF-05 (Git)         <-- ArgoCD
IF-06 (Webhook)     <-- K8s API server (calls Gatekeeper)
IF-07 (K8s API)     <-- Nextflow
IF-08 (Falco)       --> Loki (via IF-03)
```
