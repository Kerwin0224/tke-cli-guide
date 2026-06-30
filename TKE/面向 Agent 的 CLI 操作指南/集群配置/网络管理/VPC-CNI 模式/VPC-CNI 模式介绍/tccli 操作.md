# VPC-CNI 模式介绍

> 对照官方：[VPC-CNI 模式介绍](https://cloud.tencent.com/document/product/457/50355) · page_id `50355`

## 概述

VPC-CNI 模式将 VPC 弹性网卡（ENI）IP 直接分配给 Pod，使容器与节点处于同一网络平面。Pod IP 由 IPAMD（`eniipamd`）组件从弹性网卡分配，无需网桥或 VxLAN 隧道。

VPC-CNI 分为两种子模式，由 `VpcCniType` 枚举控制：

| 子模式 | `VpcCniType` 枚举 | 原理 | 特点 |
|--------|-------------------|------|------|
| 多 Pod 共享网卡 | `tke-route-eni` | 多个 Pod 共享一张弹性网卡，IPAMD 为网卡申请多个 IP 分配给不同 Pod | Pod 密度高，支持固定 Pod IP |
| Pod 间独占网卡 | `tke-direct-eni` | 每个 Pod 独占一张弹性网卡 | 性能最优，Pod 密度受机型 ENI 数量限制 |

### VPC-CNI 与 GR 模式对比

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 时延敏感（数据库、中间件、微服务） | VPC-CNI | 少一层网桥转发，性能提升约 10% |
| 传统架构迁移（需固定 IP） | VPC-CNI（共享网卡） | 支持固定 Pod IP |
| IP 级安全策略 | VPC-CNI | 支持 Pod 安全组 |
| 简单固定规模、无特殊需求 | GR | 配置简单、Pod 启动快 |

### 核心约束

1. 容器子网应与 VPC 内其他资源（CVM、CLB 等）隔离，使用**专用子网**
2. 节点和容器子网**必须在同一可用区**，否则 Pod 无法调度
3. 单节点 VPC-CNI Pod 上限 = 机型弹性网卡数 x 每网卡绑定 IP 上限
4. 固定 Pod IP **不支持跨可用区调度**
5. VPC-CNI 需关闭 `rp_filter`，`tke-eni-agent` 组件自动设置

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- CAM 权限：`tke:DescribeClusters`、`vpc:DescribeSubnets`、`vpc:DescribeVpcs`。
- 如需开启 VPC-CNI（写操作），另需 `tke:EnableVpcCniNetworkType`。

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId、secretKey、region 均已配置

tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
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
| 查看 VPC-CNI 集群网络配置 | `tccli tke DescribeClusters --region <Region>`（筛选 `Cni = true`） | 是 |
| 查看容器子网列表 | `DescribeClusters → ClusterNetworkSettings.Subnets` | 是 |
| 查看子网详情 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | 是 |
| 新建 VPC-CNI 集群 | `tccli tke CreateCluster`（设置 `Subnets`、不设 `ClusterCIDR` 或两者皆设） | 否 |
| 为已有 GR 集群开启 VPC-CNI | `tccli tke EnableVpcCniNetworkType --region <Region> --ClusterId <ClusterId> --VpcCniType <Type> --Subnets '["<SubnetId>"]'` | 否 |

### EnableVpcCniNetworkType API 参数参考

| 参数 | 类型 | 必填 | 取值与约束 |
|------|------|:--:|------|
| `ClusterId` | String | 是 | 目标集群 ID |
| `VpcCniType` | String | 是 | `tke-route-eni`（共享网卡）或 `tke-direct-eni`（独占网卡） |
| `Subnets` | Array of String | 是 | 容器子网 ID 列表，须与节点同 AZ |
| `EnableStaticIp` | Boolean | 否 | 是否启用固定 Pod IP（仅 `tke-route-eni` 支持） |

### DescribeClusters 中 VPC-CNI 关键字段

| 字段 | 类型 | 含义 | VPC-CNI 集群预期值 |
|------|------|------|-------------------|
| `ClusterNetworkSettings.Cni` | Boolean | VPC-CNI 开关 | `true` |
| `ClusterNetworkSettings.Subnets` | Array of String | 容器子网列表 | 非空，如 `["<SubnetId>"]` |
| `ClusterNetworkSettings.ClusterCIDR` | String | Pod CIDR | 可能为空或与子网 CIDR 对应 |
| `Property.NetworkType` | String | 控制台网络类型标签 | `"VPC-CNI"`（位于 `Property` JSON 中） |

## 操作步骤

### 查看存量 VPC-CNI 集群

```bash
# 列出所有 Cni = true 的 VPC-CNI 集群
tccli tke DescribeClusters --region <Region>
# expected: 返回集群列表，筛选 Cni = true 的条目
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

### 查看单个 VPC-CNI 集群完整网络配置

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: Cni = true，Subnets 非空，Property 含 NetworkType = "VPC-CNI"
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

### 查看容器子网详情

```bash
# 获取子网 ID 列表后查询子网详情
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# expected: SubnetSet 含 CidrBlock、Zone、AvailableIpAddressCount
```

### 区分 VpcCniType（共享 vs 独占网卡）

`VpcCniType` 不在 `DescribeClusters` 中直接暴露。判定方法：

- 查看 `Property` 字段中是否含 `VpcCniType` 信息
- 共享网卡（`tke-route-eni`）：Pod 共享 ENI，支持固定 IP，Pod 密度高
- 独占网卡（`tke-direct-eni`）：需提交工单开通，仅支持特定机型，Pod 密度受 ENI 配额限制

```bash
# 检查 Property 中是否含 VpcCniType
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: 查看 Property JSON 内容
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

## 验证

### VPC-CNI 模式关键特征验证

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| `Cni` | `DescribeClusters → ClusterNetworkSettings.Cni` | `true` |
| `Subnets` | `DescribeClusters → ClusterNetworkSettings.Subnets` | 非空列表 |
| `NetworkType` | `DescribeClusters → Property` JSON 中 | `"VPC-CNI"` |
| 子网可用区 | `vpc DescribeSubnets` | Pod 子网与节点同 AZ |
| 子网剩余 IP | `vpc DescribeSubnets → AvailableIpAddressCount` | >= 预期 Pod 数 |

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 |
| `vpc DescribeSubnets` 返回 `InvalidParameter.SubnetId` | 核对子网 ID 格式和地域 | 子网 ID 格式错误或不在当前地域 | 先 `DescribeSubnets --region <Region>` 列出全部子网核对 |

### 网络模式疑问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 不确定集群是共享还是独占网卡 | `DescribeClusters` 查看 `Property` | `VpcCniType` 可能不直接暴露 | 共享模式 Pod 密度高（数十/节点），独占模式 Pod 密度低 |
| VPC-CNI Pod 无法调度 | `DescribeSubnets` 查看可用区和可用 IP | 子网与节点 AZ 不一致或 IP 耗尽 | 确保同 AZ；添加子网或增大子网 CIDR |
| `EnableVpcCniNetworkType` 调用失败 | 检查 `VpcCniType` 参数 | 填了非法的 `VpcCniType` | 仅允许 `tke-route-eni` 或 `tke-direct-eni` |

## 下一步

- [多 Pod 共享网卡模式](../多%20Pod%20共享网卡模式/tccli%20操作.md) -- `tke-route-eni` 原理与 IP 分配
- [Pod 间独占网卡模式](../Pod%20间独占网卡模式/tccli%20操作.md) -- `tke-direct-eni` 原理与限制
- [容器网络概述](../../容器网络概述/tccli%20操作.md) -- 三种方案对比
- [容器集群网络规划](../../容器集群网络规划/tccli%20操作.md) -- CIDR 规划与 AZ 规划

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：VPC-CNI 集群基本信息页「集群网络」展示「VPC-CNI」及容器子网列表。
