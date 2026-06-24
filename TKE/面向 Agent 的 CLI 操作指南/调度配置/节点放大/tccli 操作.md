# 节点放大（tccli）

> 对照官方：[节点放大](https://cloud.tencent.com/document/product/457/79032) · page_id `79032`

## 概述

节点放大（Node Amplification）是 Crane 调度器提供的资源超卖能力，通过**降低节点系统资源预留比例**来释放更多资源给业务 Pod，从而提高节点装箱率、降低集群整体资源成本。

在 Kubernetes 中，每个节点默认预留一部分资源给系统守护进程（kubelet、containerd、kube-proxy、操作系统等）。节点放大功能允许在保证节点稳定性的前提下，有策略地降低预留比例，将节省出来的资源调度给业务 Pod。

核心价值：在不增加节点数量的情况下提升集群可调度资源总量，降低基础设施成本。

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Kubernetes 节点资源预留机制（`system-reserved`、`kube-reserved`、`eviction-hard`）
- 理解节点资源超卖的概念和风险
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
| 查看节点列表和资源规格 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看节点资源使用概况 | `tccli tke DescribeResourceUsage --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 配置节点放大策略 | 控制台 → 组件管理 → Crane → 节点放大（CLI 不可直接配置参数） | 否 |
| 查看节点放大配置 | `kubectl` 查询 Crane CRD（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：确认 Crane 调度器已安装

节点放大是 Crane 调度器的子功能，首先确认组件状态：

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
    "RequestId": "a7b8c9d0-e1f2-3456-7890-abcdef123456"
}
```

### 步骤 2：理解节点放大工作原理

**节点资源分配示意**（以 4C8G 节点为例）：

```
节点总资源 (4C8G)
├── 系统预留 (默认 ~20%)
│   ├── kube-reserved:    CPU 200m, Mem 500Mi
│   ├── system-reserved:  CPU 200m, Mem 500Mi
│   └── eviction-threshold: Mem 800Mi
│
└── 可调度资源 (Allocatable)
    ├── 默认：~3.6C / ~6.2G
    └── 节点放大后：~3.8C / ~7.0G（预留比例降低到 ~10%）
```

**节点放大机制**：

1. **预留比例调整**：Crane 通过 CRD 修改各节点的资源预留参数
   - 默认 `system-reserved` 和 `kube-reserved` 保守预留以确保极端情况下的节点稳定
   - 实际生产环境中，许多节点系统组件用量远低于预留值，节点放大可回收这部分 "闲置预留"
2. **安全水位线**：配置节点最大利用率阈值（如 85%），防止节点过载
3. **逐节点差异化**：不同规格的节点（如 4C8G vs 16C32G）可配置不同的放大比例，大规格节点通常有更大的放大空间

**关键配置参数**（通过 Crane CRD 设置，需 kubectl）：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `reserved-resource-ratio` | 总预留资源比例（相对于节点总资源） | 0.10 ~ 0.20（默认 0.20） |
| `max-utilization-threshold` | 节点最大安全利用率 | 0.80 ~ 0.90 |
| `amplification-ratio` | 放大系数（释放的额外倍率） | 1.0 ~ 1.5 |
| `target-load-threshold` | 目标负载水位（可调度资源使用率上限） | 0.75 ~ 0.85 |

### 步骤 3：查看节点资源使用概况

```bash
tccli tke DescribeResourceUsage \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json
# expected: 返回集群资源使用概况（CPU/内存总量、已分配量等）
```

**预期输出**：

```json
{
    "ResourceUsage": {
        "TotalCPU": 12.0,
        "AllocatedCPU": 5.5,
        "TotalMemory": 24.0,
        "AllocatedMemory": 15.2,
        "NodeCount": 3
    },
    "RequestId": "b8c9d0e1-f2a3-4567-8901-bcdef1234567"
}
```

| 字段 | 说明 |
|------|------|
| `TotalCPU` / `TotalMemory` | 集群所有节点可调度资源总量（Allocatable 级别） |
| `AllocatedCPU` / `AllocatedMemory` | 已分配给 Pod 的 `requests` 总和 |
| `NodeCount` | 节点数量 |

> 节点放大生效后，`TotalCPU` 和 `TotalMemory` 会相应增加（因为降低了系统预留比例）。

### 步骤 4：查看节点列表及资源规格

```bash
tccli tke DescribeClusterInstances \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.InstanceSet[] | {InstanceId, InstanceType, CPU, Memory, InstanceState}'
# expected: 返回各节点详细信息和状态
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-a1b2c3d4",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.1.10",
            "InstanceType": "S5.LARGE8",
            "CPU": 4,
            "Memory": 8
        },
        {
            "InstanceId": "ins-e5f6g7h8",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.1.11",
            "InstanceType": "S5.LARGE8",
            "CPU": 4,
            "Memory": 8
        },
        {
            "InstanceId": "ins-i9j0k1l2",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.1.12",
            "InstanceType": "S5.LARGE8",
            "CPU": 4,
            "Memory": 8
        }
    ],
    "TotalCount": 3,
    "RequestId": "c0d1e2f3-a4b5-6789-0123-def456789abc"
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

# 4. 查看集群资源使用概况（对比节点放大前后的 TotalCPU/TotalMemory）
tccli tke DescribeResourceUsage \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.ResourceUsage | {TotalCPU, AllocatedCPU, TotalMemory, AllocatedMemory, NodeCount}'
# expected: 返回资源使用概况，TotalCPU 应反映放大后的可调度资源量
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

本页为功能说明与集群状态查询，未创建新资源，无需清理。若需关闭节点放大，可在控制台组件管理中调整 Crane 调度器的预留资源配置，将 `reserved-resource-ratio` 恢复为默认值（如 0.20）。

**风险提示**：关闭节点放大后，节点的可调度资源（Allocatable）将减少，如果集群中 Pod `requests` 已超过新的 Allocatable 总量，Pod 将进入 `Pending` 状态，需扩容节点或降低 Pod requests。

## 排障

### 功能未生效

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 节点可调度资源未增加 | 1. `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName cranescheduler --region ap-guangzhou` 确认组件状态；2. 通过 kubectl 检查节点 `allocatable` 字段（需 VPN/IOA） | Crane 未安装或节点放大策略未配置 | 安装 Crane 并配置节点放大策略 |
| `DescribeAddon cranescheduler` 返回空 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 列出全部组件 | Crane 调度器未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 调度器 |
| 控制台无节点放大配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |

### 稳定性问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 节点放大后 Pod OOMKilled 增多 | 1. `tccli tke DescribeResourceUsage --ClusterId cls-xxxxxxxx --region ap-guangzhou` 查看资源分配率；2. 检查 `max-utilization-threshold` 配置是否过高 | 放大比例过高导致节点过载，触发 OOM Killer | 降低放大比例或提高 `max-utilization-threshold`，并确保配置了合理的驱逐阈值 |
| 节点 CPU 持续高负载 | 1. 通过 kubectl top nodes 确认（需 VPN/IOA）；2. 检查 Crane `target-load-threshold` | 业务 Pod 实际使用超过预留比例，节点过载 | 降低放大比例，或增加节点数量分摊负载 |
| 节点 NotReady | 检查系统组件（kubelet、containerd）是否有 OOM 或 CPU 限流 | 预留资源不足导致核心组件异常 | 立即恢复默认预留比例，排查后重新调整 |

## 下一步

- [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) -- page_id `79774`
- [自动规整集群资源](../资源利用率优化调度/自动规整集群资源/tccli%20操作.md) -- page_id `--`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → Crane 调度器](https://console.cloud.tencent.com/tke2/cluster?rid=1) 中的节点放大策略配置，提供可视化的预留比例滑块、安全水位设置和逐节点差异化配置。控制台还会展示放大前后可调度资源的对比预览，帮助运维人员直观评估收益。CLI 可用于查询组件状态和资源概况，但策略配置需通过控制台或 kubectl 操作 CRD。
