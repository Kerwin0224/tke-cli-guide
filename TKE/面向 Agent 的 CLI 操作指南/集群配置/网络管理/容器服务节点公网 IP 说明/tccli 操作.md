# 容器服务节点公网 IP 说明

> 对照官方：[容器服务节点公网 IP 说明](https://cloud.tencent.com/document/product/457/9085) · page_id `9085`

## 概述

TKE 集群节点访问公网有三种方式：**直接分配公网 IP**、**NAT 网关** 和 **弹性公网 IP（EIP）**。三种方案可独立使用或组合使用（如 NAT 网关 + EIP），各有不同的出网路径、带宽限制和计费方式。

### 方案对比

| 方案 | 出网路径 | 带宽限制 | 流量费用 | 适用场景 |
|------|---------|---------|---------|---------|
| 仅公网 IP | 节点直出 | 受 CVM 公网带宽上限限制 | 按 CVM 网络计费模式 | 需要 SSH 登录节点，或少量出网流量 |
| 仅 NAT 网关 | NAT 网关转发 | 不受 CVM 公网带宽限制 | NAT 网关费用（独立计费） | 节点无需公网 IP 但需访问外网（如拉取公网镜像） |
| 仅 EIP | EIP 直出 | 受 CVM 公网带宽上限限制 | 按 CVM 网络计费模式 | 需要固定公网 IP，或公网入站访问 |
| NAT 网关 + EIP | 主动出网走 NAT，入站回包走 EIP | 出网不受 CVM 限制；EIP 回包受 CVM 限制 | NAT 网关费用 + CVM 网络费用 | 混合场景——节点需同时支持主动出网和被公网访问 |

## 前置条件

- [环境准备](../../../环境准备.md)

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
#    cvm:DescribeInstances
#    vpc:DescribeVpcs, vpc:DescribeSubnets, vpc:DescribeNatGateways
#    vpc:DescribeRouteTables, vpc:DescribeAddresses
#    vpc:AllocateAddresses, vpc:AssociateAddress（如需分配 EIP）
#    vpc:CreateNatGateway（如需创建 NAT 网关）
#    验证：执行 DescribeClusters 确认 TKE 权限
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
# 4. 查询集群所在 VPC
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.VpcId'
# expected: 返回 VpcId（如 "vpc-xxxxxxxx"）

# 5. 查询 VPC 下已有的 NAT 网关
tccli vpc DescribeNatGateways --region <Region> \
    | jq '.NatGatewaySet[] | select(.VpcId == "VPC_ID") | {NatGatewayId, State, CreatedTime}'
# expected: 返回 NAT 网关列表（可为空）

# 6. 查询 VPC 下已有的 EIP
tccli vpc DescribeAddresses --region <Region> \
    | jq '.AddressSet[] | select(.AddressStatus != "RELEASED") | {AddressId, AddressIp, AddressStatus, InstanceId}'
# expected: 返回未释放的 EIP 列表（可为空）
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
| 查看集群 VPC 信息 | `tke DescribeClusters`（查询 `ClusterNetworkSettings.VpcId`） | 是 |
| 查看节点公网 IP | `tke DescribeClusters` + `cvm DescribeInstances`（联合查询 `PublicIpAddresses`） | 是 |
| 查看 NAT 网关列表 | `vpc DescribeNatGateways` | 是 |
| 查看弹性公网 IP | `vpc DescribeAddresses` | 是 |
| 查看路由表 | `vpc DescribeRouteTables` | 是 |
| 申请弹性公网 IP | `vpc AllocateAddresses` | 否 |
| 绑定 EIP 到 CVM | `vpc AssociateAddress` | 否 |
| 创建 NAT 网关 | `vpc CreateNatGateway` | 否 |
| 新增路由策略（下一跳 NAT） | `vpc CreateRoutes` | 否 |

### 概念 ↔ API 枚举映射

节点公网 IP 三种方案在 API 中的对应关系：

| 控制台概念 | API 实现 | 查询方式 |
|-----------|---------|---------|
| 节点公网 IP（创建时分配） | CVM `PublicIpAddresses` | `cvm DescribeInstances` → `PublicIpAddresses` 非空 |
| NAT 网关 | VPC NatGateway 资源 | `vpc DescribeNatGateways` → `State: "AVAILABLE"` |
| 弹性公网 IP（EIP） | VPC Address 资源 | `vpc DescribeAddresses` → `AddressStatus: "BIND"` |
| 节点无公网 IP 也无 NAT | 仅内网 IP | `PublicIpAddresses` 为空 + NAT 网关列表为空 |

## 操作步骤

### 通过 DescribeClusters + CVM DescribeInstances 联合查询节点公网 IP

```bash
# 步骤 1：从集群获取节点实例 ID 列表
# 节点信息可通过 DescribeClusterInstances 或 DescribeClusters 的节点列表获取
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.InstanceSet[] | {InstanceId, LanIP: .LanIP, InstanceRole}'
# expected: 返回集群节点列表及其实例 ID

# 步骤 2：根据实例 ID 查询 CVM 公网 IP
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["INS_ID"]' \
    | jq '.InstanceSet[] | {InstanceId, PublicIpAddresses, PrivateIpAddresses, InternetAccessible}'
# expected: 返回实例的公网 IP 和内网 IP；PublicIpAddresses 为空表示未分配公网 IP
```

**预期输出**（有公网 IP 的节点）：

```json
{
    "InstanceId": "ins-xxxxxxxx",
    "PublicIpAddresses": ["1.2.3.4"],
    "PrivateIpAddresses": ["172.16.0.10"],
    "InternetAccessible": {
        "InternetMaxBandwidthOut": 10,
        "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR"
    }
}
```

**预期输出**（无公网 IP 的节点）：

```json
{
    "InstanceId": "ins-xxxxxxxx",
    "PublicIpAddresses": null,
    "PrivateIpAddresses": ["172.16.0.11"],
    "InternetAccessible": {
        "InternetMaxBandwidthOut": 0,
        "InternetChargeType": null
    }
}
```

### 批量查询所有集群节点的公网 IP 状态

```bash
# 获取集群所有节点实例 ID
INSTANCE_IDS=$(tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.InstanceSet[].InstanceId' \
    | jq -R -s -c 'split("\n")[:-1]')

# 批量查询 CVM 公网 IP
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds "$INSTANCE_IDS" \
    | jq '.InstanceSet[] | {
        InstanceId,
        PrivateIp: .PrivateIpAddresses[0],
        PublicIp: (.PublicIpAddresses // [])[0],
        HasPublicIp: (.PublicIpAddresses != null)
    }'
# expected: 列出每个节点的内网 IP 和公网 IP（无公网 IP 为 null）
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 查看 NAT 网关方案

```bash
# 查看集群 VPC 下的 NAT 网关
tccli vpc DescribeNatGateways --region <Region> \
    | jq '.NatGatewaySet[] | select(.VpcId == "VPC_ID") | {
        NatGatewayId,
        State,
        MaxConcurrentConnection,
        PublicIpAddressSet: [.PublicIpAddressSet[]?.PublicIpAddress]
    }'
# expected: 返回 NAT 网关及公网出口 IP
```

**预期输出：**

```json
{
    "NatGatewayId": "nat-xxxxxxxx",
    "State": "AVAILABLE",
    "MaxConcurrentConnection": 5000000,
    "PublicIpAddressSet": ["2.3.4.5"]
}
```

### 查看路由表（确认下一跳为 NAT 网关）

```bash
tccli vpc DescribeRouteTables --region <Region> \
    | jq '.RouteTableSet[] | select(.VpcId == "VPC_ID") | {
        RouteTableId,
        DefaultRoute: [.RouteSet[]? | select(.DestinationCidrBlock == "0.0.0.0/0") | {GatewayType, GatewayId}]
    }'
# expected: 返回默认路由（0.0.0.0/0）及下一跳类型
```

### 查看 EIP 绑定状态

```bash
tccli vpc DescribeAddresses --region <Region> \
    | jq '.AddressSet[] | select(.AddressStatus != "RELEASED") | {
        AddressId,
        AddressIp,
        AddressStatus,
        InstanceId,
        CreatedTime
    }'
# expected: 返回所有未释放的 EIP 及绑定实例
```

## 验证

### Control plane（tccli）

- `DescribeClusters` 返回集群 VPC 信息，`VpcId` 非空。
- `cvm DescribeInstances` 返回节点实例信息，含 `PublicIpAddresses` 和 `InternetAccessible`。
- `vpc DescribeNatGateways` 能正确列出 VPC 下的 NAT 网关及其状态（`AVAILABLE` 表示可用）。
- `vpc DescribeAddresses` 能正确列出未释放的 EIP 及其绑定状态（`BIND`/`UNBIND`）。

### 验证节点公网可达性

| 场景 | 验证命令 | 预期 |
|------|---------|------|
| 节点有公网 IP | `cvm DescribeInstances → PublicIpAddresses` | 非空列表 |
| 节点经 NAT 网关出网 | `vpc DescribeNatGateways` + 路由表有 `0.0.0.0/0 → NAT` | NAT 状态 AVAILABLE |
| 节点经 EIP 出网 | `vpc DescribeAddresses → AddressStatus: "BIND"` + InstanceId 匹配 | 已绑定 |

## 清理

本页为概念说明与只读查询，无额外待清理资源。

若测试性执行了 `AllocateAddresses` 或 `CreateNatGateway`，需按以下顺序手动清理：

1. 解绑并释放 EIP：`vpc DisassociateAddress` → `vpc ReleaseAddresses`
2. 删除路由策略（如有）：`vpc DeleteRoutes`
3. 删除 NAT 网关：`vpc DeleteNatGateway`

删除后验证：

```bash
tccli vpc DescribeAddresses --region <Region> \
    | jq '.AddressSet[] | select(.AddressId == "ADDRESS_ID")'
# expected: 空列表或 RELEASED 状态

tccli vpc DescribeNatGateways --region <Region> \
    | jq '.NatGatewaySet[] | select(.NatGatewayId == "NAT_ID")'
# expected: 空列表或 DELETING/DELETED 状态
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回空 | `tccli tke DescribeClusters --region <Region>` 确认集群存在 | 集群 ID 拼写错误或集群无节点 | 检查 ClusterId，确认集群已创建节点 |
| `cvm DescribeInstances` 返回 `InvalidInstanceId` | 核对实例 ID 格式 `ins-xxxxxxxx` | 实例 ID 格式错误或实例已销毁 | 从 `DescribeClusterInstances` 重新获取正确的 InstanceId |
| `DescribeAddresses` 返回空列表 | `tccli vpc DescribeAddresses --region <Region>` 不加过滤 | 账号下无未释放的 EIP | 为正常情况——按需通过 `AllocateAddresses` 申请 EIP |
| `DescribeNatGateways` 返回空列表 | `tccli vpc DescribeNatGateways --region <Region>` 不加 VpcId 过滤 | VPC 下未创建 NAT 网关 | 为正常情况——按需通过 `CreateNatGateway` 创建 |
| `AllocateAddresses` 返回 `LimitExceeded` | `tccli vpc DescribeAddressQuota --region <Region>` 查询配额 | EIP 数量达到配额上限（此为环境限制，非命令错误） | 释放不再使用的 EIP 后重试，或提交工单提升配额 |
| `CreateNatGateway` 返回 `UnauthorizedOperation` | 检查 CAM 策略是否含 `vpc:CreateNatGateway` | 账号无 NAT 网关创建权限（此为环境限制，非命令错误） | 联系管理员添加权限或在控制台操作 |

### 网络连通性问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点无法访问外网（NAT 网关模式） | `vpc DescribeRouteTables` 检查默认路由 | 路由表缺少指向 NAT 网关的 `0.0.0.0/0` 策略 | 用 `vpc CreateRoutes` 新增默认路由，下一跳为 NAT 网关 |
| 节点无法访问外网（EIP 模式） | `vpc DescribeAddresses --Filters '[{"Name":"instance-id","Values":["INS_ID"]}]'` 检查 EIP 绑定状态 | EIP 已解绑或未绑定到目标 CVM | 用 `vpc AssociateAddress` 重新绑定 EIP |
| `PublicIpAddresses` 为空但节点需外网访问 | `cvm DescribeInstances → InternetAccessible.InternetMaxBandwidthOut` | 节点创建时未分配公网 IP 且未配置 NAT/EIP | 配置 NAT 网关或为节点绑定 EIP |
| NAT 网关出带宽过高导致费用异常 | `vpc DescribeNatGateways` 检查带宽上限 | NAT 网关出带宽未做限制 | 调整 NAT 网关带宽上限或切换计费模式 |

## 下一步

- [容器集群网络规划](../容器集群网络规划/tccli%20操作.md) -- CIDR 规划与节点子网
- [容器集群网络方案选型](../容器集群网络方案选型/tccli%20操作.md) -- 网络方案决策
- [集群启用 IPVS](../集群启用%20IPVS/tccli%20操作.md) -- Service 转发模式
- [GlobalRouter 模式介绍](../GlobalRouter%20模式/GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式节点网络

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：节点管理 → 节点列表展示「公网 IP」列。VPC 控制台可管理 NAT 网关和弹性公网 IP。
