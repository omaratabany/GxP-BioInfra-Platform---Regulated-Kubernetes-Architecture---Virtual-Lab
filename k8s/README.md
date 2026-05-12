# Kubernetes Manifests

This directory contains deployable cluster configuration.

## Structure

| Directory | Purpose |
|---|---|
| `apps/` | Platform application manifests and ArgoCD Application definitions |
| `apps/authentik/` | Authentik SSO deployment and sealed OIDC client secrets |
| `apps/forgejo/` | Forgejo deployment, storage, ingress, and homelab CA trust |
| `apps/monitoring/` | Grafana SSO overlay for the existing monitoring stack |
| `apps/platform-network/` | Shared ingress and load balancer configuration |
| `tests/` | Temporary verification manifests used during build validation |

## Rule

Manifests in `apps/` are intended to represent desired cluster state. Manifests in `tests/` are used for verification and should not leave test workloads running after evidence is collected.

`apps/monitoring/grafana-oidc-deployment-patch.yaml` is a strategic merge patch for the current Grafana Deployment. Do not apply it as a full Deployment manifest.
