# 注册节点概述

> 对照官方：[注册节点概述](https://cloud.tencent.com/document/product/457/57916) · page_id `57916`

## 概述

注册节点（External Node）用于将非腾讯云的 IDC/第三方云主机注册到 TKE 集群中进行统一管理。集群通过**代理组件**与注册节点建立安全隧道，节点无需位于腾讯云 VPC 内即可加入集群。

TKE 提供两种网络接入模式：

| 维度 | 专线版 | 公网版 |
|------|--------|--------|
| 网络通道 | 专线/VPN 连通 VPC | 公网 TLS 隧道 |
| `NetworkType` API 值 | `CiliumBGP` | `HostNetwork` |
| 前置依赖 | VPC 与 IDC 间已通过专线/VPN 互通 | 集群需安装 `externaledge` Addon |
| 所需 CIDR | 需指定 `ClusterCIDR`（Pod IP 段） | 无需指定 `ClusterCIDR` |
| 网络性能 | 低延迟、高带宽（专线保障） | 依赖公网质量 |
| 适用场景 | IDC 与云上 VPC 已有专线，要求低延迟 | 快速接入、无专线环境、测试/开发 |
| 端口要求 | 10250（kubelet API） | 10250（kubelet API）+ 公网访问 |

注册节点接入三步流程：

1. `EnableExternalNodeSupport` — 为集群开启注册节点支持（配置网络模式）
2. `CreateExternalNodePool` — 创建注册节点池（指定运行时、标签、污点）
3. `DescribeExternalNodeScript` — 获取初始化脚本，在目标主机上执行完成注册

> 本文档中所有 ID（集群 ID、节点池 ID、子网 ID 等）均为占位符，请替换为实际值。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    tke:EnableExternalNodeSupport
#    tke:CreateExternalNodePool
#    tke:DescribeExternalNodeScript
#    tke:DescribeExternalNodeSupportConfig
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
# 4. 确认目标集群存在且为托管集群
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterType 为 MANAGED_CLUSTER

# 5. 检查集群是否已开启注册节点支持
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 查看 Enabled 字段确认当前状态
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
| 查看注册节点功能状态 | `DescribeExternalNodeSupportConfig` | 是 |
| 开启注册节点支持（专线版） | `EnableExternalNodeSupport --NetworkType CiliumBGP` | 是 |
| 开启注册节点支持（公网版） | `InstallAddon externaledge` → `EnableExternalNodeSupport --NetworkType HostNetwork` | 部分是 |
| 查看注册节点池 | `DescribeExternalNodePools` | 是 |
| 查看注册节点 | `DescribeExternalNode` | 是 |

### API 能力对比

以下 API 在两种网络模式下的支持情况：

| API | 专线版（CiliumBGP） | 公网版（HostNetwork） |
|-----|:--:|:--:|
| `EnableExternalNodeSupport` | `NetworkType=CiliumBGP`，需 `ClusterCIDR` + `SubnetId` | `NetworkType=HostNetwork`，需 `SubnetId`，无需 `ClusterCIDR` |
| `CreateExternalNodePool` | 支持 | 支持 |
| `DescribeExternalNodeScript` | 支持（`Internal=true` 推荐） | 支持（`Internal=false` 推荐） |
| `DescribeExternalNodePools` | 支持 | 支持 |
| `DescribeExternalNode` | 支持 | 支持 |
| `DescribeExternalNodeSupportConfig` | 支持 | 支持 |
| `ModifyExternalNodePool` | 支持 | 支持 |
| `DeleteExternalNode` | 支持 | 支持 |
| `DeleteExternalNodePool` | 支持 | 支持 |
| `DrainExternalNode` | 支持 | 支持 |
| `InstallAddon externaledge` | 不需要 | **必须**（公网版前置） |

#### EnableExternalNodeSupport 关键参数

| 参数 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `NetworkType` | String | 是 | `HostNetwork`（公网版）或 `CiliumBGP`（专线版） | 填 `DirectConnect`/`Public` 等控制台概念 → `InvalidParameter` |
| `SubnetId` | String | 是 | 已存在的子网 ID，格式 `subnet-xxxxxxxx` | 子网不存在 → `InvalidParameter.SubnetId` |
| `ClusterCIDR` | String | 条件必填 | 专线版（`CiliumBGP`）必填，Pod IP 段，如 `10.200.0.0/16`；公网版（`HostNetwork`）不需要 | CIDR 与 VPC 或已有集群冲突 → `InvalidParameter` |

## 操作步骤

本页为概念说明页，具体操作参见各子页面：

- 专线版接入：[创建注册节点（专线版）](../创建注册节点（专线版）/tccli%20操作.md)
- 公网版接入：[创建注册节点（公网版）](../创建注册节点（公网版）/tccli%20操作.md)

### 查询当前集群的注册节点支持状态

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回当前集群的注册节点配置信息
```

**预期输出**（未开启时）：

```json
{
    "ClusterCIDR": "",
    "NetworkType": "",
    "SubnetId": "",
    "Enabled": false,
    "Status": "",
    "EnabledPublicConnect": false,
    "RequestId": "..."
}
```

**预期输出**（专线版已开启时）：

```json
{
    "ClusterCIDR": "10.200.0.0/16",
    "NetworkType": "CiliumBGP",
    "SubnetId": "subnet-example",
    "Enabled": true,
    "Status": "Running",
    "EnabledPublicConnect": false,
    "RequestId": "..."
}
```

## 验证

本页为概念说明页，无需要验证的操作。

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeExternalNodeSupportConfig` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出所有集群确认 ID | 集群 ID 格式错误或集群不存在于当前地域 | 用 `tccli tke DescribeClusters --region <Region>` 获取正确集群 ID |
| `DescribeExternalNodeSupportConfig` 返回 `FailedOperation` | 检查集群状态：`tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 集群状态非 `Running`（如 `Creating`、`Abnormal`） | 等待集群进入 `Running` 状态后重试 |
| `EnableExternalNodeSupport` 返回 `InvalidParameter.NetworkType` | 检查请求中的 `NetworkType` 值 | 填了控制台概念（`DirectConnect`/`Public`）而非 API 枚举 | 专线版用 `"CiliumBGP"`，公网版用 `"HostNetwork"` |

### 模式选择疑问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 不确定选专线版还是公网版 | 检查 IDC 与 VPC 间是否有专线/VPN：已建立则选专线版 | 专线版延迟更低但需要网络基础设施；公网版部署更快 | 已有专线 → 专线版（`CiliumBGP`）；无专线 → 公网版（`HostNetwork`） |
| 公网版 `InstallAddon externaledge` 失败 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName externaledge` 检查已安装的 addon | `externaledge` 仅托管集群（`MANAGED_CLUSTER`）支持 | 确认集群类型为 `MANAGED_CLUSTER` |

## 下一步

- [网络模式（专线版）](https://cloud.tencent.com/document/product/457/79748) — 专线版网络模式详细说明
- [创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/57917) — 专线版全流程操作指南
- [创建注册节点（公网版）](https://cloud.tencent.com/document/product/457/101532) — 公网版全流程操作指南
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：查看和管理注册节点功能。
