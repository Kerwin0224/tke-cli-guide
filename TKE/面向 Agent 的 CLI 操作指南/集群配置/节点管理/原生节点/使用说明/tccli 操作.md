# 使用说明

> 对照官方：[使用说明](https://cloud.tencent.com/document/product/457/102456) · page_id `102456`

## 概述

原生节点的使用限制、配额、前提条件与兼容性说明。包含集群支持原生节点的版本要求、`DescribeVersions` 查询兼容版本、以及创建前需满足的资源条件。

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
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeVersions
#    tke:CreateClusterNodePool, vpc:DescribeVpcs, vpc:DescribeSubnets
#    cvm:DescribeInstances, cvm:DescribeInstanceConfigInfos, cvm:DescribeImages
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 VPC 权限
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，返回 VPC 列表（可为空）

# 验证 CVM 权限
tccli cvm DescribeInstanceConfigInfos --region <Region>
# expected: exit 0，返回机型配置信息（可为空）
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
# 4. 确认目标集群存在且 K8s 版本满足要求
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterVersion ≥ 1.28（原生节点推荐版本）

# 5. 查询可用 K8s 版本
tccli tke DescribeVersions --region <Region>
# expected: exit 0，返回可用版本列表

# 6. 确认 VPC 和子网存在
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，至少有一个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: exit 0，至少有一个子网且 AvailableIpCount ≥ 期望节点数
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
| 查看集群版本 | `DescribeClusters` | 是 |
| 查看可用 K8s 版本 | `DescribeVersions` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看 VPC 和子网 | `DescribeVpcs` / `DescribeSubnets` | 是 |

## 操作步骤

### 使用限制与配额

| 限制项 | 说明 | 查询方式 |
|-------|------|---------|
| K8s 版本 | 推荐 ≥ 1.28，最低 ≥ 1.16（部分能力有更高要求） | `tccli tke DescribeVersions --region <Region>` |
| 集群类型 | 仅托管集群（MANAGED_CLUSTER）和独立集群（INDEPENDENT_CLUSTER）支持 | `DescribeClusters` 的 ClusterType 字段 |
| 镜像 | 仅支持 TencentOS Server（tlinux）系列 | `tccli tke DescribeOSImages --region <Region>` |
| 运行时 | 仅支持 containerd | `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion K8S_VERSION` |
| 机型 | 必须使用 CXM 机型（非普通 CVM 机型） | `tccli cvm DescribeInstanceConfigInfos --region <Region>` |
| 计费 | 按量计费 / 包年包月（同 CVM 计费模式） | 创建时通过 `InstanceChargeType` 指定 |
| 节点池类型 | 创建时必须指定 `Type="Native"` | 通过 `DescribeClusterNodePools` 确认 NodePoolType |
| 地域 | 需要地域支持原生节点能力 | 参考官方文档支持的地域列表 |

### 查询集群是否支持原生节点

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterType 为 MANAGED_CLUSTER 或 INDEPENDENT_CLUSTER，ClusterVersion ≥ 1.28
```

预期输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.30.0",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.1.0.0/16",
                "ServiceCIDR": "10.2.0.0/16",
                "VpcId": "vpc-example"
            }
        }
    ]
}
```

关键检查项：
- `ClusterType` 为 `MANAGED_CLUSTER` 或 `INDEPENDENT_CLUSTER` → 支持
- `ClusterVersion` ≥ `1.28` → 推荐，部分能力完整可用
- `ClusterStatus` 为 `Running` → 集群正常运行

### 查询可用 K8s 版本（兼容性）

```bash
tccli tke DescribeVersions --region <Region>
# expected: exit 0，返回该地域当前支持的所有 K8s 版本
```

预期输出：

```json
{
    "TotalCount": 5,
    "VersionInstanceSet": [
        {
            "Version": "1.34.1",
            "Status": "ENABLE",
            "Remark": ""
        },
        {
            "Version": "1.32.2",
            "Status": "ENABLE",
            "Remark": ""
        },
        {
            "Version": "1.30.0",
            "Status": "ENABLE",
            "Remark": ""
        },
        {
            "Version": "1.28.3",
            "Status": "ENABLE",
            "Remark": ""
        },
        {
            "Version": "1.26.1",
            "Status": "DISABLE",
            "Remark": "已停止新建"
        }
    ]
}
```

版本选择建议：Status 为 `ENABLE` 表示可创建新集群或升级至此版本；`DISABLE` 表示不再接受新建但存量集群可用。

### 查询已有原生节点池

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
```

预期输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-native",
            "Name": "native-pool-prod",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 3,
            "AutoscalingEnabled": false
        }
    ]
}
```

### 控制台概念与 API 枚举映射

| 控制台名称 | API 枚举值 | 所在字段 |
|-----------|-----------|---------|
| 托管集群 | `MANAGED_CLUSTER` | `ClusterType` |
| 独立集群 | `INDEPENDENT_CLUSTER` | `ClusterType` |
| 原生节点池 | `Native` | `NodePoolType` / `Type` |
| 普通节点池 | `Regular` | `NodePoolType` / `Type` |
| 版本可用 | `ENABLE` | `DescribeVersions.Status` |
| 版本不可用（停止新建） | `DISABLE` | `DescribeVersions.Status` |
| 按量计费 | `POSTPAID_BY_HOUR` | `InstanceChargeType` |
| 包年包月 | `PREPAID` | `InstanceChargeType` |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `VPC_ID` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `K8S_VERSION` | K8s 主版本号 | 如 `1.30` | `tccli tke DescribeVersions --region <Region>` |

## 验证

### 控制面（tccli）

```bash
# 验证集群 K8s 版本
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterVersion ≥ 1.28，ClusterType 为 MANAGED_CLUSTER 或 INDEPENDENT_CLUSTER

# 验证可用版本
tccli tke DescribeVersions --region <Region>
# expected: exit 0，VersionInstanceSet 中包含 Status="ENABLE" 且 Version ≥ 1.28 的条目

# 验证节点池列表（确认是否已有原生节点池）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
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
| `DescribeVersions` 返回错误 | 检查 region 参数和 CAM 权限 | 地域不支持或缺少 `tke:DescribeVersions` 权限 | 切换到支持的地域，或检查 CAM 策略中是否包含 `tke:DescribeVersions` |
| `DescribeClusterNodePools` 返回 `ResourceNotFound.ClusterNotFound` | 执行 `DescribeClusters --region <Region>` 确认集群是否存在 | CLUSTER_ID 不存在或已删除 | 使用正确的 CLUSTER_ID |
| `DescribeInstanceConfigInfos` 无 CXM 机型 | 检查 region 和 API 返回的 InstanceFamily | 该地域可能不提供 CXM 机型，此为环境限制 | 尝试其他支持原生节点的地域；此为非命令错误 |

### 查询成功但条件不满足

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 集群 ClusterVersion 低于 1.28 | `DescribeClusters` 查看版本 | 集群版本过低，部分原生节点能力不可用 | 升级集群 K8s 版本：`tccli tke UpdateClusterVersion --region <Region> --ClusterId CLUSTER_ID --DstVersion TARGET_VERSION` |
| 集群 ClusterType 为 `MANAGED_CLUSTER` 但 Version < 1.16 | `DescribeClusters` 查看版本 | 极旧集群，无法创建原生节点池 | 创建新集群（≥ 1.28），在新集群中创建原生节点池 |
| `DescribeVersions` 返回的版本 Status 均为 `DISABLE` | 检查 region 参数 | 该地域可能正在维护中或已停止服务 | 尝试其他地域 |
| VPC 子网 IP 不足 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 查看 AvailableIpCount | 子网可用 IP 少于期望创建的节点数 | 选择 AvailableIpCount 更大的子网，或扩展子网 CIDR |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) — 创建原生节点池
- [原生节点概述](https://cloud.tencent.com/document/product/457/78197) — 理解原生节点概念
- [原生节点功能支持说明](https://cloud.tencent.com/document/product/457/78222) — 功能矩阵
- [原生节点生命周期](https://cloud.tencent.com/document/product/457/78200) — 状态管理

## 控制台替代

[容器服务控制台 - 集群节点管理](https://console.cloud.tencent.com/tke2/cluster)
