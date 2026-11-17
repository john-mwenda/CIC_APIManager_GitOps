# longhorn

Values for the pinned upstream `longhorn` chart. Before adopting an existing release,
compare these files with `helm get values --all <release> -n <namespace>`.
Do not change StatefulSet names, release names, storage classes, or PVC templates
without a backup and an explicit migration.

## StorageClasses

| Class | Replicas | Managed by | Purpose |
|---|---:|---|---|
| `longhorn` (default) | 3 | Longhorn Helm chart | Critical / single-copy data |
| `longhorn-1` | 1 | `manifests/storageclasses/` | Rebuildable / app-replicated data (Loki WAL, Redis) |
| `minio-local` | n/a (Local PV) | `manifests/storageclasses/` | MinIO dedicated SSD Local PVs |

Custom classes live in one file per environment:

- `manifests/storageclasses/staging/00-storageclasses.yaml`
- `manifests/storageclasses/production/00-storageclasses.yaml`

They are synced by the Longhorn Argo CD Applications. Sync Longhorn before MinIO and Loki.
