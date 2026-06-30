# CPU 超线程隔离（tccli）

> 对照官方：[CPU 超线程隔离](https://cloud.tencent.com/document/product/457/79777) · page_id `79777`

## 概述

CPU 超线程（Hyper-Threading）隔离是 QoSAgent 提供的物理核级别 CPU 隔离能力。在同一物理 CPU 核上，两个超线程（HT）共享 L1/L2 Cache 和执行单元，若不做隔离，一个 Pod 的高负载会干扰同物理核的另一 Pod（Noisy Neighbor 效应），导致延迟敏感型业务抖动。

该功能通过 QoSAgent 识别关键业务 Pod，将其独占物理 CPU 核，其他 Pod 使用不同物理核的超线程，实现物理核级别的隔离。

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 CPU 超线程（Hyper-Threading / SMT）概念和 Linux cgroup cpuset
- 集群需为 TKE 原生节点类型，且物理机开启超线程
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
| 查看节点 CPU 信息（含超线程） | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 启用/配置超线程隔离 | 控制台 → 组件管理 → QoSAgent → 高级配置（CLI 不可直接配置策略） | 否 |
| 查看超线程隔离策略 | `kubectl` 查询 Crane CRD（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：确认 QoSAgent 组件已安装

超线程隔离依赖 QoSAgent 的 NUMA 感知和 cpuset 管理能力：

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
            "RawValues": "{\"nodeSelector\":{\"matchLabels\":{}},\"replicas\":1,\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"512Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"128Mi\"}}}",
            "Phase": "Active"
        }
    ],
    "RequestId": "c2d3e4f5-a6b7-8901-cdef-ab1234567890"
}
```

### 步骤 2：理解超线程隔离工作原理

在支持超线程（Intel HT / AMD SMT）的物理机上，一个物理核包含 2 个逻辑核：

```
物理核 0:
  ├── 逻辑核 0 (cpu0)  ─┐
  └── 逻辑核 1 (cpu1)  ─┘ 共享 L1/L2 Cache、执行单元
```

**Noisy Neighbor 问题**：

- Pod A 调度到 cpu0，Pod B 调度到 cpu1（同物理核）
- Pod A 执行密集计算 → Cache 污染 → Pod B 命中率下降 → 延迟飙升
- 延迟敏感型业务（如金融交易、实时推荐）受严重影响

**超线程隔离策略**（QoSAgent 实现）：

1. QoSAgent 识别关键业务 Pod（通过 Pod Annotation 或 PriorityClass 标记）
2. 通过 cgroup `cpuset.cpus` 将关键 Pod 绑定到指定物理核的完整逻辑核组
3. 将该物理核的其余逻辑核标记为 "隔离"，不再调度其他容器
4. 其他非关键 Pod 使用不同物理核的资源

**隔离级别**（通过 Crane CRD 配置，需 kubectl）：

| 级别 | 说明 | 适用场景 |
|------|------|---------|
| 物理核独占 | 关键 Pod 独占整个物理核（两个逻辑核都给同一个 Pod） | 极致低延迟要求 |
| 物理核隔离 | 一个物理核只放同一类 Pod，不混合不同业务 | 多租户隔离 |
| NUMA 感知 | 跨 NUMA node 调度感知，避免跨 NUMA 访存延迟 | 大内存/NUMA 架构 |

### 步骤 3：查看节点规格以了解超线程能力

```bash
tccli tke DescribeClusterInstances \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.InstanceSet[] | {InstanceId, InstanceType, CPU, Memory}'
# expected: 返回各节点 CPU 核数和内存
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
    "RequestId": "d4e5f6a7-b8c9-0d1e-f234-567890abcdef"
}
```

> **注意**：`CPU` 字段显示的是逻辑核数量。若节点支持超线程，物理核数 = 逻辑核数 / 2（如 4 逻辑核 = 2 物理核）。实际超线程拓扑需在节点内通过 `lscpu` 或 `/proc/cpuinfo` 确认（需 VPN/IOA 内网 SSH 登录）。

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

# 3. 确认集群版本支持超线程隔离
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

> 超线程隔离策略是否生效需在内网通过 `kubectl describe node` 查看节点 `tke.cloud.tencent.com/ht-isolation` annotation，或通过 `cat /sys/fs/cgroup/cpuset/.../cpuset.cpus` 确认 cgroup 绑核情况。

## 清理

本页为功能说明与集群状态查看，未创建新资源，无需清理。若需关闭超线程隔离，可在控制台组件管理中修改 QoSAgent 高级配置。

## 排障

### 功能未生效

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 关键 Pod 仍有延迟抖动 | 1. `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName qosagent --region ap-guangzhou` 确认组件状态；2. 通过 kubectl 检查 Pod 是否有 `tke.cloud.tencent.com/ht-isolation` annotation；3. 在节点上通过 `lscpu` 确认超线程已开启 | QoSAgent 未安装、策略未配置或节点超线程未开启 | 安装 QoSAgent，配置超线程隔离策略，确认 BIOS 超线程已启用 |
| `DescribeAddon` 返回空 | 执行 `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 查看已安装组件列表 | QoSAgent 组件未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 套件（包含 QoSAgent） |
| 控制台无超线程隔离配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |

### 性能问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 隔离后非关键 Pod 性能下降 | 检查节点物理核分配情况，可能非关键 Pod 被挤压到少数物理核 | 关键 Pod 独占过多物理核，非关键 Pod 资源不足 | 调整关键 Pod 范围，仅对真正敏感的 Pod 开启独占模式 |
| 节点可用 CPU 明显减少 | 超线程隔离导致部分逻辑核被孤立，不可调度 | 物理核独占模式下资源利用率下降 | 评估是否需要物理核独占级别，可降级为物理核隔离级别 |

## 下一步

- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) -- page_id `79774`
- [CPU Burst](../CPU%20Burst/tccli%20操作.md) -- page_id `79776`
- [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → QoSAgent](https://console.cloud.tencent.com/tke2/cluster?rid=1) 中的高级配置，包含超线程隔离策略设置。控制台提供可视化配置界面，CLI 仅支持查询组件状态和集群信息。
