# alloy

Grafana Alloy (`grafana/alloy` chart `1.16.0`, image `v1.22.0`) replaces Promtail.
It runs as a DaemonSet and pushes pod logs to the Loki gateway.

Before adopting an existing release, compare these files with
`helm get values --all alloy -n monitoring`.

Remove the old `promtail` Helm release before the first Alloy sync. Leaving both
running sends every log line to Loki twice.

```bash
argocd app delete promtail-staging --yes
```
