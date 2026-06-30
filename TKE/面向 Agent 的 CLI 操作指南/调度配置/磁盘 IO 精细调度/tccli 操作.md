# 磁盘 IO 精细调度（tccli）

> 对照官方：[磁盘 IO 精细调度](https://cloud.tencent.com/document/product/457/79781) · page_id `79781`

## 概述

磁盘 IO 精细调度是 QoSAgent 提供的容器级别磁盘 IO 资源管理能力，基于 Linux cgroup blkio 控制器实现 IO 权重分配、IOPS 限制和吞吐量（BPS）限制。该功能可避免 IO 密集型容器（如日志采集、数据库、大数据处理）占用过多磁盘 IO 带宽，影响同节点上的其他容器。

核心价值：
- **IO 隔离**：为不同优先级的容器设置 IO 权重，确保关键业务获得充足的磁盘 IO 资源
- **IOPS/BPS 限流**：防止单个容器的突发 IO 压垮存储后端
- **混部保障**：在线业务和离线任务混部时，保障在线业务的 IO 服务质量

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Linux cgroup blkio 子系统（`blkio.weight`、`blkio.throttle.read_iops_device`、`blkio.throttle.write_iops_device` 等）
- 了解磁盘 IO 性能指标（IOPS、吞吐量、延迟）
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
| 启用/配置 IO 精细调度 | 控制台 → 组件管理 → QoSAgent → IO 隔离策略（CLI 不可直接配置） | 否 |
| 查看容器 IO 使用情况 | `kubectl` / Prometheus 查询（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：确认 QoSAgent 组件已安装且 IO 隔离功能已启用

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json
# expected: Addons 数组非空，Phase = "Active"，RawValues 中 ioIsolationEnabled 为 true
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "qosagent",
            "AddonVersion": "1.2.1",
            "AddonStatus": "Running",
            "RawValues": "{\"nodeSelector\":{\"matchLabels\":{}},\"replicas\":1,\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"512Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"128Mi\"}},\"qosConfig\":{\"cpuBurstEnabled\":true,\"ioIsolationEnabled\":true,\"htIsolationEnabled\":false,\"ioConfig\":{\"diskType\":\"cbs\",\"defaultWeight\":500,\"highPriorityWeight\":800}}}",
            "Phase": "Active"
        }
    ],
    "RequestId": "e5f6a7b8-c9d0-1234-ef56-7890abcdef12"
}
```

可通过 jq 快速查看 IO 隔离配置：

```bash
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0].RawValues | fromjson | .qosConfig | {ioIsolationEnabled, ioConfig}'
# expected: ioIsolationEnabled = true
```

### 步骤 2：理解磁盘 IO 精细调度工作原理

**cgroup blkio 控制器架构**：

```
容器 A（高优先级，IO 权重 800）
容器 B（中优先级，IO 权重 500）
容器 C（低优先级，IO 权重 200）

       ┌─────────┐  ┌─────────┐  ┌─────────┐
       │  Pod A  │  │  Pod B  │  │  Pod C  │
       └────┬────┘  └────┬────┘  └────┬────┘
            │            │            │
       ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
       │ blkio:  │  │ blkio:  │  │ blkio:  │
       │ weight  │  │ weight  │  │ weight  │
       │  = 800  │  │  = 500  │  │  = 200  │
       └────┬────┘  └────┬────┘  └────┬────┘
            │            │            │
            ▼            ▼            ▼
   ┌─────────────────────────────────────────┐
   │          Linux CFQ/bfq 调度器            │
   │   按 io 权重比例分配磁盘 IO 时间片        │
   │   800:500:200 = GPU:CPU:IO = 53%:33%:13% │
   └───────────────────┬─────────────────────┘
                       │
                       ▼
                 ┌──────────┐
                 │  磁盘设备  │ (/dev/vdb, /dev/vdc ...)
                 └──────────┘
```

**三种 IO 控制能力**：

| 能力 | cgroup 参数 | 说明 | 典型场景 |
|------|-----------|------|---------|
| **IO 权重分配** | `blkio.weight`（100 ~ 1000） | 按权重比例分配 IO 带宽，多容器竞争 IO 时生效 | 多业务共享磁盘，按优先级分配 IO |
| **IOPS 限制** | `blkio.throttle.read_iops_device` / `write_iops_device` | 对指定块设备限制每秒读写次数 | 防止日志采集容器压垮磁盘 |
| **BPS 限制** | `blkio.throttle.read_bps_device` / `write_bps_device` | 对指定块设备限制每秒读写字节数 | 限制大数据处理任务的 IO 吞吐 |

**关键配置参数**（通过 QoSAgent 策略配置，Crane CRD）：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `ioIsolationEnabled` | IO 隔离功能总开关 | `true` / `false` |
| `diskType` | 目标磁盘类型，`cbs`（云硬盘）或 `local`（本地盘） | `cbs` |
| `defaultWeight` | 默认容器 IO 权重（100 ~ 1000） | `500` |
| `highPriorityWeight` | 高优先级容器 IO 权重 | `800` |
| `maxIOPS` | 单容器最大 IOPS（可选，0 表示不限） | `0`（不限）或 `1000` |
| `maxThroughput` | 单容器最大吞吐量（MB/s，可选） | `0`（不限）或 `50` |

### 步骤 3：查看节点列表（了解磁盘类型和规格）

```bash
tccli tke DescribeClusterInstances \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.InstanceSet[] | {InstanceId, InstanceType, CPU, Memory, InstanceState}'
# expected: 返回各节点信息
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
    "RequestId": "f6a7b8c9-d0e1-2345-f678-90abcdef1234"
}
```

> S5.LARGE8 实例通常挂载 CBS 云硬盘作为数据盘。IO 加权策略的 `diskType` 应与之匹配。实际磁盘设备名（如 `/dev/vdb`）需通过 SSH 登录节点确认（VPN/IOA 内网）。

### 步骤 4：适用场景分析

| 场景 | IO 问题 | IO 隔离策略 | 配置建议 |
|------|--------|-----------|---------|
| 日志采集 + 在线业务 | fluentd 大量写日志导致磁盘 IO 饱和，在线业务读写数据库受阻 | 限制日志采集容器的 IOPS 和 BPS，给在线业务高 IO 权重 | `maxIOPS=500`, `highPriorityWeight=800` |
| 大数据 ETL + 在线服务 | Spark 任务大量读写 HDFS/COS 占用全部磁盘 IO | 限制 ETL Job 的 BPS，保障在线服务的 IO 响应 | `maxThroughput=50`, `defaultWeight=300` |
| 数据库混部 | 多个 MySQL/PostgreSQL 实例共享节点磁盘 | 按数据库优先级分配不同 IO 权重 | `defaultWeight=500`，核心库设为 800 |
| AI 训练 + 模型推理 | 训练任务读写 checkpoint 影响推理服务加载模型 | 限制训练任务 IOPS，保障推理服务 IO 权重 | `maxIOPS=1000`, `highPriorityWeight=800` |

## 验证

```bash
# 1. 确认集群状态为 Running
tccli tke DescribeClusterStatus \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.ClusterStatusSet[0].ClusterState'
# expected: "Running"

# 2. 确认 QoSAgent 组件正常运行且 IO 隔离已启用
tccli tke DescribeAddon \
    --ClusterId cls-xxxxxxxx \
    --AddonName qosagent \
    --region ap-guangzhou --output json \
    | jq '.Addons[0] | {Phase, AddonStatus, ioIsolation: (.RawValues | fromjson | .qosConfig.ioIsolationEnabled)}'
# expected: Phase = "Active"，AddonStatus = "Running"，ioIsolation = true

# 3. 确认集群版本支持 IO 精细调度
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

> IO 隔离策略是否生效需在内网通过以下方式验证：
> - `cat /sys/fs/cgroup/blkio/kubepods/.../blkio.weight` 查看 IO 权重值
> - 使用 `fio` 工具在内网节点上对容器做 IO 压测，观察 IOPS/BPS 是否受限于配置值
> - 通过 Prometheus + Grafana 查看容器级别的 IO 指标

## 清理

本页为功能说明与集群状态查询，未创建新资源，无需清理。若需关闭 IO 精细调度，可在控制台组件管理中修改 QoSAgent 的 IO 隔离策略，将 `ioIsolationEnabled` 设为 `false`。

**注意**：关闭 IO 隔离后，容器的 cgroup blkio 参数可能会被设为默认值（weight=500, no throttle），已在运行的 Pod 不受影响。

## 排障

### 功能未生效

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| IO 权重/限流未生效 | 1. `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName qosagent --region ap-guangzhou` 检查 `ioIsolationEnabled`；2. 在节点上执行 `cat /sys/fs/cgroup/blkio/.../blkio.weight` 确认 cgroup 参数已写入（需 VPN/IOA）；3. 确认磁盘调度器为 CFQ 或 bfq（`cat /sys/block/vdb/queue/scheduler`） | QoSAgent 未安装、IO 隔离未启用或磁盘调度器不支持 blkio 权重分配 | 安装 QoSAgent 并启用 IO 隔离，确认磁盘调度器为 CFQ/bfq |
| IOPS 限制不生效 | 同上，检查 `blkio.throttle.read_iops_device` 和 `write_iops_device` 参数 | QoSAgent 未写入 cgroup throttle 参数 | 检查 QoSAgent 日志（kubectl logs），确认策略配置被正确解析 |
| BPS 限制不生效 | 同上，检查 `blkio.throttle.read_bps_device` 和 `write_bps_device` 参数 | 同上 | 同上 |

### 配置问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 控制台无 IO 隔离配置入口 | 确认集群类型和版本 | 非原生节点集群或版本 < 1.20 不支持 | 使用原生节点集群，升级集群版本 |
| 权重分配不均衡 | 检查 `blkio.weight` 值，确认磁盘调度器为 CFQ（非 none 模式） | 磁盘调度器为 none（NVMe SSD 常见），不支持 blkio weight | 对于 NVMe SSD，使用 IOPS/BPS 限流替代权重分配 |
| IO 隔离后在线业务性能下降 | 检查是否误把在线业务容器也限制了 IOPS/BPS | 策略配置错误，限流了不该限流的容器 | 修正 QoSAgent 策略，仅限制 IO 重型容器 |

### 兼容性问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| NVMe 本地盘 IO 权重无效 | 检查磁盘调度器 `/sys/block/nvme0n1/queue/scheduler` | NVMe 设备默认使用 `none` 调度器，不支持 blkio.weight | 使用 IOPS/BPS 限流替代，或切换为 CFQ（需内核支持） |
| `DescribeAddon` 返回空 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` 列出全部组件 | QoSAgent 未安装 | 在 TKE 控制台 → 组件管理 安装 Crane 套件（勾选 QoSAgent） |

## 下一步

- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) -- page_id `79774`
- [CPU Burst](../CPU%20Burst/tccli%20操作.md) -- page_id `79776`
- [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) -- page_id `79777`
- [网络精细调度](../../调度配置/) -- page_id `--`
- [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 组件管理 → QoSAgent → IO 隔离策略](https://console.cloud.tencent.com/tke2/cluster?rid=1) 提供可视化配置磁盘 IO 权重、IOPS/BPS 限流参数。控制台支持按容器标签（如 `app=db`、`app=log`）指定不同的 IO 策略。CLI 可通过 `DescribeAddon` 查询组件状态和策略开关，但策略的精细配置需通过控制台或直接修改 Crane CRD。
