# 连接集群

> 对照官方：[连接集群](https://cloud.tencent.com/document/product/457/32191) · page_id `32191` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 kubectl 从本地客户端机器连接到 TKE 集群。TKE 提供两种连接方案：

| 方案 | 访问方式 | 适用场景 |
|------|---------|---------|
| 方案一：Cloud Shell 连接 | 控制台内嵌终端，无需配置本地网络 | 快速临时操作、无本地 kubectl 环境 |
| 方案二：本地连接 | kubectl + kubeconfig，通过集群端点连接 | 日常开发运维、CI/CD 集成 |

集群端点分为两种类型：

| 端点类型 | API 参数 | 网络可达性 | 说明 |
|---------|---------|-----------|------|
| 公网端点 | `IsExtranet=true` | 公网可达（需放通 TCP 443） | 需绑定安全组；受组织 CAM 策略限制 |
| 内网端点 | `IsExtranet=false` | VPC 内部可达 | 需指定子网；外部访问需 IOA/VPN/专线/同 VPC 跳板 |

> **注意**：开启公网访问前，确保集群内已有工作节点。空集群无法创建公网端点。如果公网端点被组织级 CAM 策略拒绝，回退到内网端点方案。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:CreateClusterEndpoint, tke:DeleteClusterEndpoint
#    tke:DescribeClusterEndpoints, tke:DescribeClusterEndpointStatus
#    tke:SwitchClusterEndpoint, tke:DeleteClusterEndpointVip
#    tke:DescribeClusterEndpointVipStatus
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 版本
kubectl version --client --short 2>/dev/null || kubectl version --client
# expected: Client Version 已安装
```

### 资源检查

```bash
# 5. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群基本信息 | `DescribeClusters` | 是 |
| 开启内网/公网访问 | `CreateClusterEndpoint` | 否 |
| 查看端点创建状态 | `DescribeClusterEndpointStatus` | 是 |
| 查看端点详情（IP/域名） | `DescribeClusterEndpoints` | 是 |
| 查看 VIP 状态 | `DescribeClusterEndpointVipStatus` | 是 |
| 切换公网/内网访问 | `SwitchClusterEndpoint` | 否 |
| 删除 VIP | `DeleteClusterEndpointVip` | 是 |
| 关闭访问入口 | `DeleteClusterEndpoint` | 是 |

### 关键字段说明

以下说明端点管理相关 API 的主要参数。

#### CreateClusterEndpoint

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx`。`DescribeClusters` 获取 | 集群不存在或非 Running → `ResourceUnavailable.ClusterState` |
| `IsExtranet` | Boolean | 是 | `true`（公网）/ `false`（内网）。公网需安全组 + 工作节点 | 公网但受 CAM 限制 → `InvalidParameter.Param`（CAM deny：strategyId 拒绝 `tke:clusterExtranetEndpoint=true`） |
| `SubnetId` | String | 否* | 内网端点必填。格式 `subnet-xxxxxxxx`。`DescribeSubnets` 查询 | 子网不存在或不属于集群 VPC → `InvalidParameter.SubnetId` |
| `SecurityGroup` | String | 否* | 公网端点必填。安全组需放通 TCP 443 入站。`DescribeSecurityGroups` 查询 | 不传 → `InvalidParameter: SecurityGroup can not be empty` |
| `Domain` | String | 否 | 自定义域名，不传则使用默认 `cls-xxx.ccs.tencent-cloud.com` | 域名格式不合法 → `InvalidParameter.Domain` |
| `ExtensiveParameters` | String | 否 | 扩展参数，CLB 相关高级配置 | — |
| `ExistedLoadBalancerId` | String | 否 | 已有 CLB 实例 ID，不传则自动创建 | CLB 不存在或不可用 → `InvalidParameter.ExistedLoadBalancerId` |

\* SubnetId 在内网模式下必填，SecurityGroup 在公网模式下必填。

#### SwitchClusterEndpoint

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `IsExtranet` | Boolean | 是 | 切换目标：`true`（公网）/ `false`（内网） | 目标端点类型不存在 → `ResourceUnavailable.ClusterState` |
| `Rollback` | Boolean | 否 | `true` 回滚到切换前状态。默认 `false` | — |

#### 端点类型选择决策

| 选择 | 条件 | 优势 | 限制 |
|------|------|------|------|
| 内网端点（`IsExtranet=false`） | 推荐默认选择 | 无需安全组，VPC 内安全隔离，不受公网 CAM 策略限制 | 外部需 IOA/VPN/专线/同 VPC 跳板才能 kubectl 连接 |
| 公网端点（`IsExtranet=true`） | 需公网可达 + 组织 CAM 策略允许 + 集群有工作节点 | kubectl 可直接从公网连接 | 需安全组放通 443，受 CAM 策略限制，需集群有节点 |

> **本次决策**：选择内网端点（`IsExtranet=false`）。公网端点被组织级 CAM 策略 `strategyId:240463971` 以条件 `tke:clusterExtranetEndpoint=true` 硬拒绝，无法创建。

## 操作步骤

### 步骤 1：查询集群基本信息

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "MaxNodePodNum": 64,
                "MaxClusterServiceNum": 4096,
                "VpcId": "<VpcId>",
                "SubnetId": "<SubnetId>",
                "Cni": true
            }
        }
    ],
    "RequestId": "..."
}
```

确认 `ClusterStatus` 为 `Running` 后再继续。记录 `VpcId` 和 `SubnetId`，后续创建端点时需要。

### 步骤 2：创建内网端点

#### 选择依据

- **端点类型**：选择内网端点（`IsExtranet=false`）。公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝。内网端点通过 IOA/VPN/专线或同 VPC CVM 跳板连接集群。
- **子网选择**：使用与集群创建时相同的子网，确保内网端点 IP 与集群在同一 VPC 内可达。查询子网：`tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]' --region <Region>`。
- **常见错误**：创建公网端点不传 `SecurityGroup` → `InvalidParameter: SecurityGroup can not be empty`；内网端点不传 `SubnetId` → `InvalidParameter.SubnetId`。

#### 最小创建（内网端点，必填字段）

```bash
tccli tke CreateClusterEndpoint \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --SubnetId <SubnetId> \
    --region <Region>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，状态必须 Running | `tccli tke DescribeClusters --region <Region>` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，必须属于集群 VPC | `tccli vpc DescribeSubnets --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看 |

#### 增强配置（公网端点，仅供参考——可能被 CAM 拒绝）

公网端点无法在当前环境创建（受 CAM `strategyId:240463971` 限制），以下命令仅供参考：

```bash
# 创建公网端点（需先确保 CAM 策略放通 + 集群有工作节点）
tccli tke CreateClusterEndpoint \
    --ClusterId <ClusterId> \
    --IsExtranet true \
    --SecurityGroup <SecurityGroupId> \
    --region <Region>
# expected: 如受 CAM 限制 → InvalidParameter.Param，strategyId:240463971
```

> **注意**：创建公网端点前需确保安全组已放通 TCP 443 入站规则。安全组获取方式：`tccli vpc DescribeSecurityGroups --region <Region>`。

### 步骤 3：轮询端点创建状态

端点创建是异步操作，需轮询至状态为 `Created`。创建耗时约 2 分钟。

```bash
tccli tke DescribeClusterEndpointStatus \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --region <Region>
# expected: Status: "Created"
```

**预期输出**：

```json
{
    "Status": "Created",
    "ErrorMsg": "Created",
    "RequestId": "..."
}
```

| 状态 | 含义 | 操作 |
|------|------|------|
| `Creating` | 创建中 | 等待 10-15 秒后重新轮询 |
| `Created` | 创建成功 | 继续下一步 |
| `CreateFailed` | 创建失败 | 查看 `ErrorMsg` → 参见 [排障](#排障) |

### 步骤 4：查询集群端点详细信息

端点创建成功后，查询内网 IP 和域名。

```bash
tccli tke DescribeClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，ClusterIntranetEndpoint 为内网 IP
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "<EndpointIp>",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterExternalACL": null,
    "ClusterExternalDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetDomain": "",
    "SecurityGroup": "",
    "ClusterIntranetSubnetId": "<SubnetId>",
    "IntranetSecurityGroup": "",
    "CertificationAuthority": "-----BEGIN CERTIFICATE-----\nMIIDCjCCAfKgAwIBAgIBAD...\n-----END CERTIFICATE-----"
}
```

| 字段 | 说明 |
|------|------|
| `ClusterIntranetEndpoint` | 内网端点 IP，仅 VPC 内部可达。作为 kubectl 的 `--server` 地址 |
| `ClusterExternalEndpoint` | 公网端点。空字符串表示未开启公网访问 |
| `ClusterDomain` | 集群默认域名 |
| `ClusterIntranetSubnetId` | 内网端点绑定的子网 ID |
| `CertificationAuthority` | 集群 CA 证书，用于 kubectl 连接 |

### 步骤 5：查询 VIP 状态

VIP 用于公网端点的安全策略管理。未创建公网端点时状态为 `Deleted`，这是正常行为。

```bash
tccli tke DescribeClusterEndpointVipStatus \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: 未创建 VIP 时 Status: "Deleted"
```

**预期输出**：

```json
{
    "Status": "Deleted",
    "ErrorMsg": "Deleted",
    "RequestId": "..."
}
```

### 步骤 6：切换集群端点

`SwitchClusterEndpoint` 用于切换公网/内网访问模式，或在公网访问异常时触发切换（配合 `Rollback` 参数回滚）。

```bash
# 保持内网模式（当前推荐）
tccli tke SwitchClusterEndpoint \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --Rollback false \
    --region <Region>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认端点状态为 Created
tccli tke DescribeClusterEndpointStatus \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --region <Region> \
    | jq '.Status'
# expected: "Created"

# 2. 确认内网 IP 已分配
tccli tke DescribeClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.ClusterIntranetEndpoint'
# expected: 非空字符串，如 "172.x.x.x"

# 3. 确认子网绑定正确
tccli tke DescribeClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.ClusterIntranetSubnetId'
# expected: "<SubnetId>"

# 4. 确认证书已生成
tccli tke DescribeClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.CertificationAuthority' | head -c 50
# expected: "-----BEGIN CERTIFICATE-----"
```

验证维度汇总：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeClusterEndpointStatus` | `Status: "Created"` |
| 内网 IP | `DescribeClusterEndpoints` → `ClusterIntranetEndpoint` | 非空 IP |
| 子网绑定 | `DescribeClusterEndpoints` → `ClusterIntranetSubnetId` | 与创建参数一致 |
| CA 证书 | `DescribeClusterEndpoints` → `CertificationAuthority` | 以 `-----BEGIN CERTIFICATE-----` 开头 |

### 数据面（kubectl）

内网端点 IP 仅 VPC 内部可达。从外部需通过 IOA/VPN/专线或同 VPC 跳板机访问。

```bash
# 1. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq -r '.Kubeconfig' > ~/.kube/config-tke

# 2. 验证连通性（需网络可达）
kubectl cluster-info --kubeconfig ~/.kube/config-tke
# expected: Kubernetes control plane is running at https://<EndpointIp>
```

```text
Kubernetes control plane is running at https://<EndpointIp>
CoreDNS is running at https://<EndpointIp>/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
# 3. 查看命名空间
kubectl get ns --kubeconfig ~/.kube/config-tke
# expected: 返回 default、kube-system 等系统命名空间
```

```text
NAME              STATUS   AGE
default           Active   ...
kube-node-lease   Active   ...
kube-public       Active   ...
kube-system       Active   ...
```

> **注意**：`kubectl cluster-info` 返回 `i/o timeout` 或 `connection refused` 说明网络不可达。确认当前网络环境能否访问集群内网 IP `<EndpointIp>:443`。如需从公网访问，参见 [排障](#排障) 中网络不可达排查。

## 清理

> **警告**：删除内网端点后 kubectl 连接将立即断开，生产环境操作前通知相关人员。端点删除不影响集群本身和已部署的工作负载，仅关闭控制面访问通道。端点删除操作幂等——端点已删除时再次执行也不会报错。

### 1. 删除 VIP

即使当前无 VIP（未创建公网端点），`DeleteClusterEndpointVip` 也是幂等操作。

```bash
tccli tke DeleteClusterEndpointVip \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

### 2. 删除内网端点

```bash
tccli tke DeleteClusterEndpoint \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --region <Region>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> **注意**：`DeleteClusterEndpoint` 必须指定 `--IsExtranet true/false`，与创建时的值一致。

### 3. 验证端点已删除

```bash
tccli tke DescribeClusterEndpointStatus \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    --region <Region>
# expected: Status: "Deleted"
```

**预期输出**：

```json
{
    "Status": "Deleted",
    "ErrorMsg": "Deleted",
    "RequestId": "..."
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterEndpoint`（公网）返回 `InvalidParameter: SecurityGroup can not be empty` | 检查命令中是否缺少 `--SecurityGroup` 参数 | 公网端点必须绑定安全组，未传则报此错误 | 添加 `--SecurityGroup <SecurityGroupId>`。安全组需放通 TCP 443 入站。`tccli vpc DescribeSecurityGroups --region <Region>` 查询可用安全组 |
| `CreateClusterEndpoint`（公网）返回 `InvalidParameter.Param`，含 `CAM deny` 和 `strategyId:240463971` | `tccli configure list` 检查当前账号。查看错误消息中 condition `tke:clusterExtranetEndpoint=true` | 公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝。此为环境限制，非命令错误 | 使用内网端点（`--IsExtranet false`）替代。内网访问需通过 IOA/VPN/专线或同 VPC CVM 跳板 |
| `CreateClusterEndpointVip` 返回 `ResourceUnavailable.ClusterState`，含 `cluster without worker Node` | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>` 查 `ClusterNodeNum` | VIP 需要集群存在工作节点 + 已开启公网端点。当前集群为空集群 | 先通过 `CreateClusterNodePool` 添加至少一个工作节点，确保 `ClusterNodeNum >= 1` 且公网端点已开启后再操作 |
| `ModifyClusterEndpointSP` 返回 `OperationDenied`，含 `only for managed cluster with extranet endpoint` | 检查当前是否使用内网端点 | `ModifyClusterEndpointSP` 仅用于公网端点的安全策略配置，内网端点不支持 | 如使用内网端点，通过子网关联的安全组实现网络层访问控制。仅在使用 `--IsExtranet true` 时使用此 API |
| `CreateClusterEndpoint` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]' --region <Region>` 查询可用子网 | 子网不存在或不属于目标 VPC | 使用与集群同 VPC 的子网 ID。`DescribeSubnets` 确认 `VpcId` 与集群 `VpcId` 一致 |
| `DeleteClusterEndpoint` 返回 `MissingParameter: IsExtranet` | 检查命令中是否缺少 `--IsExtranet` 参数 | `DeleteClusterEndpoint` 必须指定 `--IsExtranet true/false` | 添加 `--IsExtranet false`（内网）或 `--IsExtranet true`（公网），与创建时的值一致 |
| `DescribeClusterEndpoints` 返回空结果或 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 确认集群存在 | 集群 ID 格式错误或集群不在当前账号/地域 | 核对 `ClusterId` 拼写和地域，`DescribeClusters` 列出所有集群确认 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 端点创建返回 RequestId 但 3 分钟后 `DescribeClusterEndpointStatus` 返回 `CreateFailed` | `tccli tke DescribeClusterEndpointStatus --ClusterId <ClusterId> --IsExtranet false --region <Region>` 查看 `ErrorMsg` | CLB 创建失败或子网不可用。记录 `ErrorMsg` 和 `RequestId` | 检查子网 IP 是否充足（`DescribeSubnets` 查看 `AvailableIpCount`）。如 IP 不足，选其他子网或释放闲置 IP 后重试 |
| 内网端点 `Created` 但 `kubectl cluster-info` 返回 `i/o timeout` | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region <Region>` 获取 `ClusterIntranetEndpoint`，用 `ping`/`telnet` 测试可达性 | 当前网络环境无法访问 VPC 内部 IP。内网端点仅 VPC 内可达 | 方案一：通过 IOA/VPN/专线 接入 VPC。方案二：同 VPC 内创建 CVM 跳板机，通过跳板执行 kubectl。方案三：（如 CAM 策略允许）改用公网端点 |
| 公网端点无法创建但未见明确错误码 | 保留 `RequestId`、`ClusterId`、`region`、创建 JSON | 组织 CAM 策略可能静默拒绝 | 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |

## 下一步

- [创建集群](../../../../集群配置/集群管理/创建集群/tccli%20操作.md) — 通过 CLI 创建托管集群
- [集群生命周期](../../../../集群配置/集群管理/集群生命周期/tccli%20操作.md) — 集群状态流转与生命周期管理
- [集群扩缩容](../../../../集群配置/集群管理/集群扩缩容/tccli%20操作.md) — 托管集群规格调整
- [删除集群](../../../../集群配置/集群管理/删除集群/tccli%20操作.md) — 通过 CLI 删除集群

## 控制台替代

[TKE 控制台 → 集群列表 → 选择集群 → 基本信息 → 集群 APIServer 信息](https://console.cloud.tencent.com/tke2/cluster)：查看/操作公网和内网访问入口。
