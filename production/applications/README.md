# production applications

Plain Argo CD `Application` resources for production. Workloads consume ordinary YAML
directories; platform components consume pinned upstream Helm charts. Applications
use manual sync intentionally. Enable automated sync only after live-state adoption.

Applications are grouped into `platform/`, `monitoring/`, and `workloads/`. Each
directory is managed by its corresponding root Application in
`production/bootstrap/root.yaml`:

- `cic-production-platform-root` → `platform/`
- `cic-production-monitoring-root` → `monitoring/`
- `cic-production-workloads-root` → `workloads/`

Sync `kube-prometheus-stack-production` before `ingress-nginx-production`. The
ingress-nginx chart creates a ServiceMonitor, and that CRD is installed by
kube-prometheus-stack.
