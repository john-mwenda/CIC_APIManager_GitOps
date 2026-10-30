# staging applications

Plain Argo CD `Application` resources for staging. Workloads consume ordinary YAML
directories; platform components consume pinned upstream Helm charts. Applications
use manual sync intentionally. Enable automated sync only after live-state adoption.

Applications are grouped into `platform/`, `monitoring/`, and `workloads/`. Each
directory is managed by its corresponding root Application in
`bootstrap/root-staging.yaml`:

- `cic-staging-platform-root` → `platform/`
- `cic-staging-monitoring-root` → `monitoring/`
- `cic-staging-workloads-root` → `workloads/`
