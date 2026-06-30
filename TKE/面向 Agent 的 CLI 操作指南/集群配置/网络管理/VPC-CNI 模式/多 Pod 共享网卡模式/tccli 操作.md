# 多 Pod 共享网卡模式（tccli）

> 对照官方：[多 Pod 共享网卡模式](https://cloud.tencent.com/document/product/457/50356) · page_id `50356`

## 概述

多 Pod 共享网卡模式（`VpcCniType = tke-route-eni`）是 VPC-CNI 的子模式之一。多个 Pod 共享同一张弹性网卡（ENI），IPAMD（`eniipamd`）组件为网卡申请多个辅助 IP 分配给不同 Pod，并通过策略路由实现流量分发。

**核心特点**：Pod 密度高（受限于 ENI 辅助 IP 配额），支持**固定 Pod IP**，适用大多数 VPC-CNI 场景。

### IP 地址管理原理

| 模式 | IP 池级别 | 扩缩容策略 | 扩展资源 |
|------|----------|-----------|---------|
| 非固定 IP | 节点维度 | Pod 数 +/- 预绑定阈值伸缩 IP 池 | `tke.cloud.tencent.com/eni-ip` |
| 固定 IP | 集群维度 | 按需分配网卡，销毁保留 IP | 见 [固定 IP 使用方法](../固定%20IP%20模式使用说明/固定%20IP%20使用方法/tccli%20操作.md) |

### 数据面

Pod 出包经策略路由（`ip rule` / `ip route show table <id>`）到对应弹性网卡。`tke-eni-agent` 组件将 `rp_filter` 设为 0，**必须保持关闭**，否则网络不通。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群已创建且为 VPC-CNI 模式，或为 GR 模式准备开启 VPC-CNI。
- 如需开启 VPC-CNI：CAM 权限含 `tke:EnableVpcCniNetworkType`。
- 查看集群信息：`tke:DescribeClusters`、`tke:DescribeAddon`。

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId、secretKey、region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建时选择 VPC-CNI（共享网卡） | `CreateCluster` 时指定 `VpcCniType = "tke-route-eni"` | 否 |
| 存量 GR 集群开启 VPC-CNI | `tccli tke EnableVpcCniNetworkType --VpcCniType "tke-route-eni" --Subnets '["<SubnetId>"]'` | 否 |
| 查看集群网络配置 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 IPAMD 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName eniipamd` | 是 |
| 查看子网剩余 IP | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | 是 |

### EnableVpcCniNetworkType API 参数参考（共享网卡）

| 参数 | 类型 | 必填 | 取值与约束 |
|------|------|:--:|------|
| `ClusterId` | String | 是 | 目标集群 ID |
| `VpcCniType` | String | 是 | `"tke-route-eni"`（共享网卡模式） |
| `Subnets` | Array of String | 是 | 容器子网 ID 列表，须与节点同 AZ |
| `EnableStaticIp` | Boolean | 否 | 是否启用固定 Pod IP，默认 `false` |

### DescribeAddon 返回字段（eniipamd）

| 字段 | 类型 | 说明 |
|------|------|------|
| `AddonName` | String | 组件名称，固定为 `eniipamd` |
| `AddonVersion` | String | 组件版本号，如 `3.8.7` |
| `Phase` | String | 组件状态：`Succeeded`（正常）、`Installing`（安装中）、`Failed`（失败） |

## 操作步骤

### 查看集群 VPC-CNI 网络配置

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterNetworkSettings 含 Cni = true、Subnets 列表
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

### 查看 IPAMD 组件状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd
# expected: Phase = "Succeeded"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

`eniipamd` 组件负责弹性网卡 IP 池管理和策略路由配置。若 `Phase` 非 `Succeeded`，说明组件异常，Pod 网络可能受影响。

### 查看容器子网剩余 IP

```bash
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# expected: AvailableIpAddressCount 表示子网剩余可用 IP 数
```

> 当子网 IP 耗尽时，新 Pod 将无法获取 IP 而调度失败。需提前规划子网 CIDR 大小。

### 开启 VPC-CNI（存量 GR 集群）

```bash
# 为已有 GR 集群开启 VPC-CNI 共享网卡模式
tccli tke EnableVpcCniNetworkType --region <Region> \
    --ClusterId <ClusterId> \
    --VpcCniType 'tke-route-eni' \
    --Subnets '["<SubnetId>"]'
# expected: exit 0，开启后 DescribeClusters 中 Cni = true
```

> 开启后 `eniipamd` 组件自动安装，`tke-eni-agent` 自动将节点 `rp_filter` 设为 0。

## 验证

### 控制面验证

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| VPC-CNI 已启用 | `DescribeClusters → Cni` | `true` |
| 子网已配置 | `DescribeClusters → Subnets` | 非空列表 |
| IPAMD 正常 | `DescribeAddon(eniipamd) → Phase` | `Succeeded` |
| 子网有可用 IP | `DescribeSubnets → AvailableIpAddressCount` | > 0 |

## 清理

本页为概念说明与只读查询，无需清理资源。若曾为测试开启 VPC-CNI 共享网卡模式，通过控制台关闭即可（关闭 VPC-CNI 无 tccli 直接 API）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon` 报 `ResourceNotFound`（eniipamd） | 检查集群网络模式 | 集群未开启 VPC-CNI，无 IPAMD 组件 | GR 集群仅执行 `DescribeClusters` |
| `ClusterId` 不存在 | 检查地域和账号 | 集群 ID 错误或地域不匹配 | 核对 `DescribeClusters` 返回的 ClusterId |
| Pod 网络不通 | 检查 `rp_filter` | 内核参数 `rp_filter` 未关闭 | `tke-eni-agent` 应自动设置，检查其运行状态 |
| 子网 IP 耗尽 | `DescribeSubnets` 查 `AvailableIpAddressCount` | 子网 CIDR 过小或 Pod 过多 | 添加容器子网或扩大子网 CIDR |
| `EnableVpcCniNetworkType` 失败 | 检查 `VpcCniType` 值 | 参数值非法 | 必须为 `"tke-route-eni"` |

## 下一步

- [Pod 间独占网卡模式](../Pod%20间独占网卡模式/tccli%20操作.md) -- `tke-direct-eni` 模式
- [非固定 IP 模式使用说明](../非固定%20IP%20模式使用说明/tccli%20操作.md) -- 非固定 IP 配置
- [固定 IP 使用方法](../固定%20IP%20模式使用说明/固定%20IP%20使用方法/tccli%20操作.md) -- 固定 IP 配置（仅共享网卡支持）
- [eniipamd 组件介绍](../VPC-CNI（eniipamd）组件介绍/tccli%20操作.md) -- 组件详解

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 基本信息 → VPC-CNI 模式 → 开启/关闭。开启时在弹窗中选「共享网卡模式」。
