# redis

OpsTree **redis-operator** (`redis-operator` chart `0.24.0`) plus the **`redis-replication`** chart
(`0.17.1`) with embedded Sentinel, on **`longhorn-1`**. Images are public
(`quay.io/opstree/redis`, `redis-sentinel`, `redis-exporter`); no registry login is required.

Two Applications per environment, synced in this order:

1. `redis-operator-<env>` (operator + CRDs, namespace `redis-operator`)
2. `redis-<env>` (the `RedisReplication` resource, namespace `redis`)

## Durability model

| Layer | Setting | Purpose |
|---|---|---|
| App HA | 3 Redis pods + 3 Sentinels, quorum 2 (operator default), master `mymaster` | Survive pod/node failure |
| Writes | Service `redis-master` | Always the current primary |
| Scheduling | Hard pod anti-affinity (`app: redis`, `app: redis-s`) | One Redis pod and one Sentinel per node |
| PDB | `maxUnavailable: 1` | Voluntary disruptions take one pod at a time |
| Resources | Per-env in `staging/helm/redis/values.yaml` and `production/helm/redis/values.yaml` | Staging minimal; production recommended |
| Disk class | `longhorn-1`, 10Gi | Single CSI replica; Redis already replicates |
| AOF | `appendonly yes`, `appendfsync everysec` | ~1 second durability window |
| RDB | `save 900 1` / `300 10` / `60 10000` | Periodic snapshots for restore/backups |
| Memory | `maxmemory` per env (256mb staging, 700mb production), `noeviction` | Writes fail instead of silently evicting |

Replication is asynchronous, so an unplanned primary failure can still lose very recent acknowledged writes.

## Clients

| Use | Address |
|---|---|
| Writes (non-Sentinel clients) | `redis-master.redis.svc.cluster.local:6379` |
| Sentinel | `redis-s-hl.redis.svc.cluster.local:26379`, master `mymaster` |

Authenticate with Secret `redis`, key `redis-password`. After the first sync confirm the names with
`kubectl -n redis get svc,pods --show-labels`.

## Backups

### Option A: Longhorn volume backup

1. Ensure Longhorn has an external backup target (for example Azure Blob).
2. Create a recurring backup/snapshot job for the Redis PVCs in namespace `redis`.
3. Test restore of a PVC into a scratch instance before relying on it.

### Option B: RDB copy

```bash
kubectl -n redis exec -it redis-0 -- redis-cli -a "$REDIS_PASSWORD" BGSAVE
kubectl -n redis cp redis-0:/data/dump.rdb ./redis-dump-$(date +%F).rdb
```

Store the dump outside the cluster.

## Notes

- Secret `redis` (`redis-password`) must exist outside Git.
- The chart disables Sentinel by default; `sentinel.enabled: true` is set in `values-common.yaml`.
  The embedded Sentinel always monitors `mymaster`; the chart has no `masterGroupName`/`quorum` keys.
- `externalConfig` is a top-level chart key (not under `redisReplication`), and its `data` string is
  replaced, not merged, by the per-environment files. Keep it complete there.
- The operator images are rebuilt by OpsTree on their own schedule and can lag upstream Redis
  security releases. Review tags periodically.
- Changing `storageClass` on existing PVCs is not in-place. A previous Redis release does not convert
  to this one; install it before the first sync, or as a new release.
- Upgrading the operator chart does not update CRDs. Apply the CRDs from the release notes first.
