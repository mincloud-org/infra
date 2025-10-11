# PostgreSQL High Availability Cluster Deployment

This is a Kubernetes-based PostgreSQL primary-replica high availability deployment solution with auto-scaling support.

## Architecture Features

- **1 Primary + 2 Replicas**: 1 primary for writes, 2 replicas for reads
- **Streaming Replication**: Uses PostgreSQL native streaming replication for primary-replica synchronization
- **Auto-scaling**: Automatically scales replicas based on CPU and memory utilization (2-5 replicas)
- **Service Separation**: Provides separate read and write service endpoints
- **Health Checks**: Includes liveness and readiness probes
- **Persistent Storage**: Uses StatefulSet and PVC for data persistence

## Directory Structure

```
postgres/
├── base/                           # Base configuration
│   ├── kustomization.yaml          # Kustomize configuration
│   ├── postgres-config.yaml        # PostgreSQL config and scripts
│   ├── postgres-secret.yaml        # Password configuration (needs modification)
│   ├── postgres-primary-statefulset.yaml    # Primary database
│   ├── postgres-replica-statefulset.yaml    # Replica databases
│   ├── postgres-replica-hpa.yaml   # HPA auto-scaling
│   └── services.yaml               # Service definitions
└── staging/                        # Staging environment
    ├── kustomization.yaml          # Environment-specific configuration
    ├── application.yaml            # ArgoCD application definition
    ├── postgres-primary-patch.yaml # Primary resource adjustments
    ├── postgres-replica-patch.yaml # Replica resource adjustments
    └── postgres-secret-patch.yaml  # Environment passwords
```

## Component Description

### 1. Primary Database
- **StatefulSet**: `postgres-primary`
- **Replicas**: 1
- **Function**: Handles all write operations, serves as replication source
- **Service**: `postgres-write` (write endpoint)

### 2. Replica Databases
- **StatefulSet**: `postgres-replica`
- **Replicas**: 2-5 (auto-scaling)
- **Function**: Handles read operations, streams data from primary
- **Service**: `postgres-read` (read endpoint)

### 3. Service Endpoints

| Service Name | Type | Purpose | Connection Example |
|--------------|------|---------|-------------------|
| `postgres-write` | ClusterIP | Write operations (connects to primary) | `postgres-write.staging.svc.cluster.local:5432` |
| `postgres-read` | ClusterIP | Read operations (load balanced across all replicas) | `postgres-read.staging.svc.cluster.local:5432` |
| `postgres-primary` | Headless | Direct access to primary pod | `postgres-primary-0.postgres-primary.staging.svc.cluster.local:5432` |
| `postgres-replica` | Headless | Direct access to replica pods | `postgres-replica-0.postgres-replica.staging.svc.cluster.local:5432` |

### 4. Auto-scaling (HPA)

Replicas will auto-scale based on the following metrics:
- **CPU Utilization**: Target 70%
- **Memory Utilization**: Target 80%
- **Minimum Replicas**: 2
- **Maximum Replicas**: 5
- **Scale-up Policy**: Fast scale-up (within 30s, up to 100% or 2 pods)
- **Scale-down Policy**: Smooth scale-down (5-min stabilization, up to 50% or 1 pod)

## Deployment Steps

### 1. Configure Secrets (Important!)

**⚠️ This is a public repository - DO NOT commit real passwords!**

Choose one of the following secret management approaches:

#### Option A: External Secrets (Recommended for Production)
Use External Secrets Operator with AWS Secrets Manager, HashiCorp Vault, or similar.
See [SECRETS-MANAGEMENT.md](../SECRETS-MANAGEMENT.md) for detailed setup.

#### Option B: Sealed Secrets (Recommended for GitOps)
Encrypt secrets using Sealed Secrets controller:
```bash
# See SECRETS-MANAGEMENT.md for complete instructions
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

#### Option C: Manual Secret Creation (Simple, for Testing)
Create secrets directly in cluster before deploying:
```bash
kubectl create secret generic postgres-secret \
  --from-literal=postgres-user=postgres \
  --from-literal=postgres-password='YOUR-STRONG-PASSWORD' \
  --from-literal=replication-password='YOUR-REPLICATION-PASSWORD' \
  --namespace=staging
```

Then remove `postgres-secret-patch.yaml` from staging kustomization.

📖 **See [SECRETS-MANAGEMENT.md](../SECRETS-MANAGEMENT.md) for complete security guide**

### 2. Deploy to Kubernetes

#### Option 1: Using ArgoCD (Recommended)

```bash
# Apply application.yaml to cluster
kubectl apply -f workloads/postgres/staging/application.yaml

# Check sync status
argocd app get postgres-staging
argocd app sync postgres-staging
```

#### Option 2: Using kubectl + kustomize

```bash
# Preview resources to be deployed
kubectl kustomize workloads/postgres/staging

# Deploy to staging namespace
kubectl apply -k workloads/postgres/staging

# Check deployment status
kubectl get all -n staging -l app.kubernetes.io/name=postgres
```

### 3. Verify Deployment

```bash
# Check pod status
kubectl get pods -n staging -l app.kubernetes.io/name=postgres

# Expected output:
# NAME                      READY   STATUS    RESTARTS   AGE
# staging-postgres-primary-0   1/1     Running   0          5m
# staging-postgres-replica-0   1/1     Running   0          5m
# staging-postgres-replica-1   1/1     Running   0          5m

# Check primary-replica replication status
kubectl exec -n staging staging-postgres-primary-0 -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"

# Check HPA status
kubectl get hpa -n staging staging-postgres-replica-hpa
```

## Usage

### Connecting to Database

#### Write Operations (connect to primary)
```bash
# Connect from inside a pod
psql -h postgres-write.staging.svc.cluster.local -U postgres -d postgres

# Port forward to local
kubectl port-forward -n staging svc/staging-postgres-write 5432:5432
psql -h localhost -U postgres -d postgres
```

#### Read Operations (connect to replicas)
```bash
# Connect from inside a pod
psql -h postgres-read.staging.svc.cluster.local -U postgres -d postgres

# Port forward to local
kubectl port-forward -n staging svc/staging-postgres-read 5433:5432
psql -h localhost -p 5433 -U postgres -d postgres
```

### Application Connection Examples

```python
# Python example - using psycopg2
import psycopg2

# Write connection
write_conn = psycopg2.connect(
    host="postgres-write.staging.svc.cluster.local",
    port=5432,
    user="postgres",
    password="your-password",
    database="postgres"
)

# Read connection
read_conn = psycopg2.connect(
    host="postgres-read.staging.svc.cluster.local",
    port=5432,
    user="postgres",
    password="your-password",
    database="postgres"
)
```

```javascript
// Node.js example - using pg
const { Pool } = require('pg');

// Write connection pool
const writePool = new Pool({
  host: 'postgres-write.staging.svc.cluster.local',
  port: 5432,
  user: 'postgres',
  password: 'your-password',
  database: 'postgres',
  max: 20
});

// Read connection pool
const readPool = new Pool({
  host: 'postgres-read.staging.svc.cluster.local',
  port: 5432,
  user: 'postgres',
  password: 'your-password',
  database: 'postgres',
  max: 20
});
```

## Operations

### Check Replication Lag

```bash
# Check replication status on primary
kubectl exec -n staging staging-postgres-primary-0 -- psql -U postgres -c "
SELECT 
    client_addr,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag
FROM pg_stat_replication;
"

# Check replication lag on replica
kubectl exec -n staging staging-postgres-replica-0 -- psql -U postgres -c "
SELECT 
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int AS lag_seconds;
"
```

### Manual Scaling of Replicas

```bash
# Temporarily adjust replica count (will be overridden by HPA)
kubectl scale statefulset -n staging staging-postgres-replica --replicas=3

# Permanently adjust replica range (modify HPA)
kubectl patch hpa -n staging staging-postgres-replica-hpa --type merge -p '
spec:
  minReplicas: 3
  maxReplicas: 8
'
```

### Backup and Restore

#### Backup
```bash
# Create logical backup from primary
kubectl exec -n staging staging-postgres-primary-0 -- pg_dumpall -U postgres > backup-$(date +%Y%m%d).sql

# Create physical backup from primary
kubectl exec -n staging staging-postgres-primary-0 -- pg_basebackup -D /tmp/backup -F tar -z -P -U replicator
```

#### Restore
```bash
# Restore logical backup
kubectl exec -i -n staging staging-postgres-primary-0 -- psql -U postgres < backup-20231010.sql
```

### Troubleshooting

```bash
# View primary logs
kubectl logs -n staging staging-postgres-primary-0 --tail=100

# View replica logs
kubectl logs -n staging staging-postgres-replica-0 --tail=100

# Enter container for troubleshooting
kubectl exec -it -n staging staging-postgres-primary-0 -- bash

# Check configuration files
kubectl exec -n staging staging-postgres-primary-0 -- cat /var/lib/postgresql/data/pgdata/postgresql.conf

# Check replication slots
kubectl exec -n staging staging-postgres-primary-0 -- psql -U postgres -c "SELECT * FROM pg_replication_slots;"
```

### Primary Failover

If the primary fails, you need to manually promote a replica to primary:

```bash
# 1. Stop the primary (if still running)
kubectl scale statefulset -n staging staging-postgres-primary --replicas=0

# 2. Select a replica to promote to primary
kubectl exec -n staging staging-postgres-replica-0 -- pg_ctl promote -D /var/lib/postgresql/data/pgdata

# 3. Update application connection configuration to point to new primary

# 4. Reconfigure other replicas to point to new primary
```

## Performance Optimization Recommendations

### 1. Storage Optimization
- Use SSD storage for better I/O performance
- Use separate StorageClass for primary and replicas
- Consider using local PV for best performance

### 2. Resource Adjustment
Adjust resource limits based on actual load:
```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "2000m"
  limits:
    memory: "8Gi"
    cpu: "4000m"
```

### 3. PostgreSQL Configuration Optimization
Edit configuration in `postgres-config.yaml`:
```ini
# Connection pooling
max_connections = 500

# Memory
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 16MB

# Concurrency
max_worker_processes = 8
max_parallel_workers = 8
```

### 4. Read-Write Splitting
- All write operations connect to `postgres-write`
- All read operations connect to `postgres-read`
- Implement read-write splitting logic at application layer

## Monitoring Metrics

Recommended key metrics to monitor:

1. **Replication Lag**: Time replica lags behind primary
2. **Connection Count**: Current active connections
3. **Query Performance**: Slow query statistics
4. **Resource Usage**: CPU, memory, disk I/O
5. **HPA Status**: Current replicas vs target replicas
6. **Disk Usage**: Data directory and WAL directory space

Can integrate with Prometheus and Grafana for monitoring.

## Security Recommendations

1. **Change Default Passwords**: Must modify all passwords before deployment
2. **Use Secret Management**: Passwords stored in Kubernetes Secrets
3. **Network Isolation**: Use NetworkPolicy to restrict access
4. **Regular Backups**: Establish regular backup strategy
5. **Encrypted Connections**: Enable SSL/TLS encrypted connections
6. **Limit Permissions**: Create dedicated users for applications, don't use postgres superuser

## Known Limitations

1. **Automatic Failover**: Currently does not support automatic failover, requires manual promotion of replica
2. **Synchronous Replication**: Uses asynchronous replication by default, potential data loss risk
3. **Cross-region**: Current configuration suitable for single-region deployment

## Future Improvements

- [ ] Integrate Patroni/Stolon for automatic failover
- [ ] Add connection pooling (PgBouncer)
- [ ] Implement synchronous replication option
- [ ] Add Prometheus metrics exporter
- [ ] Support multi-region deployment
- [ ] Automated backup to S3/MinIO

## References

- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [Kubernetes StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
