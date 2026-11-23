# Version baseline for Kubernetes 1.35.9

Pinned Helm chart / application versions for platform and monitoring components.
Staging and production Applications use the same pins.

Target cluster: **Kubernetes 1.35.9** (supported branch; plan next upgrade before EOL).

Image tags are also pinned in `shared/helm/*/values-common.yaml` so releases do not
follow floating chart defaults.

## Platform

| Component | Chart / source | Chart version | Image pin(s) | K8s 1.35.9 notes |
|---|---|---|---|---|
| Longhorn | `longhorn/longhorn` | `1.13.0` | engine/manager/ui/IM/share/BIM `v1.13.0`; CSI + support-bundle kit pinned | Requires Kubernetes ≥ 1.34 |
| MetalLB | `metallb/metallb` | `0.16.2` | controller/speaker `v0.16.2` | L2-only (`frrk8s.enabled: false`) |
| Ingress NGINX | `ingress-nginx/ingress-nginx` | `4.15.1` | controller `v1.15.1`; webhook `v1.6.9` | E2E-tested on Kubernetes 1.35; project retired (legacy retain) |
| Strimzi | `strimzi-kafka-operator` | `1.3.1` | operator `1.3.1` | Supports Kubernetes 1.32–1.37 |
| Apache Kafka | Strimzi-managed | n/a | Kafka `version: 4.3.1` in CR | Supported by Strimzi 1.3.1 |
| Redis Operator (OpsTree) | `ot-container-kit` `redis-operator` | `0.24.0` | operator `0.24.0` | Install first; installs CRDs. Upgrades need manual CRD apply |
| Redis (OpsTree) | `ot-container-kit` `redis-replication` | `0.17.1` | `quay.io/opstree/redis` + `redis-sentinel` `v7.2.13`; exporter `v1.44.0` | Embedded Sentinel, 3 replicas, `longhorn-1`. Public images, no registry login |
| MinIO (Silo) | `pgsty/silo` `helm/silo`, image `docker.io/pgsty/silo` | `7.0.5` | server and mc jobs `RELEASE.2026-11-12T00-00-00Z` | `nameOverride`/`fullnameOverride` keep Service `minio`; standard image, not distroless |

## Monitoring

| Component | Chart / source | Chart version | Image pin(s) | K8s 1.35.9 notes |
|---|---|---|---|---|
| kube-prometheus-stack | `prometheus-community` | `93.2.2` | Prometheus `v3.16.0-distroless`; Grafana `13.3.1-distroless`; operator/reloader `v0.95.2`; Alertmanager `v0.35.0`; kube-state-metrics `v2.20.0`; node-exporter `v1.12.1-distroless` | Update CRDs on major bumps |
| Loki | OCI `ghcr.io/grafana-community/helm-charts` | `21.2.3` | Loki `3.7.10`; gateway nginx `1.31-alpine` | SimpleScalable + external MinIO |
| Alloy | `grafana/alloy` | `1.16.0` | `v1.22.0` | DaemonSet log collector; replaces Promtail |

## Operational notes

1. **Longhorn 1.7.2 → 1.13.0** is not a single-hop upgrade. Use supported intermediate Longhorn versions on brownfield clusters, or install 1.13.0 on a new 1.35.9 cluster.
2. **kube-prometheus-stack** major upgrades require manual CRD updates before sync.
3. **Ingress NGINX** is retained for legacy support; it no longer receives security fixes after project retirement.
4. Apply updated AppProjects before syncing children that use the OpsTree Helm repository (`https://ot-container-kit.github.io/helm-charts/`) or Loki OCI (`ghcr.io/grafana-community/helm-charts`).
5. Validate in staging: Helm template/diff, PVC binding, ingress, Kafka RF, MinIO quorum, Loki write/read through gateway.
6. Prefer digest pins later if your registry allows retags of mutable tags.
