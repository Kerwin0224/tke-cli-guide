---
doc_type: How-to
subtype: 6A
fused: false
---
# 准备 VPC 与子网

> TKE 集群必须部署在 VPC 内。若你已有 VPC+子网，可跳过本文，用 `tccli vpc DescribeSubnets` 验证。本文给"从零创建一个最小可用 VPC+子网"的闭环。
> VPC 是腾讯云网络服务，完整能力见 [VPC 产品文档](https://cloud.tencent.com/document/product/215)。

## 概述

TKE 集群节点从子网分配内网 IP，Pod/Service 用 VPC CIDR 通信。创建集群前需备好：1 个 VPC + 1 个可用子网（可用 IP ≥ 10）。

## 触发条件

- 需创建 TKE 集群但账号下还没有 VPC 或可用子网（`CreateCluster` 必传 `VpcId`）— 用本文从零建一个最小 VPC+子网
- 已有 VPC+子网但不确定是否可用（可用 IP 是否 ≥10、CIDR 是否冲突）— 跳到 [验证](#验证) 段核对

## 决策依据

| 项 | 选项 | 推荐 |
|:---|:-----|:-----|
| VPC CIDR | `10.0.0.0/16`（65536 IP）/ `192.168.0.0/16` | `10.0.0.0/16`（与 IDC 冲突少） |
| 子网 CIDR | VPC CIDR 的子段，如 `10.0.1.0/24`（254 IP） | `10.0.1.0/24` |
| 可用区 | `ap-guangzhou-3` 等 | 选离你近的（`tccli cvm DescribeZones` 查看） |

> CIDR 不可与已有 VPC 重叠。集群创建后 VPC CIDR 无法更改，子网可后加。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 3.1.117.1 或更高

tccli cvm DescribeRegions --filter "TotalCount" --output text
# expected: 数字（如 49）→ 凭证有效（凭证配置见 [配置凭证](credentials.md)）
```

### 资源检查

```bash
# 查可用区（cvm 服务提供，vpc 服务无 DescribeZones）
tccli cvm DescribeZones --region <REGION> \
  --filter "ZoneSet[0].{zone:Zone,state:ZoneState}"
# expected: AVAILABLE 状态的可用区，如 ap-guangzhou-3

# 查已有 VPC（避免 CIDR 冲突）
tccli vpc DescribeVpcs --region <REGION> \
  --filter "VpcSet[].{id:VpcId,cidr:CidrBlock,name:VpcName}" --output text
# expected: 已有 VPC 列表（核对 10.0.0.0/16 是否已被占用）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<ZONE>` | 可用区 | 如 `ap-guangzhou-3` | `tccli cvm DescribeZones --region <REGION> --filter "ZoneSet[?ZoneState=='AVAILABLE'].Zone" --output text` |
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

### 2. 创建子网

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

## 验证

```bash
tccli vpc DescribeSubnets --region <REGION> \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[0].{id:SubnetId,cidr:CidrBlock,avail:AvailableIpAddressCount}"
# expected: 含创建的子网，AvailableIpAddressCount ≥ 250
```

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `InvalidParameter.VpcCidrConflict` | `tccli vpc DescribeVpcs` 看已有 CIDR | VPC CIDR 与已有重叠 | 换 CIDR（如 `10.1.0.0/16`） |
| `InvalidParameter.SubnetConflict` | `tccli vpc DescribeSubnets --region <REGION>` | 子网 CIDR 与同 VPC 子网重叠 | 换子网段（如 `10.0.2.0/24`） |
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
# 一次性核对：VPC + 子网均存在，且子网可用 IP 充足
tccli vpc DescribeSubnets --region <REGION> \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[0].{subnet:SubnetId,vpc:VpcId,cidr:CidrBlock,avail:AvailableIpAddressCount}" --output text
# expected: 返回创建的子网行，avail ≥ 10，可据此进入创建集群
```

## 下一步

- [TKE 快速入门](../quickstart/tke-first-cluster.md) — VPC 就绪后创建集群
- [配置凭证](credentials.md) — 若尚未配凭证
