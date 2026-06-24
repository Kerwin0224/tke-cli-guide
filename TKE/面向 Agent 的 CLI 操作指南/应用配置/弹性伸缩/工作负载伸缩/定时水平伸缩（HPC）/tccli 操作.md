# 定时水平伸缩（HPC）（tccli）

> 对照官方：[定时水平伸缩（HPC）](https://cloud.tencent.com/document/product/457/112023) · page_id `112023`

## 概述

实例（Pod）定时水平扩缩容（Horizontal Pod CronScaler，HPC）可根据预设的时间点，定时扩容、缩减服务的 Pod 数量。当 HPA 和 HPC 同时作用于一个工作负载时可能出现规则冲突，为避免冲突，建议将 HPC 直接作用于 HPA。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 已安装 HPC 组件（参见 [HPC 说明](https://cloud.tencent.com/document/product/457/56753)）
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

| 控制台操作 | CLI / YAML 字段 | 幂等 |
|-----------|----------------|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 创建 HPC | `kubectl apply -f hpc.yaml` | 是 |
| 查看 HPC 列表 | `kubectl get hpc -A` | 是 |
| 查看 HPC 详情 | `kubectl describe hpc/<name>` | 是 |
| 修改 HPC 配置 | `kubectl edit hpc/<name>` | 否 |
| 删除 HPC | `kubectl delete hpc/<name>` | 是 |
| HPC 名称 | `metadata.name` | - |
| 命名空间 | `metadata.namespace` | - |
| 关联对象类型 | `spec.scaleTargetRef.kind`（`Deployment` 或 `HorizontalPodAutoscaler`） | - |
| 关联工作负载/HPA | `spec.scaleTargetRef.name` | - |
| 定时规则（Cron） | `spec.crons` | - |
| 目标副本数 | `spec.crons[].targetReplicas` | - |
| Cron 表达式 | `spec.crons[].schedule` | - |

## 操作步骤

### 步骤 1：理解 HPC 和 HPA 的冲突解决

当 HPA 和 HPC 同时作用于一个工作负载时，会同时生效，workload 按照 HPC 设定的时间进行扩缩容，同时根据 HPA 设定的当前使用率规则再次调整实例数。**为避免冲突，建议将 HPC 直接作用于 HPA。**

### 步骤 2：创建 HPC（作用于 HPA，推荐）

YAML 清单：

```yaml
apiVersion: autoscaling.crane.io/v1alpha1
kind: HorizontalPodCronScaler
metadata:
  name: <hpc-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: <hpa-name>
  crons:
  - name: "scale-up-morning"
    schedule: "0 8 * * 1-5"
    targetReplicas: 10
    excludeDates:
    - "* * * * 0,6"
  - name: "scale-down-evening"
    schedule: "0 20 * * 1-5"
    targetReplicas: 1
```

```bash
kubectl apply -f hpc.yaml
# expected: horizontalpodcronscaler.autoscaling.crane.io/<hpc-name> created
```

**预期输出**：

```text
horizontalpodcronscaler.autoscaling.crane.io/<hpc-name> created
```

### 步骤 3：创建 HPC（直接作用于 Deployment）

```yaml
apiVersion: autoscaling.crane.io/v1alpha1
kind: HorizontalPodCronScaler
metadata:
  name: <hpc-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  crons:
  - name: "scale-up-morning"
    schedule: "0 8 * * 1-5"
    targetReplicas: 10
  - name: "scale-down-evening"
    schedule: "0 20 * * 1-5"
    targetReplicas: 1
```

```bash
kubectl apply -f hpc.yaml
# expected: horizontalpodcronscaler.autoscaling.crane.io/<hpc-name> created
```

### 步骤 4：查看 HPC 状态

```bash
kubectl get hpc -A
# expected: 显示 HPC 列表和调度状态
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get hpc <hpc-name> -n <namespace> -o yaml
# expected: 显示 HPC 完整配置和状态
```

```text
NAME  STATUS  AGE
...
```

### 步骤 5：修改 HPC 配置

```bash
kubectl edit hpc <hpc-name> -n <namespace>
```

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
kubectl get hpc -A
# expected: HPC 资源存在

kubectl describe hpc <hpc-name> -n <namespace>
# expected: 显示 Cron 规则和执行状态

kubectl get pods -n <namespace> -l app=<app-label>
# expected: 副本数按定时规则调整
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete hpc <hpc-name> -n <namespace>
# expected: horizontalpodcronscaler.autoscaling.crane.io "<hpc-name>" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `the server doesn't have a resource type "hpc"` | `kubectl get crd` 检查 CRD | HPC 组件未安装 | 安装 HPC 组件，参见 [HPC 说明](https://cloud.tencent.com/document/product/457/56753) |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| HPC 未按时间执行 | `kubectl describe hpc <name>` 查看 Events | cron 表达式错误或 HPC controller 未运行 | 确认 cron 表达式格式，检查 HPC controller Pod 状态 |
| HPC 和 HPA 重复扩缩容 | `kubectl get hpa` 和 `kubectl get hpc` 对比 | HPC 直接作用于 Deployment 与 HPA 冲突 | 将 HPC 改为作用于 HPA |
| `FailedGetScaleTarget` | `kubectl describe hpc <name>` 查看 Events | scaleTargetRef 指向的资源不存在 | 确认 HPA/Deployment 名称和命名空间正确 |
| 目标副本数超出范围 | `kubectl describe hpc <name>` 查看 Events | HPC 指定的副本数超出 HPA min/max 范围 | 调整 HPC targetReplicas 或 HPA min/max |

## 下一步

- [HPC 说明](https://cloud.tencent.com/document/product/457/56753)
- [水平伸缩基本操作](../水平伸缩（HPA）/水平伸缩基本操作/tccli%20操作.md)
- [预测水平伸缩（EHPA）](../预测水平伸缩（EHPA）/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 自动伸缩 → HorizontalPodCronscaler → 新建，选择关联对象类型（Deployment 或 HPA），配置 cron 规则和目标副本数。
