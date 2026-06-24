# 超级节点概述

> 对照官方：[超级节点概述](https://cloud.tencent.com/document/product/457/74014) · page_id `74014`

## 概述

超级节点是 TKE 提供的可用区级别、支持自定义规格的 Serverless 虚拟节点。超级节点无实际 CVM 实例，不计入 CVM 配额，通过 `CreateClusterVirtualNodePool` API 创建和管理。提供**包年包月**和**按量计费**两种计费模式。

| 维度 | 超级节点 | 普通节点 | 原生节点 |
|------|---------|---------|---------|
| 节点类型 | 虚拟节点，后端自动调度 | 实际 CVM 实例 | CVM/CXM 实例 + 声明式管理 |
| 资源规格 | 0.25C~64C，自定义 CPU/内存比 | 由 CVM 机型决定 | 由机型决定，支持规格微调 |
| 计费模式 | 包年包月 / 按量 | 包年包月 / 按量 | 包年包月 / 按量 |
| 管理 API | `CreateClusterVirtualNodePool` | `CreateClusterNodePool` 等 | `CreateClusterNodePool`（`Type: Native`） |
| CVM 配额 | 不占用 | 占用 | 占用 |
| 弹性速度 | 秒级（按量） | 分钟级 | 分钟级 |
| DaemonSet | 需 Annotation 配置 | 默认支持 | 默认支持 |
| 日志采集 | 需 Annotation 配置 | 组件安装后即可 | 组件安装后即可 |
| 镜像缓存 | 支持 | 无需（本地已有镜像层） | 无需 |
| 适用场景 | 弹性扩缩、短生命周期任务 | 常驻业务 | 声明式节点运维 |

**选择建议**：需要快速弹性、不关心底层 CVM 的场景选超级节点。常驻固定规格业务选普通/原生节点。

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
#    tke:CreateClusterVirtualNodePool, tke:DescribeClusterVirtualNodePools
#    tke:DeleteClusterVirtualNodePool, tke:DescribeClusters
#    vpc:DescribeSubnets, vpc:DescribeSecurityGroups
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
# 4. 确认集群存在且为 MANAGED_CLUSTER
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterType 为 MANAGED_CLUSTER，ClusterStatus 为 Running

# 5. 查询集群已有超级节点池（确认是否重复创建）
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 NodePoolSet（可为空）

# 6. 查询可用子网
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 至少返回 1 个子网，可用 IP 充足

# 7. 查询安全组
tccli vpc DescribeSecurityGroups --region <Region>
# expected: 至少返回 1 个安全组
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
| 查看超级节点概览 | `DescribeClusterVirtualNodePools` | 是 |
| 查看超级节点列表 | `DescribeClusterVirtualNodePools --ClusterId` | 是 |
| 查看单个超级节点详情 | `DescribeClusterVirtualNodes --NodePoolId` | 是 |
| 新建超级节点池 | `CreateClusterVirtualNodePool` | 否 |
| 删除超级节点池 | `DeleteClusterVirtualNodePool` | 是 |
| 扩容超级节点池 | `CreateClusterVirtualNode` | 否 |
| 移除超级节点 | `DeleteClusterVirtualNode` | 是 |

## 操作步骤

### 步骤 1：查询集群超级节点池状态

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 NodePoolSet 数组
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "NodePoolSet": [],
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `TotalCount` | 超级节点池数量，0 表示尚未创建 |
| `NodePoolSet[].NodePoolId` | 超级节点池 ID，用于后续操作 |
| `NodePoolSet[].LifeState` | 节点池生命周期状态：`normal` 表示正常 |
| `NodePoolSet[].NodeCountSummary` | 节点池内各状态节点数量汇总 |

### 步骤 2：创建超级节点池入口

创建超级节点池是后续所有操作的前置步骤。详见 [新建超级节点](../使用超级节点/新建超级节点/tccli%20操作.md)。

## 验证

### 确认超级节点能力可用

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，RequestId 非空
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| API 可调用性 | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | exit 0，返回 JSON（TotalCount 可为 0） |
| 集群类型 | `DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | `ClusterType: "MANAGED_CLUSTER"` |
| 集群状态 | 同上 | `ClusterStatus: "Running"` |

## 清理

本页为概念说明页，仅执行只读操作，无资源需清理。

> 如已创建超级节点池并需清理，参见 [新建超级节点](../使用超级节点/新建超级节点/tccli%20操作.md) 的清理章节。删除超级节点池前须先移除池内所有超级节点，确保无 Pod 调度在超级节点上。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterVirtualNodePools` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterId` 参数格式：`tccli tke DescribeClusters --region <Region>` 列出全部集群确认 ID | 集群 ID 格式错误（应为 `cls-xxxxxxxx`）或集群不属于当前账号/地域 | 用 `tccli tke DescribeClusters --region <Region>` 确认正确的 ClusterId；检查 `tccli configure list` 确认 region 与集群所在区域一致 |
| `DescribeClusterVirtualNodePools` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusterVirtualNodePools` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusterVirtualNodePools` |
| `DescribeClusterVirtualNodePools` 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 集群不存在或已被删除 | 确认 ClusterId 正确，或用 `DescribeClusters` 查找现有集群 |

### 操作成功但不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `TotalCount` 为 0，但控制台显示有节点池 | `tccli configure list` 检查 region；`tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 再次确认 | region 参数与控制台不一致 | 确认 `tccli configure list` 中 region 与控制台一致 |

## 下一步

- [新建超级节点](https://cloud.tencent.com/document/product/457/78328) — 通过 CLI 创建超级节点池和虚拟节点
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — nodeSelector/tolerations 调度 Pod 到超级节点
- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — Annotation 控制资源规格、调度策略
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度 Pod 规格与限制
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 超级节点](https://console.cloud.tencent.com/tke2/cluster)
