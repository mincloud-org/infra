# RabbitMQ via Cluster Operator

This Kustomize base installs the official RabbitMQ Cluster Operator and creates a `RabbitmqCluster` named `rabbitmq`.

Notes:
- The operator and CRDs are installed from the official release URL. Pin to a specific version for deterministic rollouts (replace `latest` with a tag, e.g. `v2.9.0`).
- The CR `rabbitmq-cluster.yaml` is namespaced and will be created in the overlay namespace (e.g., `staging`).
- Override `persistence.storageClassName` as needed. Empty string uses the cluster default StorageClass.

Expose Management UI:
- The default image includes the management plugin at `:15672`.
- Create a Service/Ingress against the `rabbitmq` Service if external access is required.

Metrics:
- Annotations are included for Prometheus scraping on the service port `9419` (requires `prometheus_rabbitmq_exporter`).

Pin operator version:
```
resources:
  - https://github.com/rabbitmq/cluster-operator/releases/download/v2.9.0/cluster-operator.yml
  - rabbitmq-cluster.yaml
```

Argo CD:
- See `../staging/application.yaml` for an example Application manifest.