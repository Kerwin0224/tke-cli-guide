# 设置节点的启动脚本

> 对照官方：[设置节点的启动脚本](https://cloud.tencent.com/document/product/457/32206) · page_id `32206`

## 概述

节点启动脚本（UserScript）用于在节点加入 TKE 集群时自动执行自定义初始化操作。TKE 支持两种启动脚本：

- **UserScript**：在 kubelet 启动后执行，适用于大多数初始化任务（安装监控 Agent、配置内核参数等）。
- **PreStartUserScript**：在 kubelet 启动前执行，适用于系统级预配置（格式化磁盘、调整系统服务等）。

脚本内容必须以 Base64 编码后传入 API。支持两种操作场景：

| 场景 | API | 使用时机 |
|------|-----|---------|
| 修改已有节点池的启动脚本 | `ModifyClusterNodePool` | 节点池已存在，需新增或修改启动脚本，仅对新扩容节点生效 |
| 创建节点池时指定启动脚本 | `CreateClusterNodePool` | 新建节点池，初始配置即带启动脚本 |

> **注意**：`ModifyClusterNodePool` 设置的 UserScript **仅对新扩容节点生效**，不会在已有运行节点上重新执行。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail
#    tke:ModifyClusterNodePool
#    tke:CreateClusterNodePool
#    tke:DeleteClusterNodePool
#    tke:DescribeSupportedRuntime
#    cvm:DescribeZoneInstanceConfigInfos
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）
```

### 资源检查

```bash
# 4. 确认集群已存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterStatus 为 "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 5. 列出节点池（确认目标节点池存在，用于修改场景）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点池列表，记录目标 NodePoolId
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-01",
            "Name": "example-np",
            "LifeState": "normal",
            "UserScript": ""
        },
        {
            "NodePoolId": "np-example-02",
            "Name": "example-np-2",
            "LifeState": "normal",
            "UserScript": ""
        }
    ],
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池启动脚本 | `DescribeClusterNodePoolDetail`（查看 `UserScript` / `PreStartUserScript` 字段） | 是 |
| 修改已有节点池启动脚本 | `ModifyClusterNodePool` | 是 |
| 新建节点池并设置启动脚本 | `CreateClusterNodePool`（`InstanceAdvancedSettings.UserScript`） | 否 |

## 关键字段说明

`ModifyClusterNodePool` 和 `CreateClusterNodePool` 中与启动脚本相关的参数说明：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `UserScript` | String | 否 | Base64 编码的 Shell 脚本。通过 `base64 < script.sh` 生成。在 kubelet 启动后执行 | 传明文不编码 → 换行/引号等特殊字符可能使脚本损坏 |
| `PreStartUserScript` | String | 否 | Base64 编码的 Shell 脚本。在 kubelet 启动前执行。生成方式同 UserScript | 同上 |
| `EnableAutoscale` | Boolean | 是 | `true` 或 `false`（仅 `CreateClusterNodePool`） | 缺失 → `MissingParameter` |
| `RuntimeVersion` | String | 是 | 如 `1.6.9`。必须与集群 K8s 版本匹配。`DescribeSupportedRuntime` 查询 | 不匹配 → `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError` |
| `LaunchConfigurePara` | String | 是 | JSON 字符串（仅 `CreateClusterNodePool`）。必须包含 `InstanceType`、`InstanceChargeType`、`SecurityGroupIds`、`SystemDisk`、`InternetAccessible`、`LoginSettings`。**不可含 `ImageId`**（由 `NodePoolOs` 控制） | 缺必填字段 → `InvalidParameter.Param` / `MissingParameter` |

## 操作步骤

### 步骤 1：修改已有节点池的启动脚本

#### 选择依据

- **操作方式**：选择 `ModifyClusterNodePool` 而非删除重建节点池。UserScript 修改不触发已有节点重启——仅对新扩容节点生效。如需已有节点也执行脚本，应通过扩容节点池使新节点加入。
- **UserScript vs PreStartUserScript**：UserScript 在 kubelet 启动后执行，适合大多数应用层初始化。PreStartUserScript 在 kubelet 启动前执行，适合系统级预配置。两者互不冲突，可同时设置。执行顺序：先 PreStartUserScript，后 kubelet 启动，最后 UserScript。
- **脚本编码**：UserScript 和 PreStartUserScript 必须 Base64 编码传入。`base64 < script.sh` 生成编码内容。编解码是 API 契约——直接传明文 Shell 脚本可能导致换行/引号等特殊字符损坏。

#### 最小修改（仅设置 UserScript）

`modify-np-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "UserScript": "<USER_SCRIPT_BASE64>",
    "DeletionProtection": false
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-np-minimal.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` |
| `<USER_SCRIPT_BASE64>` | Base64 编码的 Shell 脚本 | `base64 < script.sh` 生成 | 自定义 Shell 脚本内容 |

#### 增强配置（同时设置 UserScript 和 PreStartUserScript）

`modify-np-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "UserScript": "<USER_SCRIPT_BASE64>",
    "PreStartUserScript": "<PRESTART_SCRIPT_BASE64>",
    "DeletionProtection": false
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-np-enhanced.json
# expected: exit 0，返回 RequestId
```

### 步骤 2：创建节点池时指定启动脚本

#### 选择依据

新建节点池时通过 `InstanceAdvancedSettings.UserScript` 和 `InstanceAdvancedSettings.PreStartUserScript` 指定启动脚本。

- **RuntimeVersion**：通过 `DescribeSupportedRuntime --K8sVersion` 查询支持的 containerd 版本。填写错误的版本号（如 `1.6.28` 而非 `1.6.9`）会触发 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`。
- **机型选择**：通过 `cvm DescribeZoneInstanceConfigInfos` 在目标可用区确认在售机型。部分机型（如 SA9 系列）在特定可用区已停售，触发 `FailedOperation.AsCommon`。建议先用 Describe 命令确认可用区机型后再填写 `LaunchConfigurePara.InstanceType`。
- **LaunchConfigurePara 编排规则**：`LaunchConfigurePara` 为 JSON 字符串。**不可设置 `ImageId`**（镜像由 `NodePoolOs` 控制），**不可设置 `AutoScalingGroupName`**（系统自动生成）。必须包含：`InstanceChargeType`、`InstanceType`、`SystemDisk`、`InternetAccessible`、`SecurityGroupIds`、`LoginSettings`。

#### 最小创建

`create-np-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "<NodePoolName>",
    "EnableAutoscale": false,
    "AutoScalingGroupPara": "{\"MaxSize\":1,\"MinSize\":0,\"DesiredCapacity\":0,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"],\"RetryPolicy\":\"IMMEDIATE_RETRY\"}",
    "LaunchConfigurePara": "{\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"InstanceType\":\"<InstanceType>\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"InternetAccessible\":{\"InternetMaxBandwidthOut\":0,\"PublicIpAssigned\":false},\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"LoginSettings\":{\"Password\":\"<Password>\"}}",
    "InstanceAdvancedSettings": {
        "UserScript": "<USER_SCRIPT_BASE64>"
    },
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "<RuntimeVersion>",
    "NodePoolOs": "<NodePoolOs>",
    "DeletionProtection": false
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-np-minimal.json
# expected: exit 0，返回 NodePoolId 和 RequestId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolName>` | 节点池名称 | 长度 1-60，以字母开头 | 自定义 |
| `<VpcId>` | VPC ID | 集群所在 VPC | `tccli vpc DescribeVpcs --region <Region>` |
| `<SubnetId>` | 子网 ID | 可用 IP ≥3 | `tccli vpc DescribeSubnets --region <Region>` |
| `<InstanceType>` | 实例机型 | 需在目标可用区在售 | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region>` |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `<Password>` | 节点登录密码 | 8-16 字符，含大小写字母和数字 | 自定义 |
| `<RuntimeVersion>` | 容器运行时版本 | 与 K8s 版本匹配 | `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion <Version>` |
| `<NodePoolOs>` | 节点操作系统 | 如 `tlinux3.1x86_64` | `tccli tke DescribeClusterNodePoolDetail` 查询已有节点池 |
| `<USER_SCRIPT_BASE64>` | Base64 编码的脚本 | `base64 < script.sh` 生成 | 自定义 Shell 脚本内容 |

## 验证

### 控制面（tccli）

修改或创建后，确认 UserScript 已设置：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，NodePool.UserScript 非空，LifeState 为 normal
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-np",
        "LifeState": "normal",
        "UserScript": "SET",
        "PreStartUserScript": "SET",
        "DesiredNodesNum": 0,
        "NodeCountSummary": {
            "Total": 0
        }
    },
    "RequestId": "..."
}
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| UserScript | `DescribeClusterNodePoolDetail` 检查 `NodePool.UserScript` | 非空字符串（已设置的 Base64 编码内容） |
| PreStartUserScript | 同上，检查 `NodePool.PreStartUserScript` | 如设置则非空，未设置则为空字符串 |
| 节点池状态 | 同上，检查 `NodePool.LifeState` | `normal` |
| 节点数 | 同上，检查 `NodePool.NodeCountSummary.Total` | 若 `DesiredNodesNum=0`，预期为 `0`。这是预期行为——UserScript 在后续扩容新节点时执行 |

> **关于空节点池**：`DesiredCapacity=0` 的节点池无实际运行节点。UserScript 不会立即执行，而是在后续扩容的新节点上执行。节点数为 0 是预期行为，不代表配置失败。

## 清理

> **警告**：`DeleteClusterNodePool` 配合 `KeepInstance false`（默认值）会**级联销毁**节点池内的所有 CVM 实例及关联磁盘。如果 ModifyClusterNodePool 操作对象是已有生产节点池，恢复 UserScript 为空即可，**不要删除节点池**。

### 场景 A：清除 UserScript（恢复默认，适用于 ModifyClusterNodePool）

`modify-cleanup.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "UserScript": "",
    "PreStartUserScript": "",
    "DeletionProtection": false
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-cleanup.json
# expected: exit 0，返回 RequestId
```

### 场景 B：删除测试节点池（适用于 CreateClusterNodePool）

#### 1. 清理前状态检查

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 确认是待删除的目标节点池，记录 NodePoolId、DesiredNodesNum、NodeCountSummary.Total
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "test-np-to-delete",
        "LifeState": "normal",
        "DesiredNodesNum": 0,
        "NodeCountSummary": {
            "Total": 0
        }
    },
    "RequestId": "..."
}
```

#### 2. 删除节点池

```bash
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false
# KeepInstance false 会级联删除节点池内所有 CVM 实例及磁盘
```

#### 3. 验证已删除

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回列表中不含已删除的 NodePoolId
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-01",
            "Name": "example-np",
            "LifeState": "normal"
        }
    ],
    "RequestId": "..."
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `MissingParameter`，提示缺 `EnableAutoscale` | 检查请求 JSON 是否含 `EnableAutoscale` | `EnableAutoscale` 是 `CreateClusterNodePool` 的必传顶层参数，即使设为 `false` 也必须显式传入 | 在请求 JSON 中添加 `"EnableAutoscale": false` |
| `CreateClusterNodePool` 返回 `MissingParameter`，提示缺 `LaunchConfigurePara` | 检查请求 JSON 中 `LaunchConfigurePara` 字段 | `LaunchConfigurePara` 是必传的 JSON 字符串，缺或格式错误 | 添加 `"LaunchConfigurePara": "{...}"`（JSON 字符串），确保含 `InstanceChargeType`、`InstanceType`、`SystemDisk`、`InternetAccessible`、`SecurityGroupIds`、`LoginSettings` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`，提示 `autoscaling group name can not be set` | 检查 `AutoScalingGroupPara` JSON 字符串是否含 `AutoScalingGroupName` | TKE 不允许自定义伸缩组名称，系统自动生成 | 从 `AutoScalingGroupPara` 中移除 `AutoScalingGroupName` 字段 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`，提示 `image id can not be set` | 检查 `LaunchConfigurePara` JSON 字符串是否含 `ImageId` | TKE 不允许自定义镜像 ID，镜像由 `NodePoolOs` 参数控制 | 从 `LaunchConfigurePara` 中移除 `ImageId` 字段，镜像通过 `NodePoolOs` 指定 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`，提示 `InstanceType is not set` | 检查 `LaunchConfigurePara` 中是否含 `InstanceType` | `LaunchConfigurePara` 必须包含 `InstanceType` | 在 `LaunchConfigurePara` 中添加 `"InstanceType": "<InstanceType>"`，通过 `tccli cvm DescribeZoneInstanceConfigInfos` 确认在售机型 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`，提示 `security group ids is not set` | 检查 `LaunchConfigurePara` 中是否含 `SecurityGroupIds` | `LaunchConfigurePara` 必须包含 `SecurityGroupIds` | 在 `LaunchConfigurePara` 中添加 `"SecurityGroupIds": ["<SecurityGroupId>"]` |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`，提示 `InstanceChargeType must be set` | 检查 `LaunchConfigurePara` 中是否含 `InstanceChargeType` | `LaunchConfigurePara` 必须包含 `InstanceChargeType`（值 `POSTPAID_BY_HOUR` 或 `PREPAID`） | 在 `LaunchConfigurePara` 中添加 `"InstanceChargeType": "POSTPAID_BY_HOUR"` |
| `CreateClusterNodePool` 返回 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError` | `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion <K8sVersion>` 查询支持的运行时版本列表 | 填写的 `RuntimeVersion` 与 K8s 版本不匹配 | 将 `RuntimeVersion` 改为 `DescribeSupportedRuntime` 返回的有效版本。注意版本号精确：如 K8s 1.32.2 支持 containerd `1.6.9`，填写 `1.6.28` 会报错 |
| `CreateClusterNodePool` 返回 `FailedOperation.AsCommon` / `ResourceUnavailable.InstanceType` | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"zone","Values":["<Zone>"]}]'` 确认可用机型 | 目标机型在所选可用区已停售（此为环境限制） | 切换到 `DescribeZoneInstanceConfigInfos` 确认在售的替代机型，如 SA9 系列不可用则改用 S5 等同规格机型 |

### 操作成功但不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回成功但节点未执行脚本 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 查看 `UserScript` 和 `NodeCountSummary` | UserScript 仅对新扩容节点生效，已有运行节点不会重新执行脚本 | 扩容节点池（增加 `DesiredCapacity`）触发新节点创建，脚本将在新节点上执行。可在节点 `/usr/local/qcloud/tke/userscript` 路径查看脚本内容 |
| `ModifyClusterNodePool` 后节点池 `LifeState` 长时间为 `updating` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 轮询查看状态 | 节点池更新中，通常片刻完成 | 等待更新完成。超过 10 分钟仍为 `updating` → 保留 `Region`、`ClusterId`、`NodePoolId`、`RequestId`，登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/node-pool) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| 脚本执行失败但节点正常 `Running` | SSH 登录节点查看 `/usr/local/qcloud/tke/userscript` 路径下的脚本及日志 | 脚本自身错误（语法错误、权限不足、依赖缺失），TKE 不因 UserScript 失败而阻止节点加入集群 | 修正脚本内容后重新设置 UserScript，扩容新节点验证脚本执行结果 |

## 下一步

- [创建集群](https://cloud.tencent.com/document/product/457/32191) — 如尚未创建集群
- [创建节点池](https://cloud.tencent.com/document/product/457/32205) — 了解节点池完整创建流程
- [查看节点池列表](https://cloud.tencent.com/document/product/457/32204) — 查看和管理已有节点池
- [TKE API 文档 — ModifyClusterNodePool](https://cloud.tencent.com/document/api/457/61402)
- [TKE API 文档 — CreateClusterNodePool](https://cloud.tencent.com/document/api/457/61401)

## 控制台替代

[TKE 控制台 → 节点管理 → 节点池](https://console.cloud.tencent.com/tke2/node-pool)：选择目标节点池 → 「更多」→ 「设置启动脚本」。
