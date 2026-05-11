# Kubernetes Manifests

This directory contains deployable cluster configuration.

## Structure

| Directory | Purpose |
|---|---|
| `apps/` | Platform application manifests and ArgoCD Application definitions |
| `apps/platform-network/` | Shared ingress and load balancer configuration |
| `tests/` | Temporary verification manifests used during build validation |

## Rule

Manifests in `apps/` are intended to represent desired cluster state. Manifests in `tests/` are used for verification and should not leave test workloads running after evidence is collected.
