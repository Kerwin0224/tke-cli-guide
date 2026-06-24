# 新建超级节点

> 对照官方：[新建超级节点](https://cloud.tencent.com/document/product/457/78328) · page_id `78328` · tccli ≥3.1.107.1 · API 2018-05-25

## 概述

超级节点是 TKE 的 Serverless 节点类型，无需购买和管理底层 CVM，Pod 直接调度到虚拟节点上运行，按实际使用资源计费。创建超级节点分两步：**先建超级节点池**（配置安全组、子网、标签等），**再在池内创建超级节点**（支持指定 DisplayName、Labels、Quota 资源规格）。

| 操作 | 说明 |
|------|------|
| 创建超级节点池 | `CreateClusterVirtualNodePool` — 定义节点池的基础配置（子网、安全组、标签、污点），作为超级节点的配置模板 |
| 创建超级节点 | `CreateClusterVirtualNode` — 在已就绪的节点池内创建单个超级节点，可配置资源配额和标签 |

超级节点池相当于超级节点的**配置模板**，超级节点是真正承载 Pod 的运行时单元。一个节点池下可以有多个超级节点。

## 前置条件

- [环境准备](https://cloud.tencent.com/document/product/457/78328) — 确保已安装 tccli ≥3.1.107.1 并完成凭据配置。详见《面向 Agent 的 CLI 操作指南》的「环境准备」章节

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限
#    需要: tke:CreateClusterVirtualNodePool, tke:CreateClusterVirtualNode,
#          tke:DescribeClusterVirtualNodePools, tke:DescribeClusterVirtualNode,
#          tke:ModifyClusterVirtualNodePool, tke:DrainClusterVirtualNode,
#          tke:DeleteClusterVirtualNode, tke:DeleteClusterVirtualNodePool,
#          tke:DescribeClusters, vpc:DescribeSubnets, vpc:CreateSecurityGroup
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
# 验证：执行 DescribeClusterVirtualNodePools 确认超级节点相关权限
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）

**预期输出**：

```json
{
    "TotalCount": 0,
    "NodePoolSet": [],
    "RequestId": "..."
}
```
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "RequestId": "..."
}
```

> `<VpcId>` 来源：从上一步 `DescribeClusters` 输出中提取，路径 `$.Clusters[0].ClusterNetworkSettings.VpcId`。

```bash
# 5. 查询集群 VPC 下的可用子网（超级节点需要至少 1 个子网）
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]' \
    | jq '.SubnetSet[] | {SubnetId, CidrBlock, AvailableIpAddressCount}'
# expected: 至少返回 1 个子网，AvailableIpAddressCount ≥ 3
```

**预期输出**：

```json
{
    "SubnetId": "subnet-example",
    "CidrBlock": "10.0.4.0/22",
    "AvailableIpAddressCount": 4093
}
```

```bash
# 6. 提前创建安全组（如无可用的安全组）
tccli vpc CreateSecurityGroup --region <Region> \
    --GroupName kerwinwjyan-super-node-sg \
    --GroupDescription 超级节点安全组
# expected: 返回 SecurityGroupId
```

**预期输出**：

```json
{
    "SecurityGroup": {
        "SecurityGroupId": "sg-example",
        "SecurityGroupName": "kerwinwjyan-super-node-sg"
    },
    "RequestId": "..."
}
```

> 安全组需提前创建。超级节点池创建时不接受安全组名称，只接受安全组 ID。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池列表 | `DescribeClusterVirtualNodePools` | 是 |
| 新建超级节点池 | `CreateClusterVirtualNodePool` | 否 |
| 修改超级节点池 | `ModifyClusterVirtualNodePool` | 是 |
| 新建超级节点 | `CreateClusterVirtualNode` | 否 |
| 查看超级节点列表 | `DescribeClusterVirtualNode` | 是 |
| 驱逐超级节点 | `DrainClusterVirtualNode` | 是 |
| 删除超级节点 | `DeleteClusterVirtualNode` | 否 |
| 删除超级节点池 | `DeleteClusterVirtualNodePool` | 否 |

## 关键字段说明

以下说明 `CreateClusterVirtualNodePool` 和 `CreateClusterVirtualNode` 的主要参数。

### CreateClusterVirtualNodePool

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx`。集群必须状态为 `Running`，且网络模式兼容超级节点（GR 或已开启 VPC-CNI 的集群） | 集群不存在或状态异常 → `ResourceUnavailable.ClusterState` |
| `Name` | String | 是 | 节点池名称，长度 1-60 字符。不可与已有节点池重名 | 名称重复 → `InvalidParameter.DuplicateName` |
| `SubnetIds` | Array[String] | 是 | 子网 ID 列表，如 `["subnet-xxxxxxxx"]`。子网必须属于集群 VPC，且可用 IP ≥ 3 | 子网不在集群 VPC 中 → `InvalidParameter.SubnetId` |
| `SecurityGroupIds` | Array[String] | 是 | 安全组 ID 列表，如 `["sg-xxxxxxxx"]`。安全组需提前通过 `vpc CreateSecurityGroup` 创建 | 安全组不存在 → `InvalidParameter.SecurityGroupId` |
| `Labels` | Array[Object] | 否 | 节点池标签，格式 `[{"Name":"key","Value":"val"}]`。标签会级联应用到池内所有超级节点 | 格式错误 → `InvalidParameter.Labels` |
| `Taints` | Array[Object] | 否 | 节点池污点，格式 `[{"Key":"key","Value":"val","Effect":"NoSchedule"}]` | — |
| `DeletionProtection` | Boolean | 否 | 删除保护开关，默认 `false`。`true` 时需先关闭才能删除节点池 | 忘开删除保护 → 可能误删节点池 |
| `VirtualNodes` | Array[Object] | 否 | 创建节点池时同步创建超级节点列表，每个元素含 `DisplayName`、`SubnetId`、`Quota` 等 | — |

### CreateClusterVirtualNode

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 已就绪的超级节点池 ID，格式 `np-xxxxxxxx`。节点池 LifeState 必须为 `normal` | 节点池未就绪 → `ResourceUnavailable.NodePoolStateNotNormal` |
| `VirtualNodes` | Array[Object] | 是 | 超级节点配置列表。每个元素含 `DisplayName`（显示名称）、`SubnetId`（子网 ID）、`Labels`（可选）、`Quota`（可选） | — |
| `VirtualNodes[].DisplayName` | String | 否 | 超级节点在集群中的显示名称 | — |
| `VirtualNodes[].SubnetId` | String | 是 | 超级节点绑定的子网 ID，须在节点池 `SubnetIds` 范围内 | 子网不在节点池范围 → `InvalidParameter.SubnetId` |
| `VirtualNodes[].Labels` | Array[Object] | 否 | 节点标签，格式 `[{"Name":"key","Value":"val"}]`。用于 Pod 的 nodeSelector 调度 | — |
| `VirtualNodes[].Quota` | Object | 否 | 资源配额。含 `QuotaType`（`fuzzy` 模糊 / `exact` 精确）及对应的 CPU、Memory 等规格 | `exact` 类型缺 Cpu/Memory → 节点创建失败 |
| `VirtualNodes[].Quota.QuotaType` | String | 否 | `fuzzy`（模糊配额，按总量弹性分配）或 `exact`（精确配额，需同时指定 Cpu、Memory、Num） | — |

## 操作步骤

### 步骤 1：查询现有超级节点池

先确认集群下是否已有超级节点池，避免创建重名池。

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）
```

**预期输出**（无现有节点池）：

```json
{
    "TotalCount": 0,
    "NodePoolSet": [],
    "RequestId": "..."
}
```

### 步骤 2：创建超级节点池

#### 选择依据

*为什么选这些值：*

- **网络模式**：当前集群为 GR（GlobalRouter）模式，与超级节点池兼容。如果使用 VPC-CNI 模式的集群创建超级节点，需确认 `EnableVpcCniNetworkType` 已启用。

- **子网选择**：选择集群 VPC 内可用 IP 充足的子网（≥ 3 个）。超级节点占用子网 IP，预留足够 IP 避免后续扩容受限。查询可用子网：
  `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]' | jq '.SubnetSet[] | {SubnetId, CidrBlock, AvailableIpAddressCount}'`

- **安全组**：安全组需提前创建，不接受名称只接受 ID。安全组控制超级节点上 Pod 的网络出入规则。创建安全组：
  `tccli vpc CreateSecurityGroup --region <Region> --GroupName kerwinwjyan-super-node-sg --GroupDescription 超级节点安全组`

- **标签**：推荐在节点池级别设置标签，如 `{"Name":"env","Value":"production"}`，标签会级联应用到池内所有超级节点，方便 Pod 调度和资源组织。

- **删除保护**：建议开启（`DeletionProtection: true`），防止误删节点池导致超级节点全部销毁。删除前需先关闭保护。

#### 最小创建（只含必填字段）

`vnode-pool-minimal.json`：

```json
{
    "ClusterId": "cls-example",
    "Name": "demo-super-node-pool",
    "SubnetIds": ["subnet-example"],
    "SecurityGroupIds": ["sg-example"]
}
```

```bash
tccli tke CreateClusterVirtualNodePool --region <Region> \
    --cli-input-json file://vnode-pool-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "..."
}
```

#### 增强配置（加标签、污点、删除保护）

`vnode-pool-enhanced.json`：

```json
{
    "ClusterId": "cls-example",
    "Name": "<NodePoolName>",
    "SubnetIds": ["subnet-example"],
    "SecurityGroupIds": ["sg-example"],
    "Labels": [
        {"Name": "env", "Value": "production"},
        {"Name": "team", "Value": "platform"}
    ],
    "Taints": [
        {"Key": "dedicated", "Value": "serverless", "Effect": "NoSchedule"}
    ],
    "DeletionProtection": true
}
```

```bash
tccli tke CreateClusterVirtualNodePool --region <Region> \
    --cli-input-json file://vnode-pool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<VpcId>` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs` |

### 步骤 3：轮询确认节点池已就绪

节点池创建后最初状态为 `creating`，需轮询至 `LifeState` 变为 `normal`（约 2-3 分钟）后才可创建超级节点。

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.NodePoolSet[] | select(.NodePoolId == "<NodePoolId>") | {NodePoolId, LifeState}'
# expected: LifeState: "normal"
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "LifeState": "normal"
}
```

> 超时未变为 `normal`，参见 [排障](#排障) 中"节点池 LifeState 长时间不变为 normal"条目。

### 步骤 4：创建超级节点

#### 选择依据

*为什么选这些值：*

- **配额类型**：推荐 `fuzzy`（模糊配额）。模糊配额按资源总量弹性分配，适合通用场景——超级节点会自动根据 Pod 请求分配资源，无需精确预设。`exact`（精确配额）需指定具体的 CPU、Memory、Num 等字段，适合需要精确控制资源规格的场景，但必须同时填写完整规格，否则节点创建失败。

- **--cli-input-json vs 行内参数**：`CreateClusterVirtualNode` 的 `VirtualNodes` 参数是嵌套数组，`--cli-unfold-argument` 行内方式在嵌套字段上会产生 `ambiguous option` 错误。**强烈推荐使用 `--cli-input-json file://` 方式传入参数。**

- **Labels**：在超级节点上设置 Labels，配合 Pod 的 `nodeSelector` 可实现定向调度。如设置 `{"Name":"app","Value":"demo"}`，Pod 可通过 `nodeSelector: {app: demo}` 调度到该节点。

- **DisplayName**：设置有意义的显示名称，便于在集群中识别不同超级节点。

#### 4.1 最简形式（无额外配置）

> `<SubnetId>` 来源：使用步骤 2 中 `vnode-pool-minimal.json` 或 `vnode-pool-enhanced.json` 的 `SubnetIds` 数组中的子网 ID。超级节点的 `SubnetId` 必须在节点池的 `SubnetIds` 范围内。

```bash
tccli tke CreateClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --SubnetId <SubnetId>
# expected: exit 0，返回 NodeName
```

**预期输出**：

```json
{
    "NodeName": "eklet-subnet-subnet-example-xxxxxxxxx",
    "RequestId": "..."
}
```

#### 4.2 带 Labels 标签

`create-vnode-labels.json`：

```json
{
    "ClusterId": "cls-example",
    "NodePoolId": "np-example",
    "VirtualNodes": [
        {
            "DisplayName": "demo-with-labels",
            "SubnetId": "subnet-example",
            "Labels": [
                {"Name": "app", "Value": "demo"},
                {"Name": "env", "Value": "test"}
            ]
        }
    ]
}
```

```bash
tccli tke CreateClusterVirtualNode --region <Region> \
    --cli-input-json file://create-vnode-labels.json
# expected: exit 0，返回 NodeName
```

**预期输出**：

```json
{
    "NodeName": "eklet-subnet-subnet-example-xxxxxxxxx",
    "RequestId": "..."
}
```

#### 4.3 带 Quota 资源规格

`create-vnode-quota.json`：

```json
{
    "ClusterId": "cls-example",
    "NodePoolId": "np-example",
    "VirtualNodes": [
        {
            "DisplayName": "demo-with-quota",
            "SubnetId": "subnet-example",
            "Quota": {
                "Cpu": 2.0,
                "Memory": 4.0,
                "QuotaType": "fuzzy"
            }
        }
    ]
}
```

```bash
tccli tke CreateClusterVirtualNode --region <Region> \
    --cli-input-json file://create-vnode-quota.json
# expected: exit 0，返回 NodeName
```

**预期输出**：

```json
{
    "NodeName": "eklet-subnet-subnet-example-xxxxxxxxx",
    "RequestId": "..."
}
```

> **注意**：`--cli-unfold-argument` 方式在 `VirtualNodes` 嵌套数组上会产生 `ambiguous option: --VirtualNodes could match --VirtualNodes.0.DisplayName, ...` 错误。始终使用 `--cli-input-json file://` 方式。

### 步骤 5：查看超级节点列表

```bash
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回节点列表
```

**预期输出**：

```json
{
    "Nodes": [
        {
            "Name": "eklet-subnet-subnet-example-xxxxxxxxx",
            "SubnetId": "subnet-example",
            "Phase": "Running",
            "CreatedTime": "2026-06-23T08:16:04Z"
        },
        {
            "Name": "eklet-subnet-subnet-example-yyyyyyyyy",
            "SubnetId": "subnet-example",
            "Phase": "Pending",
            "CreatedTime": "2026-06-23T08:15:04Z"
        }
    ],
    "TotalCount": 2,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `Name` | 超级节点名称，格式 `eklet-subnet-xxxxxxxxx-xxxxxxxxx` |
| `Phase` | 节点状态：`Pending`（启动中）/ `Running`（就绪）/ `Deleting`（删除中） |
| `SubnetId` | 节点绑定的子网 ID |

### 步骤 6：修改超级节点池（可选）

可用于修改节点池的 Labels、Taints、名称、安全组、删除保护等属性。

```bash
tccli tke ModifyClusterVirtualNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --Labels '[{"Name":"modified","Value":"true"}]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> **副作用声明**：
> - **影响范围**：修改 Labels 会级联应用到池内**所有**超级节点。如果 Pod 通过 `nodeSelector` 绑定了被修改/删除的 Label，后续新建 Pod 将无法调度到这些节点。已调度的 Pod 不受影响。
> - **不可逆**：Labels 修改后立即生效，如需回滚需再次执行 `ModifyClusterVirtualNodePool` 恢复原 Labels。
> - **修改 Taints 同样级联**：修改 Taints 也会级联应用到池内所有超级节点，影响 Pod 的容忍调度策略。

### 步骤 7：驱逐超级节点（清理前操作）

删除超级节点前，建议先驱逐节点上的 Pod，确保工作负载被重新调度。

> `<NodeName>` 来源：从步骤 5 `DescribeClusterVirtualNode` 输出中提取，路径 `$.Nodes[].Name`，格式为 `eklet-subnet-xxxxxxxxx-xxxxxxxxx`。每次驱逐一个节点。

```bash
tccli tke DrainClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodeName <NodeName>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> **注意**：参数名为 `--NodeName`（单数），非 `--NodeNames`。每次驱逐一个节点。驱逐前确保 Pod 可被调度到其他超级节点。
>
> **副作用声明**：
> - **影响范围**：`DrainClusterVirtualNode` 会驱逐**目标超级节点上的所有 Pod**，Pod 被终止后由控制器（Deployment/StatefulSet 等）在其他可用超级节点上重新创建。**不影响**节点池内的其他超级节点。
> - **不可逆**：驱逐是单向操作，Pod 被终止后无法原地恢复。驱逐前请确保 Pod 的控制器能正常在其他节点上重建 Pod。
> - **区别于 `--Force true`**：`DeleteClusterVirtualNode --Force true` 跳过 Pod 检查直接删除节点，Pod 会被**直接终止而不等待优雅关闭**，与 Drain 的优雅驱逐行为不同。生产环境应优先使用 Drain → Delete 两步操作。

## 验证

### 控制面（tccli）

```bash
# 确认节点池处于 normal 状态
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.NodePoolSet[] | select(.NodePoolId == "<NodePoolId>") | {NodePoolId, LifeState, Name}'
# expected: LifeState: "normal"
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "LifeState": "normal",
    "Name": "demo-super-node-pool"
}
```

```bash
# 确认超级节点列表
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    | jq '.Nodes[] | {Name, Phase, SubnetId}'
# expected: 返回已创建的超级节点，Phase 为 Running
```

**预期输出**：

```json
{
    "Name": "eklet-subnet-subnet-example-xxxxxxxxx",
    "Phase": "Running",
    "SubnetId": "subnet-example"
}
{
    "Name": "eklet-subnet-subnet-example-yyyyyyyyy",
    "Phase": "Running",
    "SubnetId": "subnet-example"
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池状态 | `DescribeClusterVirtualNodePools` 检查 `LifeState` | `normal` |
| 节点数量 | `DescribeClusterVirtualNode` 检查 `TotalCount` | 与创建数量一致 |
| 节点状态 | 同上，检查 `Phase` | 所有节点 `Phase: "Running"` |
| 节点配置 | 同上，检查 `SubnetId` | 与创建时指定的子网一致 |

### 数据面（kubectl）

```bash
kubectl get nodes -o wide
# expected: 返回包含超级节点的节点列表
```

> **网络可达性说明**：超级节点通过 kubectl 访问时，需集群端点可达。如果集群仅配置了内网端点且当前网络环境无法直接访问（未通过 IOA/VPN/专线 或同 VPC CVM），`kubectl` 可能返回 `http: server gave HTTP response to HTTPS client` 错误。这是因为内网端点部署了 HTTP 代理，kubectl 的 HTTPS 请求被代理返回 HTTP 响应。可尝试用域名 `<ClusterId>.ccs.tencent-cloud.com` 连接，或通过 `DescribeClusterSecurity` 获取备选 kubeconfig。完整排查路径见 [排障](#排障)。

## 清理

> **⚠️ 警告**：
> - 删除超级节点池前需**先删除池内所有超级节点**，否则会报 `ResourceInUse.ExistRunningPod`。
> - 使用 `--Force true` 强制删除会**跳过 Pod 检查**，可能导致运行中 Pod 被直接终止，造成业务中断。
> - 节点池/节点删除后状态先变为 `deleting`，资源实际释放需数分钟，最终从列表中消失。
> - 确认节点池删除后，请检查关联资源（安全组、子网）是否需要单独清理——`DeleteClusterVirtualNodePool` 不会删除安全组和子网。

### 控制面（tccli）

#### 1. 清理前状态检查

```bash
# 确认节点池内超级节点列表（待删除）
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，确认是待删除的目标节点
```

**预期输出**：

```json
{
    "Nodes": [
        {
            "Name": "eklet-subnet-subnet-example-xxxxxxxxx",
            "SubnetId": "subnet-example",
            "Phase": "Running"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

#### 2. 驱逐超级节点

> `<NodeName>` 来源：从步骤 1 清理前状态检查（`DescribeClusterVirtualNode`）输出中提取，路径 `$.Nodes[].Name`。如有多个节点，逐个执行驱逐。

```bash
# 对每个超级节点执行驱逐，确保 Pod 被重新调度
tccli tke DrainClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodeName <NodeName>
# 如有多个节点，逐个执行
```

#### 3. 关闭删除保护（如启用）

```bash
tccli tke ModifyClusterVirtualNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DeletionProtection false
```

#### 4. 删除超级节点

```bash
# 如节点上有运行中 Pod，加 --Force true 跳过 Pod 检查
tccli tke DeleteClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodeNames '["<NodeName>"]' \
    --Force true
# ⚠️ --Force true 会跳过 Pod 检查，可能导致 Pod 被直接终止
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> 非强制删除时，如果节点上有运行中 Pod 会报 `ResourceInUse.ExistRunningPod`。错误消息中会列出具体的 Pod 名称。可先 `kubectl drain` 排空 Pod，或使用 `--Force true` 强制删除。

#### 5. 删除超级节点池

```bash
# 确认池内所有节点已删除后，删除节点池
tccli tke DeleteClusterVirtualNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --Force true
# ⚠️ --Force true 会级联强制删除池内所有超级节点
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

#### 6. 验证已删除

```bash
# 验证节点池已删除
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.NodePoolSet | map(select(.NodePoolId == "<NodePoolId>"))'
# expected: 空列表 []
```

**预期输出**：

```json
[]
```

#### 7. 清理安全组和子网（可选）

`DeleteClusterVirtualNodePool` 不会自动删除安全组和子网。如果前置条件中自建了安全组和子网，且确认无其他资源占用，按需单独清理。

> **注意**：安全组/子网为共享基础设施，可能被集群内其他资源（如其他节点池、服务）引用。删除前请确认无其他依赖。

```bash
# 7a. 清理自建安全组（确认无其他资源引用）
tccli vpc DescribeSecurityGroups --region <Region> \
    --Filters '[{"Name":"group-name","Values":["kerwinwjyan-super-node-sg"]}]'
# 确认返回的 SecurityGroupId 无其他资源引用后执行删除
tccli vpc DeleteSecurityGroup --region <Region> \
    --SecurityGroupId <SecurityGroupId>
# expected: exit 0

# 7b. 清理子网（确认无其他资源占用，且非集群默认子网）
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# 确认返回的子网仅被当前节点池使用后执行删除
tccli vpc DeleteSubnet --region <Region> \
    --SubnetId <SubnetId>
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteClusterVirtualNode`（非强制）返回 `ResourceInUse.ExistRunningPod` | 错误消息中列出具体 Pod 名称：`pods still running on node eklet-subnet-xxxxxxxxx-xxxxxxxxx, pods: kube-system/coredns-...` | 超级节点上有运行中 Pod，非强制删除拒绝执行 | 先通过 `kubectl drain <NodeName>` 排空 Pod 后重试；或执行 `tccli tke DeleteClusterVirtualNode --ClusterId <ClusterId> --NodeNames '["<NodeName>"]' --Force true --region <Region>` 强制删除 |
| `CreateClusterVirtualNode`（`--cli-unfold-argument`）返回 `ambiguous option: --VirtualNodes could match --VirtualNodes.0.DisplayName, ...` | 检查命令参数格式是否为 `--cli-unfold-argument` 方式 | `--cli-unfold-argument` 在处理嵌套数组参数时产生歧义，无法区分 `--VirtualNodes` 和 `--VirtualNodes.0.DisplayName` | 改用 `--cli-input-json file://create-vnode.json` 方式传入参数。示例 JSON 见 [步骤 4.2](#42-带-labels-标签) 和 [步骤 4.3](#43-带-quota-资源规格) |
| `CreateClusterVirtualNode` 返回 `ResourceUnavailable.NodePoolStateNotNormal` | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId> \| jq '.NodePoolSet[] \| {NodePoolId, LifeState}'` | 节点池 LifeState 不是 `normal`，尚在 `creating` 阶段 | 等待节点池 LifeState 变为 `normal`（约 2-3 分钟）后重试。如果长时间不变 normal，保留 RequestId + ClusterId + NodePoolId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看节点池状态 |
| `DrainClusterVirtualNode` 返回 `error: the following arguments are required: --NodeName` | 检查命令参数名 | 误用了 `--NodeNames`（复数），正确参数名为 `--NodeName`（单数） | 改为 `--NodeName <NodeName>`，每次驱逐一个节点 |
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 确认子网存在 | 子网 ID 错误或子网不属于集群 VPC | 用 `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` 获取集群 VPC 下的可用子网 |
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.SecurityGroupId` | `tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["<SecurityGroupId>"]'` 确认安全组存在 | 安全组 ID 错误或安全组不存在 | 用 `tccli vpc CreateSecurityGroup --region <Region> --GroupName kerwinwjyan-super-node-sg --GroupDescription 超级节点安全组` 创建安全组后重试 |
| `CreateClusterVirtualNodePool` 返回 `LimitExceeded` | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>` 查看已有节点池数量 | 超级节点池数量达到上限（此为环境限制，非命令错误） | 删除不再使用的超级节点池后重试 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 超级节点 Phase 长时间为 `Pending` | `tccli tke DescribeClusterVirtualNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 查看 `Phase` | 节点初始化缓慢或子网 IP 不足 | 检查子网 `AvailableIpAddressCount`；等待 5 分钟后仍为 Pending → 保留 ClusterId + NodePoolId + NodeName → [提交工单](https://console.cloud.tencent.com/workorder) |
| 节点池 LifeState 长时间不变为 `normal` | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId> \| jq '.NodePoolSet[] \| {NodePoolId, LifeState}'` | 后端创建慢或子网/安全组配置问题 | 检查安全组 ID 是否存在、子网是否属于集群 VPC。超过 5 分钟 → 保留 region + ClusterId + NodePoolId + RequestId → 登录控制台查看详细状态 |
| `kubectl get nodes` 返回 `http: server gave HTTP response to HTTPS client` | `kubectl cluster-info` 确认 API server 可达性 | 内网端点部署了 HTTP 代理，kubectl 的 HTTPS 请求被代理返回 HTTP 响应。集群无公网端点（`ClusterExternalEndpoint` 为空），当前网络环境下 kubectl 不可达 | 内网端点需通过 IOA/VPN/专线 或同 VPC 内 CVM 才能正常访问。可尝试用域名 `<ClusterId>.ccs.tencent-cloud.com` 连接，或通过 `DescribeClusterSecurity` 获取备选 kubeconfig |

## 下一步

- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — 为超级节点 Pod 指定 CPU/内存/GPU 规格
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 使用 toleration/nodeSelector 将 Pod 调度到超级节点
- [超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) — 在超级节点上运行 DaemonSet 的配置说明

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 超级节点](https://console.cloud.tencent.com/tke2/cluster) → 新建超级节点池 → 新建超级节点。
