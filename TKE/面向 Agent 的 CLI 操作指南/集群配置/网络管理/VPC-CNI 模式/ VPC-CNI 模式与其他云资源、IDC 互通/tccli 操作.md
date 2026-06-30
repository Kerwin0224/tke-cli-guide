# VPC-CNI 模式与其他云资源、IDC 互通（tccli）

> 对照官方：[VPC-CNI 模式与其他云资源、IDC 互通](https://cloud.tencent.com/document/product/457/50359) · page_id `50359`

## 概述

对照[官方 VPC-CNI 模式与其他云资源、IDC 互通](https://cloud.tencent.com/document/product/457/50359)，保留产品与限制说明，并用 **tccli** 完成可映射的 VPC 网络信息查询、CCN 发现与安全组审计步骤。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `ap-guangzhou` 与凭证已配置。
- 新建集群见 [创建集群](../../../%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86/%E5%88%9B%E5%BB%BA%E9%9B%86%E7%BE%A4/tccli%20%E6%93%8D%E4%BD%9C.md)；下载 kubeconfig 见 [连接集群](../../../集群管理/连接集群/tccli%20操作.md)。

## 控制台与 CLI 参数映射

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 同 VPC 云资源互通 | Pod 在 VPC 网段内 | `DescribeClusters` → `VpcId` + VPC 产品互通文档 | 是 |
| 访问公网 | EIP、NAT、CLB | EIP 见 [Pod 直接绑定 EIP](../Pod%20%E7%9B%B4%E6%8E%A5%E7%BB%91%E5%AE%9A%E5%BC%B9%E6%80%A7%E5%85%AC%E7%BD%91%20IP%20%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md)，NAT/CLB 见 [容器服务节点公网 IP](../../%E5%AE%B9%E5%99%A8%E6%9C%8D%E5%8A%A1%E8%8A%82%E7%82%B9%E5%85%AC%E7%BD%91%20IP%20%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md) | 是 |
| 跨 VPC 互通 | 对等连接、CCN | `DescribeVpcPeeringConnections`、`DescribeCcns` | 是 |
| 连接 IDC | VPN、专线、CCN | `DescribeVpnConnections`、`DescribeDirectConnectGateways`、`DescribeCcns` | 是 |
| VPC 连接方案 | 官方总览 | [VPC 连接方案概述](https://cloud.tencent.com/document/product/215/37053) | 是 |

## 操作步骤

VPC-CNI 容器网段由 VPC 管理，可通过 VPC 产品能力与其他云资源、IDC 互通：

- **同 VPC 内**：Pod IP 是 VPC IP，与同 VPC 的 CVM、数据库等直接路由互通。
- **访问公网**：通过 EIP、NAT 网关、CLB 等。
- **跨 VPC 通信**：通过对等连接（Peering Connection）或云联网（CCN）。
- **连接 IDC**：通过 VPN、专线或 CCN。

本页提供基于 tccli 的网络信息审计入口，具体互通配置见各 VPC 产品文档。

#### tccli：查看集群 VPC 与容器网段

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterNetworkSettings:ClusterNetworkSettings}"
```

```json
{
  "ClusterId": "<ClusterId>",
  "ClusterNetworkSettings": {
    "VpcId": "vpc-example",
    "Cni": true,
    "EnableStaticIp": true,
    "Subnets": ["subnet-example"],
    "VpcCniType": "tke-route-eni"
  }
}
```

# expected: exit 0, contains "VpcId"

#### tccli：查询 CCN 信息（跨 VPC / IDC 场景）

```bash
tccli vpc DescribeCcns --region ap-guangzhou --output json \
  --Limit 5 --filter "CcnSet[0:3].{CcnId:CcnId,CcnName:CcnName,State:State}"
```

```json
[
  {
    "CcnId": "ccn-example",
    "CcnName": "tke-vpc-ccn",
    "State": "AVAILABLE"
  },
  {
    "CcnId": "ccn-example-2",
    "CcnName": "hybrid-cloud-ccn",
    "State": "AVAILABLE"
  }
]
```

# expected: exit 0, contains "CcnId"

#### tccli：查询对等连接（跨 VPC 场景）

```bash
tccli vpc DescribeVpcPeeringConnections --region ap-guangzhou --output json \
  --filter "PeerConnectionSet[?State=='ACTIVE'] | [0:3].{PeeringConnectionId:PeeringConnectionId,PeeringConnectionName:PeeringConnectionName,SourceVpcId:SourceVpcId,DestinationVpcId:DestinationVpcId,State:State}"
```

```json
[
  {
    "PeeringConnectionId": "pcx-example",
    "PeeringConnectionName": "vpc-cni-peering",
    "SourceVpcId": "vpc-example",
    "DestinationVpcId": "vpc-example-b",
    "State": "ACTIVE"
  }
]
```

# expected: exit 0, contains "PeeringConnectionId"

#### tccli：查询 VPC 子网路由表（确认出向路由）

```bash
tccli vpc DescribeRouteTables --region ap-guangzhou --output json \
  --Filters "[{\"Name\":\"vpc-id\",\"Values\":[\"<VpcId>\"]}]" \
  --filter "RouteTableSet[0].{RouteTableId:RouteTableId,RouteCount:length(Routes[])}"
```

```json
{
  "RouteTableId": "rtb-example",
  "RouteCount": 5
}
```

# expected: exit 0, contains "RouteTableId"

#### 互通原理

| 互通方向 | VPC-CNI 原理 | 对应文档 |
|----------|-------------|----------|
| Pod → 公网 | Pod IP 经 EIP / NAT / CLB 出公网 | [访问公网](https://cloud.tencent.com/document/product/215/36697) |
| Pod ↔ 同城其他 VPC | 对等连接 + 路由表双向添加子网 | [连接其他 VPC](https://cloud.tencent.com/document/product/215/36698) |
| Pod ↔ IDC | VPN / 专线 / CCN + SPD + 路由 | [连接本地数据中心](https://cloud.tencent.com/document/product/215/36699) |

## 验证

### Control plane (tccli)

- `DescribeClusters` 返回 `VpcId` 与 VPC-CNI 子网信息，确认集群已启用 VPC-CNI。
- `DescribeCcns` / `DescribeVpcPeeringConnections` 确认互通基础设施已就绪。
- `DescribeRouteTables` 确认子网路由表配置与互通需求一致。

## 清理

本页为说明与只读查询，无额外资源。删除本地临时 kubeconfig（若曾导出）即可。

## 排障

| 现象 | 处理 |
|------|------|
| Pod 无法访问公网 | 检查 Pod 所在子网路由表是否有 NAT 网关 / EIP 路由；检查 ip-masq-agent 配置 |
| Pod 与跨 VPC 资源不通 | 确认对等连接或 CCN 已建立；确认对端安全组、ACL 已放行 Pod 子网 CIDR |
| Pod 与 IDC 不通 | 确认 VPN/专线 SPD 策略包含容器子网 CIDR；确认 IDC 侧回程路由 |
| `DescribeVpcPeeringConnections` 无结果 | 确认地域与对等连接所在区域一致 |

## 下一步

- [VPC-CNI 模式安全组使用说明](../VPC-CNI%20%E6%A8%A1%E5%BC%8F%E5%AE%89%E5%85%A8%E7%BB%84%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md) · [Pod 直接绑定 EIP 使用说明](../Pod%20%E7%9B%B4%E6%8E%A5%E7%BB%91%E5%AE%9A%E5%BC%B9%E6%80%A7%E5%85%AC%E7%BD%91%20IP%20%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md) · [VPC 连接方案概述](https://cloud.tencent.com/document/product/215/37053)

## 控制台替代

[控制台 → VPC](https://console.cloud.tencent.com/vpc) 中对等连接、云联网、VPN 连接、专线网关对应各互通场景的配置入口。
