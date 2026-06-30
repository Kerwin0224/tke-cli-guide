# 预测水平伸缩（EHPA）（tccli）

> 对照官方：[预测水平伸缩（EHPA）](https://cloud.tencent.com/document/product/457/112024) · page_id `112024`

## 概述

EffectiveHorizontalPodAutoscaler（EHPA）是 [Crane](https://gocrane.io/zh-cn/docs/) 提供的弹性伸缩产品，基于社区 HPA 进行底层弹性控制，支持更丰富的弹性触发策略（预测、观测、周期），让弹性更加高效并保障服务质量。TKE 将该功能产品化，用户可在控制台一键安装 craned 组件，创建 EHPA。

EHPA 相比社区 HPA 的优势：在流量高峰来临前提前扩容、流量降后立刻升时不进行无效缩容、弹性次数更少但更高效。本页为 tccli + kubectl 混合操作：tccli 完成 craned 组件安装（控制面），kubectl 完成 EHPA 创建和管理（数据面）。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`、`tke:DescribeAddonValues`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

```bash
# 检查集群状态
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

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI / YAML 字段 | 幂等 |
|-----------|----------------|:--:|
| 安装 craned 组件 | `tccli tke InstallAddon --ClusterId <ClusterId> --AddonName craned` | 是 |
| 更新 craned 组件 | `tccli tke UpdateAddon --ClusterId <ClusterId> --AddonName craned` | 是 |
| 查看 craned 状态 | `tccli tke DescribeAddonValues --ClusterId <ClusterId> --AddonName craned` | 是 |
| 删除 craned 组件 | `tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName craned` | 是 |
| 创建 EHPA | `kubectl apply -f ehpa.yaml` | 是 |
| 查看 EHPA 列表 | `kubectl get ehpa -A` | 是 |
| 查看 EHPA 详情 | `kubectl describe ehpa/<name>` | 是 |
| 修改 EHPA 配置 | `kubectl edit ehpa/<name>` | 否 |
| 删除 EHPA | `kubectl delete ehpa/<name>` | 是 |
| EHPA 名称 | `metadata.name` | - |
| 关联工作负载 | `spec.scaleTargetRef.name` | - |
| 实例范围（min/max） | `spec.minReplicas` / `spec.maxReplicas` | - |
| 触发策略（CPU/内存） | `spec.metrics` | - |
| 弹性预测配置 | `spec.prediction` | - |
| Scale Strategy | `spec.scaleStrategy`（`Auto` / `Preview`） | - |
| 预测算法 | `spec.prediction.predictionAlgorithm` | - |
| 预测窗口 | `spec.prediction.predictionWindowSeconds` | - |

## 操作步骤

### 步骤 1：安装和配置 craned 组件（控制面）

EHPA Controller 集成在 craned 中，依赖 Prometheus。

```bash
tccli tke InstallAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName craned \
  --AddonVersion <Version>
# expected: 返回 RequestId
```

若已有 craned 组件，升级至最新版本并更新配置：

```bash
tccli tke UpdateAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName craned \
  --AddonVersion <LatestVersion>
# expected: 返回 RequestId
```

> **说明**：安装/更新 craned 时需勾选 EHPA 并配置 Prometheus 数据查询地址、用户名和密码。

### 步骤 2：查看 craned 组件状态

```bash
tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName craned
# expected: 返回组件配置，含 craned
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

```bash
kubectl get deploy -n crane-system
# expected: craned Deployment Running
```

### 步骤 3：创建 EHPA（数据面）

YAML 清单：

```yaml
apiVersion: autoscaling.crane.io/v1alpha1
kind: EffectiveHorizontalPodAutoscaler
metadata:
  name: <ehpa-name>
  namespace: <namespace>
spec:
  maxReplicas: 10
  minReplicas: 1
  metrics:
  - resource:
      name: cpu
      target:
        averageValue: "2"
        type: AverageValue
    type: Resource
  prediction:
    predictionAlgorithm:
      algorithmType: dsp
      dsp:
        estimators: {}
        historyLength: 7d
        sampleInterval: 60s
    predictionWindowSeconds: 3600
  scaleStrategy: Auto
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
```

```bash
kubectl apply -f ehpa.yaml
# expected: effectivehorizontalpodautoscaler.autoscaling.crane.io/<ehpa-name> created
```

**预期输出**：

```text
effectivehorizontalpodautoscaler.autoscaling.crane.io/<ehpa-name> created
```

### 步骤 4：Scale Strategy 说明

| 策略 | 说明 |
|------|------|
| `Auto` | EHPA 自动执行弹性行为，创建社区 HPA 并自动接管其生命周期。**默认策略**。EHPA 删除时底层 HPA 也一并删除 |
| `Preview` | 不自动执行弹性，可通过 `desiredReplicas` 字段观测 EHPA 计算的副本数。可随时与 Auto 模式切换。`spec.specificReplicas` 为空时不对应用执行弹性 |

### 步骤 5：查看 EHPA 状态

```bash
kubectl get ehpa -A
# expected: 显示 EHPA 列表和状态
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get ehpa <ehpa-name> -n <namespace> -o yaml
# expected: 显示 EHPA 完整配置和 status

kubectl describe ehpa <ehpa-name> -n <namespace>
# expected: 显示 EHPA 详情和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 6：修改 EHPA 配置

```bash
kubectl edit ehpa <ehpa-name> -n <namespace>
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
kubectl get ehpa -A
# expected: EHPA 资源存在

kubectl get ehpa <ehpa-name> -n <namespace> -o jsonpath='{.status.recommendReplicas}'
# expected: 显示推荐副本数

kubectl get hpa -n <namespace>
# expected: EHPA 创建的底层 HPA

kubectl get pods -n <namespace> -l app=<app-label>
# expected: Pod 数量按预测策略调整
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ehpa <ehpa-name> -n <namespace>
# expected: effectivehorizontalpodautoscaler.autoscaling.crane.io "<ehpa-name>" deleted
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName craned
# expected: 返回 RequestId
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `the server doesn't have a resource type "ehpa"` | `kubectl get crd` 检查 CRD | craned 组件未安装或未就绪 | 安装 craned 组件并等待 CRD 注册完成 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| EHPA 不生效 | `kubectl describe ehpa <name>` 检查 status | craned 组件未配置 Prometheus | 确认 craned 已安装并配置 Prometheus 数据查询地址 |
| 预测指标无数据 | `kubectl get ehpa <name> -o yaml` 检查 status | Prometheus 连接失败或历史数据不足 | 检查 Prometheus 连接，确认历史数据长度满足预测算法要求（默认 7d） |
| EHPA 创建的 HPA 被手动修改 | `kubectl get hpa <name> -o yaml` 对比 | 手动修改了 EHPA 管理的底层 HPA | 不建议手动修改，通过 EHPA 配置调整 |
| EHPA 组件升级失败 | `tccli tke DescribeAddonValues --AddonName craned` 查看状态 | 组件升级错误 | 参见 [组件升级常见错误和处理](https://cloud.tencent.com/document/product/457/130406) |

## 下一步

- [定时水平伸缩（HPC）](../定时水平伸缩（HPC）/tccli%20操作.md)
- [水平伸缩基本操作](../水平伸缩（HPA）/水平伸缩基本操作/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 运维中心 → 自动伸缩 → EffectiveHorizontalPodAutoscaler → 新建，配置触发策略（CPU/内存）和弹性预测配置（Auto/Preview）。
