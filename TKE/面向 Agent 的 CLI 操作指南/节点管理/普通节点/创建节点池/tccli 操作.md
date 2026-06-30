# 创建节点池

> 对照官方：[创建节点池](https://cloud.tencent.com/document/product/457/43735) · page_id `43735` · tccli ≥1.0.0 · API 2018-05-25

## 概述

通过 `CreateClusterNodePool` 在已有集群中创建节点池。节点池通过弹性伸缩组（Auto Scaling Group）自动创建 CVM 实例并加入集群。创建节点池需传入两个 JSON 字符串参数：`LaunchConfigurePara`（CVM 启动配置）和 `AutoScalingGroupPara`（弹性伸缩组配置），参数层级和字段名必须严格匹配。

| 决策项 | 取值 | 原因 |
|--------|------|------|
| 机型 | `S5.MEDIUM2`（2C2G） | 标准型 S5 最小规格，适合功能验证。经 `DescribeZoneInstanceConfigInfos` 确认目标可用区有库存 |
| 操作系统 | `ubuntu22.04x86_64` | Ubuntu 22.04 LTS 稳定版本，通过 `NodePoolOs` 参数直接指定。禁止在 `LaunchConfigurePara` 中重复指定 `ImageId` |
| 系统盘 | `CLOUD_PREMIUM` 50GB | 高性能云硬盘最小容量，适合测试。可替换为 `CLOUD_SSD` 获得更好性能 |
| 弹性伸缩 | `EnableAutoscale: false` | 创建固定大小节点池（MinSize=MaxSize=1），不启用弹性伸缩。**此参数为必填**，即使不启用也必须显式传入 `false` |
| 计费模式 | `POSTPAID_BY_HOUR` | 按量付费，用完即删，约 0.08 元/小时 |
| 删除保护 | `DeletionProtection: false` | 测试节点池无需删除保护。生产环境建议开启 |

## 前置条件

- [环境准备](../../../环境准备.md) <!-- 发布时由 sync 脚本替换为平台兼容链接 -->

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域
```

```bash
# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateClusterNodePool, tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterStatus, tke:DescribeOSImages, tke:DescribeClusterInstances
#    tke:DeleteClusterNodePool, vpc:DescribeSubnets, cvm:DescribeZoneInstanceConfigInfos
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running"
        }
    ]
}
```

```bash
# 验证 VPC 权限
tccli vpc DescribeSubnets --region <Region>
# expected: exit 0，返回子网列表（可为空）
```

```json
{
    "SubnetSet": [
        {
            "SubnetId": "subnet-example",
            "Zone": "ap-guangzhou-3",
            "AvailableIpAddressCount": 4079
        }
    ]
}
```

```bash
# 验证 CVM 权限
tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"zone","Values":["<Zone>"]}]'
# expected: exit 0，返回机型列表（可为空）
```

```json
{
    "InstanceTypeQuotaSet": [
        {
            "Zone": "ap-guangzhou-3",
            "InstanceType": "S5.MEDIUM2",
            "Status": "SELL"
        }
    ]
}
```

### 资源检查

```bash
# 4. 确认集群状态（Running 才能创建节点池）
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
    "ClusterStatus": "Running"
}
```

```bash
# 5. 确认子网可用（有充足 IP 可分配节点）
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'
# expected: AvailableIpAddressCount > 0
```

```json
{
    "SubnetId": "subnet-xxxxxxxx",
    "Zone": "ap-guangzhou-3",
    "AvailableIpAddressCount": 4079,
    "VpcId": "vpc-xxxxxxxx",
    "CidrBlock": "172.24.0.0/20"
}
```

```bash
# 5a. 确认 VPC 存在
tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>"]'
# expected: 返回 VPC 详情
```

```json
{
    "VpcId": "vpc-xxxxxxxx",
    "VpcName": "default",
    "CidrBlock": "172.24.0.0/20"
}
```

```bash
# 5b. 确认安全组可用
tccli vpc DescribeSecurityGroups --region <Region> --Filters '[{"Name":"security-group-name","Values":["default"]}]'
# expected: 返回安全组列表，SecurityGroupId 为 sg-xxxxxxxx
```

```json
{
    "SecurityGroupId": "sg-xxxxxxxx",
    "SecurityGroupName": "default",
    "SecurityGroupDesc": "System created default security group"
}
```

```bash
# 5c. 确认可用区
tccli cvm DescribeZones --region <Region>
# expected: 返回可用区列表
```

```json
{
    "Zone": "ap-guangzhou-3",
    "ZoneName": "广州三区",
    "ZoneState": "AVAILABLE"
}
```

```bash
# 6. 确认机型库存（目标可用区有在售机型）
tccli cvm DescribeZoneInstanceConfigInfos --region <Region> \
    --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["S5"]},{"Name":"instance-type","Values":["S5.MEDIUM2"]}]'
# expected: Status "SELL"
```

```json
{
    "InstanceType": "S5.MEDIUM2",
    "Cpu": 2,
    "Memory": 2,
    "Status": "SELL"
}
```

```bash
# 7. 确认操作系统镜像可用
tccli tke DescribeOSImages --region <Region>
# expected: 返回 OS 镜像列表，含 ubuntu22.04x86_64 且 Status 为 online
```

```json
{
    "OsName": "ubuntu22.04x86_64",
    "ImageId": "img-xxxxxxxx",
    "Status": "online"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建节点池 | `CreateClusterNodePool` | 否（重复创建不同名节点池） |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 查看集群工作节点 | `DescribeClusterInstances` | 是 |
| 删除节点池 | `DeleteClusterNodePool` | 是 |

## 关键字段说明

以下说明 `CreateClusterNodePool` 的主要参数。`LaunchConfigurePara` 和 `AutoScalingGroupPara` 为 JSON 字符串类型，内部嵌套结构在子表中展开。

### 顶层参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx`。集群必须为 `Running` 状态 | 集群不存在返回 `ResourceNotFound`；集群非 Running 返回操作拒绝 |
| `EnableAutoscale` | Boolean | 是 | 是否启用弹性伸缩。`false` = 固定大小节点池；`true` = 启用自动扩缩容 | **缺少此参数返回 `MissingParameter`**，即使不启用弹性伸缩也必须显式传入 `false` |
| `InstanceAdvancedSettings` | Object | 是 | 节点高级设置。无需自定义时传空对象 `{}` | 缺少返回 `MissingParameter: 缺少参数 InstanceAdvancedSettings` |
| `LaunchConfigurePara` | String (JSON) | 是 | CVM 启动配置的 JSON 字符串，见下方子表 | 缺少返回 `MissingParameter`；内部字段错误返回 `InvalidParameter.Param` |
| `AutoScalingGroupPara` | String (JSON) | 是 | 弹性伸缩组配置的 JSON 字符串，见下方子表 | 缺少返回 `MissingParameter`；内部字段错误返回 `InvalidParameter.Param` |
| `Name` | String | 否 | 节点池名称，2-64 字符 | 不传则使用系统默认名称 |
| `NodePoolOs` | String | 否 | 操作系统镜像标识符，如 `ubuntu22.04x86_64`。`tccli tke DescribeOSImages` 获取可用值 | 指定后禁止在 `LaunchConfigurePara` 中设置 `ImageId`，否则返回 `InvalidParameter.Param: image id can not be set` |
| `ContainerRuntime` | String | 否 | 容器运行时：`containerd`（推荐）或 `docker` | 仅独立集群支持 docker |
| `RuntimeVersion` | String | 否 | 运行时版本，如 `1.6.9` | 版本不匹配返回错误 |
| `DeletionProtection` | Boolean | 否 | 删除保护，默认 `false`。`true` 时需先关闭才能删节点池 | 忘开删除保护可能导致节点池被误删 |

### LaunchConfigurePara（CVM 启动配置，JSON 字符串内部字段）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceType` | String | 是 | CVM 机型，如 `S5.MEDIUM2`。`tccli cvm DescribeZoneInstanceConfigInfos` 查询库存 | 机型不在目标可用区售卖返回 `InvalidParameterValue` |
| `InstanceChargeType` | String | 是 | `POSTPAID_BY_HOUR`（按量）或 `PREPAID`（包年包月） | 缺少返回 `InvalidParameter.Param: InstanceChargeType must be set` |
| `SystemDisk.DiskType` | String | 否 | 系统盘类型：`CLOUD_PREMIUM`、`CLOUD_SSD` 等 | 无效类型返回错误 |
| `SystemDisk.DiskSize` | Integer | 否 | 系统盘大小（GB），最小 50 | 小于最小值返回错误 |
| `SecurityGroupIds` | Array\<String\> | 是 | 安全组 ID 列表 | 安全组不存在返回错误；缺少返回 `InvalidParameter.Param: security group ids is not set` |
| `InternetAccessible.InternetChargeType` | String | 否 | 公网计费类型：`TRAFFIC_POSTPAID_BY_HOUR` 等 | — |
| `InternetAccessible.InternetMaxBandwidthOut` | Integer | 否 | 公网出带宽上限（Mbps），0 表示不分配公网 IP | — |

### AutoScalingGroupPara（弹性伸缩组配置，JSON 字符串内部字段）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `VpcId` | String | 是 | VPC ID，格式 `vpc-xxxxxxxx` | VPC 不存在返回 `ResourceNotFound` |
| `SubnetIds` | Array\<String\> | 是 | **复数 `SubnetIds`**（非 `SubnetId`），子网 ID 数组 | 写成 `SubnetId`（单数）返回 `FailedOperation.AsCommon: 未定义参数 SubnetId` |
| `MinSize` | Integer | 是 | 最小节点数，需 <= MaxSize | 缺少返回 `InvalidParameter.Param: MinSize and MaxSize must be set` |
| `MaxSize` | Integer | 是 | 最大节点数，需 >= MinSize | 缺少返回 `InvalidParameter.Param: MinSize and MaxSize must be set` |
| `DesiredCapacity` | Integer | 否 | 期望节点数，需在 Min/Max 区间内。建议设置以确保立即创建节点 | 超出范围返回错误 |

> **关键注意事项：**
> - `EnableAutoscale` 是必填参数——即使创建固定大小节点池也必须显式传入 `false`。
> - `NodePoolOs` 已指定操作系统时，禁止在 `LaunchConfigurePara` 中设置 `ImageId`。
> - `AutoScalingGroupPara` 中必须使用 `SubnetIds`（复数、数组格式），不能写成 `SubnetId`（单数）。
> - `MinSize` 和 `MaxSize` 为必填参数，即使创建固定大小节点池也必须设置。
> - `InstanceChargeType` 为必填参数，必须放在 `LaunchConfigurePara` 中。
> - `LaunchConfigurePara` 和 `AutoScalingGroupPara` 是 JSON **字符串**（非嵌套对象），在请求 JSON 中需转义引号。

## 操作步骤

### 步骤 1：创建节点池

#### 选择依据

- **机型**：选择 `S5.MEDIUM2`（标准型 S5，2C2G）。这是最小规格，适合功能验证。经 `DescribeZoneInstanceConfigInfos` 确认目标可用区有库存。替代方案：`S5.MEDIUM4`（2C4G）或 `S5.LARGE8`（4C8G），生产环境建议根据工作负载选择更大规格。
- **操作系统**：选择 `ubuntu22.04x86_64`（Ubuntu 22.04 LTS）。通过 `NodePoolOs` 参数直接指定，禁止在 `LaunchConfigurePara` 中额外指定 `ImageId`，否则返回 `InvalidParameter.Param: image id can not be set`。
- **系统盘**：选择 `CLOUD_PREMIUM` 50GB。高性能云硬盘最小容量，适合测试。可替换为 `CLOUD_SSD` 获得更好 I/O 性能。
- **弹性伸缩**：选择 `EnableAutoscale: false`。创建固定大小节点池（MinSize=MaxSize=1），不需要弹性伸缩。若需自动扩缩，设 `EnableAutoscale=true` 并指定 `MinSize < MaxSize`。**此参数为必填**，即使不启用弹性伸缩也必须显式传入 `false`。
- **计费模式**：选择 `POSTPAID_BY_HOUR`（按量付费）。用完即删，验证成本最低。
- **删除保护**：选择 `false`。测试节点池无需删除保护。生产环境建议开启，防止误删。

#### 最小创建（只含必填字段）

`create-nodepool-minimal.json`：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "EnableAutoscale": false,
    "InstanceAdvancedSettings": {},
    "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SecurityGroupIds\":[\"sg-xxxxxxxx\"]}",
    "AutoScalingGroupPara": "{\"VpcId\":\"vpc-xxxxxxxx\",\"SubnetIds\":[\"subnet-xxxxxxxx\"],\"MinSize\":1,\"MaxSize\":1,\"DesiredCapacity\":1}"
}
```

```bash
tccli tke CreateClusterNodePool \
    --cli-input-json file://create-nodepool-minimal.json \
    --region <Region>
# expected: exit 0，返回 NodePoolId
```

```json
{
    "NodePoolId": "np-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（加操作系统、运行时、磁盘、安全组、删除保护）

`create-nodepool-enhanced.json`：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "Name": "NODE_POOL_NAME",
    "NodePoolOs": "ubuntu22.04x86_64",
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "1.6.9",
    "DeletionProtection": false,
    "EnableAutoscale": false,
    "InstanceAdvancedSettings": {},
    "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"InternetAccessible\":{\"InternetChargeType\":\"TRAFFIC_POSTPAID_BY_HOUR\",\"InternetMaxBandwidthOut\":0},\"SecurityGroupIds\":[\"sg-xxxxxxxx\"],\"InstanceChargeType\":\"POSTPAID_BY_HOUR\"}",
    "AutoScalingGroupPara": "{\"VpcId\":\"vpc-xxxxxxxx\",\"SubnetIds\":[\"subnet-xxxxxxxx\"],\"MinSize\":1,\"MaxSize\":1,\"DesiredCapacity\":1}"
}
```

```bash
tccli tke CreateClusterNodePool \
    --cli-input-json file://create-nodepool-enhanced.json \
    --region <Region>
# expected: exit 0，返回 NodePoolId
```

```json
{
    "NodePoolId": "np-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NODE_POOL_NAME` | 节点池名称 | 2-64 字符 | 自定义 |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，必须为 Running 状态 | `tccli tke DescribeClusters` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，AvailableIpAddressCount > 0 | `tccli vpc DescribeSubnets` |
| `<VpcId>` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs` |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups`，从返回的 `SecurityGroupId` 字段取值 |
| `<Zone>` | 可用区 | 如 `ap-guangzhou-3` | `tccli cvm DescribeZones` |

## 验证

节点池创建是异步操作。CVM 初始化并加入集群通常需 3-5 分钟。轮询直到节点状态为 `Normal`。

### 1. 查看节点池列表

`DescribeClusterNodePools` 返回集群下所有节点池。注意：此命令不支持 `--NodePoolIds` 参数过滤单个节点池，需在返回列表中查找目标。

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: TotalCount >= 1，列表含新建节点池，LifeState 为 normal
```

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-xxxxxxxx",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "NodePoolOs": "ubuntu22.04x86_64"
        },
        {
            "NodePoolId": "np-yyyyyyyy",
            "Name": "OTHER_NODE_POOL_NAME",
            "LifeState": "normal",
            "NodePoolOs": "tlinux3.1x86_64"
        }
    ]
}
```

### 2. 查看节点池详情（多维度验证）

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: LifeState: normal, AutoscalingGroupStatus: disabled, AutoscalingAdded.Total >= 1
```

```json
{
    "NodePoolId": "np-xxxxxxxx",
    "Name": "NODE_POOL_NAME",
    "ClusterInstanceId": "cls-xxxxxxxx",
    "LifeState": "normal",
    "LaunchConfigurationId": "asc-xxxxxxxx",
    "AutoscalingGroupId": "asg-xxxxxxxx",
    "AutoscalingGroupStatus": "disabled",
    "MaxNodesNum": 1,
    "MinNodesNum": 1,
    "DesiredNodesNum": 1,
    "RuntimeConfig": {
        "RuntimeType": "containerd",
        "RuntimeVersion": "1.6.9"
    },
    "NodePoolOs": "ubuntu22.04x86_64",
    "OsCustomizeType": "GENERAL",
    "ImageId": "",
    "DesiredPodNum": 64,
    "UserScript": "",
    "Tags": [],
    "DeletionProtection": false,
    "NodeCountSummary": {
        "ManuallyAdded": {
            "Joining": 0,
            "Initializing": 0,
            "Normal": 0,
            "Total": 0
        },
        "AutoscalingAdded": {
            "Joining": 1,
            "Initializing": 0,
            "Normal": 0,
            "Total": 1
        }
    }
}
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 生命周期 | `LifeState` | `normal` |
| 伸缩组状态 | `AutoscalingGroupStatus` | `disabled`（EnableAutoscale=false 时） |
| OS 匹配 | `NodePoolOs` | `ubuntu22.04x86_64`（与创建参数一致） |
| 运行时 | `RuntimeConfig.RuntimeType` / `RuntimeVersion` | `containerd` / `1.6.9` |
| 节点规模 | `MinNodesNum` / `MaxNodesNum` / `DesiredNodesNum` | `1` / `1` / `1` |
| 节点加入 | `NodeCountSummary.AutoscalingAdded` | `Joining` 或 `Normal` 总和 = `DesiredNodesNum` |

> 节点正在加入（`Joining: 1`）是正常的——CVM 初始化和加入集群通常需 3-5 分钟。`AutoscalingGroupStatus` 为 `disabled` 是因为 `EnableAutoscale=false`（固定大小节点池不启用弹性伸缩），属预期行为。

### 3. 查看集群工作节点实例

```bash
tccli tke DescribeClusterInstances \
    --ClusterId <ClusterId> \
    --InstanceRole "WORKER" \
    --region <Region>
# expected: 返回 WORKER 实例列表，含 NodePoolId 与新建节点池匹配的实例
```

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-xxxxxxxx",
            "InstanceRole": "WORKER",
            "InstanceState": "initializing",
            "LanIP": "172.24.0.45",
            "NodePoolId": ""
        },
        {
            "InstanceId": "ins-yyyyyyyy",
            "InstanceRole": "WORKER",
            "InstanceState": "initializing",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-xxxxxxxx"
        }
    ]
}
```

> `InstanceState` 为 `initializing` 是正常的——节点正在初始化。`NodePoolId` 匹配新建节点池的实例即为目标节点池创建的 CVM。初始化完成后状态变为 `running`。

## 清理

> **⚠️ 警告**：`DeleteClusterNodePool` 配合 `--KeepInstance false` 会**同时销毁节点池内所有 CVM 实例及其云硬盘**，数据不可恢复。
> 生产环境执行前务必用 `DescribeClusterNodePoolDetail` 确认 `NodePoolId` 正确，建议先缩容至 0 再删除，避免误删运行中的工作负载。
> 如果不删除这些资源，将持续产生费用。CVM 按量计费（S5.MEDIUM2 约 0.08 元/小时），验证完成后及时清理。

### 1. 清理前状态检查

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# 确认是待删除的目标节点池，记录 NodePoolId、Name、NodeCountSummary
# expected: 返回节点池详情，LifeState 为 normal
```

```json
{
    "NodePoolId": "np-xxxxxxxx",
    "Name": "NODE_POOL_NAME",
    "LifeState": "normal",
    "NodeCountSummary": {
        "AutoscalingAdded": {
            "Total": 1
        }
    }
}
```

### 2. 删除节点池

```bash
tccli tke DeleteClusterNodePool \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false \
    --region <Region>
# ⚠️ --KeepInstance false 会删除节点池中的 CVM 实例及云硬盘
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 如仅需删除节点池记录而保留 CVM 实例，使用 `--KeepInstance true`。

> **注意**：节点池删除是异步操作，`DescribeClusterNodePools` 在删除请求发出后可能仍显示该节点池（`LifeState` 为 `deleting`）。通常 1-2 分钟后节点池从列表中消失。建议等待后重试，或接受 `LifeState: "deleting"` 为已发起删除的确认信号。

### 3. 验证已删除

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: 列表中不含已删除的 NodePoolId
```

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-yyyyyyyy",
            "Name": "OTHER_NODE_POOL_NAME",
            "LifeState": "normal",
            "NodePoolOs": "tlinux3.1x86_64"
        }
    ]
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `MissingParameter: 请求缺少必传参数 EnableAutoscale` | 检查请求 JSON 顶层是否包含 `EnableAutoscale` 字段 | `EnableAutoscale` 是必填参数，现有文档常遗漏此参数。即使不启用弹性伸缩也必须显式传入 `false` | 在请求 JSON 中添加 `"EnableAutoscale": false`（固定大小节点池）或 `"EnableAutoscale": true`（启用弹性伸缩） |
| `CreateClusterNodePool` 返回 `MissingParameter: 缺少参数 InstanceAdvancedSettings` | 检查请求 JSON 顶层是否包含 `InstanceAdvancedSettings` 字段 | API 要求传入 `InstanceAdvancedSettings`，即使不需要高级设置也必须传空对象 | 添加 `"InstanceAdvancedSettings": {}` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param: InstanceChargeType must be set` | 检查 `LaunchConfigurePara` JSON 字符串中是否包含 `InstanceChargeType` | `InstanceChargeType` 为必填参数 | 在 `LaunchConfigurePara` 中添加 `"InstanceChargeType": "POSTPAID_BY_HOUR"` 或 `"PREPAID"` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param: MinSize and MaxSize must be set` | 检查 `AutoScalingGroupPara` JSON 字符串中是否包含 `MinSize` 和 `MaxSize` | `MinSize` 和 `MaxSize` 为必填参数，即使固定大小节点池也必须设置 | 在 `AutoScalingGroupPara` 中添加 `"MinSize": 1, "MaxSize": 1` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param: image id can not be set` | 检查 `LaunchConfigurePara` 中是否设置了 `ImageId`，同时检查顶层 `NodePoolOs` | `NodePoolOs` 已指定操作系统后，禁止在 `LaunchConfigurePara` 中重复指定 `ImageId` | 移除 `LaunchConfigurePara` 中的 `ImageId` 字段，仅通过 `NodePoolOs` 指定操作系统 |
| `CreateClusterNodePool` 返回 `FailedOperation.AsCommon: 未定义参数 SubnetId` | 检查 `AutoScalingGroupPara` 中的参数名称拼写 | 应使用 `SubnetIds`（复数、数组格式），而非 `SubnetId`（单数） | 将 `"SubnetId"` 改为 `"SubnetIds": ["subnet-xxxxxxxx"]` |
| `CreateClusterNodePool` 返回 `InvalidParameterValue` — `InstanceType` 不可用 | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["S5"]}]'` | 指定机型在目标可用区无库存或不在售 | 更换机型或更换可用区，确认 `Status` 为 `SELL` |
| `CreateClusterNodePool` 返回 `ResourceNotFound` — Vpc 或 Subnet | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>"]'` + `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | VpcId 或 SubnetIds 不存在或格式错误 | 确认 VPC 和子网 ID 正确，子网 `AvailableIpAddressCount > 0` |
| `DeleteClusterNodePool` 返回 `FailedOperation` — 删除保护已开启 | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>` 检查 `DeletionProtection` | 节点池开启了删除保护 | 先通过 `ModifyClusterNodePool` 关闭删除保护，再执行删除 |

### 创建成功但节点未就绪

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回了 `NodePoolId` 但 `DescribeClusterNodePoolDetail` 显示 `NodeCountSummary.AutoscalingAdded.Joining` 长时间为 1 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole "WORKER" --region <Region>` 查看 `InstanceState` | CVM 初始化和加入集群通常需 3-5 分钟，属正常等待 | 继续等待；超过 10 分钟仍为 `Joining` 则保留 region、ClusterId、NodePoolId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/nodepool) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| 节点池 `LifeState` 为 `abnormal` | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>` 查看详情 | 通常是 CVM 创建失败（配额不足、机型售罄、安全组规则错误等） | 检查 CVM 配额：`tccli cvm DescribeAccountQuota`；检查机型库存：`tccli cvm DescribeZoneInstanceConfigInfos`；修正后删除异常节点池重新创建 |
| `DescribeClusterNodePools` 返回列表但找不到新建节点池 | 确认 `--ClusterId` 是否正确，检查 `TotalCount` | 传错了 `ClusterId`，节点池创建在了其他集群 | 用 `tccli tke DescribeClusters --region <Region>` 确认正确的 `ClusterId` |

### 格式要求

- 错误码如 `MissingParameter`、`InvalidParameter.Param` → 必须完整写出，不可简写
- 排障表分两组：**命令返回错误**（exit ≠ 0）和 **创建已提交但状态异常**（exit = 0 但状态不对）
- 修复命令必须可直接 COPY-PASTE
- 涉及 RequestId 的排查路径 → 提醒保留 RequestId 以备工单/日志查询
- 已知不可抗力（`LimitExceeded` / `ResourcesSoldOut` / `CamNoAuth`）→ 注明"此为环境限制，非命令错误"

## 下一步

- [查看节点池](https://cloud.tencent.com/document/product/457/43736) — 查询节点池详情和节点状态
- [调整节点池](https://cloud.tencent.com/document/product/457/43737) — 修改节点池配置和规模
- [删除节点池](https://cloud.tencent.com/document/product/457/43738) — 删除不再需要的节点池
- [环境准备](../../../环境准备.md) — tccli 安装与配置 <!-- 发布时由 sync 脚本替换为平台兼容链接 -->

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 新建](https://console.cloud.tencent.com/tke2/nodepool/create)
