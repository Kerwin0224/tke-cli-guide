# 新建原生节点池（tccli）

> 对照官方：[新建原生节点](https://cloud.tencent.com/document/product/457/78198) · page_id `78198`

## 概述

通过 `CreateClusterNodePool`（`Type="Native"`）在已有托管集群中创建**原生节点池**。原生节点池区别于普通节点池：使用 CXM 机型、支持 tlinux4（TencentOS Server 4）镜像、支持 Management 参数控制管理和自愈行为、支持 Annotation 声明式管理。节点池创建为**异步操作**，需轮询确认节点就绪。

原生节点池底层使用 CXM（容器专用虚拟机）实例，创建完成后节点自动加入集群并注册到 Kubernetes。

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
#    tke:CreateClusterNodePool, tke:DescribeClusterNodePools
#    tke:DescribeClusters, vpc:DescribeSubnets
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空，证明凭证有效）
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
# 4. 确认集群状态
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"

# 5. 确认 VPC 和子网
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'
# expected: 返回子网详情，AvailableIpAddressCount > 0

# 6. 确认安全组
tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["SECURITY_GROUP_ID"]'
# expected: 返回安全组详情，SecurityGroupId 存在

# 7. 确认 SSH 密钥（如需密钥登录）
tccli cvm DescribeKeyPairs --region <Region> --KeyIds '["KEY_ID"]'
# expected: 返回密钥对详情
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 创建原生节点池 | `CreateClusterNodePool` | 否 |
| 查看节点池 | `DescribeClusterNodePools` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `Name` | String | 是 | 节点池名称，2-64 字符 | 重名 → `InvalidParameterValue` |
| `Type` | String | 是 | `"Native"` 固定值 | 填错 → 创建为普通节点池 |
| `VpcId` | String | 是 | VPC ID，格式 `vpc-` 开头 | 与集群 VPC 不匹配 → `InvalidParameterValue.VpcId`` |
| `SubnetIds` | String[] | 是 | 子网 ID 列表，格式 `subnet-` 开头 | 子网不可用 → `ResourceUnavailable.SubnetUnavailable` |
| `InstanceTypes` | String[] | 是 | CXM 机型列表，如 `CXM.S5.MEDIUM4` | 非 CXM 机型 → `InvalidParameterValue.InstanceType` |
| `MaxNodesNum` | Int | 是 | 最大节点数，>= 期望节点数 | 小于期望数 → `InvalidParameterValue.MaxNodesNum` |
| `DesiredNodesNum` | Int | 否 | 期望节点数，默认 = 1 | 超配额 → `QuotaExceeded.NodeCount` |
| `MinNodesNum` | Int | 否 | 最小节点数，默认 = 0 | 大于期望数 → `InvalidParameterValue.MinNodesNum` |
| `SecurityGroupIds` | String[] | 否 | 安全组 ID 列表 | 安全组不存在 → `InvalidParameterValue.SecurityGroupId` |
| `KeyIds` | String[] | 否 | SSH 密钥 ID 列表 | 密钥不存在 → `InvalidParameterValue.KeyId` |
| `Management` | Object | 否 | 管理参数，控制自愈和行为 | 配置错误 → 节点管理功能异常 |
| `Annotations` | Object[] | 否 | 声明式注解，Key-Value 对 | 格式错误 → `InvalidParameterValue.Annotation` |
| `Labels` | Object[] | 否 | Kubernetes 标签，Key-Value 对 | 格式错误 → `InvalidParameterValue.Label` |
| `Taints` | Object[] | 否 | Kubernetes 污点，Key-Value-Effect | 格式错误 → `InvalidParameterValue.Taint` |

### Management 参数说明

`Management` 对象控制原生节点的管理和自愈行为：

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `Nameservers` | String[] | DNS 服务器列表 |
| `Hosts` | String[] | /etc/hosts 条目 |
| `KernelArgs` | String[] | 内核参数 |

## 操作步骤

### 步骤1：创建原生节点池

#### 选择依据

- **节点类型 `Type="Native"`**：选择 Native 而非空字符串（普通节点池），因为原生节点支持 CXM 机型、声明式管理、故障自愈等高级特性。
- **CXM 机型**：原生节点池必须使用 CXM 机型（如 `CXM.S5.MEDIUM4`、`CXM.S5.LARGE8`），普通 CVM 机型（如 `S5.MEDIUM4`）不被原生节点池支持。CXM 是容器专用虚拟机，对容器工作负载有针对性优化。
- **tlinux4 镜像**：tlinux4（TencentOS Server 4）是腾讯云推荐的容器化操作系统，对容器运行时和内核参数有针对性优化，原生节点池默认使用该镜像。

#### 最小创建（仅必填字段）

`create-native-nodepool-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "native-pool-example",
  "Type": "Native",
  "VpcId": "VPC_ID",
  "SubnetIds": ["SUBNET_ID"],
  "InstanceTypes": ["CXM.S5.MEDIUM4"],
  "MaxNodesNum": 3
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-native-nodepool-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "Response": {
        "NodePoolId": "np-example",
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

#### 增强配置（含 Management、Annotations、Labels、Taints）

`create-native-nodepool-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "native-pool-enhanced",
  "Type": "Native",
  "VpcId": "VPC_ID",
  "SubnetIds": ["SUBNET_ID"],
  "InstanceTypes": ["CXM.S5.LARGE8"],
  "DesiredNodesNum": 2,
  "MinNodesNum": 1,
  "MaxNodesNum": 5,
  "SecurityGroupIds": ["SECURITY_GROUP_ID"],
  "KeyIds": ["KEY_ID"],
  "Management": {
    "Nameservers": ["183.60.83.19", "183.60.82.98"],
    "Hosts": ["127.0.0.1 localhost"],
    "KernelArgs": ["net.core.somaxconn=32768"]
  },
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/example",
      "Value": "demo"
    }
  ],
  "Labels": [
    {
      "Name": "env",
      "Value": "production"
    }
  ],
  "Taints": [
    {
      "Key": "dedicated",
      "Value": "gpu",
      "Effect": "NoSchedule"
    }
  ]
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-native-nodepool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "Response": {
        "NodePoolId": "np-example",
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤2：轮询节点池创建状态

创建为异步操作，需轮询确认节点池就绪：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: LifeState 从 "creating" 变为 "normal"
```

**轮询期间预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-enhanced",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "DesiredNodesNum": 2,
            "MinNodesNum": 1,
            "MaxNodesNum": 5,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 0},
                "AutoscalingAdded": {"Total": 2}
            },
            "AutoscalingGroupStatus": "normal"
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤3：等待节点就绪

节点池 `LifeState` 变为 `normal` 后，节点可能仍在初始化中。进一步确认节点注册到集群：

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceRole "WORKER"
# expected: 返回 InstanceSet，InstanceState 为 "running"
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 2,
        "InstanceSet": [
            {
                "InstanceId": "ins-example-01",
                "InstanceRole": "WORKER",
                "InstanceState": "running",
                "FailedReason": "",
                "NodePoolId": "np-example"
            }
        ],
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 存在性 | `DescribeClusterNodePools` → `TotalCount` | >= 1 |
| 生命周期 | `DescribeClusterNodePoolDetail` → `LifeState` | `normal` |
| 类型 | `DescribeClusterNodePoolDetail` → `Type` | `Native` |
| 节点数 | `DescribeClusterNodePoolDetail` → `NodeCountSummary` | 与 `DesiredNodesNum` 一致 |

### 数据面（kubectl）

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID

# 确认节点已注册（将 Kubeconfig 内容写入 KUBECONFIG_PATH 后执行）
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 返回原生节点，STATUS 为 Ready
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

```bash
kubectl --kubeconfig KUBECONFIG_PATH get ns
# expected: 能正常列出命名空间

kubectl --kubeconfig KUBECONFIG_PATH get nodes -o wide
# expected: 所有原生节点 STATUS 为 Ready
```

## 清理

> **警告**：删除节点池会销毁或退还其下的 CXM 实例。请先确认工作负载已迁移。

### 可选：先缩容到 0 再删除

```bash
# 修改期望节点数为 0
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --DesiredNodesNum 0
# expected: exit 0

# 轮询确认节点数为 0
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: NodeCountSummary 各字段 Total 为 0
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 删除节点池

```bash
# 确认节点池当前状态
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 返回节点池详情

# 执行删除
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，RequestId 返回

# 确认已删除
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: TotalCount 减 1，被删除的 NodePoolId 不再出现
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameterValue` — InstanceType | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"instance-type","Values":["CXM.S5.MEDIUM4"]}]'` 确认 CXM 机型可用 | CXM 机型在指定可用区不可用或已售罄 | 更换可用区或 CXM 机型，如 `CXM.S5.LARGE8` |
| 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --ClusterIds '["CLUSTER_ID"]' --region <Region>` | 集群 ID 不存在或 REGION 不匹配 | 确认 CLUSTER_ID 和 REGION 正确 |
| 返回 `InvalidParameterValue.VpcId` | `tccli vpc DescribeVpcs --region <Region>` 确认 VPC 存在 | VpcId 与集群 VPC 不一致或 VPC 不存在 | 查询集群 VPC 后使用正确 VpcId |
| 返回 `ResourceUnavailable.SubnetUnavailable` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 查看 `AvailableIpAddressCount` | 子网 IP 地址耗尽或不属于集群 VPC | 更换子网或清理不用的 IP 分配 |
| 返回 `QuotaExceeded.NodeCount` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 统计已有节点数 | 集群节点总数即将超过配额 | 申请提额或删除旧节点池释放配额 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 `LifeState` 长时间为 `creating` | `tccli tke DescribeClusterNodePoolDetail` 轮询查看状态变化 | CXM 实例创建缓慢（通常 3-5 分钟） | 等待 5-10 分钟后重查，超过 15 分钟提交工单 |
| 节点池 `LifeState` 为 `normal` 但 `NodeCountSummary` 为 0 | `tccli tke DescribeClusterNodePoolDetail` 查看 `AutoscalingGroupStatus` | 弹性伸缩组未触发创建 | 检查 `DesiredNodesNum` ≥ `MinNodesNum`，确认伸缩组状态正常 |
| 实例 `InstanceState` 为 `failed` | `tccli tke DescribeClusterInstances` → `FailedReason` | 安全组规则阻止加入集群、密钥不匹配或镜像拉取失败 | 检查安全组放行 TCP 6443（apiserver）、10250（kubelet），确认密钥对存在 |
| 节点已 `running` 但 `kubectl get nodes` 显示 `NotReady` | `kubectl --kubeconfig KUBECONFIG_PATH describe node <name>` | kubelet 未就绪或 CNI 未配置 | 等待 2-3 分钟（kubelet 自动初始化），超过 5 分钟登录节点排查 kubelet 日志 |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [原生节点概述](../原生节点概述/tccli%20操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 新建](https://console.cloud.tencent.com/tke2/cluster)
