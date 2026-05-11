# OPS-08 Developer and Operator Onboarding Guide

> Part of [[README]] | See also: [[OPS-02 Reference Commands]], [[INF-01 Infrastructure Baseline]], [[INF-07 Network Topology]]

Complete setup guide for a new operator or developer getting access to this platform for the first time. Covers Mac workstation setup, tool installation, cluster access configuration, local DNS, and first-contact verification. Read this before opening any other file in the vault.

---

## Who This Is For

Anyone who needs to manage or operate this cluster from a Mac. That includes the primary operator (me) setting up after a machine reset, and any collaborator who is given access in the future.

---

## Mac Prerequisites

All cluster management is done from the Mac. The cluster nodes run Talos -- they have no SSH access and no shell. Everything goes through `kubectl`, `talosctl`, `helm`, and the ArgoCD CLI.

### Required Tools

Install in this order. Version pins matter -- tool versions must be compatible with the cluster.

```bash
# 1. Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. kubectl -- match the minor version of the cluster (currently K8s v1.35.x)
brew install kubectl
# Verify: kubectl version --client
# Expected: v1.35.x

# 3. talosctl -- must match the Talos version exactly (currently v1.12.6)
brew install siderolabs/tap/talosctl
# Or pin to a specific version:
# curl -Lo /usr/local/bin/talosctl https://github.com/siderolabs/talos/releases/download/v1.12.6/talosctl-darwin-arm64
# chmod +x /usr/local/bin/talosctl
# Verify: talosctl version

# 4. helm -- v3.x required
brew install helm
# Verify: helm version
# Expected: v3.x.x

# 5. kubeseal -- must match the Sealed Secrets controller version in the cluster
# Check controller version first:
# kubectl get deployment sealed-secrets-controller -n sealed-secrets -o jsonpath='{.spec.template.spec.containers[0].image}'
brew install kubeseal
# Verify: kubeseal --version

# 6. MinIO client (mc)
brew install minio/stable/mc
# Verify: mc --version

# 7. Nextflow (for running pipelines in Phase 6+)
brew install nextflow
# Verify: nextflow -version

# 8. ArgoCD CLI
brew install argocd
# Verify: argocd version --client

# 9. jq (JSON processing -- used throughout reference commands)
brew install jq

# 10. helm-diff plugin (for safe Helm upgrades)
helm plugin install https://github.com/databus23/helm-diff

# 11. Git
brew install git
# Configure:
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### Optional but Useful

```bash
# k9s -- terminal UI for Kubernetes
brew install derailed/k9s/k9s

# stern -- multi-pod log following
brew install stern

# watch -- run command repeatedly
brew install watch

# gpg -- for encrypting the Sealed Secrets key backup
brew install gnupg
```

---

## Kubeconfig and Talosconfig Setup

These files grant access to the cluster. They must be on the Mac and must not be committed to any Git repository or synced to any cloud service.

```bash
# Create the directory structure
mkdir -p ~/Kuber/talos-init
mkdir -p ~/Kuber/snapshots
mkdir -p ~/Kuber/patches
mkdir -p ~/Kuber/baselines

# The kubeconfig file should already exist at ~/Kuber/kubeconfig
# If starting fresh, generate it from Talos:
talosctl kubeconfig ~/Kuber/kubeconfig \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Set KUBECONFIG environment variable
echo 'export KUBECONFIG=~/Kuber/kubeconfig' >> ~/.zshrc
source ~/.zshrc

# Verify cluster access
kubectl get nodes
# Expected: both nodes Ready
```

```bash
# Verify talosconfig works
talosctl -n 192.168.0.134 version
# Expected: both client and server versions displayed

talosctl -n 192.168.0.202 version
# Expected: Beelink node version displayed
```

---

## Local DNS Setup for *.homelab

All platform services are accessible via `*.homelab` hostnames internally. This resolution must work on the Mac.

### Option A -- /etc/hosts Entries (Simplest)

Add one line per service to `/etc/hosts`:

```bash
sudo tee -a /etc/hosts <<EOF
# GxP BioInfra Platform -- *.homelab
192.168.0.200 argocd.homelab
192.168.0.200 forgejo.homelab
192.168.0.200 auth.homelab
192.168.0.200 grafana.homelab
192.168.0.200 minio.homelab
192.168.0.200 minio-console.homelab
192.168.0.200 prometheus.homelab
EOF
```

Verify:
```bash
ping -c 1 argocd.homelab
# Expected: response from 192.168.0.200
```

**Limitation:** Every new service needs a manual `/etc/hosts` entry. Adding a new ingress requires updating this file.

### Option B -- dnsmasq Wildcard (Recommended)

Configure dnsmasq to resolve all `*.homelab` to `192.168.0.200` automatically. New services work immediately without `/etc/hosts` changes.

```bash
# Install dnsmasq
brew install dnsmasq

# Configure wildcard resolution
echo 'address=/.homelab/192.168.0.200' > $(brew --prefix)/etc/dnsmasq.conf

# Start dnsmasq as a service
sudo brew services start dnsmasq

# Configure macOS to use dnsmasq for .homelab domains
sudo mkdir -p /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/homelab'

# Verify
scutil --dns | grep homelab
ping -c 1 anything.homelab
# Expected: response from 192.168.0.200
```

**Restart required:** macOS may require a restart for the resolver configuration to take effect.

---

## TLS Certificate Trust

The cluster uses a self-signed CA managed by Cert-Manager. The Mac's browser will warn about this unless the CA certificate is trusted.

```bash
# Extract the CA certificate from the cluster
kubectl get secret -n cert-manager root-ca-secret -o jsonpath='{.data.ca\.crt}' | \
  base64 -d > /tmp/homelab-ca.crt

# If the CA secret name is different, check:
kubectl get secrets -n cert-manager

# Add to macOS trust store
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /tmp/homelab-ca.crt

# Verify: open https://argocd.homelab in Safari or Chrome
# Expected: no certificate warning
```

---

## ArgoCD CLI Authentication

```bash
# Log in to ArgoCD via CLI
# After Authentik SSO is live (Phase 2+):
argocd login argocd.homelab --sso

# Before Phase 2 (using admin password):
argocd login argocd.homelab --username admin --password <admin-password>

# Verify
argocd app list
```

---

## MinIO Client Setup

```bash
# Configure mc alias for the local MinIO instance
# Replace with actual credentials from the SealedSecret
mc alias set homelab https://minio.homelab <access-key> <secret-key>

# Verify
mc ping homelab
mc ls homelab/
```

---

## SSH Key for Forgejo

```bash
# Generate an SSH key pair for Forgejo (separate from any existing SSH keys)
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_forgejo

# Add the public key to Forgejo
# Forgejo -> User Settings -> SSH / GPG Keys -> Add Key
cat ~/.ssh/id_forgejo.pub

# Add to SSH config
cat >> ~/.ssh/config <<EOF

Host forgejo.homelab
  HostName forgejo.homelab
  Port 2222
  User git
  IdentityFile ~/.ssh/id_forgejo
EOF

# Test SSH clone
git clone ssh://git@forgejo.homelab:2222/admin/Home-Lab-Infra-as-code.git /tmp/test-clone
# Expected: clone succeeds without password prompt
```

---

## First-Contact Verification Checklist

Run this after completing the setup above. Every item must pass before working with the cluster.

```bash
# 1. Both nodes Ready
kubectl get nodes
# Expected: talos-asj-72z Ready, talos-v3h-4m1 Ready

# 2. Control plane health
talosctl -n 192.168.0.134 health
# Expected: no errors

# 3. All pods running
kubectl get pods -A | grep -v Running | grep -v Completed
# Expected: no output (all pods Running)

# 4. StorageClass available
kubectl get sc
# Expected: local-hdd (default) listed

# 5. ArgoCD CLI works
argocd app list
# Expected: list of applications

# 6. MinIO reachable
mc ping homelab
# Expected: OK

# 7. Grafana reachable
curl -I https://grafana.homelab
# Expected: HTTP/2 200

# 8. ArgoCD web UI reachable
curl -I https://argocd.homelab
# Expected: HTTP/2 200

# 9. Forgejo reachable
curl -I https://forgejo.homelab
# Expected: HTTP/2 200 (after Phase 1)

# 10. Sealed Secrets controller running
kubectl get pods -n sealed-secrets
# Expected: sealed-secrets-controller-* Running

# 11. etcd healthy
talosctl -n 192.168.0.134 etcd members
# Expected: one member listed (Omen)
```

---

## What to Do If Access Is Lost

### kubeconfig expired or lost

```bash
# Regenerate from Talos
talosctl kubeconfig ~/Kuber/kubeconfig \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig
```

### talosconfig lost

The talosconfig cannot be regenerated from the cluster alone -- it was created during cluster bootstrap. If the file is truly lost and not backed up:
1. Retrieve the client certificate from the Talos bootstrap state (requires physical access to the Omen)
2. Re-generate talosconfig using the existing cluster's machine configs

**Prevention:** Back up `~/Kuber/talos-init/talosconfig` to an encrypted external drive.

### Cluster API unreachable (kubectl times out)

```bash
# Check if the Omen is reachable at all
ping 192.168.0.134

# Check Talos API
talosctl -n 192.168.0.134 version

# If Talos API works but kubectl does not:
# The API server may be down -- check Talos service status
talosctl -n 192.168.0.134 service apiserver

# If neither works: Omen is likely down or on a different IP (DHCP)
# Check UniFi controller for current Omen IP
```

---

## Tool Version Compatibility Matrix

| Tool | Current Required Version | Cluster Component | Why Version Matters |
|---|---|---|---|
| kubectl | 1.35.x | Kubernetes 1.35.2 | Minor version must be within ±1 of cluster |
| talosctl | 1.12.6 | Talos 1.12.6 | Must match exactly |
| helm | 3.x | ArgoCD Helm support | v2 is EOL |
| kubeseal | Match controller | Sealed Secrets controller | Key format changes between versions |
| nextflow | 24.x+ | nf-core/rnaseq 3.14 | nf-core pipelines require recent Nextflow |
| argocd CLI | Match ArgoCD server | ArgoCD server | API version compatibility |
