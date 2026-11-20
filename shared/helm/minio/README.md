# minio

Distributed MinIO for a **6 nodes × 1 dedicated SSD** topology.

## Topology

| Setting | Value |
|---|---|
| Mode | `distributed` |
| Replicas / nodes | 6 |
| Drives per node | 1 |
| StorageClass | `minio-local` (static Local PVs, not Longhorn) |
| Erasure coding | `EC:2` |
| Chart | `minio` `5.4.0` |
| Image | `quay.io/minio/minio:RELEASE.2024-12-18T13-15-44Z` |
| Credentials secret | `minio-auth` (`rootUser`, `rootPassword`) |

MinIO erasure coding protects object data. Do **not** place these PVCs on Longhorn.

## Node preparation

On each of the six MinIO nodes (workers and/or labelled control-plane nodes):

1. Label the node so MinIO can select it:
   `kubectl label node <node> workload.cic.io/minio=true`
2. Attach a dedicated SSD (not the OS/etcd disk).
3. Format it (XFS preferred) and mount it at `/data/minio`.
4. Persist the mount in `/etc/fstab`.
5. Ensure the path is empty before the first MinIO sync.
6. Set ownership for UID/GID `1000` to match the chart security context.
7. Confirm `kubectl get nodes -o wide` hostnames match the Local PV `nodeAffinity` values in:
   - `staging/manifests/minio-local-storage/01-persistent-volumes.yaml`
   - `production/manifests/minio-local-storage/01-persistent-volumes.yaml`

Helm values use:

- `nodeSelector: workload.cic.io/minio=true` so only labelled nodes are eligible
- control-plane / master `tolerations` so labelled control-plane nodes can receive pods
- hard pod anti-affinity so one MinIO pod runs per host

Default placeholder hostnames are `api-manager-worker-01` … `api-manager-worker-04`. Replace them before syncing.

PVC `persistence.size` and each Local PV `capacity.storage` must not exceed the usable filesystem size. Both currently use `100Gi`.

## GitOps layout

- Shared Helm values: `shared/helm/minio/values-common.yaml`
- Environment overlays: `staging/helm/minio/values.yaml` and `production/helm/minio/values.yaml`
- StorageClass `minio-local`: `staging/manifests/storageclasses/00-storageclasses.yaml` and `production/manifests/storageclasses/00-storageclasses.yaml` (synced with Longhorn)
- Local PVs: `staging/manifests/minio-local-storage/` and `production/manifests/minio-local-storage/`
- Argo CD MinIO Applications sync Local PVs + the upstream Helm chart

## Sync notes

Before adopting or migrating an existing MinIO release:

1. Back up buckets.
2. Compare live values with `helm get values --all minio -n minio`.
3. Do not change StatefulSet names, release names, storage classes, or PVC templates without an explicit migration plan.
4. Changing from Longhorn-backed PVCs to Local PVs is a data migration, not an in-place StorageClass edit.


## Loki buckets

The chart creates:

- `loki-chunks`
- `loki-ruler`
- `loki-admin`

Point Loki at `minio.minio.svc.cluster.local:9000` and store Loki credentials in
`loki-minio-credentials` outside Git.
