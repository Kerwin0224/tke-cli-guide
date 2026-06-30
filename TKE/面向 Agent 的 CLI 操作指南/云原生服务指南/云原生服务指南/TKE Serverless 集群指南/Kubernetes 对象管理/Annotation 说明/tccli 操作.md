# Annotation 说明（tccli）

> 对照官方：[Annotation 说明](https://cloud.tencent.com/document/product/457/60413) · page_id `60413`

## 概述

TKE Serverless 集群通过 Kubernetes Annotation 为超级节点提供丰富的定制化能力。在 Pod 模板的 `metadata.annotations` 中指定特定的 Annotation，可控制 Pod 的资源规格、网络配置、安全组等属性。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下 `kubectl` 命令和 YAML 示例是正确的 CLI 操作方式，实际执行需解决 CAM 权限或使用 Cloud Shell 连接。

## 前置条件

- [环境准备](../../环境准备.md)
- 已创建 Serverless 集群且状态为 `Running`，参见[创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)
- 已[连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md)，`kubectl` 可用

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|------|
| 通过 Annotation 指定 CPU | 在 Pod/Deployment YAML 中设置 `eks.tke.cloud.tencent.com/cpu` | 是（apply 幂等） |
| 通过 Annotation 指定内存 | 在 Pod/Deployment YAML 中设置 `eks.tke.cloud.tencent.com/mem` | 是 |
| 通过 Annotation 指定安全组 | 在 Pod/Deployment YAML 中设置 `eks.tke.cloud.tencent.com/security-group-id` | 是 |
| 通过 Annotation 指定子网 | 在 Pod/Deployment YAML 中设置 `eks.tke.cloud.tencent.com/subnet-id` | 是 |
| 查看 Pod Annotation | `kubectl describe pod <name>` 或 `kubectl get pod <name> -o json` | 是 |

## 操作步骤

### 常用 Annotation 列表

| Annotation | 作用 | 示例值 |
|-----------|------|--------|
| `eks.tke.cloud.tencent.com/cpu` | 指定 Pod 的 CPU 规格（核） | `"1"`, `"2"`, `"4"`, `"8"`, `"16"` |
| `eks.tke.cloud.tencent.com/mem` | 指定 Pod 的内存规格 | `"1Gi"`, `"2Gi"`, `"4Gi"`, `"8Gi"`, `"16Gi"`, `"32Gi"` |
| `eks.tke.cloud.tencent.com/security-group-id` | 指定 Pod 绑定的安全组 ID | `"sg-example"` |
| `eks.tke.cloud.tencent.com/subnet-id` | 指定 Pod 所在的子网 ID | `"subnet-example"` |
| `eks.tke.cloud.tencent.com/gpu-type` | 指定 GPU 类型 | `"V100"`, `"T4"`, `"A10"` |
| `eks.tke.cloud.tencent.com/gpu-count` | 指定 GPU 数量 | `"1"`, `"2"`, `"4"` |
| `eks.tke.cloud.tencent.com/root-cbs-size` | 指定系统盘大小（GiB） | `"50"`, `"100"` |
| `eks.tke.cloud.tencent.com/eip-attributes` | 是否分配 EIP | JSON 格式字符串 |
| `eks.tke.cloud.tencent.com/role-name` | 绑定 CAM 角色名称 | 角色名 |

### 1. 指定 CPU 和内存规格

Pod 的资源规格通过 `resources` 字段和 Annotation 共同指定。**Annotation 定义的规格决定计费**，`resources.requests` 用于 Kubernetes 调度。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-spec
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
```

```bash
kubectl apply -f pod-spec.yaml
```

### 2. 指定安全组

每个 Pod 可单独指定安全组，未指定时使用集群默认安全组。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-sg
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
    eks.tke.cloud.tencent.com/security-group-id: "sg-example01"
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl apply -f pod-security-group.yaml
```

> **注意：** 每个 Pod 等同于一台 CVM 实例，若工作负载有大量副本，需确保安全组配额足够容纳所有 Pod。参见[购买限制](../../购买%20TKE%20Serverless%20集群/购买限制/tccli%20操作.md)。

### 3. 指定子网

Pod 默认调度到集群配置的子网，可通过 Annotation 指定特定子网。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-subnet
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
    eks.tke.cloud.tencent.com/subnet-id: "subnet-example-b"
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl apply -f pod-subnet.yaml
```

> **注意：** 指定的子网需在集群 VPC 内，且子网需有充足的可用 IP。

### 4. 在 Deployment 中使用 Annotation

将 Annotation 设置在 Deployment 的 Pod 模板中，所有副本自动继承。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: annotated-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: annotated-app
  template:
    metadata:
      labels:
        app: annotated-app
      annotations:
        eks.tke.cloud.tencent.com/cpu: "2"
        eks.tke.cloud.tencent.com/mem: "4Gi"
        eks.tke.cloud.tencent.com/security-group-id: "sg-example01"
        eks.tke.cloud.tencent.com/subnet-id: "subnet-example-a"
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

```bash
kubectl apply -f deployment-annotated.yaml
```

```text
# command executed successfully
```

### 5. 查看 Annotation

```bash
# 查看 Pod 的 Annotation
kubectl describe pod <pod-name> --namespace=default

# 以 JSON 格式查看（含 Annotation 详细信息）
kubectl get pod <pod-name> --namespace=default -o json | jq '.metadata.annotations'
```

示例输出（`kubectl describe pod` 中的 Annotations 部分）：

```text
Annotations:
  eks.tke.cloud.tencent.com/cpu: 2
  eks.tke.cloud.tencent.com/mem: 4Gi
  eks.tke.cloud.tencent.com/security-group-id: sg-example01
  eks.tke.cloud.tencent.com/subnet-id: subnet-example-a
```

### 6. 通过 tccli 查看容器实例的资源配置

Annotation 最终影响底层容器实例的资源分配，可通过 tccli 验证：

```bash
tccli tke DescribeEKSContainerInstances \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 3,
    "EksCiSet": [
        {
            "EksCiId": "eksci-example01",
            "EksCiName": "annotated-deployment-xxx-abc1",
            "Status": "Running",
            "Cpu": 2.0,
            "Memory": 4.0,
            "SecurityGroupIds": ["sg-example01"],
            "VpcId": "vpc-example",
            "SubnetId": "subnet-example-a",
            "CreationTime": "2024-01-15T10:35:00Z"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| Pod Annotation 已设置 | `kubectl describe pod <pod-name> -n default` | Annotations 区域显示指定值 |
| 容器实例规格匹配 | `tke DescribeEKSContainerInstances --region ap-guangzhou` | `Cpu`/`Memory` 与 Annotation 一致 |
| 安全组已绑定 | 同上 | `SecurityGroupIds` 与 Annotation 一致 |

## 清理

```bash
kubectl delete pod <pod-name> --namespace=default
kubectl delete deployment <deployment-name> --namespace=default
```

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| Pod 创建后资源规格与 Annotation 不一致 | `resources.requests` 与 Annotation 值不匹配 | 确保两处值一致，Annotation 决定计费 |
| 安全组 Annotation 不生效 | 安全组 ID 格式错误或不存在 | 确认 `sg-` 前缀和安全组存在性 |
| 子网 Annotation 不生效 | 子网不在集群 VPC 内或 IP 不足 | 确认子网 ID 格式和可用 IP |
| GPU Annotation 不生效 | GPU 类型在当前地域不支持 | 查阅[地域和可用区](../../购买%20TKE%20Serverless%20集群/地域和可用区/tccli%20操作.md)确认 GPU 可用地域 |
| `PodAnnotations` 变化后 Pod 未重建 | Annotation 修改仅对新 Pod 生效 | 删除 Pod 让 ReplicaSet 重建，或执行 `kubectl rollout restart deployment <name>` |

## 下一步

- [工作负载管理](../工作负载管理/tccli%20操作.md) — 将带 Annotation 的 Pod 模板用于 Deployment、Job 等工作负载
- [注意事项](../../TKE%20Serverless%20集群管理/注意事项/tccli%20操作.md) — 了解 Serverless 集群特有的使用限制

## 控制台替代

[容器服务控制台 → 集群 → 目标集群 → 工作负载 → 新建](https://console.cloud.tencent.com/tke2/cluster) — 创建页面「高级设置」中可配置资源规格和安全组，控制台会自动生成对应的 Annotation。
