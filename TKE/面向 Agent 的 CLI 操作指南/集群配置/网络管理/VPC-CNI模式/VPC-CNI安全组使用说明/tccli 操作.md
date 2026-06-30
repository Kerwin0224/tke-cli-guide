# VPC-CNI安全组使用说明
> 对照官方：[VPC-CNI安全组使用说明](https://cloud.tencent.com/document/product/457/50360) · page_id `50360`

## 概述

VPC-CNI 模式下 Pod 直接使用 VPC 弹性网卡（ENI）IP 地址，可通过安全组对 Pod 实施精细化的网络访问控制。安全组的生效粒度取决于 VPC-CNI 子模式：

- **共享网卡模式**（`tke-route-eni`）：同一节点上的多个 Pod 共享节点 ENI，安全组绑定在节点 ENI 级别，同节点所有 Pod 共享相同的安全组规则。
- **独占网卡模式**（`tke-direct-eni`）：每个 Pod 拥有独立的 ENI，通过 Pod Annotation `tke.cloud.tencent.com/pod-security-groups` 指定安全组 ID，实现 Pod 级网络隔离。

安全组规则通过 VPC API 管理（`tccli vpc`），绑定方式因子模式而异：共享模式通过 VPC 控制台或 API 绑定至节点 ENI；独占模式通过 kubectl 在 Pod 声明中指定。

## 前置条件

- 集群已开启 VPC-CNI 网络模式。验证：

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region> | jq '.Clusters[0].Property'
# expected: {"NetworkType":"VPC-CNI","VpcCniType":"tke-route-eni"...} 或 ..."VpcCniType":"tke-direct-eni"...
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

- [环境准备](../../../../环境准备.md)已完成，`tccli` 已配置并可调用 VPC 与 TKE 服务。验证：

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region> --output json > /dev/null && echo "OK"
# expected: OK
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

- 共享网卡模式下，需知道目标节点的 ENI ID。验证：

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region> \
  --filter "InstanceSet[0].InstanceId"
# expected: "ins-example..."
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

- 独占网卡模式下，kubectl 需可访问集群 APIServer。由于公网端点受 CAM 策略限制（`tke:clusterExtranetEndpoint` 被 deny），需在 VPC 内 CVM 或通过 VPN/专线执行 kubectl 命令。验证：

```bash
kubectl cluster-info 2>&1 | head -1
# expected: Kubernetes control plane is running at https://...
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 VPC-CNI 子模式 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看安全组列表 | `tccli vpc DescribeSecurityGroups --region <Region>` | 是 |
| 创建安全组 | `tccli vpc CreateSecurityGroup --GroupName <Name> --GroupDescription <Desc>` | 否 |
| 查看安全组规则 | `tccli vpc DescribeSecurityGroupPolicies --SecurityGroupId <SecurityGroupId>` | 是 |
| 添加安全组规则 | `tccli vpc CreateSecurityGroupPolicies --cli-input-json file://sg-policies.json` | 否 |
| 重置安全组规则 | `tccli vpc ModifySecurityGroupPolicies --cli-input-json file://sg-modify.json` | 否 |
| 删除安全组规则 | `tccli vpc DeleteSecurityGroupPolicies --cli-input-json file://sg-del-policies.json` | 否 |
| 查看节点/ENI 已绑定安全组 | `tccli vpc DescribeNetworkInterfaces --Filters '[{"Name":"attachment.instance-id","Values":["<InstanceId>"]}]'` | 是 |
| 绑定安全组至 ENI（共享模式） | `tccli vpc AssociateNetworkInterfaceSecurityGroups --NetworkInterfaceIds '["<EniId>"]' --SecurityGroupIds '["<SgId>"]'` | 否 |
| 解绑安全组（共享模式） | `tccli vpc DisassociateNetworkInterfaceSecurityGroups --NetworkInterfaceIds '["<EniId>"]' --SecurityGroupIds '["<SgId>"]'` | 否 |
| 删除安全组 | `tccli vpc DeleteSecurityGroup --SecurityGroupId <SecurityGroupId>` | 否 |
| 为 Pod 绑定安全组（独占模式） | `kubectl apply -f pod-with-sg.yaml`（Annotation `tke.cloud.tencent.com/pod-security-groups`） | 否 |

## 操作步骤

### 1. 确认集群 VPC-CNI 子模式

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "demo-cluster",
      "ClusterNetworkSettings": {
        "VpcId": "vpc-example",
        "ClusterCIDR": "172.16.0.0/16",
        "Subnets": ["subnet-example"]
      },
      "Property": "{\"NetworkType\":\"VPC-CNI\",\"VpcCniType\":\"tke-direct-eni\"}"
    }
  ],
  "RequestId": "req-example-001"
}
```

从 `Property.VpcCniType` 确认子模式：`tke-route-eni`（共享网卡）或 `tke-direct-eni`（独占网卡）。下文按子模式分述。

### 2. 创建安全组并配置规则

创建一个用于 Pod 流量的安全组，并添加入站/出站规则。

#### 2.1 创建安全组

```bash
tccli vpc CreateSecurityGroup \
  --GroupName pod-sg-<ClusterId> \
  --GroupDescription "安全组 for VPC-CNI Pod on <ClusterId>" \
  --region <Region>
```

```json
{
  "SecurityGroup": {
    "SecurityGroupId": "sg-xxxxxxxx",
    "SecurityGroupName": "pod-sg-cls-example",
    "SecurityGroupDesc": "安全组 for VPC-CNI Pod on cls-example",
    "CreatedTime": "2026-06-15 10:00:00"
  },
  "RequestId": "req-example-002"
}
```

记录输出中的 `SecurityGroupId`（下文以 `<SecurityGroupId>` 指代，示例用 `sg-example`）。

#### 2.2 添加入站规则（允许同 VPC 内 HTTP 访问）

参数多，使用 `--cli-input-json file://` 格式。创建 `sg-policies.json`：

```json
{
  "SecurityGroupId": "<SecurityGroupId>",
  "SecurityGroupPolicySet": {
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "10.0.0.0/8",
        "Action": "ACCEPT",
        "PolicyDescription": "允许 VPC 内 HTTP 入站"
      }
    ]
  }
}
```

```bash
tccli vpc CreateSecurityGroupPolicies --cli-input-json file://sg-policies.json --region <Region>
```

```json
{
  "RequestId": "req-example-003"
}
```

#### 2.3 添加出站规则（允许全部出站）

```bash
tccli vpc CreateSecurityGroupPolicies \
  --SecurityGroupId <SecurityGroupId> \
  --SecurityGroupPolicySet '{"Egress":[{"Protocol":"ALL","CidrBlock":"0.0.0.0/0","Action":"ACCEPT","PolicyDescription":"允许全部出站"}]}' \
  --region <Region>
```

```json
{
  "RequestId": "req-example-004"
}
```

#### 2.4 查看安全组规则

```bash
tccli vpc DescribeSecurityGroupPolicies --SecurityGroupId <SecurityGroupId> --region <Region>
```

```json
{
  "SecurityGroupPolicySet": {
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "10.0.0.0/8",
        "Action": "ACCEPT",
        "PolicyDescription": "允许 VPC 内 HTTP 入站",
        "PolicyIndex": 0
      }
    ],
    "Egress": [
      {
        "Protocol": "ALL",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "允许全部出站",
        "PolicyIndex": 0
      }
    ],
    "Version": "1"
  },
  "RequestId": "req-example-005"
}
```

### 3. 绑定安全组至 Pod/节点

#### 3.1 共享网卡模式 — 绑定至节点 ENI

共享模式下，安全组直接绑定至节点 CVM 的 ENI，同节点所有 Pod 继承该安全组规则。

**步骤 A — 获取节点 ENI ID：**

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region>
```

```json
{
  "TotalCount": 2,
  "InstanceSet": [
    {
      "InstanceId": "ins-example-01",
      "InstanceRole": "WORKER",
      "LanIP": "10.0.0.5",
      "InstanceState": "running"
    }
  ],
  "RequestId": "req-example-006"
}
```

```bash
tccli vpc DescribeNetworkInterfaces \
  --Filters '[{"Name":"attachment.instance-id","Values":["<InstanceId>"]}]' \
  --region <Region>
```

```json
{
  "NetworkInterfaceSet": [
    {
      "NetworkInterfaceId": "eni-example-01",
      "Primary": true,
      "MacAddress": "20:90:6F:xx:xx:xx",
      "GroupSet": ["sg-default"]
    }
  ],
  "RequestId": "req-example-007"
}
```

记录 `NetworkInterfaceId`，下文以 `<EniId>` 指代。

**步骤 B — 绑定安全组至 ENI：**

```bash
tccli vpc AssociateNetworkInterfaceSecurityGroups \
  --NetworkInterfaceIds '["<EniId>"]' \
  --SecurityGroupIds '["<SecurityGroupId>"]' \
  --region <Region>
```

```json
{
  "RequestId": "req-example-008"
}
```

**步骤 C — 验证绑定：**

```bash
tccli vpc DescribeNetworkInterfaces \
  --NetworkInterfaceIds '["<EniId>"]' \
  --region <Region> \
  --filter "NetworkInterfaceSet[0].GroupSet"
```

```json
["sg-default", "sg-example"]
```

#### 3.2 独占网卡模式 — 通过 Pod Annotation 绑定

独占模式下，每个 Pod 拥有独立的 ENI，通过 Annotation 声明安全组 ID。

创建 `pod-with-sg.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secured-nginx
  namespace: default
  annotations:
    tke.cloud.tencent.com/pod-security-groups: "<SecurityGroupId>"
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod-with-sg.yaml
```

```text
pod/secured-nginx created
```

> **注意**：独占网卡模式要求 Pod 在 `tke-direct-eni` 子网段中有可用 IP。若 Pod 一直处于 `Pending` 状态，请检查子网 IP 是否耗尽。

### 4. 修改安全组规则（以更新为例）

若需修改安全组规则，使用 `ReplaceSecurityGroupPolicies` 全量替换，或 `ModifySecurityGroupPolicies` 增量修改。以下示例通过 `--cli-input-json` 替换入站规则：

`sg-replace.json`：

```json
{
  "SecurityGroupId": "<SecurityGroupId>",
  "SecurityGroupPolicySet": {
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "443",
        "CidrBlock": "10.0.0.0/8",
        "Action": "ACCEPT",
        "PolicyDescription": "允许 VPC 内 HTTPS 入站"
      },
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "10.0.0.0/8",
        "Action": "ACCEPT",
        "PolicyDescription": "允许 VPC 内 HTTP 入站"
      }
    ],
    "Egress": [
      {
        "Protocol": "ALL",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "允许全部出站"
      }
    ]
  }
}
```

```bash
tccli vpc ReplaceSecurityGroupPolicies --cli-input-json file://sg-replace.json --region <Region>
```

```json
{
  "RequestId": "req-example-009"
}
```

## 验证

### 控制面（tccli）

验证安全组存在且规则正确：

```bash
tccli vpc DescribeSecurityGroupPolicies --SecurityGroupId <SecurityGroupId> --region <Region>
```

验证节点 ENI 已绑定安全组（共享模式）：

```bash
tccli vpc DescribeNetworkInterfaces \
  --Filters '[{"Name":"group-id","Values":["<SecurityGroupId>"]}]' \
  --region <Region>
```

```json
{
  "NetworkInterfaceSet": [
    {
      "NetworkInterfaceId": "eni-example-01",
      "GroupSet": ["sg-default", "sg-example"],
      "Primary": true
    }
  ],
  "RequestId": "req-example-010"
}
```

### 数据面（kubectl）

> **注意**：公网端点创建受 CAM 策略限制（`tke:clusterExtranetEndpoint` 被 deny），需在 VPC 内 CVM 或通过 VPN/专线执行以下 kubectl 命令。

验证独占模式下 Pod 的 Annotation 包含安全组 ID：

```bash
kubectl get pod secured-nginx -o jsonpath='{.metadata.annotations.tke\.cloud\.tencent\.com/pod-security-groups}'
```

```text
sg-example
```

验证 Pod 正在运行且分配了 VPC IP：

```bash
kubectl get pod secured-nginx -o wide
```

```text
NAME            READY   STATUS    RESTARTS   AGE   IP           NODE
secured-nginx   1/1     Running   0          2m    10.0.0.100   ins-example-01
```

共享模式下，验证同节点 Pod 共享安全组（在节点上执行）：

```bash
kubectl get pod -o wide --field-selector spec.nodeName=<NodeName>
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **副作用警告**：删除安全组后，所有绑定该安全组的资源（ENI、Pod）的安全策略立即失效，可能导致网络连通性问题。
>
> **计费警告**：安全组本身免费，不产生计费影响；ENI 可能产生少量费用。

### 数据面（kubectl）

删除独占模式下创建的测试 Pod：

```bash
kubectl delete pod secured-nginx
```

```text
pod "secured-nginx" deleted
```

### 控制面（tccli）

**步骤 1 — 解绑安全组（共享模式）：**

```bash
tccli vpc DisassociateNetworkInterfaceSecurityGroups \
  --NetworkInterfaceIds '["<EniId>"]' \
  --SecurityGroupIds '["<SecurityGroupId>"]' \
  --region <Region>
```

```json
{
  "RequestId": "req-example-011"
}
```

**步骤 2 — 删除安全组规则（可选，也可直接删安全组）：**

`sg-del-policies.json`：

```json
{
  "SecurityGroupId": "<SecurityGroupId>",
  "SecurityGroupPolicySet": {
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "10.0.0.0/8",
        "Action": "ACCEPT",
        "PolicyDescription": "允许 VPC 内 HTTP 入站"
      }
    ],
    "Egress": [
      {
        "Protocol": "ALL",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "允许全部出站"
      }
    ]
  }
}
```

```bash
tccli vpc DeleteSecurityGroupPolicies --cli-input-json file://sg-del-policies.json --region <Region>
```

**步骤 3 — 删除安全组：**

```bash
tccli vpc DeleteSecurityGroup --SecurityGroupId <SecurityGroupId> --region <Region>
```

```json
{
  "RequestId": "req-example-012"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateSecurityGroup` 返回 `LimitExceeded` | `tccli vpc DescribeSecurityGroupLimits --region <Region>` 查看配额 | 安全组数量达到地域配额上限 | 清理不再使用的安全组或提工单申请扩容 |
| `AssociateNetworkInterfaceSecurityGroups` 返回 `ResourceNotFound` | 检查 `<EniId>` 是否正确 | ENI ID 不存在或已被释放 | 使用 `tccli vpc DescribeNetworkInterfaces` 确认 ENI ID |
| `AssociateNetworkInterfaceSecurityGroups` 返回 `LimitExceeded.SecurityGroupInstanceLimit` | ENI 绑定的安全组数超限 | 单个 ENI 最多绑定 5 个安全组 | 解绑不再需要的安全组后重试 |
| `kubectl apply` 返回 `connection refused` | 检查集群公网端点状态 | 公网端点受 CAM 策略限制或未开启 | 在 VPC 内 CVM 或通过 VPN/专线执行 kubectl 命令 |
| `DeleteSecurityGroup` 返回 `ResourceInUse` | 安全组仍被 ENI/Pod 引用 | 安全组正在被资源使用，无法直接删除 | 先解绑所有引用该安全组的资源，再删除 |
| `CreateSecurityGroupPolicies` 返回 `InvalidParameterValue.CidrBlock` | 检查 `CidrBlock` 格式 | CIDR 格式不正确 | 使用合法 CIDR 格式，如 `10.0.0.0/8` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 独占模式 Pod 处于 `Pending`，Events 显示 `eni-ip insufficient` | `tccli tke DescribeClusters` 查看 `Subnets` 和 `ClusterNetworkSettings` | VPC-CNI 子网 IP 已耗尽 | 扩容子网或添加子网：`tccli tke AddVpcCniSubnets --ClusterId <ClusterId> --SubnetIds '["<SubnetId>"]'` |
| Pod Annotation 指定安全组后未生效 | `kubectl describe pod <PodName>` 检查 Annotation；`tccli tke DescribeClusters` 确认 `VpcCniType` | 集群子模式不是 `tke-direct-eni`（独占网卡） | Annotation 仅在独占网卡模式下生效；共享模式需通过 `AssociateNetworkInterfaceSecurityGroups` 绑定至节点 ENI |
| 安全组已绑定但 Pod 仍无法通信 | `tccli vpc DescribeSecurityGroupPolicies --SecurityGroupId <SecurityGroupId>` 检查规则 | 安全组规则未覆盖目标端口/协议/来源 IP | 使用 `CreateSecurityGroupPolicies` 添加缺失的规则 |
| 共享模式 Pod 流量不符合预期 | 检查节点上其他 Pod 的网络行为 | 同节点 Pod 共享 ENI 安全组，可能被非预期 Pod 流量触发规则 | 若需 Pod 级隔离，考虑切换至独占网卡模式或使用网络策略（NetworkPolicy） |
| `ModifySecurityGroupPolicies` 后规则未立即生效 | 等待 30 秒后重新验证 | 安全组规则下发到 ENI 存在秒级延迟 | 等待数秒后重试 `DescribeSecurityGroupPolicies` |

## 下一步

- [Pod 间独占网卡模式](../Pod 间独占网卡模式/tccli 操作.md) — 独占网卡模式详细配置
- [多 Pod 共享网卡模式](../多 Pod 共享网卡模式/tccli 操作.md) — 共享网卡模式详细说明
- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI 模式 Pod 数量限制/tccli 操作.md) — 了解子网 IP 配额与 Pod 上限
- [VPC-CNI 模式与其他云资源、IDC 互通](../VPC-CNI 模式与其他云资源、IDC 互通/tccli 操作.md) — 跨网络访问场景
- [固定 IP 模式使用说明](../固定 IP 模式使用说明/tccli 操作.md) — VPC-CNI 固定 IP 模式
- [容器服务安全组设置](../../../../安全和稳定性/容器服务安全组设置/tccli 操作.md) — 集群节点级安全组配置

## 控制台替代

- 安全组管理：[VPC 控制台 → 安全组](https://console.cloud.tencent.com/vpc/securitygroup) 创建/查看/编辑安全组规则
- 集群 VPC-CNI 信息：控制台 → [容器服务](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → 基本信息 → 网络信息 → VPC-CNI 模式
- ENI 安全组绑定：控制台 → [云服务器](https://console.cloud.tencent.com/cvm/instance) → 选择节点 → 安全组 → 管理安全组
