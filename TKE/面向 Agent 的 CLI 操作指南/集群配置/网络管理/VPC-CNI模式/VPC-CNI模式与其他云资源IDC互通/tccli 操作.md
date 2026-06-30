# VPC-CNI模式与其他云资源IDC互通（tccli）

> 对照官方：[VPC-CNI模式与其他云资源IDC互通](https://cloud.tencent.com/document/product/457/50359) · page_id `50359`

## 概述

VPC-CNI 模式下 Pod 直接使用 VPC 子网 IP，因此 Pod 可与 VPC 内的其他云资源（CVM、CLB、CDB、Redis 等）直接互通，无需经过 NAT 转换。相比 GlobalRouter 模式，VPC-CNI 的优势包括：

- **无源 NAT**：Pod 访问 VPC 资源时保留原始 Pod IP，便于基于 IP 的访问控制和安全审计
- **直接 VPC 路由**：Pod 与 VPC 资源的网络路径更短，延迟更低
- **IDC 互通**：通过 VPC 对等连接、云联网（CCN）或专线接入（Direct Connect），Pod 可直接与 IDC 内服务通信
- **跨地域互通**：配合云联网可实现跨地域 VPC 内 Pod 互通

限制：Pod IP 直接消耗 VPC 子网 IP，需要合理规划子网 CIDR；安全组规则需要覆盖 Pod 子网段。

当前集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）使用 GR 模式，VPC-CNI 未开启（EnableIPAMD: false）。VPC 为 `vpc-of73262z`（172.16.0.0/12）。

## 前置条件

- 集群已开启 VPC-CNI 网络模式
- VPC 子网规划充足，确保 Pod 有可用 IP
- 如需 IDC 互通：已建立 VPC 对等连接 / 云联网 / 专线接入
- 安全组规则已正确配置，允许 Pod 子网段与目标资源互通

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 VPC-CNI 状态 | tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx | 是 |
| 查询集群 VPC 信息 | tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' | 是 |
| 查询 VPC 子网 | tccli vpc DescribeSubnets --region ap-guangzhou --Filters '[{"Name":"vpc-id","Values":["vpc-of73262z"]}]' | 是 |
| 查询云联网实例 | tccli vpc DescribeCcns --region ap-guangzhou | 是 |
| 查询专线网关 | tccli vpc DescribeDirectConnectGateways --region ap-guangzhou | 是 |
| 查询对等连接 | tccli vpc DescribeVpcPeeringConnections --region ap-guangzhou | 是 |
| 查询安全组 | tccli vpc DescribeSecurityGroups --region ap-guangzhou | 是 |

## 操作步骤

本页为概念介绍页，无实际操作步骤。

## 验证

- `DescribeIPAMD` 确认 `EnableIPAMD: true`，集群已启用 VPC-CNI
- Pod IP（通过 `kubectl get pod -o wide` 查看）属于 VPC 子网段
- 从 Pod 内可 ping 通同 VPC 的 CVM 实例 IP
- 从 Pod 内可访问同 VPC 的 CDB/Redis 等云资源私有 IP
- 若配置了云联网/专线，从 Pod 内可访问对端 IDC 的服务 IP

## 清理

VPC-CNI 与其他云资源 IDC 互通的连接本身无需特殊清理。若需清理具体网络资源：

- 关闭 VPC-CNI（`DisableVpcCniNetworkType`）会断开 Pod 与 VPC 的直接互通
- 删除云联网/专线/对等连接等网络通道需通过 VPC 控制台或对应 API 操作
- 安全组规则修改需注意不要影响其他正在运行的业务

**警告**：`DisableVpcCniNetworkType` 会改变所有 Pod IP，需重建 Pod，可能影响业务。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法访问同 VPC 的 CVM | 从 Pod 内 ping CVM 私有 IP，检查安全组规则 | CVM 安全组未放通 Pod 子网段 | 在 CVM 安全组入站规则中添加 Pod 所在子网段的允许规则 |
| Pod 无法访问 IDC 服务 | 检查云联网/专线/对等连接状态，确认路由表配置 | VPC 路由表未配置通往 IDC 的路由，或云联网未关联 VPC | 确认 VPC 路由表中有目的网段指向云联网/专线网关的路由项 |
| Pod 无法被 VPC 内其他资源访问 | 检查 Pod 所在节点的安全组和网络 ACL | 安全组或 ACL 限制了 Pod 子网段的入站流量 | 在安全组入站规则中放通目标 Pod 子网段 |
| IDC 服务无法访问 Pod | 从 IDC 机器 traceroute 到 Pod IP，检查路由走向 | IDC 侧路由未正确指向腾讯云专线/云联网 | 检查 IDC 侧路由配置，确保 Pod 子网段路由指向云联网或专线网关 |
| GR 集群查询 VPC-CNI 状态报错 | DescribeIPAMD 返回 FailedOperation.EnableVPCCNIFailed | cluster is not vpc-cni cluster | 先调用 EnableVpcCniNetworkType 开启 VPC-CNI，再进行 VPC 互通验证 |

## 下一步

- [容器服务安全组设置](https://cloud.tencent.com/document/product/457/50360) -- page_id `50360`
- [VPC-CNI 模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster/sub/detail/basic/info?rid=1&clusterId=cls-xxxxxxxx)，在集群详情页的「基本信息」中可查看当前网络模式。VPC 互通配置（云联网、专线、对等连接等）需通过 [私有网络控制台](https://console.cloud.tencent.com/vpc/) 进行管理。
