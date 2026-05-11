# SEC-03 Incident Response Playbook

> Part of [[README]] | See also: [[SEC-01 Security Architecture]], [[SEC-05 Network Security Policy]], [[OPS-05 Disaster Recovery Plan]]

Incident response procedures for this platform. Each scenario has a detection source, a severity classification, a containment procedure, and a recovery path. All incidents must be logged in [[OPS-03 Implementation Log]] regardless of severity.

---

## Severity Classification

| Severity | Definition | Response Time | Example |
|---|---|---|---|
| P1 -- Critical | Active breach or data loss in progress | Immediate | Privilege escalation detected in pipeline pod |
| P2 -- High | Confirmed security violation, no active breach | Within 1 hour | Gatekeeper violation on image digest |
| P3 -- Medium | Anomalous behaviour requiring investigation | Within 4 hours | Unexpected outbound connection from pipeline pod |
| P4 -- Low | Policy deviation or misconfiguration | Within 24 hours | Missing resource limits on a non-critical pod |

---

## Scenario 1 -- Privilege Escalation in Pipeline Pod

**Detection source:** Falco CRITICAL rule -- `Privilege Escalation Attempt`
**Severity:** P1

**Containment:**
```bash
# 1. Immediately cordon the node the pod is running on
kubectl cordon <node-name>

# 2. Identify the pod
kubectl get pods -n pipelines -o wide | grep <node-name>

# 3. Capture pod state before killing it
kubectl describe pod <pod-name> -n pipelines > ~/incident-$(date +%Y%m%d-%H%M)/pod-describe.txt
kubectl logs <pod-name> -n pipelines > ~/incident-$(date +%Y%m%d-%H%M)/pod-logs.txt

# 4. Kill the pod
kubectl delete pod <pod-name> -n pipelines --grace-period=0

# 5. Preserve Falco evidence
# In Grafana -> Loki: {job="falco", priority="CRITICAL"} -- export the full event log

# 6. Check if any files were written outside /tmp before the pod was killed
# Falco File Write rule will have captured this
```

**Investigation:**
```bash
# Check which image the pod was running
# Pull the image reference from the pod describe output
# Verify the digest matches the approved digest

# Check ArgoCD for any recent changes to the pipelines namespace
argocd app history <app-name>

# Check if the privilege escalation succeeded
kubectl get clusterrolebindings | grep pipeline
# Any new bindings added by the pod indicate a successful escalation
```

**Recovery:**
1. Uncordon the node after investigation: `kubectl uncordon <node-name>`
2. Document the full timeline in [[OPS-03 Implementation Log]]
3. If image was compromised: update the approved-registries constraint and rotate the image digest
4. If pipeline data was accessed: review the Falco file write events and assess what data was touched
5. Update the OQ to reflect the incident and re-run OQ-10

---

## Scenario 2 -- Unauthorised Exec Into Pipeline Pod

**Detection source:** Falco AUDIT rule -- `Exec Into Pipeline Pod`
**Severity:** P2 if unexpected user, P4 if operator performing maintenance

**Triage:**
```bash
# Check the Falco event for the user field
# {job="falco"} |= "exec into pipeline pod"
# If user=operator performing documented maintenance: P4, log and close
# If user is unknown or a service account: P2, escalate
```

**Containment (P2):**
```bash
# Identify the source of the exec
kubectl get events -n pipelines | grep exec

# Check if any data was exfiltrated
# Falco outbound connection rule will have fired if data was sent out

# Rotate any credentials the pod had access to
# The pod has access to MinIO credentials via env vars -- rotate those immediately
mc admin user remove homelab nextflow-sa
mc admin user add homelab nextflow-sa <new-password>
# Update the SealedSecret and sync via ArgoCD
```

---

## Scenario 3 -- Unexpected Outbound Connection From Pipeline

**Detection source:** Falco WARNING rule -- `Unexpected Outbound Connection from Pipeline`
**Severity:** P2

**Investigation:**
```bash
# Get the destination IP from the Falco event
# {job="falco"} |= "unexpected outbound connection"

# Resolve the destination
nslookup <dest-ip>

# Check if the pipeline process has a legitimate reason to call that address
# nf-core/rnaseq should only call: MinIO (minio.minio.svc), Loki (loki.monitoring.svc)
# Any external call is a P2 event

# Check if data was exfiltrated
# If the connection was successful and not immediately closed, treat as P1
```

**Containment:**
```bash
# Immediately delete the offending pod
kubectl delete pod <pod-name> -n pipelines --grace-period=0

# Apply an emergency NetworkPolicy to block all egress from the pipelines namespace
# This is a temporary measure until Cilium is deployed
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emergency-deny-egress
  namespace: pipelines
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress: []
EOF
# Note: This only works once Cilium is deployed -- Flannel ignores NetworkPolicy
```

---

## Scenario 4 -- Gatekeeper Admission Denial Spike

**Detection source:** Gatekeeper audit violations spike in Grafana
**Severity:** P3

**Investigation:**
```bash
# Find what is being denied
kubectl get events -A | grep "admission webhook"
kubectl get constraints -A -o json | \
  jq '.items[] | {name: .metadata.name, violations: .status.violations}'

# Determine if this is a legitimate new deployment failing policy
# or an existing workload that changed unexpectedly
argocd app diff <affected-app>
```

**Resolution:**
- If legitimate deployment failing: fix the manifest (add resource limits, pin image digest)
- If unexpected: check Git log for unauthorised changes to the manifest
- Document in [[OPS-03 Implementation Log]] with the constraint violated and resolution

---

## Scenario 5 -- Sealed Secrets Controller Key Loss

**Detection source:** SealedSecrets fail to decrypt after controller reinstallation
**Severity:** P1

**Prevention:**
The Sealed Secrets private key backup at `~/Kuber/sealed-secrets-key-backup.yaml` is the only recovery mechanism. This file must be encrypted and stored outside the cluster. See [[SEC-04 Secrets and Key Management]].

**Recovery:**
```bash
# Restore the key from backup
kubectl apply -f ~/Kuber/sealed-secrets-key-backup.yaml

# Verify decryption is working
kubectl get secret forgejo-admin-secret -n forgejo
# Should show the secret fields -- if error, key is not restored correctly
```

**If the key is permanently lost:** All SealedSecrets must be re-created from scratch. This means:
1. Re-generate all credentials (new passwords, new tokens)
2. Re-seal them with the new controller key
3. Redeploy all applications via ArgoCD
4. Update all IQ entries to reflect the new credential versions

---

## Scenario 6 -- etcd Data Corruption

**Detection source:** `kubectl get nodes` returns errors or all nodes show NotReady
**Severity:** P1

See [[OPS-05 Disaster Recovery Plan]] Scenario 1 for the full step-by-step etcd recovery procedure.

---

## Scenario 7 -- Beelink HDD Failure

**Detection source:** MinIO pods CrashLoop, PVCs fail to mount, disk IO errors in `talosctl dmesg`
**Severity:** P1

See [[OPS-05 Disaster Recovery Plan]] Scenario 2 for the full step-by-step storage recovery procedure.

---

## Incident Log Template

Every incident gets an entry in [[OPS-03 Implementation Log]] using this format:

```
### [DATE] -- INCIDENT: [SHORT DESCRIPTION]
**Severity:** P1 / P2 / P3 / P4
**Detection source:** Falco / Grafana alert / Manual observation
**Phase active:** PH-XX

**Timeline:**
- [TIME]: Incident detected
- [TIME]: Containment started
- [TIME]: Containment complete
- [TIME]: Investigation started
- [TIME]: Root cause identified
- [TIME]: Recovery complete

**Root cause:**

**Evidence collected:**
- Falco event log: [Grafana query and exported results]
- Pod describe: [file path]
- Affected data: [what was accessed or modified]

**Resolution:**

**Preventive action:**

**OQ tests re-run post-incident:**
```

---

## Contact and Escalation

This is a solo homelab project. Escalation path for reference when operating in a team context:

| Severity | Owner | Escalation |
|---|---|---|
| P1 | Operator on call | Platform lead within 15 minutes |
| P2 | Platform engineer | Platform lead within 1 hour |
| P3 | Platform engineer | Weekly review |
| P4 | Platform engineer | Sprint backlog |
