# 新增游离普通节点

> 对照官方：[新增游离普通节点](https://cloud.tencent.com/document/product/457/32203) · page_id `32203` · tccli ≥3.1.107 · API 2018-05-25

## 概述

将 CVM 实例以节点形式加入 TKE 集群，有两种方式：

| 方式 | API | 说明 |
|------|-----|------|
| **添加已有实例** | `AddExistedInstances` | 将已有 CVM 实例添加到集群，可作为游离节点或加入节点池 |
| **新建实例并加入** | `CreateClusterInstances` | 新建 CVM 实例并自动加入集群 |

游离节点与节点池节点的区别：

| 维度 | 游离节点 | 节点池节点 |
|------|---------|-----------|
| 归属 | 不属于任何节点池 | 属于指定节点池 |
| 弹性伸缩 | 不支持 | 支持（需在节点池中启用） |
| 批量管理 | 需逐个操作 | 通过节点池统一管理 |
| 创建方式 | `AddExistedInstances` / `CreateClusterInstances` | `CreateClusterNodePool` |

> **注意：** `AddExistedInstances` 要求 CVM 实例满足以下条件：与集群同 VPC、未被其他集群占用、实例处于运行中（`RUNNING`）状态。`CreateClusterInstances` 自动创建满足条件的 CVM 实例，无需预先准备 CVM。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances, tke:CreateClusterInstances
#    tke:AddExistedInstances, tke:DeleteClusterInstances, tke:DescribeClusterEndpoints
#    cvm:DescribeInstances, cvm:DescribeZoneInstanceConfigInfos
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 CVM 权限
tccli cvm DescribeInstances --region <Region>
# expected: exit 0，返回 CVM 列表（可为空）
```

> **关于写权限验证**：`tke:CreateClusterInstances`、`tke:AddExistedInstances`、`tke:DeleteClusterInstances` 等写 Action 权限在执行对应命令时自然验证——若权限不足，写命令会返回 `UnauthorizedOperation` 错误。无需在执行前单独验证。

### 资源检查

```bash
# 4. 确认集群状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"，ContainerRuntime: "containerd"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "<ClusterName>",
            "ClusterVersion": "1.32.2",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterNodeNum": 0,
            "ContainerRuntime": "containerd",
            "RuntimeVersion": "1.6.9",
            "DeletionProtection": false,
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "MaxNodePodNum": 64,
                "VpcId": "<VpcId>",
                "Cni": true,
                "SubnetId": "<SubnetId>"
            }
        }
    ]
}
```

```bash
# 5. 检查集群现有节点（创建前）
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region>
# expected: 确认当前节点数量，TotalCount 应为 0（空集群）或非 0（已有节点）
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "InstanceSet": []
}
```

```bash
# 6. 查看集群端点（确认 kubectl 连通性）
tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region <Region>
# expected: ClusterIntranetEndpoint 非空（内网端点），ClusterExternalEndpoint 可能为空
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "<InternalIp>",
    "ClusterDomain": "<ClusterId>.ccs.tencent-cloud.com",
    "ClusterIntranetSubnetId": "<SubnetId>"
}
```

```bash
# 7. 查询可用机型（CreateClusterInstances 方式）
tccli cvm DescribeZoneInstanceConfigInfos \
    --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["S5"]}]' \
    --region <Region>
# expected: 返回可用机型列表，S5.MEDIUM2 等可供选择
```

**预期输出**：

```json
{
    "InstanceTypeQuotaSet": [
        {
            "Zone": "<Zone>",
            "InstanceType": "S5.MEDIUM2",
            "InstanceFamily": "S5",
            "Cpu": 2,
            "Memory": 2,
            "Status": "SELL",
            "TypeName": "标准型S5",
            "InstanceBandwidth": 1.5,
            "InstancePps": 30,
            "CpuType": "Intel Xeon Cascade Lake"
        }
    ],
    "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 添加已有节点 | `tccli tke AddExistedInstances` | 否（重复添加返回 `FailedOperation.Param`） |
| 新建节点加入集群 | `tccli tke CreateClusterInstances` | 否 |
| 查看集群节点 | `tccli tke DescribeClusterInstances` | 是 |
| 查看集群节点池 | `tccli tke DescribeClusterNodePools` | 是 |
| 查看 CVM 实例 | `tccli cvm DescribeInstances` | 是 |
| 移除集群节点 | `tccli tke DeleteClusterInstances` | 否 |

### 关键字段说明 — CreateClusterInstances

以下说明 `CreateClusterInstances` 的主要参数。完整参数定义见 `tccli tke CreateClusterInstances --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `<ClusterId>` | 集群不存在 → `ResourceNotFound` |
| `RunInstancePara` | String | 是 | CVM RunInstances 参数的 JSON 字符串，需二次序列化。内含 `InstanceType`、`ImageId`、`VirtualPrivateCloud`、`SystemDisk`、`InstanceChargeType`、`Placement` 等子字段 | JSON 格式错误或嵌套转义错误 → `InvalidParameter` |
| `SkipValidateOptions` | Array | 否 | 跳过校验项列表。VPC-CNI 集群（`Cni: true`）子网 IP 不足时需传 `["VpcCniCIDRCheck"]`。生产环境优先 `AddVpcCniSubnets` 扩展 IP 池 | 不加且子网 IP 不足 → `FailedOperation.NetworkScaleError` |
| `InstanceAdvancedSettings` | Object | 否 | 节点高级设置，含 `MountTarget`（数据盘挂载点，如 `/var/lib/containerd`）、`DockerGraphPath`、`ExtraArgs` 等 | 挂载点无效 → 节点初始化失败 |

### 关键字段说明 — AddExistedInstances

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `<ClusterId>` | 集群不存在 → `ResourceNotFound` |
| `InstanceIds` | Array | 是 | CVM 实例 ID 列表，格式 `["<InstanceId>"]`。实例必须与集群同 VPC、未被其他集群占用、状态 `RUNNING` | 实例已加入同集群 → `FailedOperation.Param: instance already in cluster` |
| `NodePoolId` | String | 否 | 目标节点池 ID。不传则作为游离节点加入 | 节点池不存在 → `ResourceNotFound` |
| `InstanceAdvancedSettings.MountTarget` | String | 否 | 数据盘挂载点，如 `/var/lib/containerd` | 挂载点无效 → 节点初始化失败 |
| `LoginSettings.Password` | String | 否 | 节点 SSH 登录密码 | — |
| `EnhancedService.SecurityService.Enabled` | Boolean | 否 | 是否开启云安全服务，默认 `true` | — |

### 关键字段说明 — DeleteClusterInstances

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `<ClusterId>` | 集群不存在 → `ResourceNotFound` |
| `InstanceIds` | Array | 是 | 要移除的实例 ID 列表 | 实例不在集群中 → `NotFoundInstanceIds` |
| `InstanceDeleteMode` | String | 是 | `terminate`（销毁 CVM）或 `unused`（仅从集群移除，保留 CVM） | 选 `unused` 后忘记手动退还 CVM → 持续计费 |
| `ForceDelete` | Boolean | 否 | 是否强制删除。节点初始化中/异常状态时需要设为 `true`。会跳过安全确认 | 生产环境误用 → 可能误删运行中节点 |

## 操作步骤

### 方式一：CreateClusterInstances — 新建 CVM 并加入集群（推荐）

#### 选择依据

*为什么选这些参数值：*

- **计费模式**：选择 `POSTPAID_BY_HOUR`（按量计费），而非 `PREPAID`（包年包月）。按量计费用完即删，避免费用锁定，测试/临时场景最优选。
- **机型**：选择 `S5.MEDIUM2`（标准型 S5，2 核 4G），测试场景最小可用规格。查询可用机型：`tccli cvm DescribeZoneInstanceConfigInfos --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["S5"]}]' --region <Region>`
- **VPC-CNI 网络校验**：集群 `Cni: true`（VPC-CNI 模式）时，`CreateClusterInstances` 默认校验子网 IP 是否充足。若子网可用 IP 不足，需添加 `SkipValidateOptions: ["VpcCniCIDRCheck"]` 跳过验证。生产环境推荐先用 `AddVpcCniSubnets` 扩展子网再创建节点：
  `tccli tke AddVpcCniSubnets --ClusterId <ClusterId> --SubnetIds '["<SubnetId>"]' --VpcId <VpcId> --region <Region>`
- **数据盘**：`CLOUD_PREMIUM`（高性能云硬盘）50GB，节点最小可用配置。
- **安全加固**：关闭 `SecurityService`（云安全），开启 `MonitorService`（云监控）和 `AutomationService`（自动化助手），便于后续排障。

#### 最小创建（必填字段，含 SkipValidateOptions）

`create-cluster-instances.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "RunInstancePara": "{\"Placement\":{\"Zone\":\"<Zone>\"},\"InstanceType\":\"S5.MEDIUM2\",\"ImageId\":\"<ImageId>\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"VirtualPrivateCloud\":{\"VpcId\":\"<VpcId>\",\"SubnetId\":\"<SubnetId>\"},\"InstanceCount\":1,\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"LoginSettings\":{\"Password\":\"<Password>\"},\"InternetAccessible\":{\"InternetMaxBandwidthOut\":0,\"PublicIpAssigned\":false}}",
    "SkipValidateOptions": [
        "VpcCniCIDRCheck"
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<Zone>` | 可用区 | 如 `ap-guangzhou-3` | `tccli cvm DescribeZones` 或与集群子网同可用区 |
| `<ImageId>` | 镜像 ID | 需与集群操作系统兼容 | `tccli cvm DescribeImages` 或 `DescribeClusters` 返回的 `ImageId` |
| `<VpcId>` | VPC ID | 与集群同 VPC | `tccli tke DescribeClusters` → `ClusterNetworkSettings.VpcId` |
| `<SubnetId>` | 子网 ID | 与集群同子网 | `tccli tke DescribeClusters` → `ClusterNetworkSettings.SubnetId` |
| `<Password>` | SSH 登录密码 | 8-30 字符，含大小写字母、数字和特殊字符 | 自定义 |

> **关于 SkipValidateOptions**：VPC-CNI 模式下，集群的子网 IP 配额可能不足以支撑新增节点所需的 Pod IP。首次执行不加此参数会返回 `FailedOperation.NetworkScaleError: VPC IP for pods in zone <Zone> is not enough`。添加 `SkipValidateOptions: ["VpcCniCIDRCheck"]` 可跳过此校验让命令成功执行。但须注意：跳过校验后若子网 IP 真的不足，节点创建后 Pod 可能无法分配 IP。生产环境应先用 `AddVpcCniSubnets` 扩展 IP 池。

> **计费提醒**：`InstanceChargeType: "POSTPAID_BY_HOUR"`（按量计费）模式从 CVM 实例创建成功起即按小时计费。以 `S5.MEDIUM2`（2 核 4G）为例，约 0.34 元/小时。测试完成后务必执行 ## 清理 段操作销毁节点，避免持续产生费用。

执行：

```bash
tccli tke CreateClusterInstances --region <Region> \
    --cli-input-json file://create-cluster-instances.json
# expected: exit 0，返回 InstanceIdSet 含新实例 ID
```

**预期输出**：

```json
{
    "InstanceIdSet": [
        "<InstanceId>"
    ],
    "RequestId": "<RequestId>"
}
```

#### 增强配置（加安全加固、监控、自动化助手）

`create-cluster-instances-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "RunInstancePara": "{\"Placement\":{\"Zone\":\"<Zone>\"},\"InstanceType\":\"S5.MEDIUM2\",\"ImageId\":\"<ImageId>\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"VirtualPrivateCloud\":{\"VpcId\":\"<VpcId>\",\"SubnetId\":\"<SubnetId>\"},\"InstanceCount\":1,\"InstanceName\":\"<InstanceName>\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"LoginSettings\":{\"Password\":\"<Password>\"},\"EnhancedService\":{\"SecurityService\":{\"Enabled\":false},\"MonitorService\":{\"Enabled\":true},\"AutomationService\":{\"Enabled\":true}},\"InternetAccessible\":{\"InternetMaxBandwidthOut\":0,\"PublicIpAssigned\":false}}",
    "SkipValidateOptions": [
        "VpcCniCIDRCheck"
    ]
}
```

> `InstanceName` 建议包含有意义的标识（如 `<env>-<user>-<role>`），便于在 CVM 控制台识别。

### 方式二：AddExistedInstances — 添加已有 CVM 实例

> **注意**：本方式要求已有可用的 CVM 实例。若被 CAM 策略拒绝 `cvm:RunInstances` 权限，将无法独立创建 CVM 进行此操作。此时应优先使用方式一 `CreateClusterInstances`（该 API 走 TKE 路径，不受 `cvm:RunInstances` 限制）。

#### 选择依据

- **游离节点 vs 节点池节点**：不传 `NodePoolId` 则作为游离节点加入。游离节点适合一次性测试节点、临时扩容，不受节点池约束。长期管理推荐加入节点池。
- **实例要求**：CVM 必须与集群同 VPC、同地域、未被其他集群占用、处于 `RUNNING` 状态。可先用 `DescribeInstances` 确认。

#### 加入游离节点（不入节点池）

`add-existed-instances.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "InstanceIds": [
        "<InstanceId>"
    ],
    "InstanceAdvancedSettings": {
        "MountTarget": "/var/lib/containerd"
    }
}
```

执行：

```bash
tccli tke AddExistedInstances --region <Region> \
    --cli-input-json file://add-existed-instances.json
# expected: exit 0，返回 SuccInstanceIds / FailedInstanceIds
```

**预期输出**：

```json
{
    "SuccInstanceIds": [
        "<InstanceId>"
    ],
    "FailedInstanceIds": [],
    "TimeoutInstanceIds": [],
    "RequestId": "<RequestId>"
}
```

#### 加入已有节点到节点池

指定 `NodePoolId` 将节点加入已有节点池：

`add-existed-instances-nodepool.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "InstanceIds": [
        "<InstanceId>"
    ],
    "NodePool": {
        "AddToNodePool": true,
        "NodePoolId": "<NodePoolId>"
    },
    "InstanceAdvancedSettings": {
        "MountTarget": "/var/lib/containerd"
    }
}
```

### 查看集群节点（创建后确认）

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region>
# expected: TotalCount ≥ 1，InstanceState: "running"，NodePoolId: ""（游离节点）
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "<InstanceId>",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "LanIP": "<PrivateIp>",
            "NodePoolId": "",
            "CreatedTime": "2026-06-23T08:13:02Z"
        }
    ]
}
```

> `NodePoolId` 为空字符串即表示为**游离节点**。节点创建后状态从 `initializing` → `running` 约需 5 分钟。在此期间 `NodePoolId` 为空表示游离节点。

### 关键字段说明（`DescribeClusterInstances` 返回字段）

| 字段 | 类型 | 说明 |
|------|------|------|
| `InstanceId` | String | CVM 实例 ID |
| `InstanceRole` | String | 节点角色：`WORKER`（工作节点）/ `MASTER_ETCD`（Master 节点） |
| `InstanceState` | String | 节点状态：`running` / `failed` / `initializing` |
| `FailedReason` | String | 节点加入失败原因（成功时为空） |
| `LanIP` | String | 节点内网 IP |
| `NodePoolId` | String | 所属节点池 ID（空字符串表示游离节点） |
| `CreatedTime` | String | 节点创建/加入时间 |

## 验证

### 控制面（tccli）

```bash
# 验证节点已加入集群
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region>
# expected: TotalCount 包含新加入的节点，InstanceState: "running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "<InstanceId>",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "LanIP": "<PrivateIp>",
            "NodePoolId": "",
            "CreatedTime": "2026-06-23T08:13:42Z"
        }
    ],
    "RequestId": "<RequestId>"
}
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点存在 | `TotalCount` 包含新加入的节点 | 新增节点可见 |
| 节点状态 | `InstanceState` | `running` |
| 无失败原因 | `FailedReason` | 空字符串 |
| 游离节点标识 | `NodePoolId` | 空字符串（如未指定 NodePoolId） |
| 内网 IP | `LanIP` | 与子网 CIDR 范围一致 |

### 数据面（kubectl）

> **重要**：kubectl 命令的可达性取决于集群端点配置。若集群无公网端点（`ClusterExternalEndpoint` 为空），内网端点 `<InternalIp>` 从本地网络可能不可达，需要 IOA/VPN/专线 或同 VPC 内的 CVM 才能连接。

若组织级 CAM 策略（如 `strategyId:240463971`）以 `tke:clusterExtranetEndpoint=true` 条件拒绝公网端点创建，则无法从本地执行 kubectl 命令。此时可通过以下方式验证节点：

```bash
# 获取 kubeconfig（优先尝试公网端点）
tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId> --region <Region> --IsExtranet true > kubeconfig.yaml
# expected: 返回 kubeconfig 内容，写入 kubeconfig.yaml
# 若公网端点不可用（ClusterExternalEndpoint 为空），改用内网端点：
# tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId> --region <Region> --IsExtranet false > kubeconfig.yaml
```

**预期输出**（写入 `kubeconfig.yaml`）：

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <base64-encoded-cert>
    server: https://<ClusterId>.ccs.tencent-cloud.com
  name: <ClusterId>
contexts:
- context:
    cluster: <ClusterId>
    user: <ClusterId>
  name: <ClusterId>
current-context: <ClusterId>
users:
- name: <ClusterId>
  user:
    client-certificate-data: <base64-encoded-cert>
    client-key-data: <base64-encoded-key>
```

```bash
# 使用获取的 kubeconfig 验证节点（仅在端点可达时有效）
# 若不可达，此命令会返回 "Unable to connect to the server"
KUBECONFIG=kubeconfig.yaml kubectl get nodes
# expected: 返回 Ready 状态的节点列表，或在端点不可达时返回连接错误
```

**预期输出**（端点可达时）：

```
NAME         STATUS   ROLES    AGE   VERSION
<InstanceId> Ready    <none>   2m    v1.32.2
```

端点不可达时的替代验证方式：通过 `DescribeClusterInstances` 确认 `InstanceState: "running"` 且 `FailedReason` 为空，即可确认节点已成功加入集群。

## 清理

> **⚠️ 副作用警告**：`InstanceDeleteMode: "terminate"` 会同时销毁 CVM 实例及关联磁盘，不可恢复。`InstanceDeleteMode: "unused"` 仅从集群移除节点，CVM 继续计费，需手动退还。移除前确认节点上的 Pod 已安全迁移（若有关键工作负载）。`ForceDelete: true` 会跳过安全确认，仅在节点初始化中/异常状态时使用。

### 1. 清理前状态检查

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: 确认目标节点 InstanceId、InstanceState、NodePoolId
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "<InstanceId>",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "LanIP": "<PrivateIp>",
            "NodePoolId": "",
            "CreatedTime": "2026-06-23T08:13:42Z"
        }
    ],
    "RequestId": "<RequestId>"
}
```

### 2. 移除并销毁游离节点

```bash
tccli tke DeleteClusterInstances --region <Region> \
    --cli-input-json file://delete-cluster-instances.json
```

`delete-cluster-instances.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "InstanceIds": [
        "<InstanceId>"
    ],
    "InstanceDeleteMode": "terminate",
    "ForceDelete": true
}
```

> **`ForceDelete` 使用场景**：节点初始化中（`initializing`）或异常状态时，正常删除可能失败（返回 `FailedInstanceIds` 含目标实例）。`ForceDelete: true` 可强制移除。生产环境正常运行的节点无需此参数。

**预期输出**：

```json
{
    "SuccInstanceIds": null,
    "FailedInstanceIds": null,
    "NotFoundInstanceIds": null
}
```

### 3. 验证已移除

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: TotalCount 为 0，InstanceSet 为空数组
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "InstanceSet": []
}
```

| `InstanceDeleteMode` | 含义 | 适用场景 |
|---------------------|------|---------|
| `terminate` | 移除节点并销毁 CVM 实例（不可恢复） | 测试环境，用完即删 |
| `unused` | 仅从集群移除节点，保留 CVM 实例 | 需保留 CVM 数据；之后需手动 `tccli cvm TerminateInstances` 退还 |

> **注意**：`unused` 模式下 CVM 继续按量计费，务必在不需要时手动退还。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterInstances` 返回 `FailedOperation.NetworkScaleError`：`VPC IP for pods in zone <Zone> is not enough` | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 确认 `Cni` 是否为 `true`。`tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]'` 查看 `AvailableIpAddressCount` | VPC-CNI 模式下子网可用 IP 不足以分配给新增节点所需的 Pod IP | **短期**：添加 `SkipValidateOptions: ["VpcCniCIDRCheck"]` 跳过校验。**长期**：`tccli tke AddVpcCniSubnets --ClusterId <ClusterId> --SubnetIds '["<SubnetId>"]' --VpcId <VpcId> --region <Region>` 扩展子网 IP 池 |
| `CreateClusterEndpoint (IsExtranet=true)` 返回 `InvalidParameter.Param`：`CAM deny strategyId:240463971, condition tke:clusterExtranetEndpoint=true` | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region <Region>` 确认 `ClusterExternalEndpoint` 为空 | 组织级 CAM 硬策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件拒绝所有公网端点创建。此为环境限制，非命令错误 | 使用内网端点。kubectl 需在同 VPC CVM 上执行，或通过 IOA/VPN/专线 连接内网端点 `<InternalIp>`。此限制不影响节点创建和加入集群 |
| `AddExistedInstances` 返回 `FailedOperation.Param`：`instance [<InstanceId>] be used in cluster <ClusterId>` | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region>` 检查该实例是否已在节点列表中 | CVM 实例已经加入此集群，重复添加被拒绝 | 若需重新加入，先从集群移除并销毁 CVM，再通过 `CreateClusterInstances` 新建实例加入。或使用集群中已有的该节点 |
| `cvm:RunInstances` 返回 `UnauthorizedOperation`：`you are not authorized with condition billing tag` | `tccli cvm DescribeInstances --region <Region>` 检查是否有可用 CVM | CAM 策略带标签条件拒绝独立创建 CVM 实例。此为环境限制 | 改用 `CreateClusterInstances`（该 API 走 TKE 路径，不受 `cvm:RunInstances` 限制）。或请管理员调整 CAM 策略标签条件 |
| `AddExistedInstances` 返回 `FailedInstanceIds` 含目标实例 | `tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]'` 检查实例状态和 VPC | CVM 不满足加入条件：不同 VPC、实例状态非 `RUNNING`、或被其他集群占用 | 确认实例与集群同 VPC、同地域、状态 `RUNNING`。若实例已占用，先注销原集群再重试 |
| `CreateClusterInstances` 返回 `InvalidParameter`（JSON 解析失败） | 检查 `RunInstancePara` 字段的 JSON 字符串嵌套转义 | `RunInstancePara` 是二次 JSON 序列化的字符串，嵌套引号需转义 | 使用 `--cli-input-json file://...` 方式传入，tccli 会自动处理转义。不要直接在命令行拼接 JSON 字符串 |
| 返回 `InvalidParameterValue` — `InstanceIds` | `tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]'` 确认实例存在 | 实例 ID 格式错误或实例不存在 | 确认 InstanceId 格式为 `<InstanceId>` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点加入后 `InstanceState` 长期 `initializing`（超过 5 分钟） | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]' --region <Region>` 查看 `InstanceState` 和 `FailedReason` | 节点仍在安装组件和初始化（正常）或网络/磁盘问题（异常） | 继续等待。超过 10 分钟仍 `initializing` → 使用 `DeleteClusterInstances --ForceDelete true --InstanceDeleteMode terminate` 删除该节点 → 重新创建 |
| 节点加入后 `InstanceState` 为 `failed` | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]' --region <Region>` 查看 `FailedReason` | 节点初始化失败：kubelet 未启动、网络不通、镜像不兼容 | 登录节点检查 kubelet/containerd 日志。修正后 `DeleteClusterInstances` 删除该节点 → 重新创建 |
| `DescribeClusterInstances` 返回 `FailedInstanceIds` 含目标实例 | 检查 `FailedReason` | `DeleteClusterInstances` 时节点处于初始化中状态，正常删除被拒 | 添加 `ForceDelete: true` 重试删除。保留 region、ClusterId、InstanceId、RequestId 以备工单查询 |
| kubectl 不可达：`Unable to connect to the server: dial tcp: no such host` 或 `server gave HTTP response to HTTPS client` | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region <Region>` 查看端点状态 | 集群无公网端点（CAM 硬策略拒绝），内网端点从本地网络不可达 | 在同 VPC 的 CVM 上执行 kubectl 命令，或通过 IOA/VPN/专线 连接内网端点。或使用 `DescribeClusterInstances` 代替 kubectl 验证节点状态 |

## 下一步

- [查看节点池](../查看节点池/tccli%20操作.md) — 查看节点池内节点状态
- [创建节点池](../创建节点池/tccli%20操作.md) — 创建节点池来批量管理节点
- [普通节点支持的 CVM 机型](../普通节点支持的%20CVM%20机型/tccli%20操作.md) — 查询可用机型

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点 → 添加已有节点](https://console.cloud.tencent.com/tke2/cluster/sub/create/addExistingNode) — 通过控制台将已有 CVM 加入集群。也可通过 [新建节点](https://console.cloud.tencent.com/tke2/cluster/sub/create/addNode) 新建 CVM 并加入集群。
