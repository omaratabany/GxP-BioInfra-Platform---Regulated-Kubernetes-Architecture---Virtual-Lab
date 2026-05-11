# OPS-04 Operational Runbook

> Part of [[README]] | See also: [[OPS-02 Reference Commands]], [[OPS-05 Disaster Recovery Plan]], [[SEC-03 Incident Response Playbook]]

Day-to-day operational procedures for maintaining the platform. Each procedure is a discrete task with a trigger, a checklist, and a verification step. Log completion in [[OPS-03 Implementation Log]].

---

## Daily Checks

These take under 5 minutes. Run each morning when the platform is in active use.

```bash
# 1. Cluster node health
kubectl get nodes
# Expected: both nodes Ready

# 2. Any pods not Running
kubectl get pods -A | grep -v Running | grep -v Completed
# Investigate any pod not in Running or Completed state

# 3. Storage capacity on Beelink HDD
kubectl exec -n minio <minio-pod> -- df -h /export
# Alert if usage > 80%

# 4. Any Gatekeeper violations introduced since yesterday
kubectl get constraints -A -o json | \
  jq '.items[] | select(.status.totalViolations > 0) | {name: .metadata.name, violations: .status.totalViolations}'

# 5. Falco alert summary
# Grafana -> Falco dashboard -> check for any CRITICAL events in last 24 hours
```

---

## Weekly Tasks

### WK-01 -- etcd Snapshot

**Trigger:** Weekly (Sundays) or before any significant change
**Time required:** 2 minutes

```bash
mkdir -p ~/Kuber/snapshots
SNAPSHOT_NAME="weekly-$(date +%Y%m%d).snapshot"

talosctl etcd snapshot ~/Kuber/snapshots/$SNAPSHOT_NAME \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Verify snapshot is non-zero size
ls -lh ~/Kuber/snapshots/$SNAPSHOT_NAME

# Rotate: keep only last 5 snapshots
ls -t ~/Kuber/snapshots/*.snapshot | tail -n +6 | xargs rm -f
```

**Verification:** File exists and size is > 0. Log in [[OPS-03 Implementation Log]].

---

### WK-02 -- MinIO Bucket Mirror

**Trigger:** Weekly (Sundays) or after a significant pipeline run
**Time required:** Variable (depends on data volume)

```bash
# Mirror pipeline output to external target
mc mirror homelab/pipeline-output <external-alias>/pipeline-output --overwrite

# Mirror loki-chunks (audit trail)
mc mirror homelab/loki-chunks <external-alias>/loki-chunks --overwrite

# Verify mirror completed without errors
mc diff homelab/pipeline-output <external-alias>/pipeline-output
```

**Verification:** `mc diff` shows no differences. Log in [[OPS-03 Implementation Log]].

---

### WK-03 -- Certificate Expiry Check

**Trigger:** Weekly
**Time required:** 1 minute

```bash
kubectl get certificates -A
# Check READY=True and confirm no certificates are approaching expiry
# Cert-Manager auto-renews 30 days before expiry -- any cert NOT ready is an alert
```

**Action if cert not renewing:** Check `kubectl describe certificate <name> -n <namespace>` for the reason. Common causes: ClusterIssuer misconfigured, DNS not resolving.

---

## Monthly Tasks

### MO-01 -- Secret Audit

**Trigger:** Monthly (first Monday)
**Time required:** 20 minutes
**Procedure:** Full audit as defined in [[SEC-04 Secrets and Key Management]] Secret Audit Procedure section.

Log results in [[OPS-03 Implementation Log]].

---

### MO-02 -- Hardening Verification

**Trigger:** Monthly or after any security incident
**Time required:** 15 minutes
**Procedure:** Run the full Hardening Verification Checklist from [[SEC-02 Component Hardening Guide]].

---

### MO-03 -- Risk Register Review

**Trigger:** Monthly or after any incident
**Procedure:** Review [[REG-03 Risk Register]] -- check if any residual risk ratings have changed, if any planned mitigations have been implemented, and if any new risks should be added.

---

### MO-04 -- Dependency and Version Review

**Trigger:** Monthly
**Time required:** 30 minutes

```bash
# Check for available updates to Helm charts
helm repo update
helm list -A
# For each release, check: helm search repo <chart> --versions | head -3
# Note any charts more than one minor version behind

# Check Talos version
talosctl -n 192.168.0.134 version
# Compare to latest at https://github.com/siderolabs/talos/releases

# Check nf-core/rnaseq latest release
# https://github.com/nf-core/rnaseq/releases
```

Record findings in [[MOD-02 Modularity and Dependency Map]] Platform Version Matrix.

---

## Operational Procedures

### PROC-01 -- User Onboarding

**When:** New user needs access to the platform
**Time required:** 15 minutes

```
1. Create user account in Authentik (Directory -> Users -> Create)
2. Assign to appropriate group: platform-admin, developer, or readonly
3. User activates account via Authentik invitation email
4. If MFA enforcement is active: user must enrol before first login
5. Verify access by having the user log in to:
   - Forgejo (https://forgejo.homelab)
   - ArgoCD (https://argocd.homelab)
   - Grafana (https://grafana.homelab)
6. Log the onboarding in OPS-03 Implementation Log:
   - User account name (not password)
   - Group assigned
   - Date
   - Operator who created the account
```

**Verification:** User can log in and access only what their group permits.

---

### PROC-02 -- User Offboarding

**When:** User leaves the team or no longer needs access
**Time required:** 5 minutes
**Urgency: HIGH -- complete within same day**

```bash
# 1. Disable the user in Authentik immediately
# Authentik Admin -> Directory -> Users -> select user -> Disable
# This immediately invalidates all active sessions and OIDC tokens

# 2. Verify session invalidation
# Log in attempt with the disabled user should fail immediately

# 3. Check if the user had any active SSH keys in Forgejo
# Forgejo Admin -> Users -> select user -> SSH Keys
# Remove any remaining SSH keys

# 4. Check if the user had any personal access tokens
# Forgejo Admin -> Users -> select user -> Applications
# Revoke any remaining tokens

# 5. Log in OPS-03 Implementation Log:
#    - User account disabled
#    - Date and time
#    - Operator who disabled
#    - Any active sessions found and invalidated
```

---

### PROC-03 -- Adding a New Pipeline

**When:** A new nf-core pipeline needs to be run on the platform

```
1. Check the pipeline is in the Gatekeeper approved-registries list
   - nf-core pipelines pull from quay.io/nf-core/ and biocontainers
   - Both registries are on the approved list

2. Pin the pipeline image to a specific version and digest:
   nextflow pull nf-core/<pipeline-name> -r <version>
   docker inspect quay.io/nf-core/<pipeline>:<version> --format='{{index .RepoDigests 0}}'

3. Update nextflow.config with the pinned container reference

4. Update the Gatekeeper approved-registries constraint if the new pipeline uses an additional registry
   - This requires a Forgejo PR

5. Run the pipeline with the test profile first:
   nextflow run nf-core/<pipeline-name> -profile test,k8s -r <version>

6. Verify Falco events are generated during the test run

7. If the test profile passes, document the pipeline in:
   - OPS-03 Implementation Log (new pipeline approved for use)
   - IQ (component added: nf-core/<pipeline-name> at version <x>)
```

---

### PROC-04 -- Responding to a Gatekeeper Violation Alert

**When:** Gatekeeper audit shows new violations

```bash
# 1. Identify what is violating
kubectl get constraints -A -o json | \
  jq '.items[] | select(.status.totalViolations > 0) | 
  {constraint: .metadata.name, violations: .status.violations}'

# 2. Determine if it is a legitimate workload that needs fixing
#    or an unexpected workload that should not exist

# 3a. Legitimate workload (e.g., missing resource limits on a new deployment)
#     Fix the Helm values, open a PR in Forgejo, ArgoCD syncs
#     The pod will be re-admitted after the fix and will pass

# 3b. Unexpected workload
#     This is a P3 incident -- follow SEC-03 Incident Response Playbook Scenario 4
#     Delete the offending pod immediately
#     Investigate how it was deployed outside of GitOps

# 4. Verify violation count drops to zero
kubectl get constraints -A | grep -c "0" 
```

---

### PROC-05 -- Rotating MinIO Service Account Credentials

**When:** 90-day rotation schedule or on suspicion of compromise

Follow the full procedure in [[SEC-04 Secrets and Key Management]] Rotation Procedure section.

Quick reference:
```bash
# Generate new credential
NEW_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)

# Re-seal and push via PR
# ArgoCD syncs new SealedSecret
# Update MinIO: mc admin user add homelab <sa-name> $NEW_PASSWORD
# Verify application function (Nextflow test, Loki log check)
# Log in OPS-03
```

---

### PROC-06 -- Pre-Phase Transition Checklist

**When:** Before starting a new phase (PH-00 through PH-07)

```bash
# 1. Take etcd snapshot
talosctl etcd snapshot ~/Kuber/snapshots/pre-phase-$(date +%Y%m%d).snapshot \
  --nodes 192.168.0.134 --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# 2. Verify all running pods are healthy
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v Terminating

# 3. Verify no Gatekeeper violations
kubectl get constraints -A -o json | jq '.items[].status.totalViolations' | \
  grep -v "^0$" && echo "VIOLATIONS FOUND -- resolve before proceeding"

# 4. Verify storage capacity
kubectl exec -n minio <pod> -- df -h /export
# Alert if > 75% used

# 5. Check certificate validity
kubectl get certificates -A | grep -v "True"
# Any not-Ready certs must be resolved first

# 6. Open the phase file in Obsidian and review the dependency requirements
# 7. Confirm all dependencies from the previous phase exit criteria are met

# 8. Log the pre-transition check in OPS-03 Implementation Log
```


---

## G-20 -- Homepage and Kubernetes Dashboard Removal

INF-03 identifies these as candidates for removal to recover RAM. This is the procedure.

**RAM to recover:** ~200-300Mi from Homepage, ~150-200Mi from Kubernetes Dashboard.

```bash
# Option A: Remove via ArgoCD (if deployed as ArgoCD Applications)
argocd app delete homepage --cascade
argocd app delete kubernetes-dashboard --cascade

# Option B: Remove via Helm directly
helm uninstall homepage -n homepage
helm uninstall kubernetes-dashboard -n kubernetes-dashboard

# Option C: Scale to zero (preserves the app definition if you want to bring it back)
kubectl scale deployment homepage -n homepage --replicas=0
kubectl scale deployment kubernetes-dashboard -n kubernetes-dashboard --replicas=0

# After removal: delete the namespaces if no longer needed
kubectl delete namespace homepage
kubectl delete namespace kubernetes-dashboard

# Remove the ArgoCD application manifests from the repo via Forgejo PR
# Delete: apps/homepage/ and apps/kubernetes-dashboard/ directories
# ArgoCD with prune: true will remove the remaining resources on next sync
```

**Verification:**
```bash
kubectl get pods -A | grep -E "homepage|kubernetes-dashboard"
# Expected: no output

kubectl top nodes
# Omen and Beelink should show reduced memory consumption
```

**This procedure should be run during Phase 0 or Phase 1 -- before deploying any new workloads.** Removing these frees RAM on the Omen that is needed for Authentik and Gatekeeper in later phases.

---

## PROC-07 -- Remove Homepage and Kubernetes Dashboard

**When:** Before Phase 3 to recover RAM on both nodes. INF-03 identifies this as a priority action.
**RAM recovered:** approximately 250-400Mi total.
**Time required:** 5 minutes.

```bash
# Option A: Delete via ArgoCD (removes the app and all resources)
argocd app delete homepage --cascade
argocd app delete kubernetes-dashboard --cascade

# Verify pods are gone
kubectl get pods -n homepage
kubectl get pods -n kubernetes-dashboard
# Expected: No resources found

# Option B: Scale to zero without deleting the app definition
kubectl scale deployment homepage -n homepage --replicas=0
kubectl scale deployment kubernetes-dashboard -n kubernetes-dashboard --replicas=0

# Verify the RAM was actually freed
kubectl describe node talos-asj-72z | grep -A 5 "Allocated resources"
kubectl describe node talos-v3h-4m1 | grep -A 5 "Allocated resources"
```

**After removal:** Update the README's "Currently Deployed Stack" section (or INF-01 if it references these) to reflect they are no longer running. Log in [[OPS-03 Implementation Log]].

**Note:** If the ArgoCD applications for Homepage and Kubernetes Dashboard are removed, ArgoCD will not try to re-deploy them. If only the pods are scaled to zero, ArgoCD `selfHeal: true` will restart them on the next sync. Use Option A for a permanent removal.
