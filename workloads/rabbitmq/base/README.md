# RabbitMQ Cluster (StatefulSet)

## Overview

This provides a 3-node RabbitMQ cluster using the official rabbitmq:4.0-management image, clustered via the built-in k8s peer discovery plugin.

## Components
- StatefulSet `rabbitmq` with 3 replicas
- Headless service `rabbitmq-headless` for peer discovery
- ClusterIP service `rabbitmq` for clients (5672, 15672)
- ConfigMap `rabbitmq-config` with rabbitmq.conf and enabled_plugins
- Secret `rabbitmq-secret` for default user/pass and Erlang cookie
- PodDisruptionBudget to keep quorum during maintenance

## Connect
- AMQP: `rabbitmq:5672`
- HTTP management: `rabbitmq:15672`

## Notes
- Change credentials and Erlang cookie per environment using overlays
- PVC size defaults to 10Gi; patch per environment if needed
- Probes use `rabbitmq-diagnostics`; tune thresholds for your workloads
