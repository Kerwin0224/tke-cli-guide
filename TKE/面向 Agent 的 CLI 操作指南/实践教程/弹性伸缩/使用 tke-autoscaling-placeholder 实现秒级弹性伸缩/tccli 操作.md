# 使用 tke-autoscaling-placeholder 实现秒级弹性伸缩（tccli）

> 对照官方：[使用 tke-autoscaling-placeholder 实现秒级弹性伸缩](https://cloud.tencent.com/document/product/457/50073) · page_id `50073`

## 概述

通过部署低优先级占位 Pod（placeholder）预占资源，当业务 Pod 需要扩容时，占位 Pod 被驱逐并立即为业务 Pod 腾出资源，实现秒级扩容（无需等待节点创建）。

## 前置条件

- [环境准备](../../../环境准备.md)
- Cluster Autoscaler 已启用

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 PriorityClass | `kubectl apply -f priority-class.yaml` | 是 |
| 部署占位 Pod | `kubectl apply -f placeholder.yaml` | 是 |
| 查看占位 Pod | `kubectl get pods -l app=placeholder` | 是 |

## 操作步骤

### 1. 创建优先级

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: placeholder-priority
value: -10
globalDefault: false
```

### 2. 部署占位 Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: placeholder
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: placeholder-priority
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
```

```bash
kubectl apply -f placeholder.yaml
```

```text
# command executed successfully
```

当业务 Pod 需要资源时，低优先级的占位 Pod 被自动驱逐，业务 Pod 秒级调度成功。

## 验证

```bash
kubectl get pods -l app=placeholder
kubectl describe node <node> | grep Allocated
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| 占位 Pod 未被驱逐、业务 Pod Pending | `tccli tke DescribeClusterAsGroupOption --region ap-guangzhou --ClusterId <ClusterId>` | Cluster Autoscaler 未启用，无法触发节点扩容 | 启用 Cluster Autoscaler；确认占位 Pod 优先级低于业务 Pod |
| 扩容仍需等节点创建 | `kubectl get pods -l app=placeholder` | 占位 Pod 数量不足或未占满节点资源 | 增加 placeholder 副本数，使占位资源 >= 业务 Pod request |

## 清理

```bash
kubectl delete deploy placeholder
kubectl delete priorityclass placeholder-priority
```

## 下一步

- [集群弹性伸缩实践](../集群弹性伸缩实践/tccli%20操作.md)
- [HPA 弹性伸缩](../在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)

## 控制台替代

无控制台界面。
