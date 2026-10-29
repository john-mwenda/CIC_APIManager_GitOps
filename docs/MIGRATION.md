# Brownfield migration

Do not sync these samples directly over the running cluster.

1. Back up Longhorn volumes and export all live Helm values.
2. Match installed chart versions, release names, labels, selectors, and PVC templates.
3. Replace the public echo images with immutable CIC Harbor image digests.
4. Adopt one stateless workload first with pruning disabled.
5. Adopt operators before their custom resources: MetalLB before its pool, Strimzi
   before Kafka, and kube-prometheus-stack before monitoring custom resources.
6. Never commit generated Pods, ReplicaSets, EndpointSlices, operator-owned Secrets,
   Longhorn engines/replicas, or Helm release Secrets.
7. Enable automated sync and pruning only after repeated clean diffs.
