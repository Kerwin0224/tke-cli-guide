# 原生节点概述

> 对照官方：[原生节点概述](https://cloud.tencent.com/document/product/457/78197) · page_id `78197`

## 概述

原生节点是 TKE 推出的增强型节点类型，在普通节点能力之上新增声明式基础设施管理、Pod 原地升降配、专用调度器、故障自愈等功能。原生节点通过**原生节点池**统一管理，创建方式包括控制台、Kubernetes API 和云 API（`CreateClusterNodePool` 指定 `Type="Native"`）。

方案选择：
- **原生节点**：需要声明式管理、故障自愈、原地升降配、精细调度 → 选此
- **普通节点**：仅需基础容器运行，用户自行管理节点生命周期 → 选此
- **超级节点**：无节点运维、按 Pod 计费、弹性伸缩 → 选此

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
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
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
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"

# 5. 查看集群的 K8s 版本（原生节点对版本有要求）
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterVersion ≥ 1.16（推荐 ≥ 1.28）
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 创建原生节点池 | `CreateClusterNodePool`（Type="Native"） | 否 |

## 操作步骤

### 原生节点与普通节点对比

| 维度 | 原生节点 | 普通节点 |
|------|----------|----------|
| 管理模式 | 节点管家辅助管理 | 用户全自助管理 |
| 声明式管理 | 支持（通过 Annotations） | 不支持 |
| Pod 原地升降配 | 支持 | 不支持 |
| 故障自愈 | 支持（可配置自愈规则） | 不支持（需手动干预） |
| 内存压缩 | 支持 | 不支持 |
| 节点自动伸缩 | 支持 | 支持（HPA/HPC） |
| 计费模式 | 按量计费 / 包年包月 | 按量计费 / 包年包月 |
| 运行时 | 仅 containerd | containerd / docker（取决于版本） |
| 节点类型标识 | `Type="Native"` | `Type="Regular"` |

### 计费模式

原生节点与普通节点采用相同的 CVM 计费模式：

- **按量计费**：按小时结算，适合短期、测试环境
- **包年包月**：预付费，适合长期、生产环境

节点池级别的计费由创建时所选的 `InstanceChargeType` 参数决定。

### 通过 API 识别节点池类型

创建节点池时指定 `Type="Native"` 即创建原生节点池。查询现有节点池，通过 `NodePoolType` 字段区分：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中包含 NodePoolType 字段
```

预期输出：

```json
{
    "TotalCount": 3,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-native-1",
            "Name": "native-pool-prod",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 5
        },
        {
            "NodePoolId": "np-example-native-2",
            "Name": "native-pool-test",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 2
        },
        {
            "NodePoolId": "np-example-regular",
            "Name": "regular-pool",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular",
            "DesiredNodesNum": 3
        }
    ]
}
```

### 查看原生节点池详情

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，NodePool.Type 为 "Native"
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native-1",
        "Name": "native-pool-prod",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "NodeCountSummary": {
            "ManuallyAdded": {
                "Joining": 0,
                "Initializing": 0,
                "Normal": 5,
                "Abnormal": 0
            }
        },
        "AutoscalingAdded": {
            "Joining": 0,
            "Initializing": 0,
            "Normal": 0,
            "Abnormal": 0
        },
        "Management": {
            "Nameservers": ["183.60.83.19", "183.60.82.98"],
            "Hosts": [],
            "KernelArgs": [],
            "KubeletArgs": []
        }
    }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx`，集群必须处于 Running 状态 | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

### 控制台概念与 API 枚举映射

| 控制台名称 | API 枚举值 | 所在参数 |
|-----------|-----------|---------|
| 原生节点 | `Native` | `CreateClusterNodePool.Type` / `NodePool.Type` |
| 普通节点 | `Regular` | `CreateClusterNodePool.Type` / `NodePool.Type` |
| 按量计费 | `POSTPAID_BY_HOUR` | `InstanceChargeType` |
| 包年包月 | `PREPAID` | `InstanceChargeType` |
| 正常 | `normal` | `NodePool.LifeState` |
| 创建中 | `creating` | `NodePool.LifeState` |

## 验证

### 控制面（tccli）

```bash
# 验证集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterStatus 为 "Running"

# 验证节点池列表可查（含 Type 字段）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中每项含 NodePoolType

# 验证原生节点池详情
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，Type 为 "Native"
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

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region>` 确认集群是否存在 | 集群 ID 不存在或已删除 | 使用正确的 CLUSTER_ID，通过 `tccli tke DescribeClusters --region <Region>` 查询可用集群 |
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound.NodePoolNotFound` | 检查传入的 NODE_POOL_ID 是否属于该集群 | 节点池 ID 不存在或不属于指定集群 | 通过 `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 获取正确的 NODE_POOL_ID |

### 查询成功但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 Type 为 "Regular" | 检查创建时使用的 `CreateClusterNodePool` 参数 | 创建时 Type 未指定为 "Native"，默认为普通节点池 | 普通节点池不具备原生节点能力。如需要原生节点，创建新的 Type="Native" 节点池 |
| `DescribeClusterNodePools` 返回的节点池为 0 | 检查集群是否已创建过节点池 | 空集群无节点池 | 前往 [新建原生节点](../新建原生节点/tccli%20操作.md) 创建原生节点池 |
| `NodeCountSummary` 中所有状态计数为 0 | 检查 `DesiredNodesNum` 和 `LifeState` | 节点池可能处于 `updating` 或 `deleting` 状态，或期望节点数为 0 | 查看 `LifeState`；若为 `normal` 且 `DesiredNodesNum` = 0，调整期望节点数 |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) — 创建原生节点池
- [原生节点功能支持说明](https://cloud.tencent.com/document/product/457/78222) — 功能矩阵详情
- [原生节点生命周期](https://cloud.tencent.com/document/product/457/78200) — 状态流转说明
- [镜像和内核参数说明](https://cloud.tencent.com/document/product/457/124612) — OS 镜像与内核配置

## 控制台替代

[容器服务控制台 - 集群节点管理](https://console.cloud.tencent.com/tke2/cluster)
