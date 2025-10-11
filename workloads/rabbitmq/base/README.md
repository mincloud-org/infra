# RabbitMQ Cluster Operator

这个目录包含了基于 RabbitMQ Cluster Operator 的 RabbitMQ 高可用部署配置。

## 架构概述

- **RabbitMQ Cluster Operator**: 管理 RabbitMQ 集群生命周期的 Kubernetes Operator
- **RabbitMQ 集群**: 3节点高可用 RabbitMQ 集群 (RabbitMQ 4.0)
- **持久化存储**: 使用 PVC 确保数据持久化
- **服务发现**: 通过 Kubernetes DNS 进行节点发现
- **监控**: 集成 Prometheus 监控

## 组件说明

### 核心资源

- `rabbitmq-operator.yaml`: RabbitMQ Cluster Operator 部署配置
- `rabbitmq-cluster.yaml`: RabbitmqCluster 自定义资源定义
- `rabbitmq-secret.yaml`: 认证凭据和 TLS 证书
- `services.yaml`: 服务暴露和网络策略配置
- `rabbitmq-pdb.yaml`: Pod 中断预算配置
- `serviceMonitor.yaml`: Prometheus 监控配置

### 功能特性

#### 高可用性
- 3节点 RabbitMQ 集群
- 自动故障转移
- Quorum Queues (RabbitMQ 4.0 默认)
- Pod 中断预算保护

#### 安全性
- TLS 加密通信
- 自定义用户凭据
- 网络策略隔离
- 非 root 用户运行

#### 监控
- Prometheus 指标暴露
- 健康检查探针
- 自定义指标收集

#### 存储
- 持久化卷声明
- 可配置存储类
- 数据持久化保证

## 默认配置

### 资源配置
```yaml
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 1000m
    memory: 2Gi
```

### 存储配置
```yaml
persistence:
  storageClassName: standard
  storage: 10Gi
```

### 端口配置
- **5672**: AMQP 协议端口
- **5671**: AMQP over TLS 端口  
- **15672**: Management UI 端口
- **15692**: Prometheus 指标端口
- **25672**: 集群通信端口

## 默认凭据

⚠️ **安全警告**: 以下是默认凭据，生产环境请务必修改：

- **用户名**: admin
- **密码**: admin123
- **Erlang Cookie**: rabbitmq-erlang-cookie

## 部署要求

### 先决条件
- Kubernetes 1.19+
- 存储类支持动态卷供应
- 足够的计算资源 (至少 3 CPU cores, 6Gi 内存)

### 可选组件
- Prometheus Operator (用于监控)
- cert-manager (用于 TLS 证书管理)
- nginx-ingress (用于外部访问)

## 访问方式

### 内部访问
- AMQP: `rabbitmq-amqp.{namespace}.svc.cluster.local:5672`
- Management UI: `rabbitmq-management.{namespace}.svc.cluster.local:15672`

### 外部访问 (通过 Ingress)
- Management UI: `https://rabbitmq.example.com` (需要配置域名)

## 生产环境建议

1. **修改默认凭据**: 更新 `rabbitmq-secret.yaml` 中的密码
2. **配置真实域名**: 修改 `services.yaml` 中的 Ingress 配置
3. **调整资源限制**: 根据实际负载调整 CPU 和内存配置
4. **配置备份策略**: 设置 RabbitMQ 数据备份
5. **启用 TLS**: 配置有效的 TLS 证书
6. **网络安全**: 根据需要调整网络策略

## 扩展配置

### 增加集群节点
```yaml
spec:
  replicas: 5  # 增加到5个节点
```

### 调整存储
```yaml
persistence:
  storageClassName: ssd
  storage: 50Gi
```

### 自定义 RabbitMQ 配置
在 `rabbitmq-cluster.yaml` 的 `additionalConfig` 部分添加配置项。

## 故障排除

### 常见问题

1. **Pod 启动失败**: 检查资源限制和存储配置
2. **集群节点无法加入**: 验证 Erlang Cookie 配置
3. **Management UI 无法访问**: 检查 Ingress 和服务配置
4. **数据丢失**: 确认 PVC 配置正确

### 有用的命令

```bash
# 查看 RabbitmqCluster 状态
kubectl get rabbitmqclusters

# 查看 Pod 日志
kubectl logs -l app.kubernetes.io/name=rabbitmq

# 查看集群状态
kubectl exec -it rabbitmq-server-0 -- rabbitmqctl cluster_status

# 检查队列状态
kubectl exec -it rabbitmq-server-0 -- rabbitmqctl list_queues
```