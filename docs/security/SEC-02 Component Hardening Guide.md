# SEC-02 Component Hardening Guide

> Part of [[README]] | See also: [[SEC-01 Security Architecture]], [[ADR-00 Decision Log]], [[OPS-01 Build Instructions]]

Specific hardening steps for every component in the stack. Applied during each phase. Each step references the security control it implements and the threat it mitigates.

---

## Talos OS (Baseline -- All Phases)

Talos is hardened by design. These are the explicit confirmations to make after deployment.

```bash
# Confirm no SSH access exists
talosctl -n 192.168.0.134 get machineconfig | grep ssh
# Expected: no output -- SSH is not a Talos concept

# Confirm API is only accessible on the internal endpoint
talosctl -n 192.168.0.134 get endpoints
# Should show only internal cluster IPs, not public

# Confirm kernel modules are restricted
talosctl -n 192.168.0.134 read /proc/modules | wc -l
# Talos loads far fewer modules than a general-purpose Linux

# Confirm no package manager
talosctl -n 192.168.0.134 ls /usr/bin/ | grep -E "apt|yum|dnf|pacman"
# Expected: no output
```

**Controls active from Talos by default:**
- Immutable root filesystem -- no writes to OS directories
- No shell, no SSH, no interactive login
- SELinux-equivalent enforcement via Talos's own access control
- Kernel lockdown mode preventing unsigned kernel modules
- API server certificate rotation handled by Talos automatically

---

## Kubernetes API Server

```bash
# Confirm anonymous auth is disabled
kubectl get configmap -n kube-system kubeadm-config -o yaml | grep anonymous
# For Talos: anonymous auth is disabled by default

# Confirm audit logging is configured
talosctl -n 192.168.0.134 read /etc/kubernetes/manifests/kube-apiserver.yaml | grep audit
# Should show audit-log-path and audit-policy-file flags

# Confirm RBAC is enabled
kubectl api-versions | grep rbac
# Should show rbac.authorization.k8s.io/v1

# Confirm admission plugins
kubectl get pods -n kube-system kube-apiserver-talos-asj-72z -o yaml | grep enable-admission-plugins
# Should include: NamespaceLifecycle,LimitRanger,ServiceAccount,TLSBootstrapping,
#                 DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,
#                 ValidatingAdmissionWebhook,ResourceQuota
```

**Hardening applied:**
- NodeRestriction admission plugin: nodes can only modify their own resources
- ValidatingAdmissionWebhook: enables Gatekeeper
- RBAC mode: only mode enabled, no ABAC, no AlwaysAllow

---

## ArgoCD

```bash
# Disable local admin password after SSO is configured
argocd account update-password  # change to something long
# Better: disable it entirely once Authentik OIDC is working
kubectl patch configmap argocd-cm -n argocd \
  --type merge \
  -p '{"data":{"admin.enabled":"false"}}'

# Confirm repo server is not exposed externally
kubectl get svc -n argocd argocd-repo-server
# Should be ClusterIP only -- never LoadBalancer or NodePort

# Confirm no plaintext secrets in ArgoCD repos
kubectl get secrets -n argocd | grep repo-
# All repo credentials should reference SealedSecrets, not be plaintext

# Set resource limits on ArgoCD components
kubectl get pods -n argocd -o json | jq '.items[].spec.containers[].resources'
# Every container should have requests and limits set
```

**Hardening applied:**
- Admin account disabled after Authentik OIDC is live
- All ArgoCD secrets are SealedSecrets in the Git repo
- Repo server is ClusterIP only -- not externally reachable
- RBAC policy: platform-admin group gets admin, developer group gets read-only

---

## Forgejo

```bash
# After Authentik SSO is live, disable local user password login
# In Forgejo Admin Panel: Site Administration -> Authentication -> Disable local login

# Confirm SSH host key fingerprint
# Record this in the IQ document -- any change is a tampering indicator
ssh-keyscan -p 2222 forgejo.homelab 2>/dev/null | ssh-keygen -lf -

# Confirm webhook secret is set (prevents webhook spoofing)
# Forgejo webhook config should include a secret token that ArgoCD verifies

# Disable Forgejo's built-in HTTPS if Ingress-NGINX is handling TLS
# Forgejo should listen on HTTP internally; Ingress terminates TLS

# Confirm repository visibility -- all repos should be private by default
# Forgejo Admin Panel -> Default: Repository Visibility = Private
```

**Hardening applied:**
- Local password login disabled post-PH-02
- SSH host key fingerprint recorded in IQ
- All repos private by default
- Webhook secret token prevents spoofed push events

---

## Authentik

```bash
# Enforce MFA for admin accounts
# Authentik Admin -> System -> Policies -> Enforce MFA for platform-admin group

# Confirm session expiry is set
# Authentik Admin -> System -> Settings -> Token validity: 12h maximum for admin

# Disable Authentik's default flows that are not in use
# Remove: source-oauth-*, source-saml-* if not using those protocols

# Confirm outpost is not exposed without authentication
kubectl get svc -n authentik | grep outpost
# Outpost services should be ClusterIP only

# Set Authentik PostgreSQL password complexity
# The PostgreSQL password in the SealedSecret should be >32 chars, random
```

**Hardening applied:**
- MFA enforced for platform-admin group (TOTP or WebAuthn)
- Session tokens expire after 12 hours for admin accounts
- Unused authentication flows disabled
- PostgreSQL credentials are 32+ character random strings in SealedSecrets

---

## MinIO

```bash
# Create a dedicated service account for each consumer instead of using rootUser
mc admin user add homelab nextflow-sa <strong-password>
mc admin user add homelab loki-sa <strong-password>

# Assign minimal permissions per consumer
mc admin policy create homelab nextflow-input-readonly apps/minio/policies/nextflow-input-policy.json
mc admin policy attach homelab nextflow-input-readonly --user nextflow-sa

# Disable the rootUser after service accounts are created
mc admin user disable homelab admin

# Enable object versioning on pipeline-output to prevent accidental overwrites
mc version enable homelab/pipeline-output

# Enable bucket lock on loki-chunks to prevent log deletion
mc retention set --default GOVERNANCE 30d homelab/loki-chunks

# Confirm TLS is active
mc ping homelab
# Should show: TLS verified
```

**MinIO IAM policy example for Nextflow:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::pipeline-input/*", "arn:aws:s3:::pipeline-input"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::pipeline-work/*", "arn:aws:s3:::pipeline-work"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": ["arn:aws:s3:::pipeline-output/*", "arn:aws:s3:::pipeline-output"]
    }
  ]
}
```

**Hardening applied:**
- Dedicated service accounts per consumer -- no shared credentials
- rootUser disabled after service accounts are provisioned
- Object versioning on pipeline-output
- Bucket lock (GOVERNANCE mode) on loki-chunks prevents audit log deletion
- All service account credentials in SealedSecrets

---

## OPA Gatekeeper

```bash
# Confirm webhook failurePolicy is set to Fail after Gatekeeper is proven stable
kubectl get validatingwebhookconfiguration gatekeeper-validating-webhook-configuration \
  -o jsonpath='{.webhooks[*].failurePolicy}'
# Should be: Fail (not Ignore) in production state

# Confirm Gatekeeper itself has resource limits
kubectl get pods -n gatekeeper-system -o json | \
  jq '.items[].spec.containers[].resources.limits'
# Both controller-manager pods must have CPU and memory limits

# Confirm audit interval is set appropriately
kubectl get constraintpodstatus -A | head -5
# Audit should run at least every 5 minutes

# Confirm no constraints are stuck in warn mode that should be deny
kubectl get constraints -A -o json | \
  jq '.items[] | {name: .metadata.name, action: .spec.enforcementAction}'
# All production constraints should show "deny" not "warn"
```

**Hardening applied:**
- failurePolicy: Fail -- if Gatekeeper is down, no new workloads are admitted
- Gatekeeper controller itself has resource limits (enforced by its own constraints)
- Audit runs every 5 minutes catching constraint violations in existing workloads

---

## Falco

```bash
# Confirm eBPF driver is active (not kernel module)
kubectl exec -n monitoring <falco-pod> -- falco --version
# Should show: Driver: bpf

# Confirm Falco cannot be disabled by a pipeline pod
kubectl exec -n pipelines <pipeline-pod> -- kill -9 <falco-pid>
# Expected: permission denied -- Falco runs with host PID namespace restrictions

# Confirm Falcosidekick has a valid Loki connection
kubectl logs -n monitoring <falcosidekick-pod> | grep -i loki
# Should show: Loki OK

# Confirm custom rules are loaded
kubectl exec -n monitoring <falco-pod> -- \
  falco -r /etc/falco/rules.d/gxp-audit.yaml --validate
# Expected: Rules validation OK
```

**Hardening applied:**
- eBPF driver: operates below the application layer, cannot be disabled from within a container
- Falcosidekick connection to Loki is verified at startup
- Custom GxP rules validated before rolling restart
- Falco itself runs with minimal required host capabilities only

---

## Sealed Secrets

```bash
# Verify the controller's private key is backed up
# The Sealed Secrets controller generates a key pair on first install
# If the controller is deleted without backing up the key, all SealedSecrets become unrecoverable

# Backup the key:
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > ~/Kuber/sealed-secrets-key-backup.yaml

# Store this file encrypted, NOT in the Git repo
# This is the single most critical backup in the entire platform

# Confirm key rotation schedule
# By default, Sealed Secrets generates a new key every 30 days
# Old keys are kept to decrypt existing secrets
kubectl get secrets -n sealed-secrets | grep -c "sealed-secrets-key"
# Should show at least 1 active key
```

**Hardening applied:**
- Private key backed up to an encrypted location outside Git
- Key rotation every 30 days (Sealed Secrets default)
- Old keys retained for decrypting existing SealedSecrets

---

## Hardening Verification Checklist

Run this after completing each phase. Record results in [[OPS-03 Implementation Log]].

| Check | Command | Expected Result | Phase |
|---|---|---|---|
| No SSH access to nodes | `talosctl -n 192.168.0.134 get machineconfig \| grep ssh` | No output | PH-00 |
| API server RBAC enabled | `kubectl api-versions \| grep rbac` | `rbac.authorization.k8s.io/v1` | PH-00 |
| ArgoCD admin disabled | `kubectl get cm argocd-cm -n argocd -o yaml \| grep admin.enabled` | `false` | PH-02 |
| Local Forgejo login disabled | Manual: try logging in with local password | Login rejected | PH-02 |
| MinIO rootUser disabled | `mc admin user info homelab admin` | `status: disabled` | PH-03 |
| Gatekeeper failurePolicy Fail | `kubectl get validatingwebhookconfiguration \| grep Fail` | `Fail` | PH-04 |
| All constraints in deny mode | `kubectl get constraints -A -o json \| jq '.items[].spec.enforcementAction'` | All `deny` | PH-04 |
| Falco using eBPF driver | `kubectl exec -n monitoring <pod> -- falco --version` | `Driver: bpf` | PH-05 |
| Sealed Secrets key backed up | File exists at `~/Kuber/sealed-secrets-key-backup.yaml` | Exists, encrypted | PH-01 |
| MinIO bucket lock on loki-chunks | `mc retention info homelab/loki-chunks` | GOVERNANCE 30d | PH-03 |
