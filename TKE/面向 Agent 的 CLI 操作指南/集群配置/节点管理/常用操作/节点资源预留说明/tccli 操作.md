# 查询节点资源预留与可分配资源

> 对照官方：[节点资源预留说明](https://cloud.tencent.com/document/product/457/76057) · page_id `76057`

## 概述

TKE 节点的 `Allocatable`（可分配资源）= `Capacity`（总量）− 系统预留 − kube-reserved − cluster-autoscaler 额外预留。系统按 CPU/内存分段公式为 kubelet 和系统组件预留资源，cluster-autoscaler 另有额外的内存预留。本页说明如何用 tccli 查询节点底层实例规格、用 kubectl 查看 `Capacity`/`Allocatable`，并对照官方公式核对差值。

| 查询维度 | 工具 | 关键字段 |
|---------|------|---------|
| 节点可分配资源（实时） | kubectl | `status.capacity` / `status.allocatable` |
| 节点底层实例规格 | tccli (CVM) | `CPU` / `Memory` |
| 节点所属集群与角色 | tccli (TKE) | `InstanceRole` / `InstanceId` |
| 预留公式参数 | 官方分段表（本页） | 见下方「CPU 预留规则」「内存预留规则」 |

> **kubectl 数据面可达性**：`kubectl describe node` 需 APIServer 端点可达。公网端点可能受 CAM 策略限制，内网端点需 VPN/IOA。若 kubectl 不可达，可用 tccli 查询 CVM 实例规格并按本页公式推算预留量。

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
#    cvm:DescribeInstances
# 验证：执行 DescribeClusterInstances 确认权限
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet 列表

# 4. 检查 kubectl（可选，仅数据面验证需要）
kubectl version --client
# expected: Client Version v1.28.x+
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 资源检查

```bash
# 5. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus "Running"

# 6. 查询集群节点及底层实例 ID
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: 返回 InstanceSet，每条含 InstanceId

# 7. 通过 CVM API 查询节点实例规格（CPU/内存）
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: 返回 CPU（核数）和 Memory（MB）
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
| `<InstanceId>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 的 `InstanceId` 字段 |
| `<NodeName>` | kubernetes 节点名 | 非 CVM ID | `kubectl get nodes` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点列表 | `DescribeClusterInstances --ClusterId` | 是 |
| 查看节点底层实例规格 | `cvm DescribeInstances --InstanceIds` | 是 |
| 查看节点可分配资源（数据面） | `kubectl describe node <NodeName>` | 是 |

## 操作步骤

### 步骤 1：查询节点底层实例规格（控制面）

通过 TKE API 获取集群节点的 CVM 实例 ID，再通过 CVM API 查询 CPU/内存规格，用于套用预留公式。

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet，每条含 InstanceId
```

**预期输出**（截取关键字段）：

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "InstanceName": "node-example-1"
        },
        {
            "InstanceId": "ins-example-2",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "InstanceName": "node-example-2"
        }
    ]
}
```

用返回的 `InstanceId` 查询 CVM 实例规格：

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

> `CPU` 单位为核（核数），`Memory` 单位为 MB。本例 8 核 16GB 节点，套用下方公式。

### 步骤 2：查看节点可分配资源（数据面，须 VPN/IOA）

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
  cpu:                7800m
  memory:             15723344Ki
  pods:               64
```

只提取 Capacity 和 Allocatable 段（Linux/macOS 用 grep，Windows 用 findstr）：

```bash
kubectl describe node <NodeName>
# expected: 输出含 Capacity 和 Allocatable 两段
```

```text
NAME  STATUS  AGE
...
```

> 如需在 Linux/macOS 下过滤，可在终端执行 `kubectl describe node <NodeName>` 后人工核对 `Capacity` 与 `Allocatable` 两段；不要在 tccli 命令中使用管道符 `|`。

### 步骤 3：对照官方预留公式核对差值

#### 节点 CPU 预留规则

| 节点 CPU 核数 | kubelet+系统预留 CPU |
|--------------|---------------------|
| 1 ≤ CPU ≤ 4 | 固定 0.1 核 |
| 4 < CPU ≤ 64 | 0.1 + 超过 4 核部分的 2.5% |
| 64 < CPU ≤ 128 | 分段累加，见官方文档 |
| CPU > 128 | 分段累加，见官方文档 |

**示例**（8 核节点）：预留 = 0.1 + (8 − 4) × 2.5% = 0.1 + 0.1 = 0.2 核。`Allocatable.cpu` ≈ 7.8 核。

#### 节点内存预留规则

| 节点内存 | kubelet+系统预留内存 |
|---------|---------------------|
| ≤ 4 GB | 25% 内存 |
| 4 GB < 内存 ≤ 64 GB | 1 GB + 超过 4 GB 部分的 12.5% |
| > 64 GB | 分段累加，见官方文档 |

**示例**（16 GB 节点，即 16384 MB）：预留 = 1 GB + (16 − 4) × 12.5% = 1 + 1.5 = 2.5 GB。`Allocatable.memory` ≈ 13.5 GB。

#### cluster-autoscaler 额外内存预留

当集群启用 cluster-autoscaler 时，节点额外预留内存：

| 节点内存范围 | 额外预留内存 |
|-------------|-------------|
| 1800 MB ≤ 内存 ≤ 64 GB | 256 MB |
| 64 GB < 内存 ≤ 128 GB | 512 MB |
| 内存 > 128 GB | 768 MB |

**示例**（16 GB 节点）：额外预留 256 MB。总预留 = 2.5 GB + 256 MB ≈ 2.75 GB。

### 步骤 4：核对差值

对比 `Capacity` 与 `Allocatable` 的差值，应等于「系统预留 + kube-reserved + cluster-autoscaler 额外预留」之和：

```bash
kubectl describe node <NodeName>
# expected: Allocatable < Capacity，差值符合官方公式量级
```

```text
NAME  STATUS  AGE
...
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| CPU 预留 | `kubectl describe node <NodeName>` 检查 `Capacity.cpu` 与 `Allocatable.cpu` | 差值符合 CPU 预留规则（如 8 核差约 0.2 核） |
| 内存预留 | 同上，检查 `Capacity.memory` 与 `Allocatable.memory` | 差值符合内存预留规则 + cluster-autoscaler 额外预留（如 16 GB 差约 2.75 GB） |
| Pod 上限 | 同上，检查 `Capacity.pods` 与 `Allocatable.pods` | 通常相等，pods 预留为 0 |

> 若差值明显大于公式计算值，可能存在自定义 kubelet 预留参数（见 [节点资源预留算法（新）](../节点资源预留算法（新）/tccli%20操作.md)）。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet

tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回 CPU 和 Memory 字段
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 数据面（须 VPN/IOA）

```bash
kubectl describe node <NodeName>
# expected: 输出含 Capacity 和 Allocatable 两段，Allocatable < Capacity
```

```text
NAME  STATUS  AGE
...
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 实例规格 | `cvm DescribeInstances` 检查 `CPU`/`Memory` | 与控制台节点规格一致 |
| 可分配资源 | `kubectl describe node` 检查 `Allocatable` | `Allocatable` < `Capacity`，差值符合官方公式 |
| 公式核对 | 用 `CPU`/`Memory` 套用步骤 3 公式 | 计算值与 `Allocatable` 差值误差 < 5% |

## 清理

只读操作，无资源清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回 `InvalidParameter` | `tccli configure list` 检查 region | `region` 与集群地域不一致，或 `ClusterId` 格式错误 | 用 `tccli tke DescribeClusters --region <Region>` 确认集群 ID 与地域 |
| `cvm DescribeInstances` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cvm:DescribeInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudCVMDataAccessOnly` 或最小权限 `cvm:DescribeInstances` |
| `kubectl describe node` 返回 `Unable to connect to the server` | `kubectl cluster-info` 检查 APIServer 可达性 | 公网端点受 CAM 策略拒绝或内网端点未连通 | 通过 VPN/IOA 连接内网端点；或用 tccli 查询 CVM 规格按公式推算 |
| `kubectl describe node` 返回 `NotFound` | `kubectl get nodes` 确认节点名 | `<NodeName>` 拼写错误或节点已移出 | 从 `kubectl get nodes` 输出复制节点名 |

### 查询成功但结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Allocatable.cpu` 差值明显大于公式计算值 | `kubectl describe node <NodeName>` 查看 `Allocatable`；检查节点是否启用 cluster-autoscaler | 节点启用了 cluster-autoscaler 额外内存预留，或节点池设置了自定义 kube-reserved 参数 | 对照本页公式 + cluster-autoscaler 额外预留计算总预留；自定义参数见 [节点资源预留算法（新）](../节点资源预留算法（新）/tccli%20操作.md) |
| `Allocatable.memory` 差值为负或为 0 | `kubectl describe node <NodeName>` 查看 `Capacity.memory` 与 `Allocatable.memory` | 节点内存过小（如 1 GB），预留后无可分配内存，或 kubelet 配置异常 | 升级 CVM 实例规格；检查节点池 `ExtraArgs.Kubelet` 是否含异常 `--system-reserved`/`--kube-reserved` 参数 |
| `Capacity.memory` 与 CVM 规格不一致 | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 对比 `Memory` 与 `Capacity.memory` | 单位换算误差（CVM `Memory` 单位为 MB，kubectl `Capacity.memory` 单位为 KiB，1 MB ≈ 976 KiB） | 按 1 MB = 1024 × 1024 字节、1 KiB = 1024 字节换算；1 MB ≈ 976 KiB |
| Pod 调度失败 `Insufficient memory` / `Insufficient cpu` | `kubectl describe node <NodeName>` 查看 `Allocatable` 与已分配量 | 按 `Capacity` 而非 `Allocatable` 估算可用资源导致超卖 | 用 `Allocatable` 减去 `Allocated resources` 计算真实可用；必要时扩容节点或封锁后升级 CVM 规格 |
| `cvm DescribeInstances` 返回 `Memory` 与控制台规格不符 | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 检查 `InstanceType` | 实例规格与预期不符（如创建时选错规格） | 控制台确认实例规格；如需变更，参考 [CVM 调整实例配置](https://cloud.tencent.com/document/product/213/2178) |

## 下一步

- [节点资源预留算法（新）](https://cloud.tencent.com/document/product/457/110495) — 新版资源预留算法与 kubelet 参数配置
- [自定义控制面组件参数](https://cloud.tencent.com/document/product/457/47775) — 集群级 kubelet 参数调整
- [节点生命周期](https://cloud.tencent.com/document/product/457/32202) — 节点状态流转与异常排查
- [新增节点](https://cloud.tencent.com/document/product/457/32200) — 向集群添加工作节点并验证规格

## 控制台替代

[容器服务控制台 → 节点管理 → 节点详情 → 资源信息](https://console.cloud.tencent.com/tke2/cluster?tab=instance)
