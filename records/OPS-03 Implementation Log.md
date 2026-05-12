# OPS-03 Implementation Log

> Part of [[README]] | See also: [[OPS-01 Build Instructions]], [[OPS-02 Reference Commands]]

Running log of what I actually built, commands I ran, observations, issues I hit, and exactly how I resolved them. Filled in during the build as things happen -- not reconstructed after the fact.

This log feeds directly into the IQ document in [[PH-07 GxP Validation Documentation]] -- every entry here is a timestamped record of installation activity.

---

## How to Use This Log

Add an entry every time something meaningful happens during the build:
- A phase step is completed
- A command produces an unexpected result
- An issue is encountered
- An issue is resolved
- A design decision is changed mid-build

Format:

```
### [DATE] -- [SHORT DESCRIPTION]
**Phase:** PH-XX
**Status:** Completed / Issue / Resolved / Observation

[What happened, what was run, what the result was]

Commands run:
[exact commands]

Output / Result:
[relevant output or observation]

Resolution (if issue):
[what fixed it]
```

---

## Log

### [2026-05-11] -- Phase 0 completed

**Phase:** PH-00
**Status:** Completed

Cluster is running Talos v1.12.6 / Kubernetes v1.35.2. Live verification showed the Omen still had the control-plane `NoSchedule` taint despite the earlier documentation saying it had been removed. The taint was removed, both nodes were confirmed labeled, and local-path-provisioner was deployed from the GxP platform directory.

HDD mount and storage provisioning were verified:

```bash
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/apps/local-path-provisioner.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/tests/local-hdd-test.yaml

talosctl --talosconfig /Users/omaratabany/Kuber/talos-init/talosconfig \
  --nodes 192.168.0.134 --endpoints 192.168.0.134 \
  etcd snapshot ../snapshots/phase0-complete.snapshot
```

Output:
```
local-hdd is default StorageClass.
local-path-provisioner pod running on talos-asj-72z.
Test PVC bound to PV with selected node talos-v3h-4m1.
Test pod completed on talos-v3h-4m1.
Test log: hdd pvc write test 2026-05-11T13:32:53+00:00
etcd snapshot saved to ../snapshots/phase0-complete.snapshot
snapshot hash c6298587, revision 3782554, total keys 903, total size 17235968
```

The etcd snapshot is intentionally outside the Git repository because it may contain sensitive cluster state.

Issues resolved:
- Provisioner initially could not schedule because the Omen still had the control-plane taint. Removed the taint.
- Provisioner RBAC initially lacked helper pod and event permissions. Added scoped pod create/delete and event create/update/patch permissions.
- local-path helper pod was blocked by baseline PodSecurity because it requires hostPath. Labelled only the `local-path-storage` namespace as privileged and documented the exception.
- Test PVC/PV and HDD directories were cleaned after evidence collection so PH-00 snapshot contains the foundation only, not test artifacts.

---

### [2026-05-11] -- Forgejo deployed

**Phase:** PH-01
**Status:** In progress

Forgejo was deployed through ArgoCD using the upstream Forgejo Helm chart `17.0.1` from `code.forgejo.org/forgejo-helm/forgejo`. The workload is pinned to the Beelink storage node and uses the `local-hdd` StorageClass. Admin credentials were generated locally and committed only as a SealedSecret.

Commands run:
```bash
brew install kubeseal helm

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/apps/homelab-ca-issuer.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/apps/forgejo/namespace.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/apps/forgejo/sealed-secret.yaml

kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig \
  apply -f k8s/apps/forgejo/application.yaml
```

Output / Result:
```
ArgoCD application forgejo: Synced, Healthy
Forgejo health endpoint: status pass
Forgejo pod node: talos-v3h-4m1
Forgejo PVC: forgejo-data, 20Gi, Bound, local-hdd
Forgejo PV selected node: talos-v3h-4m1
Forgejo PV path: /var/mnt/hdd/pvc-ed75640b-14d6-4a4c-a89c-bfbfc2d4b03e_forgejo_forgejo-data
Forgejo image: code.forgejo.org/forgejo/forgejo@sha256:4f4d168b4e792d0f73e5f4da0548f3b54b9c9d03fb85f277c97eb985cb9a290a
Forgejo TLS certificate: Ready
SSH NodePort: 192.168.0.202:32222 reachable
HTTPS through ingress NodePort: https://forgejo.homelab:30550 returns HTTP 200
```

Issues resolved:
- The first ArgoCD OCI source URL used the wrong format. Changed the repo URL from `oci://code.forgejo.org/forgejo-helm` to `code.forgejo.org/forgejo-helm`.
- The initial bootstrap username `admin` was rejected by Forgejo as reserved. Regenerated the SealedSecret with `gxp-admin`.
- Forgejo pod audit warned about missing seccomp. Added `RuntimeDefault` at pod level.

Bootstrap completion:
- Internal Forgejo repository `gxp-admin/gxp-platform` created as private.
- Current platform tree pushed to Forgejo on branch `main`.
- Push webhook configured to `http://argocd-server.argocd.svc.cluster.local/api/webhook`.
- Admin password rotated and sealed after bootstrap.
- Temporary bootstrap tokens removed after repository and webhook setup.
- PH-01 etcd snapshot saved outside Git at `../snapshots/phase1-forgejo-complete.snapshot`. Snapshot hash `091d315d`, revision `3794347`, total keys `982`, size `17235968` bytes.

Resolved network issue:
- MetalLB VIP `192.168.0.200` is reachable from the Mac on ports 80 and 443. `https://forgejo.homelab` through the VIP returns HTTP 200.
- MetalLB L2 advertisement was found pinned to `eno1`. Omen uses `eno1`; Beelink uses `enp1s0`. The desired platform configuration records both interfaces so either node can be used for VIP advertisement if ownership is moved.

---

### [2026-05-12] -- Authentik SSO deployed and wired

**Phase:** PH-02
**Status:** In progress

Authentik was deployed through ArgoCD using chart `2026.2.2`. The server and worker run on the Omen infra node and PostgreSQL runs on the Beelink storage node using `local-hdd`.

Commands run:
```bash
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig apply -f k8s/apps/authentik/namespace.yaml
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig apply -f k8s/apps/authentik/sealed-secret.yaml
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig apply -f k8s/apps/authentik/application.yaml
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig apply -f k8s/apps/platform-network/coredns-homelab.yaml
kubectl --kubeconfig /Users/omaratabany/Kuber/kubeconfig rollout restart deployment/coredns -n kube-system
```

Output / Result:
```
ArgoCD application authentik: Synced, Healthy
Authentik TLS certificate: Ready
Authentik endpoint: https://auth.homelab returns HTTP 302 to the login flow
Authentik PostgreSQL PVC: data-authentik-postgresql-0, 10Gi, Bound, local-hdd
Authentik server image: ghcr.io/goauthentik/server@sha256:40f0df709957c11324420fa387f1135c427f16086f12ca266b2d883d39c71fe3
Authentik PostgreSQL image: docker.io/library/postgres@sha256:47f917f7409eacd22fc5dfb1dee634e1b55cf0c01d1a7eb701be2227a03e0641
```

OIDC providers were created in Authentik for:
- Forgejo: `https://forgejo.homelab/user/oauth2/authentik/callback`
- ArgoCD: `https://argocd.homelab/auth/callback`
- Grafana: `https://grafana.homelab/login/generic_oauth`

Groups created:
- `platform-admin`
- `developer`
- `readonly`

Issues resolved:
- Authentik worker health probes were too tight for the minimal cluster. Increased worker probe timeout and startup failure threshold, then rolled the app.
- Pods could not resolve homelab hostnames. Added a CoreDNS `hosts` block for the MetalLB VIP and restarted CoreDNS.
- Forgejo could not verify Authentik TLS through the homelab CA. Mounted the public homelab root CA into the Forgejo container and set `SSL_CERT_FILE`.

Current remaining work:
- Perform browser login tests for Forgejo, ArgoCD, and Grafana.
- Confirm group claim mapping in live login tokens.
- Disable local password fallback only after SSO login works end to end.
- Migrate the existing monitoring stack under this GXP repo so Grafana SSO is not only a live overlay.
- Add a sealed ArgoCD repository credential for the internal Forgejo mirror or restore GitHub push access before moving the root Application to this GXP repo.

---

*Subsequent entries are added here as the build progresses.*

---

## Issue Tracker

Running list of issues encountered across all phases. Each issue also gets a full entry in the log above.

| Date | Phase | Issue | Status | Resolution |
|---|---|---|---|---|
| 2026-05-11 | PH-00 | Omen still had control-plane NoSchedule taint | Resolved | Removed taint with `kubectl taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-` |
| 2026-05-11 | PH-00 | local-path-provisioner RBAC too narrow for helper pods/events | Resolved | Added scoped pod and event permissions to `local-path-provisioner-role` |
| 2026-05-11 | PH-00 | local-path helper pod blocked by baseline PodSecurity hostPath restriction | Resolved | Labelled `local-path-storage` namespace as privileged and documented the exception |
| 2026-05-11 | PH-01 | ArgoCD OCI source URL format was invalid | Resolved | Changed Forgejo Application repo URL to `code.forgejo.org/forgejo-helm` |
| 2026-05-11 | PH-01 | Forgejo rejected bootstrap username `admin` as reserved | Resolved | Regenerated SealedSecret with `gxp-admin` |
| 2026-05-11 | PH-01 | MetalLB VIP not reachable from Mac on 80 or 443 | Resolved | Reconciled MetalLB L2 config and verified ports 80 and 443 plus Forgejo HTTPS through `192.168.0.200` |
| 2026-05-11 | PH-01 | Temporary Forgejo setup tokens needed for repo bootstrap | Resolved | Created repo and webhook, rotated admin password, and deleted bootstrap tokens |
| 2026-05-12 | PH-02 | Authentik worker probe timeouts on minimal hardware | Resolved | Increased worker probe timeout and startup failure threshold |
| 2026-05-12 | PH-02 | Pods could not resolve homelab hostnames | Resolved | Added CoreDNS `hosts` entries for the MetalLB VIP |
| 2026-05-12 | PH-02 | Forgejo could not verify Authentik TLS | Resolved | Mounted the homelab root CA and set `SSL_CERT_FILE` |
| 2026-05-12 | PH-02 | Grafana is owned by older monitoring ApplicationSet | Open | Captured SSO as a live overlay pending migration into the GXP repo |
| 2026-05-12 | PH-02 | ArgoCD root Application still points to the older home lab repo | Open | Keep current root until repository credentials for the GXP mirror are sealed or GitHub push access is restored |

---

## Phase Completion Checklist

| Phase | Exit Criteria Met | etcd Snapshot Taken | IQ Entry Written | Date |
|---|---|---|---|---|
| PH-00 | [x] | [x] | [x] | 2026-05-11 |
| PH-01 | [x] | [x] | [x] | 2026-05-11 |
| PH-02 | [ ] | [ ] | [ ] | |
| PH-03 | [ ] | [ ] | [ ] | |
| PH-04 | [ ] | [ ] | [ ] | |
| PH-05 | [ ] | [ ] | [ ] | |
| PH-06 | [ ] | [ ] | [ ] | |
| PH-07 | [ ] | [ ] | [ ] | |

---

## Component Version Registry

Filled in as each component is deployed. Used directly in the IQ document.

| Component | Chart Version | Image Digest | Namespace | Deployed | Verified |
|---|---|---|---|---|---|
| local-path-provisioner | n/a | rancher/local-path-provisioner:v0.0.31 | local-path-storage | 2026-05-11 | 2026-05-11 |
| Forgejo | 17.0.1 | code.forgejo.org/forgejo/forgejo@sha256:4f4d168b4e792d0f73e5f4da0548f3b54b9c9d03fb85f277c97eb985cb9a290a | forgejo | 2026-05-11 | 2026-05-11 |
| Authentik | 2026.2.2 | ghcr.io/goauthentik/server@sha256:40f0df709957c11324420fa387f1135c427f16086f12ca266b2d883d39c71fe3 | authentik | 2026-05-12 | 2026-05-12 |
| Authentik PostgreSQL | 16.7.27 | docker.io/library/postgres@sha256:47f917f7409eacd22fc5dfb1dee634e1b55cf0c01d1a7eb701be2227a03e0641 | authentik | 2026-05-12 | 2026-05-12 |
| MinIO | | | minio | | |
| OPA Gatekeeper | | | gatekeeper-system | | |
| Falco | | | monitoring | | |
| Falcosidekick | | | monitoring | | |
| Nextflow (runner version) | | | pipelines | | |

---

## Config Checksum Registry

SHA256 checksums of deployed Helm values files. Used in IQ for configuration integrity verification.

```bash
sha256sum apps/<app>/helm-release.yaml
```

| Component | File | SHA256 | Date |
|---|---|---|---|
| local-path-provisioner | k8s/apps/local-path-provisioner.yaml | 9b304f59e150f333efd87dd465e2231b00dcc07c167fbcf05277e5fa88225afe | 2026-05-11 |
| Forgejo values | k8s/apps/forgejo/values.yaml | b6eb1ca113c2b9e10e13a3b4fe70951a6b443a7f5c489b1a7c326963a91b56ed | 2026-05-11 |
| Forgejo ArgoCD Application | k8s/apps/forgejo/application.yaml | bd055fcfc8120c6d88feecb0cdac91ddd47f9956a2091739d526de930e3a142e | 2026-05-11 |
| Forgejo SealedSecret | k8s/apps/forgejo/sealed-secret.yaml | 5a19c0c734010b4f98dc1228c42a0f0aeb2887f67a09419b0c66bc8c74ddb653 | 2026-05-11 |
| homelab CA issuer | k8s/apps/homelab-ca-issuer.yaml | e074b64f30df844254ff566f51672a46e2155eed596388375cb530808f44bcab | 2026-05-11 |
| Authentik values | k8s/apps/authentik/values.yaml | 2f2877c356e3b1235b5b84a62f194eff40fb298f8562ee0651fcf8697596e9d0 | 2026-05-12 |
| Authentik ArgoCD Application | k8s/apps/authentik/application.yaml | 48c349635a1424354b263d42d51302984e08f2c43aa478a5215d54db928c06fa | 2026-05-12 |
| Authentik SealedSecret | k8s/apps/authentik/sealed-secret.yaml | 6c653b4b10c3afccfd1511e60959d0f3ca0504f08b5d98c41fcad13898da14f1 | 2026-05-12 |
| OIDC client SealedSecrets | k8s/apps/authentik/oidc-client-secrets.yaml | 85d71c56cc289e294d80bee59fe82e4bc874d099ff743e91ef4c08e5f2c90198 | 2026-05-12 |
| ArgoCD OIDC patch | k8s/apps/argocd/argocd-oidc-config-patch.yaml | 4be51d3d02bd4a7fbf943555dcb324c6c4383a57b9aa8906159d780040e1cd3d | 2026-05-12 |
| ArgoCD RBAC patch | k8s/apps/argocd/argocd-rbac-config-patch.yaml | 94859018fa515d1bd1fd1b7c13859d17350ba8c59510edecd6920960670cf1ef | 2026-05-12 |
| CoreDNS homelab hosts | k8s/apps/platform-network/coredns-homelab.yaml | fb9f2ed64f00d4b3d07159c471c1e8b13bc2304d147945a2d4eb129808fccd07 | 2026-05-12 |
| Forgejo homelab CA trust | k8s/apps/forgejo/homelab-ca-configmap.yaml | 07de9513e4836539c39e2619df9c9f0d123b078cfe8f859accbbdc3007f49b6f | 2026-05-12 |
| Grafana OIDC config | k8s/apps/monitoring/grafana-oidc-configmap.yaml | bd7e1edf352cf6e0a80c719c36d9f5a4357cb8ed5d7eea6c300ac5d8f647ed9b | 2026-05-12 |
| Grafana OIDC Deployment patch | k8s/apps/monitoring/grafana-oidc-deployment-patch.yaml | d05c49d3ed5f815a79d150c70765f9930da1af4f1ecba37cf0c4bec57cec15b7 | 2026-05-12 |
| Monitoring homelab CA trust | k8s/apps/monitoring/homelab-ca-configmap.yaml | 8a1f4cafdcd4c97871cf23ac102fa712c7b629604989b7d33f5756877ec0a974 | 2026-05-12 |
| MinIO | apps/minio/helm-release.yaml | | |
| Gatekeeper | apps/gatekeeper/helm-release.yaml | | |
| Falco | apps/falco/helm-release.yaml | | |
