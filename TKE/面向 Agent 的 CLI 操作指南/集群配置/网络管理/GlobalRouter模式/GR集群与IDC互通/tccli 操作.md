# GR集群与IDC互通
> 对照官方：[GR集群与IDC互通](https://cloud.tencent.com/document/product/457/32199) · page_id `32199`

## 概述

GlobalRouter（GR）模式集群的容器网络独立于 VPC 网络，Pod 从独立于 VPC CIDR 的容器网段分配 IP。要使集群内 Pod 与 IDC（企业数据中心/私有云）服务器实现双向互通，需要在 VPC 路由表和 IDC 侧分别添加指向容器网段的路由，借助 VPN 隧道或云联网（CCN）等已建立的 VPC-IDC 连接完成流量转发。

控制面操作包括查询集群网络配置、查询 VPN 网关状态、查询和配置 VPC 路由表。数据面操作包括在集群内部署测试 Pod 并验证与 IDC 的连通性。本页为 hybrid 页面：tccli 命令正常提供；数据面 kubectl 命令因 CAM 策略限制公网端点（tke:clusterExtranetEndpoint 被 deny），需在 VPC 内 CVM 或通过 VPN/专线执行。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群为 GlobalRouter 模式，通过控制台或 CLI 确认容器网段。
- VPC 与 IDC 之间已通过 VPN 隧道或云联网（CCN）建立连接，且 VPC 侧子机与 IDC 侧服务器已可互通。
- 有 VPC 路由表写入权限（`vpc:CreateRoutes`）。

```bash
# 验证集群存在且获取 ClusterNetworkSettings
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>
# expected: 返回集群详情，ClusterNetworkSettings.ClusterCIDR 不为空

# 验证 VPN 连接已建立
tccli vpc DescribeVpnConnections --region <Region> --VpnConnectionIds '["<VpnConnectionId>"]'
# expected: State 为 "AVAILABLE"

# 验证 VPC 路由表可查询
tccli vpc DescribeRouteTables --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 返回至少一条路由表，RouteTableId 不为空
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

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 集群基本信息 → 容器网络 | 查看容器网段 | `DescribeClusters` → `ClusterNetworkSettings.ClusterCIDR` | 是 |
| VPN 网关管理 | 查看 VPN 网关 | `DescribeVpnGateways` | 是 |
| VPN 通道管理 | 查看 VPN 连接状态 | `DescribeVpnConnections` | 是 |
| 路由表 → 新增路由策略 | 添加指向容器网段的路由 | `CreateRoutes`，DestinationCidrBlock 填容器网段，GatewayType 填 VPN 或 CCN，GatewayId 填网关 ID | 否（重复执行报 `RouteAlreadyExists`） |
| 路由表 → 查看路由策略 | 查看当前 VPC 路由 | `DescribeRouteTables`，过滤 `vpc-id` | 是 |

## 操作步骤

### 操作场景

GR 模式集群的容器与节点处于不同网段。VPC-IDC 互通后，默认只打通 VPC CIDR 与 IDC 网段，容器网段流量会被 VPC 丢弃。需在 VPC 路由表中增加一条目的端为容器网段、下一跳为 VPN/CCN 网关的路由，IDC 侧同样需将容器网段指向 VPC，实现双向互通。

### 1. 获取集群容器网段

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>
```

预期输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "gr-cluster-demo",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "172.16.0.0/16",
                "MaxClusterServiceNum": 256,
                "MaxNodePodNum": 64,
                "VpcId": "vpc-example",
                "ServiceCIDR": "10.0.0.0/20"
            },
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ]
}
```

记录输出中的 `ClusterCIDR`（例如 `172.16.0.0/16`），后续路由配置中将其作为目的端。

### 2. 查询 VPN 网关

```bash
tccli vpc DescribeVpnGateways --region <Region> --VpnGatewayIds '["<VpnGwId>"]'
```

预期输出：

```json
{
    "TotalCount": 1,
    "VpnGatewaySet": [
        {
            "VpnGatewayId": "vpngw-example001",
            "VpnGatewayName": "idc-vpn-gw",
            "VpcId": "vpc-example",
            "Type": "IPSEC",
            "State": "AVAILABLE",
            "PublicIpAddress": "1.2.3.4",
            "InternetMaxBandwidthOut": 100
        }
    ]
}
```

### 3. 查询 VPC 路由表

```bash
tccli vpc DescribeRouteTables --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
```

预期输出：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-example001",
            "VpcId": "vpc-example",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "10.0.0.0/16",
                    "GatewayType": "LOCAL",
                    "GatewayId": "local",
                    "RouteDescription": "默认 Local 路由"
                }
            ]
        }
    ]
}
```

记录 `RouteTableId`（例如 `rtb-example001`），后续添路由时使用。

### 4. 添加 VPC 路由：容器网段 → VPN/CCN 网关

使用 `--cli-input-json file://` 格式添加路由：

**route-input.json（VPC 路由表添加容器网段路由）：**

```json
{
    "RouteTableId": "rtb-example001",
    "Routes": [
        {
            "DestinationCidrBlock": "172.16.0.0/16",
            "GatewayType": "VPN",
            "GatewayId": "vpngw-example001",
            "RouteDescription": "pod-cidr-to-idc-via-vpn"
        }
    ]
}
```

执行添加：

```bash
tccli vpc CreateRoutes --cli-input-json file://route-input.json --region <Region>
```

预期输出：

```json
{
    "TotalCount": 1,
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-example001",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "GatewayType": "VPN",
                    "GatewayId": "vpngw-example001",
                    "RouteDescription": "pod-cidr-to-idc-via-vpn",
                    "Enabled": true
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 5. IDC 侧配置回程路由

在 IDC 侧路由器/防火墙上添加静态路由：目的网段为容器 `ClusterCIDR`，下一跳指向 VPN 隧道对端地址或 CCN 出口。具体操作依 IDC 网络设备品牌而异，此处不展开。

## 验证

### 控制面（tccli）

验证路由策略已写入 VPC 路由表：

```bash
tccli vpc DescribeRouteTables --region <Region> --RouteTableIds '["<RouteTableId>"]'
```

预期输出中应包含 `DestinationCidrBlock` 为容器网段、`GatewayType` 为 VPN（或 CCN）的路由条目：

```json
{
    "RouteTableSet": [
        {
            "RouteTableId": "rtb-example001",
            "RouteSet": [
                {
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "GatewayType": "VPN",
                    "GatewayId": "vpngw-example001",
                    "RouteDescription": "pod-cidr-to-idc-via-vpn",
                    "Enabled": true
                }
            ]
        }
    ]
}
```

验证 VPN 连接状态：

```bash
tccli vpc DescribeVpnConnections --region <Region> --VpnConnectionIds '["<VpnConnectionId>"]'
# expected: State 为 "AVAILABLE"
```

### 数据面（kubectl）

**注意：公网端点创建受 CAM 策略限制（tke:clusterExtranetEndpoint 被 deny）。以下 kubectl 命令需在 VPC 内 CVM 上或通过 VPN/专线连接到集群 APIServer 后执行。**

```bash
# 部署测试 Pod
kubectl run test-to-idc --image=busybox:1.28 --restart=Never -- sleep 300
```

```bash
# 从测试 Pod ping IDC 侧服务器
kubectl exec test-to-idc -- ping -c 3 <IDC-Server-IP>
# expected: 3 packets transmitted, 3 received, 0% packet loss
```

## 清理

**副作用警告：删除 VPC 路由会中断集群与 IDC 的互通，确认无生产流量后再执行。**

**计费警告：路由表条目本身不产生额外费用；VPN/CCN 资源按使用产生费用，删除路由不会删除 VPN 网关或 CCN，相关资源如不再需要请手动释放。**

### 控制面（tccli）

```bash
tccli vpc DeleteRoutes --RouteTableId <RouteTableId> --Routes '[{"DestinationCidrBlock":"172.16.0.0/16","GatewayType":"VPN","GatewayId":"<VpnGwId>"}]' --region <Region>
```

### 数据面（kubectl）

```bash
kubectl delete pod test-to-idc
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateRoutes` 返回 `RouteAlreadyExists` | 查询路由表确认路由是否已存在 | 幂等冲突，路由策略已存在 | 确认已有路由正确，无需重复添加；如需修改请先 `DeleteRoutes` |
| `CreateRoutes` 返回 `InvalidParameterValue.VpnGatewayNotFound` | VPN 网关 ID 不存在或已删除 | 网关已销毁或 ID 输入错误 | 核对 VPN 网关 ID：`tccli vpc DescribeVpnGateways --region <Region>` |
| `CreateRoutes` 返回 `InvalidParameterValue.SubnetConflict` | 目的 CIDR 与已有路由重叠 | 容器网段与现有 VPC 子网或路由冲突 | 核对 `ClusterCIDR` 是否正确，确认不与 VPC CIDR 重叠 |
| `CreateRoutes` 返回 `UnauthorizedOperation` | 检查 CAM 权限 | 当前子账号无 `vpc:CreateRoutes` 权限 | 主账号授予 `QcloudVPCFullAccess` 或自定义策略 |
| `DescribeRouteTables` 过滤 `vpc-id` 无结果 | VPC ID 不正确或已删除 | 集群绑定的 VPC 可能已变更 | 用 `DescribeClusters` 重新确认 `VpcId` |
| `DescribeVpnConnections` 返回空或状态非 `AVAILABLE` | VPN 通道未建立或已断开 | VPN 连接异常 | 检查 VPN 通道配置；IKESA/IPSEC Status 须为 `CONNECTED` |
| 数据面 kubectl 命令提示 `connection refused` | 无法连接集群 APIServer | 公网端点被 CAM 策略 deny | 在 VPC 内 CVM 或通过 VPN/专线执行 kubectl 命令 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 路由已写入但 Pod 无法 ping 通 IDC | 从 Pod exec ping IDC IP | IDC 侧缺少回程路由 | 在 IDC 路由器上添加指向容器网段的静态路由，下一跳为 VPN 隧道对端地址 |
| Pod 可 ping IDC 但 IDC 无法 ping Pod | 从 IDC 服务器 ping Pod IP | VPC 安全组或 TKE 安全组阻拦 ICMP 入站 | 检查 VPC 安全组规则，放行 IDC 网段到容器网段的流量 |
| 路由条目存在但流量不通 | 在 VPN 网关上抓包 | VPN 隧道 SPD 策略未包含容器网段 | VPN 通道的本端/对端网段中增加容器 `ClusterCIDR` |
| 间歇性丢包 | 从 Pod 持续 ping IDC，观察丢包率 | VPN 带宽不足或 MTU 不匹配 | 升级 VPN 带宽，调整 MTU 或启用 MSS 钳制 |
| 单个协议（如 TCP）不通但 ICMP 通 | 从 Pod telnet IDC 端口 | IDC 侧防火墙或安全组屏蔽特定端口 | 检查 IDC 防火墙规则，放行业务端口 |

## 下一步

- [同地域及跨地域 GlobalRouter 模式集群间互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [注册 GlobalRouter 模式集群到云联网](../注册%20GlobalRouter%20模式集群到云联网/tccli%20操作.md)
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md)

## 控制台替代

[控制台 → 私有网络 → 路由表](https://console.cloud.tencent.com/vpc/route) → 选择集群对应 VPC 的路由表 → 新增路由策略，目的端填容器网段，下一跳类型选择 VPN 网关或云联网。
