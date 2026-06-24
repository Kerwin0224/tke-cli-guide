# 创建注册节点（专线版）

> 对照官方：[创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/57917) · page_id `57917` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过专线版方式将 IDC 或第三方云主机注册到 TKE 集群。专线版使用 `HostNetwork` 网络模式，宿主机与 Pod 共享同一网络命名空间，Pod IP 即宿主机 IP，通过专线/VPN 与 TKE VPC 互通。另一种选择 `CiliumBGP` 用于公网版。

完整流程分三步：

1. `EnableExternalNodeSupport` — 开启注册节点支持（异步操作，约 3-5 分钟）
2. `CreateExternalNodePool` — 创建注册节点池
3. `DescribeExternalNodeScript` — 获取初始化脚本并在 IDC 主机上执行

| 网络模式 | 适用场景 | 网络要求 |
|---------|---------|---------|
| `HostNetwork`（专线版） | IDC 通过专线/VPN 与 VPC 互通 | 宿主机与 Pod 共享网络命名空间，Pod IP 即宿主机 IP |
| `CiliumBGP`（公网版） | IDC 无专线，通过公网注册 | 需要 BGP 支持 |

> 本文档中所有 ID 均为占位符，请替换为实际值。

## 前置条件

- [环境准备](../../../../环境准备.md)（发布到 iWiki/写写前需转换为平台链接）

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
#    tke:DescribeExternalNodeSupportConfig
#    tke:EnableExternalNodeSupport
#    tke:CreateExternalNodePool
#    tke:DescribeExternalNodePools
#    tke:DescribeExternalNodeScript
#    tke:DescribeExternalNode
#    tke:DeleteExternalNodePool
#    tke:DrainExternalNode
#    tke:DeleteExternalNode
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running，ClusterType 为 MANAGED_CLUSTER
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterVersion": "1.32.2",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example",
                "Cni": true
            },
            "ClusterNodeNum": 0,
            "ClusterStatus": "Running",
            "ContainerRuntime": "containerd",
            "EnableExternalNode": false
        }
    ]
}
```

```bash
# 5. 检查注册节点支持是否已开启（开启前应为 Disabled）
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status 为 "Disabled"（未开启）；如已为 "Enabled" 则跳过步骤 1
```

```json
{
    "ClusterCIDR": "",
    "NetworkType": "GR",
    "SubnetId": "",
    "Enabled": false,
    "Status": "Disabled",
    "EnabledPublicConnect": false
}
```

```bash
# 6. 确认专线/VPN 已建立（IDC 与 VPC 互通）
#    专线版要求 IDC 网络通过专线或 VPN 与 TKE VPC 互通
#    无对应 tccli 直接检查命令，请登录控制台或联系网络管理员确认
```

```bash
# 7. 获取集群 VPC 下的子网 ID
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回子网列表，记录目标子网的 SubnetId（$.SubnetSet[0].SubnetId）
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看注册节点支持配置 | `DescribeExternalNodeSupportConfig` | 是 |
| 开启注册节点支持 | `EnableExternalNodeSupport` | 否 |
| 创建注册节点池 | `CreateExternalNodePool` | 否 |
| 查看注册节点池列表 | `DescribeExternalNodePools` | 是 |
| 获取注册节点初始化脚本 | `DescribeExternalNodeScript` | 是 |
| 查看注册节点列表 | `DescribeExternalNode` | 是 |
| 删除注册节点池 | `DeleteExternalNodePool` | 是 |

### 关键字段说明

以下说明三个核心 API 的参数。完整参数定义见 `tccli tke <Action> help --detail`。

#### EnableExternalNodeSupport 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound` |
| `ClusterExternalConfig.NetworkType` | String | 是 | `HostNetwork`（专线版）或 `CiliumBGP`（公网版）。专线版必须选 `HostNetwork` | 专线版填 `CiliumBGP` → `FailedOperation.NetworkTypeMismatch` |
| `ClusterExternalConfig.SubnetId` | String | 是 | 子网 ID，格式 `subnet-xxxxxxxx`，必须属于集群所在 VPC | 子网不存在或不属于集群 VPC → `InvalidParameter.SubnetId` |
| `ClusterExternalConfig.ClusterCIDR` | String | 否 | `HostNetwork` 模式无需填写 | 填写会被忽略 |
| `ClusterExternalConfig.Enabled` | Boolean | 否 | 已废弃字段，无需填写 | — |

#### CreateExternalNodePool 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 集群不存在 → `ResourceNotFound` |
| `Name` | String | 是 | 节点池名称，1-60 字符 | 命名违规 → `InvalidParameter` |
| `ContainerRuntime` | String | 是 | `containerd`（推荐）。Docker 运行时已停止维护 | 填 `docker` → 不支持 |
| `RuntimeVersion` | String | 是 | 如 `1.7.28`，需与运行时版本匹配 | 版本不匹配 → `InvalidParameter` |
| `Labels` | Array | 否 | Label 对象数组 `[{"Name":"key","Value":"val"}]` | 格式错误 → 请求被拒 |
| `Taints` | Array | 否 | Taint 对象数组 `[{"Key":"k","Value":"v","Effect":"NoSchedule"}]` | Effect 值非法 → `InvalidParameter` |
| `DeletionProtection` | Boolean | 否 | 默认 `false`。`true` 时需先关闭才能删除 | 忘开删除保护 → 可能误删节点池 |
| `NodeType` | String | 否 | `CPU` 或 `GPU` | — |

#### DescribeExternalNodeScript 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 集群不存在 → `ResourceNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-xxxxxxxx` | 节点池不存在 → `ResourceNotFound` |
| `Interface` | String | 否 | 网卡名 | — |
| `Name` | String | 否 | 节点名称 | — |
| `Internal` | Boolean | 否 | 是否内网获取 | — |

## 操作步骤

### 步骤 1：开启注册节点支持（专线版）

#### 选择依据

- **网络类型**：选择 `HostNetwork`（专线版）。专线版注册节点使用 HostNetwork 网络模式，宿主机与 Pod 共享同一网络命名空间，Pod IP 即宿主机 IP，通过专线/VPN 与 TKE VPC 互通。另一种选择 `CiliumBGP` 用于公网版，专线版填 `CiliumBGP` 会导致网络模式不匹配。
- **子网**：使用集群所在 VPC 下的子网。注册节点的 Proxy 组件将部署在该子网。
- **ClusterCIDR**：`HostNetwork` 模式无需填写此字段。

#### 最小创建

专线版 `EnableExternalNodeSupport` 只有 3 个必填参数（ClusterId、NetworkType、SubnetId），使用 `--cli-unfold-argument` 行内传递嵌套对象：

```bash
tccli tke EnableExternalNodeSupport --region <Region> \
    --cli-unfold-argument \
    --ClusterId <ClusterId> \
    --ClusterExternalConfig.NetworkType HostNetwork \
    --ClusterExternalConfig.SubnetId <SubnetId>
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，须属于集群 VPC | 从 DescribeClusters 输出的 `$.ClusterNetworkSettings.SubnetId` 获取，或 `tccli vpc DescribeSubnets --region <Region>` |

#### 轮询确认（异步操作）

`EnableExternalNodeSupport` 是异步操作，约 3-5 分钟完成。过程中 `Status` 会经历 `Initializing` → `Enabled`。轮询直到 `Status` 变为 `Enabled`：

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Enabled"，Progress 所有步骤 Status 为 "success"
```

```json
{
    "ClusterCIDR": "",
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Enabled": false,
    "Status": "Enabled",
    "Master": "10.0.0.2",
    "Proxy": "10.0.0.19",
    "Progress": [
        {"Name": "EnsureNodeMasterCommunicate", "Status": "success"},
        {"Name": "EnsureInClusterAccess", "Status": "success"},
        {"Name": "EnsureExternalNodeProxy", "Status": "success"},
        {"Name": "EnsureBootStrap", "Status": "success"},
        {"Name": "EnsureAdmissionController", "Status": "success"},
        {"Name": "AdaptNetworkPlugin", "Status": "success"},
        {"Name": "InstallUnderlayCilium", "Status": "success"},
        {"Name": "AdaptTkePlatApp", "Status": "success"},
        {"Name": "CompleteEnable", "Status": "success"}
    ],
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "203.0.113.10:9000"
}
```

> **注意**：`Enabled` 字段已废弃，以 `Status` 为准。`Status` 为 `Initializing` 是正常中间状态，继续轮询即可。在 `Initializing` 期间 API 允许调用 `CreateExternalNodePool`，但建议等待 `Enabled` 后再创建节点池。最长等待约 5 分钟，超时参见 [排障](#排障)。

轮询验证维度：

| 维度 | 预期 |
|------|------|
| Status | `Enabled` |
| NetworkType | `HostNetwork` |
| SubnetId | 与开启时指定一致 |
| Progress | 所有 9 个步骤 `Status` 为 `success` |
| Master | 已分配 Master IP |
| Proxy | 已分配 Proxy IP |

### 步骤 2：创建注册节点池

#### 选择依据

- **运行时**：选择 `containerd`，版本 `1.7.28`。Docker 运行时已停止维护，新节点统一用 containerd。通过 `DescribeExternalNodePools` 查看 `RuntimeConfig` 可确认运行时配置。
- **删除保护**：生产环境建议开启 `DeletionProtection: true`，防止误删节点池。

#### 最小创建

`CreateExternalNodePool` 有 4 个必填参数，使用 `--cli-input-json file://` 传递：

`node-pool-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "<NodePoolName>",
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "1.7.28"
}
```

```bash
cat > node-pool-minimal.json <<'EOF'
{"ClusterId":"<ClusterId>","Name":"<NodePoolName>","ContainerRuntime":"containerd","RuntimeVersion":"1.7.28"}
EOF
tccli tke CreateExternalNodePool --region <Region> \
    --cli-input-json file://node-pool-minimal.json
# expected: exit 0，返回 NodePoolId
```

```json
{
    "NodePoolId": "np-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolName>` | 节点池名称 | 1-60 字符 | 自定义 |

#### 增强配置

`node-pool-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "<NodePoolName>",
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "1.7.28",
    "Labels": [
        {"Name": "env", "Value": "idc"},
        {"Name": "node-type", "Value": "external"}
    ],
    "Taints": [
        {"Key": "idc-only", "Value": "true", "Effect": "NoSchedule"}
    ],
    "DeletionProtection": true
}
```

```bash
cat > node-pool-enhanced.json <<'EOF'
{"ClusterId":"<ClusterId>","Name":"<NodePoolName>","ContainerRuntime":"containerd","RuntimeVersion":"1.7.28","Labels":[{"Name":"env","Value":"idc"},{"Name":"node-type","Value":"external"}],"Taints":[{"Key":"idc-only","Value":"true","Effect":"NoSchedule"}],"DeletionProtection":true}
EOF
tccli tke CreateExternalNodePool --region <Region> \
    --cli-input-json file://node-pool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

#### 查看节点池详情

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount >= 1，LifeState 为 "normal"
```

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.7.28"
            },
            "Labels": [
                {"Name": "tke.cloud.tencent.com/nodepool-id", "Value": "np-example"},
                {"Name": "node.kubernetes.io/instance-type", "Value": "external"},
                {"Name": "tke.cloud.tencent.com/cbs-mountable", "Value": "false"}
            ],
            "Taints": [],
            "DeletionProtection": false,
            "NodeType": "CPU"
        }
    ]
}
```

### 步骤 3：获取注册节点初始化脚本

#### 选择依据

- **脚本用途**：`DescribeExternalNodeScript` 返回的 `Command` 是 `add2tkectl` 工具的下载命令。在 IDC 或第三方云主机上执行后，自动安装 kubelet、containerd 等组件并加入集群。
- **Token 时效性**：返回的 `Token` 是 COS 临时令牌，有时效性。过期后需重新调用此 API 获取新 Token 和下载链接。

```bash
tccli tke DescribeExternalNodeScript --region <Region> \
    --cli-unfold-argument \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回 Link, Token, Command
```

```json
{
    "Link": "http://externalnode-gz-maz-1253687700.cos.ap-guangzhou.myqcloud.com/user-pkgs/.../add2tkectl?...",
    "Token": "<Token>",
    "Command": "wget --header=\"x-cos-token:<Token>\" 'https://example.cos.<Region>.myqcloud.com/user-pkgs/<ClusterId>/add2tkectl?...' -O add2tkectl-<ClusterId>-<NodePoolId> && chmod +x add2tkectl-<ClusterId>-<NodePoolId>",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | 步骤 2 返回的 `NodePoolId` |

#### 在 IDC 主机上执行脚本

> **注意**：以下命令必须在 IDC 或第三方云主机上执行，当前 CLI 环境无 IDC 主机，无法直接执行。专线版要求 IDC 通过专线/VPN 与 TKE VPC 连通。

在目标 IDC 主机上执行返回的 `Command`，然后运行下载的工具完成节点注册：

```bash
# 在 IDC 主机上执行（非当前 CLI 环境）
# 1. 下载 add2tkectl 工具（使用 API 返回的 Command）
#    wget --header="x-cos-token:<Token>" '<Link>' -O add2tkectl-<ClusterId>-<NodePoolId> && chmod +x add2tkectl-<ClusterId>-<NodePoolId>
#
# 2. 执行工具完成节点注册
#    ./add2tkectl-<ClusterId>-<NodePoolId>
#
# expected: 节点注册成功，约 1-3 分钟后状态变为 Normal
```

脚本执行后约 1-3 分钟，节点状态变为 `Normal`。如长时间未注册成功，检查 IDC 到 VPC 的专线/VPN 连通性，参见 [排障](#排障)。

## 验证

### 多维度验证

```bash
# 1. 确认注册节点支持已开启
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Enabled", NetworkType: "HostNetwork"
```

```json
{
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Enabled": false,
    "Status": "Enabled",
    "Master": "10.0.0.2",
    "Proxy": "10.0.0.19",
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "203.0.113.10:9000"
}
```

```bash
# 2. 确认节点池创建成功
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount >= 1，LifeState 为 "normal"
```

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.7.28"
            },
            "NodeType": "CPU"
        }
    ]
}
```

```bash
# 3. 查看注册节点（脚本未在 IDC 执行前预期为空）
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId>
# expected: 脚本未执行时 TotalCount 为 0；脚本执行后 TotalCount >= 1
```

```json
{
    "Nodes": [],
    "TotalCount": 0
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 注册功能状态 | `DescribeExternalNodeSupportConfig` | `Status=Enabled` |
| 网络类型 | 同上 | `NetworkType=HostNetwork` |
| Proxy 已分配 | 同上 | `Master` 和 `Proxy` 字段非空 |
| 节点池状态 | `DescribeExternalNodePools` | `LifeState=normal` |
| 运行时配置 | 同上 | `RuntimeType=containerd`，`RuntimeVersion` 与创建参数一致 |
| 节点数 | `DescribeExternalNode` | 脚本未执行时为 0（预期）；执行后 ≥ 1 |

> 脚本未在 IDC 主机执行时，`DescribeExternalNode` 返回空列表是预期行为。节点注册需在 IDC 主机上执行初始化脚本后才会完成。

## 清理

> **⚠️ 警告**：`DeleteExternalNodePool` 不会自动清理已注册的外部节点——需先 `DrainExternalNode` 再 `DeleteExternalNode`，最后才能安全删除节点池。
> `EnableExternalNodeSupport` 目前没有对应的 `Disable` API（一旦开启无法通过 CLI 关闭），请确认业务需要再执行。
> 生产环境执行前务必先 `DescribeExternalNodePools` 确认节点池 ID 和名称。
> 删除节点池后，先前通过 `DescribeExternalNodeScript` 获取的注册脚本 Link 和 Token 将失效，如需重新注册请重新调用 `DescribeExternalNodeScript`。

### 清理前状态检查

```bash
# 检查当前节点池
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# 确认待删除的节点池，记录 NodePoolId 和 Name

# 检查注册节点
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId>
# 确认节点列表，如有节点需先驱逐并删除节点再删节点池
# NodeName 从 DescribeExternalNode 输出的 $.Nodes[0].Name 获取
```

### 1. 驱逐节点（如有已注册节点）

```bash
# 如有已注册节点，先驱逐
tccli tke DrainExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --Name <NodeName>
# expected: exit 0，Pod 被调度到其他节点
```

### 2. 删除注册节点（如有已注册节点）

```bash
tccli tke DeleteExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --Names '["<NodeName>"]'
# expected: exit 0，返回 RequestId
```

### 3. 删除注册节点池

```bash
tccli tke DeleteExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]'
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 4. 验证已删除

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount 为 0 或目标节点池不在列表中
```

```json
{
    "TotalCount": 0,
    "NodePoolSet": []
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateExternalNodePool` 返回 `FailedOperation.ExternalNodeNotSupport` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 查看 `Status` | 忘记先调用 `EnableExternalNodeSupport` 就直接创建节点池，或开启过程仍在 `Initializing` | 先执行 `EnableExternalNodeSupport`，轮询等待 `Status=Enabled` 再创建节点池 |
| `EnableExternalNodeSupport` 返回 `FailedOperation.NetworkTypeMismatch` | 检查 `ClusterExternalConfig.NetworkType` 值 | 网络模式选错——专线版选了 `CiliumBGP` 或公网版选了 `HostNetwork` | 专线版用 `HostNetwork`，公网版用 `CiliumBGP` |
| `DescribeExternalNodeScript` 下载脚本失败（COS 签名过期） | 检查脚本下载时间与获取时间间隔 | 返回的 `Token` 是 COS 临时令牌，有时效性，过期后下载失败 | 重新调用 `DescribeExternalNodeScript` 获取新的 `Token` 和下载链接 |
| `EnableExternalNodeSupport` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 确认子网存在 | 子网不存在或不属于集群所在 VPC | 使用集群 VPC 下的子网 ID，通过 `DescribeClusters` 获取 `VpcId` 后查子网 |

### 节点注册后状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 脚本执行成功但节点状态为 `NotReady` | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId>` 查看 `Status`；在 IDC 主机 `ping -c 4 <ProxyIP>` 测试连通性 | 专线版 IDC 主机未通过专线/VPN 连通 TKE VPC | 确认 IDC 到 VPC 的专线或 VPN 已建立且路由可达，`ping -c 4 <ProxyIP>`（如 `10.0.0.19`） |
| `DescribeExternalNode` 返回空列表 | 同上，确认脚本已在 IDC 主机执行 | 脚本未执行或正在初始化中（1-3 分钟） | 等待 3-5 分钟后重试；确认 IDC 主机上 `add2tkectl` 已执行成功 |
| `Status` 长时间为 `Initializing` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 查看 `Progress` 各步骤状态 | 某个初始化步骤卡住（如网络组件安装失败） | 查看 `Progress` 中非 `success` 的步骤名称 → 保留 region、ClusterId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |

> **排障时请保留 RequestId 和 ClusterId**，以便提工单时使用。

## 下一步

- [编辑注册节点池](https://cloud.tencent.com/document/product/457/79765) — 修改节点池配置
- [移除注册节点](https://cloud.tencent.com/document/product/457/79767) — 删除注册节点或节点池
- [流量接入](https://cloud.tencent.com/document/product/457/79749) — 注册节点上的流量接入方式
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：选择专线模式并创建节点池，获取注册脚本。
