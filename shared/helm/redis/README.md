# redis

Bitnami Redis (replication + Sentinel) on **`longhorn-1`**.

## Durability model

| Layer | Setting | Purpose |
|---|---|---|
| App HA | Sentinel mode: 3 `redis-node` pods | Survive pod/node failure |
| Scheduling | `podAntiAffinityPreset: hard` | One Redis pod per worker (3 workers) |
| Resources | Per-env in `staging/helm/redis/values.yaml` and `production/helm/redis/values.yaml` | Staging minimal; production recommended |
| Disk class | `longhorn-1` | Single CSI replica; avoid 3× Longhorn under Redis RF |
| AOF | `appendonly yes`, `appendfsync everysec` | ~1 second durability window |
| RDB | `save 900 1` / `300 10` / `60 10000` | Periodic snapshots for restore/backups |

Redis replication is asynchronous, so a primary failure can still lose very recent acknowledged writes. AOF + RDB reduce that window; backups cover disaster recovery.

## Backups

### Option A — Longhorn volume backup (recommended with your platform)

1. Ensure Longhorn has an external backup target (for example Azure Blob).
2. Create a recurring backup/snapshot job for Redis PVCs (`data-redis-*` in namespace `redis`).
3. Test restore of a PVC into a scratch Redis before relying on it.

### Option B — Application-level RDB copy

```bash
kubectl -n redis exec -it redis-node-0 -- redis-cli -a "$REDIS_PASSWORD" BGSAVE
kubectl -n redis cp redis-node-0:/data/dump.rdb ./redis-dump-$(date +%F).rdb
```

Store the dump outside the cluster (object storage / backup share).

### Option C — Velero

Include namespace `redis` and its PVCs in Velero schedules if Velero is already used.

## Clients

Sentinel is enabled, so Bitnami does not create a `redis-master` Service.
The Redis Service is `redis.redis.svc.cluster.local` (port 6379). Sentinel
clients use `redis-headless.redis.svc.cluster.local:26379` and master name
`mymaster`.

## Notes

- Secret `redis` (`redis-password`) must exist outside Git.
- Staging pulls Bitnami Secure Images (`8.10.3-photon-5-r0`, kubectl
  `1.37.1-r9`) and requires Secret `bitnami-regcred` in namespace `redis`.
- Changing `storageClass` on existing PVCs is not in-place; plan migration or new volumes.
- Compare live values with `helm get values --all redis -n redis` before adopting.
