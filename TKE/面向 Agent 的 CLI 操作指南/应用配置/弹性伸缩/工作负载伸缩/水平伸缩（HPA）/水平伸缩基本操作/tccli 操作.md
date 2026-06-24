# 水平伸缩基本操作（tccli）

> 对照官方：[水平伸缩基本操作](https://cloud.tencent.com/document/product/457/37384) · page_id `37384`

## 概述

实例（Pod）水平自动扩缩容（Horizontal Pod Autoscaler，HPA）根据目标实例 CPU 利用率的平均值等指标自动扩展、缩减服务的 Pod 数量。HPA 后台组件每隔 15 秒向腾讯云可观测平台拉取容器和 Pod 监控指标，根据当前指标数据、当前副本数和目标值计算期望副本数，触发 Deployment 进行 Pod 副本数量调整。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 工作负载已设置 `resources.requests`（HPA 基于 request 计算利用率）
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 创建 Deployment + HPA（YAML） | `kubectl apply -f <yaml>` | 是 |
| 为已有 Deployment 创建 HPA | `kubectl autoscale deployment <name>` | 否（重复创建报错） |
| 创建单独的 HPA 对象 | `kubectl apply -f hpa.yaml` | 是 |
| 更新 HPA 配置 | `kubectl edit hpa <name>` | 否（冲突） |
| 查看 HPA 状态 | `kubectl get hpa -A` | 是 |
| 调整 min/max 副本数 | `kubectl patch hpa <name>` | 是 |
| 删除 HPA | `kubectl delete hpa <name>` | 是 |

## 操作步骤

### 步骤 1：工作原理

HPA 后台组件每隔 15 秒拉取监控指标，根据当前指标数据、当前副本数和指标目标值计算期望副本数。以 CPU 利用率为例：当前有 2 个实例，平均 CPU 利用率 90%，目标 CPU 为 60%，则调整实例数量为 90% × 2 / 60% = 3 个。

> **注意**：设置多个弹性伸缩指标时，HPA 依据各指标分别计算目标副本数，取最大值进行扩缩容操作。

### 步骤 2：注意事项

- CPU 利用率（占 Request）时，必须为容器设置 CPU Request
- 策略指标目标设置合理，例如设置 70% 给容器和应用，预留 30% 余量
- 保持 Pod 和 Node 健康（避免 Pod 频繁重建）
- 保证用户请求的负载均衡稳定运行
- HPA 计算目标副本数时有 10% 波动因子，波动范围内不调整副本
- 如果 Deployment `spec.replicas` 为 0，HPA 不起作用
- 单个 Deployment 同时绑定多个 HPA 会同时生效，造成副本重复扩缩

### 步骤 3：创建 HPA

#### 方式一：通过 Deployment 创建时一并设置

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment-name>
  namespace: <namespace>
spec:
  replicas: 2
  selector:
    matchLabels:
      app: <app-label>
  template:
    metadata:
      labels:
        app: <app-label>
    spec:
      containers:
      - name: <container-name>
        image: <image>
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <hpa-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

```bash
kubectl apply -f deployment-with-hpa.yaml
# expected: deployment.apps/<deployment-name> created
# expected: horizontalpodautoscaler.autoscaling/<hpa-name> created
```

#### 方式二：为已有 Deployment 创建 HPA

命令行快速创建（基于 CPU）：

```bash
kubectl autoscale deployment <deployment-name> \
  --min=1 --max=10 --cpu-percent=60 \
  -n <namespace>
# expected: horizontalpodautoscaler.autoscaling/<deployment-name> autoscaled
```

或使用 YAML：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <hpa-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

```bash
kubectl apply -f hpa.yaml
# expected: horizontalpodautoscaler.autoscaling/<hpa-name> created
```

### 步骤 4：查看 HPA 状态

```bash
kubectl get hpa -A
# expected: 显示 TARGETS、MINPODS、MAXPODS、REPLICAS
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe hpa <hpa-name> -n <namespace>
# expected: 显示扩缩容事件和指标详情
```

```text
NAME  STATUS  AGE
...
```

### 步骤 5：更新自动扩缩容规则

```bash
kubectl edit hpa <hpa-name> -n <namespace>
```

或使用 patch 修改 min/max：

```bash
kubectl patch hpa <hpa-name> -n <namespace> \
  -p '{"spec":{"minReplicas":2,"maxReplicas":20}}'
# expected: horizontalpodautoscaler.autoscaling/<hpa-name> patched
```

### 步骤 6：指标类型

| 指标 | 说明 |
|------|------|
| CPU 利用率（占 Request） | 根据容器设置的 CPU Request 计算利用率百分比 |
| CPU 使用量 | 直接指定 CPU 使用量阈值 |
| 内存利用率（占 Request） | 根据容器设置的内存 Request 计算利用率百分比 |
| 内存使用量 | 直接指定内存使用量阈值 |

详细指标说明参见 [水平自动伸缩指标说明](../水平自动伸缩指标说明/tccli%20操作.md)。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面（kubectl）

```bash
kubectl get hpa <hpa-name> -n <namespace> -o yaml
# expected: 显示 HPA 完整配置和状态

kubectl get pods -n <namespace> -l app=<app-label>
# expected: Pod 数量在 minReplicas 和 maxReplicas 之间

kubectl top pods -n <namespace>
# expected: 显示实际 CPU/内存使用
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete hpa <hpa-name> -n <namespace>
# expected: horizontalpodautoscaler.autoscaling "<hpa-name>" deleted

kubectl delete deployment <deployment-name> -n <namespace>
# expected: deployment.apps "<deployment-name>" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| HPA 创建报错 `already exists` | `kubectl get hpa` 查看同名资源 | 重复创建同名 HPA | 删除已有 HPA 或换名 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| HPA 不生效 | `kubectl describe hpa <name>` 检查 | Deployment `spec.replicas` 为 0 或 metrics-server 未运行 | 确保 replicas > 0，检查 metrics-server 状态 |
| TARGETS 显示 `<unknown>` | `kubectl get hpa` 查看 TARGETS 列 | metrics-server 未部署或指标收集延迟 | 确认 metrics-server 运行，检查容器已设置 CPU Request |
| 副本数未按预期变化 | `kubectl describe hpa <name>` 查看 Events | HPA 有 10% 波动因子，波动范围内不调整 | 属正常行为，检查指标和阈值设置 |
| HPA 频繁扩缩（抖动） | `kubectl get hpa -w` 观察变化 | 指标阈值设置不当 | 调整 `averageUtilization`，增加缩容稳定窗口 |

## 下一步

- [水平自动伸缩指标说明](../水平自动伸缩指标说明/tccli%20操作.md) — HPA 指标详解
- [定时水平伸缩（HPC）](../../定时水平伸缩（HPC）/tccli%20操作.md) — 定时扩缩容
- [预测水平伸缩（EHPA）](../../预测水平伸缩（EHPA）/tccli%20操作.md) — 智能预测伸缩

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Deployment → 新建，设置实例数量为自动调节；或自动伸缩 → HorizontalPodAutoscaler → 新建。
