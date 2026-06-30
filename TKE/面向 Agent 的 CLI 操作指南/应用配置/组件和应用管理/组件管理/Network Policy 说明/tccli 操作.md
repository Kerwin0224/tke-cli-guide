# Network Policy 说明（tccli）

> 对照官方：[Network Policy 说明](https://cloud.tencent.com/document/product/457/50841) · page_id `50841`

## 概述

NetworkPolicy 是 Kubernetes 网络策略资源，用于控制 Pod 之间的网络流量。TKE 通过 NetworkPolicy 组件实现基于标签选择器的入站/出站流量控制。

### 功能特性

- 基于 Pod Selector 定义流量规则
- 支持 Ingress（入站）和 Egress（出站）规则
- 支持 namespaceSelector 和 podSelector 组合
- 支持 IP Block 规则

### 示例

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

## 前置条件

- [环境准备](../../../../环境准备.md)
- 熟悉 Kubernetes 相关概念，已了解 TKE 集群架构
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 Network Policy 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy` | 是 |
| 安装 Network Policy 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy` | 否 |
| 升级 Network Policy 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy --AddonVersion <AddonVersion>` | 否 |
| 卸载 Network Policy 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy` | 是 |
| 创建 NetworkPolicy | `kubectl apply -f network-policy.yaml` | 否（同名报错） |
| 查看 NetworkPolicy | `kubectl get networkpolicies` | 是 |

## 操作步骤

### 步骤 1：查看 Network Policy 组件信息（控制面）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: 返回 Network Policy 组件详细信息
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 步骤 2：安装 Network Policy 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName network-policy
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：创建 NetworkPolicy（数据面）

```bash
kubectl apply -f network-policy.yaml
# expected: networkpolicy.networking.k8s.io/<name> created
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：卸载 Network Policy 组件（控制面）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName network-policy
# expected: exit 0，组件卸载请求已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（kubectl）

```bash
kubectl get networkpolicies -A
# expected: 返回 NetworkPolicy 列表

kubectl describe networkpolicy <name> -n <namespace>
# expected: 返回策略详情
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

```bash
# 删除 NetworkPolicy
kubectl delete networkpolicy <name> -n <namespace>
# expected: networkpolicy.networking.k8s.io "<name>" deleted

# 卸载组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName network-policy
# expected: exit 0

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy \
    | jq '.Addons[] | select(.AddonName == "network-policy")'
# expected: 空结果
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| 组件安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName network-policy` 查看 Status 与 reason | 集群版本不兼容或资源不足 | 检查集群版本要求；扩容节点后重试 |
| NetworkPolicy 不生效 | `kubectl describe networkpolicy <name>` 检查规则；`kubectl get pods -n kube-system` 检查组件 Pod | 组件 Pod 异常或网络插件不支持 | 检查组件 Pod 状态；确认网络插件支持 NetworkPolicy |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)
- [eNP（ebpf Network Policies）使用说明](../eNP（ebpf%20Network%20Policies）使用说明/tccli 操作.md)
- [NginxIngress 说明](../NginxIngress 说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理。
