# 新建原生节点

> 对照官方：[新建原生节点](https://cloud.tencent.com/document/product/457/78198) · page_id `78198` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 `CreateClusterNodePool` 在已有集群中创建**原生节点池**。原生节点池区别于普通节点池：使用 CXM 机型、支持 tlinux4（TencentOS Server 4）镜像、支持 Management 参数控制管理和自愈行为、支持 Annotation 声明式管理。节点池创建为**异步操作**，需轮询确认节点就绪。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateClusterNodePool, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusters
#    vpc:DescribeSubnets, vpc:DescribeSecurityGroups
#    cvm:DescribeKeyPairs, cvm:DescribeZoneInstanceConfigInfos
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空，证明凭证有效）
```

预期输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-existing01",
            "Name": "existing-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

> 从上述输出中获取 `<ClusterId>`：使用 `DescribeClusters` 返回的 `$.Clusters[0].ClusterId`。

### 资源检查

```bash
# 4. 确认集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"
```

预期输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.28.3",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "Subnets": ["subnet-example01", "subnet-example02"]
            }
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

> 从上述输出获取 `<VpcId>`：`$.Clusters[0].ClusterNetworkSettings.VpcId`；`<SubnetId>`：`$.Clusters[0].ClusterNetworkSettings.Subnets[0]`。

```bash
# 5. 确认子网可用 IP
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# expected: AvailableIpAddressCount > 0
```

预期输出：

```json
{
    "TotalCount": 1,
    "SubnetSet": [
        {
            "SubnetId": "subnet-example01",
            "SubnetName": "example-subnet",
            "VpcId": "vpc-example",
            "CidrBlock": "10.0.1.0/24",
            "AvailableIpAddressCount": 200,
            "Zone": "ap-guangzhou-3"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 6. 确认安全组存在
tccli vpc DescribeSecurityGroups --region <Region> \
    --SecurityGroupIds '["<SecurityGroupId>"]'
# expected: 返回安全组详情
```

预期输出：

```json
{
    "TotalCount": 1,
    "SecurityGroupSet": [
        {
            "SecurityGroupId": "sg-example",
            "SecurityGroupName": "example-sg",
            "SecurityGroupDesc": "Default security group",
            "ProjectId": "0"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 7. 确认 SSH 密钥（如需密钥登录）
tccli cvm DescribeKeyPairs --region <Region> \
    --KeyIds '["<KeyId>"]'
# expected: 返回密钥对详情
```

预期输出：

```json
{
    "TotalCount": 1,
    "KeyPairSet": [
        {
            "KeyId": "skey-example",
            "KeyName": "example-key",
            "ProjectId": "0"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 8. 确认 CXM 机型可用
tccli cvm DescribeZoneInstanceConfigInfos --region <Region> \
    --Filters '[{"Name":"instance-type","Values":["CXM.S5.MEDIUM4"]}]'
# expected: 返回机型信息，Status 为 "SELL"
```

预期输出：

```json
{
    "InstanceTypeQuotaSet": [
        {
            "Zone": "ap-guangzhou-3",
            "InstanceType": "CXM.S5.MEDIUM4",
            "InstanceChargeType": "POSTPAID_BY_HOUR",
            "Status": "SELL",
            "Cpu": 2,
            "Memory": 4
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

## 控制台与 CLI 参数映射

> **重要**：以下参数为 `CreateClusterNodePool` API 的主要参数。执行前请使用 `tccli tke CreateClusterNodePool --generate-cli-skeleton` 获取最新的完整参数签名和嵌套结构。

| 控制台参数 | CLI 参数 | 类型 | 必填 | 取值与约束 | 幂等 | 错误后果 |
|-----------|---------|------|:--:|------|:--:|------|
| 集群 ID | `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 是 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| 节点池名称 | `Name` | String | 是 | 节点池名称，2-64 字符 | 否（重名冲突） | 重名 → `InvalidParameterValue` |
| 伸缩组参数 | `AutoScalingGroupPara` | String | 是 | JSON 字符串，内部含 `VpcId`/`SubnetIds`/`InstanceTypes`/`MaxNodesNum` 等 | 否 | 缺失 → `MissingParameter.AutoScalingGroupPara` |
| 启动配置参数 | `LaunchConfigurePara` | String | 是 | JSON 字符串，内部含 `SecurityGroupIds`/`KeyIds`/`ImageId` 等 | 否 | 缺失 → `MissingParameter.LaunchConfigurePara` |
| 实例高级设置 | `InstanceAdvancedSettings` | Object | 是 | 含 `MountTarget`/`DockerGraphPath`/`UserScript` 等 | 否 | 配置错误 → 节点初始化失败 |
| 启用自动伸缩 | `EnableAutoscale` | Boolean | 是 | `true` 或 `false` | 是 | 默认值导致非预期行为 |
| CXM 机型 | `AutoScalingGroupPara.InstanceTypes` | String[] | 是 | CXM 机型列表，如 `["CXM.S5.MEDIUM4"]` | 否 | 非 CXM 机型 → `InvalidParameterValue.InstanceType` |
| 最大节点数 | `AutoScalingGroupPara.MaxNodesNum` | Int | 是 | 最大节点数，>= 期望节点数 | 是（相同值） | 小于期望数 → `InvalidParameterValue.MaxNodesNum` |
| 期望节点数 | `AutoScalingGroupPara.DesiredNodesNum` | Int | 否 | 期望节点数，默认 1 | 否（触发伸缩） | 超配额 → `QuotaExceeded.NodeCount` |
| 最小节点数 | `AutoScalingGroupPara.MinNodesNum` | Int | 否 | 最小节点数，默认 0 | 是（相同值） | 大于期望数 → `InvalidParameterValue.MinNodesNum` |
| 安全组 | `LaunchConfigurePara.SecurityGroupIds` | String[] | 否 | 安全组 ID 列表 | 否 | 安全组不存在 → `InvalidParameterValue.SecurityGroupId` |
| SSH 密钥 | `LaunchConfigurePara.KeyIds` | String[] | 否 | SSH 密钥 ID 列表 | 否 | 密钥不存在 → `InvalidParameterValue.KeyId` |
| Management 参数 | `Management` | Object | 否 | 管理参数，控制 DNS/hosts/内核/kubelet | 否 | 配置错误 → 节点管理功能异常 |
| 声明式注解 | `Annotations` | Object[] | 否 | 声明式注解，Key-Value 对 | 否 | 格式错误 → `InvalidParameterValue.Annotation` |
| Kubernetes 标签 | `Labels` | Object[] | 否 | Kubernetes 标签，Key-Value 对 | 否 | 格式错误 → `InvalidParameterValue.Label` |
| Kubernetes 污点 | `Taints` | Object[] | 否 | Kubernetes 污点，Key-Value-Effect | 否 | 格式错误 → `InvalidParameterValue.Taint` |

> **注意**：`Type` 参数不是 `CreateClusterNodePool` API 的有效字段。原生节点类型由 `LaunchConfigurePara` 中的机型参数推断（使用 CXM 机型即表示原生节点池）。请使用 `--generate-cli-skeleton` 确认当前支持的参数列表。

### Management 参数说明

`Management` 对象控制原生节点的管理和自愈行为，仅对新增节点生效：

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `Nameservers` | String[] | DNS 服务器列表 |
| `Hosts` | String[] | /etc/hosts 条目 |
| `KernelArgs` | String[] | 内核参数 |
| `KubeletArgs` | String[] | Kubelet 启动参数 |

## 操作步骤

### 步骤 1：创建原生节点池

#### 选择依据

- **CXM 机型**：原生节点池必须使用 CXM 机型（如 `CXM.S5.MEDIUM4`、`CXM.S5.LARGE8`），普通 CVM 机型（如 `S5.MEDIUM4`）不被支持。CXM 是容器专用虚拟机，对容器工作负载有针对性优化。
- **tlinux4 镜像**：tlinux4（TencentOS Server 4）是腾讯云推荐的容器化操作系统，原生节点池默认使用该镜像。
- **参数结构**：`CreateClusterNodePool` 使用嵌套 JSON 字符串参数（`AutoScalingGroupPara`、`LaunchConfigurePara`）。建议先用 `tccli tke CreateClusterNodePool --generate-cli-skeleton` 获取最新参数签名。

#### 方案 A：最小创建（仅必填字段）

`create-native-nodepool-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "native-pool-example",
  "EnableAutoscale": true,
  "AutoScalingGroupPara": "{\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"],\"InstanceTypes\":[\"CXM.S5.MEDIUM4\"],\"MaxNodesNum\":3,\"MinNodesNum\":0,\"DesiredNodesNum\":1,\"InstanceChargeType\":\"POSTPAID_BY_HOUR\"}",
  "LaunchConfigurePara": "{\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"InstanceType\":\"CXM.S5.MEDIUM4\"}",
  "InstanceAdvancedSettings": {}
}
```

> **注意**：以上 JSON 结构基于 `--generate-cli-skeleton` 交叉验证的方向性修复。因集群已销毁无法实测，请执行前运行 `tccli tke CreateClusterNodePool --generate-cli-skeleton` 确认准确的参数签名和嵌套层级。若 skeleton 签名与以上结构不一致，以 skeleton 为准。

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-native-nodepool-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

#### 方案 B：增强配置（含 Management、Annotations、Labels、Taints）

`create-native-nodepool-enhanced.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "native-pool-enhanced",
  "EnableAutoscale": true,
  "AutoScalingGroupPara": "{\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"],\"InstanceTypes\":[\"CXM.S5.LARGE8\"],\"MaxNodesNum\":5,\"MinNodesNum\":1,\"DesiredNodesNum\":2,\"InstanceChargeType\":\"POSTPAID_BY_HOUR\"}",
  "LaunchConfigurePara": "{\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"KeyIds\":[\"<KeyId>\"],\"InstanceType\":\"CXM.S5.LARGE8\"}",
  "InstanceAdvancedSettings": {
    "MountTarget": "/var/lib/container",
    "DockerGraphPath": "/var/lib/docker"
  },
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

> **注意**：`AutoScalingGroupPara` 和 `LaunchConfigurePara` 为 JSON 字符串（需转义内部引号）。`InstanceAdvancedSettings`、`Management`、`Annotations`、`Labels`、`Taints` 为顶层对象参数。执行前请使用 `--generate-cli-skeleton` 确认准确结构。

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-native-nodepool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 步骤 2：轮询节点池创建状态

创建为异步操作，需轮询确认节点池就绪：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: LifeState 从 "creating" 变为 "normal"
```

**轮询期间预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "native-pool-enhanced",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
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
```

### 步骤 3：等待节点就绪

节点池 `LifeState` 变为 `normal` 后，节点可能仍在初始化中。进一步确认节点注册到集群：

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole "WORKER"
# expected: 返回 InstanceSet，InstanceState 为 "running"
```

**预期输出**：

```json
{
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
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 存在性 | `DescribeClusterNodePools` -> `TotalCount` | >= 1 |
| 生命周期 | `DescribeClusterNodePoolDetail` -> `LifeState` | `normal` |
| 节点数 | `DescribeClusterNodePoolDetail` -> `NodeCountSummary` | 与 `DesiredNodesNum` 一致 |

### 数据面（kubectl）

```bash
# 获取 kubeconfig 并对内容进行 base64 解码后写入文件
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' | base64 -d > <KubeconfigPath>
# expected: exit 0，Kubeconfig 写入 <KubeconfigPath>

# 确认节点已注册
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 返回原生节点，STATUS 为 Ready
```

预期输出：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3Rlcjo...",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
kubectl --kubeconfig <KubeconfigPath> get ns
# expected: 能正常列出命名空间，证明 kubectl 可达集群

kubectl --kubeconfig <KubeconfigPath> get nodes -o wide
# expected: 所有原生节点 STATUS 为 Ready
```

## 清理

> **⚠️ 副作用警告**：
> - `DeleteClusterNodePool` 会**销毁**其下所有 CXM 实例，不可恢复。节点上的本地存储（emptyDir 等）数据会一并丢失。
> - 缩容期间 Pod 可能被驱逐，影响正在运行的业务。
> - 计费影响：创建节点池会产生 CXM 实例费用；删除节点池则停止计费。
> - 操作前请先确认工作负载已迁移，并在低峰期执行。

### 可选：先缩容到 0 再删除

```bash
# 修改伸缩组期望容量为 0（释放所有 CXM 实例）
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DesiredCapacity 0
# expected: exit 0

# 轮询确认节点数为 0
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: NodeCountSummary 各字段 Total 为 0
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "native-pool-enhanced",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "NodeCountSummary": {
            "ManuallyAdded": {"Total": 0},
            "AutoscalingAdded": {"Total": 0}
        }
    },
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 删除节点池

```bash
# 确认节点池当前状态
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回节点池详情，确认是待删除的目标

# 执行删除（KeepInstance=false 表示级联删除 CVM 和磁盘）
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false
# expected: exit 0，返回 RequestId

# 确认已删除
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount 减 1，被删除的 NodePoolId 不再出现
```

预期输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-existing01",
            "Name": "existing-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `MissingParameter.AutoScalingGroupPara` | 确认 JSON 中是否包含 `AutoScalingGroupPara` 字段 | API 要求 `AutoScalingGroupPara` 为必填 JSON 字符串参数，文档旧版将其内部字段展平为顶层参数 | 使用 `--generate-cli-skeleton` 获取正确参数结构，将 VpcId/SubnetIds/InstanceTypes/MaxNodesNum 等放入 `AutoScalingGroupPara` JSON 字符串 |
| 返回 `InvalidParameterValue.InstanceType` | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"instance-type","Values":["CXM.S5.MEDIUM4"]}]'` 确认 CXM 机型可用 | CXM 机型在指定可用区不可用或已售罄 | 更换可用区或 CXM 机型，如 `CXM.S5.LARGE8` |
| 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 集群 ID 不存在或 REGION 不匹配 | 确认 `<ClusterId>` 和 `<Region>` 正确 |
| 返回 `InvalidParameterValue.VpcId` | `tccli vpc DescribeVpcs --region <Region>` 确认 VPC 存在 | VpcId 与集群 VPC 不一致或 VPC 不存在 | 从 `DescribeClusters` 输出 `$.Clusters[0].ClusterNetworkSettings.VpcId` 获取集群 VPC ID |
| 返回 `ResourceUnavailable.SubnetUnavailable` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 查看 `AvailableIpAddressCount` | 子网 IP 地址耗尽或不属于集群 VPC | 更换子网或清理不用的 IP 分配 |
| 返回 `QuotaExceeded.NodeCount` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 统计已有节点数 | 集群节点总数超过配额 | 申请提额或删除旧节点池释放配额 |
| `DeleteClusterNodePool` 返回参数不识别 | 检查命令参数名 | `--NodePoolId`（单数）应改为 `--NodePoolIds`（数组），缺少必填参数 `--KeepInstance` | 改为 `--NodePoolIds '["<NodePoolId>"]' --KeepInstance false` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 `LifeState` 长时间为 `creating` | `tccli tke DescribeClusterNodePoolDetail` 轮询查看状态变化 | CXM 实例创建缓慢（通常 3-5 分钟） | 等待 5-10 分钟后重查；超过 15 分钟则保留 `<ClusterId>`、`<NodePoolId>`、`RequestId` -- 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 |
| 节点池 `LifeState` 为 `normal` 但 `NodeCountSummary` 为 0 | `tccli tke DescribeClusterNodePoolDetail` 查看 `AutoscalingGroupStatus` | 弹性伸缩组未触发创建 | 检查 `DesiredNodesNum` >= `MinNodesNum`，确认伸缩组状态正常 |
| 实例 `InstanceState` 为 `failed` | `tccli tke DescribeClusterInstances` -> `FailedReason` | 安全组规则阻止加入集群、密钥不匹配或镜像拉取失败 | 检查安全组放行 TCP 6443（apiserver）、10250（kubelet），确认密钥对存在 |
| 节点已 `running` 但 `kubectl get nodes` 显示 `NotReady` | `kubectl --kubeconfig <KubeconfigPath> describe node <NodeName>` | kubelet 未就绪或 CNI 未配置 | 等待 2-3 分钟（kubelet 自动初始化），超过 5 分钟登录节点排查 kubelet 日志 |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) -- page_id `103599`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) -- page_id `78648`
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78209) -- page_id `78650`
- [声明式操作实践](https://cloud.tencent.com/document/product/457/78649) -- page_id `78649`

## 控制台替代

[容器服务控制台 -> 集群 -> 节点管理 -> 原生节点 -> 新建](https://console.cloud.tencent.com/tke2/cluster)
