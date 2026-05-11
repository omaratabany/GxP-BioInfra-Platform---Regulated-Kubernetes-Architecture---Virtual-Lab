# PH-02 Authentik SSO

> Part of [[README]] | Previous: [[PH-01 Forgejo]] | Next: [[PH-03 MinIO Object Storage]]
> CKA domains: RBAC, ServiceAccounts, ClusterRoles, ClusterRoleBindings

**Status: NOT STARTED**
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

- Login to Forgejo via Authentik works -- no local password accepted
- Login to ArgoCD via Authentik works
- Login to Grafana via Authentik works
- Group membership controls access level across all three apps
- No shared accounts exist anywhere on the platform

---

## CKA Coverage

RBAC: ClusterRole, ClusterRoleBinding, Role, RoleBinding. ServiceAccounts and token projection.
