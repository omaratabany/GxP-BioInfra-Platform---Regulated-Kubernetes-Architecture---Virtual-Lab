# PH-01 Forgejo

> Part of [[README]] | Previous: [[PH-00 Cluster Preparation]] | Next: [[PH-02 Authentik SSO]]
> CKA domains: Helm deployments, Ingress, TLS secrets, PVC with StorageClass, StatefulSet behaviour

**Status: COMPLETE**
**Depends on:** [[PH-00 Cluster Preparation]] -- `local-hdd` StorageClass must be working

---

## Goal

All my project code lives in a self-hosted Forgejo instance on the cluster. GitHub drops to a public mirror. ArgoCD watches Forgejo on push via webhook. This is also my change control mechanism -- all changes go through Forgejo PRs with mandatory review before ArgoCD syncs, satisfying the Change Control SOP in [[PH-07 GxP Validation Documentation]].

---

## Repo Structure

```
k8s/
  apps/
    platform-network/
      metallb-l2-config.yaml
    forgejo/
      application.yaml
      namespace.yaml
      sealed-secret.yaml
      values.yaml
```

---

## Key Config Targets

- Forgejo at `forgejo.homelab` with TLS
- Persistent storage on Beelink HDD via `local-hdd` StorageClass
- SSH clone on port 2222 via NodePort
- Webhook to ArgoCD on push
- Admin credentials sealed via Sealed Secrets

---

## Helm Values (targets)

```yaml
gitea:
  admin:
    existingSecret: forgejo-admin-secret
  config:
    server:
      DOMAIN: forgejo.homelab
      SSH_DOMAIN: forgejo.homelab
      SSH_PORT: 2222
    database:
      DB_TYPE: sqlite3

persistence:
  enabled: true
  storageClass: local-hdd
  size: 20Gi

ingress:
  enabled: true
  hosts:
    - host: forgejo.homelab
      paths:
        - path: /
  tls:
    - secretName: forgejo-tls
      hosts:
        - forgejo.homelab
```

---

## Sealed Secret

```bash
kubectl create secret generic forgejo-admin-secret \
  --from-literal=username=gxp-admin \
  --from-literal=password=<password> \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > \
  apps/forgejo/sealed-secret.yaml
```

---

## ArgoCD Webhook

After Forgejo is running: repo settings -> Webhooks -> Add
- URL: `http://argocd-server.argocd.svc.cluster.local/api/webhook`
- Events: push events

---

## OIDC (Post-PH-02)

After [[PH-02 Authentik SSO]] is live, Forgejo is configured as an OIDC client. Local user passwords are removed. All logins go via Authentik.

---

## Exit Criteria

- SSH NodePort is reachable from the Mac on `192.168.0.202:32222`
- HTTPS route works through the MetalLB VIP at `https://forgejo.homelab`
- ArgoCD manages the Forgejo Helm release from the upstream OCI chart
- Admin credentials exist only as a SealedSecret
- Forgejo PVC is bound to Beelink HDD

Bootstrap completion:
- Internal repository created: `gxp-admin/gxp-platform`
- Current platform tree pushed to Forgejo on branch `main`
- Push webhook configured to `http://argocd-server.argocd.svc.cluster.local/api/webhook`
- Temporary setup tokens removed after admin password rotation

---

## CKA Coverage

Helm-based deployments and values override, Ingress with TLS secret, PVC referencing a named StorageClass, StatefulSet behaviour.
