# Bootstrap

## Staging

1. Review chart versions and run `helm get values --all` for existing releases.
2. Create secret material outside Git: `harbor-regcred`, TLS secrets, `redis`,
   `minio-auth`, and `prometheus-grafana`.
3. Commit and push the Application directory and root changes. Argo CD cannot read
   changes that exist only in the local checkout.
4. Align every Application `targetRevision` with the published branch. Roots and
   child Applications currently use `master`.
5. Apply the environment projects before the roots that reference them:
   `kubectl apply -f projects/staging.yaml`.
6. Verify the bootstrap project:
   `kubectl get appproject bootstrap-staging -n argocd`.
7. Apply the staging roots:
   `kubectl apply -f bootstrap/root-staging.yaml`.
8. Compare and sync `cic-staging-platform-root` first, then
   `cic-staging-monitoring-root`, and finally `cic-staging-workloads-root`.
9. Compare each child Application diff and sync one child at a time.

## Production

Follow the same sequence with production files:

1. `kubectl apply -f projects/production.yaml`
2. `kubectl get appproject bootstrap-production -n argocd`
3. `kubectl apply -f bootstrap/root-production.yaml`
4. Sync `cic-production-platform-root`, then `cic-production-monitoring-root`,
   then `cic-production-workloads-root`
5. Compare each child Application diff and sync one child at a time

The roots and children use manual sync. This is deliberate for brownfield adoption.
Updating an AppProject takes effect when it is applied and does not require an
Argo CD sync command.
