# longhorn

Values for the pinned upstream `longhorn` chart. Before adopting an existing release,
compare these files with `helm get values --all <release> -n <namespace>`.
Do not change StatefulSet names, release names, storage classes, or PVC templates
without a backup and an explicit migration.

## StorageClasses

| Class | Replicas | Managed by | Purpose |
|---|---:|---|---|
| `longhorn` (default) | 3 | Longhorn Helm chart | Critical / single-copy data |
| `longhorn-1` | 1 | `<env>/manifests/storageclasses/` | Rebuildable / app-replicated data (Loki WAL, Redis) |
| `minio-local` | n/a (Local PV) | `<env>/manifests/storageclasses/` | MinIO dedicated SSD Local PVs |

Custom classes live in one file per environment:

- `staging/manifests/storageclasses/00-storageclasses.yaml`
- `production/manifests/storageclasses/00-storageclasses.yaml`

They are synced by the Longhorn Argo CD Applications. Sync Longhorn before MinIO and Loki.

## Control plane nodes

The chart does not tolerate the kubeadm control-plane taint. `values-common.yaml` adds that toleration for user-deployed pods (`global.tolerations`) and for system-managed pods (`defaultSettings.taintToleration`).

Disks are created only on nodes labeled `node.longhorn.io/create-default-disk=true`, and only at `/data/longhorn`. Label a control plane after a dedicated disk is mounted there. Leave the label off a node whose only disk also holds etcd.

```bash
kubectl label node <node> node.longhorn.io/create-default-disk=true
```
