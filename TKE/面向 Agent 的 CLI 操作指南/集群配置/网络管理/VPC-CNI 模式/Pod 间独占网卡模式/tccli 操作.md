# Pod 间独占网卡模式（tccli）

> 对照官方：[Pod 间独占网卡模式](https://cloud.tencent.com/document/product/457/50357) · page_id `50357`

## 概述

Pod 间独占网卡模式（`VpcCniType = tke-direct-eni`）是 VPC-CNI 的子模式之一。每个 Pod 独占一张弹性网卡（ENI），将 ENI 直接配置到 Pod 网络命名空间，性能最优。

**核心特点**：
- 每个 Pod 独享 ENI，无共享网卡的策略路由开销，网络性能接近裸金属
- 支持 EIP 绑定、NAT 网关、固定 IP、CLB 直通
- Pod 密度严格受限于**机型最大可绑定 ENI 数 - 1**（保留一张给节点）
- **仅新建集群支持**该网络方案，存量集群不支持变更

### IP 地址管理原理

| 模式 | 网卡池 | 扩缩容 | 网卡回收 |
|------|--------|--------|---------|
| 非固定 IP | 节点网卡池 | Pod 数 +/- 预绑定阈值伸缩 | Pod 销毁网卡归池，不删除 VPC ENI |
| 固定 IP | 无节点网卡池 | Pod 创建时绑定网卡 | 固定 IP Pod 销毁仅解绑，保留 IP |

### 功能限制

- 仅部分机型支持：S5、SA2、IT5、SA3、M5、SA5、S8、ITA5、BMS 系列等（以官方最新机型列表为准）
- 单节点 Pod 上限 ≈ 最大可绑定 ENI 数 - 1
- **仅新集群**支持，存量 TKE 不支持变更为独占网卡方案
- 需提交工单开通白名单

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群为 VPC-CNI 模式且采用独占网卡方案（`VpcCniType = "tke-direct-eni"`）。
- 已开通独占网卡白名单（提交工单）。
- CAM 权限：`tke:DescribeClusters`。

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
| 创建时选择独占网卡模式 | `CreateCluster` 时指定 `VpcCniType = "tke-direct-eni"` | 否 |
| 查看集群网络配置 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 Pod 数量限制 | 参见 [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md) | 是 |

### EnableVpcCniNetworkType API 参数参考（独占网卡）

| 参数 | 类型 | 必填 | 取值与约束 |
|------|------|:--:|------|
| `ClusterId` | String | 是 | 目标集群 ID |
| `VpcCniType` | String | 是 | `"tke-direct-eni"`（独占网卡模式） |
| `Subnets` | Array of String | 是 | 容器子网 ID 列表，须与节点同 AZ |
| `EnableStaticIp` | Boolean | 否 | 是否启用固定 Pod IP |

> 独占网卡模式需先提交工单开通白名单，否则 API 调用会失败。

## 操作步骤

### 查看集群网络配置

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterNetworkSettings 含 Cni = true
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

### 判断是否为独占网卡模式

`VpcCniType` 不在 `DescribeClusters` 直接返回。判定方法：

- 查看 `Property` JSON 中是否含 `VpcCniType` 字段
- 观察节点 Pod 数量：独占网卡模式下节点 Pod 数远低于共享网卡模式（受 ENI 配额限制）
- 在控制台集群详情页查看网络模式标签

### 查看机型 ENI 配额（影响 Pod 上限）

独占网卡模式 Pod 数量上限 = 机型最大可绑定 ENI 数 - 1。不同机型 ENI 配额不同，参见 [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md)。

## 验证

### 控制面验证

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| VPC-CNI 已启用 | `DescribeClusters → Cni` | `true` |
| 子网已配置 | `DescribeClusters → Subnets` | 非空列表 |

## 清理

本页为概念说明与只读查询，无需清理资源。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ClusterId` 不存在 | 检查地域和账号 | 集群 ID 错误 | 核对 `DescribeClusters` 返回的 ClusterId |
| 独占网卡模式不可选 | 检查集群是否新建 | 存量集群不支持变更为独占网卡方案 | 新建集群时选择 |
| `EnableVpcCniNetworkType` 返回不支持 | 检查白名单状态 | 独占网卡模式需白名单 | 提交工单开通 |
| Pod 数量上限低 | 检查机型 | 不同机型 ENI 配额不同 | 选择 ENI 数量多的机型，如 S5、SA2 系列 |
| `VpcCniType` 填错 | 检查参数值 | 参数需为 `"tke-direct-eni"` | 修正参数，注意引号 |

## 下一步

- [多 Pod 共享网卡模式](../多%20Pod%20共享网卡模式/tccli%20操作.md) -- `tke-route-eni` 模式
- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md) -- 机型 ENI 配额
- [Pod 直接绑定弹性公网 IP 使用说明](../Pod%20直接绑定弹性公网%20IP%20使用说明/tccli%20操作.md) -- EIP 绑定
- [固定 IP 相关特性](../固定%20IP%20模式使用说明/固定%20IP%20相关特性/tccli%20操作.md) -- 固定 IP 配置

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 创建集群 → 网络插件选择「VPC-CNI」→ 选「Pod 间独占网卡」。需先开通白名单。
