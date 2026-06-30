# 镜像概述

> 对照官方：[镜像概述](https://cloud.tencent.com/document/product/457/68289) · page_id `68289`

## 概述

TKE 提供**公共镜像**与**自定义镜像**两种镜像来源。公共镜像由腾讯云官方维护，经 Kubernetes 环境兼容性适配（网络、存储、节点初始化、GPU 驱动等），所有用户可用。自定义镜像由用户基于 TKE 基础镜像自行制作，需自行保证 Kubernetes 兼容性，TKE 原则上不提供 SLA。

镜像选择存在**集群级别**和**节点池级别**两个作用域：集群级别影响新增节点、添加已有节点、节点升级的默认 OS；节点池级别仅影响该节点池内扩容和添加已有节点。修改 OS 仅影响后续新增或重装节点，不改变存量节点 OS。

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
#    tke:DescribeOSImages, tke:DescribeClusters, tke:DescribeClusterNodePools
# 验证：执行 DescribeOSImages 确认权限
tccli tke DescribeOSImages --region <REGION>
# expected: exit 0，返回 OSImageSeriesSet（可为空）
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在（如需查询集群/节点池级别 OS）
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount >= 1，目标集群信息可查
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
| 查看 TKE 支持的公共镜像列表 | `DescribeOSImages` | 是 |
| 查看集群级别操作系统 | `DescribeClusters` → `ClusterOs`、`OsCustomizeType` | 是 |
| 查看节点池级别操作系统 | `DescribeClusterNodePools` → `OsName`、`ImageId` | 是 |
| 创建集群时选择镜像 | `CreateCluster` → `ClusterBasicSettings.ClusterOs`（公共镜像传 OsName，自定义传 ImageId） | 否 |

## 操作步骤

### 步骤 1：查询 TKE 支持的公共镜像列表

TKE 公共镜像与 CVM 公版镜像列表**不完全一致**。推荐优先选择 **TencentOS Server** 系列。

```bash
tccli tke DescribeOSImages --region <REGION>
# expected: exit 0，返回 OSImageSeriesSet 数组
```

**预期输出**（摘要）：

```json
{
    "OSImageSeriesSet": [
        {
            "OsName": "tlinux4_x86_64_public",
            "SeriesName": "TencentOS Server 4 for x86_64",
            "ImageId": "img-6n21msk1",
            "Status": "online",
            "Alias": "Tencent OS Server",
            "Arch": "x86_64"
        },
        {
            "OsName": "tlinux3.1x86_64",
            "SeriesName": "TencentOS Server 3.1 (TK4)",
            "ImageId": "img-eb30mz89",
            "Status": "online",
            "Alias": "Tencent OS Server",
            "Arch": "x86_64"
        },
        {
            "OsName": "ubuntu22.04x86_64",
            "SeriesName": "ubuntu22.04x86_64",
            "ImageId": "img-487zeit5",
            "Status": "online",
            "Alias": "Ubuntu",
            "Arch": "x86_64"
        },
        {
            "OsName": "centos7.6.0_x64",
            "SeriesName": "centos7.6.0_x64",
            "ImageId": "img-9qabwvbn",
            "Status": "online",
            "Alias": "CentOS",
            "Arch": "x86_64"
        }
    ],
    "TotalCount": 56,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `OsName` | 镜像的 API 标识名，创建集群时 `ClusterOs` 字段传此值 |
| `ImageId` | CVM 侧镜像 ID，格式 `img-xxxxxxxx`，地域相关 |
| `Status` | `online`（可用）/ `offline`（已下线） |
| `SeriesName` | 镜像系列展示名 |
| `Alias` | 控制台展示分类名 |

### 步骤 2：按条件过滤镜像列表

```bash
# 仅查看在线的 x86_64 镜像
tccli tke DescribeOSImages --region <REGION> \
    | jq '.OSImageSeriesSet | map(select(.Status == "online" and .Arch == "x86_64"))'
# expected: 返回 Status 均为 online 的 x86_64 镜像列表
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

```bash
# 确认指定 OsName 是否在线
tccli tke DescribeOSImages --region <REGION> \
    | jq '.OSImageSeriesSet[] | select(.OsName == "tlinux3.1x86_64")'
# expected: 返回该 OsName 的完整信息，Status 应为 online
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

### 步骤 3：查看集群级别操作系统

```bash
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0] | {ClusterId, ClusterOs, OsCustomizeType, ImageId}'
# expected: exit 0，返回集群当前 OS 配置
```

**预期输出**：

```json
{
    "ClusterId": "cls-example",
    "ClusterOs": "tlinux4_x86_64_public",
    "OsCustomizeType": "GENERAL",
    "ImageId": ""
}
```

| 字段 | 说明 |
|------|------|
| `ClusterOs` | 集群级别 OS 的 OsName（公共镜像）或 ImageId（自定义镜像） |
| `OsCustomizeType` | `GENERAL`（公共镜像）/ `CUSTOMIZE`（自定义镜像） |
| `ImageId` | 若为自定义镜像，此项为具体 ImageId；公共镜像为空 |

### 步骤 4：查看节点池级别操作系统

```bash
tccli tke DescribeClusterNodePools --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    | jq '.NodePoolSet[] | {Name, OsName, ImageId}'
# expected: exit 0，返回各节点池 OS 配置
```

**预期输出**：

```json
{
    "Name": "np-example",
    "OsName": "tlinux4_x86_64_public",
    "ImageId": ""
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <REGION>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看当前地域 |

## 验证

```bash
# 1. 确认 DescribeOSImages 返回非空列表
tccli tke DescribeOSImages --region <REGION> \
    | jq '.OSImageSeriesSet | length'
# expected: 返回值 > 0

# 2. 确认目标 OsName 在线
tccli tke DescribeOSImages --region <REGION> \
    | jq '[.OSImageSeriesSet[] | select(.OsName == "tlinux4_x86_64_public")] | length'
# expected: 1（表示该镜像在线）

# 3. 确认集群 OS 可读取
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0].ClusterOs'
# expected: 返回 OsName 字符串（如 "tlinux4_x86_64_public"）

# 4. 确认节点池 OS 可读取（如有节点池）
tccli tke DescribeClusterNodePools --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    | jq '.NodePoolSet[0].OsName'
# expected: 返回 OsName 字符串或 null（无节点池时）
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

## 清理

本页为只读查询，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeOSImages` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeOSImages` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeOSImages` |
| `DescribeOSImages` 返回空的 `OSImageSeriesSet` | `tccli configure list` 检查 `region` 配置 | 当前地域无可用镜像，或 API 超时 | 换地域重试（如 `ap-guangzhou`），或检查网络连通性 |
| `DescribeClusters` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterIds` 中的集群 ID 格式 | 集群 ID 格式错误或集群不属于当前账号/地域 | 用 `tccli tke DescribeClusters --region <REGION>` 列出全部集群确认 ID |

### 镜像查询结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 控制台可见某镜像，但 `DescribeOSImages` 未返回 | `tccli tke DescribeOSImages --region <REGION> \| jq '.OSImageSeriesSet[] \| select(.OsName == "OS_NAME")'` 精确查询 | 该镜像在 CVM 侧未全量开放至 TKE，或地域不匹配 | 在 CVM 控制台确认该镜像是否可选；若找不到可 [提交工单](https://console.cloud.tencent.com/workorder) |
| `DescribeOSImages` 返回的 `SeriesName` 与控制台展示名不一致 | `tccli tke DescribeOSImages --region <REGION> \| jq '.OSImageSeriesSet[] \| {OsName, SeriesName, Alias}'` 对比三元组 | 控制台展示名与 API 字段映射不同；以 API 返回的 `OsName` 为准 | 创建集群时使用 `OsName` 值，不要使用控制台展示名 |
| 自定义镜像在集群创建向导中不可选 | 检查镜像是否符合 TKE 基础镜像要求（见 [自定义镜像说明](../自定义镜像说明/tccli%20操作.md)） | 镜像非基于 TKE 公共镜像制作，或地域与集群不一致 | 使用 TKE 公共镜像重新制作自定义镜像，并确保与集群同地域 |

## 下一步

- [自定义镜像说明](../自定义镜像说明/tccli%20操作.md) — 基于 TKE 基础镜像制作自定义镜像
- [TencentOS Rainbow 镜像介绍](../TencentOS%20Rainbow/镜像介绍/tccli%20操作.md) — 云原生轻量级容器操作系统
- [创建集群](../../集群管理/创建集群/tccli%20操作.md) — 创建集群时选择镜像
- [更改集群操作系统](../../集群管理/更改集群操作系统/tccli%20操作.md) — 修改存量集群的 OS
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群配置 → 镜像](https://console.cloud.tencent.com/tke2) 中的镜像列表与创建向导 OS 选项，对应 `DescribeOSImages` 与 `CreateCluster` 中的 `ClusterOs` / `ImageId` 字段。
