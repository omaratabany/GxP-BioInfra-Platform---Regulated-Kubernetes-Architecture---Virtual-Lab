# OPS-01 Build Instructions

> Part of [[README]] | See also: [[OPS-02 Reference Commands]], [[OPS-03 Implementation Log]], [[INF-05 Hardware Scaling Guide]]

Step-by-step implementation guide. Each phase has its own prerequisites, commands, validation steps, and a failure playbook for the most likely issues you will hit.

---

## Minimum Infrastructure Required

| Resource | Minimum | Notes |
|---|---|---|
| Nodes | 1 (untainted control plane) | 2 nodes recommended |
| CPU | 4 cores allocatable | 8+ recommended before running Nextflow |
| RAM | 8GB allocatable | 16GB+ before you run pipeline workloads |
| Storage | 50GB fast + 200GB bulk | SSD for OS/state, HDD acceptable for pipeline data |
| Network | L2 LAN reachable | MetalLB L2 mode requires nodes on same broadcast domain |
| OS | Talos v1.12+ or any K8s-compatible Linux | This guide assumes Talos |
| kubectl | Any recent version | Match minor version to cluster when possible |
| talosctl | Match your Talos version | Version mismatch causes confusing errors |
| Helm | v3.x | v2 is EOL |

Single-node minimum: untaint the control plane, everything runs on one machine. 8 cores / 16GB / 200GB is the floor for a single-node deployment that does not OOM on Nextflow. See [[INF-05 Hardware Scaling Guide]] Tier 1 for single-node config adjustments.

---

## Phase 0 -- Cluster Preparation

### Step 1 -- Verify cluster health

```bash
kubectl get nodes -o wide
talosctl -n 192.168.0.134 health
```

Both nodes must show `Ready`. Talos health check must return no errors.

### Step 2 -- Remove control-plane taint

```bash
kubectl taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-
kubectl describe node talos-asj-72z | grep Taint
# Expected: <none>
```

### Step 3 -- Apply node labels

```bash
kubectl label node talos-asj-72z node-role=infra
kubectl label node talos-v3h-4m1 node-role=storage
kubectl get nodes --show-labels | grep node-role
```

### Step 4 -- Apply Beelink HDD mount patch

```bash
talosctl patch machineconfig \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --patch @~/Kuber/patches/beelink-hdd.yaml
```

Reboots Beelink -- takes 2-3 minutes. Verify:

```bash
talosctl get discoveredvolumes \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig

talosctl read /proc/mounts \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig | grep hdd
```

### Step 5 -- Deploy local-path-provisioner via ArgoCD

```bash
kubectl apply -f apps/local-path-provisioner/argocd-application.yaml
kubectl get pods -n local-path-storage -o wide -w
```

Pod should schedule on the Omen.

### Step 6 -- Test PVC provisioning

```bash
kubectl apply -f apps/local-path-provisioner/test-pvc.yaml
kubectl logs -n default hdd-test-pod
# Expected: HDD mount working
kubectl delete -f apps/local-path-provisioner/test-pvc.yaml
```

### Step 7 -- Confirm StorageClass

```bash
kubectl get sc
# local-hdd should show (default)
```

### Step 8 -- etcd snapshot

```bash
mkdir -p ~/Kuber/snapshots
talosctl etcd snapshot ~/Kuber/snapshots/phase0-complete.snapshot \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig
```

**Phase 0 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| `sdb` not appearing after reboot | Patch did not apply cleanly | Run `talosctl dmesg -n 192.168.0.202` -- look for disk errors, re-apply patch |
| `/var/mnt/hdd` not in mounts | extraMounts block not applied | Check `talosctl get machineconfig -n 192.168.0.202` -- verify kubelet extraMounts is present |
| PVC stuck in `Pending` | WaitForFirstConsumer -- normal | Create a pod that uses the PVC. Binding is deferred until pod placement is decided |
| Provisioner pod not on Omen | nodeSelector mismatch | Verify `kubectl get node --show-labels` and check provisioner Deployment nodeSelector |
| etcd snapshot fails | etcd unhealthy | Run `talosctl -n 192.168.0.134 etcd members` to confirm etcd is healthy |

---

## Phase 1 -- Forgejo

### Step 1 -- Create SealedSecret for admin credentials

```bash
kubectl create secret generic forgejo-admin-secret \
  --from-literal=username=admin \
  --from-literal=password=<password> \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > \
  apps/forgejo/sealed-secret.yaml
```

### Step 2 -- Deploy via ArgoCD

```bash
kubectl apply -f apps/forgejo/argocd-application.yaml
argocd app sync forgejo
kubectl get pods -n forgejo -w
```

### Step 3 -- Verify PVC binding

```bash
kubectl get pvc -n forgejo
# Should show: Bound, storageClass: local-hdd, node: talos-v3h-4m1
```

### Step 4 -- Test SSH clone

```bash
git clone ssh://git@forgejo.homelab:2222/admin/test-repo.git
```

### Step 5 -- Configure ArgoCD webhook

In Forgejo UI: repo settings -> Webhooks -> Add webhook
- URL: `http://argocd-server.argocd.svc.cluster.local/api/webhook`
- Events: push

**Phase 1 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| Pod stuck in `Init:0/1` | PVC pending -- WaitForFirstConsumer | Check `kubectl describe pvc -n forgejo` -- pod scheduling triggers PVC binding |
| TLS cert not issuing | Cert-Manager ClusterIssuer not ready or DNS not resolving | Check `kubectl get certificaterequest -n forgejo` and verify `forgejo.homelab` resolves on Mac |
| SSH clone fails | NodePort not exposed or wrong port | Check `kubectl get svc -n forgejo` -- SSH service must be NodePort or LoadBalancer |
| Webhook not triggering ArgoCD | ArgoCD not reachable from Forgejo pod | Use full K8s DNS: `http://argocd-server.argocd.svc.cluster.local/api/webhook` |
| SealedSecret not decrypting | Wrong public key used during sealing | Fetch current key: `kubeseal --fetch-cert --controller-namespace sealed-secrets` |

---

## Phase 2 -- Authentik

### Step 1 -- Deploy

```bash
argocd app sync authentik
kubectl get pods -n authentik -w
```

Full startup takes 2-4 minutes. Both worker and server pods must be Running.

### Step 2 -- Initial bootstrap

```bash
kubectl logs -n authentik deployment/authentik-server | grep "Initial admin password"
```

Change it immediately via the Authentik UI at `https://auth.homelab`.

### Step 3 -- Create OIDC providers

In Authentik UI: Applications -> Providers -> Create -> OAuth2/OIDC. Create one each for Forgejo, ArgoCD, Grafana.

### Step 4 -- Create groups

Directory -> Groups -> Create: `platform-admin`, `developer`, `readonly`

### Step 5 -- Validate SSO

Test login to each app with a user assigned to each group.

**Phase 2 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| Authentik server CrashLoopBackOff | `SECRET_KEY` env var not set or PostgreSQL not ready | Check `kubectl logs -n authentik deployment/authentik-server` |
| OIDC redirect fails with `invalid_client` | Client ID or secret mismatch | Copy from Authentik provider page exactly -- no trailing spaces |
| ArgoCD login returns blank page | `issuer` URL format wrong | Must end with `/` -- `https://auth.homelab/application/o/argocd/` |
| Groups not propagating to ArgoCD RBAC | policy.csv group name case mismatch | Exact match required including case |

---

## Phase 3 -- MinIO

### Step 1 -- Seal credentials

```bash
kubectl create secret generic minio-credentials \
  --from-literal=rootUser=admin \
  --from-literal=rootPassword=<password> \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > \
  apps/minio/sealed-secret.yaml
```

### Step 2 -- Deploy

```bash
argocd app sync minio
kubectl get pods -n minio -w
```

### Step 3 -- Create buckets

```bash
mc alias set homelab https://minio.homelab admin <password>
mc mb homelab/pipeline-input
mc mb homelab/pipeline-output
mc mb homelab/pipeline-work
mc mb homelab/loki-chunks
mc ilm add --expiry-days 30 homelab/pipeline-work
```

### Step 4 -- Migrate Loki to MinIO backend

Update Loki Helm values with `s3forcepathstyle: true` and the MinIO endpoint, then sync via ArgoCD. Verify in Grafana -> Explore -> Loki returns results.

**Phase 3 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| MinIO pod OOMKilled | Default memory limit too low | Increase `resources.limits.memory` to at least `1Gi` |
| `mc: Unable to connect` | TLS cert not ready or DNS not resolving | Test `curl -v https://minio.homelab` from Mac |
| Loki stops showing logs after migration | S3 path style not set | MinIO requires `s3forcepathstyle: true` -- virtual-hosted style is not supported |
| PVC not binding to Beelink | StorageClass nodeSelector mismatch | Confirm Beelink has label `node-role=storage` |

---

## Phase 4 -- OPA Gatekeeper

### Step 1 -- Deploy

```bash
argocd app sync gatekeeper
kubectl get pods -n gatekeeper-system -w
```

### Step 2 -- Deploy ConstraintTemplates first, then Constraints

```bash
kubectl apply -f apps/gatekeeper/templates/
kubectl get constrainttemplates
kubectl apply -f apps/gatekeeper/constraints/
```

All constraints start in `warn` mode.

### Step 3 -- Monitor for 48 hours

```bash
kubectl get constraints -A -o json | jq '.items[] | {name: .metadata.name, violations: .status.violations}'
```

Fix all violations before switching to deny mode.

### Step 4 -- Flip to deny one at a time

Order: `require-labels` -> `require-resource-limits` -> `disallow-privileged` -> `approved-registries` -> `require-image-digest`

**Phase 4 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| Existing pods not affected | Gatekeeper only validates on admission | Constraints apply on next create/update -- rotate pods to force re-admission |
| Gatekeeper audit shows monitoring namespace violations | kube-prometheus-stack uses `:latest` or missing limits | Fix monitoring Helm values before flipping to deny |
| Webhook timeout causes pod creation to fail | Gatekeeper pods not running | Set `failurePolicy: Ignore` during initial rollout, switch to `Fail` after Gatekeeper is proven stable |

---

## Phase 5 -- Falco

### Step 1 -- Deploy

```bash
argocd app sync falco
kubectl get pods -n monitoring -l app.kubernetes.io/name=falco -o wide
# One pod per node expected
```

### Step 2 -- Apply custom rules

```bash
kubectl apply -f apps/falco/custom-rules-configmap.yaml
kubectl rollout restart daemonset/falco -n monitoring
```

### Step 3 -- Validate event generation

```bash
kubectl run audit-test --image=busybox:1.36 -n pipelines -it --rm -- /bin/sh
# Exit, then in Grafana: {job="falco"} |= "exec into pipeline pod"
# Event should appear within 5-10 seconds
```

**Phase 5 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| Falco pod stuck in `Init` or CrashLoop | Kernel module load attempt -- blocked on Talos | Set `driver.kind: ebpf` in Helm values -- eBPF is required for Talos |
| No events in Loki | Falcosidekick Loki endpoint wrong | Loki URL must be `http://loki.monitoring.svc:3100` |
| High CPU on Beelink | eBPF driver overhead | Cap Falco at `limits.cpu: 500m` on Beelink |

---

## Phase 6 -- Nextflow

### Step 1 -- Create namespace and RBAC

```bash
kubectl apply -f apps/nextflow/namespace.yaml
kubectl apply -f apps/nextflow/rbac.yaml
kubectl get sa,role,rolebinding -n pipelines
```

### Step 2 -- Create pipeline-work PVC

```bash
kubectl apply -f apps/nextflow/pipeline-work-pvc.yaml
kubectl get pvc -n pipelines
```

### Step 3 -- Run nf-core/rnaseq test profile

```bash
nextflow run nf-core/rnaseq \
  -profile test,k8s \
  -r 3.14.0 \
  --outdir s3://pipeline-output/rnaseq-test-$(date +%Y%m%d-%H%M) \
  -resume
```

### Step 4 -- Validate output and audit trail

```bash
mc ls homelab/pipeline-output/
# Grafana -> Loki: {job="falco"} | namespace="pipelines" | json
# Grafana -> K8s -> Pod resource usage: Omen elevated, Beelink disk IO only
```

**Phase 6 failure playbook:**

| Issue | Likely Cause | Resolution |
|---|---|---|
| Pipeline pods OOMKilled | Test profile still needs 4-6GB -- Omen must be the target | Verify `nodeSelector: node-role=infra` in Nextflow K8s config |
| `s3://pipeline-work` not accessible | MinIO endpoint unreachable from pipelines namespace | Test `kubectl run test --image=minio/mc -n pipelines --rm -it -- mc ls homelab/pipeline-work` |
| Nextflow fails to create pods | ServiceAccount missing permissions | Check `kubectl auth can-i create pods --as=system:serviceaccount:pipelines:nextflow-runner -n pipelines` |

---

## Phase 7 -- GxP Validation Docs

### Step 1 -- Create docs structure in Forgejo

```
validation-docs/
  IQ-Installation-Qualification.md
  OQ-Operational-Qualification.md
  PQ-Performance-Qualification.md
  SOPs/
    change-control.md
    audit-trail.md
    backup-recovery.md
    access-control.md
```

### Step 2 -- Fill IQ in real time during Phases 0-6

Record exact component versions, installation dates, and verification commands as each phase completes. Do not reconstruct after the fact.

### Step 3 -- Run OQ test scripts post-Phase 6

Run each scripted test case, record pass/fail with timestamp and operator name.

### Step 4 -- Run PQ benchmark twice

Record wall time, peak CPU and RAM from Grafana, MD5 checksum of MultiQC report. Verify checksums match across both runs.

### Step 5 -- Verify Change Control SOP is in active use

At least 3 PRs must have been merged through Forgejo before finalising PH-07. If not, the Change Control SOP cannot be claimed as a validated process.
