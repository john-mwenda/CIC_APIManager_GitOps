# Bootstrap

## Staging

1. Review chart versions against `docs/VERSION-BASELINE-K8S-1.35.9.md`
   and run `helm get values --all` for existing releases.
2. Create secret material outside Git: `harbor-regcred`, TLS secrets, `redis`
   (key `redis-password`, used by Redis and Sentinel), `minio-auth`,
   `loki-minio-credentials` (in `monitoring`), and `prometheus-grafana`.
3. Prepare MinIO local disks before syncing MinIO Applications: four worker
   nodes, each with a dedicated SSD mounted at `/data/minio`, and Local PV
   hostnames updated under `<env>/manifests/minio-local-storage/`. Sync Longhorn
   first (creates custom StorageClasses in `<env>/manifests/storageclasses/`), then
   MinIO, then Loki so object-storage buckets and credentials are available.

4. Commit and push the Application directory and root changes. Argo CD cannot read
   changes that exist only in the local checkout.
5. Align every Application `targetRevision` with the published branch. Roots and
   child Applications currently use `master`.
6. Apply the environment projects before the roots that reference them:
   `kubectl apply -f staging/projects.yaml`.
7. Verify the bootstrap project:
   `kubectl get appproject bootstrap-staging -n argocd`.
8. Apply the staging roots:
   `kubectl apply -f staging/bootstrap/root.yaml`.
9. Compare and sync `cic-staging-platform-root` first, then
   `cic-staging-monitoring-root`, and finally `cic-staging-workloads-root`.
10. Compare each child Application diff and sync one child at a time.
    Sync `kube-prometheus-stack-staging` before `ingress-nginx-staging`.
    Ingress NGINX creates a ServiceMonitor, and that CRD comes from
    kube-prometheus-stack.

## Production

Follow the same sequence with production files:

1. `kubectl apply -f production/projects.yaml`
2. `kubectl get appproject bootstrap-production -n argocd`
3. `kubectl apply -f production/bootstrap/root.yaml`
4. Sync `cic-production-platform-root`, then `cic-production-monitoring-root`,
   then `cic-production-workloads-root`
5. Compare each child Application diff and sync one child at a time

The roots and children use manual sync. This is deliberate for brownfield adoption.
Updating an AppProject takes effect when it is applied and does not require an
Argo CD sync command.
