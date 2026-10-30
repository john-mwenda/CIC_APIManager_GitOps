# CIC API Manager GitOps

Argo CD GitOps repository for managing deployments and configurations across the CIC API Manager `staging` and `production` Kubernetes clusters.

## Layout

- `bootstrap/`: manually applied root Applications per environment.
- `applications/staging/{platform,monitoring,workloads}/`: staging child Applications.
- `applications/production/{platform,monitoring,workloads}/`: production child Applications.
- `manifests/`: plain, directly applicable Kubernetes YAML per environment.
- `helm/`: values for pinned upstream Helm charts.
- `projects/`: environment-scoped AppProject permissions.
- `docs/`: bootstrap and brownfield migration procedures.

The repository uses the app-of-apps pattern. Each environment uses three root
Applications, one per category directory under `applications/<env>/`. Each root
creates and updates only that category's child `Application` resources. Each child
references an AppProject that restricts its allowed source repositories,
destinations, and resource kinds. AppProjects authorize deployments; they do not
deploy resources or control sync order.

## Validate

```powershell
kubectl apply --dry-run=client -f manifests/api-manager/staging/
kubectl apply --dry-run=client --recursive -f applications/staging/
helm template redis bitnami/redis -f helm/redis/values-common.yaml -f helm/redis/values-staging.yaml
```

The workload manifests use `hashicorp/http-echo:1.0.0` so the sample can run without
access to CIC's private registry. Replace each image with its approved Harbor digest.
Secret manifests are intentionally absent; provision them using a secret manager.

## Getting Started

Argo CD must already be running in the target cluster. The Git repository must be
reachable from the Argo CD repo-server; Argo CD cannot read a local workstation
directory.

### 1. Configure and publish the repository

Replace the example repository URL in `bootstrap/`, `applications/`, and `projects/`.
Align every Application `targetRevision` with the branch you publish. Roots and child
Applications currently use `master`.

```bash
git add .
git commit -m "Configure staging GitOps repository"
git push origin master
```

### 2. Register a private repository

Public repositories require no credentials. For private HTTPS Git:

```bash
argocd repo add https://gitlab.example.com/platform/cic-gitops.git \
  --username YOUR_USERNAME \
  --password YOUR_ACCESS_TOKEN

argocd repo list
```

Use a deploy key or token with read-only repository access. Do not store credentials
in this repository. For a private CA, register the CA certificate with Argo CD rather
than disabling TLS verification.

### 3. Apply the staging AppProjects

AppProjects authorize repositories, destination namespaces, and resource kinds:

```bash
kubectl apply -f projects/staging.yaml
kubectl get appprojects -n argocd
```

`projects/staging.yaml` creates these staging-specific projects:

- `bootstrap-staging`
- `platform-staging`
- `monitoring-staging`
- `workloads-staging`

The `argocd` namespace will also normally contain Argo CD's built-in `default`
project and may contain projects for other environments.

The repository URL in each project's `sourceRepos` must exactly match the
Application `repoURL`.

The staging root Applications use `bootstrap-staging`, which only permits
Application resources from this repository to be deployed into the `argocd`
namespace. Apply AppProject changes before applying the roots that reference them.
No Argo CD sync command is required for an AppProject update itself.

### 4. Apply and synchronize the root Applications

Commit and push the Application directory and root changes before synchronizing;
Argo CD reads them from the remote repository rather than the local checkout.

```bash
kubectl apply -f bootstrap/root-staging.yaml
kubectl get applications -n argocd

argocd app get cic-staging-platform-root
argocd app diff cic-staging-platform-root
argocd app sync cic-staging-platform-root

argocd app diff cic-staging-monitoring-root
argocd app sync cic-staging-monitoring-root

argocd app diff cic-staging-workloads-root
argocd app sync cic-staging-workloads-root
```

Each root reads one directory:

- `cic-staging-platform-root` → `applications/staging/platform`
- `cic-staging-monitoring-root` → `applications/staging/monitoring`
- `cic-staging-workloads-root` → `applications/staging/workloads`

Sync the roots in that order. They create or update child Application resources but
do not sync those children automatically.

`projects/staging.yaml` is outside the roots' source paths and is therefore applied
manually whenever its configuration changes.

### 5. Synchronize child Applications

Review every diff before synchronization. A suitable dependency order is:

```bash
argocd app diff longhorn-staging
argocd app sync longhorn-staging

argocd app diff metallb-staging
argocd app sync metallb-staging
argocd app wait metallb-staging --health
argocd app diff metallb-config-staging
argocd app sync metallb-config-staging

argocd app diff ingress-nginx-staging
argocd app sync ingress-nginx-staging

argocd app diff strimzi-kafka-operator-staging
argocd app sync strimzi-kafka-operator-staging
argocd app wait strimzi-kafka-operator-staging --health
argocd app diff kafka-cluster-staging
argocd app sync kafka-cluster-staging

argocd app diff redis-staging
argocd app sync redis-staging
argocd app diff minio-staging
argocd app sync minio-staging

argocd app diff kube-prometheus-stack-staging
argocd app sync kube-prometheus-stack-staging
argocd app diff loki-staging
argocd app sync loki-staging
argocd app diff promtail-staging
argocd app sync promtail-staging

argocd app diff api-manager-staging
argocd app sync api-manager-staging
argocd app diff middleware-staging
argocd app sync middleware-staging
argocd app diff external-portal-staging
argocd app sync external-portal-staging
argocd app diff ingress-routes-staging
argocd app sync ingress-routes-staging
```

MetalLB configuration requires the MetalLB CRDs. Kafka resources require the
Strimzi operator and its CRDs.

YAML file order does not determine synchronization order. The roots and children use
manual sync for brownfield adoption, so the commands above provide the current
workload ordering. Argo CD sync waves only order resources within the same
Application. The three separate roots do not enforce ordering across one another.

### 6. Verify

```bash
argocd app list
kubectl get applications -n argocd
kubectl get pods -A
kubectl get pvc -A
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp
```

Before syncing against an existing cluster, compare Helm versions, release names,
values, selectors, and PVC templates with the live resources. Also configure the
required external Secrets, verify the MetalLB address pool, and replace the sample
images with approved CIC Harbor image digests.

## Production

Production mirrors the staging layout:

1. Apply `projects/production.yaml` (`bootstrap-production`, `platform-production`,
   `monitoring-production`, `workloads-production`).
2. Apply and sync the three roots from `bootstrap/root-production.yaml` in order:
   `cic-production-platform-root`, `cic-production-monitoring-root`,
   `cic-production-workloads-root`.
3. Review diffs and sync child Applications with the same dependency order as
   staging, using the `-production` Application names.
