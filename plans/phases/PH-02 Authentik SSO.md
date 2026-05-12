# PH-02 Authentik SSO

> Part of [[README]] | Previous: [[PH-01 Forgejo]] | Next: [[PH-03 MinIO Object Storage]]
> CKA domains: RBAC, ServiceAccounts, ClusterRoles, ClusterRoleBindings

**Status: IN PROGRESS**
**Depends on:** [[PH-01 Forgejo]] -- Forgejo must be running before OIDC client is configured

---

## Goal

A single OIDC provider for the entire platform. All tools authenticate through Authentik. After this phase, no local user passwords exist anywhere on the cluster. This directly satisfies the GxP access control requirement from Annex 11 -- centralised identity, no shared accounts, full login audit trail.

---

## Key Config Targets

- Authentik at `auth.homelab`
- OIDC application configured for Forgejo, ArgoCD, and Grafana
- Groups: `platform-admin`, `developer`, `readonly`
- All group membership managed in Authentik, reflected in K8s RBAC

---

## OIDC Config Per App

### ArgoCD (argocd-cm)

```yaml
oidc.config: |
  name: Authentik
  issuer: https://auth.homelab/application/o/argocd/
  clientID: argocd
  clientSecret: $oidc.authentik.clientSecret
  requestedScopes:
    - openid
    - profile
    - email
    - groups
```

RBAC in argocd-rbac-cm:
```yaml
policy.csv: |
  g, platform-admin, role:admin
  g, developer, role:readonly
```

### Grafana

```ini
[auth.generic_oauth]
enabled = true
name = Authentik
client_id = grafana
auth_url = https://auth.homelab/application/o/authorize/
token_url = https://auth.homelab/application/o/token/
api_url = https://auth.homelab/application/o/userinfo/
role_attribute_path = contains(groups[*], 'platform-admin') && 'Admin' || 'Viewer'
```

---

## Groups & Access Matrix

| Group | Forgejo | ArgoCD | Grafana |
|---|---|---|---|
| platform-admin | Owner | Admin | Admin |
| developer | Write | Read-only | Viewer |
| readonly | Read | Read-only | Viewer |

---

## Exit Criteria

- [x] Authentik deployed at `auth.homelab`
- [x] Authentik PostgreSQL PVC bound on the Beelink storage node through `local-hdd`
- [x] Platform groups created: `platform-admin`, `developer`, `readonly`
- [x] OIDC providers created for Forgejo, ArgoCD, and Grafana
- [x] OIDC client secrets sealed into Git
- [x] Forgejo OIDC source configured against Authentik
- [x] ArgoCD OIDC settings and RBAC configured against Authentik
- [x] Grafana Generic OAuth option configured against Authentik
- [ ] Browser login to Forgejo via Authentik works
- [ ] Browser login to ArgoCD via Authentik works
- [ ] Browser login to Grafana via Authentik works
- [ ] Group membership controls access level across all three apps
- [ ] Local password fallback disabled after SSO is tested

---

## Current Build Notes

Authentik is running from chart `2026.2.2` with server image `ghcr.io/goauthentik/server:2026.2.2`. PostgreSQL is enabled as the backing database and pinned to the storage node with a 10Gi `local-hdd` PVC.

The current cluster needs local homelab DNS from inside pods. CoreDNS now maps `auth.homelab`, `forgejo.homelab`, `argocd.homelab`, `grafana.homelab`, and related platform hostnames to MetalLB VIP `192.168.0.200`.

Grafana is still owned by the older `kube-prometheus-stack` ApplicationSet. The current SSO settings are captured under `k8s/apps/monitoring/` as a live overlay until the monitoring stack is migrated under this GXP repo.

## CKA Coverage

RBAC: ClusterRole, ClusterRoleBinding, Role, RoleBinding. ServiceAccounts and token projection.
