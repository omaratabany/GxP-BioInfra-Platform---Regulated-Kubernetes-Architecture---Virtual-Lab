# SEC-04 Secrets and Key Management

> Part of [[README]] | See also: [[SEC-01 Security Architecture]], [[SEC-02 Component Hardening Guide]], [[ADR-00 Decision Log]] D-12

How I create, store, rotate, audit, and retire secrets across the platform. Every credential in this system has a defined lifecycle. Nothing exists as plaintext in Git, in environment variables without encryption at rest, or in any shared location.

---

## Secret Categories and Storage Locations

| Category | Examples | Storage Mechanism | In Git? |
|---|---|---|---|
| Service credentials | Forgejo admin, MinIO root, Authentik secret key | SealedSecret | Yes -- encrypted |
| OIDC client secrets | ArgoCD client secret, Grafana client secret | SealedSecret | Yes -- encrypted |
| MinIO service account credentials | nextflow-sa, loki-sa passwords | SealedSecret | Yes -- encrypted |
| Sealed Secrets controller private key | RSA private key for decryption | Backed up to encrypted file outside Git | No |
| TLS certificates | *.homelab wildcard, per-service certs | Cert-Manager managed, stored in K8s secrets | No |
| etcd encryption keys | etcd data encryption at rest | Managed by Talos | No |
| talosconfig | Machine config bootstrap credentials | `~/Kuber/talos-init/talosconfig` | No -- Mac only |
| kubeconfig | kubectl credentials | `~/Kuber/kubeconfig` | No -- Mac only |

---

## Secret Lifecycle

### Creation

All secrets start as plain Kubernetes secrets generated with `--dry-run=client`, then immediately sealed before any commit occurs:

```bash
# Step 1: Generate the plain secret (NEVER commit this)
kubectl create secret generic <name> \
  --from-literal=key=<value> \
  --dry-run=client -o yaml > /tmp/raw-secret.yaml

# Step 2: Seal it immediately
kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  -o yaml < /tmp/raw-secret.yaml > apps/<app>/sealed-secret.yaml

# Step 3: Destroy the plaintext immediately
rm -f /tmp/raw-secret.yaml

# Step 4: Commit sealed-secret.yaml to Forgejo via PR
```

Password generation standard -- never use memorable passwords for service credentials:

```bash
# Generate a 32-character random password
openssl rand -base64 32 | tr -d '/+=' | head -c 32
```

### Storage Rules

| What | Where | How |
|---|---|---|
| SealedSecrets | Git repo `apps/<app>/sealed-secret.yaml` | Encrypted, safe to commit |
| Sealed Secrets controller key | `~/Kuber/sealed-secrets-key-backup.yaml` | Encrypted with GPG or stored on an encrypted volume |
| talosconfig | `~/Kuber/talos-init/talosconfig` | Mac only, not synced to any cloud service |
| kubeconfig | `~/Kuber/kubeconfig` | Mac only, not synced to any cloud service |
| Plaintext secrets | Nowhere | Never exist in a persisted state |

### Rotation Schedule

| Secret | Rotation Trigger | Rotation Frequency |
|---|---|---|
| MinIO service account credentials | On suspicion of compromise or every 90 days | 90 days |
| Authentik admin password | Every 90 days or on platform team change | 90 days |
| Forgejo admin credentials | Post-PH-02 (replaced by Authentik SSO) | N/A -- local login disabled |
| OIDC client secrets | On suspicion of compromise | On demand |
| TLS certificates | Cert-Manager auto-renews 30 days before expiry | Automatic |
| Sealed Secrets controller key | New key generated every 30 days by controller | Automatic (controller default) |
| talosconfig | On cluster rebuild or node replacement | As needed |
| MinIO rootUser credentials | Disabled after service accounts are provisioned | N/A -- account disabled |

### Rotation Procedure (Service Account Credentials)

```bash
# 1. Generate new credential
NEW_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)

# 2. Create new SealedSecret
kubectl create secret generic <secret-name> \
  --from-literal=password=$NEW_PASSWORD \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > apps/<app>/sealed-secret.yaml

# 3. Open a PR in Forgejo with the change
# PR description must include: what is being rotated, why, and the date

# 4. Merge PR -- ArgoCD syncs the new SealedSecret to the cluster

# 5. Update the credential in the target system
# For MinIO service accounts:
mc admin user add homelab <username> $NEW_PASSWORD
mc admin user remove homelab <username>  # removes old, recreates with new password

# 6. Verify the application still functions with the new credential
# For Nextflow: run the test pipeline
# For Loki: verify logs appear in Grafana

# 7. Record the rotation in OPS-03 Implementation Log
# Include: date, what was rotated, PR number, verification result
```

### Retirement

When a credential is no longer needed (application removed, user offboarded):

```bash
# 1. Disable the credential immediately
# For MinIO: mc admin user disable homelab <username>
# For Authentik: disable the user account in the UI

# 2. Remove the SealedSecret from the repo via PR

# 3. Delete the K8s secret from the cluster
kubectl delete secret <secret-name> -n <namespace>

# 4. For MinIO: remove the user entirely after confirming no active sessions
mc admin user remove homelab <username>

# 5. Record in OPS-03 Implementation Log
```

---

## Sealed Secrets Controller Key Backup Procedure

This is the most critical backup in the entire platform. Loss of this key means all SealedSecrets become unrecoverable.

```bash
# Export the active key
kubectl get secret \
  -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > ~/Kuber/sealed-secrets-key-backup.yaml

# Encrypt it with GPG before storing anywhere
gpg --symmetric --cipher-algo AES256 ~/Kuber/sealed-secrets-key-backup.yaml
# This creates ~/Kuber/sealed-secrets-key-backup.yaml.gpg

# Store the encrypted file in at least two locations:
# 1. Local encrypted external drive
# 2. Password manager (as an attachment)
# NEVER store in: iCloud, Google Drive, GitHub, Git, Forgejo, any unencrypted cloud

# Verify the backup is readable before relying on it
gpg --decrypt ~/Kuber/sealed-secrets-key-backup.yaml.gpg | head -5
# Should show: apiVersion: v1 kind: List
```

---

## Secret Audit Procedure

Run quarterly. Verify that:

```bash
# 1. No plaintext secrets exist in the cluster with known-bad patterns
kubectl get secrets -A -o json | \
  jq '.items[] | {name: .metadata.name, ns: .metadata.namespace, type: .type}' | \
  grep -v "kubernetes.io/service-account-token" | \
  grep -v "helm.sh/release"
# Manual review of any unexpected secret types

# 2. All SealedSecrets have corresponding sealed-secret.yaml in Git
# Compare: kubectl get sealedsecrets -A vs files in apps/*/sealed-secret.yaml

# 3. TLS certificates are not approaching expiry
kubectl get certificates -A
# Check the READY and AGE columns -- Cert-Manager should auto-renew

# 4. MinIO service accounts are still restricted to their assigned policies
mc admin user list homelab
mc admin user info homelab nextflow-sa
# Verify policy assignment matches the IAM policy file in apps/minio/policies/

# 5. Sealed Secrets controller key backup was created less than 90 days ago
ls -la ~/Kuber/sealed-secrets-key-backup.yaml.gpg
```

Record audit results in [[OPS-03 Implementation Log]].

---

## Secret Naming Convention

All SealedSecrets follow the pattern: `<app>-<purpose>-secret`

| App | Secret Name | Contains |
|---|---|---|
| Forgejo | `forgejo-admin-secret` | username, password |
| MinIO | `minio-credentials` | rootUser, rootPassword |
| MinIO | `minio-nextflow-sa-secret` | accessKey, secretKey for nextflow-sa |
| MinIO | `minio-loki-sa-secret` | accessKey, secretKey for loki-sa |
| Authentik | `authentik-secret-key-secret` | secretKey, postgresPassword, redisPassword |
| ArgoCD | `argocd-oidc-secret` | oidc.authentik.clientSecret |
