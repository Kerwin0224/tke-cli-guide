# CPU Burst（tccli）

> 对照官方：[CPU Burst](https://cloud.tencent.com/document/product/457/79776) · page_id `79776`

## 概述

CPU Burst 是 TKE 原生节点 QoSAgent 提供的 CPU 弹性增强能力，允许容器在短时间窗口内突破 `limits` 限制使用更多 CPU，适用于流量突发场景（如秒杀、消息队列消费峰值），无需修改容器的 CPU limits 配置即可临时获得更高的 CPU 算力。

核心价值：避免因短暂的 CPU 限流（throttling）导致 P99 延迟飙升，同时不破坏 Kubernetes 资源模型。

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Kubernetes CPU 资源管理机制（`requests` / `limits` / `throttling`）
- 集群需为 TKE 原生节点类型（非 Super Node）
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
| 查看节点列表及资源使用 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 启用/配置 CPU Burst | 控制台 → 组件管理 → QoSAgent → 策略配置（CLI 不可直接配置） | 否 |
| 查看 CPU Burst 策略 | `kubectl` 查询 Crane CRD（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：确认 QoSAgent 组件已安装

CPU Burst 依赖 QoSAgent 组件。先确认组件状态：

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json
# expected: Addons 数组非空，Phase = "Running"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "qosagent",
            "AddonVersion": "1.2.1",
            "AddonStatus": "Running",
            "RawValues": "{\"nodeSelector\":{\"matchLabels\":{}},\"replicas\":1,\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"512Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"128Mi\"}}}",
            "Phase": "Active"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-fg2345678901"
}
```

若返回空 `Addons` 数组，说明 QoSAgent 未安装，需在控制台 "组件管理" 中安装 Crane 套件。

### 步骤 2：理解 CPU Burst 工作原理

CPU Burst 基于 Linux cgroup 的 CFS（Completely Fair Scheduler）带宽控制机制实现：

- **正常状态**：容器 CPU 使用受 `limits` 上限约束，cgroup CFS bandwidth quota 按 limits 值分配
- **触发条件**：QoSAgent 监测到容器存在 **CPU Throttling**（即限流事件），且节点有空闲 CPU
- **Burst 状态**：QoSAgent 动态调整 cgroup `cpu.cfs_quota_us`，允许容器临时使用更多 CPU 时间片
- **安全边界**：Burst 仅使用节点 **空闲** CPU 资源，不抢占其他容器的份额；累计 Burst 时长受配置上限约束
- **恢复**：Burst 时间窗口结束后，自动恢复为原始 `limits` 值，进入冷却期

**关键配置参数**（通过 Crane 自定义资源定义设置，需 kubectl）：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `cpu-burst-factor` | 允许 Burst 的最大 CPU 倍数（相对 limits） | 1.5 ~ 3.0 |
| `cpu-burst-window` | Burst 统计时间窗口 | 10s ~ 60s |
| `cpu-burst-duration` | 单次 Burst 最大持续时间 | 5s ~ 30s |
| `cool-down-period` | 一次 Burst 后的冷却时间 | 30s ~ 120s |

### 步骤 3：查看集群节点资源状况（辅助诊断）

```bash
tccli tke DescribeClusterInstances \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.InstanceSet[] | {InstanceId, InstanceRole, InstanceState}'
# expected: 返回节点列表及其角色和状态
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
    "RequestId": "c3d4e5f6-a7b8-9c0d-ef12-34567890abcd"
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

# 2. 确认 QoSAgent 组件正常运行
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].Phase'
# expected: "Active"

# 3. 确认集群版本支持 CPU Burst（需 TKE >= 1.20）
tccli tke DescribeClusters \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.Clusters[0].ClusterVersion'
# expected: >= "1.20.0"（演示集群 v1.30.0，满足要求）
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

本页为功能说明与集群状态查看，未创建新资源，无需清理。若需关闭 CPU Burst，可在控制台组件管理中修改 QoSAgent 策略配置。

## 排障

### 功能未生效

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 容器 CPU 使用未突破 limits | 1. `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName qosagent --region ap-guangzhou` 确认组件状态；2. 通过 kubectl 检查 Crane CRD `cpu-burst-factor` 配置（需 VPN/IOA） | QoSAgent 未安装或策略未配置 Burst 倍率 | 在控制台安装 QoSAgent 并配置 Burst 策略 |
| CPU Burst 触发但 P99 延迟未改善 | 1. 确认节点有空闲 CPU（`tccli tke DescribeClusterInstances` 查看节点规格）；2. 检查 Burst 倍数是否过低 | 节点 CPU 资源已用满，无可用的空闲 CPU | 扩容节点或降低其他容器 limits |
| `DescribeAddon` 返回空 | 执行 `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 查看已安装组件列表 | QoSAgent 组件未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 套件（包含 QoSAgent） |

### 其他常见问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 控制台无 CPU Burst 配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |
| Burst 后容器 OOM | 检查容器内存 limits 是否足够 | CPU Burst 仅针对 CPU，内存仍受 limits 约束；CPU 使用突增可能间接增加内存消耗 | 适当提高内存 limits |

## 下一步

- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) -- page_id `79774`
- [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) -- page_id `79777`
- [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → QoSAgent](https://console.cloud.tencent.com/tke2/cluster?rid=1) 中的策略配置，对应 CPU Burst 参数配置。控制台提供可视化配置界面，CLI 仅支持查询组件状态和集群信息。
