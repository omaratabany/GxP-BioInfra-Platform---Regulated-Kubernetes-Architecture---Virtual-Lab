# OPS-05 Disaster Recovery Plan

> Part of [[README]] | See also: [[OPS-04 Operational Runbook]], [[SEC-03 Incident Response Playbook]], [[REG-03 Risk Register]]

Step-by-step recovery procedures for each failure scenario on this platform. Each scenario has a detection method, a recovery procedure, and a validation sequence. After recovery, the relevant OQ tests are re-run to confirm the platform is functioning correctly.

---

## Recovery Priority Order

In a scenario where multiple components fail simultaneously:

```
1. Restore Talos node health (if node is down)
2. Restore etcd (if API server is inaccessible)
3. Restore MinIO (audit trail and pipeline data)
4. Restore Forgejo (change control mechanism)
5. Restore ArgoCD (GitOps sync -- will pull all other apps from Forgejo)
6. Let ArgoCD reconcile Authentik, Gatekeeper, Falco, Nextflow
7. Validate via hardening verification checklist
```

ArgoCD handles steps 6-7 automatically once it has a healthy cluster and Forgejo access. The hard work is steps 1-5.

---

## Scenario 1 -- etcd Corruption or Loss

**Symptoms:** `kubectl get nodes` returns errors, API server unresponsive, Talos health check fails
**Risk:** R-02 in [[REG-03 Risk Register]]
**Time to recover:** 15-30 minutes

### Step 1 -- Verify etcd is the cause

```bash
# Check Talos health
talosctl -n 192.168.0.134 health

# Check etcd member status
talosctl -n 192.168.0.134 etcd members
# If this command hangs or errors, etcd is the problem
```

### Step 2 -- Stop the API server

On Talos, this is managed automatically -- if etcd is broken, the API server will fail on its own.

### Step 3 -- Find the most recent clean snapshot

```bash
ls -lt ~/Kuber/snapshots/*.snapshot
# Take the most recent file that is non-zero size
```

### Step 4 -- Restore etcd from snapshot

```bash
# Talos etcd restore is a bootstrap operation
# It requires bringing up a single-member etcd from the snapshot

talosctl etcd recover \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --filename ~/Kuber/snapshots/<snapshot-name>.snapshot
```

### Step 5 -- Verify cluster is healthy

```bash
kubectl get nodes
kubectl get pods -A
argocd app list
```

### Step 6 -- Assess data loss window

The snapshot captures the cluster state at the time it was taken. Any changes made after the most recent snapshot are lost. Check:
- ArgoCD application states (may need to re-sync)
- SealedSecrets (in Git -- no data loss)
- etcd-stored data (RBAC bindings, ConfigMaps, Secrets) -- may need to be re-applied

### Step 7 -- Re-run validation

```bash
# Run OQ-07, OQ-08 to verify storage is intact
# Run OQ-04, OQ-05 to verify access control is intact
# Run OQ-06 to verify GitOps sync is working
```

**Log in [[OPS-03 Implementation Log]]:** Date, snapshot used, data loss window, recovery duration, OQ results.

---

## Scenario 2 -- Beelink HDD Failure (Storage Data Loss)

**Symptoms:** MinIO pods in CrashLoop, PVCs fail to mount, `talosctl dmesg` shows disk I/O errors on Beelink
**Risk:** R-01 in [[REG-03 Risk Register]]
**Time to recover:** 2-4 hours (depending on data volume and external mirror bandwidth)

### Step 1 -- Confirm HDD failure

```bash
# Check disk errors in dmesg
talosctl dmesg \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig | grep -i -E "error|fail|i/o"

# Check disk health
talosctl -n 192.168.0.202 disks
# If sdb is not showing or showing errors, HDD failure is confirmed
```

### Step 2 -- Drain Beelink

```bash
kubectl drain talos-v3h-4m1 --ignore-daemonsets --delete-emptydir-data --force
```

### Step 3 -- Replace the HDD

Physical operation: replace the failed 320GB HDD on the Beelink with a new drive.

### Step 4 -- Re-apply HDD mount patch to Beelink

The new drive is /dev/sdb. The patch formats it and mounts it at /var/mnt/hdd:

```bash
talosctl patch machineconfig \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --patch @~/Kuber/patches/beelink-hdd.yaml
```

Wait for Beelink to reboot. Verify mount:

```bash
talosctl read /proc/mounts \
  --nodes 192.168.0.202 --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig | grep hdd
```

### Step 5 -- Restore MinIO data from external mirror

```bash
# Restore from external mirror
mc mirror <external-alias>/pipeline-output homelab/pipeline-output --overwrite
mc mirror <external-alias>/loki-chunks homelab/loki-chunks --overwrite

# Restore Forgejo data (if Forgejo repo was on the HDD)
# Forgejo PVC is on Beelink HDD -- Forgejo repos need to be re-cloned from GitHub (public mirror)
# or from any developer's local clone
```

### Step 6 -- Uncordon Beelink

```bash
kubectl uncordon talos-v3h-4m1
```

### Step 7 -- Verify all PVCs re-bind

```bash
kubectl get pvc -A
# All PVCs should show Bound status
```

### Step 8 -- Re-run validation

```bash
# OQ-07: PVC provisioning
# OQ-08: MinIO accessibility from pipeline pod
# Check Grafana -> Loki to confirm log ingestion resumed
```

**Calculate data loss window:** Time of last successful mirror to time of HDD failure. Document in [[OPS-03 Implementation Log]].

---

## Scenario 3 -- Omen Node Failure (Control Plane Down)

**Symptoms:** `kubectl get nodes` times out, API server unreachable, Talos health check fails on 192.168.0.134
**Risk:** R-02 in [[REG-03 Risk Register]]
**Time to recover:** 30-60 minutes (hardware-dependent)

### Step 1 -- Determine if it is a software or hardware issue

```bash
# Try reaching Talos API directly
talosctl -n 192.168.0.134 version
# If this works: K8s component issue, not hardware
# If this fails: hardware or OS issue
```

**Software issue (Talos API reachable):**

```bash
# Check K8s component health
talosctl -n 192.168.0.134 health
# Follow the health check output to identify which component failed

# Restart a failed component via Talos service management
talosctl -n 192.168.0.134 service restart kubelet
```

**Hardware issue (Talos API unreachable):**

Physical intervention required. If the Omen had a power or hardware failure:
1. Power cycle the Omen
2. Talos will auto-start all K8s components on boot
3. etcd will recover from its data directory if the OS disk is intact
4. If OS disk is also failed: restore from snapshot (Scenario 1)

### Step 2 -- Verify recovery

```bash
kubectl get nodes
# Both nodes should show Ready
kubectl get pods -A | grep -v Running | grep -v Completed
# Investigate any non-running pods
```

### Step 3 -- Check for in-flight pipeline interruption

If a Nextflow pipeline was running when the Omen went down, the pipeline executor (head pod) was also on the Omen. The run is lost.

```bash
# Check MinIO for partial output
mc ls homelab/pipeline-output/

# Check MinIO for the work directory
mc ls homelab/pipeline-work/

# If partial output exists: clean it up and restart the run
mc rm --recursive --force homelab/pipeline-work/<run-id>/
# Restart: nextflow run nf-core/rnaseq ... -resume
# Note: -resume will re-use any completed task outputs from pipeline-work
```

---

## Scenario 4 -- Sealed Secrets Controller Deleted

**Symptoms:** New pods fail to start with `Error: secret not found`, SealedSecrets are not being decrypted
**Risk:** R-06 in [[REG-03 Risk Register]]
**Time to recover:** 15 minutes (if backup exists)

### Step 1 -- Confirm controller is missing

```bash
kubectl get pods -n sealed-secrets
# If no pods: controller was deleted
kubectl get sealedsecrets -A
# SealedSecrets still exist but are not being decrypted
```

### Step 2 -- Restore controller and key from backup

```bash
# Decrypt the backup
gpg --decrypt ~/Kuber/sealed-secrets-key-backup.yaml.gpg > /tmp/sealed-secrets-key.yaml

# Re-apply the key first (before the controller starts)
kubectl apply -f /tmp/sealed-secrets-key.yaml

# Destroy the plaintext key
rm -f /tmp/sealed-secrets-key.yaml

# Reinstall the Sealed Secrets controller via Helm
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --create-namespace
```

### Step 3 -- Verify decryption

```bash
# Wait for the controller to start
kubectl get pods -n sealed-secrets -w

# Verify a known SealedSecret is now decrypted
kubectl get secret forgejo-admin-secret -n forgejo
# Should return the secret fields
```

**If the backup key is also lost:** All SealedSecrets must be re-created from scratch. Generate new credentials for all services, re-seal them with the new controller key, commit to Forgejo, and sync via ArgoCD. This is a full platform credential rotation.

---

## Scenario 5 -- ArgoCD Out of Sync After Cluster Event

**Symptoms:** ArgoCD shows applications as OutOfSync, apps are in Unknown state, sync fails
**Time to recover:** 10 minutes

```bash
# Check ArgoCD server is running
kubectl get pods -n argocd

# Force refresh all applications
argocd app list -o name | xargs -I{} argocd app get {} --refresh

# Check for sync errors
argocd app list | grep -v Synced

# For each out-of-sync app, check the diff
argocd app diff <app-name>

# If the diff is expected (ArgoCD just lost its state): hard refresh and sync
argocd app sync <app-name> --force

# If the diff shows unexpected changes: investigate before syncing
# Follow PROC-04 from OPS-04 Operational Runbook
```

---

## Post-Recovery Validation Sequence

After any recovery scenario, run these in order:

```bash
# 1. Cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# 2. Storage health
kubectl get pvc -A | grep -v Bound
kubectl exec -n minio <pod> -- mc ping homelab

# 3. GitOps health
argocd app list | grep -v Synced

# 4. Identity health
# Log in to Forgejo via Authentik in browser -- verify SSO works

# 5. Policy enforcement health
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates | wc -l

# 6. Audit trail health
# Grafana -> Loki -> query {job="falco"} -> verify recent events appear

# 7. Run abbreviated OQ (core tests)
# OQ-03: exec into pipeline pod -> Falco event in Loki
# OQ-04: Forgejo login via Authentik
# OQ-07: PVC binding
# OQ-08: MinIO from pipeline pod
# OQ-09: etcd snapshot creation
```

**Log all recovery actions and OQ results in [[OPS-03 Implementation Log]].**
