# staging applications

Plain Argo CD `Application` resources for staging. Workloads consume ordinary YAML
directories; platform components consume pinned upstream Helm charts. Applications
use manual sync intentionally. Enable automated sync only after live-state adoption.

Applications are grouped into `platform/`, `monitoring/`, and `workloads/`. Each
directory is managed by its corresponding root Application in
`staging/bootstrap/root.yaml`:

- `cic-staging-platform-root` → `platform/`
- `cic-staging-monitoring-root` → `monitoring/`
- `cic-staging-workloads-root` → `workloads/`

Sync `kube-prometheus-stack-staging` before `ingress-nginx-staging`. The
ingress-nginx chart creates a ServiceMonitor, and that CRD is installed by
kube-prometheus-stack.
