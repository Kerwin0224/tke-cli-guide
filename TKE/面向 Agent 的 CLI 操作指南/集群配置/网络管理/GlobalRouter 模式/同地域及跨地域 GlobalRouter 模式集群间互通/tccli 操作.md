# 同地域及跨地域 GlobalRouter 模式集群间互通

> 对照官方：[同地域及跨地域 GlobalRouter 模式集群间互通](https://cloud.tencent.com/document/product/457/32197) · page_id `32197`

## 概述

GlobalRouter（GR）模式下容器 IP 分配自独立的容器网段，每个节点有独立路由表项，Pod IP 通过 VPC 路由表转发。集群之间默认无法直接通信。实现跨集群互通的核心思路是打通两集群所在 VPC 的网络层，并在路由表中添加指向对方容器网段的路由。

| 方案 | 适用场景 | 核心 API | 验证状态 |
|------|---------|---------|:--:|
| 对等连接（VPC Peering） | 同地域两集群互通 | `CreateVpcPeeringConnection` + `CreateRoutes` | 已实跑验证 |
| 云联网（CCN） | 跨地域多集群互通 | `CreateCcn` + `AttachCcnInstances` | 跨地域容器路由需客服开通 |

> 本文档同地域对等连接全流程命令行验证通过，跨地域云联网为基础命令参考（部分步骤需控制台/客服配合）。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 获取当前账号 Uin（创建对等连接时如跨账号需填写）
tccli sts GetCallerIdentity
# expected: 返回 AccountId（即 Uin）

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterRouteTables, tke:DescribeClusterRoutes
#    vpc:CreateVpcPeeringConnection, vpc:DescribeVpcPeeringConnections, vpc:DeleteVpcPeeringConnection
#    vpc:CreateRoutes, vpc:DeleteRoutes, vpc:DescribeRouteTables
#    vpc:DescribeVpcs, vpc:DescribeSubnets
#    sts:GetCallerIdentity
# 验证 TKE 权限：
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 VPC 权限：
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，返回 VPC 列表
```

### 资源检查

```bash
# 5. 确认源集群为 GR 模式
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: Property.NetworkType == "GR"（非 GR 模式路由逻辑不同，不可用对等连接互通）

# 6. 确认两端 VPC CIDR 不重叠
#    若 VPC CIDR 重叠，对等连接无法建立，需改用 CCN 方案
tccli vpc DescribeVpcs --region <Region> \
    --VpcIds '["<VpcId>", "<PeerVpcId>"]'
# expected: 两端 CidrBlock 无交集
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群基本信息（VPC、容器网段） | `tccli tke DescribeClusters` | 是 |
| 查看可用 VPC 列表 | `tccli vpc DescribeVpcs` | 是 |
| 查看子网信息 | `tccli vpc DescribeSubnets` | 是 |
| 查看路由表 | `tccli vpc DescribeRouteTables` | 是 |
| 查看已有对等连接 | `tccli vpc DescribeVpcPeeringConnections` | 是 |
| 新建对等连接 | `tccli vpc CreateVpcPeeringConnection` | 否 |
| 添加路由策略（容器网段 → 对等连接） | `tccli vpc CreateRoutes` | 否 |
| 删除路由策略 | `tccli vpc DeleteRoutes` | 是 |
| 删除对等连接 | `tccli vpc DeleteVpcPeeringConnection` | 是 |
| 获取当前账号 Uin | `tccli sts GetCallerIdentity` | 是 |

### 关键字段说明

以下说明 `CreateVpcPeeringConnection` 和 `CreateRoutes` 的主要参数。完整参数定义见 `tccli vpc <Action> --generate-cli-skeleton`。

**CreateVpcPeeringConnection**：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `SourceVpcId` | String | 是 | 源端 VPC ID，格式 `vpc-xxxxxxxx`。`tccli vpc DescribeVpcs` 获取 | VPC 不存在 → `InvalidParameterValue.VpcId` |
| `DestinationVpcId` | String | 是 | 对端 VPC ID，不能与 `SourceVpcId` 相同，两端 CIDR 不能重叠 | VPC CIDR 重叠 → 创建被拒绝 |
| `PeeringConnectionName` | String | 是 | 对等连接名称，长度 1-60 字符，自定义 | — |
| `DestinationRegion` | String | 是 | 对端 VPC 所在地域，如 `ap-guangzhou` | 地域不存在 → `InvalidParameterValue.DestinationRegion` |
| `DestinationUin` | String | 否 | 对端账号 Uin。**同账号可省略**（自动使用当前账号），跨账号必填。`tccli sts GetCallerIdentity` 获取 | 跨账号不填 → `InvalidParameterValue.DestinationUin` |

**CreateRoutes**：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `RouteTableId` | String | 是 | 路由表 ID，格式 `rtb-xxxxxxxx`。`tccli vpc DescribeRouteTables` 获取 | 路由表不存在 → `InvalidParameterValue.RouteTableId` |
| `Routes[].DestinationCidrBlock` | String | 是 | 目的网段。此处填**对端集群容器 CIDR**（非 VPC CIDR），不能与已有路由或 VPC CIDR 重叠 | CIDR 冲突 → 路由创建被拒绝 |
| `Routes[].GatewayType` | String | 是 | 下一跳类型。枚举值 `PEERCONNECTION`（全大写，无空格），**不是控制台概念名"对等连接"** | 填"对等连接"等中文名 → `InvalidParameter.GatewayType` |
| `Routes[].GatewayId` | String | 是 | 下一跳实例 ID，即对等连接 ID，格式 `pcx-xxxxxxxx` | 对等连接不存在 → `InvalidParameterValue.GatewayId` |
| `Routes[].Enabled` | Boolean | 否 | 是否启用，默认 `true` | — |
| `Routes[].RouteDescription` | String | 否 | 路由描述，自定义，便于识别 | — |

> **注意**：`RouteType` 仅做出参使用，不作为入参传递。在请求 JSON 中加入 `RouteType` 字段会导致 `UnknownParameter` 错误。

## 操作步骤

### 步骤 1：获取源集群网络信息

#### 选择依据

互通前必须先明确源集群的网络参数。GR 模式下关键信息包括：
- **VPC ID**：确定对等连接源端
- **ClusterCIDR**：容器网段，将作为对方路由表的目的地址
- **Property.NetworkType**：确认当前集群为 GR 模式（非 GR 模式路由逻辑不同，不可用此方案）

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: Property.NetworkType = "GR"，记录 VpcId、ClusterCIDR
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "example-cluster",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterVersion": "1.32.2",
      "VpcId": "vpc-example",
      "ClusterCIDR": "10.0.0.0/16",
      "ServiceCIDR": "10.1.0.0/20",
      "SubnetId": "subnet-example",
      "Property": {
        "NetworkType": "GR"
      }
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 关键字段 | 用途 |
|---------|------|
| `VpcId` | 对等连接源端 VPC |
| `ClusterCIDR` | 容器网段——对方路由表需添加的目的地址 |
| `Property.NetworkType` | 确认网络模式为 GR |

### 步骤 2：选择对端 VPC 并获取 AccountUin

#### 选择依据

- **同账号已有 VPC**：同账号同地域的对等连接创建后自动接受，无需手动 `AcceptVpcPeeringConnection`。跨账号则需对端在控制台或通过 CLI 接受。
- **CIDR 不重叠原则**：两端 VPC CIDR 不能有交集，否则对等连接无法建立。如需对接 CIDR 重叠的 VPC，改用 CCN 方案。

查询可用 VPC 并选择 CIDR 不重叠的对端：

```bash
tccli vpc DescribeVpcs --region <Region>
# expected: 返回 VPC 列表，选择 CidrBlock 与源 VPC CIDR 不重叠的 VPC 作为对端
```

**预期输出**：

```json
{
  "VpcSet": [
    {"VpcId": "vpc-example", "VpcName": "source-cluster-vpc", "CidrBlock": "172.16.0.0/12"},
    {"VpcId": "vpc-peer", "VpcName": "peer-cluster-vpc", "CidrBlock": "10.0.0.0/12"}
  ]
}
```

获取当前账号 Uin（同账号可省略，跨账号必填）：

```bash
tccli sts GetCallerIdentity
# expected: 返回 AccountId
```

**预期输出**：

```json
{
  "AccountId": "100012345678",
  "PrincipalId": "100012345678",
  "Type": "Root",
  "UserId": "100012345678"
}
```

> **CIDR 互斥检查**：若两端 VPC CIDR 或 ClusterCIDR 重叠，对等连接无法建立。执行以下命令确认无冲突：
> ```bash
> tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>", "<PeerVpcId>"]'
> # expected: 两端 CidrBlock 无交集
> ```

### 步骤 3：创建对等连接

#### 选择依据

- **同地域同账号**：创建后自动接受，状态立即为 `ACTIVE`，无需手动 `AcceptVpcPeeringConnection`。
- **跨账号**：需对端在控制台接受连接，或调用 `AcceptVpcPeeringConnection`。
- **GatewayType 枚举**：在后续添加路由时使用 `PEERCONNECTION`（不是控制台概念名"对等连接"），填错会导致 `InvalidParameter.GatewayType`。

```bash
tccli vpc CreateVpcPeeringConnection --region <Region> \
    --SourceVpcId <VpcId> \
    --PeeringConnectionName '<Name>' \
    --DestinationVpcId <PeerVpcId> \
    --DestinationRegion <Region> \
    --DestinationUin <AccountUin>
# expected: 返回 PeeringConnectionId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<VpcId>` | 源集群 VPC ID | 格式 `vpc-xxxxxxxx` | 步骤 1 输出中的 `VpcId` |
| `<Name>` | 对等连接名称 | 长度 1-60 字符，自定义 | 自定义 |
| `<PeerVpcId>` | 对端 VPC ID | 格式 `vpc-xxxxxxxx`，CIDR 不与源 VPC 重叠 | 步骤 2 中 `DescribeVpcs` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<AccountUin>` | 当前账号 Uin | 同账号可省略此参数 | 步骤 2 中 `GetCallerIdentity` |

**预期输出**：

```json
{
  "PeeringConnectionId": "pcx-example",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：验证对等连接已建立

对等连接创建后，验证其状态为 `ACTIVE`：

```bash
tccli vpc DescribeVpcPeeringConnections --region <Region> \
    --PeeringConnectionIds '["<PeeringConnectionId>"]'
# expected: State = "ACTIVE"
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "PeerConnectionSet": [
    {
      "PeeringConnectionId": "pcx-example",
      "PeeringConnectionName": "example-peering",
      "SourceVpcId": "vpc-example",
      "DestinationVpcId": "vpc-peer",
      "State": "ACTIVE"
    }
  ]
}
```

### 步骤 5：在路由表上添加对等连接路由

#### 选择依据

对等连接打通后，VPC 路由表需要知道"对端容器网段的下一跳是对等连接"。

- **路由目的端**：必须填写**对端集群的容器 CIDR**（`ClusterCIDR`），而非 VPC CIDR。这是 GR 模式的核心——Pod IP 经过 VPC 路由表转发，路由表需精确指向对端容器网段。
- **CIDR 不能冲突**：目的网段不能与已有路由重叠，且不能在本 VPC CIDR 范围内。冲突检查命令：
  ```bash
  tccli vpc DescribeRouteTables --region <Region> \
      --RouteTableIds '["<RouteTableId>"]' | jq '.RouteTableSet[0].RouteSet[].DestinationCidrBlock'
  ```
- **GatewayType**：枚举值 `PEERCONNECTION`（全大写，无空格）。

先查询路由表 ID：

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 返回路由表列表，记录 RouteTableId
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "RouteTableSet": [
    {
      "RouteTableId": "rtb-example",
      "RouteTableName": "default",
      "VpcId": "vpc-example",
      "Main": true,
      "RouteSet": []
    }
  ]
}
```

添加路由（参数 ≥ 4 个，使用 `--cli-input-json`）：

`create-routes.json`：

```json
{
  "RouteTableId": "<RouteTableId>",
  "Routes": [
    {
      "DestinationCidrBlock": "<PeerClusterCIDR>",
      "GatewayType": "PEERCONNECTION",
      "GatewayId": "<PeeringConnectionId>",
      "RouteDescription": "route-to-peer-cluster-pods",
      "Enabled": true
    }
  ]
}
```

```bash
tccli vpc CreateRoutes --region <Region> \
    --cli-input-json file://create-routes.json
# expected: exit 0，返回 RouteItemId
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<RouteTableId>` | 源 VPC 路由表 ID | 上方 `DescribeRouteTables` 输出中的 `RouteTableId` |
| `<PeerClusterCIDR>` | 对端集群容器 CIDR | 对端集群 `DescribeClusters` 输出中的 `ClusterCIDR` |
| `<PeeringConnectionId>` | 对等连接 ID | 步骤 3 输出中的 `PeeringConnectionId` |

**预期输出**：

```json
{
  "TotalCount": 1,
  "RouteTableSet": [
    {
      "RouteSet": [
        {
          "RouteItemId": "rti-example",
          "RouteId": 6721504,
          "DestinationCidrBlock": "192.168.100.0/24",
          "GatewayType": "PEERCONNECTION",
          "GatewayId": "pcx-example",
          "Enabled": true
        }
      ]
    }
  ]
}
```

> **双向路由**：上述步骤在源集群侧添加了指向对端容器网段的路由。如需双向互通，在对端 VPC 的路由表上也需添加指向本集群容器网段的路由，`GatewayId` 使用同一个对等连接 ID。

### 步骤 6：验证路由已生效

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# expected: RouteSet 含 DestinationCidrBlock = <PeerClusterCIDR>、GatewayType = PEERCONNECTION 的条目
```

**预期输出**：

```json
{
  "RouteTableSet": [
    {
      "RouteTableId": "rtb-example",
      "RouteSet": [
        {
          "RouteItemId": "rti-example",
          "DestinationCidrBlock": "192.168.100.0/24",
          "GatewayType": "PEERCONNECTION",
          "GatewayId": "pcx-example",
          "Enabled": true
        }
      ]
    }
  ]
}
```

### 跨地域互通（云联网 CCN）

跨地域 GR 集群互通使用云联网。CCN 关联 VPC 后自动传播各 VPC 主 CIDR 路由，但容器网段是独立子网，**不会自动传播**。跨地域容器路由需联系在线客服开通——此步骤无法通过 CLI 自动化。

```bash
# 步骤 1：创建云联网实例
tccli vpc CreateCcn --CcnName '<CcnName>' \
    --QosLevel 'AG' \
    --BandwidthLimitType 'INTER_REGION_LIMIT'
# expected: 返回 CcnId
# QosLevel: PT（白金）、AU（金）、AG（银）、CU（铜）

# 步骤 2：关联源集群 VPC
tccli vpc AttachCcnInstances --region <Region-A> \
    --CcnId <CcnId> \
    --Instances '[{"InstanceType":"VPC","InstanceId":"<VpcId-A>","InstanceRegion":"<Region-A>"}]'
# expected: exit 0

# 步骤 3：关联对端集群 VPC
tccli vpc AttachCcnInstances --region <Region-B> \
    --CcnId <CcnId> \
    --Instances '[{"InstanceType":"VPC","InstanceId":"<VpcId-B>","InstanceRegion":"<Region-B>"}]'
# expected: exit 0

# 步骤 4：验证 CCN 状态
tccli vpc DescribeCcns --CcnIds '["<CcnId>"]'
# expected: State = "AVAILABLE"
```

> **注意**：CCN 关联后需登录 [云联网控制台](https://console.cloud.tencent.com/vpc/ccn) 手动发布容器网段路由。跨地域 GR 模式此操作可能需配合在线客服。

## 验证

### 控制面（tccli）

```bash
# 1. 确认对等连接状态
tccli vpc DescribeVpcPeeringConnections --region <Region> \
    --PeeringConnectionIds '["<PeeringConnectionId>"]'
# expected: State = "ACTIVE"

# 2. 确认路由表含对端容器网段路由
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# expected: RouteSet 含 DestinationCidrBlock = <PeerClusterCIDR>、GatewayType = PEERCONNECTION 的条目
```

### 数据面（kubectl）

```bash
# 在对端集群中部署测试 Pod，从本集群 Pod 中 ping 对端 Pod IP
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec test-pod -- ping -c 3 <PeerPodIP>
# expected: ping 成功，确认容器网络互通
```

> **kubectl 命令需在 VPC 内网可达环境执行**（IOA/VPN/同 VPC CVM）。公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝，无法通过公网直接访问集群。集群内网端点已就绪，可通过同 VPC 内 CVM 访问。

## 清理

> **⚠️ 警告**：
> - **必须先删路由，再删对等连接**。如果先删对等连接再删路由，路由表残留引用已不存在的对等连接，会导致路由表状态异常。
> - 对等连接删除后**立即生效**，依赖该对等连接的跨 VPC 通信将中断。确认无业务依赖后再操作。
> - 路由删除后务必验证已无残留（`DescribeRouteTables` 确认不再含 `GatewayType = PEERCONNECTION` 的路由条目）。

### 数据面（kubectl）

```bash
# 删除测试 Pod
kubectl delete pod test-pod
# expected: pod "test-pod" deleted
```

### 控制面（tccli）

#### 1. 删除路由策略

`delete-routes.json`：

```json
{
  "RouteTableId": "<RouteTableId>",
  "Routes": [
    {"RouteItemId": "<RouteItemId>"}
  ]
}
```

```bash
tccli vpc DeleteRoutes --region <Region> \
    --cli-input-json file://delete-routes.json
# expected: exit 0
```

**预期输出**：

```json
{
  "RouteSet": [
    {"RouteItemId": "rti-example"}
  ]
}
```

#### 2. 验证路由已删除

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --RouteTableIds '["<RouteTableId>"]'
# expected: 不含 GatewayType = PEERCONNECTION 的路由条目
```

#### 3. 删除对等连接

> 确认路由已删除后再执行此步骤。

```bash
tccli vpc DeleteVpcPeeringConnection --region <Region> \
    --PeeringConnectionId <PeeringConnectionId>
# expected: exit 0
```

#### 4. 验证对等连接已删除

```bash
tccli vpc DescribeVpcPeeringConnections --region <Region> \
    --PeeringConnectionIds '["<PeeringConnectionId>"]'
# expected: ResourceNotFound（表示资源已成功删除，这是预期结果）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateVpcPeeringConnection` 返回 `InvalidParameterValue.DestinationUin` | `tccli sts GetCallerIdentity` 确认 AccountId | 未先获取 AccountUin，或跨账号场景未填写对端 Uin | 执行 `tccli sts GetCallerIdentity` 获取 `AccountId`，填入 `--DestinationUin <AccountUin>` |
| `CreateRoutes` 返回 `InvalidParameter.GatewayType` | 检查请求 JSON 中 `GatewayType` 字段值 | 填了控制台概念名"对等连接"而非 API 枚举 `PEERCONNECTION` | 改为 `"GatewayType": "PEERCONNECTION"`（全大写，无空格） |
| `CreateRoutes` 返回路由创建被拒绝 | 执行 `tccli vpc DescribeRouteTables --RouteTableIds '["<RouteTableId>"]' \| jq '.RouteTableSet[0].RouteSet[].DestinationCidrBlock'` 检查已有路由，对比 VPC CIDR：`tccli vpc DescribeVpcs --VpcIds '["<VpcId>"]'` | 路由目的端 CIDR 与 VPC CIDR 重叠，或与已有路由冲突 | 确保 `DestinationCidrBlock` 不在 VPC CIDR 范围内，且不与已有路由条目重叠 |
| `CreateRoutes` 返回 `UnknownParameter` | 检查请求 JSON 中是否含 `RouteType` 字段 | `RouteType` 仅做出参使用，不作为入参传递 | 从请求 JSON 中移除 `RouteType` 字段 |
| `DeleteVpcPeeringConnection` 返回删除失败 | `tccli vpc DescribeRouteTables --RouteTableIds '["<RouteTableId>"]'` 检查是否仍有引用该对等连接的路由条目 | 先删了对等连接再删路由，路由表残留引用已不存在的对等连接 | 严格按 **先 `DeleteRoutes`，后 `DeleteVpcPeeringConnection`** 顺序清理 |
| `DeleteVpcPeeringConnection` 返回 `ResourceNotFound` | `tccli vpc DescribeVpcPeeringConnections --PeeringConnectionIds '["<PeeringConnectionId>"]'` | 资源已被删除（可能是前次清理已完成） | 在清理流程中，`ResourceNotFound` 表示资源已成功删除，这是预期结果 |
| `CreateClusterEndpoint`（公网端点）返回 `InvalidParameter.Param` | 检查 CAM 策略 | 组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝公网端点创建 | 此为环境限制，非命令错误。kubectl 命令需在 VPC 内网可达环境（IOA/VPN/同 VPC CVM）执行。集群内网端点（如 `172.x.x.x`）不受此策略限制 |

### 配置成功但通信异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 对等连接 `ACTIVE` 但容器 ping 不通 | `tccli vpc DescribeRouteTables --RouteTableIds '["<RouteTableId>"]'` 检查两端路由表是否含对方容器网段路由 | 路由表缺少指向对方容器网段的路由（`GatewayType=PEERCONNECTION` 的条目） | 在两端 VPC 路由表上分别添加指向对方容器 CIDR 的路由 |
| 对等连接 `ACTIVE`、路由就绪，但容器 ping 不通 | 检查 CVM 安全组入站规则是否放通对端容器网段 | 安全组未放通对端容器网段流量 | 在两端安全组中添加允许对方容器网段（`ClusterCIDR`）的入站规则 |
| kubectl 无法访问集群 | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId>` 确认端点状态 | 公网端点被 CAM 策略 `strategyId:240463971` 拒绝（`tke:clusterExtranetEndpoint=true`），需 VPC 内网访问 | 通过 IOA/VPN/专线或同 VPC CVM 访问集群内网端点。保留 `region`、`ClusterId`、`RequestId` 以备工单查询 |

## 下一步

- [注册 GlobalRouter 模式集群到云联网](../注册%20GlobalRouter%20模式集群到云联网/tccli%20操作.md) — 将 GR 集群注册到已有 CCN
- [GlobalRouter 模式集群与 IDC 互通](../GlobalRouter%20模式集群与%20IDC%20互通/tccli%20操作.md) — GR 集群与自建 IDC 互通
- VPC 官方文档：[对等连接](https://cloud.tencent.com/document/product/553) · [云联网](https://cloud.tencent.com/document/product/877)

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) 查看集群基本信息，通过 [私有网络控制台](https://console.cloud.tencent.com/vpc) > 对等连接 / 云联网 完成互通配置。
