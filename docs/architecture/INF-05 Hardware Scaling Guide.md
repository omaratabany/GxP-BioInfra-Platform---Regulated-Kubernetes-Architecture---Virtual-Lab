# INF-05 Hardware Scaling Guide

> Part of [[README]] | See also: [[INF-03 Infrastructure Analysis]], [[INF-04 System Tier Comparison]], [[OPS-01 Build Instructions]]

How this platform adapts to any hardware tier. Every component has a known minimum, a known optimal, and a known ceiling. This covers how to identify which tier applies, how to configure for that tier, and how to migrate between tiers without rebuilding from scratch.

---

## How to Identify Your Tier

```bash
kubectl describe node <node-name>
kubectl top nodes
talosctl get hardwareinfo -n <node-ip> --talosconfig <path>
```

| Hardware | Tier | Expected Capability |
|---|---|---|
| 1 node, 4 cores, 8GB RAM, 100GB disk | Minimal | Test profiles only, no concurrent runs, no HA |
| 1 node, 8+ cores, 16GB+ RAM, 200GB+ disk | Single-node comfortable | Full stack with monitoring, test profiles, dev use |
| 2 nodes, 12+ cores total, 20GB+ RAM, 300GB+ storage | Current (my setup) | Full GxP stack, test profiles, 1-2 concurrent runs |
| 3+ nodes, 48+ cores, 128GB+ RAM, 1TB+ storage | Production entry | Real genomics data, multiple concurrent runs |
| 8+ nodes, 200+ cores, 512GB+ RAM, 10TB+ NVMe | Ideal | Full production regulated environment |

---

## Tier 1 -- Minimal (Single Node)

One machine, untainted control plane, everything co-located. Storage on local disk. No compute/storage separation.

**Target hardware:** 8 cores / 16GB RAM / 200GB SSD minimum

### Components to skip or defer

| Component | Action | Reason |
|---|---|---|
| Authentik | Skip -- use ArgoCD basic auth | ~1.5GB RAM for Authentik + PG + Redis is too heavy on 8GB |
| OPA Gatekeeper | Deploy warn-only, no deny mode | Misconfigured deny on single node can lock you out |
| Falco | Optional -- basic rules only | eBPF overhead on 4 cores is noticeable |
| MinIO | Required | Loki needs an S3 backend -- MinIO standalone on 200GB works |

### Single-node StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  nodePath: /var/lib/local-path-provisioner
```

### Node labels for single-node

```bash
kubectl label node <node-name> node-role=infra
kubectl label node <node-name> node-role=storage --overwrite
```

Both labels on one node lets all workloads schedule without nodeSelector conflicts. When a second node arrives, relabel and migrate storage workloads.

### Nextflow on single-node

```groovy
k8s {
  namespace = 'pipelines'
  serviceAccount = 'nextflow-runner'
  storageClaimName = 'pipeline-work-pvc'
  workDir = 's3://pipeline-work'
  // No nodeSelector -- single node, everything goes here
}
```

Keep `process { cpus = 1; memory = '2 GB' }` as defaults. No concurrent runs.

### Resource limits tuned for single-node

Scale all platform component requests and limits down 40-50% compared to current-tier values to leave 6-8GB headroom for pipeline pods.

---

## Tier 2 -- Current (Two-Node Setup)

Two nodes: Omen for compute and control plane, Beelink for storage. Full platform stack. This is fully documented throughout [[INF-01 Infrastructure Baseline]], [[INF-03 Infrastructure Analysis]], and [[OPS-01 Build Instructions]].

### nodeSelector patterns I use

```yaml
# Storage workloads (MinIO, Loki, Forgejo PVC)
nodeSelector:
  node-role: storage

# Compute workloads (Nextflow workers, Authentik, ArgoCD)
nodeSelector:
  node-role: infra
```

In Helm values:
```yaml
# MinIO
nodeSelector:
  node-role: storage

# Nextflow executor config
k8s:
  pod:
    - nodeSelector: 'node-role=infra'
```

### Adding a third node at this tier

```bash
# Install Talos on new node, then label
kubectl label node <new-node> node-role=worker
```

If the new node has significant RAM: add `node-role=infra` label -- Nextflow will schedule compute pods on both Omen and the new node.

If the new node has significant storage: relabel for storage and begin migrating PVCs from Beelink HDD to the new node's storage.

---

## Tier 3 -- Scaling to Production

### Control Plane -- Single to HA

Going from one to three control plane nodes is a Talos operation. Reuse existing cluster secrets -- do NOT regenerate them.

```bash
# Generate configs for nodes 2 and 3 using existing secrets
talosctl gen config <cluster-name> https://<vip>:6443 \
  --with-secrets <existing-secrets-file>

# Apply to new nodes
talosctl apply-config --insecure \
  --nodes <new-cp-ip> \
  --file controlplane.yaml

# Join etcd
talosctl etcd member add <new-cp-ip>
```

Use a VIP for the control plane endpoint so `kubeconfig` and `talosconfig` endpoints do not change when nodes are added or removed:

```yaml
# In Talos machineconfig for control plane nodes
machine:
  network:
    interfaces:
      - interface: eth0
        vip:
          ip: 192.168.0.220
```

### Worker Node Expansion

```bash
talosctl apply-config --insecure \
  --nodes <worker-ip> \
  --file worker.yaml

kubectl label node <worker> node-role=compute
```

### Storage -- local-path to Longhorn

When multiple nodes each have local disks and replicated block storage is needed:

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-replicated
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

Migrate PVCs from `local-hdd` to `longhorn-replicated` by scaling workloads to zero, taking a data snapshot, creating a new PVC, restoring data, and scaling back up.

### MinIO Standalone to Distributed

Requires exactly 4, 8, 16, or 32 drives across nodes. With 4 storage nodes each with one HDD:

```yaml
mode: distributed

statefulset:
  replicaCount: 4
  drivesPerNode: 1

persistence:
  storageClass: local-hdd
  size: 200Gi
```

Backup all buckets before switching modes -- distributed MinIO is not backward compatible with standalone data layout.

### CNI -- Flannel to Cilium

```bash
# Remove Flannel
kubectl delete -f /run/flannel/flannel.yaml

# Install Cilium
cilium install --version 1.15.0

# Verify
cilium status
cilium connectivity test
```

On Talos, update `cluster.network.cni.name: none` in machineconfig before installing Cilium. After Cilium is running, NetworkPolicy objects that Flannel ignored start being enforced -- review all existing NetworkPolicy objects before migration.

---

## Dynamic Configuration Without Rebuilding

### Changing node assignment

```bash
# Relabel a node
kubectl label node talos-v3h-4m1 node-role=compute --overwrite

# Drain first if removing a label that pins things to the node
kubectl drain talos-v3h-4m1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon talos-v3h-4m1
```

Label changes affect future scheduling only. Running pods are not evicted.

### Scaling platform components

```bash
# Scale a deployment
kubectl scale deployment authentik-server -n authentik --replicas=2

# Update resource limits via Helm (always through Forgejo PR -> ArgoCD sync)
helm upgrade authentik authentik/authentik \
  --namespace authentik \
  --set resources.server.limits.memory=2Gi \
  --reuse-values
```

### Scaling pipeline throughput

```groovy
// Increase as more RAM becomes available
process {
  cpus = 4
  memory = '8 GB'

  withLabel: process_high {
    cpus = 8
    memory = '32 GB'  // Only viable on 128GB RAM workers
  }
}
```

```yaml
# Increase ResourceQuota as cluster grows
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "80Gi"
    pods: "100"
```

### Re-enabling deferred components

**Authentik (after running on basic auth):** Deploy, configure OIDC providers, test SSO, remove local passwords.

**Gatekeeper (after skipping it):** Deploy in `warn` mode, fix all audit violations, then flip to `deny` one constraint at a time with 48 hours between each.

**Falco (after running without it):** Deploy DaemonSet with eBPF driver, apply custom rules ConfigMap, verify events in Loki. No impact on running workloads -- Falco is read-only.

---

## Migration Path Summary

| From | To | Key Actions |
|---|---|---|
| Minimal (1 node) | Current (2 nodes) | Add node, relabel, deploy local-path-provisioner pinned to HDD, migrate storage PVCs |
| Current (2 nodes) | 3-node HA | Add 2 CP nodes, configure VIP, promote etcd members |
| Single storage | Longhorn | Install Longhorn, create replicated StorageClass, migrate PVCs one at a time |
| Longhorn | Ceph (Rook) | Requires 3+ storage nodes -- install Rook operator, create CephCluster, migrate PVCs |
| MinIO standalone | MinIO distributed | Backup all buckets, requires 4 drives, redeploy in distributed mode, restore |
| Flannel | Cilium | Minimise cluster load, remove Flannel, install Cilium, verify NetworkPolicy enforcement |
| Sealed Secrets | Vault + External Secrets | Deploy Vault, migrate secrets one namespace at a time |

No step requires rebuilding the cluster from scratch. Each migration is a controlled upgrade of one layer.
