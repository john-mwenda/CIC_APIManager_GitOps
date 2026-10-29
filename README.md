# CIC API Manager GitOps

Argo CD repository for staging and production CIC API Manager Kubernetes clusters.

## Layout

- `bootstrap/`: one manually applied root Application per environment.
- `applications/`: child Argo CD Applications; no Kustomize is used.
- `manifests/`: plain, directly applicable Kubernetes YAML per environment.
- `helm/`: values for pinned upstream Helm charts.
- `projects/`: environment-scoped Argo CD AppProjects.
- `docs/`: bootstrap and brownfield migration procedures.

## Validate

```powershell
kubectl apply --dry-run=client -f manifests/api-manager/staging/
kubectl apply --dry-run=client -f applications/staging/
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

Replace the example repository URL in `bootstrap/`, `applications/`, and `projects/`:

```bash
git add .
git commit -m "Configure staging GitOps repository"
git push origin main
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

Expected projects:

- `workloads-staging`
- `platform-staging`
- `monitoring-staging`

The repository URL in each project's `sourceRepos` must exactly match the
Application `repoURL`.

The root Application uses the Argo CD `default` project. Verify that it allows this
repository and the in-cluster `argocd` destination:

```bash
kubectl get appproject default -n argocd -o yaml
```

If the default project is restricted, permit the repository and destination or
create a dedicated bootstrap AppProject.

### 4. Apply and synchronize the root Application

```bash
kubectl apply -f bootstrap/root-staging.yaml
kubectl get applications -n argocd

argocd app get cic-staging-root
argocd app diff cic-staging-root
argocd app sync cic-staging-root
```

The root reads `applications/staging/` and creates the child Applications. Sync is
manual by design, so creating an Application does not immediately deploy its
resources.

`projects/staging.yaml` is outside the root's source path and is therefore applied
manually whenever its configuration changes.

### 5. Synchronize child Applications

Review every diff before synchronization. A suitable dependency order is:

```bash
argocd app sync longhorn-staging

argocd app sync metallb-staging
argocd app wait metallb-staging --health
argocd app sync metallb-config-staging

argocd app sync ingress-nginx-staging

argocd app sync strimzi-kafka-operator-staging
argocd app wait strimzi-kafka-operator-staging --health
argocd app sync kafka-cluster-staging

argocd app sync redis-staging
argocd app sync minio-staging

argocd app sync kube-prometheus-stack-staging
argocd app sync loki-staging
argocd app sync promtail-staging

argocd app sync api-manager-staging
argocd app sync middleware-staging
argocd app sync external-portal-staging
argocd app sync ingress-routes-staging
```

MetalLB configuration requires the MetalLB CRDs. Kafka resources require the
Strimzi operator and its CRDs.

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
