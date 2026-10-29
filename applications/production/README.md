# production applications

Plain Argo CD `Application` resources for production. Workloads consume ordinary YAML
directories; platform components consume pinned upstream Helm charts. Applications
use manual sync intentionally. Enable automated sync only after live-state adoption.
