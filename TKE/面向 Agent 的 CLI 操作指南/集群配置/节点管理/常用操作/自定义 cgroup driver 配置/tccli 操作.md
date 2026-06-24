# 自定义 cgroup driver 配置

> 对照官方：[自定义 cgroup driver 配置](https://cloud.tencent.com/document/product/457/132099) · page_id `132099` · tccli ≥3.1.14 · API 2018-05-25

## 概述

TKE 支持在创建节点池时通过 Kubelet 自定义参数配置 cgroup driver，以满足不同容器运行时和操作系统的管理需求。cgroup driver 决定容器运行时（如 containerd）如何与 Linux cgroup 子系统交互。

| cgroup driver | 适用场景 | containerd 配置 |
|---------------|---------|----------------|
| `systemd`（推荐） | 使用 systemd 作为 init 系统的 Linux（TencentOS、Ubuntu），可避免两个 cgroup 管理器共存 | `SystemdCgroup = true` |
| `cgroupfs` | 传统模式，兼容非 systemd 系统 | `SystemdCgroup = false` |

**适用范围**：TKE 标准集群（普通节点池），支持 Kubernetes 1.22 及以上版本。配置方式：在创建节点池时，通过 `InstanceAdvancedSettings.ExtraArgs.Kubelet` 指定 `cgroup-driver=systemd`（**key=value 格式，无 `--` 前缀**）。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域
```

### 权限检查

需要以下 Action 名：`tke:DescribeClusters`、`tke:DescribeClusterAvailableExtraArgs`、`tke:DescribeClusterNodePools`、`tke:CreateClusterNodePool`、`tke:DescribeClusterNodePoolDetail`、`tke:DeleteClusterNodePool`、`vpc:DescribeVpcs`、`vpc:DescribeSubnets`、`vpc:DescribeSecurityGroups`、`cvm:DescribeInstances`。

```bash
# 验证 DescribeClusters 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

**预期输出**（CAM 权限正常时）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 验证 VPC 权限（节点池创建需要）
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，返回 VPC 列表（可为空）
```

**预期输出**（CAM 权限正常时）：

```json
{
    "TotalCount": 1,
    "VpcSet": [
        {
            "VpcId": "vpc-example",
            "VpcName": "example-vpc",
            "CidrBlock": "172.16.0.0/12",
            "IsDefault": false
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 验证 CVM 权限（节点池创建 CVM 实例需要）
tccli cvm DescribeInstances --region <Region>
# expected: exit 0，返回 CVM 列表（可为空）
```

**预期输出**（CAM 权限正常时）：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceType": "S5.MEDIUM2",
            "InstanceName": "example-instance",
            "InstanceState": "RUNNING"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 资源检查

```bash
# 4. 查询目标集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0, ClusterStatus 为 Running, ClusterType 为 MANAGED_CLUSTER
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [{
        "ClusterId": "cls-example",
        "ClusterName": "example-cluster",
        "ClusterStatus": "Running",
        "ClusterType": "MANAGED_CLUSTER",
        "ClusterVersion": "1.32.2",
        "ContainerRuntime": "containerd",
        "ClusterNetworkSettings": {
            "VpcId": "vpc-example",
            "ClusterCIDR": "10.0.0.0/16",
            "ServiceCIDR": "10.1.0.0/20",
            "SubnetId": "subnet-example"
        }
    }],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 5. 查询集群 VPC
tccli vpc DescribeVpcs --region <Region> \
    --VpcIds '["<VpcId>"]'
# expected: 返回目标 VPC 信息
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "VpcSet": [
        {
            "VpcId": "vpc-example",
            "VpcName": "example-vpc",
            "CidrBlock": "172.16.0.0/12",
            "IsDefault": false,
            "EnableMulticast": false,
            "DnsServerSet": ["183.60.83.19", "183.60.82.98"],
            "EnableDhcp": true
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 6. 查询集群子网
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpAddressCount ≥ 2
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "SubnetSet": [
        {
            "VpcId": "vpc-example",
            "SubnetId": "subnet-example",
            "SubnetName": "example-subnet",
            "CidrBlock": "172.24.0.0/20",
            "Zone": "ap-guangzhou-3",
            "AvailableIpAddressCount": 4074,
            "TotalIpAddressCount": 4093,
            "IsDefault": false
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 7. 查询安全组（节点池需要）
tccli vpc DescribeSecurityGroups --region <Region>
# expected: 返回安全组列表，至少含 1 个可用安全组
```

**预期输出**：

```json
{
    "SecurityGroupSet": [
        {
            "SecurityGroupId": "sg-example",
            "SecurityGroupName": "example-sg",
            "SecurityGroupDesc": "默认安全组",
            "IsDefault": true
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 8. 查询可用的 Kubelet 自定义参数（含 cgroup-driver）
tccli tke DescribeClusterAvailableExtraArgs --region <Region> \
    --ClusterVersion "1.32.2" \
    --ClusterType "MANAGED_CLUSTER"
# expected: exit 0, AvailableExtraArgs.Kubelet 含 cgroup-driver 参数及其约束
```

**预期输出**：

```json
{
    "AvailableExtraArgs": {
        "Kubelet": [
            {
                "Name": "cgroup-driver",
                "Type": "string",
                "Usage": "Driver that the kubelet uses to manipulate cgroups on the host.",
                "Default": "cgroupfs",
                "Constraint": "('cgroupfs','systemd')"
            }
        ]
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 版本与规格选择

- **K8s 版本**：≥ 1.22。确认集群版本：`tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'`
- **集群类型**：`MANAGED_CLUSTER`（托管集群）
- **cgroup driver**：推荐 `systemd`。确认可用的 Kubelet 参数：
  `tccli tke DescribeClusterAvailableExtraArgs --region <Region> --ClusterVersion "1.32.2" --ClusterType "MANAGED_CLUSTER" | jq '.AvailableExtraArgs.Kubelet[] | select(.Name=="cgroup-driver")'`

## 控制台与 CLI 参数映射

本页涉及的 API 操作：

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查询可用 Kubelet 自定义参数 | `DescribeClusterAvailableExtraArgs` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 创建节点池（含 cgroup driver 配置） | `CreateClusterNodePool` | 否 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 删除节点池 | `DeleteClusterNodePool` | 是 |

以下说明 `CreateClusterNodePool` 和 `DeleteClusterNodePool` 中与 cgroup driver 配置相关的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 已存在的集群 ID，格式 `cls-xxxxxxxx` | 不存在 → `ResourceNotFound` |
| `AutoScalingGroupPara` | String | 是 | JSON 字符串。**必含字段**：`MaxSize`/`MinSize`/`DesiredCapacity`/`VpcId`/`SubnetIds`。**禁止字段**：`AutoScalingGroupName`（TKE 自动生成）、`Tags`（TKE 管理） | 含 `AutoScalingGroupName` → `InvalidParameter.Param`: "autoscaling group name can not be set" |
| `LaunchConfigurePara` | String | 是 | JSON 字符串。**必含字段**：`InstanceType`/`SystemDisk`/`InternetAccessible`/`SecurityGroupIds`/`InstanceChargeType`。**禁止字段**：`ImageId`（由 `NodePoolOs` 指定）、`LaunchConfigurationName`（TKE 自动生成） | 含 `ImageId` → `InvalidParameter.Param`: "image id can not be set" |
| `InstanceAdvancedSettings.ExtraArgs.Kubelet` | Array | 否 | **格式 `["cgroup-driver=systemd"]`**（key=value 格式，无 `--` 前缀）。可用值和约束见 `DescribeClusterAvailableExtraArgs` | 使用 `--cgroup-driver=systemd` → `InvalidParameter.Param`: "is not in --cgroup-driver available args list" |
| `EnableAutoscale` | Boolean | 是 | `true`（开启自动伸缩）或 `false`（关闭） | — |
| `Name` | String | 否 | 节点池名称，1-60 字符 | — |
| `ContainerRuntime` | String | 否 | `containerd`（推荐）或 `docker`。**不传 `RuntimeVersion`**，由 TKE 按 K8s 版本自动匹配 | 传不兼容的 `RuntimeVersion` → `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`: "k8s version and runtime (type/version) not match" |
| `NodePoolOs` | String | 否 | 如 `tlinux3.1x86_64`。镜像由该参数指定，不在 `LaunchConfigurePara` 中传 `ImageId` | — |
| `DeletionProtection` | Boolean | 否 | `true`/`false`，默认 `false` | 忘开启 → 可能误删生产节点池 |

`DeleteClusterNodePool` 参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | — |
| `NodePoolIds` | Array | 是 | 节点池 ID 列表 | — |
| `KeepInstance` | Boolean | 是 | `true`（保留 CVM 实例）/ `false`（销毁）。零节点时选 `false` | 不传 → `MissingParameter`: "--KeepInstance" |

## 操作步骤

### 步骤 1：检查集群状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0, ClusterStatus 为 Running, ContainerRuntime 为 containerd
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2",
            "ContainerRuntime": "containerd",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "SubnetId": "subnet-example"
            }
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

### 步骤 2：查询可用的 Kubelet 自定义参数

在配置 cgroup driver 之前，先确认当前 K8s 版本支持的 Kubelet 参数及其约束。

```bash
tccli tke DescribeClusterAvailableExtraArgs --region <Region> \
    --ClusterVersion "1.32.2" \
    --ClusterType "MANAGED_CLUSTER"
# expected: exit 0, Kubelet 列表含 cgroup-driver 参数
```

**预期输出**（cgroup-driver 条目）：

```json
{
    "AvailableExtraArgs": {
        "Kubelet": [
            {
                "Name": "cgroup-driver",
                "Type": "string",
                "Usage": "Driver that the kubelet uses to manipulate cgroups on the host.",
                "Default": "cgroupfs",
                "Constraint": "('cgroupfs','systemd')"
            }
        ]
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **`Constraint` 解读**：`('cgroupfs','systemd')` 表示只接受这两个值。`Default: cgroupfs` 表示不指定时默认使用 cgroupfs，因此生产环境推荐显式配置为 `systemd`。

### 步骤 3：查看已有节点池

确认目标集群中没有正在使用相同名称的节点池，避免命名冲突。

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0, 返回节点池列表
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {"NodePoolId": "np-example-1", "Name": "example-sn-np", "LifeState": "normal"},
        {"NodePoolId": "np-example-2", "Name": "example-np", "LifeState": "normal"}
    ],
    "TotalCount": 2,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：创建节点池并配置 cgroup driver 为 systemd

#### 选择依据

*为什么选这些值而不是其他：*

- **cgroup driver**：选择 `systemd` 而非 `cgroupfs`。在使用 systemd 作为 init 系统的操作系统（TencentOS、Ubuntu）上，`systemd` 是推荐驱动，可以避免系统中两个 cgroup 管理器共存的问题。containerd 默认也推荐 `systemd`。替代方案为 `cgroupfs`（仅用于兼容非 systemd 系统）。
- **ExtraArgs 格式**：Kubelet 参数使用 `key=value` 格式（`cgroup-driver=systemd`），**不是** `--cgroup-driver=systemd`。TKE API 不接受 `--flag=value` 格式——这是通过 `CreateClusterNodePool` 实际操作验证的。控制台显示 `--cgroup-driver=systemd` 但 API 需要的是 `cgroup-driver=systemd`。直接复制控制台格式会导致 `InvalidParameter.Param`。
- **零节点模式**：设置 `DesiredCapacity=0`、`MinSize=0`，创建零节点节点池。仅验证 API 参数传递，不启动 CVM 实例，不产生计费。生产环境应将 `DesiredCapacity` 设为实际所需节点数。
- **精简 AutoScalingGroupPara**：只传 `MaxSize`/`MinSize`/`DesiredCapacity`/`VpcId`/`SubnetIds`。不传 `AutoScalingGroupName`（TKE 自动生成），不传 `Tags`（TKE 管理），违者返回 `PARAM_ERROR`。
- **精简 LaunchConfigurePara**：只传 `InstanceType`/`SystemDisk`/`InternetAccessible`/`SecurityGroupIds`/`InstanceChargeType`。不传 `ImageId`（镜像由 `NodePoolOs` 指定），不传 `LaunchConfigurationName`（TKE 自动生成），违者返回 `PARAM_ERROR`。
- **不传 RuntimeVersion**：由 TKE 按 K8s 版本自动匹配合适的容器运行时版本。手动指定可能与 K8s 版本不兼容，导致 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`。如需手动指定，先用 `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion 1.32.2` 查询兼容版本。

如何查询可用的 cgroup driver 值：

```bash
tccli tke DescribeClusterAvailableExtraArgs --region <Region> \
    --ClusterVersion "1.32.2" \
    --ClusterType "MANAGED_CLUSTER" \
    | jq '.AvailableExtraArgs.Kubelet[] | select(.Name=="cgroup-driver")'
```

#### 最小创建（零节点，只含必填字段）

`cgroup-np-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "AutoScalingGroupPara": "{\"MaxSize\":1,\"MinSize\":0,\"DesiredCapacity\":0,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"],\"RetryPolicy\":\"IMMEDIATE_RETRY\"}",
  "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"InternetAccessible\":{\"InternetChargeType\":\"TRAFFIC_POSTPAID_BY_HOUR\",\"InternetMaxBandwidthOut\":0,\"PublicIpAssigned\":false},\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"InstanceChargeType\":\"POSTPAID_BY_HOUR\"}",
  "InstanceAdvancedSettings": {
    "ExtraArgs": {
      "Kubelet": ["cgroup-driver=systemd"]
    }
  },
  "EnableAutoscale": false
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://cgroup-np-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（含可选字段，生产环境参考）

`cgroup-driver-enhanced.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "<NodePoolName>",
  "AutoScalingGroupPara": "{\"MaxSize\":5,\"MinSize\":1,\"DesiredCapacity\":2,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"],\"RetryPolicy\":\"IMMEDIATE_RETRY\"}",
  "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":100},\"InternetAccessible\":{\"InternetChargeType\":\"TRAFFIC_POSTPAID_BY_HOUR\",\"InternetMaxBandwidthOut\":0,\"PublicIpAssigned\":false},\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"InstanceChargeType\":\"POSTPAID_BY_HOUR\"}",
  "InstanceAdvancedSettings": {
    "ExtraArgs": {
      "Kubelet": ["cgroup-driver=systemd"]
    }
  },
  "EnableAutoscale": true,
  "ContainerRuntime": "containerd",
  "NodePoolOs": "tlinux3.1x86_64",
  "OsCustomizeType": "GENERAL",
  "DeletionProtection": true
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://cgroup-driver-enhanced.json
# expected: exit 0，返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<NodePoolName>` | 节点池名称 | 1-60 字符，以字母开头 | 自定义 |
| `<VpcId>` | VPC ID | 格式 `vpc-xxxxxxxx` | 步骤 1 输出中 `ClusterNetworkSettings.VpcId` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，需有可用 IP | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups` |

## 验证

### 控制面（tccli）

节点池创建是异步操作。轮询直到状态为 `normal`：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: LifeState 为 normal, ExtraArgs.Kubelet 含 cgroup-driver=systemd
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-cgroup-np",
        "LifeState": "normal",
        "DesiredNodesNum": 0,
        "RuntimeConfig": {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.9"
        },
        "NodePoolOs": "tlinux3.1x86_64",
        "ExtraArgs": {
            "Kubelet": ["cgroup-driver=systemd"]
        }
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

验证维度：

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 状态 | `LifeState` | `normal` |
| cgroup driver | `ExtraArgs.Kubelet` | 含 `cgroup-driver=systemd` |
| 运行时 | `RuntimeConfig.RuntimeType` | `containerd`（与创建参数一致） |
| 节点数 | `DesiredNodesNum` | 零节点模式下为 `0`（预期行为） |
| OS | `NodePoolOs` | 与创建参数一致 |

> 零节点模式下 `DesiredNodesNum=0` 是预期行为——仅验证 API 参数传递，不产生 CVM 实例。

## 清理

> **警告**：`DeleteClusterNodePool --KeepInstance false` 会**级联销毁**该节点池创建的所有 CVM 实例并释放磁盘。生产环境执行前务必先 `DescribeClusterNodePoolDetail` 确认 `NodePoolId` 和节点数。节点数为 0 时无 CVM 实例可删，无额外影响。
> 如果不删除该节点池，将持续产生节点池关联资源的费用。请按以下步骤清理。

### 1. 清理前状态检查

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# 确认是待删除的目标节点池，记录 NodePoolId、Name、DesiredNodesNum
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-cgroup-np",
        "LifeState": "normal",
        "DesiredNodesNum": 0
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 2. 关闭删除保护（如启用）

```bash
# 如果在增强配置中开启了 DeletionProtection，需先关闭
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DeletionProtection false
```

### 3. 删除节点池

```bash
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false
# ⚠️ --KeepInstance false 会删除该节点池创建的 CVM 实例并释放磁盘
# 零节点时无 CVM 实例可删，安全
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 4. 验证已删除

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 255, ResourceNotFound, "record not found"
```

**预期输出**（节点池已删除）：

```text
[TencentCloudSDKException]
code: FailedOperation.NodePoolQueryFailed
message: related node pool query err(get nodepool 'np-example' failed:
         [E501001 DBRecordNotFound] record not found)
requestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

> `FailedOperation.NodePoolQueryFailed` + `record not found` 表示节点池已成功删除。`exit 255` 为此类错误的正常退出码。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`: "Kubelet args is invalid: [...] is not in --cgroup-driver available args list" | 检查 `InstanceAdvancedSettings.ExtraArgs.Kubelet` 中的参数格式 | 使用了 `--cgroup-driver=systemd` 格式（带 `--` 前缀）。TKE API 不接受 `--flag=value` 格式 | 改为 `cgroup-driver=systemd`（`key=value` 格式，无 `--` 前缀）。正确写法：`"Kubelet": ["cgroup-driver=systemd"]` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`: "is not in systemd available args list" | 检查 Kubelet 参数是否被拆分传入 | 将 `--cgroup-driver` 和 `systemd` 作为两个独立值传入（如 `["--cgroup-driver", "systemd"]`），被 API 误解 | 合并为一个 `key=value` 字符串：`["cgroup-driver=systemd"]` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`: "autoscaling group name can not be set" | 检查 `AutoScalingGroupPara` JSON 字符串 | 传入了 `AutoScalingGroupName` 字段。TKE 自动生成该字段，不接受外部传入 | 从 `AutoScalingGroupPara` 中移除 `AutoScalingGroupName` 字段 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`: "image id can not be set" | 检查 `LaunchConfigurePara` JSON 字符串 | 传入了 `ImageId` 字段。TKE 通过 `--NodePoolOs` 指定镜像，不接受在 `LaunchConfigurePara` 中传 `ImageId` | 从 `LaunchConfigurePara` 中移除 `ImageId` 字段，改用 `--NodePoolOs` 参数（如 `tlinux3.1x86_64`） |
| `CreateClusterNodePool` 返回 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`: "k8s version and runtime (type/version) not match" | `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion 1.32.2` 查询兼容的运行时版本 | 手动指定的 `RuntimeVersion` 与 K8s 版本不匹配 | 不传 `RuntimeVersion`，由 TKE 自动匹配。或使用 `DescribeSupportedRuntime` 查询兼容版本后指定 |
| `DeleteClusterNodePool` 返回 `MissingParameter`: "--KeepInstance" | 检查命令是否含 `--KeepInstance` 参数 | `DeleteClusterNodePool` 要求必传 `--KeepInstance` | 添加 `--KeepInstance false`（零节点时安全）或 `--KeepInstance true`（保留 CVM 实例时） |
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认 NodePoolId | ClusterId 或 NodePoolId 不存在 / 已删除 / 地域不匹配 | 核对 ClusterId、NodePoolId 和 `--region` 参数 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回 `NodePoolId` 但 `LifeState` 长时间非 `normal` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 查看 `LifeState` | 后端创建缓慢（正常）或卡住（异常） | 继续等待（通常 2-5 分钟）。超过 10 分钟则保留 `region`、`ClusterId`、`NodePoolId`、`RequestId` → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 → 仍未解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| `ExtraArgs.Kubelet` 不含 `cgroup-driver=systemd` | 同上，检查 `ExtraArgs` 字段 | ExtraArgs 参数在 API 请求中被遗漏或格式错误导致被忽略 | 确认 `InstanceAdvancedSettings.ExtraArgs.Kubelet` 使用 `["cgroup-driver=systemd"]` 格式，重新创建节点池 |
| `LifeState` 为 `abnormal` | 同上，查看完整响应中的状态信息 | 通常是网络配置、安全组或子网不可用导致 | 检查 `VpcId` 和 `SubnetIds` 是否与集群同 VPC；检查安全组是否存在；修正后重新创建 |

> **保留凭证**：排查时务必保留 `RequestId`、`ClusterId`、`NodePoolId` 和完整的创建 JSON 文件，以便向技术支持提供准确的上下文信息。

## 下一步

- [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) — 完整的节点池创建流程
- [设置节点的启动脚本](../设置节点的启动脚本/tccli%20操作.md) — 节点初始化时注入自定义脚本
- [设置节点 Label](../设置节点%20Label/tccli%20操作.md) — 节点标签调度配置
- [节点资源预留说明](../节点资源预留说明/tccli%20操作.md) — Kubelet 资源预留参数
- [Kubernetes cgroup driver 官方文档](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)

## 控制台替代

[容器服务控制台 - 节点池管理](https://console.cloud.tencent.com/tke2/cluster?rid=1)
