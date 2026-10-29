# Bootstrap

1. Review chart versions and run `helm get values --all` for existing releases.
2. Create secret material outside Git: `harbor-regcred`, TLS secrets, `redis`,
   `minio-auth`, and `prometheus-grafana`.
3. Apply the environment projects:
   `kubectl apply -f projects/staging.yaml`.
4. Apply the root:
   `kubectl apply -f bootstrap/root-staging.yaml`.
5. Compare each Argo CD diff and sync one child Application at a time.

The root and children use manual sync. This is deliberate for brownfield adoption.
