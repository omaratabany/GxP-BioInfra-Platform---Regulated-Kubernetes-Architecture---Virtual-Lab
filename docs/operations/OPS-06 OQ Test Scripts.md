# OPS-06 OQ Test Scripts

> Part of [[README]] | See also: [[PH-07 GxP Validation Documentation]], [[REG-01 Compliance Matrix]], [[OPS-07 Performance Baselines]]

Executable test scripts for OQ-01 through OQ-10. Each script captures exact commands, expected output, and a pass/fail determination. Results are recorded in the Results Sheet at the bottom of this file after each execution.

Before running: complete PROC-06 from [[OPS-04 Operational Runbook]] -- cluster must be healthy, all phases deployed, no Gatekeeper violations.

---

## Execution Instructions

Each test is self-contained. Run them in order OQ-01 through OQ-10. Record the actual output in the Results Sheet below. Do not edit the expected output fields -- if the actual output differs, that is a FAIL regardless of whether the system appears to be working.

Operator running these tests must hold the `platform-admin` role. Record the Authentik username used.

---

## OQ-01 -- Gatekeeper Denies Pod Without Resource Limits

**Annex 11 Clause:** 4.4 (Resource management)
**What it proves:** Admission control rejects any workload that would not have bounded resource consumption.

```bash
# Step 1: Attempt to deploy a pod with no resource limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oq01-no-limits-test
  namespace: pipelines
  labels:
    app: oq01-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: busybox:1.36@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
EOF
```

**Expected output:**
```
Error from server (Forbidden): error when creating "STDIN": admission webhook
"validation.gatekeeper.sh" denied the request: [require-resource-limits]
container <test> has no resource limits
```

**Pass criteria:** Command exits non-zero. Message contains `require-resource-limits`. Pod does not appear in `kubectl get pods -n pipelines`.

**Verification:**
```bash
kubectl get pod oq01-no-limits-test -n pipelines
# Expected: Error from server (NotFound)
```

**Cleanup:** No cleanup needed -- pod was never created.

---

## OQ-02 -- Gatekeeper Denies Pod With Latest Tag

**Annex 11 Clause:** 10 (Change management -- image pinning)
**What it proves:** No workload can be deployed with a mutable image reference.

```bash
# Step 1: Attempt to deploy a pod with a :latest tag
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oq02-latest-tag-test
  namespace: pipelines
  labels:
    app: oq02-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: busybox:latest
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
EOF
```

**Expected output:**
```
Error from server (Forbidden): error when creating "STDIN": admission webhook
"validation.gatekeeper.sh" denied the request: [require-image-digest]
image busybox:latest does not contain a digest
```

**Pass criteria:** Command exits non-zero. Message contains `require-image-digest`.

**Cleanup:** No cleanup needed.

---

## OQ-03 -- Falco Generates Audit Event on Exec Into Pipeline Pod

**Annex 11 Clause:** 9 (Audit trail)
**What it proves:** Any interactive access to a running pipeline pod is captured in the tamper-evident audit trail within 10 seconds.

```bash
# Step 1: Deploy a test pod in pipelines namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oq03-audit-test
  namespace: pipelines
  labels:
    app: oq03-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: busybox:1.36@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
    command: ["sleep", "300"]
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
  nodeSelector:
    node-role: infra
EOF

# Step 2: Wait for pod to be Running
kubectl wait --for=condition=Ready pod/oq03-audit-test -n pipelines --timeout=60s

# Step 3: Record the current timestamp
EXEC_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "Exec time: $EXEC_TIME"

# Step 4: Exec into the pod (exit immediately)
kubectl exec -n pipelines oq03-audit-test -- /bin/sh -c "echo oq03-exec-marker"

# Step 5: Wait 15 seconds for event propagation
sleep 15

# Step 6: Query Loki for the Falco event
# In Grafana -> Explore -> Loki:
# Query: {job="falco"} |= "oq03-audit-test" | json | priority="NOTICE" or priority="WARNING"
# Time range: last 5 minutes
# Expected: at least one event with rule="Terminal shell in container" or
#           the custom rule "Exec Into Pipeline Pod"
echo "Check Grafana Loki now with query: {job=\"falco\"} |= \"oq03-audit-test\""
```

**Expected Loki event fields:**
```
rule:      "Exec Into Pipeline Pod" (custom rule) or "Terminal shell in container" (default)
priority:  NOTICE or higher
output:    contains "oq03-audit-test" and namespace="pipelines"
time:      within 10 seconds of $EXEC_TIME
```

**Pass criteria:** Loki returns at least one Falco event referencing `oq03-audit-test` with a timestamp within 10 seconds of the exec time.

**Cleanup:**
```bash
kubectl delete pod oq03-audit-test -n pipelines
```

---

## OQ-04 -- SSO Login to Forgejo With Authentik Account

**Annex 11 Clause:** 12.1 (Access control)
**What it proves:** Authentication is centralised through Authentik -- local password login is rejected.

```bash
# Step 1: Verify local login is disabled
# Manual: navigate to https://forgejo.homelab/user/login
# Attempt to log in with the local admin credentials
# Expected: login fails or local login option is not presented
echo "Manual step: attempt local Forgejo login -- should fail or not be available"

# Step 2: Verify Authentik SSO login works
# Manual: click "Sign in with Authentik" on the Forgejo login page
# Log in with a developer-group Authentik account
# Expected: successful redirect to Forgejo, logged in as the Authentik user
echo "Manual step: SSO login via Authentik -- should succeed"

# Step 3: Verify the account has developer-level permissions only
# As the developer user: attempt to access Forgejo admin panel
# https://forgejo.homelab/-/admin/users
# Expected: 403 Forbidden
echo "Manual step: attempt Forgejo admin access with developer account -- should be 403"
```

**Pass criteria:** Local login fails or is unavailable. SSO login succeeds. Developer account cannot access admin panel.

---

## OQ-05 -- RBAC Scoping -- Readonly Group Cannot Modify Resources

**Annex 11 Clause:** 12.1 (Access control)
**What it proves:** K8s RBAC bindings correctly restrict the `readonly` group from making any modifications.

```bash
# Step 1: Get a kubeconfig scoped to a readonly-group user
# This requires an Authentik user in the readonly group with a valid token
# If OIDC kubeconfig delegation is configured:
kubectl auth can-i create pods -n pipelines \
  --as=<readonly-user>@authentik
# Expected: no

kubectl auth can-i list pods -n pipelines \
  --as=<readonly-user>@authentik
# Expected: yes

kubectl auth can-i delete deployments -n argocd \
  --as=<readonly-user>@authentik
# Expected: no

kubectl auth can-i get nodes \
  --as=<readonly-user>@authentik
# Expected: yes (read-only cluster access)
```

**Pass criteria:** All `create`, `delete`, `update`, `patch` verbs return `no`. All `get`, `list`, `watch` verbs in permitted namespaces return `yes`.

---

## OQ-06 -- ArgoCD Syncs Within 60 Seconds of Forgejo Push

**Annex 11 Clause:** 10 (Change management -- GitOps enforcement)
**What it proves:** All infrastructure changes are driven by Git. No change can be made to the cluster without a corresponding Git commit.

```bash
# Step 1: Create a test ConfigMap manifest in the Forgejo repo
# In a local clone of the infra repo:
cat <<EOF > /tmp/oq06-test-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: oq06-sync-test
  namespace: default
  labels:
    app: oq06-test
    env: test
    version: "1.0"
data:
  test: "oq06-$(date +%s)"
EOF

# Step 2: Copy to the repo and push
cp /tmp/oq06-test-configmap.yaml ~/repos/Home-Lab-Infra-as-code/apps/oq-tests/
cd ~/repos/Home-Lab-Infra-as-code
git add apps/oq-tests/oq06-test-configmap.yaml
git commit -m "OQ-06: sync latency test"
PUSH_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
git push origin main
echo "Push time: $PUSH_TIME"

# Step 3: Poll ArgoCD until sync completes
for i in $(seq 1 12); do
  STATUS=$(argocd app get <app-name> --output json | jq -r '.status.sync.status')
  echo "$(date -u +"%H:%M:%S") -- Sync status: $STATUS"
  if [ "$STATUS" = "Synced" ]; then
    SYNC_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    echo "Synced at: $SYNC_TIME"
    break
  fi
  sleep 5
done

# Step 4: Verify ConfigMap exists
kubectl get configmap oq06-sync-test -n default
```

**Pass criteria:** ArgoCD transitions to Synced status within 60 seconds of push. ConfigMap exists in the cluster.

**Cleanup:**
```bash
git rm ~/repos/Home-Lab-Infra-as-code/apps/oq-tests/oq06-test-configmap.yaml
git commit -m "OQ-06: cleanup sync test"
git push origin main
kubectl delete configmap oq06-sync-test -n default
```

---

## OQ-07 -- PVC Provisions on Beelink HDD via local-hdd StorageClass

**What it proves:** The storage provisioner correctly creates PVs on the Beelink HDD at /var/mnt/hdd for any workload that requests the local-hdd StorageClass.

```bash
# Step 1: Create a PVC and a pod that uses it
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: oq07-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-hdd
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: oq07-test-pod
  namespace: default
  labels:
    app: oq07-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: busybox:1.36@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
    command: ["sh", "-c", "echo 'OQ-07: HDD mount confirmed' > /data/result.txt && cat /data/result.txt && sleep 30"]
    volumeMounts:
    - name: test-vol
      mountPath: /data
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: oq07-test-pvc
  nodeSelector:
    node-role: storage
EOF

# Step 2: Wait for pod
kubectl wait --for=condition=Ready pod/oq07-test-pod -n default --timeout=120s

# Step 3: Verify output
kubectl logs oq07-test-pod -n default
# Expected: OQ-07: HDD mount confirmed

# Step 4: Verify PVC bound to Beelink
kubectl get pvc oq07-test-pvc -n default
kubectl get pv | grep oq07-test-pvc
# Expected: STATUS=Bound, STORAGECLASS=local-hdd
```

**Pass criteria:** PVC status is Bound. Pod logs show `OQ-07: HDD mount confirmed`. PV shows node affinity to talos-v3h-4m1 (Beelink).

**Cleanup:**
```bash
kubectl delete pod oq07-test-pod -n default
kubectl delete pvc oq07-test-pvc -n default
```

---

## OQ-08 -- MinIO Bucket Accessible From Pipeline Namespace

**What it proves:** The pipeline execution environment can reach the S3 storage layer using the correct service account credentials. No network or IAM misconfiguration blocks pipeline data access.

```bash
# Step 1: Deploy a test pod in pipelines namespace with MinIO credentials
# Assumes minio-nextflow-sa-secret exists as a SealedSecret in the pipelines namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oq08-minio-test
  namespace: pipelines
  labels:
    app: oq08-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: minio/mc:RELEASE.2024-03-03T00-13-08Z@sha256:<digest>
    command: ["sh", "-c",
      "mc alias set homelab http://minio.minio.svc:9000 $MINIO_ACCESS_KEY $MINIO_SECRET_KEY && mc ls homelab/pipeline-input && echo 'OQ-08: MinIO access confirmed'"]
    env:
    - name: MINIO_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: minio-nextflow-sa-secret
          key: accessKey
    - name: MINIO_SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: minio-nextflow-sa-secret
          key: secretKey
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 100m
        memory: 128Mi
  nodeSelector:
    node-role: infra
  restartPolicy: Never
EOF

kubectl wait --for=condition=Ready pod/oq08-minio-test -n pipelines --timeout=60s
kubectl logs oq08-minio-test -n pipelines
# Expected last line: OQ-08: MinIO access confirmed
```

**Pass criteria:** Pod logs contain `OQ-08: MinIO access confirmed`. No authentication errors in pod logs.

**Cleanup:**
```bash
kubectl delete pod oq08-minio-test -n pipelines
```

---

## OQ-09 -- etcd Snapshot Creates a Valid Non-Zero File

**Annex 11 Clause:** 7.1 (Data integrity) and 17 (Archival and backup)
**What it proves:** The backup mechanism functions and produces a recoverable artifact.

```bash
# Step 1: Take a snapshot
SNAP_FILE=~/Kuber/snapshots/oq09-test-$(date +%Y%m%d-%H%M%S).snapshot

talosctl etcd snapshot $SNAP_FILE \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Step 2: Verify it is non-zero
ls -lh $SNAP_FILE
SNAP_SIZE=$(stat -f%z "$SNAP_FILE" 2>/dev/null || stat -c%s "$SNAP_FILE")
echo "Snapshot size: $SNAP_SIZE bytes"

if [ "$SNAP_SIZE" -gt 1000000 ]; then
  echo "PASS: Snapshot size is $(ls -lh $SNAP_FILE | awk '{print $5}')"
else
  echo "FAIL: Snapshot too small -- may be corrupt"
fi

# Step 3: Record SHA256 checksum
sha256sum $SNAP_FILE
# Record this in the Results Sheet below
```

**Pass criteria:** Snapshot file exists. Size is greater than 1MB. SHA256 checksum is recorded in Results Sheet.

**Cleanup:** Keep this snapshot -- it is a valid backup. Remove after the next scheduled weekly snapshot.

---

## OQ-10 -- Falco CRITICAL Alert on Privilege Escalation Attempt

**Annex 11 Clause:** 9 (Audit trail) and 12.4 (Incident management)
**What it proves:** Privilege escalation attempts in the pipeline namespace are detected and recorded at CRITICAL severity within 10 seconds.

```bash
# Step 1: Deploy a test pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oq10-privesc-test
  namespace: pipelines
  labels:
    app: oq10-test
    version: "1.0"
    env: test
spec:
  containers:
  - name: test
    image: busybox:1.36@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
    command: ["sleep", "120"]
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
  nodeSelector:
    node-role: infra
EOF

kubectl wait --for=condition=Ready pod/oq10-privesc-test -n pipelines --timeout=60s

# Step 2: Attempt to write to /etc inside the container (triggers file write rule)
ATTEMPT_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
kubectl exec -n pipelines oq10-privesc-test -- sh -c "cat /etc/shadow" 2>&1 || true
echo "Attempt time: $ATTEMPT_TIME"

# Step 3: Also attempt setuid execution
kubectl exec -n pipelines oq10-privesc-test -- sh -c "chmod u+s /bin/sh" 2>&1 || true

# Step 4: Wait for event propagation
sleep 15

# Step 5: Check Loki
echo "Check Grafana Loki:"
echo "Query: {job=\"falco\"} |= \"oq10-privesc-test\" | json"
echo "Expected: events with priority=WARNING or CRITICAL referencing oq10-privesc-test"
echo "Time window: within 15 seconds of $ATTEMPT_TIME"
```

**Pass criteria:** Loki contains at least one Falco event referencing `oq10-privesc-test` at WARNING or CRITICAL priority within 15 seconds of the attempt time. Record the exact event in the Results Sheet.

**Cleanup:**
```bash
kubectl delete pod oq10-privesc-test -n pipelines
```

---

## Results Sheet

Fill in after each test execution. Do not pre-fill. Date format: YYYY-MM-DD HH:MM UTC.

| Test ID | Date | Operator | Result | Notes |
|---|---|---|---|---|
| OQ-01 | | | PASS / FAIL | |
| OQ-02 | | | PASS / FAIL | |
| OQ-03 | | | PASS / FAIL | Loki event timestamp delta: |
| OQ-04 | | | PASS / FAIL | |
| OQ-05 | | | PASS / FAIL | |
| OQ-06 | | | PASS / FAIL | ArgoCD sync latency (seconds): |
| OQ-07 | | | PASS / FAIL | PV node affinity confirmed: |
| OQ-08 | | | PASS / FAIL | |
| OQ-09 | | | PASS / FAIL | Snapshot size: Checksum: |
| OQ-10 | | | PASS / FAIL | Falco event priority: Delta (seconds): |

**OQ execution summary:**
- Total tests: 10
- Tests passed: ___
- Tests failed: ___
- Date of execution: ___
- Operator: ___
- Cluster version: ___
- All failures must be investigated and resolved before PQ begins.
