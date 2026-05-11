# INF-08 Platform Prerequisites

> Part of [[README]] | See also: [[OPS-08 Developer Onboarding]], [[INF-07 Network Topology]], [[OPS-01 Build Instructions]]

Everything that must exist and be verified before Phase 0 begins. Cert-Manager, MetalLB, Ingress-NGINX, and the local DNS configuration are assumed to already be deployed -- this file documents their exact configuration so the IQ can reference it and so any operator can reproduce the setup.

---

## What Counts as a Prerequisite

These components are not deployed as part of Phases 0-7 -- they were already running when this project began, or they are deployed as the very first bootstrapping step before Phase 0 can proceed. Without them, nothing else works.

| Component | Why It Must Exist First |
|---|---|
| MetalLB | Without it, Ingress-NGINX has no IP. Without Ingress-NGINX, no service is reachable. |
| Ingress-NGINX | All *.homelab routing goes through it. Phase 1 (Forgejo) needs an Ingress object. |
| Cert-Manager | All Ingress objects request TLS. Without Cert-Manager, TLS cert requests hang. |
| CoreDNS | Already bundled with Kubernetes. Needed for inter-pod service discovery. |
| Local DNS | Mac must resolve *.homelab to 192.168.0.200 for any browser or CLI tool to work. |

---

## MetalLB

### Deployment

```bash
# Add Helm repo
helm repo add metallb https://metallb.github.io/metallb
helm repo update

# Deploy
helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  --version <pin-version> \
  -f apps/metallb/helm-release.yaml

# Verify
kubectl get pods -n metallb-system
# Expected: controller-* Running, speaker-* Running on both nodes
```

### IPAddressPool Configuration

```yaml
# apps/metallb/ip-address-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.200-192.168.0.210
```

```yaml
# apps/metallb/l2-advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool
```

```bash
# Apply
kubectl apply -f apps/metallb/ip-address-pool.yaml
kubectl apply -f apps/metallb/l2-advertisement.yaml

# Verify VIP is assigned
kubectl get svc -A | grep LoadBalancer
# After Ingress-NGINX is deployed: should show 192.168.0.200
```

### Troubleshooting

```bash
# If VIP is not responding to ARP on the LAN:
kubectl logs -n metallb-system -l component=speaker | grep -i error

# If two nodes are both trying to advertise the VIP:
kubectl get l2advertisement -n metallb-system -o yaml
# Should show a single advertisement object

# Check which node currently owns the VIP
kubectl get events -n metallb-system | grep -i announce
```

---

## Ingress-NGINX

### Deployment

```bash
# Add Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Deploy
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --version <pin-version> \
  -f apps/ingress-nginx/helm-release.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
# Expected: ingress-nginx-controller service type LoadBalancer with EXTERNAL-IP 192.168.0.200
```

### Helm Values (apps/ingress-nginx/helm-release.yaml)

```yaml
# chart: ingress-nginx/ingress-nginx
# version: <pin>

controller:
  nodeSelector:
    node-role: infra
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  service:
    type: LoadBalancer
  # Force HTTP -> HTTPS redirect globally
  config:
    ssl-redirect: "true"
    use-forwarded-headers: "true"
    # Enable real IP from Cloudflare Tunnel
    proxy-real-ip-cidr: "0.0.0.0/0"
```

### Standard Ingress Object Template

Every service that needs external access uses this pattern:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app>-ingress
  namespace: <namespace>
  labels:
    app.kubernetes.io/name: <app>
    app.kubernetes.io/managed-by: argocd
    env: dev
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - <app>.homelab
    secretName: <app>-tls
  rules:
  - host: <app>.homelab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: <port>
```

### Troubleshooting

```bash
# Check Ingress-NGINX logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx | tail -50

# Verify an Ingress object is being processed
kubectl describe ingress <name> -n <namespace>
# Look for: "Address: 192.168.0.200" in the output

# Test routing from Mac
curl -v https://<app>.homelab
# Expected: TLS handshake, 200 response (or redirect)

# If 502 Bad Gateway:
# The backend pod is not running or not ready
kubectl get endpoints <service-name> -n <namespace>
# Should show pod IPs, not empty list
```

---

## Cert-Manager

### Deployment

```bash
# Add Helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Deploy (CRDs must be installed separately)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<version>/cert-manager.crds.yaml

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version <pin-version> \
  -f apps/cert-manager/helm-release.yaml

# Verify
kubectl get pods -n cert-manager
# Expected: cert-manager-*, cert-manager-webhook-*, cert-manager-cainjector-* Running
```

### ClusterIssuer -- Self-Signed CA

The homelab uses a self-signed CA managed by Cert-Manager. The CA certificate and private key are stored as a Kubernetes Secret (SealedSecret in Git).

```yaml
# Step 1: Create a self-signed root CA SealedSecret
# First generate the CA key and cert:
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=homelab-ca/O=GxP BioInfra Platform"

# Create the K8s secret (then seal it)
kubectl create secret tls homelab-ca-secret \
  --cert=ca.crt \
  --key=ca.key \
  --namespace cert-manager \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets \
  --controller-name sealed-secrets -o yaml > \
  apps/cert-manager/sealed-ca-secret.yaml

# Clean up plaintext key immediately
rm -f ca.key

# Keep ca.crt -- needs to be trusted on Mac
```

```yaml
# apps/cert-manager/cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: homelab-ca-issuer
spec:
  ca:
    secretName: homelab-ca-secret
```

```bash
# Apply the ClusterIssuer
kubectl apply -f apps/cert-manager/cluster-issuer.yaml

# Verify
kubectl get clusterissuer homelab-ca-issuer
# Expected: READY=True

# Test by creating a certificate
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-cert
  namespace: default
spec:
  secretName: test-cert-tls
  issuerRef:
    name: homelab-ca-issuer
    kind: ClusterIssuer
  dnsNames:
  - test.homelab
EOF

kubectl get certificate test-cert -n default
# Expected: READY=True within 30 seconds
kubectl delete certificate test-cert -n default
```

### Trusting the CA on the Mac

```bash
# Extract the CA certificate from the cluster
kubectl get secret homelab-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/homelab-ca.crt

# Add to macOS system trust store
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /tmp/homelab-ca.crt

# Verify in Keychain Access:
# System -> Certificates -> homelab-ca (should show "This root certificate is trusted")

# Test in browser: open https://argocd.homelab
# Expected: green padlock, no certificate warning
```

### Certificate Renewal

Cert-Manager automatically renews certificates 30 days before expiry. No manual action required. Monitor via:

```bash
kubectl get certificates -A
# READY column should be True for all
# AGE column shows when the cert was issued

# Alert if any certificate shows READY=False
# See INF-06 Observability and Alerting -- CertificateExpiringSoon alert rule
```

---

## CoreDNS (Pre-configured by Kubernetes)

CoreDNS is deployed automatically by Kubernetes and requires no additional configuration for standard service discovery. Verify it is running:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Expected: coredns-* Running (2 replicas)

# Test internal DNS resolution from within the cluster
kubectl run dns-test --image=busybox:1.36 \
  --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
# Expected: resolves to the ClusterIP of the kubernetes service
```

---

## Local DNS on Mac (*.homelab Resolution)

See [[OPS-08 Developer Onboarding]] for full setup instructions. Quick reference:

**Option A -- /etc/hosts (simple):**
```bash
sudo bash -c 'echo "192.168.0.200 argocd.homelab forgejo.homelab auth.homelab grafana.homelab minio.homelab minio-console.homelab prometheus.homelab" >> /etc/hosts'
```

**Option B -- dnsmasq wildcard (recommended):**
```bash
brew install dnsmasq
echo 'address=/.homelab/192.168.0.200' > $(brew --prefix)/etc/dnsmasq.conf
sudo brew services start dnsmasq
sudo mkdir -p /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/homelab'
```

---

## Prerequisite Verification Checklist

Run before starting Phase 0. All must PASS.

```bash
#!/bin/bash
echo "=== Platform Prerequisites Check ==="

# 1. MetalLB IP pool assigned
echo -n "MetalLB IP pool: "
kubectl get ipaddresspool -n metallb-system homelab-pool &>/dev/null && echo "PASS" || echo "FAIL"

# 2. Ingress-NGINX has LoadBalancer IP
echo -n "Ingress-NGINX LoadBalancer IP: "
IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
[[ "$IP" == "192.168.0.200" ]] && echo "PASS ($IP)" || echo "FAIL (got: $IP)"

# 3. Cert-Manager ClusterIssuer ready
echo -n "Cert-Manager ClusterIssuer: "
READY=$(kubectl get clusterissuer homelab-ca-issuer \
  -o jsonpath='{.status.conditions[0].status}' 2>/dev/null)
[[ "$READY" == "True" ]] && echo "PASS" || echo "FAIL (status: $READY)"

# 4. CoreDNS running
echo -n "CoreDNS: "
kubectl get pods -n kube-system -l k8s-app=kube-dns \
  --field-selector=status.phase=Running &>/dev/null && echo "PASS" || echo "FAIL"

# 5. Local DNS resolves *.homelab
echo -n "Local DNS (argocd.homelab): "
RESOLVED=$(dig +short argocd.homelab 2>/dev/null || getent hosts argocd.homelab 2>/dev/null | awk '{print $1}')
[[ "$RESOLVED" == "192.168.0.200" ]] && echo "PASS" || echo "FAIL (got: $RESOLVED)"

# 6. CA certificate trusted on Mac
echo -n "CA certificate trusted: "
curl -s --fail https://argocd.homelab &>/dev/null && echo "PASS" || echo "FAIL (TLS error or service down)"

echo ""
echo "All checks must PASS before starting Phase 0."
```
