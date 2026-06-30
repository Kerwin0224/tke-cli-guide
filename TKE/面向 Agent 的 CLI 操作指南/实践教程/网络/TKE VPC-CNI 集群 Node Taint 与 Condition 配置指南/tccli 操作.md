# TKE VPC-CNI 集群 Node Taint 与 Condition 配置指南（tccli）

> 对照官方：[TKE VPC-CNI 集群 Node Taint 与 Condition 配置指南](https://cloud.tencent.com/document/product/457/129804) · page_id `129804`

## 概述

VPC-CNI 模式下节点 IP 资源有限，通过 Taint 和 Condition 自动管理节点 IP 耗尽时的调度策略。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running", VPC-CNI 模式
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

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看节点 Taint | `kubectl describe node NODE_NAME`（需 VPN/IOA） | 是 |
| 添加 Taint | `kubectl taint nodes NODE_NAME key=value:effect`（需 VPN/IOA） | 是 |
| 查看节点池配置 | `tccli tke DescribeClusterNodePools` | 是 |
| 查看 IP 使用 | `tccli vpc DescribeSubnets --SubnetIds '["SUBNET_ID"]'` | 是 |

## 操作步骤

### 步骤 1：查看 VPC-CNI 节点 IP 状态

#### 选择依据

VPC-CNI 模式下每个节点可分配的 Pod IP 数受限于 ENI 配额和子网大小。当节点 IP 即将耗尽时，TKE 自动给节点打上 `node.kubernetes.io/network-unavailable` Taint，阻止新 Pod 调度。

```bash
# 控制面：查看节点 IP 资源
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 InstanceSet

# 查看子网 IP 余量
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 含 AvailableIpAddressCount
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

### 步骤 2：手动管理节点 Taint

数据面（需 VPN/IOA）：

```bash
# 查看当前 Taint
kubectl describe node NODE_NAME | grep Taints
# expected: 如含 node.kubernetes.io/network-unavailable:NoSchedule（IP 耗尽）

# 手动添加 Taint（IP 耗尽时）
kubectl taint nodes NODE_NAME node.kubernetes.io/network-unavailable=:NoSchedule
# expected: node/NODE_NAME tainted

# 移除 Taint（IP 恢复后）
kubectl taint nodes NODE_NAME node.kubernetes.io/network-unavailable:NoSchedule-
# expected: node/NODE_NAME untainted
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：配置节点池 IP 策略

```bash
# 查看节点池配置
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: NodePoolSet 含节点池参数

# 更新节点池子网（扩展 IP 池）
cat > update-pool-subnet.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "NodePoolId": "NODE_POOL_ID",
    "SubnetIds": ["EXISTING_SUBNET_ID", "NEW_SUBNET_ID"]
}
EOF
tccli tke ModifyClusterNodePool --region <Region> --cli-input-json file://update-pool-subnet.json
# expected: exit 0
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

### 步骤 4：确认 Pod 调度不受影响

```bash
kubectl get pods -A -o wide | grep NODE_NAME
# expected: 节点上 Pod 正常运行

kubectl describe node NODE_NAME | grep -A5 "Conditions:"
# expected: NetworkUnavailable 条件为 False（IP 充足）
```

```text
NAME  STATUS  AGE
...
```

## 验证

```bash
# 节点 IP 资源状态
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'
# expected: AvailableIpAddressCount > 0

# 节点池子网配置
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: SubnetIds 含新增子网

# 数据面
kubectl describe node NODE_NAME | grep -E "Taints|NetworkUnavailable"
# expected: 无 network-unavailable Taint
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

## 清理

Taint 配置页无需特殊清理。如需移除手动添加的 Taint：

```bash
kubectl taint nodes NODE_NAME node.kubernetes.io/network-unavailable:NoSchedule-
# expected: taint removed
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 新 Pod 无法调度到节点 | `kubectl describe node NODE_NAME` 查看 Taints | VPC-CNI 子网 IP 耗尽，节点被 Taint | 扩展节点池子网或删除不用的 Pod 释放 IP |
| 节点 IP 分配异常 | `tccli vpc DescribeNetworkInterfaces --Filters '[{"Name":"attachment.instance-id","Values":["INSTANCE_ID"]}]'` | ENI 配额用完 | 升级节点规格（更高 ENI 配额） |
| Taint 不自动解除 | `kubectl get events -A` | IP 恢复但控制器未更新 Taint | 手动 `kubectl taint ... NoSchedule-` 或重启节点 |
| 子网 IP 已满无法扩展 | `tccli vpc DescribeSubnets --SubnetIds '["SUBNET_ID"]'` | 子网容量不足且无可用新子网 | 创建更大的子网并加入节点池 |

## 下一步

- [共享网卡固定 IP 模式移除子网](../共享网卡固定%20IP%20模式%20TKE%20集群移除子网/tccli%20操作.md) -- page_id `126847`
- [在 TKE 上对 Pod 进行带宽限速](../在%20TKE%20上对%20Pod%20进行带宽限速/tccli%20操作.md) -- page_id `48766`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 节点管理 -> 节点池 -> 编辑 -> 子网配置。
