# 创建注册节点（公网版）

> 对照官方：[创建注册节点（公网版）](https://cloud.tencent.com/document/product/457/101532) · page_id `101532` · tccli ≥3.1.107.1 · API 2018-05-25

## 概述

通过公网版方式将 IDC/第三方云主机注册到 TKE 集群。公网版使用 `HostNetwork` 模式，IDC 主机通过公网 TLS 隧道接入集群，无需专线/VPN 基础设施，部署快速灵活。

公网版注册节点的核心区别在于通过 `externaledge` Addon 建立公网 TLS 隧道，使 IDC 主机无需专线即可接入集群。完整流程分五步：

1. `EnableExternalNodeSupport` — 开启注册节点支持（`NetworkType=HostNetwork`），等待完成
2. `InstallAddon externaledge` — 安装边缘代理 Addon（依赖 CLB 基础设施就绪）
3. `DescribeSupportedRuntime` — 查询兼容的运行时版本，防止版本不匹配
4. `CreateExternalNodePool` — 创建注册节点池（依赖步骤 1 完成）
5. `DescribeExternalNodeScript` — 获取初始化脚本并在目标 IDC 主机上执行

> **注意**：`externaledge` Addon 依赖集群具备公网 CLB（负载均衡）基础设施。如果集群未配置公网端点或 CLB 未就绪，`InstallAddon externaledge` 将失败（`Phase=InstallFailed`，Reason 为 `"PreInstallwait for clbIp and clbHostname empty"` 或 `"get customDomain failed"`）。这是基础设施依赖，非 CLI 命令参数错误。
>
> 本文档中所有 ID（集群 ID、节点池 ID、子网 ID 等）均为占位符，请替换为实际值。

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
#    tke:DescribeClusters
#    tke:EnableExternalNodeSupport
#    tke:DescribeExternalNodeSupportConfig
#    tke:InstallAddon
#    tke:DescribeAddon
#    tke:DeleteAddon
#    tke:DescribeSupportedRuntime
#    tke:CreateExternalNodePool
#    tke:DescribeExternalNodePools
#    tke:DescribeExternalNodeScript
#    tke:DescribeExternalNode
#    tke:DeleteExternalNodePool
#    tke:DeleteExternalNode
#    vpc:DescribeSubnets
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-cluster",
      "ClusterDescription": "示例托管集群",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且为托管集群
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterType 为 MANAGED_CLUSTER，ClusterStatus 为 Running

# 5. 查询可用子网
tccli vpc DescribeSubnets --region <Region>
# expected: 至少返回 1 个子网，记录 SubnetId

# 6. 查询兼容的运行时版本（防止版本不匹配错误）
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion <K8sVersion>
# expected: 返回 OptionalRuntimes，记录 containerd 的可用 RuntimeVersion 和 DefaultVersion

# 7. 检查 externaledge Addon 是否已安装（公网版必须）
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: 返回 Addon 信息，如未安装则 Status 为空或报错
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-cluster",
      "ClusterDescription": "示例托管集群",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running"
    }
  ]
}
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "SubnetSet": [
    {
      "SubnetId": "subnet-example",
      "SubnetName": "default",
      "CidrBlock": "172.26.0.0/20",
      "Zone": "ap-guangzhou-4",
      "AvailableIpAddressCount": 4000
    }
  ]
}
```

**DescribeSupportedRuntime 预期输出**：

```json
{
  "OptionalRuntimes": [
    {
      "RuntimeType": "containerd",
      "RuntimeVersions": ["1.6.9", "1.7.28"],
      "DefaultVersion": "1.6.9"
    }
  ]
}
```

**DescribeAddon externaledge 预期输出（未安装时）**：

```json
{
  "RequestId": "..." 
}
```

> **注意**：上述命令中使用的 `<K8sVersion>` 从 `DescribeClusters` 输出的 `$.Clusters[0].ClusterVersion` 字段获取（如 `1.32.2`）。在下文的创建节点池步骤中，`RuntimeVersion` 必须与此 K8s 版本兼容。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 开通注册节点支持 | `EnableExternalNodeSupport` | 否 |
| 查询外部节点支持状态 | `DescribeExternalNodeSupportConfig` | 是 |
| 安装 externaledge Addon | `InstallAddon` | 否 |
| 查询 Addon 状态 | `DescribeAddon` | 是 |
| 查询支持的运行时 | `DescribeSupportedRuntime` | 是 |
| 创建注册节点池 | `CreateExternalNodePool` | 否 |
| 查询注册节点池列表 | `DescribeExternalNodePools` | 是 |
| 获取注册脚本 | `DescribeExternalNodeScript` | 是 |
## 关键字段说明

以下说明四个 API 的核心参数。

### InstallAddon externaledge 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | 必须为 `externaledge` | 填错名称 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 默认最新版本 | — |

### EnableExternalNodeSupport 参数（公网版）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 集群不存在 → `InvalidParameter.ClusterId` |
| `NetworkType` | String | 是 | 公网版必须为 `HostNetwork` | 填 `Public`/`公网版` 等概念 → `InvalidParameter.NetworkType` |
| `SubnetId` | String | 是 | 子网 ID | 子网不存在 → `InvalidParameter.SubnetId` |
| `Enabled` | Boolean | 否 | **已废弃**，不再使用。用 `Status` 字段判断是否开通 | — |
| `ClusterCIDR` | String | 否 | 公网版（HostNetwork）**不需要**指定，留空 | — |

### CreateExternalNodePool 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 集群不存在 → `InvalidParameter.ClusterId` |
| `Name` | String | 是 | 节点池名称，1-60 字符 | 命名违规 → `InvalidParameter.NodePoolName` |
| `ContainerRuntime` | String | 是 | 必须为 `containerd` | 填 `docker` → `InvalidParameter.ContainerRuntime` |
| `RuntimeVersion` | String | 是 | 如 `1.6.9`。需与 K8s 版本兼容 | 版本不匹配 → `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError` |
| `NodeType` | String | 否 | 如 `CPU`。节点类型标签 | — |
| `DeletionProtection` | Boolean | 否 | 默认 `false` | 忘开删除保护 → 可能误删节点池 |

### 查询兼容的运行时版本

创建节点池前，必须确认运行时版本与集群 K8s 版本兼容：

```bash
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion <K8sVersion>
# expected: 返回 OptionalRuntimes 列表，含合法的 RuntimeType 和 RuntimeVersion
```

**预期输出**：

```json
{
    "OptionalRuntimes": [
        {
            "RuntimeType": "containerd",
            "RuntimeVersions": ["1.6.9", "1.7.28"],
            "DefaultVersion": "1.6.9"
        }
    ]
}
```

> **注意**：`RuntimeVersion` 必须从上述列表中选择。使用不兼容的版本会触发 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`。

## 操作步骤

### 步骤 1：开启注册节点支持

#### 选择依据

- **网络类型必须为 `HostNetwork`**：`EnableExternalNodeSupport` 的 `NetworkType` 参数仅支持 `HostNetwork` 和 `CiliumBGP` 两个枚举值。控制台概念"公网版"对应 API 的 `HostNetwork`，**不是 `Public`**。公网版不依赖专线/VPN 基础设施，IDC 主机通过公网连通即可。
- **CIDR**：`HostNetwork` 模式下无需指定 `ClusterCIDR`，Pod 使用宿主机网络。
- **`Enabled` 字段已废弃**：不用关注此字段的值。开通完成与否通过 `Status` 字段判断（`Initializing` → `Enabled`）。

#### 最小创建

`enable-external-host.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "ClusterExternalConfig": {
        "NetworkType": "HostNetwork",
        "SubnetId": "<SubnetId>"
    }
}
```

```bash
cat > enable-external-host.json <<'EOF'
{"ClusterId":"<ClusterId>","ClusterExternalConfig":{"NetworkType":"HostNetwork","SubnetId":"<SubnetId>"}}
EOF
tccli tke EnableExternalNodeSupport --region <Region> \
    --cli-input-json file://enable-external-host.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

开通是异步操作，轮询直到 `Status` 变为 `Enabled`（约 3-5 分钟）：

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Enabled"，所有 Progress 子步骤 status 为 "success"
```

**预期输出**：

```json
{
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Status": "Enabled",
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "...",
    "Progress": [
        {"Name": "EnsureNodeMasterCommunicate", "Status": "success"},
        {"Name": "CompleteEnable", "Status": "success"}
    ]
}
```

> **注意**：如果 `EnableExternalNodeSupport` 返回 `FailedOperation.OperationForbidden`，说明已有开通任务在执行中。执行 `DescribeExternalNodeSupportConfig` 轮询进度即可，任务完成后 `Status` 自动变为 `Enabled`。

### 步骤 2：安装 externaledge Addon

#### 选择依据

- **`externaledge` 是公网版必须组件**：它作为边缘代理，在集群与注册节点之间建立 TLS 隧道，使 IDC 主机通过公网安全接入集群。必须先完成步骤 1（`Status=Enabled`），externaledge 才能安装成功。
- **依赖项**：externaledge 需要集群具备公网 CLB（负载均衡）基础设施。如果集群未配置公网端点，externaledge 安装会失败（`Phase=InstallFailed`）。

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

等待 Addon 安装完成（约 1-2 分钟）：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: Phase 为 "Running" 或 "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "externaledge",
            "AddonVersion": "1.1.9",
            "Phase": "Running",
            "Reason": ""
        }
    ]
}
```

> **排障**：如果 `Phase=InstallFailed, Reason=get customDomain failed` 或 `Reason=PreInstallwait for clbIp and clbHostname empty`，说明集群缺少公网 CLB。参见 [排障](#排障) 节。

### 步骤 3：查询兼容的运行时版本

#### 选择依据

- **必需步骤**：注册节点池的 `RuntimeVersion` 必须与集群 K8s 版本兼容。使用不兼容的版本（如 containerd 1.6.28 搭配 K8s 1.32.2）会返回 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`。
- **查询命令**：`DescribeSupportedRuntime` 传入集群的 K8s 版本即可获取兼容列表。
- **版本选择**：推荐使用 `DefaultVersion`。托管集群仅支持 `containerd`。

```bash
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion <K8sVersion>
# expected: 返回 containerd 的兼容版本列表，记录 DefaultVersion
```

**预期输出**：

```json
{
    "OptionalRuntimes": [
        {
            "RuntimeType": "containerd",
            "RuntimeVersions": ["1.6.9", "1.7.28"],
            "DefaultVersion": "1.6.9"
        },
        {
            "RuntimeType": "docker",
            "RuntimeVersions": ["20.10"],
            "DefaultVersion": "20.10"
        }
    ]
}
```

> **记录**：从 `RuntimeVersions` 中选择一个版本（推荐 `DefaultVersion`），在下一步创建节点池时使用。

### 步骤 4：创建注册节点池

#### 选择依据

- **运行时**：托管集群强制使用 `containerd`。必须先通过 `DescribeSupportedRuntime` 查询兼容版本（见 [关键字段说明](#关键字段说明)）。
- **`RuntimeVersion`**：必须从 `DescribeSupportedRuntime` 返回的 `RuntimeVersions` 列表中选择，否则触发 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError`。

```bash
cat > node-pool-input.json <<'EOF'
{"ClusterId":"<ClusterId>","Name":"<NodePoolName>","ContainerRuntime":"containerd","RuntimeVersion":"1.6.9","NodeType":"CPU"}
EOF
tccli tke CreateExternalNodePool --region <Region> \
    --cli-input-json file://node-pool-input.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "..."
}
```

验证节点池创建成功：

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount >= 1，目标节点池 LifeState 为 "normal"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "RuntimeConfig": {"RuntimeType": "containerd", "RuntimeVersion": "1.6.9"},
            "NodeType": "CPU",
            "DeletionProtection": false
        }
    ]
}
```

> **注意**：如果返回 `InvalidParameter.Param: you need enable external node support first`，说明步骤 1 尚未完成。执行 `DescribeExternalNodeSupportConfig` 确认 `Status=Enabled` 后重试。

### 步骤 5：生成注册脚本

#### 选择依据

- **`Interface`**：公网版选择 `extranet`（外网），IDC 主机通过公网获取并执行注册脚本。
- **`Internal`**：公网版设为 `false`（默认值），IDC 主机无需通过内网代理获取脚本。

```bash
tccli tke DescribeExternalNodeScript --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --Interface extranet \
    --Internal false
# expected: exit 0，返回 Link, Token, Command
```

**预期输出**：

```json
{
    "Link": "https://externalnode-...cos.ap-guangzhou.myqcloud.com/...",
    "Token": "J6sWTjumGKiOTAWQ",
    "Command": "wget --header=\"x-cos-token:...\" 'http://externalnode-...' -O add2tkectl-... && chmod +x add2tkectl-...",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<K8sVersion>` | K8s 版本 | 如 `1.32.2` | `tccli tke DescribeClusters` 输出的 `$.Clusters[0].ClusterVersion` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | 步骤 4 返回的 `NodePoolId` |
| `<NodePoolName>` | 节点池名称 | 1-60 字符 | 自定义 |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets --region <Region>` |
| `<NodeName>` | 注册节点名称 | 格式 `node-xxxxxxxx` | `tccli tke DescribeExternalNode` 输出的 `$.Nodes[].Name` |

> **⚠️ 安全警告**：`DescribeExternalNodeScript` 返回的 `Token` 和 `Command` 是节点注册凭证，应妥善保管。泄露意味着攻击者可利用该凭证注册恶意节点到你的集群，获取集群内的网络访问权限。建议：
> - 不要在日志、版本控制系统或共享终端中明文保存 Token
> - 注册完成后 Token 自动失效，无需手动吊销
> - 如果怀疑 Token 已泄露，删除对应节点并重新生成脚本

脚本生成后，在目标 IDC 主机上执行返回的 `Command` 即可完成注册。执行后约 1-3 分钟节点状态变为 `Normal`。

> **注意**：`EnableExternalNodeSupport` 开启后无对应的关闭 API（无 `DisableExternalNodeSupport`），该状态在集群生命周期内持久保留。如果后续不再需要注册节点功能，可跳过此步骤——留下 `Enabled` 状态不会产生额外费用，也不会影响集群正常使用和销毁。

## 验证

### 多维度验证

```bash
# 1. 确认 externaledge Addon 状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: 返回 Addon 信息。Phase 可能为 Active（安装成功）或 InstallFailed（CLB 未就绪）
```

**预期输出（安装成功时）**：

```json
{
    "Addons": [
        {
            "AddonName": "externaledge",
            "AddonVersion": "1.1.9",
            "Phase": "Active"
        }
    ]
}
```

> **⚠️ 如实反映**：如果集群未配置公网 CLB，externaledge Addon 可能安装失败——`DescribeAddon` 返回 `Phase=InstallFailed`，Reason 为 `"PreInstallwait for clbIp and clbHostname empty"` 或 `"get customDomain failed"`。这表示 CLB 基础设施依赖未满足，参见 [排障](#排障) 中的修复步骤。

```bash
# 2. 确认注册节点支持状态
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Enabled"，NetworkType: "HostNetwork"
```

```json
{
    "ClusterCIDR": "",
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "EnabledPublicConnect": true,
    "Status": "Enabled",
    "PublicConnectUrl": "..."
}
```

> **注意**：`Status` 字段为 `"Enabled"` 表示开通成功（**非 `"Running"`**）。`Enabled` 布尔字段已废弃，不要用作判断依据。

```bash
# 3. 确认节点池创建成功
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount >= 1，所创建节点池 LifeState 为 "normal"
```

```json
{
  "TotalCount": 1,
  "NodePoolSet": [
    {
      "NodePoolId": "np-example",
      "Name": "<NodePoolName>",
      "LifeState": "normal",
      "RuntimeConfig": {"RuntimeType": "containerd", "RuntimeVersion": "1.6.9"},
      "NodeType": "CPU",
      "DeletionProtection": false
    }
  ]
}
```

```bash
# 4. 确认节点已注册（脚本执行后约 1-3 分钟）
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: TotalCount >= 1，Status: "Normal"
```

```json
{
  "Nodes": [
    {
      "Name": "node-example",
      "NodePoolId": "np-example",
      "Status": "Normal",
      "IP": "10.0.1.100",
      "Location": "IDC-北京机房"
    }
  ],
  "TotalCount": 1
}
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| Addon 状态 | `DescribeAddon --AddonName externaledge` | `Phase=Active`（如 CLB 未就绪可能为 `InstallFailed`，属基础设施问题） |
| 注册功能状态 | `DescribeExternalNodeSupportConfig` | `Status=Enabled`，所有 Progress 子步骤 `success` |
| 网络类型 | 同上 | `NetworkType=HostNetwork` |
| 公网连通性 | 同上 | `EnabledPublicConnect=true`，`PublicConnectUrl` 非空 |
| 节点池状态 | `DescribeExternalNodePools` | `LifeState=normal` |
| 节点状态 | `DescribeExternalNode` | `Status=Normal` |
| 节点数 | `DescribeExternalNode` 检查 `TotalCount` | ≥ 1 |

### 验证节点连通性

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。如果使用内网端点（默认的 `DescribeClusterKubeconfig` 返回内网地址），需通过 IOA/VPN/专线 连接。如果 kubeconfig 已写入 `~/.kube/config`，可用 `kubectl cluster-info` 快速验证连通性。

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config
# expected: Kubeconfig 已写入 ~/.kube/config

# 确认注册节点 Ready
kubectl get nodes -l node-type=external
# expected: 至少 1 个节点 STATUS 为 Ready
```

## 清理

> **⚠️ 警告**：
> - `DeleteExternalNodePool` 配合 `Force=true` 会**强制删除节点池及其中所有注册节点**，节点上的 Pod 将被驱逐且无法恢复。
> - `DeleteAddon externaledge` 会移除边缘代理组件，**所有已注册的外部节点将失去与集群的连接**。
> - 清理前务必先 `DescribeExternalNodePools` 和 `DescribeExternalNode` 确认目标资源。
> - 生产环境执行前务必确认节点池名称和 ID。

### 清理前状态检查

```bash
# 检查当前节点池
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>

# 检查注册节点
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
```

```json
{
  "TotalCount": 0,
  "NodePoolSet": []
}
```

### 1. 删除注册节点

```bash
# 先驱逐节点上的 Pod（推荐）
tccli tke DrainExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --Name <NodeName>
# expected: exit 0，Pod 被调度到其他节点

cat > delete-node-input.json <<'EOF'
{"ClusterId":"<ClusterId>","Names":["<NodeName>"]}
EOF
tccli tke DeleteExternalNode --region <Region> \
    --cli-input-json file://delete-node-input.json
# expected: exit 0，返回 RequestId
```

`<NodeName>` 从 `DescribeExternalNode` 输出的 `$.Nodes[].Name` 字段获取。执行前务必通过 `DescribeExternalNode` 确认节点名称，避免误删其他节点。

### 2. 删除节点池

```bash
cat > delete-pool-input.json <<'EOF'
{"ClusterId":"<ClusterId>","NodePoolIds":["<NodePoolId>"],"Force":true}
EOF
tccli tke DeleteExternalNodePool --region <Region> \
    --cli-input-json file://delete-pool-input.json
# expected: exit 0，返回 RequestId
```

### 3. 卸载 externaledge Addon

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: exit 0，返回 RequestId
```

### 4. 验证已清理

```bash
# 确认节点池已删除
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount 为 0 或目标节点池不在列表中

# 确认节点已删除
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: TotalCount 为 0

# 确认 externaledge Addon 已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName externaledge
# expected: 返回空列表或 Addon 不存在
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
| `EnableExternalNodeSupport` 返回 `InvalidParameter.Param: invalid NetworkType` | 检查 `ClusterExternalConfig.NetworkType` | 填了控制台概念名 `Public` 或 `公网版`，API 仅接受 `HostNetwork`/`CiliumBGP` | 将 `NetworkType` 改为 `"HostNetwork"`。公网版注册节点通过 externaledge Addon 实现公网通信，非通过 NetworkType 参数。 |
| `EnableExternalNodeSupport` 返回 `FailedOperation.OperationForbidden: cannot enable external node support when in task: enableExternalNode` 或 `UnsupportedOperation.AlreadyEnabled` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 检查状态 | 已有开通任务在执行中（Status=Initializing）或开通已完成（Status=Enabled），不可重复调用 | 等待当前任务完成：轮询 `DescribeExternalNodeSupportConfig` 直到 `Status=Enabled`，所有 Progress 子步骤 `success`。如 Status 已为 Enabled，则无需再次执行开通命令。这是正常行为，非错误。 |
| `DescribeAddon externaledge` 返回 `ResourceNotFound` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName externaledge` | externaledge 未安装 | 执行 `InstallAddon --AddonName externaledge` 安装。前提：`EnableExternalNodeSupport` 已完成（`Status=Enabled`）。 |
| `CreateExternalNodePool` 返回 `InvalidParameter.Param: you need enable external node support first` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` | 尚未完成 `EnableExternalNodeSupport` | 先执行 `EnableExternalNodeSupport`，等待 `Status=Enabled`，再创建节点池。 |
| `CreateExternalNodePool` 返回 `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError: k8s version and runtime (type/version) not match` | `tccli tke DescribeSupportedRuntime --region <Region> --K8sVersion <CLUSTER_VERSION>` | `RuntimeVersion` 不在 K8s 版本兼容列表中 | 从 `DescribeSupportedRuntime` 返回的 `RuntimeVersions` 列表中选择兼容版本。例如 K8s 1.32.2 支持 containerd `1.6.9` 和 `1.7.28`，不支持 `1.6.28`。 |
| `InstallAddon externaledge` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 值 | Addon 名称拼写错误 | 用 `"externaledge"`（全部小写，无空格）。 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| externaledge Addon 安装后 `Phase=InstallFailed, Reason=get customDomain failed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName externaledge` | 集群公网自定义域名未配置（`PublicCustomDomain` 为空） | `tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName externaledge` 删除失败的 Addon → 确认集群公网端点已配置（`DescribeClusterEndpoints`）→ 重新 `InstallAddon externaledge`。 |
| externaledge Addon 安装后 `Phase=InstallFailed, Reason=PreInstallwait for clbIp and clbHostname empty` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName externaledge` + `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` | 集群缺少公网 CLB（负载均衡）。externaledge 组件需要 CLB 提供公网 TLS 隧道端点。 | 确认集群有公网端点：`tccli tke CreateClusterEndpoint --region <Region> --ClusterId <ClusterId> --IsExtranet true`。CLB 创建需 1-2 分钟，待端点 `Status=Created` 后，重新安装 externaledge。**此为基础设施依赖，非 CLI 命令参数错误。** |
| 脚本执行失败 | 检查 IDC 主机网络：`curl -I https://cloud.tencent.com` 确认公网可达 | IDC 主机无法访问公网或 DNS 解析失败 | 配置 IDC 主机的公网访问和 DNS；如有限制，添加腾讯云 API 域名到白名单。 |
| 脚本执行成功但 `DescribeExternalNode` 无节点 | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` | 节点注册初始化需要 1-3 分钟 | 等待 3-5 分钟后重试。确认 `DescribeExternalNodeSupportConfig` 的 `Status=Enabled` 且 externaledge `Phase=Running`。 |
| 节点状态为 `Abnormal` | 查看节点详情和 `Reason` 字段 | externaledge 组件或网络异常 | 检查 `DescribeExternalNodeSupportConfig` 确认 `Status=Enabled` 且所有 Progress 子步骤 `success`；检查 externaledge Addon 状态（`Phase=Running`）；保留 region、ClusterId、RequestId → 登录控制台查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder)。 |

## 下一步

- [编辑注册节点池](https://cloud.tencent.com/document/product/457/79765) — 修改节点池配置
- [移除注册节点](https://cloud.tencent.com/document/product/457/79767) — 删除注册节点或节点池
- [流量接入](https://cloud.tencent.com/document/product/457/79749) — 注册节点上的流量接入方式
- [注册节点概述](https://cloud.tencent.com/document/product/457/57916) — 专线版与公网版对比
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：选择公网模式并创建节点池，获取注册脚本。
