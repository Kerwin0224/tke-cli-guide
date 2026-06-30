# 认识 KEDA（tccli）

> 对照官方：[认识 KEDA](https://cloud.tencent.com/document/product/457/106145) · page_id `106145`

## 概述

KEDA（Kubernetes Event-Driven Autoscaling）是基于事件驱动的 Kubernetes 自动伸缩组件，通过监听外部事件源（消息队列、定时器、Prometheus 指标等）动态调整工作负载的副本数。与原生 HPA（Horizontal Pod Autoscaler）的核心区别在于：KEDA 支持**缩容至零**和**非 CPU/内存指标**驱动的伸缩，内置 50+ 种 Scaler。

### KEDA vs HPA 对比

| 维度 | HPA（Horizontal Pod Autoscaler） | KEDA |
|------|------|------|
| 触发指标 | CPU、内存（需 metrics-server） | 事件源驱动：Cron、Kafka、Prometheus、Pulsar 等 50+ Scaler |
| 缩容至零 | 不支持 | **支持**（空闲时副本数可缩至 0，事件到达后自动激活） |
| 激活延迟 | 依赖 metrics-server 采集周期（默认 15s） | 事件驱动，延迟取决于事件源推送频率 |
| 多事件源组合 | 单一指标堆叠（AND 逻辑） | 多个触发器独立工作（取最大所需副本数） |
| 定时伸缩 | 不支持原生 cron | 内置 Cron Scaler，精确到分钟级 |
| CRD 复杂度 | 简单（单个 HPA 对象） | 中等（ScaledObject + 可选的 TriggerAuthentication） |
| 安装方式 | K8s 内建，无需额外安装 | TKE 组件市场一键安装或 Helm 部署 |
| 适用场景 | 流量型、资源型伸缩 | 事件型：消息队列积压、定时扩缩、多级服务联动 |

**建议**：HPA 适合常规资源型伸缩，KEDA 适合事件驱动和定时伸缩场景。两者可以共存——KEDA 伸缩到 0 后，HPA 不再消耗资源；KEDA 在副本数 > 0 时创建的 HPA 与原生 HPA 完全兼容。

### KEDA 架构

KEDA 由三个核心组件构成：

| 组件 | 功能 |
|------|------|
| **Scaler** | 连接外部事件源（Prometheus、Kafka、RabbitMQ 等），获取事件指标值 |
| **Metrics Adapter** | 将 Scaler 获取的外部指标转换为 K8s Metrics API 格式，供 HPA 消费 |
| **Controller** | 管理 ScaledObject/ScaledJob CRD，根据事件源指标创建或删除底层 HPA |

工作流程：外部事件源 → Scaler 获取指标 → Controller 计算目标副本数 → 创建/更新 HPA → HPA 控制 Deployment/StatefulSet 副本数。当副本数降至 0 时，HPA 被删除；事件到达后 Controller 重建 HPA 并激活 Pod。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在且版本满足 KEDA 要求
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"，ClusterVersion >= 1.16
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 2
        }
    ],
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

本页为概念说明页，涉及的操作仅为查询集群信息。KEDA 本身为集群内组件，安装和 CRD 管理通过 tccli（组件）和 kubectl 完成（参见 [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md)）。

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表及 K8s 版本 | `DescribeClusters` | 是 |
| 查看集群详情 | `DescribeClusters --ClusterIds` | 是 |
| 获取连接凭证 | `DescribeClusterKubeconfig` | 是 |

## 关键字段说明

KEDA 本身不是 tccli API 资源，其配置通过 CRD 管理。以下为 KEDA 核心 CRD 字段说明，供 CLI 用户理解 KEDA 工作的语义。

### ScaledObject 核心字段

ScaledObject 是 KEDA 最常用的 CRD，定义"某个工作负载由哪些事件源触发伸缩"。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.scaleTargetRef.name` | String | 是 | 目标 Deployment/StatefulSet 名称 | 名称错误 → KEDA 无法找到目标，伸缩不生效 |
| `spec.scaleTargetRef.kind` | String | 否 | `Deployment`（默认）或 `StatefulSet` | 类型不匹配 → 伸缩对象无法绑定额外 HPA |
| `spec.minReplicaCount` | Integer | 否 | 最小副本数，默认 `0`。KEDA 支持**缩容至 0** | 设为 0 → 无事件时 Pod 全部终止，新事件到达后自动激活 |
| `spec.maxReplicaCount` | Integer | 是 | 最大副本数 | 未设置 → 无法限制上限，可能无限扩容 |
| `spec.triggers[].type` | String | 是 | 触发器类型：`cron`、`prometheus`、`kafka`、`cpu`、`memory`、`pulsar`、`rabbitmq`、`workload` 等 | 类型错误 → 触发器无法创建，ScaledObject 状态异常 |
| `spec.triggers[].metadata` | Object | 是 | 触发器参数，不同 type 有不同必填字段。例如 `cron` 需 `start`/`end`/`desiredReplicas`/`timezone` | 参数缺失或格式错误 → 触发器无法就绪 |

### ScaledJob 核心字段

ScaledJob 管理 Job 型工作负载（运行完即退出的任务）。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.jobTargetRef.template` | PodTemplateSpec | 是 | 标准 K8s Job Pod 模板 | 模板不合法 → ScaledJob 无法创建 Job |
| `spec.maxReplicaCount` | Integer | 否 | 最大并发 Job 数，默认 `100` | 未设置 → 可能产生大量并发 Job 耗尽集群资源 |
| `spec.scalingStrategy.strategy` | String | 否 | `default`、`custom`、`accurate`（默认 `default`） | — |

### TriggerAuthentication 核心字段

TriggerAuthentication 用于安全地管理 Scaler 连接外部事件源所需的凭据。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.secretTargetRef` | Array | 否 | 引用 K8s Secret 中的凭据，每个元素含 `name`（Secret 名）和 `key`（键名） | Secret 不存在 → Scaler 无法认证事件源，伸缩失败 |
| `spec.env` | Array | 否 | 直接注入环境变量（不推荐，凭据明文） | 凭据暴露 → 安全风险 |

## 操作步骤

### 步骤 1：查询集群，确认 KEDA 兼容性

KEDA 要求 Kubernetes >= 1.16。先查询目标集群的 K8s 版本和状态。

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterVersion >= 1.16，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterMaterNodeNum": 1,
            "ClusterNodeNum": 2,
            "DeletionProtection": true
        }
    ],
    "RequestId": "..."
}
```

| 字段 | 说明 | KEDA 兼容性要求 |
|------|------|----------------|
| `ClusterVersion` | K8s 版本 | >= 1.16（生产推荐 >= 1.21） |
| `ClusterStatus` | 集群状态 | `Running` |
| `ClusterNodeNum` | 工作节点数 | >= 1（至少 1 个节点运行 KEDA Operator） |

### 步骤 2：检查集群中现有 HPA 配置

在安装 KEDA 前，了解集群中已有的 HPA 配置有助于评估迁移或共存策略。

```bash
kubectl get hpa --all-namespaces
# expected: 列出所有 HPA，KEDA 安装后会额外创建 keda-hpa-* 对象
```

**预期输出**：

```text
NAMESPACE   NAME          REFERENCE                TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
default     nginx-hpa     Deployment/nginx          45%/80%          1         10        3          5d
```

### 步骤 3：KEDA Scaler 类型概览

KEDA 支持 50+ 种 Scaler，按场景可分为以下类别。

#### 类别与适用场景

| 类别 | Scaler 示例 | 需要外部凭据 | 支持缩容至零 | 典型场景 |
|------|-----------|:--:|:--:|------|
| 定时 | `cron` | 否 | 是 | 办公时段扩容、夜间缩容 |
| 消息队列 | `kafka`、`rabbitmq`、`pulsar`、`aws-sqs` | 是 | 是 | 消息积压自动扩容消费者 |
| 监控指标 | `prometheus` | 否 | 是 | 基于自定义 PromQL 指标伸缩 |
| 工作负载联动 | `workload`（kubernetes-workload） | 否 | 是 | 前端扩缩→后端跟随 |
| CPU/内存 | `cpu`、`memory` | 否 | 是 | 与 HPA 类似的资源伸缩，但支持缩至 0 |
| 网络 | `nginx` | 否 | 是 | 活跃连接数驱动伸缩 |
| 数据库 | `mysql`、`postgresql` | 是 | 是 | 数据库连接数驱动伸缩 |

### 步骤 4：决策指南——何时选 KEDA，何时选 HPA

#### 选择依据

在选择伸缩方案时，按以下决策树判断：

1. **是否仅需 CPU/内存伸缩？** → 选 HPA。无需额外安装，K8s 内建支持。
2. **是否需要定时伸缩（如每日 8:00 扩容）？** → 选 KEDA Cron Scaler。
3. **是否需要缩容至零（节省成本）？** → 必须选 KEDA。HPA 不支持 minReplicas=0。
4. **伸缩是否由消息队列积压驱动？** → 选 KEDA。HPA 需要额外的 Prometheus Adapter 转换消息指标，配置复杂。
5. **是否需要多级服务联动伸缩？** → 选 KEDA Workload Scaler。一个工作负载的副本数作为另一个的伸缩指标。
6. **已有 Prometheus 监控体系？** → 选 KEDA Prometheus Scaler。无需部署 Prometheus Adapter。

**KEDA vs HPA 决策速查表**：

| 条件 | 推荐方案 | 原因 |
|------|:------:|------|
| 仅 CPU/内存伸缩 | HPA | 内建支持，配置最简单 |
| 需要定时伸缩 | KEDA | HPA 无原生定时能力 |
| 需要缩容至零 | KEDA | HPA 不支持 minReplicas=0 |
| 消息队列驱动伸缩 | KEDA | 直接读取队列深度，无需 Adapter |
| 多级服务联动 | KEDA | Workload Scaler 原生支持 |
| 已有 Prometheus + Adapter | 两者均可 | 需要投入成本低的选 KEDA |
| 最小化组件依赖 | HPA | K8s 内建，无需安装额外 Operator |

## 验证

### 确认集群兼容性

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"，ClusterVersion >= 1.16
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 2
        }
    ]
}
```

```bash
# 确认节点数
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNodeNum'
# expected: >= 1
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

### 验证维度总结

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |
| K8s 版本 | 同上，查看 `ClusterVersion` | >= `1.16.x` |
| 节点数 | 同上，查看 `ClusterNodeNum` | >= 1 |
| HPA 现状 | `kubectl get hpa -A` | 无异常，作为 KEDA 安装前基线记录 |

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 概念理解疑问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 不确定选 KEDA 还是 HPA | 列出伸缩需求：是否需要缩容至零、定时伸缩、消息队列驱动 | HPA 仅支持 CPU/内存指标，KEDA 扩展了事件类型和缩容至零能力 | 纯资源伸缩用 HPA（更简单）；事件驱动、定时、多级联动用 KEDA。两者可共存 |
| KEDA ScaledObject 创建后不触发伸缩 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 Events 和 Conditions | 触发器配置错误（metadata 中事件源连接信息不匹配）或 KEDA Operator 未安装 | 检查 `triggers[].type` 和 `triggers[].metadata` 字段是否与 Scaler 规范一致；确认 KEDA 已部署（`kubectl get pods -n keda`） |
| 不确定 Scaler 选 cron 还是 prometheus | 分析触发条件的性质：是**时间规律**还是**指标阈值** | cron 按固定时间触发，不感知外部指标；prometheus 按指标阈值触发，响应实时变化 | 可预测的时间模式（如办公时间）用 cron；不可预测的指标波动（如错误率突增）用 prometheus。两者可在同一 ScaledObject 中组合 |
| ScaledObject 与已有 HPA 冲突 | `kubectl get hpa -n NAMESPACE` 确认是否有多个 HPA 指向同一工作负载 | 同一工作负载被多个 HPA 管理会导致指标冲突 | KEDA 创建的 HPA（命名 `keda-hpa-SCALED_OBJECT_NAME`）与手工 HPA 不应指向同一 workload。有冲突时删除手工 HPA，让 KEDA 统一管理 |

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterIds` 中的集群 ID 格式：`tccli tke DescribeClusters --region <Region>` 列出全部集群 | 集群 ID 格式错误或集群不属于当前账号/地域 | 确认集群 ID 格式为 `CLUSTER_ID`；检查 `tccli configure list` 确认 region 与集群所在区域一致 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `kubectl get hpa` 返回 `the server doesn't have a resource type "hpa"` | `kubectl api-resources \| grep horizontal` 确认 API 资源组 | kubeconfig 连接了错误的集群或凭据过期 | 重新获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \| jq -r '.Kubeconfig' > ~/.kube/config` |

## 下一步

- [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md) — 通过 TKE 组件市场安装 KEDA Operator
- [定时水平伸缩（Cron 触发器）](../定时水平伸缩%20(Cron%20触发器)/tccli%20操作.md) — 基于时间表的伸缩配置
- [多级服务同步水平伸缩（Workload 触发器）](../多级服务同步水平伸缩（Workload%20触发器）/tccli%20操作.md) — 工作负载联动伸缩
- [基于 Prometheus 自定义指标的弹性伸缩](../基于%20Prometheus%20自定义指标的弹性伸缩/tccli%20操作.md) — 使用 Prometheus 指标驱动伸缩
- [KEDA 官方文档](https://keda.sh/docs/) — Scaler 完整列表与参数参考

## 控制台替代

[TKE 控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：搜索 `KEDA` 组件查看详情；或参考 [TKE KEDA 文档](https://cloud.tencent.com/document/product/457/106145)。
