# OPS-02 Reference Commands

> Part of [[README]] | See also: [[OPS-01 Build Instructions]], [[INF-01 Infrastructure Baseline]]

All commands used across the project grouped by tool and context.

---

## Cluster Health

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node talos-asj-72z
kubectl describe node talos-v3h-4m1
kubectl get pods -A -o wide
talosctl -n 192.168.0.134 health
talosctl -n 192.168.0.202 health
talosctl dmesg -n 192.168.0.134
talosctl dmesg -n 192.168.0.202
```

---

## Storage

```bash
kubectl get sc
kubectl get pv,pvc -A
kubectl describe pvc <name> -n <namespace>

talosctl get discoveredvolumes \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig

talosctl read /proc/mounts \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig | grep hdd

talosctl -n 192.168.0.202 disks
```

---

## Talos Machine Config

```bash
# Apply patch (reboots node)
talosctl patch machineconfig \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --patch @~/Kuber/patches/<patch-file>.yaml

# Read current machineconfig
talosctl get machineconfig \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig
```

---

## etcd

```bash
# Snapshot
talosctl etcd snapshot ~/Kuber/snapshots/<name>.snapshot \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Short form
talosctl -n 192.168.0.134 etcd snapshot ~/Kuber/snapshots/<name>.snapshot

talosctl -n 192.168.0.134 etcd members
talosctl -n 192.168.0.134 etcd status
```

---

## Node Management

```bash
# Taint management
kubectl taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-

# Labels
kubectl label node talos-asj-72z node-role=infra
kubectl label node talos-v3h-4m1 node-role=storage
kubectl label node talos-asj-72z node-role-   # remove label

# Drain / cordon
kubectl cordon talos-v3h-4m1
kubectl drain talos-v3h-4m1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon talos-v3h-4m1
```

---

## ArgoCD

```bash
kubectl -n argocd get applications
argocd app sync --all
argocd app sync <app-name>
argocd app get <app-name>
argocd app get <app-name> --refresh
kubectl get applications -n argocd -o wide
```

---

## Sealed Secrets

```bash
# Fetch current public key
kubeseal --fetch-cert \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets

# Seal a secret
kubectl create secret generic <secret-name> \
  --from-literal=key=value \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > apps/<app>/sealed-secret.yaml

# Verify decryption
kubectl get secret <secret-name> -n <namespace>
```

---

## Helm

```bash
helm list -A
helm get values <release-name> -n <namespace>
helm diff upgrade <release-name> <chart> -n <namespace> -f values.yaml
helm rollback <release-name> -n <namespace>
helm uninstall <release-name> -n <namespace>
```

---

## MinIO (mc client)

```bash
mc alias set homelab https://minio.homelab <access-key> <secret-key>
mc ls homelab/
mc ls homelab/pipeline-output/
mc mb homelab/pipeline-input
mc ilm add --expiry-days 30 homelab/pipeline-work
mc mirror homelab/pipeline-output <external-alias>/pipeline-output
mc ping homelab
mc admin info homelab
```

---

## OPA Gatekeeper

```bash
kubectl get constrainttemplates
kubectl get constraints -A
kubectl describe k8srequiredresources require-resource-limits
kubectl get constraints -A -o json | jq '.items[] | {name: .metadata.name, violations: .status.violations}'

# Check if a ServiceAccount can perform an action
kubectl auth can-i create pods \
  --as=system:serviceaccount:pipelines:nextflow-runner \
  -n pipelines
```

---

## Falco

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=falco -o wide
kubectl get pods -n monitoring -l app.kubernetes.io/name=falcosidekick
kubectl rollout restart daemonset/falco -n monitoring
kubectl logs -n monitoring -l app.kubernetes.io/name=falco -f

# Validate rules syntax (run inside Falco pod)
kubectl exec -n monitoring <falco-pod> -- falco -r /etc/falco/rules.d/ --validate
```

---

## RBAC Debugging

```bash
kubectl auth can-i create pods \
  --as=system:serviceaccount:pipelines:nextflow-runner \
  -n pipelines

kubectl get roles,rolebindings -n pipelines
kubectl describe role nextflow-role -n pipelines

kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(.subjects[]?.name == "nextflow-runner")'
```

---

## Nextflow

```bash
nextflow run nf-core/rnaseq \
  -profile test,k8s \
  -r 3.14.0 \
  --outdir s3://pipeline-output/rnaseq-test-$(date +%Y%m%d-%H%M) \
  -resume

nextflow log
nextflow clean -f
kubectl get pods -n pipelines -w
```

---

## Cert-Manager

```bash
kubectl get certificates -A
kubectl get certificaterequest -A
kubectl describe certificate <name> -n <namespace>
kubectl get clusterissuer
```

---

## Quick Diagnostics

```bash
# Why is a pod not running?
kubectl describe pod <pod-name> -n <namespace>

# Logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

# Events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory
```
