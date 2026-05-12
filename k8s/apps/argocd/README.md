# ArgoCD Configuration

This directory captures ArgoCD platform configuration that currently exists as live cluster state.

## Files

| File | Purpose |
|---|---|
| `argocd-oidc-config-patch.yaml` | Merge patch for Authentik OIDC settings in `argocd-cm` |
| `argocd-rbac-config-patch.yaml` | Merge patch for group-based ArgoCD RBAC in `argocd-rbac-cm` |

## Apply

Use merge patch commands so the existing ArgoCD config keys are preserved:

```bash
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  patch cm argocd-cm -n argocd --type merge \
  --patch-file k8s/apps/argocd/argocd-oidc-config-patch.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  patch cm argocd-rbac-cm -n argocd --type merge \
  --patch-file k8s/apps/argocd/argocd-rbac-config-patch.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  rollout restart deployment/argocd-server -n argocd
```

The root ArgoCD Application points at the internal Forgejo mirror and recursively reads `k8s/apps`. Patch helper files are excluded by root and must be applied with the commands above.
