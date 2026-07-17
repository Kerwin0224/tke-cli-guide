---
doc_type: How-to
subtype: 6A
fused: false
---
# 准备 VPC 与子网

> TKE 集群必须部署在 VPC 内。若你已有 VPC+子网，可跳过本文，用 `tccli vpc DescribeSubnets` 验证。本文提供从零创建最小可用 VPC+子网的闭环。
> VPC 是腾讯云网络服务，完整能力见 [VPC 产品文档](https://cloud.tencent.com/document/product/215)。

## 概述

TKE 集群节点从子网分配内网 IP，Pod/Service 用 VPC CIDR 通信。创建集群前需备好：1 个 VPC + 1 个可用子网（可用 IP ≥ 10）。

## 触发条件

- 需创建 TKE 集群但 `tccli vpc DescribeVpcs` 返回空或无可用子网（`CreateCluster` 必传 `VpcId`）— 用本文从零建一个最小 VPC+子网
- 已有 VPC+子网但 `tccli vpc DescribeSubnets` 返回 `AvailableIpAddressCount < 10` 或 CIDR 与已有 VPC 冲突 — 跳到 [验证](#验证) 段核对

## 决策依据

| 项 | 选项 | 推荐 |
|:---|:-----|:-----|
| VPC CIDR | `10.0.0.0/16`（65536 IP）/ `192.168.0.0/16` | `10.0.0.0/16`（与 IDC 冲突少） |
| 子网 CIDR | VPC CIDR 的子段，如 `10.0.1.0/24`（254 IP） | `10.0.1.0/24` |
| 可用区 | 如 `ap-guangzhou-6` | 以 `tccli cvm DescribeZones` 返回的 `ZoneState=AVAILABLE` 为准（广州当前常见为 5/6/7，勿写死已下线可用区） |

> CIDR 不可与已有 VPC 重叠。集群创建后 VPC CIDR 无法更改，子网可后加。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 输出当前已安装的 TCCLI 版本号

tccli tke DescribeRegions --filter "TotalCount" --output text
# expected: 数字（如 19；随账号/产品开通变化）→ 凭证有效 + TKE 域可达
```

凭证配置见 [配置凭证](credentials.md)

### 资源检查

```bash
# 查可用区
tccli cvm DescribeZones --region <REGION> \
  --filter "ZoneSet[0].{zone:Zone,state:ZoneState}"
# expected: AVAILABLE 状态的可用区，如 ap-guangzhou-6

# 查已有 VPC（避免 CIDR 冲突）
tccli vpc DescribeVpcs --region <REGION> \
  --filter "VpcSet[].{id:VpcId,cidr:CidrBlock,name:VpcName}" --output text
# expected: 已有 VPC 列表（核对 10.0.0.0/16 是否已被占用）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<ZONE>` | 可用区 | 如 `ap-guangzhou-6` | `tccli cvm DescribeZones --region <REGION> --filter "ZoneSet[?ZoneState=='AVAILABLE'].Zone" --output text` |
| `<VPC_NAME>` | VPC 名称 | 1-60 字符 | 自定义 |
| `<SUBNET_NAME>` | 子网名称 | 1-60 字符 | 自定义 |

## 操作步骤

### 1. 创建 VPC

```bash
tccli vpc CreateVpc --region <REGION> \
  --VpcName "<VPC_NAME>" --CidrBlock "10.0.0.0/16"
# expected: exit 0，返回 Vpc.VpcId
```
```json
{
    "Vpc": {
        "VpcId": "vpc-example",
        "VpcName": "<VPC_NAME>",
        "CidrBlock": "10.0.0.0/16"
    },
    "RequestId": "req-example"
}
```

> 记下返回的 `VpcId`，下一步用。

### 1a. 创建双栈 VPC（IPv4/IPv6） {#create-dualstack-vpc}

> 仅当集群要建 **IPv4/IPv6 双栈**（`NetworkType=VPC-CNI` + `IsDualStack=true`）时才需本步。单栈 IPv4 集群跳过，用上一步的 VPC 即可。双栈集群的前序资源约束：VPC 须已开 IPv6 + 子网须已分配 IPv6 CIDR，否则集群创建中途失败（见 [创建集群 — 集群 IP 类型决策树](../tke/clusters/create.md#集群-ip-类型决策树)）。

`CreateVpc` 本身不分配 IPv6 CIDR。双栈所需的 IPv6 地址段须创建 VPC 后用 `AssignIpv6CidrBlock` 分配；子网再 `AssignIpv6SubnetCidrBlock`。

#### 1. 创建 VPC（IPv4 CIDR；IPv6 下一步再分配）

```bash
tccli vpc CreateVpc --region <REGION> \
  --VpcName "<VPC_NAME>" --CidrBlock "10.0.0.0/16"
# expected: exit 0，返回 Vpc.VpcId
#
# 可选：--EnableRouteVpcPublishIpv6 控制 VPC 关联云联网时的 IPv6 路由发布策略
# （true=CIDR 路由发布，须工单加白名单；false=子网路由发布，创建默认）。
# 该标志**不等于**「给 VPC 开通 IPv6 地址段」，开通地址段看下一步 AssignIpv6CidrBlock。
```

#### 2. 给 VPC 分配 IPv6 CIDR（AssignIpv6CidrBlock；每个 VPC 只能申请一个 IPv6 网段）

```bash
tccli vpc AssignIpv6CidrBlock --region <REGION> --VpcId "<VPC_ID>"
# expected: { "Ipv6CidrBlock": "2402:xxxx::/56", "RequestId": "..." } → VPC 已有 IPv6 CIDR
```

#### 3. 给子网分配 IPv6 CIDR（AssignIpv6SubnetCidrBlock，子网创建后执行）

```bash
tccli vpc AssignIpv6SubnetCidrBlock --region <REGION> \
  --VpcId "<VPC_ID>" \
  --Ipv6SubnetCidrBlocks '[{"SubnetId":"<SUBNET_ID>","Ipv6SubnetCidrBlock":"2402:xxxx::/64"}]'
# expected: { "Ipv6SubnetCidrBlockSet": [{"SubnetId":"<SUBNET_ID>","Ipv6SubnetCidrBlock":"2402:xxxx::/64"}], "RequestId": "..." }
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------|:-----|:-----|:--------|
| `<VPC_ID>` | VPC ID | 须已创建 | 上一步 `CreateVpc` 返回 `Vpc.VpcId` |
| `<SUBNET_ID>` | 子网 ID | 须在 VPC 内 | 「2. 创建子网」返回的 `SubnetId` |
| `Ipv6SubnetCidrBlock` | 子网 IPv6 CIDR | 须在 VPC 的 IPv6 CIDR `/56` 范围内，子网用 `/64` | 自取（如 `2402:xxxx::/64`） |

> ⚠️ **顺序约束**：`AssignIpv6SubnetCidrBlock` 须在「2. 创建子网」之后执行（须先有子网 ID）。故双栈集群的完整准备顺序是：创建 VPC → 分配 VPC IPv6 CIDR（`AssignIpv6CidrBlock`）→ 创建子网 → 分配子网 IPv6 CIDR → 创建集群(双栈)。`Ipv6CidrBlock` 由腾讯云分配（非自取），子网 IPv6 CIDR 须落在 VPC 的 `/56` 内。

### 2. 创建子网 {#2-创建子网}

```bash
tccli vpc CreateSubnet --region <REGION> \
  --VpcId "<VPC_ID>" --SubnetName "<SUBNET_NAME>" --CidrBlock "10.0.1.0/24" --Zone "<ZONE>"
# expected: exit 0，返回 Subnet.SubnetId
```
```json
{
    "Subnet": {
        "SubnetId": "subnet-example",
        "VpcId": "vpc-example",
        "CidrBlock": "10.0.1.0/24"
    },
    "RequestId": "req-example"
}
```

### 3. 创建安全组并写规则（节点/端点常用）

> 新建安全组默认入站/出站全拒绝。写规则时 **`CreateSecurityGroupPolicies` 一次调用不能同时传 `Egress` 与 `Ingress`**，否则 `InvalidParameter.Coexist`（消息：请求中不支持同时传入参数 `Egress and Ingress`）。**分两次**调用：先 Egress，再 Ingress（或反过来）。

#### 3a. 创建安全组

```bash
tccli vpc CreateSecurityGroup --region <REGION> \
  --GroupName "<SG_NAME>" --GroupDescription "tke nodes"
# expected: SecurityGroup.SecurityGroupId
```

#### 3b. 出站（单独一次）

```bash
tccli vpc CreateSecurityGroupPolicies --region <REGION> \
  --SecurityGroupId "<SECURITY_GROUP_ID>" \
  --SecurityGroupPolicySet '{
    "Egress":[
      {"Protocol":"ALL","Port":"ALL","CidrBlock":"0.0.0.0/0","Action":"ACCEPT","PolicyDescription":"all-out"}
    ]
  }'
# expected: RequestId
```

#### 3c. 入站（单独一次；默认只放通 VPC 内通信）

```bash
tccli vpc CreateSecurityGroupPolicies --region <REGION> \
  --SecurityGroupId "<SECURITY_GROUP_ID>" \
  --SecurityGroupPolicySet '{
    "Ingress":[
      {"Protocol":"ALL","Port":"ALL","CidrBlock":"<VPC_CIDR>","Action":"ACCEPT","PolicyDescription":"vpc-all"}
    ]
  }'
# expected: RequestId
```

> 默认路径不向公网开放 NodePort 范围。确需公网业务访问时，优先通过 CLB/Ingress 暴露；若必须直连节点端口，只按需开放实际端口，并把 `CidrBlock` 限制为可信出口 CIDR（例如办公出口 `<TRUSTED_CIDR>`），不要将 `30000-32767` 整段开放给 `0.0.0.0/0`。

| 占位符 | 含义 |
|:-------|:-----|
| `<SG_NAME>` | 安全组名 |
| `<SECURITY_GROUP_ID>` | `sg-xxxxxxxx` |
| `<VPC_CIDR>` | 与 VPC 一致，如 `10.0.0.0/16` |

节点安全组放通要点见 [创建节点池 — 安全组](../tke/nodes/nodepool-create.md#安全组节点加入前)；完整默认表见 [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084)。

## 验证 {#验证}

从四个维度确认 VPC + 子网就绪：

```bash
# 维度 1: VPC 存在且 CIDR 正确
tccli vpc DescribeVpcs --region <REGION> \
  --VpcIds '["<VPC_ID>"]' \
  --filter "VpcSet[0].{id:VpcId,cidr:CidrBlock,name:VpcName}" --output json
# expected: 返回创建的 VPC，CidrBlock=10.0.0.0/16
```

```bash
# 维度 2: 子网存在且可用 IP 充足（≥10 才能创建集群）
tccli vpc DescribeSubnets --region <REGION> \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[0].{id:SubnetId,cidr:CidrBlock,avail:AvailableIpAddressCount}" --output json
# expected: 含创建的子网，AvailableIpAddressCount ≥ 240
```

```bash
# 维度 3: 子网所在可用区支持 TKE（ZoneState=AVAILABLE）
tccli cvm DescribeZones --region <REGION> \
  --filter "ZoneSet[?Zone=='<ZONE>'].{zone:Zone,state:ZoneState}" --output json
# expected: state=AVAILABLE
```

```bash
# 维度 4: 子网网段是 VPC CIDR 的子段（无冲突）
tccli vpc DescribeSubnets --region <REGION> \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[0].CidrBlock" --output text
# expected: 10.0.1.0/24（是 VPC CIDR 10.0.0.0/16 的子段）
```

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `InvalidParameter.VpcCidrConflict` | `tccli vpc DescribeVpcs` 看已有 CIDR | VPC CIDR 与已有重叠 | 换 CIDR（如 `10.1.0.0/16`） |
| `InvalidParameter.SubnetConflict` | `tccli vpc DescribeSubnets --region <REGION>` | 子网 CIDR 与同 VPC 子网重叠 | 换子网段（如 `10.0.2.0/24`） |
| `InvalidParameter.Coexist`（Egress and Ingress） | 查 `CreateSecurityGroupPolicies` 入参 | 同一次请求同时传了出站与入站 | **拆成两次**调用：先只 `Egress`，再只 `Ingress` |
| `InvalidZone.ZoneNotAvailable` | `tccli cvm DescribeZones --region <REGION> --filter "ZoneSet[?ZoneState=='AVAILABLE'].Zone" --output text` | Zone 不存在或不支持 | 用返回的可用 Zone |
| `UnauthorizedOperation.CamNoAuth` | 查子账号权限 | 无 vpc 权限 | CAM 授权 `QcloudVPCFullAccess` |

## 清理

```bash
# 删子网（须先删子网再删 VPC）
tccli vpc DeleteSubnet --region <REGION> --SubnetId "<SUBNET_ID>"
# expected: exit 0

# 删 VPC
tccli vpc DeleteVpc --region <REGION> --VpcId "<VPC_ID>"
# expected: exit 0
```

> ⚠️ 删 VPC 前须先清理其下所有子网、路由表（默认除外）、安全组等资源，否则报 `ResourceInUse`。

## 收尾确认

```bash
# 端到端确认：一次查询核对子网 ID、VPC ID、可用区、CIDR 与可用 IP 是否齐备
tccli vpc DescribeSubnets --region <REGION> \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[0].{subnet:SubnetId,vpc:VpcId,zone:Zone,cidr:CidrBlock,avail:AvailableIpAddressCount}" --output text
# expected: 返回创建的子网行，avail ≥ 10，zone 为目标可用区，三要素齐备

# 下一步前置：VPC + 子网可进入创建集群（CreateCluster 必传 VpcId/SubnetId 均就绪）
tccli tke DescribeRegions --filter "TotalCount" --output text
# expected: 数字（如 19；随账号/产品开通变化）→ TKE 域可达，VPC+子网就绪
```

VPC 存在 + 子网可用 IP ≥ 10 + 可用区支持 TKE = 网络底座三要素齐备，满足 `CreateCluster` 的 `VpcId`/`SubnetId` 前置要求，可进入 [创建集群](../quickstart/tke-first-cluster.md)。

## 下一步

- [TKE 快速入门](../quickstart/tke-first-cluster.md) — VPC 就绪后创建集群
- [配置凭证](credentials.md) — 若尚未配凭证
