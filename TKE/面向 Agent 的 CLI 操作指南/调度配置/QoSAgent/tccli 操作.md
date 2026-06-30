# QoSAgent（tccli）

> 对照官方：[QoSAgent](https://cloud.tencent.com/document/product/457/79774) · page_id `79774`

## 概述

QoSAgent 是 TKE Crane 套件中的**节点级 QoS 感知代理**组件，以 DaemonSet 方式运行在每个原生节点上，负责：实时采集节点多维度的资源使用数据，执行 CPU Burst、内存精细管理、磁盘 IO 隔离、网络带宽隔离、超线程隔离等 QoS 策略，并将节点负载数据上报给 Crane 调度器供调度决策使用。

QoSAgent 是 Crane 生态的核心数据面组件，Crane 调度器、CPU Burst、节点放大等上层功能均依赖 QoSAgent 提供的实时负载数据。

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Linux cgroup 机制（cpuset、cpu、memory、blkio）
- 了解 DaemonSet 工作原理
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
| 查看 QoSAgent 组件状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName qosagent --region ap-guangzhou` | 是 |
| 查看 Crane 调度器组件状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName cranescheduler --region ap-guangzhou` | 是 |
| 查看节点列表 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看资源使用概况 | `tccli tke DescribeResourceUsage --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 安装/配置 QoSAgent | 控制台 → 组件管理 → Crane → QoSAgent（CLI 不支持完整配置） | 否 |
| 查看 QoSAgent 采集数据 | `kubectl logs` / Prometheus 查询（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：确认 QoSAgent 组件已安装

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json
# expected: Addons 数组非空，Phase = "Active"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "qosagent",
            "AddonVersion": "1.2.1",
            "AddonStatus": "Running",
            "RawValues": "{\"nodeSelector\":{\"matchLabels\":{}},\"replicas\":1,\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"512Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"128Mi\"}},\"qosConfig\":{\"cpuBurstEnabled\":true,\"ioIsolationEnabled\":true,\"htIsolationEnabled\":false}}",
            "Phase": "Active"
        }
    ],
    "RequestId": "a7b8c9d0-e1f2-3456-7890-abcdef123456"
}
```

| 字段 | 说明 |
|------|------|
| `AddonName` | 组件名称，此处为 `qosagent` |
| `AddonVersion` | 当前安装的版本号 |
| `AddonStatus` | 组件运行状态，`Running` 表示正常工作 |
| `Phase` | 生命周期阶段，`Active` 表示已激活 |
| `RawValues` | 组件配置值（JSON 字符串），包含功能开关和资源设定 |

### 步骤 2：理解 QoSAgent 架构与功能

**架构层次**：

```
┌─────────────────────────────────────────────────────────────┐
│                      Crane 调度器                            │
│              （基于真实负载的智能调度）                         │
└────────────────────────────┬────────────────────────────────┘
                             │ 查询节点负载数据
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      QoSAgent                                │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐   │
│  │  CPU     │  Memory  │  Disk    │  Network │  HT      │   │
│  │  Burst   │  管理    │  IO 隔离 │  隔离    │  隔离    │   │
│  └────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬─────┘   │
│       │          │          │          │          │          │
│       ▼          ▼          ▼          ▼          ▼          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │               Linux cgroup 层                          │   │
│  │  cpu.cfs_quota_us / memory.limit / blkio.weight /     │   │
│  │  cpuset.cpus / net_cls                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**五大核心功能模块**：

| 模块 | 功能说明 | 下层控制器 | 关联文档 |
|------|---------|-----------|---------|
| **CPU Burst** | 监测容器 CPU 限流（throttling），当节点有空闲 CPU 时允许短时间突破 limits | cgroup CFS bandwidth | [CPU Burst](../CPU%20Burst/tccli%20操作.md) |
| **内存精细管理** | PageCache 回收策略、内存水位控制、容器 OOM 优先级管理 | cgroup memory controller | [内存精细调度](../内存精细调度/tccli%20操作.md) |
| **磁盘 IO 隔离** | 按权重分配磁盘 IO 带宽、IOPS 和 BPS 限制 | cgroup blkio controller | [磁盘 IO 精细调度](../磁盘%20IO%20精细调度/tccli%20操作.md) |
| **网络带宽隔离** | 容器级别出/入方向带宽限制 | tc / cgroup net_cls | [网络精细调度](../../调度配置/) |
| **超线程隔离** | 物理核级别 CPU 隔离，避免共享 Cache 的 Noisy Neighbor | cgroup cpuset controller | [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) |

### 步骤 3：确认 QoSAgent 依赖的 Crane 调度器状态

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName cranescheduler \
    --region ap-guangzhou --output json \
    | jq '.Addons[0] | {AddonName, AddonVersion, Phase}'
# expected: Phase = "Active"
```

**预期输出**：

```json
{
    "AddonName": "cranescheduler",
    "AddonVersion": "2.0.1",
    "Phase": "Active"
}
```

### 步骤 4：查看节点资源使用概况（反映 QoSAgent 数据采集范围）

QoSAgent 采集的节点负载数据会影响 `DescribeResourceUsage` 的统计口径：

```bash
tccli tke DescribeResourceUsage \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.ResourceUsage'
# expected: 返回集群级资源使用概况
```

**预期输出**：

```json
{
    "ResourceUsage": {
        "TotalCPU": 12.0,
        "AllocatedCPU": 6.8,
        "TotalMemory": 24.0,
        "AllocatedMemory": 16.5,
        "NodeCount": 3
    },
    "RequestId": "b8c9d0e1-f2a3-4567-8901-bcdef1234567"
}
```

> QoSAgent 采集的实时负载数据（如节点真实 CPU 使用率、内存使用量、磁盘 IO 速率）需通过 Prometheus 查询或 `kubectl top node` 获取（需 VPN/IOA 内网）。

## 验证

```bash
# 1. 确认集群状态为 Running
tccli tke DescribeClusterStatus \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.ClusterStatusSet[0].ClusterState'
# expected: "Running"

# 2. 确认 QoSAgent 组件正常运行
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0] | {AddonName, AddonVersion, AddonStatus, Phase}'
# expected: AddonStatus = "Running", Phase = "Active"

# 3. 确认 Crane 调度器组件正常运行（QoSAgent 的上层消费者）
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName cranescheduler \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].Phase'
# expected: "Active"

# 4. 确认集群版本支持 QoSAgent
tccli tke DescribeClusters \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.Clusters[0].ClusterVersion'
# expected: >= "1.20.0"（演示集群 v1.30.0，满足要求）

# 5. 查看 QoSAgent 功能开关状态
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].RawValues | fromjson | .qosConfig'
# expected: 返回各功能模块的开关状态
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

本页为功能说明与集群状态查询，未创建新资源，无需清理。若需卸载 QoSAgent，可在控制台组件管理中卸载 Crane 套件，但**注意**：

- 卸载 QoSAgent 会导致 CPU Burst、IO 隔离、超线程隔离等所有 QoS 策略立即失效
- Crane 调度器将无法获取节点真实负载数据，退化为基于 requests 的调度
- 已在运行的 Pod 不受影响（cgroup 策略不会自动回滚），但新 Pod 不再享受 QoS 能力

## 排障

### 组件状态异常

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| `DescribeAddon qosagent` 返回空 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 列出全部组件 | QoSAgent 未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 套件（勾选 QoSAgent） |
| QoSAgent Phase 非 Active | 查看 `AddonStatus` 字段是否为 `Failed` 或 `Pending` | 组件安装失败或节点资源不足 | 在控制台查看组件事件日志，确认节点有足够资源运行 QoSAgent DaemonSet |
| QoSAgent 已安装但功能未生效 | 1. 通过 kubectl 检查 qosagent DaemonSet 的 Pod 是否全部 Ready；2. 检查 `RawValues` 中功能开关是否启用 | QoSAgent Pod 未就绪，或功能开关未开启 | 确保所有节点上的 qosagent Pod 为 Running，在控制台开启需要的功能开关 |

### 功能配置问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| CPU Burst 不工作 | `DescribeAddon qosagent` 查看 `RawValues` 中 `cpuBurstEnabled` 是否为 true | CPU Burst 功能未启用 | 在控制台 QoSAgent 策略配置中开启 CPU Burst |
| IO 隔离不工作 | 同上，检查 `ioIsolationEnabled` | IO 隔离功能未启用 | 在控制台开启 IO 隔离 |
| 超线程隔离不工作 | 同上，检查 `htIsolationEnabled`，并确认节点 BIOS 开启了超线程 | 功能未启用或节点不支持超线程 | 启用功能，确认节点硬件支持 |
| 控制台无 QoSAgent 配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |

### 性能影响

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| QoSAgent 自身 CPU 开销过高 | `kubectl top pod -n kube-system \| grep qosagent`（需 VPN/IOA） | 资源采集频率过高或节点数过多 | 在 RawValues 中调整 `resources.requests` 和采集频率参数 |
| 节点负载数据延迟大 | 检查 QoSAgent 日志是否报错（`kubectl logs`，需 VPN/IOA） | 数据上报链路阻塞或 Prometheus 压力过大 | 调整上报间隔参数 |

## 下一步

- [CPU Burst](../CPU%20Burst/tccli%20操作.md) -- page_id `79776`
- [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) -- page_id `79777`
- [磁盘 IO 精细调度](../磁盘%20IO%20精细调度/tccli%20操作.md) -- page_id `79781`
- [内存精细调度](../内存精细调度/tccli%20操作.md) -- page_id `--`
- [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → Crane 套件 → QoSAgent](https://console.cloud.tencent.com/tke2/cluster?rid=1) 提供可视化安装、策略配置（各功能模块的独立开关和参数）和运行状态监控。控制台还展示 QoSAgent 各节点 Pod 的健康状态和资源消耗。CLI 可通过 `DescribeAddon` 查询组件状态，但策略配置需通过控制台或直接修改 Crane CRD。
