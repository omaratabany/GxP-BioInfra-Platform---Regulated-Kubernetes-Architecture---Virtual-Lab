# PRJ-04 CKA Study Alignment

> Part of [[README]] | See also: [[PRJ-02 Phase Map and Schedule]], [[OPS-01 Build Instructions]]

Maps each CKA exam domain to the specific project phase that reinforces it. For each domain: the exam objectives that matter, the exact kubectl commands the exam tests, and how the corresponding project phase is a live exercise of those skills. Building real things that break in real ways is better than practising in a sandbox.

---

## CKA Exam Domain Weights (Current)

| Domain | Weight |
|---|---|
| Cluster Architecture, Installation and Configuration | 25% |
| Workloads and Scheduling | 15% |
| Services and Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

Troubleshooting is the heaviest domain and the one most candidates underestimate. This project builds genuine troubleshooting muscle because every phase eventually hits a real problem that needs a real fix.

---

## Week 1-2 -- Cluster Architecture, etcd, RBAC (Phase 0)

**CKA exam objectives:**
- Manage a highly available Kubernetes cluster (understand etcd role, backup, restore)
- Use kubeadm to install a cluster
- Manage role-based access controls
- Upgrade a Kubernetes cluster

**What Phase 0 reinforces:**

| Exam skill | Phase 0 equivalent |
|---|---|
| etcd backup and restore | `talosctl etcd snapshot` -- same concept, Talos-specific API |
| Node roles and taints | Removing the control-plane taint from Omen: `kubectl taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-` |
| Node labels | `kubectl label node talos-asj-72z node-role=infra` |
| Understanding etcd quorum | Why a single-node control plane is a risk (R-02 in REG-03) |
| Cluster health verification | `kubectl get nodes`, `talosctl health` -- equivalent to post-kubeadm verification |

**Key kubectl commands for the CKA on this domain:**
```bash
# Taint management
kubectl taint nodes <node> <key>:<effect>
kubectl taint nodes <node> <key>:<effect>-     # remove a taint

# Label management
kubectl label node <node> <key>=<value>
kubectl label node <node> <key>-               # remove a label
kubectl get nodes --show-labels

# Describe for debugging
kubectl describe node <node>

# etcd snapshot (kubeadm clusters use etcdctl, not talosctl -- know both)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Practice exercise tied to this project:**
Remove the control-plane taint, verify workloads schedule on Omen, then deliberately drain Omen and confirm workloads move (or fail gracefully). This is direct exam practice.

---

## Week 3 -- Workloads and Scheduling (Phase 1 -- Forgejo)

**CKA exam objectives:**
- Understand Deployments and how to perform rolling updates and rollbacks
- Use ConfigMaps and Secrets to configure applications
- Know how to scale applications
- Understand the primitives used to create robust, self-healing application deployments

**What Phase 1 reinforces:**

| Exam skill | Phase 1 equivalent |
|---|---|
| StatefulSet vs Deployment | Forgejo is deployed as a StatefulSet (has stable network identity and PVC) |
| Secrets and ConfigMaps | SealedSecret -> decrypted K8s Secret -> mounted as env var in Forgejo |
| Liveness and readiness probes | Forgejo Helm chart configures HTTP probes -- check what they do |
| Init containers | Many Helm charts use init containers for DB migrations -- understand the pattern |
| Resource requests and limits | Required by Gatekeeper -- every container must have both |
| Rolling update strategy | `kubectl rollout status`, `kubectl rollout history`, `kubectl rollout undo` |

**Key kubectl commands for the CKA on this domain:**
```bash
# Deployment management
kubectl create deployment <name> --image=<image> --replicas=3
kubectl set image deployment/<name> <container>=<new-image>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2

# Scaling
kubectl scale deployment <name> --replicas=5

# ConfigMaps
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.properties

# Secrets (plain -- then you would seal these in real usage)
kubectl create secret generic <name> --from-literal=password=hunter2

# StatefulSet specific
kubectl get statefulset -n forgejo
kubectl describe statefulset forgejo -n forgejo
```

**Practice exercise tied to this project:**
Deliberately misconfigure the Forgejo Helm values (wrong image tag, missing env var) and observe the rollout fail. Use `kubectl rollout undo` to recover. This is exactly what the CKA tests.

---

## Week 4 -- Scheduling and Affinity (Phase 2 -- Authentik)

**CKA exam objectives:**
- Configure scheduling with nodeSelector, affinity, anti-affinity
- Understand resource limits and how the scheduler uses them
- Use DaemonSets
- Understand how taints and tolerations work

**What Phase 2 reinforces:**

| Exam skill | Phase 2 equivalent |
|---|---|
| nodeSelector | Every stateful workload uses `nodeSelector: node-role: storage` to land on Beelink |
| Resource requests and limits | Authentik PostgreSQL and the Authentik worker have very different resource profiles -- sizing them correctly is a scheduling exercise |
| Pod affinity | Authentik server and worker should prefer the same node for latency -- understand podAffinity |
| Priority classes | Understanding why platform services should not be evicted when pipelines run hot |

**Key kubectl commands for the CKA on this domain:**
```bash
# View scheduling decisions
kubectl get pods -o wide
kubectl describe pod <pod-name> | grep -A 5 "Node:"

# Affinity rules (written in pod spec, understand the syntax)
# nodeSelector (simple):
spec:
  nodeSelector:
    node-role: infra

# nodeAffinity (complex -- understand required vs preferred):
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role
            operator: In
            values: [storage]

# Taints and tolerations
kubectl taint nodes <node> dedicated=storage:NoSchedule
# Pod toleration:
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: storage
    effect: NoSchedule
```

**Practice exercise tied to this project:**
Re-apply the control-plane taint to Omen and verify the scheduling behaviour changes. Then test adding a toleration to a specific deployment so it can still schedule on Omen. This is a direct CKA scenario.

---

## Week 5 -- Networking (Phase 3 -- MinIO)

**CKA exam objectives:**
- Understand cluster networking (CNI basics)
- Deploy and configure network policies
- Understand Services (ClusterIP, NodePort, LoadBalancer)
- Know how Ingress works
- Configure CoreDNS

**What Phase 3 reinforces:**

| Exam skill | Phase 3 equivalent |
|---|---|
| Service types | MinIO needs both ClusterIP (for pod-to-pod S3 access) and LoadBalancer (for the console via Ingress) |
| Ingress configuration | MinIO console Ingress with TLS annotation and path-based routing |
| DNS resolution | `minio.minio.svc` -- understanding K8s DNS naming |
| NetworkPolicy (structure) | The pre-written NetworkPolicy YAML in SEC-05 is CKA exam format -- understand it even if Flannel doesn't enforce it |
| EndpointSlices | Understand how a Service discovers which pods to route to |

**Key kubectl commands for the CKA on this domain:**
```bash
# Service inspection
kubectl get svc -A
kubectl describe svc minio -n minio

# DNS debugging from within a pod
kubectl run dns-test --image=busybox:1.36 --rm -it -- /bin/sh
# Inside: nslookup minio.minio.svc.cluster.local
#          wget -qO- http://minio.minio.svc:9000

# NetworkPolicy (know the structure cold)
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <namespace>

# Ingress
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>

# Port forwarding for debugging
kubectl port-forward -n minio svc/minio 9000:9000
```

**Practice exercise tied to this project:**
Write a NetworkPolicy that allows only the pipelines namespace to reach MinIO port 9000. Apply it and verify it works at the logical level (even if Flannel ignores it). Check it against the pattern in [[SEC-05 Network Security Policy]].

---

## Week 6 -- Storage (Phase 4 -- OPA Gatekeeper)

**CKA exam objectives:**
- Understand persistent storage (PV, PVC, StorageClass)
- Configure applications with persistent storage
- Understand volume access modes
- Know how to expand volumes

**What Phase 4 reinforces (and Phase 0 groundwork):**

| Exam skill | Phase 0/4 equivalent |
|---|---|
| StorageClass configuration | `local-hdd` StorageClass definition -- `WaitForFirstConsumer`, `Retain` reclaim policy |
| PV/PVC lifecycle | Creating the test PVC, binding it, mounting it, verifying data persists after pod restart |
| Access modes | ReadWriteOnce vs ReadWriteMany -- local-hdd only supports RWO |
| Static vs dynamic provisioning | local-path-provisioner is dynamic -- understand the difference from static PV |
| Volume expansion | Understand the `allowVolumeExpansion: true` field and when it applies |

**Key kubectl commands for the CKA on this domain:**
```bash
# Storage class
kubectl get sc
kubectl describe sc local-hdd

# PV/PVC
kubectl get pv,pvc -A
kubectl describe pvc <name> -n <namespace>

# Check where a PVC is bound
kubectl get pvc -n minio -o jsonpath='{.items[].spec.volumeName}'
kubectl get pv <volume-name> -o jsonpath='{.spec.nodeAffinity}'

# Create PVC imperatively (for exam speed)
kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-hdd
  resources:
    requests:
      storage: 1Gi
EOF
```

**Practice exercise tied to this project:**
Delete a running pod that has a PVC. Observe that the PV shows `Released` state. Verify the data is still on disk. Recreate the pod and verify data persists. This is the exact exam scenario for understanding reclaim policies.

---

## Week 7 -- RBAC and Troubleshooting (Phase 5/6 -- Falco + Nextflow)

**CKA exam objectives:**
- Create and configure RBAC (Roles, ClusterRoles, RoleBindings, ClusterRoleBindings)
- Evaluate cluster and node logging
- Monitor applications
- Use kubectl to debug cluster issues
- Troubleshoot application failure, cluster component failure, networking

**What Phase 5/6 reinforces:**

| Exam skill | Phase 5/6 equivalent |
|---|---|
| Creating Roles and RoleBindings | nextflow-runner Role and RoleBinding -- write it from scratch |
| ServiceAccount token projection | nextflow-runner SA projected into pipeline pods |
| `kubectl auth can-i` | Verifying the nextflow-runner SA has exactly the permissions it needs |
| Reading pod logs | `kubectl logs -n monitoring <falco-pod>` -- standard exam troubleshooting |
| DaemonSets | Falco runs as a DaemonSet -- understand why and what happens when a node is added |
| Events | `kubectl get events -n pipelines --sort-by=.lastTimestamp` for pipeline debugging |

**Key kubectl commands for the CKA on this domain:**
```bash
# RBAC
kubectl create role <name> --verb=get,list,watch --resource=pods -n pipelines
kubectl create rolebinding <name> --role=<role> --serviceaccount=pipelines:nextflow-runner -n pipelines

# Verify permissions
kubectl auth can-i create pods -n pipelines \
  --as=system:serviceaccount:pipelines:nextflow-runner

# Check all permissions for a SA
kubectl auth can-i --list \
  --as=system:serviceaccount:pipelines:nextflow-runner -n pipelines

# Log collection
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous          # previous container instance
kubectl logs <pod> -n <namespace> -c <container>      # specific container
kubectl logs <pod> -n <namespace> --since=1h          # last hour

# Debugging
kubectl describe pod <pod> -n <namespace>             # events section is key
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl exec -n <namespace> <pod> -- <command>        # test from inside
```

**Practice exercise tied to this project:**
Deliberately remove a permission from the nextflow-runner Role (e.g., remove `pods/log: get`). Run a pipeline and observe it fail. Use `kubectl auth can-i` and `kubectl get events` to diagnose. Re-add the permission and confirm recovery. This is a textbook CKA troubleshooting scenario.

---

## Week 8 -- Exam Simulation

By this point, every CKA domain has been exercised through real implementation. The final week is speed practice -- all the same concepts but under time pressure.

**Recommended exam sim approach:**

```bash
# Set up kubectl aliases the way the exam allows
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Practice generating YAML fast
k run pod1 --image=nginx $do > /tmp/pod1.yaml
k create deployment dep1 --image=nginx --replicas=3 $do > /tmp/dep1.yaml
k create service clusterip svc1 --tcp=80:80 $do > /tmp/svc1.yaml
k create role role1 --verb=get,list --resource=pods $do > /tmp/role1.yaml
k create rolebinding rb1 --role=role1 --serviceaccount=default:sa1 $do > /tmp/rb1.yaml
```

**Speed benchmarks to hit before the exam:**
- Deploy a pod with resource limits, nodeSelector, and a PVC: under 4 minutes
- Create a Role and RoleBinding for a ServiceAccount: under 2 minutes
- Take an etcd snapshot: under 1 minute
- Write a NetworkPolicy that allows ingress from one namespace only: under 3 minutes
- Troubleshoot a pod stuck in Pending state: under 3 minutes

---

## Topics the Project Does NOT Cover (Study Separately)

| CKA Topic | Why Not Covered | How to Practice |
|---|---|---|
| kubeadm cluster install | Using Talos, not kubeadm | Use a kubeadm VM on CachyOS for this |
| Kubernetes upgrade via kubeadm | Talos handles upgrades differently | kubeadm upgrade practice scenario |
| Multi-master etcd cluster | Single CP node | Understand the theory; practice in simulator |
| NetworkPolicy enforcement | Flannel does not enforce | Write policies mentally; test after Cilium migration |
| kubectl top | Requires metrics-server | metrics-server is in the kube-prometheus-stack -- verify it is installed |
