# VPC-CNI 模式与其他云资源、IDC 互通

> 对照官方：[VPC-CNI 模式与其他云资源、IDC 互通](https://cloud.tencent.com/document/product/457/50359) · page_id `50359`

## 概述

VPC-CNI 模式下 Pod 直接获取 VPC 弹性网卡 IP，处于 VPC 网络平面，因此与 VPC 内其他云资源（CVM、CDB、CLB、CMQ 等）和通过专线/VPN 连接的 IDC 默认直接互通，无需额外的 NAT 转换或路由配置。连通性的唯一屏障是**安全组**和**网络 ACL**。

### 互通矩阵

| 连接方向 | 可达性 | 条件 |
|---------|:----:|------|
| Pod → 同 VPC CVM | 直接可达 | 安全组放行 Pod IP 或 Pod 所在子网 |
| Pod → 同 VPC CDB（云数据库） | 直接可达 | CDB 安全组放行 Pod IP 或 Pod 子网 |
| Pod → 同 VPC CLB（负载均衡） | 直接可达 | 无需额外配置 |
| Pod → 跨 VPC 资源 | 对等连接/CCN 后可达 | 对等连接或 CCN 打通且安全组放行 |
| IDC → Pod | 直接可达 | IDC 路由含 Pod 子网网段，专线/VPN 连通，安全组放行 |
| Pod → IDC | 直接可达 | 专线/VPN 已共享 Pod 子网路由，IDC 侧安全策略放行 |

### 与 GR 模式的区别

| 维度 | GR 模式 | VPC-CNI 模式 |
|------|---------|-------------|
| Pod IP 来源 | 独立容器网段（ClusterCIDR） | VPC 子网 IP |
| VPC 内资源互通 | 需要 NAT/路由 | 直接互通 |
| IDC 互通 | 需在专线/VPN 中额外发布容器网段 | Pod 子网路由随 VPC 自动发布 |
| 安全策略粒度 | 节点级（共享节点安全组） | Pod 级（支持独立 Pod 安全组） |

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。
- 集群已开启 VPC-CNI 模式（`ClusterNetworkSettings.Cni = true`）。
- CAM 权限：`tke:DescribeClusters`、`vpc:DescribeSubnets`、`vpc:DescribeSecurityGroups`、`vpc:DescribeNetworkAcls`。

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId、secretKey、region 均已配置

# 确认 VPC-CNI 模式已启用
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: Cni = true
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
| 查看 Pod 子网 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` → `Subnets` | 是 |
| 查看 VPC 子网详情 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | 是 |
| 查看安全组 | `tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["<SgId>"]'` | 是 |
| 查看安全组规则 | `tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId <SgId>` | 是 |
| 查看网络 ACL | `tccli vpc DescribeNetworkAcls --region <Region> --NetworkAclIds '["<AclId>"]'` | 是 |
| 创建安全组规则 | `tccli vpc CreateSecurityGroupPolicies --region <Region>` | 否 |
| 配置 Pod 独立安全组（kubectl） | `kubectl annotate pod <PodName> tke.cloud.tencent.com/security-groups='["<SgId>"]'` | 否 |

### 安全组规则字段参考

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `Protocol` | String | 协议类型 | `TCP`、`UDP`、`ICMP`、`ALL` |
| `Port` | String | 端口范围 | `80`、`443`、`30000-32767` |
| `CidrBlock` | String | 源/目标 CIDR | Pod 子网 CIDR，如 `10.100.0.0/16` |
| `Action` | String | 策略 | `ACCEPT`、`DROP` |
| `PolicyDescription` | String | 规则描述 | `VPC-CNI Pod 互通` |

### 网络 ACL 规则字段参考

| 字段 | 类型 | 说明 |
|------|------|------|
| `Protocol` | String | 协议类型 |
| `Port` | String | 端口范围 |
| `CidrBlock` | String | 源/目标 CIDR |
| `Action` | String | `ACCEPT` 或 `DROP` |
| `NetworkAclDirection` | String | `INGRESS`（入站）或 `EGRESS`（出站） |

## 操作步骤

### 查看 Pod 子网信息

```bash
# 获取集群 Pod 子网列表
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterNetworkSettings.Subnets 含 Pod 子网 ID 列表

# 查看子网 CIDR 和可用区
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# expected: SubnetSet 含 CidrBlock、Zone、AvailableIpAddressCount
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

### 查看安全组配置

```bash
# 列出 VPC 下所有安全组
tccli vpc DescribeSecurityGroups --region <Region>
# expected: SecurityGroupSet 含安全组列表

# 查看指定安全组规则
tccli vpc DescribeSecurityGroupPolicies --region <Region> \
    --SecurityGroupId <SgId>
# expected: Ingress/egress 规则列表
```

### 配置安全组放行 Pod 子网

当 VPC 内资源（CVM、CDB 等）需被 Pod 访问时，在目标资源的安全组中添加入站规则，放行 Pod 子网 CIDR：

```bash
# 为 CVM 所在安全组添加入站规则，放行 Pod 子网
tccli vpc CreateSecurityGroupPolicies --region <Region> \
    --SecurityGroupId <SgId> \
    --SecurityGroupPolicySet '{
        "Ingress": [
            {
                "Protocol": "TCP",
                "Port": "3306",
                "CidrBlock": "<PodSubnetCidr>",
                "Action": "ACCEPT",
                "PolicyDescription": "VPC-CNI Pod subnet access to CDB"
            }
        ]
    }'
```

### 查看网络 ACL 配置

```bash
tccli vpc DescribeNetworkAcls --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: NetworkAclSet 含 ACL 列表
```

> 网络 ACL 作用于子网级别，若 Pod 子网关联了 ACL，需确认规则允许所需流量通过。

### IDC 互通要点

VPC-CNI 模式下 Pod 子网属于 VPC CIDR 的一部分，当 VPC 通过专线或 VPN 与 IDC 打通时：

1. **无需额外发布 Pod 子网路由**：Pod 子网路由随 VPC 路由自动传播
2. **IDC 侧路由**：确认 IDC 路由表中包含 Pod 子网网段（通常与 VPC CIDR 一起发布）
3. **安全策略**：IDC 侧防火墙/安全策略需放行 Pod 子网

```bash
# 确认 VPC 路由表已正确关联专线/VPN
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: RouteSet 含 VPC CIDR → 专线/VPN 网关的路由
```

## 验证

### 控制面验证

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| VPC-CNI 已启用 | `DescribeClusters → Cni` | `true` |
| Pod 子网 CIDR | `DescribeSubnets → CidrBlock` | 按规划配置 |
| 安全组规则 | `DescribeSecurityGroupPolicies` | 入站规则含 Pod 子网 |
| 路由表（IDC 场景） | `DescribeRouteTables` | VPC CIDR 含 Pod 子网，已指向专线/VPN |

## 清理

仅查看查询无需清理。若曾添加测试性安全组规则，通过 `DeleteSecurityGroupPolicies` 删除对应规则。

```bash
# 删除安全组规则
tccli vpc DeleteSecurityGroupPolicies --region <Region> \
    --SecurityGroupId <SgId> \
    --SecurityGroupPolicySet '{
        "Ingress": [
            {
                "Protocol": "TCP",
                "Port": "3306",
                "CidrBlock": "<PodSubnetCidr>",
                "Action": "ACCEPT"
            }
        ]
    }'
```

## 排障

### 命令级错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` Unauthorized | `tccli configure list` 检查凭据 | CAM 权限不足 | 联系主账号授予 `tke:DescribeClusters` |
| `DescribeSubnets` InvalidParameter | 核对子网 ID | 子网 ID 格式错误或地域不匹配 | 先 `DescribeSubnets --region <Region>` 查询全部子网 |
| `DescribeSecurityGroupPolicies` 空 | 检查安全组 ID | 安全组 ID 错误 | 先 `DescribeSecurityGroups` 确认存在的安全组 |

### 网络连通性故障

| 现象 | 诊断方法 | 可能原因 | 修复步骤 |
|------|---------|---------|---------|
| Pod 无法访问 VPC 内 CVM | 检查安全组 | CVM 安全组未放行 Pod 子网 | 在 CVM 安全组中添加入站规则放行 Pod 子网 CIDR |
| Pod 无法访问 CDB | 检查 CDB 安全组 | CDB 安全组未放行 Pod 子网 | 在 CDB 安全组添加 Pod 子网入站规则 |
| IDC 无法访问 Pod | 检查 IDC 侧路由表 | IDC 路由表不含 Pod 子网网段 | 在 IDC 专线/VPN 路由中添加 Pod 子网，确认路由已同步 |
| Pod 无法访问 IDC | 检查 VPC 路由表 | VPC 路由表无 IDC 回程路由 | 确认专线/VPN 已共享 IDC 路由 |
| 子网级阻断 | 检查网络 ACL | Pod 子网 ACL 有 DROP 规则 | 修改 ACL 规则放行所需流量 |

## 下一步

- [VPC-CNI 模式安全组使用说明](../VPC-CNI%20模式安全组使用说明/tccli%20操作.md) -- Pod 独立安全组配置
- [Pod 直接绑定弹性公网 IP 使用说明](../Pod%20直接绑定弹性公网%20IP%20使用说明/tccli%20操作.md) -- EIP 绑定
- [集群管理安全组](../集群管理安全组/tccli%20操作.md) -- 安全组管理

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 集群详情页「基本信息」→「网络」查看 VPC-CNI 网络信息；[VPC 控制台 → 安全组](https://console.cloud.tencent.com/vpc/securitygroup) 管理安全组规则；[VPC 控制台 → 网络 ACL](https://console.cloud.tencent.com/vpc/acl) 管理 ACL 规则。
