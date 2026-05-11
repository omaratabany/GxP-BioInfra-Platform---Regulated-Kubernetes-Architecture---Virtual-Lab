# OPS-08 Developer and Operator Onboarding

> Part of [[README]] | See also: [[INF-07 Network Topology]], [[OPS-02 Reference Commands]], [[SEC-04 Secrets and Key Management]]

Everything needed to go from a clean Mac to fully operational on this cluster. Run through this once in order. Takes approximately 30-45 minutes the first time.

This guide assumes: MacBook running macOS (Apple Silicon), access to the LAN at 192.168.0.0/24, and the talosconfig and kubeconfig files have been shared with you by the operator who built the cluster.

---

## Step 1 -- Install Required CLI Tools

Install all tools via Homebrew. Pin versions where critical.

```bash
# Install Homebrew if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Core cluster management
brew install kubectl
brew install helm
brew tap siderolabs/tap && brew install talosctl

# GitOps and secret management
brew install argocd
brew install kubeseal

# MinIO client (for S3 operations and bucket verification)
brew install minio/stable/mc

# Bioinformatics pipeline runtime
brew install nextflow

# Utilities used in OPS-06 test scripts
brew install jq
brew install coreutils   # For sha256sum on macOS

# Helm plugins used in MOD-03 validation procedure
helm plugin install https://github.com/databus23/helm-diff
```

**Version verification after install:**
```bash
kubectl version --client
helm version
talosctl version
argocd version --client
kubeseal --version
mc --version
nextflow -version
jq --version
```

Record the installed versions in [[OPS-03 Implementation Log]] under the tooling section.

---

## Step 2 -- Configure Cluster Access

You need two credential files: `kubeconfig` for kubectl and `argocd`, and `talosconfig` for talosctl. Obtain these from the cluster operator.

```bash
# Create the directory structure
mkdir -p ~/Kuber/talos-init
mkdir -p ~/Kuber/snapshots

# Place the files (obtained from the operator)
# kubeconfig at: ~/Kuber/kubeconfig
# talosconfig at: ~/Kuber/talos-init/talosconfig

# Set kubectl to use this kubeconfig
export KUBECONFIG=~/Kuber/kubeconfig
# Add this to your shell profile to make it permanent:
echo 'export KUBECONFIG=~/Kuber/kubeconfig' >> ~/.zshrc

# Test kubectl access
kubectl get nodes -o wide
# Expected: both nodes listed as Ready

# Test talosctl access
talosctl -n 192.168.0.134 version
talosctl -n 192.168.0.202 version
# Both should return the Talos version without errors
```

**Security requirement:** Do NOT sync the kubeconfig or talosconfig to iCloud, Google Drive, GitHub, or any cloud service. These files provide full cluster access.

---

## Step 3 -- Configure ArgoCD CLI

```bash
# Login to ArgoCD (after Authentik SSO is configured in PH-02, use SSO)
# Before PH-02: use the admin password
argocd login argocd.homelab \
  --username admin \
  --password <admin-password> \
  --grpc-web

# After PH-02: use SSO
argocd login argocd.homelab --sso --grpc-web

# Verify
argocd app list
# Should show all deployed applications
```

---

## Step 4 -- Configure Local DNS for *.homelab

Without this, your browser and kubectl commands will not resolve the internal service names.

**Option A -- /etc/hosts (simple, requires updating for each new service):**

```bash
# Add all current services
sudo tee -a /etc/hosts <<EOF

# GxP BioInfra Platform
192.168.0.200  forgejo.homelab
192.168.0.200  auth.homelab
192.168.0.200  argocd.homelab
192.168.0.200  grafana.homelab
192.168.0.200  minio.homelab
192.168.0.200  minio-console.homelab
192.168.0.200  prometheus.homelab
192.168.0.200  alertmanager.homelab
EOF
```

**Option B -- dnsmasq wildcard (recommended -- no updates needed for new services):**

```bash
# Install and configure dnsmasq
brew install dnsmasq

# Add wildcard rule
echo "address=/.homelab/192.168.0.200" >> $(brew --prefix)/etc/dnsmasq.conf

# Start dnsmasq
sudo brew services start dnsmasq

# Tell macOS to use dnsmasq for .homelab TLD only
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/homelab

# Test
ping -c 1 anything.homelab
# Expected: PING anything.homelab (192.168.0.200)
```

---

## Step 5 -- Trust the Homelab CA Certificate

All platform services use TLS certificates issued by the homelab CA (managed by Cert-Manager). Without trusting this CA, every browser and CLI tool will give TLS warnings.

```bash
# Export the CA certificate from the cluster
kubectl get secret homelab-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > ~/Kuber/homelab-ca.crt

# Add to Mac keychain and trust it
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain ~/Kuber/homelab-ca.crt

# Verify: open https://forgejo.homelab in Safari -- should show no certificate warning
# For curl:
curl https://forgejo.homelab
# Should return HTML, not a TLS error
```

**Note:** Chrome and Firefox have their own certificate stores and may still show warnings. For Chrome: Settings -> Privacy -> Security -> Manage Certificates -> import the CA cert.

---

## Step 6 -- Configure MinIO Client

```bash
# Add the homelab MinIO alias
mc alias set homelab https://minio.homelab <access-key> <secret-key>

# Test connectivity
mc ping homelab

# List buckets
mc ls homelab
# Expected: pipeline-input, pipeline-output, pipeline-work, loki-chunks
```

Credentials come from the minio-credentials SealedSecret. Ask the cluster operator for the access key and secret key, or retrieve from the sealed secret if you have cluster access:

```bash
kubectl get secret minio-credentials -n minio \
  -o jsonpath='{.data.rootPassword}' | base64 -d
```

---

## Step 7 -- Configure kubeseal for Secret Creation

kubeseal needs the Sealed Secrets controller's public key to seal new secrets.

```bash
# Fetch and cache the public key
kubeseal --fetch-cert \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets > ~/Kuber/sealed-secrets-public-key.pem

# Test by sealing a dummy secret
kubectl create secret generic test-seal \
  --from-literal=foo=bar \
  --dry-run=client -o yaml | \
  kubeseal --cert ~/Kuber/sealed-secrets-public-key.pem \
  -o yaml
# Expected: SealedSecret YAML output with encrypted data field
```

---

## Step 8 -- Clone the Infrastructure Repository

```bash
# Before PH-01 (Forgejo): clone from GitHub
git clone git@github.com:omaratabany/Home-Lab-Infra-as-code.git ~/repos/Home-Lab-Infra-as-code

# After PH-01 (Forgejo): use Forgejo as primary
git remote set-url origin ssh://git@forgejo.homelab:2222/admin/Home-Lab-Infra-as-code.git

# Verify
cd ~/repos/Home-Lab-Infra-as-code
git remote -v
git pull origin main
```

**SSH key setup for Forgejo (after PH-01):**
```bash
# Generate an SSH key if you don't have one
ssh-keygen -t ed25519 -C "operator@homelab"

# Add the public key to Forgejo
# Forgejo -> User Settings -> SSH Keys -> Add Key
cat ~/.ssh/id_ed25519.pub

# Test SSH access
ssh -T git@forgejo.homelab -p 2222
# Expected: Welcome to Gitea, <username>!
```

---

## Step 9 -- Verify Full Stack Access

Run this sequence to confirm everything is working:

```bash
echo "--- Cluster nodes ---"
kubectl get nodes -o wide

echo "--- All pods healthy ---"
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v Terminating

echo "--- ArgoCD applications ---"
argocd app list | head -20

echo "--- Storage class ---"
kubectl get sc

echo "--- Certificates ---"
kubectl get certificates -A

echo "--- Gatekeeper constraints ---"
kubectl get constraints -A

echo "--- MinIO connectivity ---"
mc ping homelab --count 3

echo "--- Talos health ---"
talosctl -n 192.168.0.134 health
talosctl -n 192.168.0.202 health
```

Everything above should return clean output with no errors. Any failures need to be resolved before the operator is considered onboarded.

---

## Ongoing Access Reference

| Tool | Config location | Command to test |
|---|---|---|
| kubectl | `~/Kuber/kubeconfig` | `kubectl get nodes` |
| talosctl | `~/Kuber/talos-init/talosconfig` | `talosctl -n 192.168.0.134 health` |
| argocd CLI | Login cached after `argocd login` | `argocd app list` |
| mc (MinIO) | `~/.mc/config.json` (after `mc alias set`) | `mc ping homelab` |
| Forgejo SSH | `~/.ssh/id_ed25519` + Forgejo key upload | `ssh -T git@forgejo.homelab -p 2222` |
| Grafana | Browser: https://grafana.homelab | SSO via Authentik |
| ArgoCD UI | Browser: https://argocd.homelab | SSO via Authentik |

---

## What to Do if You Rotate Credentials

If any platform credentials are rotated (see [[SEC-04 Secrets and Key Management]]):

- **kubeconfig rotated:** Operator provides new kubeconfig. Replace `~/Kuber/kubeconfig`.
- **ArgoCD login expired:** Run `argocd login argocd.homelab --sso --grpc-web` again.
- **MinIO credentials rotated:** Run `mc alias set homelab https://minio.homelab <new-key> <new-secret>`.
- **Sealed Secrets key rotated:** Run Step 7 again to refresh the cached public key.
- **CA certificate changed:** Run Step 5 again with the new CA cert.

---

## Mac Security Requirements

Before using this Mac for cluster management:

- Full-disk encryption must be enabled (System Settings -> Privacy and Security -> FileVault)
- Screen lock must activate after no more than 5 minutes of inactivity
- kubeconfig and talosconfig must not be copied to any cloud-synced location (no iCloud Desktop, no Dropbox)
- The Sealed Secrets private key backup (`~/Kuber/sealed-secrets-key-backup.yaml.gpg`) must be encrypted and stored separately from the public key
