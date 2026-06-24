# 容器服务安全组设置（tccli）

> 对照官方：[容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084) · page_id `9084`

## 概述

TKE 集群创建时自动生成默认安全组规则，用于放通节点间通信和集群管理流量。通过 `DescribeClusterSecurity` 查看集群安全配置（含 kubeconfig、安全组 ID），通过 VPC API 查询和管理安全组规则。本页演示如何用 tccli 查询演示集群的安全组并核对默认放通规则。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，仅通过 tccli 操作控制面
- 完成 [环境准备](../../环境准备.md)，`tccli configure` 已设置 secretId/secretKey/region
- 当前账号具备 `tke:DescribeClusterSecurity`、`vpc:DescribeSecurityGroupPolicies`、`vpc:DescribeSecurityGroups` 权限

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群安全配置（含安全组 ID） | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 按名称搜索安全组 | `tccli vpc DescribeSecurityGroups --region ap-guangzhou --filters '[...]'` | 是 |
| 查看安全组规则 | `tccli vpc DescribeSecurityGroupPolicies --region ap-guangzhou --SecurityGroupId sg-xxxxxxxx` | 是 |
| 新增安全组规则 | `tccli vpc CreateSecurityGroupPolicies --region ap-guangzhou` | 否 |
| 删除安全组规则 | `tccli vpc DeleteSecurityGroupPolicies --region ap-guangzhou` | 否 |

## 操作步骤

### 1. 查看集群安全信息

`DescribeClusterSecurity` 返回集群的安全组 ID（`SecurityGroupIds`）、kubeconfig 及访问域名。

```bash
tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "UserName": "admin",
    "Domain": "cls-xxxxxxxx.ccs.tencent-cloud.com",
    "ClusterExternalEndpoint": "",
    "SecurityGroupIds": [
        "sg-myau6jjq"
    ],
    "Kubeconfig": "apiVersion: v1\nkind: Config\n...",
    "RequestId": "b1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

记录返回的 `SecurityGroupIds` 中的安全组 ID（示例 `sg-myau6jjq`），后续步骤使用。

### 2. 按名称复核安全组

```bash
tccli vpc DescribeSecurityGroups --region ap-guangzhou \
    --filters '[{"Name":"security-group-name","Values":["cls-xxxxxxxx"]}]' --output json
```

```json
{
    "SecurityGroupSet": [
        {
            "SecurityGroupId": "sg-myau6jjq",
            "SecurityGroupName": "cls-xxxxxxxx",
            "ProjectId": "0",
            "IsDefault": false,
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "TotalCount": 1,
    "RequestId": "c1c2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 3. 查看安全组规则

```bash
tccli vpc DescribeSecurityGroupPolicies --region ap-guangzhou \
    --SecurityGroupId sg-myau6jjq --output json
```

```json
{
    "SecurityGroupPolicySet": {
        "Ingress": [
            {
                "Action": "ACCEPT",
                "CidrBlock": "10.0.0.0/16",
                "Port": "ALL",
                "Protocol": "ALL",
                "PolicyDescription": "node-to-node"
            },
            {
                "Action": "ACCEPT",
                "CidrBlock": "0.0.0.0/0",
                "Port": "22",
                "Protocol": "TCP",
                "PolicyDescription": "ssh"
            },
            {
                "Action": "ACCEPT",
                "CidrBlock": "0.0.0.0/0",
                "Port": "30000-32767",
                "Protocol": "TCP",
                "PolicyDescription": "nodeport"
            }
        ],
        "Egress": [
            {
                "Action": "ACCEPT",
                "CidrBlock": "0.0.0.0/0",
                "Port": "ALL",
                "Protocol": "ALL",
                "PolicyDescription": "allow-all-out"
            }
        ]
    },
    "RequestId": "d1d2d3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 4. 默认安全组规则参考

| 方向 | 协议 | 端口 | 来源 | 用途 |
|------|------|------|------|------|
| 入站 | ALL | ALL | 安全组自身 / 容器网络 CIDR | 节点间通信 |
| 入站 | TCP | 22 | 0.0.0.0/0 | SSH |
| 入站 | TCP | 30000-32767 | 0.0.0.0/0 | NodePort |
| 出站 | ALL | ALL | 0.0.0.0/0 | 出站全放通 |

> 独立集群（Independent Cluster）Master 与 Node 通信还需放通 TCP 60001、60002、10250、2380、2379、53、17443、50055、443、61678。

## 验证

```bash
# 确认集群仍关联安全组
tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --filter "SecurityGroupIds"
```

```json
{
    "SecurityGroupIds": [
        "sg-myau6jjq"
    ],
    "RequestId": "e1e2e3e4-f5f6-7890-abcd-ef1234567890"
}
```

```bash
# 确认 NodePort 放通规则存在
tccli vpc DescribeSecurityGroupPolicies --region ap-guangzhou \
    --SecurityGroupId sg-myau6jjq --filter "SecurityGroupPolicySet.Ingress" \
    | jq '.SecurityGroupPolicySet.Ingress[] | select(.Port=="30000-32767")'
```

```json
{
    "Action": "ACCEPT",
    "CidrBlock": "0.0.0.0/0",
    "Port": "30000-32767",
    "Protocol": "TCP",
    "PolicyDescription": "nodeport"
}
```

## 清理

无需清理。本页仅执行只读查询，未修改任何资源。

> 勿随意删除默认安全组规则，可能影响集群节点间通信和 NodePort 访问。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterSecurity` 返回空 `SecurityGroupIds` | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` 确认集群存在 | 集群为独立集群且未配置安全组，或权限不足 | 在控制台「基本信息」→「安全组」手动关联安全组 |
| `DescribeSecurityGroupPolicies` 返回空 | 确认安全组 ID 正确，且安全组确属该集群 | 安全组 ID 错误，或规则已被手动清空 | 从 `DescribeClusterSecurity` 重新获取 `SecurityGroupIds` |
| 节点无法通信 | 检查入站规则是否放通容器网络 CIDR 和集群网络 CIDR 的 ALL 协议 | 入站规则缺少网段放通 | 用 `CreateSecurityGroupPolicies` 补充网段放通规则 |
| NodePort Service 外网无法访问 | 检查入站规则 TCP 30000-32767 是否对 `0.0.0.0/0` 放通 | NodePort 端口段未放通 | 补充 TCP 30000-32767 入站规则 |
| 独立集群 Master 与 Node 通信异常 | 检查 Master 安全组入站规则是否包含 60001/60002/10250/2380/2379/53/17443/50055/443/61678 | 管理端口未放通 | 补充对应端口的入站规则 |

## 下一步

- [集群安全增强能力](../集群安全增强能力/tccli%20操作.md) — 一键开启集群级安全加固
- [策略管理（OPA/Gatekeeper）](../应用安全/策略管理/tccli%20操作.md) — 准入控制与删除保护
- [使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密](../控制平面安全/使用腾讯云密钥管理系统%20KMS%20进行%20ETCD%20数据加密/tccli%20操作.md) — etcd Secret 加密

## 控制台替代

[TKE 控制台 → 集群详情](https://console.cloud.tencent.com/tke2/cluster) → 基本信息 → 安全组 → 查看规则。控制台支持可视化查看和编辑入站/出站规则，并显示规则与集群组件的关联说明。
