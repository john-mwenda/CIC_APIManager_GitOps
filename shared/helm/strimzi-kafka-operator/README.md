# strimzi-kafka-operator

Values for the pinned upstream `strimzi-kafka-operator` Helm chart.

| Item | Pin |
| --- | --- |
| Chart / operator | **1.2.0** (`https://strimzi.io/charts/`) |
| Supported Kafka (operator default set) | 4.2.0, 4.2.1, 4.3.0, **4.3.1** |
| Cluster CRs | `staging/manifests/kafka-cluster` and `production/manifests/kafka-cluster` |

Argo CD Applications set `targetRevision: 1.2.0` and merge:

- `shared/helm/strimzi-kafka-operator/values-common.yaml`
- `staging/helm/strimzi-kafka-operator/values.yaml` (`watchNamespaces: [kafka]`) or
- `production/helm/strimzi-kafka-operator/values.yaml` (`watchNamespaces: [kafka]`)

Namespaces must be lowercase DNS labels (`kafka` / `kafka`).

This chart installs the **Cluster Operator only**. Kafka version is set on the `Kafka` CR (`spec.kafka.version`), currently **4.3.1**.

Before adopting an existing release, compare these files with `helm get values --all <release> -n <namespace>`.
Do not change StatefulSet names, release names, storage classes, or PVC templates
without a backup and an explicit migration.

**Upgrade note:** Moving from Strimzi 0.45.x / Kafka 3.9.x to 1.2.0 / Kafka 4.3.x is a major upgrade.
Sync the operator app first, then the kafka-cluster app, following the Strimzi upgrade docs.
