# loki

SimpleScalable Loki using the platform MinIO cluster for object storage.

## Topology

| Component | Replicas | Storage |
|---|---:|---|
| `write` | 2 | MinIO chunks + `longhorn-1` WAL PVC (10Gi) |
| `read` | 2 | MinIO |
| `backend` | 2 | MinIO + `longhorn-1` PVC (10Gi) |
| `gateway` | 1 | Stateless |
| `singleBinary` | 0 | Disabled |

| Item | Value |
|---|---|
| `deploymentMode` | `SimpleScalable` |
| `replication_factor` | 2 |
| Object store | `minio.minio.svc.cluster.local:9000` |
| Buckets | `loki-chunks`, `loki-ruler`, `loki-admin` |
| Chart source | OCI `ghcr.io/grafana-community/helm-charts` (`21.2.3`) |
| Loki image | `grafana/loki:3.7.10` (pinned in values) |
| Built-in MinIO | Not used (platform MinIO endpoint only) |

## Credentials

Create Secret `loki-minio-credentials` in namespace `monitoring` (not in Git):

```bash
kubectl -n monitoring create secret generic loki-minio-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='<minio-access-key>' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<minio-secret-key>'
```

The Secret is mounted into write, read, and backend pods via `extraEnvFrom`.

## Client endpoint

Point Alloy at the **gateway** Service, not a singleBinary Service.

Staging exposes the gateway on MetalLB IP `192.168.124.226`.

## Sync order

1. MinIO Application healthy
2. Loki buckets present
3. Create `loki-minio-credentials`
4. Sync Loki

## Migration note

Moving from SingleBinary filesystem/Longhorn to SimpleScalable + MinIO does not migrate historical chunks.
Compare live values with `helm get values --all loki -n monitoring` before adopting.
