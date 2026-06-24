# GR集群与IDC互通

> 对照官方：[GlobalRouter 模式集群与 IDC 互通](https://cloud.tencent.com/document/product/457/32199) · page_id `32199`

## 概述

GlobalRouter（GR）模式集群的容器网络独立于 VPC 网络，Pod 从独立于 VPC CIDR 的容器网段（如 `10.0.0.0/16`）分配 IP。当 VPC（如 `172.16.0.0/12`）与 IDC 已通过 VPN 隧道或云联网（CCN）打通后，默认只转发 VPC CIDR 流量，容器网段流量会被 VPC 路由表丢弃。

要使集群内 Pod 与 IDC 服务器实现双向互通，需在 VPC 路由表中新增一条目的端为容器网段、下一跳指向 VPN 网关或 CCN 的路由，并在 IDC 侧添加指向容器网段的回程路由。

> **注意**：本文档涉及的集群 `cls-dex9m3n6` 实际网络模式为 VPC-CNI（tke-route-eni），非 GR 模式。以下 GR 模式的容器路由原理与操作步骤仍适用于 GR 模式集群，VPC-CNI 模式无需本文描述的路由配置（VPC-CNI 模式下 Pod IP 直接从 VPC 子网分配，天然与 VPC CIDR 同属一个路由域）。

本页为 hybrid 页面：控制面（tccli）操作包括查询集群网络配置、查询 VPN 网关/专线状态、配置 VPC 路由表；数据面（kubectl）操作包括部署测试 Pod 验证与 IDC 连通性。数据面 kubectl 命令因 CAM 策略限制公网端点（`tke:clusterExtranetEndpoint` 被 deny），需在同 VPC 内 CVM 上或通过 VPN/专线连接内网端点后执行。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群为 GlobalRouter 模式（`ClusterNetworkSettings.Cni` 为 `false`）。
- VPC 与 IDC 之间已通过 VPN 隧道或云联网（CCN）建立连接，VPC 侧子机与 IDC 侧服务器已可互通。
- 有 VPC 路由表写入权限（`vpc:CreateRoutes`）和 VPN 网关查询权限。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterRouteTables
#    vpc:DescribeVpnGateways, vpc:DescribeVpnConnections, vpc:DescribeDirectConnectGateways
#    vpc:DescribeRouteTables, vpc:DescribeRoutes, vpc:CreateRoutes, vpc:DeleteRoutes
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 VPC 路由表权限
tccli vpc DescribeRouteTables --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回路由表列表（可为空）
```

### 资源检查

```bash
# 4. 查询集群并确认 GR 模式及容器网段
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterNetworkSettings.ClusterCIDR 不为空，Cni 为 false（GR 模式）
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "gr-cluster-demo",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "MaxNodePodNum": 64,
                "MaxClusterServiceNum": 4096,
                "VpcId": "vpc-xxxxxxxx",
                "ServiceCIDR": "10.1.0.0/20",
                "Cni": false
            },
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录输出中的 `ClusterCIDR`（如 `10.0.0.0/16`）和 `VpcId`（如 `vpc-xxxxxxxx`）。

```bash
# 5. 查询 VPN 网关（确认 VPC-IDC 通道已建立）
tccli vpc DescribeVpnGateways --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回至少 1 个 VPN 网关，State 为 AVAILABLE
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "VpnGatewaySet": [
        {
            "VpnGatewayId": "vpngw-xxxxxxxx",
            "VpnGatewayName": "idc-vpn-gw",
            "VpcId": "vpc-xxxxxxxx",
            "Type": "IPSEC",
            "State": "AVAILABLE",
            "PublicIpAddress": "1.2.3.4",
            "InternetMaxBandwidthOut": 100
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 6. 查询 VPN 通道状态
tccli vpc DescribeVpnConnections --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回至少 1 条 VPN 通道，State 为 AVAILABLE
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "VpnConnectionSet": [
        {
            "VpnConnectionId": "vpnx-xxxxxxxx",
            "VpnConnectionName": "idc-tunnel",
            "VpnGatewayId": "vpngw-xxxxxxxx",
            "VpcId": "vpc-xxxxxxxx",
            "State": "AVAILABLE",
            "IKESAStatus": "CONNECTED",
            "IPsecStatus": "CONNECTED",
            "SPDProposalSet": [
                {
                    "LocalAddress": "10.0.0.0/0",
                    "RemoteAddress": "192.168.0.0/16"
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 7. 查询 VPC 路由表
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回至少 1 条路由表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-xxxxxxxx",
            "VpcId": "vpc-xxxxxxxx",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "172.16.0.0/12",
                    "GatewayType": "LOCAL",
                    "GatewayId": "local",
                    "RouteDescription": "默认 Local 路由",
                    "Enabled": true
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录 `RouteTableId`（如 `rtb-xxxxxxxx`）。

```bash
# 8. 查询专线网关（可选：如 VPC-IDC 通过专线连接）
tccli vpc DescribeDirectConnectGateways --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回专线网关列表（可为空）
```

**预期输出**（有专线网关时）：

```json
{
    "TotalCount": 1,
    "DirectConnectGatewaySet": [
        {
            "DirectConnectGatewayId": "dcg-xxxxxxxx",
            "DirectConnectGatewayName": "idc-dcg",
            "NetworkType": "VPC",
            "NetworkInstanceId": "vpc-xxxxxxxx",
            "GatewayType": "NORMAL"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **API 服务更正**：专线网关的查询使用 `tccli vpc DescribeDirectConnectGateways`（属于 `vpc` 服务），而非 `tccli dc DescribeDirectConnectGateways`。官方文档中部分示例可能错误引用 `dc` 服务名，务必以 `vpc` 为准。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 集群基本信息 → 容器网络 | `tke DescribeClusters` → `ClusterNetworkSettings.ClusterCIDR` | 是 |
| VPN 网关管理 → 查看 VPN 网关 | `vpc DescribeVpnGateways` | 是 |
| VPN 通道管理 → 查看 VPN 连接 | `vpc DescribeVpnConnections` | 是 |
| 专线接入 → 查看专线网关 | `vpc DescribeDirectConnectGateways` | 是 |
| 路由表 → 查看路由策略 | `vpc DescribeRouteTables`（过滤 `vpc-id`） | 是 |
| 路由表 → 新增路由策略 | `vpc CreateRoutes`，`DestinationCidrBlock` 填容器网段，`GatewayType` 填 `VPN` 或 `CCN` | 否（重复执行报 `RouteAlreadyExists`） |
| 路由表 → 删除路由策略 | `vpc DeleteRoutes` | 是 |

## 操作步骤

### 操作场景

GR 模式集群的容器与节点处于不同网段。VPC-IDC 互通默认只打通 VPC CIDR 与 IDC 网段，容器网段流量到达 VPC 后无匹配路由会被丢弃。需要：

1. 在 VPC 路由表中添加目的端为容器 `ClusterCIDR`、下一跳为 VPN 网关（或 CCN）的路由
2. 在 IDC 侧路由器/防火墙上添加目的端为容器 `ClusterCIDR`、下一跳指向 VPN 隧道对端地址（或 CCN 出口）的回程路由

### 步骤 1：确认集群容器网段与 VPC 绑定关系

已在前置条件中完成。确认 `ClusterCIDR`（如 `10.0.0.0/16`）、`VpcId`（如 `vpc-xxxxxxxx`）、`RouteTableId`（如 `rtb-xxxxxxxx`）。

### 步骤 2：查询集群路由表（可选）

GR 模式集群自带独立的集群路由表，可通过以下命令查看：

```bash
tccli tke DescribeClusterRouteTables --region <Region>
# expected: exit 0，返回集群路由表列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "clsrt-xxxxxxxx",
            "ClusterId": "cls-xxxxxxxx",
            "VpcId": "vpc-xxxxxxxx",
            "RouteTableCidrBlock": "10.0.0.0/16"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：添加 VPC 路由（容器网段 → VPN/CCN 网关）

#### 选择依据

- **下一跳类型**：选 VPN 网关（`GatewayType: "VPN"`）或云联网（`GatewayType: "CCN"`）。取决于 VPC 与 IDC 已建立的连接方式。当前环境 VPN 网关可用，VPN 通道状态为 `AVAILABLE`。
- **目的端 CIDR**：填集群 `ClusterCIDR`（如 `10.0.0.0/16`），必须与已有 VPC 子网 CIDR、其他路由目的端不冲突。
- **专线网关作为备选**：如使用专线连接，`GatewayType` 改为 `DIRECTCONNECT`，`GatewayId` 填专线网关 ID。当前环境专线网关创建失败（`InternalError`），推荐优先使用 VPN 网关。
- **下一跳存在性检查**：确保 `GatewayId` 对应的 VPN 网关/CCN 实例存在且状态为 `AVAILABLE`。下一跳不存在会导致 `ResourceNotFound` 错误。

#### 添加路由

使用 `--cli-input-json file://` 格式添加路由，`add-vpc-route.json`：

```json
{
    "RouteTableId": "rtb-xxxxxxxx",
    "Routes": [
        {
            "DestinationCidrBlock": "10.0.0.0/16",
            "GatewayType": "VPN",
            "GatewayId": "vpngw-xxxxxxxx",
            "RouteDescription": "pod-cidr-10.0.0.0-16-to-idc-via-vpn"
        }
    ]
}
```

执行添加：

```bash
tccli vpc CreateRoutes --region <Region> --cli-input-json file://add-vpc-route.json
# expected: exit 0
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-xxxxxxxx",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "10.0.0.0/16",
                    "GatewayType": "VPN",
                    "GatewayId": "vpngw-xxxxxxxx",
                    "RouteDescription": "pod-cidr-10.0.0.0-16-to-idc-via-vpn",
                    "Enabled": true
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `rtb-xxxxxxxx` | VPC 路由表 ID | 必须，格式 `rtb-xxxxxxxx` | `tccli vpc DescribeRouteTables --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `vpngw-xxxxxxxx` | VPN 网关 ID | 必须，状态须为 `AVAILABLE` | `tccli vpc DescribeVpnGateways --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `10.0.0.0/16` | 容器网段 CIDR | 必须与 `ClusterCIDR` 一致，不与已有路由冲突 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` |

#### 增强配置（CCN 下一跳）

如使用云联网（CCN）连接，替换网关类型：

`add-vpc-route-ccn.json`：

```json
{
    "RouteTableId": "rtb-xxxxxxxx",
    "Routes": [
        {
            "DestinationCidrBlock": "10.0.0.0/16",
            "GatewayType": "CCN",
            "GatewayId": "ccn-xxxxxxxx",
            "RouteDescription": "pod-cidr-10.0.0.0-16-to-idc-via-ccn"
        }
    ]
}
```

### 步骤 4：IDC 侧配置回程路由

在 IDC 侧路由器/防火墙上添加静态路由：目的网段为容器 `ClusterCIDR`（如 `10.0.0.0/16`），下一跳指向 VPN 隧道对端地址或 CCN 出口。具体操作依 IDC 网络设备品牌而异，此处不展开。

### 步骤 5：确认 VPN 通道 SPD 策略包含容器网段

容器网段加入路由后，VPN 通道的本端/对端安全策略数据库（SPD）必须包含容器网段，否则 IPSec 不会加密该网段流量。

```bash
# 查询当前 VPN 通道 SPD 策略
tccli vpc DescribeVpnConnections --region <Region> \
    --VpnConnectionIds '["<VpnConnectionId>"]'
# expected: SPDProposalSet 中包含容器网段或全域 0.0.0.0/0
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "VpnConnectionSet": [
        {
            "VpnConnectionId": "vpnx-xxxxxxxx",
            "SPDProposalSet": [
                {
                    "LocalAddress": "0.0.0.0/0",
                    "RemoteAddress": "192.168.0.0/16"
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

如果 `LocalAddress` 为 `0.0.0.0/0`，则已包含所有子网（含容器网段），无需修改。如果 SPD 精确限定为 VPC CIDR，需扩展 `LocalAddress` 以包含容器网段：

```bash
tccli vpc ModifyVpnConnectionAttribute --region <Region> \
    --VpnConnectionId <VpnConnectionId> \
    --SecurityPolicyDatabases '["{\"LocalCidrBlock\":\"172.16.0.0/12,10.0.0.0/16\",\"RemoteCidrBlock\":[\"<IDC-CIDR>\"]}"]'
# expected: exit 0
```

## 验证

### 控制面（tccli）

验证路由策略已写入 VPC 路由表：

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# expected: RouteSet 中包含容器网段路由条目，Enabled 为 true
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-xxxxxxxx",
            "VpcId": "vpc-xxxxxxxx",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "172.16.0.0/12",
                    "GatewayType": "LOCAL",
                    "GatewayId": "local",
                    "Enabled": true
                },
                {
                    "DestinationCidrBlock": "10.0.0.0/16",
                    "GatewayType": "VPN",
                    "GatewayId": "vpngw-xxxxxxxx",
                    "RouteDescription": "pod-cidr-10.0.0.0-16-to-idc-via-vpn",
                    "Enabled": true
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

验证 VPN 通道状态：

```bash
tccli vpc DescribeVpnConnections --region <Region> \
    --VpnConnectionIds '["<VpnConnectionId>"]'
# expected: State 为 AVAILABLE，IKESAStatus 和 IPsecStatus 为 CONNECTED
```

**验证维度汇总**：

| 维度 | 命令 | 预期 |
|------|------|------|
| 路由已写入 | `vpc DescribeRouteTables --RouteTableIds '["<RouteTableId>"]'` | `RouteSet` 含容器网段路由，`Enabled: true` |
| VPN 通道可用 | `vpc DescribeVpnConnections --VpnConnectionIds '["<VpnConnectionId>"]'` | `State: "AVAILABLE"` |
| SPD 覆盖容器网段 | 同上，检查 `SPDProposalSet` | `LocalAddress` 为 `0.0.0.0/0` 或包含容器网段 |

### 数据面（kubectl）

> **注意**：公网端点创建受 CAM 策略限制（`tke:clusterExtranetEndpoint` 被 `strategyId:240463971` 硬拒绝）。以下 kubectl 命令需在同 VPC 内 CVM 上或通过 VPN/专线连接集群内网端点后执行。

```bash
# 部署测试 Pod
kubectl run test-to-idc --image=busybox:1.28 --restart=Never -- sleep 300
```

**预期输出**：

```text
pod/test-to-idc created
```

```bash
# 从测试 Pod ping IDC 侧服务器
kubectl exec test-to-idc -- ping -c 3 <IDC-Server-IP>
# expected: 3 packets transmitted, 3 received, 0% packet loss
```

**预期输出**：

```text
PING <IDC-Server-IP> (<IDC-Server-IP>): 56 data bytes
64 bytes from <IDC-Server-IP>: seq=0 ttl=62 time=1.234 ms
64 bytes from <IDC-Server-IP>: seq=1 ttl=62 time=1.345 ms
64 bytes from <IDC-Server-IP>: seq=2 ttl=62 time=1.456 ms

--- <IDC-Server-IP> ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
```

```bash
# 测试 IDC → Pod 方向（从 IDC 服务器 ping Pod IP）
# 在 IDC 侧服务器执行
ping -c 3 <Pod-IP>
# expected: 3 packets transmitted, 3 received, 0% packet loss
```

## 清理

> **副作用警告**：删除 VPC 路由会立即中断集群 Pod 与 IDC 的双向通信。确认无生产流量经过后再执行。
>
> **计费警告**：路由表条目本身不产生额外费用；VPN 网关/CCN/专线资源按使用产生费用。`DeleteRoutes` 仅删除路由条目，不会释放 VPN 网关或 CCN 实例，相关资源如不再需要请手动释放。

### 控制面（tccli）

#### 1. 清理前状态检查

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# 确认待删除的路由条目存在，记录 DestinationCidrBlock、GatewayId
```

#### 2. 删除 VPC 路由

```bash
tccli vpc DeleteRoutes --region <Region> \
    --RouteTableId <RouteTableId> \
    --Routes '[{"DestinationCidrBlock":"10.0.0.0/16","GatewayType":"VPN","GatewayId":"vpngw-xxxxxxxx"}]'
# expected: exit 0
```

#### 3. 验证路由已删除

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# expected: RouteSet 中不再包含容器网段路由条目
```

### 数据面（kubectl）

```bash
kubectl delete pod test-to-idc
# expected: pod "test-to-idc" deleted
```

> **注意**：数据面清理需在同 VPC 内 CVM 或通过 VPN/专线连接集群内网端点后执行。删除 Pod 不会删除 VPC 路由条目——两者独立清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateRoutes` 返回 `ResourceNotFound`（NextHopDestination） | `tccli vpc DescribeVpnGateways --region <Region> --VpnGatewayIds '["<GatewayId>"]'` 检查网关是否存在 | 下一跳 VPN 网关/CCN 实例不存在或已删除，或 `GatewayId` 指向不存在的资源 | 确认网关 ID 正确：`tccli vpc DescribeVpnGateways --region <Region>`，确保状态为 `AVAILABLE`。如网关已销毁，需先重建 VPN 网关和通道再添加路由 |
| `CreateRoutes` 返回 `RouteAlreadyExists` | `tccli vpc DescribeRouteTables --region <Region> --RouteTableIds '["<RouteTableId>"]'` 确认路由是否已存在 | 幂等冲突，相同目的端 + 下一跳的路由策略已存在 | 确认已有路由配置正确，无需重复添加。如需修改下一跳，先 `DeleteRoutes` 再重新 `CreateRoutes` |
| `CreateRoutes` 返回 `InvalidParameterValue.SubnetConflict` | 对比 VPC CIDR 和容器 CIDR：`tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>"]'` | 容器网段与现有 VPC 子网 CIDR 或已有路由目的端重叠 | 核对 `ClusterCIDR` 是否正确（`tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'`）。确认不与 VPC CIDR（如 `172.16.0.0/12`）或已有路由的目的端重叠 |
| `CreateDirectConnectGateway` 返回 `InternalError` | 重试 2 次均返回相同错误 | 后端服务内部故障（非命令参数错误），可能是专线服务临时不可用或地域限制 | 此为环境限制，非命令错误。保留 RequestId 备用。建议改用 VPN 网关或 CCN 作为下一跳替代方案。如需使用专线，可稍后重试或 [提交工单](https://console.cloud.tencent.com/workorder) |
| `DescribeDirectConnectGateways`（`dc` 服务）返回 `InvalidService` | 检查命令使用的服务名 | 使用了错误的 API 服务名 `dc` 而非 `vpc` | 使用 `tccli vpc DescribeDirectConnectGateways`，专线网关 API 属于 `vpc` 服务，非 `dc` 服务 |
| `DescribeVpnConnections` 返回空或状态非 `AVAILABLE` | `tccli vpc DescribeVpnConnections --region <Region> --VpnConnectionIds '["<VpnConnectionId>"]'` | VPN 通道未建立或已断开 | 检查 VPN 通道配置：IKESA Status 和 IPsec Status 须均为 `CONNECTED`。如断开，检查预共享密钥、对端地址、IKE/IPSec 参数是否匹配 |
| `ModifyVpnConnectionAttribute` 返回 `InvalidParameterValue.VpnConnSpdCidrConflict` | 先执行 `tccli vpc DescribeVpnConnections --region <Region> --VpnConnectionIds '["<VpnConnectionId>"]'` 查看当前 SecurityPolicyDatabaseSet | SPD 规则的 LocalCidrBlock 与已有规则存在包含/被包含关系（如当前为 `0.0.0.0/0` 全放通，新增 `10.0.0.0/16` 时会触发冲突） | 若当前 SPD 为 `0.0.0.0/0` 全放通，已覆盖所有容器网段，无需额外修改。如需精确控制，将 LocalCidrBlock 改为不含重叠的具体 CIDR 列表（如 `[\"172.16.0.0/12\", \"10.0.0.0/16\"]`） |
| 填入 `GatewayType: "VPNGW"` 返回 `InvalidParameterValue` | 查看 API 文档中 `GatewayType` 枚举值 | `GatewayType` 枚举名为 `VPN` 而非 `VPNGW` 或 `VPN_GATEWAY` | 使用 `"GatewayType": "VPN"`，详见 `tccli vpc CreateRoutes --generate-cli-skeleton` |

### 创建成功但流量不通

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 路由已写入 VPC 路由表，Pod 无法 ping 通 IDC | 从 Pod exec ping IDC IP | IDC 侧缺少指向容器网段的回程路由 | 在 IDC 路由器上添加静态路由：目的网段 `10.0.0.0/16`，下一跳指向 VPN 隧道对端地址或 CCN 出口 |
| Pod 可 ping IDC 但 IDC 无法 ping Pod | 从 IDC 服务器 ping Pod IP | VPC 安全组或 TKE 节点安全组阻拦 ICMP 入站 | 检查 VPC 安全组规则，放行 IDC 网段（如 `192.168.0.0/16`）到容器网段（`10.0.0.0/16`）的 ICMP 入站流量 |
| 路由条目存在且正确，但流量不通 | 在 VPN 网关上查看流量统计或抓包 | VPN 隧道 SPD 策略未包含容器网段，IPSec 未加密该网段流量 | 扩展 VPN 通道的 `SecurityPolicyDatabases` 使本端网段包含容器 `ClusterCIDR`。参见 [操作步骤 5](#步骤-5确认-vpn-通道-spd-策略包含容器网段) |
| 间歇性丢包或延迟高 | 从 Pod 持续 ping IDC，观察丢包率 | VPN 带宽不足或 MTU 不匹配 | 升级 VPN 带宽；在 VPN 通道上调整 MTU 或启用 MSS 钳制 |
| TCP 不通但 ICMP（ping）通 | 从 Pod telnet IDC 业务端口（如 `telnet <IDC-IP> 3306`） | IDC 侧防火墙或安全组屏蔽特定 TCP 端口 | 检查 IDC 防火墙规则，放行业务所需端口（如 3306、80、443） |
| 数据面 kubectl 命令提示 `connection refused` | `kubectl cluster-info` 确认连通性 | 公网端点被 CAM 策略硬拒绝（`strategyId:240463971`，`tke:clusterExtranetEndpoint=true`），本机不在 VPC 内无法访问内网端点 | 必须在同 VPC 内 CVM 上或通过 IOA/VPN/专线连接集群内网端点后执行 kubectl 命令 |

### 读者常见错误

| 易错点 | 错误表现 | 正确做法 |
|--------|---------|---------|
| 使用 `dc` 服务名查专线网关 | `tccli dc DescribeDirectConnectGateways` 返回 `InvalidService` | 使用 `tccli vpc DescribeDirectConnectGateways`，专线网关属于 VPC 服务 |
| `GatewayType` 填 `VPNGW` 或 `VPN_GATEWAY` | `CreateRoutes` 返回 `InvalidParameterValue` | 枚举值为 `VPN`（非 `VPNGW`），通过 `tccli vpc CreateRoutes --generate-cli-skeleton` 确认 |
| 忘记配置 IDC 回程路由 | Pod → IDC 方向失败（单向不通） | IDC 侧必须添加目的端为容器网段的回程路由 |
| 下一跳网关不存在 | `CreateRoutes` 返回 `ResourceNotFound` | 先用 `DescribeVpnGateways` 确认网关存在且状态为 `AVAILABLE` |
| 容器网段与 VPC CIDR 重叠 | `CreateRoutes` 返回 `SubnetConflict` | GR 模式要求容器网段独立于 VPC CIDR，如 VPC 为 `172.16.0.0/12`，容器应选 `10.0.0.0/16` |
| 误认为 VPC-CNI 集群也需要本文路由配置 | VPC-CNI 模式下 Pod IP 直接从 VPC 子网分配，天然与 VPC CIDR 同属一个路由域 | 仅 GR 模式集群需要本文描述的手动路由配置。通过 `DescribeClusters` 检查 `Cni` 字段确认 |

## 下一步

- [同地域及跨地域 GlobalRouter 模式集群间互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [注册 GlobalRouter 模式集群到云联网](../注册%20GlobalRouter%20模式集群到云联网/tccli%20操作.md)
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md)
- [集群启用 IPVS](../集群启用%20IPVS/tccli%20操作.md)
- [腾讯云 VPC 路由表文档](https://cloud.tencent.com/document/product/215/20060)

## 控制台替代

[控制台 → 私有网络 → 路由表](https://console.cloud.tencent.com/vpc/route) → 选择集群对应 VPC 的路由表 → 新增路由策略，目的端填容器网段，下一跳类型选择 VPN 网关或云联网。
