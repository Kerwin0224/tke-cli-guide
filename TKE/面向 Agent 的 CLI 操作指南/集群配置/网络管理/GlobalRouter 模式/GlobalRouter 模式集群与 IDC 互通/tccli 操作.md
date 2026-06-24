# GlobalRouter 模式集群与 IDC 互通（tccli）

> 对照官方：[GlobalRouter 模式集群与 IDC 互通](https://cloud.tencent.com/document/product/457/32199) · page_id `32199`

## 概述

GlobalRouter（GR）模式集群通过 **专线（Direct Connect）** 或 **IPsec VPN** 两种方式与用户 IDC 实现网络互通。核心思路是将容器网段（ClusterCIDR）纳入 VPC-IDC 互通通道。

| 维度 | 专线 | IPsec VPN |
|------|------|-----------|
| 物理层 | 物理专线，低延迟高带宽 | 基于公网的 IPsec 隧道 |
| 网关类型 | CCN 型专线网关（推荐） | VPN 网关 |
| 容器网络激活 | 需提交工单开通容器路由 | 通过 SPD 策略 + 路由表配置 |
| 路由同步 | BGP 自动 / 手动配置 | 两端手动配置 SPD + 路由表 |
| 适用场景 | 大带宽、低延迟、高稳定性 | 测试或临时互联 |

参考集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1），ap-guangzhou，GR 模式，VPC `vpc-of73262z`（`172.16.0.0/12`），ClusterCIDR `10.200.0.0/16`。`DescribeClusterRouteTables` 返回 `TotalCount=7`，确认集群路由表存在。

> **前置假设**：VPC-IDC 通道已就绪。本文聚焦将容器网段纳入已有互通通道的 CLI 操作。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群为 GR 模式（`ClusterNetworkSettings.Cni` = `false`）。
- VPC 与 IDC 的专线或 VPN 通道已建成且互通正常。
- 容器网段（`10.200.0.0/16`）不与 VPC CIDR（`172.16.0.0/12`）、IDC 网段或其他已发布路由重叠。
- CAM 权限：`tke:DescribeClusters`、`vpc:DescribeRouteTables`、`vpc:CreateRoutes`、`vpc:DeleteRoutes`、`dc:DescribeDirectConnectGateways`、`dc:DescribeDirectConnects`、`vpc:DescribeVpnGateways`、`vpc:DescribeVpnConnections`、`vpc:ModifyVpnConnectionAttribute`。

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId、secretKey、region 均已配置

tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表，含 cls-xxxxxxxx
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
| 查看集群容器网段和 VPC | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` → `ClusterCIDR`、`VpcId` | 是 |
| 确认路由表 | `tccli tke DescribeClusterRouteTables --region <Region>` | 是 |
| 查看物理专线 | `tccli dc DescribeDirectConnects --region <Region>` | 是 |
| 查看专线通道 | `tccli dc DescribeDirectConnectTunnels --region <Region>` | 是 |
| 查看 CCN 型专线网关 | `tccli dc DescribeDirectConnectGateways --region <Region>` | 是 |
| 创建 CCN 型专线网关 | `tccli dc CreateDirectConnectGateway --region <Region>` | 否 |
| 查看 VPN 网关 | `tccli vpc DescribeVpnGateways --region <Region>` | 是 |
| 查看 VPN 通道 SPD 策略 | `tccli vpc DescribeVpnConnections --region <Region>` → `SecurityPolicyDatabaseSet` | 是 |
| 修改 VPN 通道 SPD 策略 | `tccli vpc ModifyVpnConnectionAttribute --region <Region>` | 否 |
| 查看路由表 | `tccli vpc DescribeRouteTables --region <Region>` | 是 |
| 新增路由策略（容器网段 → 专线/VPN） | `tccli vpc CreateRoutes --region <Region>` | 否 |
| 提交工单开通容器路由（专线必做） | 无 API，[在线咨询](https://cloud.tencent.com/online-service) | 否 |

### 关键 API 字段参考

#### DescribeClusters（集群容器网络）

| 字段 | 类型 | 含义 | 当前集群值 |
|------|------|------|------|
| `ClusterCIDR` | String | Pod IP 网段 | `10.200.0.0/16` |
| `VpcId` | String | 集群所属 VPC | `vpc-of73262z` |
| `Cni` | Boolean | 是否 VPC-CNI | `false`（GR 模式） |

#### DescribeDirectConnectGateways（专线网关）

| 字段 | 类型 | 含义 | 取值与约束 |
|------|------|------|------|
| `DirectConnectGatewayId` | String | 专线网关 ID | 提工单时需要提供 |
| `NetworkType` | String | 网关类型 | `CCN`（推荐）或 `NAT` |

#### DescribeVpnConnections（VPN 通道）

| 字段 | 类型 | 含义 | 取值与约束 |
|------|------|------|------|
| `VpnConnectionId` | String | VPN 通道 ID | 用于查看和修改 SPD 策略 |
| `SecurityPolicyDatabaseSet` | Array | SPD 策略集 | `LocalCidrBlock` 须包含容器网段 |
| `State` | String | 通道状态 | `AVAILABLE` 表示正常 |

#### DescribeRouteTables（VPC 路由表）

| 字段 | 类型 | 含义 | 取值与约束 |
|------|------|------|------|
| `DestinationCidrBlock` | String | 目标网段 | 应包含容器网段（`10.200.0.0/16`）和 IDC 网段 |
| `GatewayType` | String | 下一跳类型 | 专线：`DIRECTCONNECT`；VPN：`VPN` |
| `GatewayId` | String | 下一跳实例 ID | 对应专线网关 ID 或 VPN 网关 ID |
| `Enabled` | Boolean | 路由是否启用 | `true` 表示生效 |

## 操作步骤

### 方式一：专线互通

#### 步骤 1：确认路由表和专线基础设施

```bash
# 确认路由表
tccli tke DescribeClusterRouteTables --region ap-guangzhou
# expected: TotalCount = 7，含 cls-xxxxxxxx

# 查看物理专线
tccli dc DescribeDirectConnects --region <Region>
# expected: DirectConnectSet 含已有专线

# 查看 CCN 型专线网关
tccli dc DescribeDirectConnectGateways --region <Region>
# expected: DirectConnectGatewaySet 含 NetworkType = "CCN" 的网关
```

```json
{
  "TotalCount": "<TotalCount>",
  "RouteTableSet": "<RouteTableSet>",
  "RouteTableName": "<RouteTableName>",
  "RouteTableCidrBlock": "<RouteTableCidrBlock>",
  "VpcId": "<VpcId>",
  "RequestId": "<RequestId>"
}
```

> 若无 CCN 型专线网关，需先通过 `tccli dc CreateDirectConnectGateway` 创建。非 CCN 型（NAT）网关不推荐用于容器互通。

#### 步骤 2：获取集群容器网段

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]'
# expected: Cni = false，ClusterCIDR = "10.200.0.0/16"，VpcId = "vpc-of73262z"
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

#### 步骤 3：提交工单开通容器路由

准备以下信息通过 [在线咨询](https://cloud.tencent.com/online-service) 提交（容器服务 TKE 类别）：

| 信息 | 示例 | 获取方式 |
|------|------|---------|
| 地域（Region） | `ap-guangzhou` | `tccli configure list` |
| AppID | `<AppId>` | 控制台「账号信息」 |
| 集群 ID | `cls-xxxxxxxx` | `DescribeClusters` |
| VPC ID | `vpc-of73262z` | `DescribeClusters` |
| 专线网关 ID | `<DcgId>` | `DescribeDirectConnectGateways` |

> 工单开通后，后台自动将容器网段注入专线通道路由。若 IDC 使用 BGP 协议，容器网段路由自动同步到 IDC；使用静态路由则需在 IDC 侧路由设备手动添加。

#### 步骤 4：验证路由表（容器网段指向专线）

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: RouteSet 含 DestinationCidrBlock = "10.200.0.0/16"、GatewayType = DIRECTCONNECT、Enabled = true 的路由
```

---

### 方式二：IPsec VPN 互通

#### 步骤 1：确认 VPN 基础设施

```bash
# 查看 VPN 网关
tccli vpc DescribeVpnGateways --region <Region>
# expected: VpnGatewaySet 含 State = "AVAILABLE" 的网关

# 查看 VPN 通道及 SPD 策略
tccli vpc DescribeVpnConnections --region <Region>
# expected: VpnConnectionSet 含 SecurityPolicyDatabaseSet
```

#### 步骤 2：获取集群容器网段

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]'
# expected: Cni = false，ClusterCIDR = "10.200.0.0/16"
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

#### 步骤 3：配置 SPD 策略

确认 `SecurityPolicyDatabaseSet` 中 `LocalCidrBlock` 包含容器网段。若不包含，通过 `ModifyVpnConnectionAttribute` 修改：

```bash
# 先获取现有 SPD 策略
tccli vpc DescribeVpnConnections --region <Region> \
    --VpnConnectionIds '["<VpnConnectionId>"]'

# 合并容器网段后提交
tccli vpc ModifyVpnConnectionAttribute --region <Region> \
    --VpnConnectionId <VpnConnectionId> \
    --SecurityPolicyDatabases '[{"LocalCidrBlock":"<LocalCidr>","RemoteCidrBlock":["<IdcCidr>"]}]'
```

> `LocalCidrBlock` 需同时包含 VPC CIDR（`172.16.0.0/12`）和容器网段（`10.200.0.0/16`）。远端（IDC 侧 VPN 设备）也需将容器网段加入 SPD 策略对端匹配列表。

#### 步骤 4：配置路由表

```bash
# 查看当前路由表中指向 VPN 的路由
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: RouteSet 含 DestinationCidrBlock = "10.200.0.0/16"、GatewayType = VPN 的路由

# 若容器网段路由不存在，新增
tccli vpc CreateRoutes --region <Region> \
    --RouteTableId <RouteTableId> \
    --Routes '[{"DestinationCidrBlock":"10.200.0.0/16","GatewayType":"VPN","GatewayId":"<VpnGwId>"}]'
# expected: exit 0
```

**路由对称性检查清单：**

| 检查项 | 位置 | 命令 |
|--------|------|------|
| 容器网段 → IDC | 腾讯云 VPC 路由表 | `DescribeRouteTables` → `GatewayType` = `VPN` 或 `DIRECTCONNECT` |
| IDC → 容器网段 | IDC 侧路由设备 | 手动检查回程路由 |

---

### 通用注意事项

- **CIDR 冲突**：容器网段（`10.200.0.0/16`）与 IDC 网段、VPC 网段（`172.16.0.0/12`）均不能重叠。
- **MTU 问题**：VPN 隧道 IPsec 封装开销约 60-80 字节，大包不通需调整 MTU。
- **子网关联**：一个子网只能绑定一个路由表，关联多个以最后一次为准。
- **更改网络模式可能影响已在运行的 Pod 网络**，请谨慎操作。

## 验证

### 控制面验证（tccli）

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群 GR 模式 | `DescribeClusters → .Cni` | `false` |
| 容器网段 | `DescribeClusters → .ClusterCIDR` | `10.200.0.0/16` |
| 路由表存在 | `DescribeClusterRouteTables → TotalCount` | `7`（含 `cls-xxxxxxxx`） |
| 专线网关（专线方式） | `dc DescribeDirectConnectGateways` | `NetworkType = "CCN"` |
| VPN 通道 SPD（VPN 方式） | `DescribeVpnConnections → SecurityPolicyDatabaseSet` | `LocalCidrBlock` 含 `10.200.0.0/16` |
| 路由表容器网段路由 | `DescribeRouteTables` | 存在 `DestinationCidrBlock` = `10.200.0.0/16`，`GatewayType` 为 `DIRECTCONNECT` 或 `VPN`，`Enabled = true` |

## 清理

- **专线 / VPN**：物理专线和 VPN 通道为基础设施，测试环境验证后及时删除。
- **SPD 策略**：通过 `ModifyVpnConnectionAttribute` 移除容器网段相关条目。
- **路由表条目**：通过 `DeleteRoutes` 删除容器网段路由。

```bash
# 删除容器网段路由
tccli vpc DeleteRoutes --region <Region> \
    --RouteTableId <RouteTableId> \
    --Routes '[{"DestinationCidrBlock":"10.200.0.0/16","GatewayType":"VPN","GatewayId":"<VpnGwId>"}]'
```

## 排障

### 命令级错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeDirectConnectGateways` 返回空 | 检查地域 | 该地域无专线网关或多地域选错 | 确认专线网关所在地域 |
| `DescribeVpnConnections` 返回空 | 检查地域 | 该地域无 VPN 通道 | 先通过控制台或 CLI 创建 VPN 通道 |
| `DescribeRouteTables` Filter 无匹配 | 基本查询核对 | `vpc-id` 拼写错误或 VPC ID 不匹配 | 检查 `VpcId` 值（`vpc-of73262z`） |
| `ModifyVpnConnectionAttribute` 失败 | 核对参数 | SPD 策略修改需提供完整列表 | 先用 `DescribeVpnConnections` 获取现有策略，合并后提交 |
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRSettings` | 检查 `ClusterCIDR` 与 VPC CIDR（`172.16.0.0/12`） | CIDR 重叠 | 创建前用 `DescribeClusters` + `DescribeVpcs` 检查所有现有 CIDR |

### 网络连通性故障

| 现象 | 诊断方法 | 可能原因 | 修复步骤 |
|------|---------|---------|---------|
| Pod 不通 IDC（专线） | 确认工单状态；检查路由表 | 工单未开通 / 路由表缺失 | 1. 确认工单；2. 检查 VPC 路由表；3. 检查 IDC 侧路由 |
| Pod 不通 IDC（VPN） | `DescribeVpnConnections` 检查 SPD | SPD 未含容器网段 / 路由缺失 | 1. 添加 SPD（含 `10.200.0.0/16`）；2. 检查双向路由表 |
| BGP 路由未同步 | `DescribeDirectConnects` 确认专线状态 | BGP 会话未建立 | 提工单确认；检查 BGP 对等体 |
| 大包不通小包通 | `ping -s 1472 <PodIp>`（VPN） | IPsec 封装开销致分片 | 调整 TCP MSS 或接口 MTU |
| 子网关联被覆盖 | `DescribeRouteTables` 查看关联子网 | 新路由表覆盖旧表 | 检查子网当前关联；重新关联正确路由表 |
| `CreateCluster` 返回 `InvalidParameter.CidrMaskSizeOutOfRange` | 检查 `ServiceCIDR` 掩码 | ServiceCIDR 掩码写成 `/16` | 掩码必须 17-27，推荐 `/20` |

## 下一步

- [同地域及跨地域 GlobalRouter 模式集群间互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [注册 GlobalRouter 模式集群到云联网](../注册%20GlobalRouter%20模式集群到云联网/tccli%20操作.md)
- [VPC-CNI 模式介绍](../../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md)

## 控制台替代

- [VPC 控制台 → 专线网关](https://console.cloud.tencent.com/vpc/dcgw)：查看和管理专线网关。
- [VPC 控制台 → VPN 通道](https://console.cloud.tencent.com/vpc/vpnConn)：查看 VPN 通道详情和 SPD 策略。
- [VPC 控制台 → 路由表](https://console.cloud.tencent.com/vpc/route)：管理 VPC 路由策略。
- [TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：查看集群 `cls-xxxxxxxx` 基本信息和容器网段。
