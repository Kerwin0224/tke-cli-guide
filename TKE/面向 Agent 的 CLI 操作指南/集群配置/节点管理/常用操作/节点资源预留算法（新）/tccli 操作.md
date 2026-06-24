# 查询节点资源预留算法（新）

> 对照官方：[节点资源预留算法（新）](https://cloud.tencent.com/document/product/457/110495) · page_id `110495`

## 概述

TKE 新版资源预留算法调整了 kubelet 为系统组件、kube-reserved 预留资源的方式，影响节点的 `Allocatable`（可分配资源）。新版算法支持通过节点池 `ExtraArgs.Kubelet` 和集群级 `ExtraArgs` 自定义预留参数，而非仅使用默认分段公式。本页说明如何用 tccli 查询节点池和集群的 kubelet 预留参数，用 kubectl 核对 `Allocatable`，并对照新旧算法差异。

| 维度 | 旧版算法（默认） | 新版算法（自定义） |
|------|------------------|-------------------|
| 预留方式 | 固定分段公式（CPU/内存分段累加） | 节点池 `ExtraArgs.Kubelet` 自定义 `--system-reserved`/`--kube-reserved` |
| 配置层级 | 集群级，不可覆盖 | 节点池级覆盖集群级 |
| 查询方式 | 按公式推算（见 [节点资源预留说明](../节点资源预留说明/tccli%20操作.md)） | `DescribeClusterNodePoolDetail` → `ExtraArgs` |
| cluster-autoscaler 额外预留 | 256/512/768 MB（按内存分段） | 同旧版，额外叠加 |
| 适用场景 | 无需自定义预留的常规集群 | 需精确控制节点可分配资源的场景 |

> **kubectl 数据面可达性**：`kubectl describe node` 需 APIServer 端点可达。公网端点可能受 CAM 策略限制，内网端点需 VPN/IOA。若 kubectl 不可达，用 tccli 查询节点池 `ExtraArgs` 推算预留配置。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterExtraArgs
#    cvm:DescribeInstances
# 验证：执行 DescribeClusterNodePoolDetail 确认权限
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回 NodePool 详情（含 ExtraArgs）

# 验证集群级参数查询权限
tccli tke DescribeClusterExtraArgs --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回集群级 ExtraArgs

# 4. 检查 kubectl（可选，仅数据面验证需要）
kubectl version --client
# expected: Client Version v1.28.x+
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 5. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus "Running"

# 6. 查询集群节点池列表（获取 NodePoolId）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回 NodePoolSet，每条含 NodePoolId

# 7. 查询节点底层实例规格（用于核对 Allocatable）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: 返回 InstanceSet，每条含 InstanceId
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou`、`ap-beijing` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --ClusterId` |
| `<InstanceId>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 的 `InstanceId` 字段 |
| `<NodeName>` | kubernetes 节点名 | 非 CVM ID | `kubectl get nodes` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池列表 | `DescribeClusterNodePools --ClusterId` | 是 |
| 查看节点池 kubelet 参数 | `DescribeClusterNodePoolDetail --ClusterId --NodePoolId` | 是 |
| 查看集群级 kubelet 参数 | `DescribeClusterExtraArgs --ClusterId` | 是 |
| 查看节点可分配资源（数据面） | `kubectl describe node <NodeName>` | 是 |
| 查看节点底层实例规格 | `cvm DescribeInstances --InstanceIds` | 是 |

## 操作步骤

### 步骤 1：查询集群节点池列表

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 NodePoolSet 数组
```

**预期输出**（截取关键字段）：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-1",
            "Name": "worker-pool-default",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "LaunchTemplateId": "hlt-example",
            "Count": 3
        },
        {
            "NodePoolId": "np-example-2",
            "Name": "worker-pool-custom-reserved",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "LaunchTemplateId": "hlt-example",
            "Count": 2
        }
    ],
    "TotalCount": 2
}
```

> `LifeState: normal` 表示节点池正常。记录需要查询预留参数的 `NodePoolId`。

### 步骤 2：查询节点池 kubelet 预留参数

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回 NodePool 详情，含 ExtraArgs
```

**预期输出**（截取关键字段）：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-2",
        "Name": "worker-pool-custom-reserved",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "LaunchTemplateId": "hlt-example",
        "Count": 2,
        "ExtraArgs": {
            "Kubelet": "--system-reserved=cpu=500m,memory=2Gi --kube-reserved=cpu=200m,memory=1Gi --eviction-hard=memory.available<1Gi,nodefs.available<10%"
        }
    }
}
```

字段解读：

| 字段 | 含义 | 影响预留量 |
|------|------|-----------|
| `ExtraArgs.Kubelet` | 节点池级 kubelet 自定义参数 | 直接覆盖默认分段公式 |
| `--system-reserved` | 为系统组件预留的 CPU/内存 | 从 `Capacity` 扣减 |
| `--kube-reserved` | 为 kubelet 自身预留的 CPU/内存 | 从 `Capacity` 扣减 |
| `--eviction-hard` | 硬驱逐阈值，触发节点压力驱逐 | 影响 `Allocatable` 计算阈值 |

> 若 `ExtraArgs.Kubelet` 为空字符串，表示该节点池使用集群级默认参数（见步骤 3）或旧版分段公式。

### 步骤 3：查询集群级 kubelet 预留参数

```bash
tccli tke DescribeClusterExtraArgs --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回集群级 ExtraArgs
```

**预期输出**（截取关键字段）：

```json
{
    "ClusterExtraArgs": {
        "Kubelet": "--system-reserved=cpu=300m,memory=1Gi --kube-reserved=cpu=100m,memory=512Mi",
        "APIServer": "",
        "ControllerManager": "",
        "Scheduler": "",
        "Etcd": ""
    }
}
```

> 集群级 `ExtraArgs.Kubelet` 是全集群默认值。若节点池级 `ExtraArgs.Kubelet` 非空，则节点池级覆盖集群级。若两者均为空，则使用旧版分段公式（见 [节点资源预留说明](../节点资源预留说明/tccli%20操作.md)）。

### 步骤 4：数据面核对 Allocatable（须 VPN/IOA）

在可访问 APIServer 的环境中，用 kubectl 核对节点的实际可分配资源：

```bash
kubectl describe node <NodeName>
# expected: exit 0，输出含 Capacity 和 Allocatable 段
```

**预期输出**（截取关键字段）：

```text
Capacity:
  cpu:                8
  memory:             16266320Ki
  pods:               64
Allocatable:
  cpu:                7200m
  memory:             14250000Ki
  pods:               64
```

> 本例节点池设置了 `--system-reserved=cpu=500m,memory=2Gi --kube-reserved=cpu=200m,memory=1Gi`，总预留 CPU = 700m，总预留内存 = 3 Gi。`Allocatable.cpu` ≈ 8 − 0.7 = 7.3 核（7200m），`Allocatable.memory` ≈ 16 − 3 = 13 Gi。

### 步骤 5：关联 CVM 实例规格核对

通过 CVM API 查询节点底层实例规格，用于验证 `Capacity` 是否与 CVM 规格一致：

```bash
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回 CPU 和 Memory 字段
```

**预期输出**（截取关键字段）：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceName": "node-example-1",
            "CPU": 8,
            "Memory": 16384
        }
    ]
}
```

> `CPU` 单位为核，`Memory` 单位为 MB。本例 8 核 16 GB 节点，`Capacity.cpu` 应为 8，`Capacity.memory` 应约为 16384 MB（kubectl 显示为 KiB，需换算）。

## 验证

### 控制面（tccli）

```bash
# 验证节点池 kubelet 参数
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，ExtraArgs.Kubelet 含预留参数或为空

# 验证集群级 kubelet 参数
tccli tke DescribeClusterExtraArgs --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，ClusterExtraArgs.Kubelet 含预留参数或为空

# 验证节点底层规格
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，CPU 和 Memory 与 Capacity 一致
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 数据面（须 VPN/IOA）

```bash
kubectl describe node <NodeName>
# expected: Allocatable < Capacity，差值符合 ExtraArgs 预留参数
```

```text
NAME  STATUS  AGE
...
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 节点池参数 | `DescribeClusterNodePoolDetail` 检查 `ExtraArgs.Kubelet` | 含 `--system-reserved`/`--kube-reserved` 或为空 |
| 集群级参数 | `DescribeClusterExtraArgs` 检查 `ClusterExtraArgs.Kubelet` | 含预留参数或为空 |
| 参数优先级 | 对比节点池与集群级 `ExtraArgs` | 节点池非空时覆盖集群级 |
| 可分配资源 | `kubectl describe node` 检查 `Allocatable` | `Allocatable` < `Capacity`，差值 = system-reserved + kube-reserved |
| 实例规格 | `cvm DescribeInstances` 检查 `CPU`/`Memory` | 与 `Capacity` 一致（单位换算后） |

## 清理

只读操作，无资源清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认 NodePoolId | `NodePoolId` 不属于该集群或拼写错误 | 从 `DescribeClusterNodePools` 输出复制 `NodePoolId`，格式为 `np-xxxxxxxx` |
| `DescribeClusterNodePoolDetail` 返回 `InvalidParameter` | `tccli configure list` 检查 region | `region` 与集群地域不一致，或 `ClusterId` 格式错误 | 用 `tccli tke DescribeClusters --region <Region>` 确认集群 ID 与地域 |
| `DescribeClusterExtraArgs` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusterExtraArgs` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusterExtraArgs` |
| `cvm DescribeInstances` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cvm:DescribeInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudCVMDataAccessOnly` 或最小权限 `cvm:DescribeInstances` |
| `kubectl describe node` 返回 `Unable to connect to the server` | `kubectl cluster-info` 检查 APIServer 可达性 | 公网端点受 CAM 策略拒绝或内网端点未连通 | 通过 VPN/IOA 连接内网端点；或用 tccli 查询 `ExtraArgs` 推算预留参数 |

### 查询成功但结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ExtraArgs.Kubelet` 为空字符串 | `DescribeClusterExtraArgs --ClusterId <ClusterId>` 查集群级参数 | 节点池和集群均未设置自定义 kubelet 参数，使用旧版分段公式 | 对照 [节点资源预留说明](../节点资源预留说明/tccli%20操作.md) 的分段公式推算预留量 |
| `Allocatable` 差值与 `ExtraArgs.Kubelet` 不匹配 | `kubectl describe node <NodeName>` 查看 `Capacity`/`Allocatable`；`cvm DescribeInstances` 查 CVM 规格 | 1) cluster-autoscaler 额外内存预留叠加；2) 单位换算误差（KiB vs MB）；3) 节点池参数未生效 | 按 1 MB ≈ 976 KiB 换算；叠加 cluster-autoscaler 256/512/768 MB 额外预留；确认节点属于目标节点池 |
| `Allocatable.memory` 为 0 或负值 | `kubectl describe node <NodeName>` 查看 `Allocatable.memory` | `--system-reserved` + `--kube-reserved` 设置过大，超过节点总内存 | 调整节点池 `ExtraArgs.Kubelet` 中的预留值，确保总预留 < 节点内存的 50% |
| 节点池级参数未覆盖集群级 | `DescribeClusterNodePoolDetail` 检查 `ExtraArgs.Kubelet` 是否非空 | 节点池参数为空，回退到集群级默认值 | 在节点池设置 `ExtraArgs.Kubelet`（需通过控制台或 `ModifyClusterNodePool` API） |
| Pod 调度失败 `Insufficient memory` 但 `Allocatable` 仍有余量 | `kubectl describe node <NodeName>` 查看 `Allocated resources`；`kubectl get pods --field-selector spec.nodeName=<NodeName>` 查看已分配 | `--eviction-hard` 阈值过高，导致 `Allocatable` 中扣除了驱逐阈值但 `Allocated resources` 未显示 | 调整 `--eviction-hard` 阈值；或扩容节点池 |

## 下一步

- [节点资源预留说明](https://cloud.tencent.com/document/product/457/76057) — 旧版分段公式与 cluster-autoscaler 额外预留
- [自定义控制面组件参数](https://cloud.tencent.com/document/product/457/47775) — 集群级 kubelet 参数调整
- [节点生命周期](https://cloud.tencent.com/document/product/457/32202) — 节点状态流转与异常排查
- [新增节点](https://cloud.tencent.com/document/product/457/32200) — 向集群添加工作节点并验证规格
- [CVM 调整实例配置](https://cloud.tencent.com/document/product/213/2178) — 变更 CVM 实例规格

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 节点池详情](https://console.cloud.tencent.com/tke2/cluster?tab=nodepool)
