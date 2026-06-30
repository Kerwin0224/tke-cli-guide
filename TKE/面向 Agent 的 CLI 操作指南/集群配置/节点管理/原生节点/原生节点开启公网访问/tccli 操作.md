# 原生节点开启公网访问（tccli + kubectl）

> 对照官方：[原生节点开启公网访问](https://cloud.tencent.com/document/product/457/82334) · page_id `82334`

## 概述

为原生节点池配置公网访问能力：通过 `ModifyClusterNodePool` 配置 `InternetAccessible` 参数（公网带宽、计费模式、地址类型等），同时配合安全组配置放行入站流量。公网带宽配置仅对**新增节点**生效，存量节点不支持修改公网开启状态。

公网访问配置包含两个层面：控制面通过 tccli 修改节点池的 `InternetAccessible` 配置；数据面的安全组规则决定实际网络可达性。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:ModifyClusterNodePool, tke:DescribeClusterNodePoolDetail
#    vpc:DescribeSecurityGroups, vpc:DescribeSecurityGroupPolicies
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表

# 4. 检查 kubectl 可用
kubectl version --client
# expected: Client Version 显示版本号
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 5. 确认节点池存在
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: LifeState 为 "normal"，Type 为 "Native"

# 6. 确认安全组
tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["SECURITY_GROUP_ID"]'
# expected: 返回安全组详情

# 7. 确认 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 Kubeconfig 内容
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池公网配置 | `tccli tke DescribeClusterNodePoolDetail` | 是 |
| 开启公网访问 | `tccli tke ModifyClusterNodePool` — `InternetAccessible` | 否 |
| 查看安全组规则 | `tccli vpc DescribeSecurityGroupPolicies` | 是 |
| 编辑安全组规则 | `tccli vpc CreateSecurityGroupPolicies` / `DeleteSecurityGroupPolicies` | 否 |

### InternetAccessible 字段说明

| 字段 | 类型 | 取值与约束 | 说明 |
|------|------|------|------|
| `InternetChargeType` | String | `TRAFFIC_POSTPAID_BY_HOUR` / `BANDWIDTH_POSTPAID_BY_HOUR` / `BANDWIDTH_PACKAGE` | 公网计费模式 |
| `InternetMaxBandwidthOut` | Int | 1-2000（Mbps） | 公网出带宽上限 |
| `AddressType` | String | `EIP` / `HighQualityEIP` | EIP 类型：普通 BGP 或精品 BGP |
| `AddressId` | String | 已有的 EIP ID | 绑定已有 EIP（可选） |
| `PublicIpAssigned` | Bool | `true` / `false` | 是否分配公网 IP |

## 操作步骤

### 步骤1：控制面 — 为节点池配置公网访问

#### 选择依据

- **`InternetChargeType`**：按流量计费（`TRAFFIC_POSTPAID_BY_HOUR`）适合流量波动大的场景；按带宽计费（`BANDWIDTH_POSTPAID_BY_HOUR`）适合带宽稳定的场景。
- **`AddressType`**：普通 BGP EIP 成本较低，精品 BGP EIP 跨境访问延迟更低。
- **仅对新增节点生效**：修改后的公网配置只应用于后续扩容创建的新节点，存量节点公网状态不变。

`enable-public-access.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "InternetAccessible": {
    "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
    "InternetMaxBandwidthOut": 100,
    "AddressType": "EIP",
    "PublicIpAssigned": true
  }
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://enable-public-access.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤2：控制面 — 配置安全组入站规则

公网访问还需安全组放行入站流量。创建安全组规则允许 SSH（端口 22）或应用端口：

`create-sg-policies.json`：

```json
{
  "SecurityGroupId": "SECURITY_GROUP_ID",
  "SecurityGroupPolicySet": {
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "22",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "Allow SSH from public"
      },
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "Allow HTTP from public"
      }
    ]
  }
}
```

```bash
tccli vpc CreateSecurityGroupPolicies --region <Region> \
    --cli-input-json file://create-sg-policies.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤3：控制面 — 确认节点池公网配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 新扩容的节点会携带公网配置
```

**预期输出（含 InternetAccessible）**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "InternetAccessible": {
                "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
                "InternetMaxBandwidthOut": 100,
                "AddressType": "EIP",
                "PublicIpAssigned": true
            },
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 2}
            },
            "RequestId": "xxxx-xxxx-xxxx-xxxx"
        }
    }
}
```

### 步骤4：数据面 — 确认节点公网 IP

```bash
# 查看节点的公网 IP（将 Kubeconfig 写入 KUBECONFIG_PATH 后执行）
kubectl --kubeconfig KUBECONFIG_PATH get nodes -o wide
# expected: 新扩容节点的 EXTERNAL-IP 列显示公网 IP

# 查看节点详情中的公网地址
kubectl --kubeconfig KUBECONFIG_PATH describe node NODE_NAME
# expected: Addresses 中包含 ExternalIP
```

**预期输出**：

```text
NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE
node-xxx-xxx-1    Ready    <none>   5m    v1.30.0   10.0.0.10     1.2.3.4       TencentOS Server 4
```

### 步骤5：数据面 — 验证公网连通性

在外部网络环境执行：

```bash
# 测试公网 SSH 连通性
ssh -i ~/.ssh/KEY_ID root@EXTERNAL_IP
# expected: 能正常登录节点

# 测试 HTTP 端口（如节点上运行了 Web 服务）
curl http://EXTERNAL_IP:80
# expected: 返回 HTTP 响应
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点池公网配置 | `DescribeClusterNodePoolDetail` → `InternetAccessible` | 包含新配置的计费模式、带宽、地址类型 |
| 安全组规则 | `vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId SECURITY_GROUP_ID` | 包含新添加的入站规则 |
| 节点公网 IP | `DescribeClusterInstances` → 各节点 `PublicIpAddresses` | 新节点包含公网 IP |

### 数据面（kubectl）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点公网 IP | `kubectl get nodes -o wide` → `EXTERNAL-IP` | 新节点显示公网 IP |
| 节点详情 | `kubectl describe node NODE_NAME` → `ExternalIP` | 包含公网 IP 地址 |
| 公网连通性 | `ssh -i KEY_PATH root@EXTERNAL_IP` | 成功登录 |
| 端口可达 | `nc -zv EXTERNAL_IP PORT` 或 `telnet EXTERNAL_IP PORT` | 成功 |

## 清理

### 控制面（tccli）

```bash
# 删除安全组入站规则（如需）
tccli vpc DeleteSecurityGroupPolicies --region <Region> \
    --SecurityGroupId SECURITY_GROUP_ID \
    --SecurityGroupPolicySet '{"Ingress":[{"Protocol":"TCP","Port":"22","CidrBlock":"0.0.0.0/0","Action":"ACCEPT"},{"Protocol":"TCP","Port":"80","CidrBlock":"0.0.0.0/0","Action":"ACCEPT"}]}'
# expected: exit 0

# 关闭节点池公网配置（修改 InternetAccessible）
# 将 PublicIpAssigned 设为 false 即可
```

### 数据面（kubectl）

无额外清理。存量节点公网状态不可逆，需删除节点重新创建（不带公网配置）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.InternetMaxBandwidthOut` | 确认带宽值在 1-2000 范围内 | 带宽值超出限制 | 将 `InternetMaxBandwidthOut` 设置在 1-2000 之间 |
| `CreateSecurityGroupPolicies` 返回 `InvalidParameterValue` — SecurityGroupId | `tccli vpc DescribeSecurityGroups --region <Region>` | 安全组不存在或 Region 不匹配 | 确认 SECURITY_GROUP_ID 正确 |
| `ModifyClusterNodePool` 的 `InternetAccessible` 不生效 | 确认节点池 `LifeState` 为 `normal` | 节点池状态异常或当前有操作在进行 | 等待节点池恢复正常后再修改 |

### 配置成功但网络不通

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点无公网 IP | `kubectl get nodes -o wide`，EXTERNAL-IP 列为空 | 公网配置仅对新增节点生效，存量节点不变 | 扩容新节点以应用公网配置，或退还存量节点后重新创建 |
| 公网 IP 可达但端口不通 | `nc -zv EXTERNAL_IP PORT` 测试端口 | 安全组未放行端口或节点内防火墙阻止 | 检查安全组入站规则，检查节点内 iptables/firewalld |
| 精品 EIP 跨境访问延迟高 | `traceroute EXTERNAL_IP` | 未选择精品 BGP 线路 | 将 `AddressType` 修改为 `HighQualityEIP` 后扩容新节点 |
| 公网带宽费用异常 | `tccli vpc DescribeAddresses --region <Region>` 查看 EIP 计费 | 未选择合适的 `InternetChargeType` | 确认计费模式：流量型按量（`TRAFFIC_POSTPAID_BY_HOUR`） vs 带宽型按量（`BANDWIDTH_POSTPAID_BY_HOUR`） |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [原生节点登录方式](../原生节点登录方式/tccli%20操作.md)
- [容器服务安全组设置](../../../../安全和稳定性/容器服务安全组设置/tccli%20操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 详情 → 公网访问](https://console.cloud.tencent.com/tke2/cluster)
