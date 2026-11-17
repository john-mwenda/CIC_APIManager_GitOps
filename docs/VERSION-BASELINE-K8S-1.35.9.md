# Version baseline for Kubernetes 1.35.9

Pinned Helm chart / application versions for platform and monitoring components.
Staging and production Applications use the same pins.

Target cluster: **Kubernetes 1.35.9** (supported branch; plan next upgrade before EOL).

## Platform

| Component | Chart / source | Chart version | Application version | K8s 1.35.9 notes |
|---|---|---|---|---|
| Longhorn | `longhorn/longhorn` | `1.13.0` | `1.13.0` | Tested with 1.33–1.36; requires Kubernetes ≥ 1.34 |
| MetalLB | `metallb/metallb` | `0.16.2` | `0.16.2` | L2-only (`frrk8s.enabled: false`) |
| Ingress NGINX | `ingress-nginx/ingress-nginx` | `4.15.1` | controller `1.15.1` | E2E-tested on Kubernetes 1.35; project is retired (legacy retain) |
| Strimzi | `strimzi-kafka-operator` | `1.3.1` | `1.3.1` | Supports Kubernetes 1.32–1.37 |
| Apache Kafka | Strimzi-managed | n/a | `4.3.2` | Supported by Strimzi 1.3.1 |
| Redis | OCI `bitnamicharts/redis` | `28.3.10` | `8.10.3` | Image tag pinned in values |
| MinIO | `minio/minio` | `5.4.0` | `RELEASE.2024-12-18T13-15-44Z` | Legacy distributed 4×1 Local PV; validate on 1.35 |

## Monitoring

| Component | Chart / source | Chart version | Application version | K8s 1.35.9 notes |
|---|---|---|---|---|
| kube-prometheus-stack | `prometheus-community` | `93.2.2` | Operator from chart | Requires Kubernetes ≥ 1.25; update CRDs on major bumps |
| Loki | `grafana/loki` | `21.2.3` | `3.7.10` | SimpleScalable + external MinIO |
| Promtail | `grafana/promtail` | `6.17.1` | chart-managed | Works on 1.35; project is EOL — plan Alloy migration |

## Operational notes

1. **Longhorn 1.7.2 → 1.13.0** is not a single-hop upgrade. Use supported intermediate Longhorn versions on brownfield clusters, or install 1.13.0 on a new 1.35.9 cluster.
2. **kube-prometheus-stack** major upgrades require manual CRD updates before sync.
3. **Ingress NGINX** is retained for legacy support; it no longer receives security fixes after project retirement.
4. Apply updated AppProjects before syncing children that use the Bitnami OCI registry (`registry-1.docker.io/bitnamicharts`).
5. Validate in staging: Helm template/diff, PVC binding, ingress, Kafka RF, MinIO quorum, Loki write/read through gateway.
