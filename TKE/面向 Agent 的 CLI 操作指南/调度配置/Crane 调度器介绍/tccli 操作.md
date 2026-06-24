# Crane 调度器介绍（tccli）

> 对照官方：[Crane 调度器介绍](https://cloud.tencent.com/document/product/457/75472) · page_id `75472`

## 概述

Crane 调度器（Crane Scheduler）是 TKE 原生节点提供的智能调度器，基于节点**真实资源利用率**而非传统的 Pod `requests` 进行调度决策。原生 Kubernetes 默认调度器（kube-scheduler）仅基于 Pod 声明的 `requests` 值做调度，但 `requests` 往往与实际使用差距巨大，导致集群负载不均、资源浪费。

Crane 调度器的核心特性：
- **真实负载感知**：通过 QoSAgent 采集节点实时 CPU/内存利用率，调度时考虑实际负载
- **热点规避**：自动识别并避开已高负载的节点
- **历史趋势预测**：结合历史负载数据（Prometheus），预测未来负载走势
- **动态水位线**：可配置目标利用率阈值，兼顾资源利用率和可靠性

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Kubernetes 调度器工作原理（Scheduling Framework、Filter/Score 阶段）
- 了解 Kubernetes 资源模型（`requests` / `limits` / 资源超卖）
- **演示集群**：`cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running, 3 节点, tlinux3.1），**kubectl 因 CAM 策略限制不可达**（strategyId: 240463971），数据面操作需通过 VPN/IOA 内网或数据面集群执行

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region = ap-guangzhou

# 3. 确认演示集群存在且为 Running
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
# expected: TotalCount >= 1，ClusterStatus = "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-a1b2c3d4",
            "ProjectId": 0,
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群基本信息 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看 Crane 调度器组件状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName cranescheduler --region ap-guangzhou` | 是 |
| 查看 QoSAgent 组件状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName qosagent --region ap-guangzhou` | 是 |
| 查看调度策略 | `tccli tke DescribeClusterSchedulerPolicy --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看节点列表 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 安装/配置 Crane 调度器 | 控制台 → 组件管理 → Crane 调度器（CLI 不支持完整配置） | 否 |
| 修改调度策略 | `tccli tke ModifyClusterSchedulerPolicy` 或控制台 | 否 |

## 操作步骤

### 步骤 1：确认 Crane 调度器组件已安装

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName cranescheduler \
    --region ap-guangzhou --output json
# expected: Addons 数组非空，Phase = "Active"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "cranescheduler",
            "AddonVersion": "2.0.1",
            "AddonStatus": "Running",
            "RawValues": "{\"resources\":{\"limits\":{\"cpu\":\"1\",\"memory\":\"1Gi\"},\"requests\":{\"cpu\":\"500m\",\"memory\":\"512Mi\"}},\"replicas\":2,\"config\":{\"hotValue\":80,\"policyName\":\"balanced\"}}",
            "Phase": "Active"
        }
    ],
    "RequestId": "e5f6a7b8-c9d0-1234-ef56-7890abcdef12"
}
```

若返回空，说明 Crane 调度器未安装，需在控制台 "组件管理" 中安装。

### 步骤 2：确认 QoSAgent 组件状态（Crane 调度器依赖）

Crane 调度器依赖 QoSAgent 采集节点真实负载数据：

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0] | {AddonName, AddonVersion, Phase}'
# expected: Phase = "Active"
```

**预期输出**：

```json
{
    "AddonName": "qosagent",
    "AddonVersion": "1.2.1",
    "Phase": "Active"
}
```

### 步骤 3：理解 Crane 调度器工作原理

**传统 kube-scheduler 的问题**：

```
Pod A (requests: 1 CPU, actually uses: 0.1 CPU)
Pod B (requests: 1 CPU, actually uses: 0.9 CPU)

kube-scheduler 视角：Node 仍可调度 2 个 CPU（基于 requests 计算）
实际情况：Node 已接近满载（0.1 + 0.9 = 1.0 实际使用）
```

**Crane 调度器解决方案**：

```
                    ┌─────────────┐
                    │  Crane       │
                    │  Scheduler   │ ← Kubernetes Scheduling Framework 插件
                    └──────┬──────┘
                           │ 查询节点真实负载
                           ▼
                    ┌─────────────┐
                    │  QoSAgent   │ ← 节点级代理，采集 CPU/Mem/IO 数据
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Prometheus │ ← 历史负载数据（可选）
                    └─────────────┘
```

调度流程：

1. **Filter 阶段**：过滤掉负载超过水位的节点（如真实 CPU 利用率 > 80%），即使该节点 requests 层面未满
2. **Score 阶段**：对候选节点按实际负载打分，负载越低得分越高
3. **热点规避**：结合 Prometheus 历史数据预测短期负载趋势，提前规避即将过载的节点
4. **Pod 调度**：将 Pod 调度到得分最优的节点

**关键配置参数**（通过 ModifyClusterSchedulerPolicy 或控制台配置）：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `hotValue` | 热点节点判定阈值（真实 CPU 利用率百分比） | 70 ~ 85 |
| `policyName` | 调度策略名称：`balanced`（均衡）、`high-utilization`（高利用率）、`low-latency`（低延迟） | `balanced` |

### 步骤 4：查看当前调度策略

```bash
tccli tke DescribeClusterSchedulerPolicy \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json
# expected: 返回当前调度器策略配置
```

**预期输出**：

```json
{
    "SchedulerPolicy": {
        "PolicyName": "balanced",
        "HotValue": 80,
        "Enabled": true
    },
    "RequestId": "f6a7b8c9-d0e1-2345-f678-90abcdef1234"
}
```

## 验证

```bash
# 1. 确认集群状态为 Running
tccli tke DescribeClusterStatus \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.ClusterStatusSet[0].ClusterState'
# expected: "Running"

# 2. 确认 Crane 调度器组件正常运行
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName cranescheduler \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].Phase'
# expected: "Active"

# 3. 确认 QoSAgent 组件正常运行
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].Phase'
# expected: "Active"

# 4. 确认调度策略已启用
tccli tke DescribeClusterSchedulerPolicy \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.SchedulerPolicy.Enabled'
# expected: true
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

## 清理

本页为功能说明与状态查询，未创建新资源，无需清理。若需切换回默认调度器，可在控制台组件管理中卸载 Crane 调度器，但**注意**：卸载后所有基于 Crane 的调度策略将失效，新 Pod 将回退到原生 kube-scheduler 调度。

## 排障

### 组件状态异常

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| `DescribeAddon cranescheduler` 返回空 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 列出全部组件 | Crane 调度器未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 调度器 |
| `DescribeAddon cranescheduler` 状态非 Active | 查看 AddonStatus 字段，可能为 `Failed` 或 `Upgrading` | 组件安装失败或正在升级 | 在控制台查看组件事件日志，尝试重装或升级 |
| `DescribeAddon qosagent` 返回空 | Crane 调度器依赖 QoSAgent | QoSAgent 未随 Crane 一同安装 | 安装 Crane 套件时勾选 QoSAgent 组件 |

### 调度功能问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| Crane 调度器未生效 | 1. `tccli tke DescribeClusterSchedulerPolicy --ClusterId cls-xxxxxxxx --region ap-guangzhou` 确认 `Enabled` 为 true；2. 检查 Pod 是否有正确的 schedulerName | 调度策略未启用，或 Pod 未指定 `schedulerName: crane-scheduler` | 启用车调度策略，确保 Pod YAML 中 `schedulerName` 设置为 `crane-scheduler` |
| 节点负载仍不均 | 1. 确认 `hotValue` 配置是否合适；2. 确认 QoSAgent 数据采集正常 | 热点阈值配置过高，或 QoSAgent 采集延迟 | 降低 `hotValue` 阈值，检查 QoSAgent 健康状态 |
| 控制台无 Crane 调度器配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |

## 下一步

- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) -- page_id `79774`
- [CPU Burst](../CPU%20Burst/tccli%20操作.md) -- page_id `79776`
- [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) -- page_id `79777`
- [节点放大](../节点放大/tccli%20操作.md) -- page_id `79032`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → Crane 调度器](https://console.cloud.tencent.com/tke2/cluster?rid=1) 提供可视化安装、配置和卸载操作。控制台还提供调度策略配置界面（`hotValue`、`policyName`）、组件版本管理和运行状态监控。CLI 可通过 `DescribeAddon` 和 `DescribeClusterSchedulerPolicy` 查询状态，但完整配置需通过控制台或 `ModifyClusterSchedulerPolicy` API。
