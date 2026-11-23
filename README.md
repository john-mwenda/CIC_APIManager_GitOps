# CIC API Manager GitOps

Argo CD GitOps repository for managing deployments and configurations across the CIC API Manager `staging` and `production` Kubernetes clusters.

## Layout

- `shared/helm/<chart>/`: `values-common.yaml` shared by both clusters, plus the chart notes.
- `staging/` and `production/`: one directory per environment.
  - `applications/{platform,monitoring,workloads}/`: child Applications.
  - `manifests/<component>/`: plain Kubernetes YAML.
  - `helm/<chart>/values.yaml`: environment overlay merged after `values-common.yaml`.
  - `bootstrap/root.yaml`: manually applied root Applications.
  - `projects.yaml`: environment-scoped AppProject permissions.
- `docs/`: bootstrap, migration, and the Kubernetes **1.35.9** version baseline.

Platform and monitoring Helm pins target **Kubernetes 1.35.9**. See
`docs/VERSION-BASELINE-K8S-1.35.9.md`.

The repository uses the app-of-apps pattern. Each environment uses three root
Applications, one per category directory under `<env>/applications/`. Each root
creates and updates only that category's child `Application` resources. Each child
references an AppProject that restricts its allowed source repositories,
destinations, and resource kinds. AppProjects authorize deployments; they do not
deploy resources or control sync order.

## Validate

```powershell
kubectl apply --dry-run=client -f staging/manifests/api-manager/
kubectl apply --dry-run=client --recursive -f staging/applications/
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm template redis ot-helm/redis-replication --version 0.17.1 -n redis -f shared/helm/redis/values-common.yaml -f staging/helm/redis/values.yaml
```

The workload manifests use `hashicorp/http-echo:1.0.0` so the sample can run without
access to CIC's private registry. Replace each image with its approved Harbor digest.
Secret manifests are intentionally absent from Git; create them on the cluster before
syncing Applications (see [Required Secrets](#4-required-secrets)).

## Getting Started

Argo CD must already be running in the target cluster. The Git repository must be
reachable from the Argo CD repo-server; Argo CD cannot read a local workstation
directory.

### Pre-requisite - Setup argocd on the new cluster
Follow this docs to setup argocd on cluster
https://argo-cd.readthedocs.io/en/latest/getting_started/

### 1. Configure and publish the repository

Align every Application `targetRevision` with the desired branch. Roots and child
Applications currently use `master`.

```bash
git add .
git commit -m "Configure staging GitOps repository"
git push origin master
```

### 2. Register CIC API Manager GitOps repository

For private HTTPS Git:

```bash
argocd repo add https://github.com/john-mwenda/CIC_APIManager_GitOps.git \
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
kubectl apply -f staging/projects.yaml
kubectl get appprojects -n argocd
```

`staging/projects.yaml` creates these staging-specific projects:

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

### 4. Required Secrets

Secrets are not stored in this repository. Create the namespaces (if they do not
already exist) and Secrets **before** syncing the Applications that consume them.
Replace every placeholder value with approved credentials from your secret manager.

```bash
# Namespaces used by platform, monitoring, and workload Secrets
kubectl create namespace redis --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace minio --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace api-manager --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace middleware --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace external-portal --dry-run=client -o yaml | kubectl apply -f -
```

#### Platform

**Redis** (`shared/helm/redis` → Secret `redis`, key `redis-password`), run by the OpsTree
redis-operator with embedded Sentinel. The same password is used by Redis and Sentinel.
The images are public (`quay.io/opstree`); no registry secret is required.

```bash
kubectl -n redis create secret generic redis \
  --from-literal=redis-password='CHANGE_ME_REDIS_PASSWORD'
```

**MinIO** (`shared/helm/minio` → `existingSecret: minio-auth`, keys `rootUser` / `rootPassword`):

```bash
kubectl -n minio create secret generic minio-auth \
  --from-literal=rootUser='CHANGE_ME_MINIO_ROOT_USER' \
  --from-literal=rootPassword='CHANGE_ME_MINIO_ROOT_PASSWORD'
```

#### Monitoring

**Loki → MinIO credentials** (`shared/helm/loki` → Secret `loki-minio-credentials`).
Use a MinIO access key that can read/write the `loki-chunks`, `loki-ruler`, and
`loki-admin` buckets (often the same values as `minio-auth` during bootstrap):

```bash
kubectl -n monitoring create secret generic loki-minio-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='CHANGE_ME_MINIO_ROOT_USER' \
  --from-literal=AWS_SECRET_ACCESS_KEY='CHANGE_ME_MINIO_ROOT_PASSWORD'
```

**Grafana admin** (`staging/helm/kube-prometheus-stack/values.yaml` and `production/helm/kube-prometheus-stack/values.yaml` →
`grafana.admin.existingSecret: prometheus-grafana`, keys `admin-user` /
`admin-password`):

```bash
kubectl -n monitoring create secret generic prometheus-grafana \
  --from-literal=admin-user='admin' \
  --from-literal=admin-password='CHANGE_ME_GRAFANA_ADMIN_PASSWORD'
```

**Grafana SMTP** (`shared/helm/kube-prometheus-stack` →
`grafana.smtp.existingSecret: prometheus-grafana-smtp`, keys `user` / `password`):

```bash
kubectl -n monitoring create secret generic prometheus-grafana-smtp \
  --from-literal=user='CHANGE_ME_SMTP_USER' \
  --from-literal=password='CHANGE_ME_SMTP_PASSWORD'
```

#### Workloads (image pulls and Ingress TLS)

**Harbor pull credentials** (referenced as `imagePullSecrets: harbor-regcred` in
`api-manager`, `middleware`, and `external-portal` manifests). Create the same
Secret in each workload namespace:

```bash
kubectl -n api-manager create secret docker-registry harbor-regcred \
  --docker-server='harbor.example.cic.co.ke' \
  --docker-username='CHANGE_ME_HARBOR_USER' \
  --docker-password='CHANGE_ME_HARBOR_PASSWORD' \
  --docker-email='platform@cic.co.ke'

kubectl -n middleware create secret docker-registry harbor-regcred \
  --docker-server='harbor.example.cic.co.ke' \
  --docker-username='CHANGE_ME_HARBOR_USER' \
  --docker-password='CHANGE_ME_HARBOR_PASSWORD' \
  --docker-email='platform@cic.co.ke'

kubectl -n external-portal create secret docker-registry harbor-regcred \
  --docker-server='harbor.example.cic.co.ke' \
  --docker-username='CHANGE_ME_HARBOR_USER' \
  --docker-password='CHANGE_ME_HARBOR_PASSWORD' \
  --docker-email='platform@cic.co.ke'
```

**Ingress TLS** (referenced by workload Ingress resources):

```bash
kubectl -n api-manager create secret tls api-manager-tls \
  --cert=./api-manager.crt \
  --key=./api-manager.key

kubectl -n middleware create secret tls api-manager-middleware-tls \
  --cert=./api-manager-middleware.crt \
  --key=./api-manager-middleware.key

kubectl -n external-portal create secret tls api-manager-external-tls \
  --cert=./api-manager-external.crt \
  --key=./api-manager-external.key
```

| Secret | Namespace | Consumed by |
|---|---|---|
| `redis` | `redis` | Redis (OpsTree operator) Application (`redis-password`) |
| `minio-auth` | `minio` | MinIO Application |
| `loki-minio-credentials` | `monitoring` | Loki Application |
| `prometheus-grafana` | `monitoring` | kube-prometheus-stack (Grafana admin) |
| `prometheus-grafana-smtp` | `monitoring` | kube-prometheus-stack (Grafana SMTP) |
| `harbor-regcred` | `api-manager`, `middleware`, `external-portal` | Workload image pulls |
| `api-manager-tls` | `api-manager` | API Manager Ingress |
| `api-manager-middleware-tls` | `middleware` | Middleware Ingress |
| `api-manager-external-tls` | `external-portal` | External portal Ingress |

Create the same Secrets (with production values) before syncing production
Applications. Prefer a secret manager or sealed-secrets / External Secrets over
long-lived workstation shell history.

### 5. Required ConfigMaps

API Manager and Middleware pods load shared settings from ConfigMap
`api-manager-common` via `envFrom.configMapRef`. Create these ConfigMaps in the
correct namespaces **before** syncing the `api-manager` and `middleware`
Applications (namespaces should already exist from step 4).

Replace every sensitive value with approved environment credentials. Prefer
moving passwords, tokens, and API keys into Secrets later; do not commit live
credentials.

#### Create config from an env file

**api-manager** namespace (`api-manager-common`):

```bash
# Write api-manager-common.env with KEY=value lines, then:
kubectl -n api-manager create configmap api-manager-common \
  --from-env-file=./api-manager-common.env
```

Sample `api-manager-common.env` keys (replace `CHANGE_ME_*`):

```bash
cat > api-manager-common.env <<'EOF'
API_GATEWAY_URL=https://staging-api-manager.cic.co.ke
CORS_EXTERNAL_PORTAL=https://staging-external-portal.cic.co.ke
CORS_INTERNAL_PORTAL=https://staging-internal-portal.cic.co.ke
DB_PASSWORD=CHANGE_ME_DB_PASSWORD
DB_URI=postgresql://CHANGE_ME_DB_HOST:5432/esbdb
DB_USERNAME=CHANGE_ME_DB_USERNAME
EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
EUREKA_INSTANCE_PREFERIPADDRESS=true
EUREKA_SERVER=http://eureka-server:8761/eureka/
GRAFANA_API_TOKEN=CHANGE_ME_GRAFANA_API_TOKEN
GRAFANA_BASE_URL=http://CHANGE_ME_GRAFANA_HOST:30080
GRAFANA_PG_DATASOURCE_UID=CHANGE_ME_GRAFANA_PG_DATASOURCE_UID
KAFKA_BOOTSTRAP_SERVER=my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
KAFKA_BROKER=my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
MIDDLEWARE_URL=http://api-gateway-middleware-service.middleware.svc.cluster.local:8080
MINIO_ACCESS_KEY=CHANGE_ME_MINIO_ACCESS_KEY
MINIO_BUCKET_NAME=ssl-files
MINIO_ENDPOINT=http://minio.minio.svc.cluster.local:9000
MINIO_SECRET_KEY=CHANGE_ME_MINIO_SECRET_KEY
MONGO_DB_HOST=CHANGE_ME_MONGO_HOST
MONGO_DB_NAME=admin
MONGO_DB_PASSWORD=CHANGE_ME_MONGO_PASSWORD
MONGO_DB_PORT=27017
MONGO_DB_URI=mongodb://CHANGE_ME_MONGO_USER:CHANGE_ME_MONGO_PASSWORD@CHANGE_ME_MONGO_HOST:27017/admin
MONGO_DB_USER=CHANGE_ME_MONGO_USER
PESALINK_ACCOUNT_VERIFICATION_SERVICE_ID=901
PESALINK_HEX_KEY=CHANGE_ME_PESALINK_HEX_KEY
PESALINK_LOGIN_ID=CHANGE_ME_PESALINK_LOGIN_ID
PESALINK_POSTING_SERVICE_ID=2007
PESALINK_SOURCE_ACCOUNT_NUMBER=CHANGE_ME_PESALINK_SOURCE_ACCOUNT
PESALINK_SOURCE_BANK_CODE=CHANGE_ME_PESALINK_SOURCE_BANK_CODE
PESALINK_TSQ_SERVICE_ID=1943
PESALINK_URL=https://CHANGE_ME_PESALINK_HOST/pgw/webservices/PaymentWs
REDIS_MASTER=mymaster
REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD
REDIS_PORT=6379
REDIS_PRIMARY_NODE=redis-s-hl.redis.svc.cluster.local:26379
REDIS_SECONDARY_NODE=redis-s-hl.redis.svc.cluster.local:26379
ROUTES_TRIGGER=true
SPRING_DATASOURCE_PASSWORD=CHANGE_ME_DB_PASSWORD
SPRING_DATASOURCE_URL=jdbc:postgresql://CHANGE_ME_DB_HOST:5432/esbdb
SPRING_DATASOURCE_USERNAME=CHANGE_ME_DB_USERNAME
SPRING_REDIS_HOST=redis-master.redis.svc.cluster.local
SPRING_REDIS_PORT=6379
EOF

kubectl -n api-manager create configmap api-manager-common \
  --from-env-file=./api-manager-common.env
```

**middleware** namespace (`api-manager-common`):

```bash
# Write middleware-common.env with KEY=value lines, then:
kubectl -n middleware create configmap api-manager-common \
  --from-env-file=./middleware-common.env
```

Sample `middleware-common.env` keys (replace `CHANGE_ME_*`). For multiline
`SYSAID_CONFIG`, prefer Option A (`kubectl apply` of the YAML) or create the
key with `kubectl create configmap ... --from-file=SYSAID_CONFIG=./sysaid-config.json`
and merge; a single-line JSON value also works in an env file:

```bash
cat > middleware-common.env <<'EOF'
DB_PASSWORD=CHANGE_ME_DB_PASSWORD
DB_URI=postgresql://CHANGE_ME_DB_HOST:5432/esbdb
DB_USERNAME=CHANGE_ME_DB_USERNAME
ENCODER_ALG_KEY=CHANGE_ME_ENCODER_ALG_KEY
ENTRA_CLIENT_ID=CHANGE_ME_ENTRA_CLIENT_ID
ENTRA_TENANT_ID=CHANGE_ME_ENTRA_TENANT_ID
ESB_BASE_URL=http://api-gateway-service.api-manager.svc.cluster.local:8080
ESB_CHANNELS_BASE_URL=http://api-gateway-service.api-manager.svc.cluster.local:8080/manager/esb-channel-manager
ESB_LOGS_BASE_URL=http://api-gateway-service.api-manager.svc.cluster.local:8080/esb
ESB_SERVICES_BASE_URL=http://api-gateway-service.api-manager.svc.cluster.local:8080/manager/esb-service-manager
EXTERNAL_PORTAL_BASE_URL=https://staging-external-portal.cic.co.ke
GITHUB_CLIENT_ID=CHANGE_ME_GITHUB_CLIENT_ID
GITHUB_CLIENT_SECRET=CHANGE_ME_GITHUB_CLIENT_SECRET
GITHUB_REDIRECT_URI=
INTERNAL_PORTAL_BASE_URL=https://staging-internal-portal.cic.co.ke
JWT_EXPIRATION=3600000
JWT_SECRET=CHANGE_ME_JWT_SECRET
KAFKA_BOOTSTRAP_SERVERS=my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
KAFKA_BROKER=my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
MAIL_INTERNAL_DOMAINS=ke.cicinsurancegroup.com,cic.co.ke
PORTALS_BACKEND_BASEURL=https://staging-portals-api.cic.co.ke
PORTALS_EUREKA_SERVER=http://eureka-server.api-manager.svc.cluster.local:8761/eureka/
QUOTA_ALERT_RECIPIENTS=CHANGE_ME_QUOTA_ALERT_RECIPIENTS
QUOTA_ALERTS_ENABLED=true
REDIS_MASTER=mymaster
REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD
REDIS_PORT=6379
REDIS_PRIMARY_NODE=redis-s-hl.redis.svc.cluster.local:26379
REDIS_SECONDARY_NODES=redis-s-hl.redis.svc.cluster.local:26379
SENDGRID_API_KEY=CHANGE_ME_SENDGRID_API_KEY
SHAREPOINT_BLOGS_DRIVE_ID=CHANGE_ME_SHAREPOINT_BLOGS_DRIVE_ID
SHAREPOINT_BLOGS_SITE_ID=CHANGE_ME_SHAREPOINT_BLOGS_SITE_ID
SHAREPOINT_CLIENT_ID=CHANGE_ME_SHAREPOINT_CLIENT_ID
SHAREPOINT_CLIENT_SECRET=CHANGE_ME_SHAREPOINT_CLIENT_SECRET
SHAREPOINT_DRIVE_ID=CHANGE_ME_SHAREPOINT_DRIVE_ID
SHAREPOINT_SITE_ID=CHANGE_ME_SHAREPOINT_SITE_ID
SHAREPOINT_TENANT_ID=CHANGE_ME_SHAREPOINT_TENANT_ID
SPRING_DATASOURCE_PASSWORD=CHANGE_ME_DB_PASSWORD
SPRING_DATASOURCE_URL=jdbc:postgresql://CHANGE_ME_DB_HOST:5432/esbdb
SPRING_DATASOURCE_USERNAME=CHANGE_ME_DB_USERNAME
SPRING_PROFILES_ACTIVE=staging
SYSAID_ACCOUNT_ID=CHANGE_ME_SYSAID_ACCOUNT_ID
SYSAID_CLIENT_ID=CHANGE_ME_SYSAID_CLIENT_ID
SYSAID_CLIENT_SECRET=CHANGE_ME_SYSAID_CLIENT_SECRET
SYSAID_CONFIG={"type":"request","templateId":26907}
EOF

kubectl -n middleware create configmap api-manager-common \
  --from-env-file=./middleware-common.env
```

| ConfigMap | Namespace | Consumed by |
|---|---|---|
| `api-manager-common` | `api-manager` | API Manager Deployments (`envFrom`) |
| `api-manager-common` | `middleware` | Middleware Deployments (`envFrom`) |
| `api-manager-common` | `external-portal` | External Portal Deployments (`envFrom`) |


### 6. Apply and synchronize the root Applications

Commit and push the Application directory and root changes before synchronizing;
Argo CD reads them from the remote repository rather than the local checkout.

```bash
kubectl apply -f staging/bootstrap/root.yaml
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

- `cic-staging-platform-root` → `staging/applications/platform`
- `cic-staging-monitoring-root` → `staging/applications/monitoring`
- `cic-staging-workloads-root` → `staging/applications/workloads`

Sync the roots in that order (NOTE: sync order is only required if deploying a totally fresh deployment). They create or update child Application resources but
do not sync those children automatically.

`staging/projects.yaml` is outside the roots' source paths and is therefore applied
manually whenever its configuration changes.

### 7. Synchronize child Applications

Confirm the [Required Secrets](#4-required-secrets) and
[Required ConfigMaps](#5-required-configmaps) from steps 4–5 exist before syncing
the Applications that depend on them. Review every diff before synchronization. A
suitable dependency order is:

```bash
argocd app diff metallb-staging
argocd app sync metallb-staging
argocd app wait metallb-staging --health
argocd app diff metallb-config-staging
argocd app sync metallb-config-staging

argocd app diff ingress-nginx-staging
argocd app sync ingress-nginx-staging
# initialize with metrics enabled=off, then enable later after installing prometheus

argocd app diff longhorn-staging
argocd app sync longhorn-staging

# Ingress NGINX ships a ServiceMonitor. Sync kube-prometheus-stack first so
# the monitoring.coreos.com CRDs exist, then sync ingress-nginx.
argocd app diff kube-prometheus-stack-staging
argocd app sync kube-prometheus-stack-staging

argocd app diff ingress-nginx-staging
argocd app sync ingress-nginx-staging

argocd app diff strimzi-kafka-operator-staging
argocd app sync strimzi-kafka-operator-staging
argocd app wait strimzi-kafka-operator-staging --health
argocd app diff kafka-cluster-staging
argocd app sync kafka-cluster-staging

# Operator and CRDs first, then the RedisReplication resource
argocd app sync redis-operator-staging
argocd app wait redis-operator-staging --health
argocd app diff redis-staging
argocd app sync redis-staging
# MinIO requires four Local PVs on dedicated SSDs (/data/minio) before sync.
argocd app diff minio-staging
argocd app sync minio-staging

argocd app diff loki-staging
argocd app sync loki-staging
argocd app diff alloy-staging
argocd app sync alloy-staging

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
Strimzi operator and its CRDs. The ingress-nginx ServiceMonitor requires the
kube-prometheus-stack CRDs, so that Application is synced before
`ingress-nginx-staging`.

YAML file order does not determine synchronization order. The roots and children use
manual sync for brownfield adoption, so the commands above provide the current
workload ordering. Argo CD sync waves only order resources within the same
Application. The three separate roots do not enforce ordering across one another.

### 8. Verify

```bash
argocd app list
kubectl get applications -n argocd
kubectl get pods -A
kubectl get pvc -A
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp
```

Before syncing against an existing cluster, compare Helm versions, release names,
values, selectors, and PVC templates with the live resources. Confirm the
[Required Secrets](#4-required-secrets) and
[Required ConfigMaps](#5-required-configmaps) exist, verify the MetalLB address
pool, and replace the sample images with approved CIC Harbor image digests.

## Production

Production mirrors the staging layout:

1. Apply `production/projects.yaml` (`bootstrap-production`, `platform-production`,
   `monitoring-production`, `workloads-production`).
2. Create the same [Required Secrets](#4-required-secrets) in the production cluster
   (same names, namespaces, and keys; production credential values).
3. Create the production [Required ConfigMaps](#5-required-configmaps)
   (`api-manager-common` in `api-manager` and `middleware`) with production values.
4. Apply and sync the three roots from `production/bootstrap/root.yaml` in order:
   `cic-production-platform-root`, `cic-production-monitoring-root`,
   `cic-production-workloads-root`.
5. Review diffs and sync child Applications with the same dependency order as
   staging, using the `-production` Application names.

# Uninstall an existing cluster

1. Remove workloads (from master-01)
kubectl delete application --all -n argocd      # stop Argo CD from recreating things
kubectl delete ns api-manager middleware external-portal kafka redis minio monitoring argocd
kubectl get pvc -A                              # confirm all PVCs are gone

If a namespace sticks in Terminating, find the blocker and clear its finalizers:
kubectl get ns <ns> -o json | jq '.spec.finalizers=[]' | kubectl replace --raw "/api/v1/namespaces/<ns>/finalize" -f -

2. Remove storage and networking
helm list -A
helm uninstall longhorn -n longhorn-system
kubectl delete ns longhorn-system metallb-system ingress-nginx

Longhorn needs its uninstall job to finish before you delete the namespace. If you skip Helm, set the deleting-confirmation flag first:
kubectl -n longhorn-system patch settings.longhorn.io deleting-confirmation-flag --type=merge -p '{"value":"true"}'

3. Drain and remove workers (master-04, 05, 06)
kubectl drain api-manager-master-04 --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node api-manager-master-04

Repeat for 05 and 06, then on each of those nodes:
## Worker nodes cleanup
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/kubelet ~/.kube
sudo systemctl stop kubelet
sudo reboot    # clears iptables and the flannel/cni0 interfaces

4. Control-plane nodes (master-03, 02, then 01 last)
kubectl drain api-manager-master-03 --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node api-manager-master-03
Repeat for 02 and 01, then on each of those nodes:
## Control planes cleanup
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/kubelet ~/.kube
sudo systemctl stop kubelet
sudo rm -rf /var/lib/etcd /etc/kubernetes
sudo dnf remove -y kubeadm kubelet kubectl
sudo reboot


# Prepare for longhorn - ensure system modules are loaded

1. clear /data/longhorn on all worker nodes
2. dm_crypt not loaded (all three nodes)
Run on 04, 05 and 06:

bash
sudo modprobe dm_crypt
echo dm_crypt | sudo tee /etc/modules-load.d/longhorn.conf
lsmod | grep dm_crypt

3. 
findmnt /data
ls -la /data/longhorn

4. If you can’t uninstall, remove the disk from each node’s spec (Longhorn UI → Node → Edit → delete disk), wipe the directory as above, then re-add it.

Caution: the wipe destroys any replica data in that path. Only do it if no volumes there matter.

Fix dm_crypt first, then the disks. Once both are done, re-check with kubectl get nodes.longhorn.io -n longhorn-system.

5. Restart longhorn managers
kubectl -n longhorn-system rollout restart daemonset/longhorn-manager

6. Verify
kubectl -n longhorn-system describe nodes.longhorn.io | grep -B4 "Type: *KernelModulesLoaded"