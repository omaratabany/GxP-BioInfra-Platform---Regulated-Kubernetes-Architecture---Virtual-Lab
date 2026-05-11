# OPS-09 Patch Management

> Part of [[README]] | See also: [[REG-04 Supplier Assessment Register]], [[REG-05 Statement of Applicability]], [[MOD-02 Modularity and Dependency Map]]

Defines how security advisories are discovered, triaged, and responded to. Implements ISO/IEC 27001:2022 control 8.8 (Management of Technical Vulnerabilities). Every patch follows this procedure -- from alert to deployed update to recorded evidence.

---

## Advisory Sources

Subscribe to or check these sources as part of the monthly MO-04 dependency review in [[OPS-04 Operational Runbook]]. Any CRITICAL or HIGH advisory triggers an out-of-cycle patch within the response window defined below.

| Supplier | Advisory Source | Check Frequency |
|---|---|---|
| Talos OS | https://github.com/siderolabs/talos/security/advisories | Monthly + GitHub notifications |
| ArgoCD | https://github.com/argoproj/argo-cd/security/advisories | Monthly + GitHub notifications |
| Grafana | https://github.com/grafana/grafana/security/advisories | Monthly + GitHub notifications |
| Falco | https://falco.org/blog/category/security/ | Monthly |
| Kubernetes (general) | https://kubernetes.io/docs/reference/issues-security/security/ | Monthly |
| nf-core | https://github.com/nf-core/rnaseq/releases | Per pipeline use |
| Container images (general) | https://www.cvedetails.com by image name | On-demand for specific images |
| All GitHub-hosted components | GitHub Security tab per repo + GitHub Dependabot alerts if repo is forked | Monthly |

**Automated advisory capture:** Configure GitHub to send notifications for security advisories on any watched repository. This gives immediate notification for HIGH and CRITICAL CVEs without waiting for the monthly review.

---

## Severity Classification and Response Times

| CVE Severity | CVSS Score | Response Window | Action |
|---|---|---|---|
| CRITICAL | 9.0 - 10.0 | 48 hours | Emergency patch -- skip normal PR review cycle if needed, document as emergency change |
| HIGH | 7.0 - 8.9 | 7 days | Priority patch -- open PR immediately, complete within 7 days |
| MEDIUM | 4.0 - 6.9 | 30 days | Include in next scheduled update cycle |
| LOW | 0.1 - 3.9 | 90 days or next major version | Track in dependency review, update when convenient |

**Elevated response for GxP-critical components:** The following components have a stricter response window because their compromise directly affects the Annex 11 audit trail or access control:

| Component | CRITICAL | HIGH | MEDIUM |
|---|---|---|---|
| Falco | 24 hours | 48 hours | 7 days |
| Sealed Secrets controller | 24 hours | 48 hours | 7 days |
| Authentik | 48 hours | 7 days | 14 days |
| ArgoCD | 48 hours | 7 days | 14 days |

---

## Patch Procedure -- Standard (MEDIUM and LOW)

### Step 1 -- Discovery

Identified during monthly MO-04 dependency review or via GitHub advisory notification.

```bash
# Check for available Helm chart updates
helm repo update
helm list -A -o json | jq '.[] | {name: .name, chart: .chart, app_version: .app_version}'
helm search repo <chart-name> --versions | head -5
```

### Step 2 -- Triage

Answer these questions before proceeding:

```
1. What is the CVE ID and CVSS score?
2. Is the vulnerable component actually reachable in this deployment?
   (e.g., a vuln in Grafana's SMTP feature is irrelevant if SMTP is not configured)
3. What is the blast radius if exploited?
4. Is a patch available, or only a workaround?
5. Does upgrading this component require any config changes?
```

Record the triage assessment in [[OPS-03 Implementation Log]].

### Step 3 -- Prepare the Update

```bash
# For a Helm chart update
helm diff upgrade <release-name> <repo>/<chart> \
  --version <new-version> \
  --namespace <namespace> \
  -f apps/<app>/helm-release.yaml

# Review the diff carefully -- any RBAC or webhook changes are security-relevant
```

Update the chart version in `apps/<app>/helm-release.yaml`. If the update changes an image, also update the digest:

```bash
docker manifest inspect <image>:<new-tag> | jq '.manifests[].digest'
# Update the image digest in helm-release.yaml
```

### Step 4 -- Open a Forgejo PR

PR title: `[PATCH] <component> <old-version> -> <new-version> (CVE-YYYY-XXXXX)`

PR description must include:
```
## Patch: <component> security update

**CVE:** CVE-YYYY-XXXXX
**CVSS:** <score> (<severity>)
**Advisory:** <link>
**Affected versions:** <range>
**Fixed in:** <version>

## Vulnerability Summary
<one paragraph -- what the vulnerability allows an attacker to do>

## Exploitability in This Deployment
<is this component exposed? can the vuln be triggered in our configuration?>

## Changes
- Chart version: <old> -> <new>
- Image tag: <old> -> <new>
- Image digest: <old> -> <new>
- Config changes: <describe any>

## Test Plan
1. <specific test to verify the patch works>
2. Re-run OQ-<relevant test> to confirm no regression
```

### Step 5 -- Deploy via ArgoCD

After PR merge, ArgoCD syncs automatically. Monitor the rollout:

```bash
kubectl rollout status deployment/<app-name> -n <namespace>
argocd app get <app-name> --watch
```

### Step 6 -- Verify and Record

```bash
# Confirm new version is running
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].image}'

# Re-run relevant OQ test case
# For Gatekeeper patches: OQ-01 and OQ-02
# For Falco patches: OQ-03 and OQ-10
# For ArgoCD patches: OQ-06
# For Authentik patches: OQ-04 and OQ-05
```

Record in [[OPS-03 Implementation Log]]:
```
### [DATE] -- PATCH: <component> CVE-YYYY-XXXXX
**Severity:** <score>
**Component:** <name>
**Old version:** <old>
**New version:** <new>
**PR:** Forgejo PR-<number>
**Deploy time:** <minutes from PR merge to running>
**OQ tests re-run:** OQ-XX -- PASS
**Operator:** <name>
```

---

## Patch Procedure -- Emergency (CRITICAL and HIGH)

For CRITICAL CVEs, the normal PR review cycle is compressed but not eliminated. A record must still exist.

### Emergency Patch Steps

```bash
# Step 1: Assess exploitability immediately
# Is the vulnerable component directly exposed? Can the attack be triggered?
# If YES to both: proceed with emergency patch

# Step 2: Take an etcd snapshot BEFORE patching
talosctl etcd snapshot ~/Kuber/snapshots/pre-emergency-patch-$(date +%Y%m%d-%H%M).snapshot \
  --nodes 192.168.0.134 --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Step 3: Apply the patch directly to the cluster (emergency bypass of ArgoCD)
helm upgrade <release-name> <repo>/<chart> \
  --version <new-version> \
  --namespace <namespace> \
  -f apps/<app>/helm-release.yaml

# Step 4: Verify immediately
kubectl rollout status deployment/<app-name> -n <namespace>

# Step 5: WITHIN 24 HOURS: open the Forgejo PR to bring Git in sync with the cluster
# ArgoCD will show OutOfSync until this is done
# The PR description must reference this was an emergency change and why

# Step 6: After PR is merged, ArgoCD will sync and confirm Git matches cluster
argocd app sync <app-name>
```

Record the emergency change immediately in [[OPS-03 Implementation Log]] with:
- Time of discovery
- Time of deployment
- Why normal PR cycle was bypassed
- The Forgejo PR number that documented the change retroactively

---

## Talos OS Upgrade Procedure

Talos upgrades are rolling and managed via talosctl. They do not use Helm.

```bash
# Step 1: Check current version and available releases
talosctl -n 192.168.0.134 version
# Check https://github.com/siderolabs/talos/releases for latest

# Step 2: Review the Talos upgrade notes for breaking changes
# Particularly: Kubernetes compatibility matrix and CNI changes

# Step 3: Take etcd snapshot
talosctl etcd snapshot ~/Kuber/snapshots/pre-talos-upgrade-$(date +%Y%m%d).snapshot \
  --nodes 192.168.0.134 --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig

# Step 4: Upgrade Omen first (control plane)
talosctl upgrade \
  --nodes 192.168.0.134 \
  --endpoints 192.168.0.134 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --image ghcr.io/siderolabs/installer:<new-version>

# Wait for Omen to return -- takes 3-5 minutes
kubectl get nodes -w
# Both nodes should eventually return to Ready

# Step 5: Verify cluster health before upgrading Beelink
kubectl get pods -A | grep -v Running | grep -v Completed
talosctl -n 192.168.0.134 health

# Step 6: Upgrade Beelink
talosctl upgrade \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --image ghcr.io/siderolabs/installer:<new-version>

# Step 7: Post-upgrade verification
kubectl get nodes
talosctl -n 192.168.0.134 version
talosctl -n 192.168.0.202 version

# Step 8: After a Talos upgrade, verify Falco is still functioning
# eBPF probe compatibility may require a Falco update
kubectl get pods -n monitoring -l app.kubernetes.io/name=falco
kubectl logs -n monitoring <falco-pod> | grep -i "driver\|ebpf\|error"
# If Falco shows errors: check if a Falco chart update is needed for the new kernel
```

---

## Patch Tracking Register

Running log of patches applied. Updated each time a patch is applied.

| Date | Component | Old Version | New Version | CVE | Severity | PR | Operator |
|---|---|---|---|---|---|---|---|
| May 2026 | Talos OS | Initial deployment | v1.12.6 | N/A | N/A | N/A | Operator |
| | | | | | | | |

Update this table and the Platform Version Matrix in [[MOD-02 Modularity and Dependency Map]] after each patch.
