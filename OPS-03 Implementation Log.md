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

*Subsequent entries are added here as the build progresses.*

---

## Issue Tracker

Running list of issues encountered across all phases. Each issue also gets a full entry in the log above.

| Date | Phase | Issue | Status | Resolution |
|---|---|---|---|---|
| 2026-05-11 | PH-00 | Omen still had control-plane NoSchedule taint | Resolved | Removed taint with `kubectl taint nodes talos-asj-72z node-role.kubernetes.io/control-plane:NoSchedule-` |
| 2026-05-11 | PH-00 | local-path-provisioner RBAC too narrow for helper pods/events | Resolved | Added scoped pod and event permissions to `local-path-provisioner-role` |
| 2026-05-11 | PH-00 | local-path helper pod blocked by baseline PodSecurity hostPath restriction | Resolved | Labelled `local-path-storage` namespace as privileged and documented the exception |

---

## Phase Completion Checklist

| Phase | Exit Criteria Met | etcd Snapshot Taken | IQ Entry Written | Date |
|---|---|---|---|---|
| PH-00 | [x] | [x] | [x] | 2026-05-11 |
| PH-01 | [ ] | [ ] | [ ] | |
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
| Forgejo | | | forgejo | | |
| Authentik | | | authentik | | |
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
| Forgejo | apps/forgejo/helm-release.yaml | | |
| Authentik | apps/authentik/helm-release.yaml | | |
| MinIO | apps/minio/helm-release.yaml | | |
| Gatekeeper | apps/gatekeeper/helm-release.yaml | | |
| Falco | apps/falco/helm-release.yaml | | |
