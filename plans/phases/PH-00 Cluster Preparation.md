# PH-00 Cluster Preparation

> Part of [[README]] | Previous: -- | Next: [[PH-01 Forgejo]]
> CKA domains: StorageClass, PV/PVC lifecycle, node labeling, taints/tolerations

**Status: COMPLETE**

---

## Goal

My cluster foundation. Nodes are labeled, the Omen is untainted, the Beelink HDD is mounted and verified. I need the StorageClass confirmed as default and a test PVC proving HDD provisioning works before anything else starts. Every subsequent phase depends on this being solid.

---

## Task List

- [x] Remove control-plane taint from Omen
- [x] Record Beelink actual specs (4 CPU, 7.5GB RAM, sdb 320GB)
- [x] Apply HDD mount patch to Beelink
- [x] Verify HDD mounted as /var/mnt/hdd (sdb1, 320GB XFS, rw)
- [x] Label Omen as infra node
- [x] Label Beelink as storage node
- [x] Create local-path-provisioner manifest in `k8s/apps/local-path-provisioner.yaml`
- [x] Apply local-path-provisioner from the GxP platform directory
- [x] Verify provisioner pod running on Omen
- [x] Run test PVC + pod to confirm HDD provisioning works
- [x] Confirm StorageClass is default
- [x] Take etcd snapshot before proceeding to PH-01

---

## HDD Mount Patch -- Beelink

**Saved at:** `~/Kuber/patches/beelink-hdd.yaml`

```yaml
machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/hdd
  kubelet:
    extraMounts:
      - destination: /var/mnt/hdd
        type: bind
        source: /var/mnt/hdd
        options:
          - bind
          - rshared
          - rw
```

Apply command:
```bash
talosctl patch machineconfig \
  --nodes 192.168.0.202 \
  --endpoints 192.168.0.202 \
  --talosconfig ~/Kuber/talos-init/talosconfig \
  --patch @~/Kuber/patches/beelink-hdd.yaml
```

**Confirmed disk layout:**
```
sda    128GB   GPT   Talos OS (EFI, META, STATE, EPHEMERAL)
sdb    320GB   GPT   HDD -- sdb1 XFS, mounted at /var/mnt/hdd
```

---

## StorageClass Manifest

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-hdd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  nodePath: /var/mnt/hdd
```

`WaitForFirstConsumer` defers PV creation until a pod that needs the PVC is scheduled -- required for node-local storage so the PV lands on the correct node.

**Live implementation note:** local-path-provisioner uses `nodePathMap` in `k8s/apps/local-path-provisioner.yaml` to restrict provisioning to `talos-v3h-4m1:/var/mnt/hdd`. The provisioner controller itself runs on `talos-asj-72z` with `nodeSelector: node-role=infra`. The `local-path-storage` namespace is explicitly labelled `pod-security.kubernetes.io/enforce=privileged` because the helper pod requires a `hostPath` mount to create directories on the Talos-mounted HDD.

---

## Node Labels Applied

```bash
kubectl label node talos-asj-72z node-role=infra
kubectl label node talos-v3h-4m1 node-role=storage
```

Subsequent phases use `nodeSelector: node-role: storage` to pin MinIO, Loki data, and Forgejo PVCs to Beelink. Pipeline compute pods use `node-role: infra` to pin to the Omen.

---

## Exit Criteria

- `kubectl get sc` shows `local-hdd` as default StorageClass -- passed 2026-05-11
- Test PVC binds successfully to Beelink node at `/var/mnt/hdd` -- passed 2026-05-11
- etcd snapshot saved outside the Git repo at `../snapshots/phase0-complete.snapshot` -- passed 2026-05-11

---

## CKA Coverage

StorageClass creation and binding modes, PV/PVC lifecycle and dynamic provisioning, node labeling and nodeSelector, taint removal from control-plane.
