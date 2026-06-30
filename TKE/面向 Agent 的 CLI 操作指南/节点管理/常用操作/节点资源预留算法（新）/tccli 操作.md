# 节点资源预留算法（新）

> 对照官方：[节点资源预留算法（新）](https://cloud.tencent.com/document/product/457/110495) · page_id `110495` · tccli ≥ 3.1.107 · API 2018-05-25

## 概述

TKE 需要占用节点的一部分资源来运行系统组件（kubelet、kube-proxy、Runtime 等），因此节点的总资源与可分配资源之间存在差异。本文介绍适用于 **Kubernetes 版本 >= 1.30** 的普通节点和原生节点的新资源预留算法。

新算法通过大规模压测获得公式，在大规格机型上为业务提供更多可分配资源，旧算法预留固定值较为保守。

| 维度 | 旧算法 | 新算法 |
|------|--------|--------|
| 适用范围 | K8s < 1.30 | K8s >= 1.30 的普通和原生节点 |
| CPU 预留 | 固定公式 | 渐进式百分比（6%/1%/0.5%/0.25%） |
| Memory 预留 | 固定公式 | min(旧算法, 20MiB × Pod数 + 256MiB) |
| 大规格机型 | 预留资源较多 | 预留资源较少，释放更多给业务 |
| 自定义 | 不支持 | 支持 ExtraArgs.Kubelet 自定义 |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterInstances
#    tke:DescribeClusterEndpoints, tke:ModifyClusterNodePool
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "<ClusterName>",
      "ClusterVersion": "1.32.2",
      "ClusterStatus": "Running"
    }
  ],
  "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 查询目标集群，确认版本 >= 1.30
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterVersion >= 1.30
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "<ClusterName>",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "VpcId": "<VpcId>",
      "ClusterCIDR": "10.0.0.0/16",
      "ServiceCIDR": "10.1.0.0/20"
    }
  ],
  "RequestId": "..."
}
```

```bash
# 5. 查询节点池列表
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 至少返回 1 个节点池
```

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example-1",
      "Name": "<NodePoolName>",
      "LifeState": "normal"
    },
    {
      "NodePoolId": "np-example-2",
      "Name": "<NodePoolName>",
      "LifeState": "normal"
    }
  ],
  "TotalCount": 2,
  "RequestId": "..."
}
```

### 版本与规格选择

- K8s 版本：**必须 >= 1.30**，新算法才生效。低版本集群（v1.28、v1.29）仍使用旧算法。确认版本：
  `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' | jq '.Clusters[0].ClusterVersion'`
- 节点类型：**普通节点**和**原生节点**均可使用新算法。超级节点不适用。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群版本 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池 ExtraArgs 预留配置 | `DescribeClusterNodePoolDetail` | 是 |
| 查看节点实例 | `DescribeClusterInstances` | 是 |
| 查看集群端点状态 | `DescribeClusterEndpoints` | 是 |
| 修改节点池预留参数 | `ModifyClusterNodePool` | 否 |

## 关键字段说明

以下说明 `ModifyClusterNodePool` 中与资源预留相关的参数。完整参数定义见 API 文档。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 已存在的集群 ID，格式 `cls-xxxxxxxx`。`DescribeClusters` 获取 | 集群不存在 → `InvalidParameter.ClusterId` |
| `NodePoolId` | String | 是 | 已存在的节点池 ID，格式 `np-xxxxxxxx`。`DescribeClusterNodePools` 获取 | 节点池不存在 → `InvalidParameter.NodePoolId` |
| `ExtraArgs.Kubelet` | Array of String | 否 | kubelet 启动参数，如 `["kube-reserved=cpu=100m,memory=500Mi"]`。必须是数组，每个元素是字符串。注意：参数格式为 `key=value`，不需要 `--` 前缀（API 会将其传递给 kubelet 时自动添加） | 格式为非数组 → `InvalidParameter`；参数名拼写错误 → kubelet 启动失败；含 `--` 前缀 → `InvalidParameter` |
| `Name` | String | 否 | 节点池新名称，长度 1-60 | — |
| `MaxNodesNum` | Integer | 否 | 最大节点数，>= `MinNodesNum` | 小于 MinNodesNum → `InvalidParameter` |
| `MinNodesNum` | Integer | 否 | 最小节点数，<= `MaxNodesNum` | 大于 MaxNodesNum → `InvalidParameter` |

## 操作步骤

### 查看集群版本和适用范围

#### 选择依据

- **适用范围**：新资源预留算法适用于 Kubernetes 版本 **1.30 及以上**的普通和原生节点。当前集群版本 v1.32.2 >= 1.30，符合适用范围。
- **版本检查方式**：使用 `tccli tke DescribeClusters` 查询 `ClusterVersion` 字段，确认版本号。低版本集群（v1.28、v1.29）仍使用旧算法，如需使用新算法需先升级集群版本。

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterVersion >= 1.30
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2",
            "ClusterType": "MANAGED_CLUSTER",
            "VpcId": "<VpcId>",
            "ClusterCIDR": "10.0.0.0/16",
            "ServiceCIDR": "10.1.0.0/20"
        }
    ],
    "RequestId": "..."
}
```

| 字段 | 含义 | 检查要点 |
|------|------|---------|
| `ClusterVersion` | K8s 版本 | >= 1.30 使用新算法；< 1.30 使用旧算法 |
| `ClusterType` | 集群类型 | 新算法适用于 MANAGED_CLUSTER 和 INDEPENDENT_CLUSTER |
| `ClusterStatus` | 集群状态 | 必须为 `Running` |

### 查看节点池预留配置

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-1",
            "Name": "<NodePoolName>",
            "LifeState": "normal"
        },
        {
            "NodePoolId": "np-example-2",
            "Name": "<NodePoolName>",
            "LifeState": "normal"
        },
        {
            "NodePoolId": "np-example-3",
            "Name": "<NodePoolName>",
            "LifeState": "normal"
        }
    ],
    "TotalCount": 3,
    "RequestId": "..."
}
```

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回 ExtraArgs 和完整节点池配置
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-1",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "NodePoolOs": "ubuntu22.04x86_64",
        "DesiredNodesNum": 1,
        "MaxNodesNum": 1,
        "MinNodesNum": 1,
        "RuntimeConfig": {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.9"
        },
        "ExtraArgs": {
            "Kubelet": []
        },
        "Annotations": [],
        "GPUArgs": {
            "CUDA": {"Name": "", "Version": ""},
            "CUDNN": {"Name": "", "Version": "", "DevName": "", "DocName": ""},
            "CustomDriver": {"Address": ""},
            "Driver": {"Name": "", "Version": ""},
            "MIGEnable": false
        },
        "Tags": [],
        "NodeCountSummary": {
            "ManuallyAdded": {"Joining": 0, "Initializing": 0, "Normal": 0, "Total": 0},
            "AutoscalingAdded": {"Joining": 0, "Initializing": 0, "Normal": 1, "Total": 1}
        }
    },
    "RequestId": "..."
}
```

| 字段 | 含义 | 检查要点 |
|------|------|---------|
| `ExtraArgs.Kubelet` | kubelet 自定义启动参数 | 空数组 `[]` 表示使用默认预留算法；非空则已自定义 |
| `LifeState` | 节点池状态 | 必须为 `normal` |
| `RuntimeConfig.RuntimeType` | 容器运行时 | 新集群统一为 `containerd` |
| `NodeCountSummary` | 节点计数 | 确认当前节点数量 |

### 查看节点实际可分配资源

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Limit 3
# expected: exit 0，返回节点实例列表
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.34"
        },
        {
            "InstanceId": "ins-example-2",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.22"
        }
    ],
    "TotalCount": 2,
    "RequestId": "..."
}
```

```bash
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回集群端点信息
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "172.24.0.12",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetSubnetId": "<SubnetId>",
    "RequestId": "..."
}
```

> **注意**：以下 kubectl 命令需要在集群端点可达的环境下执行。内网端点仅 VPC 内可达，本地需通过 IOA/VPN/专线 或登录同 VPC CVM 才能连接。公网端点若被 CAM 策略拒绝，亦无法使用（详见 [排障](#排障)）。

查看节点 Capacity 与 Allocatable：

```bash
kubectl get nodes -o json | jq '.items[] | {
  name: .metadata.name,
  capacity: .status.capacity,
  allocatable: .status.allocatable
}'
# expected: allocatable 值 < capacity 值（差值即为预留资源）
```

**预期输出**（文本）：

```text
{
  "name": "<NodeName>",
  "capacity": {
    "cpu": "4",
    "memory": "8194304Ki",
    "pods": "64"
  },
  "allocatable": {
    "cpu": "3910m",
    "memory": "7506704Ki",
    "pods": "64"
  }
}
```

查看节点预留详情：

```bash
kubectl describe node <NodeName> | grep -A 10 'Allocatable'
# expected: 显示 allocatable 与 capacity 的对比
```

**预期输出**（文本）：

```text
Allocatable:
  cpu:                3910m
  ephemeral-storage:  95129564Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7506704Ki
  pods:               64
```

### 新资源预留算法详解

#### 节点 CPU 预留规则

由于 CPU 是可压缩资源且存在部分 per-CPU 的内核线程，新算法在大规格机型上尽可能保持一致性，同时提供更多资源给业务使用。算法如下：

- **第一个核心**：预留 **6%**
- **接下来 2 个核心**（第 2-3 核）：各预留 **1%**
- **接下来 2 个核心**（第 4-5 核）：各预留 **0.5%**
- **4 核以上**（第 6 核起）：各预留 **0.25%**

以 8 核节点为例：`6% + 1%×2 + 0.5%×2 + 0.25%×3 = 0.0975` 核（约 100m）。

> **注意**：在小规格机器上创建过多的 Pod 可能导致预留 CPU 过少，影响系统稳定性。以下行为也可能占用系统资源，可通过自定义参数适当调整：
> - 容器打印过多日志
> - exec probe 执行过于频繁
> - 在节点上部署其他系统服务

**新旧算法 CPU 对比（代表性规格）**：

| CPU 核数 | 旧算法预留（核） | 新算法预留（核） | 释放给业务 |
|---------|------------|------------|----------|
| 1 | 0.06 | 0.06 | 相同 |
| 2 | 0.07 | 0.07 | 相同 |
| 4 | 0.09 | 0.09 | 相同 |
| 8 | 0.11 | ~0.10 | 略多 |
| 16 | 0.15 | ~0.12 | 更多 |
| 32 | 0.23 | ~0.16 | 明显更多 |
| 64 | 0.39 | ~0.24 | 明显更多 |
| 128 | 0.71 | ~0.40 | 大幅更多 |

> 完整对比表见 [官方页面](https://cloud.tencent.com/document/product/457/110495)。核心规律：核心数越多，新算法优势越明显。

#### 节点 Memory 预留规则

Memory 是不可压缩资源，设置预留内存时应较为谨慎。经测试，内存预留与 Pod 数量、机器规格紧密相关。新算法为：

```
预留内存 = min(旧算法, 20MiB × Pod 数量 + 256MiB)
```

即：取旧算法和 `20MiB × Pod数量 + 256MiB` 两者的**较小值**。在 Pod 数量较少时，新算法可显著降低预留。

> **注意**：在节点上部署其他服务也可能占用系统预留资源，降低节点稳定性。

**新旧算法 Memory 对比（代表性规格）**：

| 内存 (GiB) | 旧算法 (GiB) | 新算法 16 Pod (GiB) | 新算法 32 Pod (GiB) | 新算法 64 Pod (GiB) | 新算法 128 Pod (GiB) | 新算法 256 Pod (GiB) |
|-----------|------------|------------------|------------------|------------------|-------------------|-------------------|
| 1 | 0.25 | 0.25 | 0.25 | 0.25 | 0.25 | 0.25 |
| 2 | 0.50 | 0.50 | 0.50 | 0.50 | 0.50 | 0.50 |
| 4 | 0.58 | 0.58 | 0.90 | 1.54 | 1.80 | 2.60 |
| 8 | 0.58 | 0.58 | 0.90 | 1.54 | 2.60 | 3.56 |
| 16 | 0.58 | 0.58 | 0.90 | 1.54 | 2.82 | 5.38 |
| 32 | 0.58 | 0.58 | 0.90 | 1.54 | 2.82 | 5.38 |
| 64 | 0.58 | 0.58 | 0.90 | 1.54 | 2.82 | 5.38 |
| 128 | 9.32 | 0.58 | 0.90 | 1.54 | 2.82 | 5.38 |
| 256 | 11.88 | 0.58 | 0.90 | 1.54 | 2.82 | 5.38 |

> 完整对比表见 [官方页面](https://cloud.tencent.com/document/product/457/110495)。核心规律：小规格机型新旧差异不大；大规格机型上 Pod 数少时新算法优势显著（旧算法可能高达 9-12 GiB，新算法仅约 0.6 GiB）。

### 自定义节点资源预留

#### 选择依据

- **何时自定义**：默认预留算法已满足绝大多数场景。仅在以下情况考虑自定义：部署大量系统 DaemonSet、容器日志量极大、exec probe 频繁执行导致额外 CPU 消耗。
- **kube-reserved vs system-reserved**：TKE 不会对 kubelet 组件和系统组件分别进行 cgroup 资源限制，因此预留资源全部设置在 `--kube-reserved` 上即可，单独设置 `--system-reserved` 效果没有区别。
- **kube-reserved 格式**：`--kube-reserved=cpu=<value>,memory=<value>`，其中 cpu 可用 `m`（毫核）或浮点数（核），memory 可用 `Mi`/`Gi`。
- **风险警示**：修改 kubelet 参数会触发节点滚动更新，可能导致业务中断。预留值设置过大（如 CPU >30%）会导致可分配资源严重不足，节点可能 NotReady。生产环境修改前建议先在测试节点池验证。

#### 最小自定义（仅设 kube-reserved）

`reservation-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "ExtraArgs": {
    "Kubelet": ["kube-reserved=cpu=200m,memory=500Mi"]
  }
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://reservation-minimal.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，集群状态 Running | `tccli tke DescribeClusters` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx`，LifeState normal | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

> **⚠️ 警告**：修改节点池 kubelet 参数会触发节点滚动更新，可能导致业务中断。生产环境修改前建议先在测试节点池验证。预留值设置过大（如 CPU >30%）会导致可分配资源严重不足。

节点池更新是异步操作。轮询直到 `LifeState` 恢复为 `normal`，且节点状态正常：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: LifeState: "normal"，ExtraArgs.Kubelet 包含自定义值
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-1",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "ExtraArgs": {
            "Kubelet": ["kube-reserved=cpu=200m,memory=500Mi"]
        }
    },
    "RequestId": "..."
}
```

### 常见问题

**Q1：新旧算法有什么差异？**

旧算法预留公式较为保守，需预留的资源量较大。新算法通过大规模压测获得公式，在大规格机型上提供更多的资源给业务使用。主要差异体现在：CPU 采用渐进式百分比而非固定公式，Memory 取旧算法和 Pod 数量相关公式的最小值。对于 64 核以上的大规格节点，新算法可为业务多释放约 40-50% 的预留资源。

**Q2：设置 kube-reserved 而不设置 system-reserved 是否合理？**

kubelet 按公式 `capacity - kubeReserved - systemReserved - evictionHard` 计算节点可分配资源。但只有开启 `enforce-node-allocatable` 并指定 cgroup 路径时，才会对相应组件做资源限制。TKE 不会对 kubelet 组件和系统组件分别进行 cgroup 资源限制，因此预留资源全部设置在 `--kube-reserved` 上，与分开设置在 `--kube-reserved` 和 `--system-reserved` 上实际效果没有区别。

**Q3：新算法是否支持对低版本（< 1.30）节点生效？**

暂不支持。低版本集群（v1.28、v1.29）仍使用旧算法。存量节点可通过**节点升级**（升级 Kubernetes 版本到 1.30+）来体验新算法。升级前请阅读官方升级指南，评估兼容性风险。确认当前版本：

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].ClusterVersion'
```

## 验证

### 控制面（tccli）

确认节点池 ExtraArgs 已更新：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: ExtraArgs.Kubelet 包含自定义的 --kube-reserved 参数
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-1",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "ExtraArgs": {
            "Kubelet": ["kube-reserved=cpu=200m,memory=500Mi"]
        },
        "NodeCountSummary": {
            "AutoscalingAdded": {"Normal": 1, "Total": 1}
        }
    },
    "RequestId": "..."
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池状态 | `DescribeClusterNodePoolDetail` | `LifeState: "normal"` |
| ExtraArgs 已更新 | 同上，检查 `ExtraArgs.Kubelet` | 包含自定义 `--kube-reserved` 参数 |
| 节点数量 | 同上，检查 `NodeCountSummary` | 节点数 >= `DesiredNodesNum` |
| 集群版本 | `DescribeClusters` | `ClusterVersion >= 1.30` |

### 数据面（kubectl）

> 以下命令需在集群端点可达的环境下执行（通过 IOA/VPN/专线 或同 VPC CVM）。内网端点仅 VPC 内可达，公网端点可能被 CAM 策略拒绝（详见 [排障](#排障)）。

确认节点 allocatable 反映预留变更：

```bash
kubectl describe node <NodeName> | grep -A 10 'Allocatable'
# expected: allocatable.cpu = capacity.cpu - kube-reserved cpu
# expected: allocatable.memory = capacity.memory - kube-reserved memory
```

**预期输出**（文本）：

```text
Allocatable:
  cpu:                3800m
  ephemeral-storage:  95129564Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7689434Ki
  pods:               64
```

确认节点状态正常：

```bash
kubectl get nodes
# expected: 所有节点 STATUS 为 Ready
```

**预期输出**（文本）：

```text
NAME            STATUS   ROLES    AGE   VERSION
<node-name-1>   Ready    <none>   5d    v1.32.2
```

## 清理

本页为概念说明页，默认无需清理操作。

若在"自定义节点资源预留"步骤中修改了 `ExtraArgs.Kubelet`，建议恢复原始配置（将 `Kubelet` 设回 `[]`），以还原默认预留算法：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# 先确认当前 ExtraArgs 值
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-1",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "ExtraArgs": {
            "Kubelet": ["kube-reserved=cpu=200m,memory=500Mi"]
        }
    },
    "RequestId": "..."
}
```

`restore-reservation.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "ExtraArgs": {
    "Kubelet": []
  }
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://restore-reservation.json
# expected: exit 0，ExtraArgs.Kubelet 恢复为空
```

> **⚠️ 警告**：恢复 ExtraArgs 同样会触发节点滚动更新，可能导致业务中断。生产环境操作前务必确认。

恢复后验证：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: ExtraArgs.Kubelet: []
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-1",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "ExtraArgs": {
            "Kubelet": []
        }
    },
    "RequestId": "..."
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameter`（格式错误：非数组） | 检查请求 JSON 中 `ExtraArgs.Kubelet` 是否为数组 | `Kubelet` 不是数组，或元素不是字符串。缺少方括号 `[]` 包裹参数 | 确保 `ExtraArgs.Kubelet` 是字符串数组：`{"Kubelet":["kube-reserved=cpu=200m,memory=500Mi"]}`。不要写成 `{"Kubelet":"kube-reserved=cpu=200m,memory=500Mi"}` |
| `ModifyClusterNodePool` 返回 `InvalidParameter`（含 `--` 前缀） | `tccli tke ModifyClusterNodePool help --detail` 查看 `--ExtraArgs` 参数说明 | kubelet 参数含 `--` 前缀。API 传入格式为 `key=value`，`--` 前缀是 kubelet 命令行语法，API 调用时不需携带 | 去掉所有参数值中的 `--` 前缀：`["kube-reserved=cpu=200m,memory=500Mi"]`。不要写成 `["--kube-reserved=cpu=200m,memory=500Mi"]`。可通过 `tccli tke ModifyClusterNodePool --generate-cli-skeleton` 查看示例参数格式 |
| `ModifyClusterNodePool` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 核对 ClusterId | 集群 ID 格式错误或集群不存在于当前地域 | 用 `DescribeClusters` 确认正确的 `ClusterId` |
| `ModifyClusterNodePool` 返回 `InvalidParameter.NodePoolId` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 核对 NodePoolId | 节点池 ID 格式错误或不属于该集群 | 用 `DescribeClusterNodePools` 确认正确的 `NodePoolId` |
| `ModifyClusterNodePool` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:ModifyClusterNodePool` 权限（环境限制） | 联系主账号授予 `QcloudTKEFullAccess` 或自定义策略包含 `tke:ModifyClusterNodePool` |
| kubectl 返回 `dial tcp: lookup ... : no such host` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看端点配置 | 本地 DNS 无法解析内网域名。内网端点仅 VPC 内可达，本地无 IOA/VPN 无法连接 | 通过 IOA/VPN 接入 VPC，或登录同 VPC CVM 执行 kubectl 命令 |
| kubectl 返回连接超时 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看 `ClusterExternalEndpoint` | 公网端点未开启，或被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝 | 公网端点被组织级 CAM 策略 strategyId:240463971 拒绝，自建安全组也无法绕过。改用内网端点（需 IOA/VPN/专线 或同 VPC CVM） |
| 集群版本 < 1.30 但仍然使用了新算法配置 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 确认版本 | 低版本集群（v1.28、v1.29）不使用新算法，自定义 ExtraArgs 可能不生效 | 升级集群到 >= 1.30，或继续使用旧算法默认预留 |

### 修改 ExtraArgs 后节点异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改 ExtraArgs 后节点变为 `NotReady` | `kubectl describe node <NodeName>` 查看 Conditions；`tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` 查看 InstanceState | 预留值设置过大（如 CPU >30%），导致可分配资源严重不足，或 kubelet 参数格式错误导致启动失败 | 检查 `--kube-reserved` 值是否合理（建议 CPU <30%，memory <50%）。若参数错误，通过 `ModifyClusterNodePool` 修正或恢复为空 `[]`。修复后将触发新一轮滚动更新 |
| 修改 ExtraArgs 后节点一直处于 `Initializing` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 查看 `LifeState` 和 `NodeCountSummary` | 滚动更新进行中，或更新卡住 | 等待滚动更新完成（通常 5-15 分钟/节点）。超过 30 分钟则保留 `Region`、`ClusterId`、`NodePoolId`、`RequestId` → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| 预期用完新算法但实际仍按旧算法预留 | `kubectl get nodes -o json \| jq '.items[].status.allocatable'` 对比 capacity | 集群版本 < 1.30，新算法不生效。或设置 `--kube-reserved` 后覆盖了默认算法 | 升级集群版本到 >= 1.30。如已自定义 `--kube-reserved`，kubelet 直接使用自定义值而非默认公式 |

### 保留 RequestId

涉及 `ModifyClusterNodePool` 的操作如失败，请保留 `RequestId` 以便排查：

```bash
# 示例：执行命令时保存 RequestId
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://reservation-minimal.json
# 从输出中记录 RequestId，供后续工单或日志查询使用
```

## 下一步

- [节点生命周期](https://cloud.tencent.com/document/product/457/110496) — 了解节点状态流转与生命周期管理
- [节点资源预留说明](https://cloud.tencent.com/document/product/457/110497) — 旧版本（< 1.30）资源预留算法参考
- [设置节点的启动脚本](../设置节点的启动脚本/tccli%20操作.md) — 通过 UserScript 自定义节点初始化
- [TKE 集群版本升级](https://cloud.tencent.com/document/product/457/47814) — 将低版本集群升级到 1.30+ 以启用新算法

## 控制台替代

[TKE 控制台 → 节点池详情](https://console.cloud.tencent.com/tke2/nodepool)：在节点池详情页「更多」菜单中可查看和修改 kubelet 自定义参数。
