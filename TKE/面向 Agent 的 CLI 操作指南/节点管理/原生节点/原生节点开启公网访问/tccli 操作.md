# 原生节点开启公网访问

> 对照官方：[原生节点开启公网访问](https://cloud.tencent.com/document/product/457/82334) · page_id `82334` · tccli ≥3.1.107.1 · API 2017-03-12

## 概述

原生节点实例在创建时默认不分配公网 IP（`PublicIpAssigned` 为 null）。如需从公网访问已有原生节点，可通过 VPC 的 `AssociateAddress` API 将弹性公网 IP（EIP）绑定到实例主网卡，实现公网访问。此方式与 CVM 实例 EIP 绑定一致，无需特殊权限。

**方案对比**：

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 绑定 EIP（本文档方案） | 已有原生节点需临时公网访问 | 即绑即用，支持解绑，无需重建节点 | 需额外管理 EIP 生命周期，闲置 EIP 有费用 |
| 创建节点池时开启公网 IP | 新节点池 | 自动分配，无需额外操作 | 仅对新节点生效，不可用于已有实例 |
| NAT 网关 | 多节点需统一出站 | 统一出口 IP，支持多节点共享 | 不提供公网入站访问（无独立公网 IP），需额外配置 |
| kubectl 注解 Machine CRD | 集群端点可达时 | 原生节点原生能力 | 需集群公网端点可达（当前集群仅内网端点，不可行） |

## 前置条件

> 以下每条均可 COPY-PASTE 验证。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterInstances, tke:DescribeClusterNodePoolDetail
#    cvm:DescribeInstances
#    vpc:DescribeAddresses, vpc:AllocateAddresses, vpc:AssociateAddress
#    vpc:DisassociateAddress, vpc:ReleaseAddresses, vpc:DescribeAddressQuota
# 验证：执行 DescribeClusterInstances 确认 TKE 权限
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回实例列表（可为空）

# 验证 VPC 权限（查询 EIP）
tccli vpc DescribeAddresses --region <Region>
# expected: exit 0，返回 EIP 列表（可为空）

# 验证 CVM 权限
tccli cvm DescribeInstances --region <Region>
# expected: exit 0，返回 CVM 列表（可为空）
```

### 资源检查

```bash
# 4. 确认原生节点实例存在且地域正确
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region>
# expected: InstanceSet 中含目标实例，InstanceState 为 running

# 5. 检查实例当前公网 IP 状态
tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' \
    --region <Region>
# expected: PublicIpAddresses 为 null 或空数组

# 6. 检查是否有可用的未绑定 EIP
tccli vpc DescribeAddresses --region <Region> \
    --Filters '[{"Name":"address-status","Values":["UNBIND"]}]'
# expected: 至少返回 1 个 UNBIND 状态 EIP，或空列表（需申请新 EIP）

# 7. 检查 EIP 配额（如需申请新 EIP）
tccli vpc DescribeAddressQuota --region <Region>
# expected: TOTAL_EIP_QUOTA 中 QuotaCurrent < QuotaLimit
```

## 控制台与 CLI 参数映射

### 操作映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群原生节点实例 | `tke DescribeClusterInstances` | 是 |
| 查看节点池详情 | `tke DescribeClusterNodePoolDetail` | 是 |
| 查看实例详情（含公网 IP） | `cvm DescribeInstances` | 是 |
| 查看可用 EIP | `vpc DescribeAddresses` | 是 |
| 检查 EIP 配额 | `vpc DescribeAddressQuota` | 是 |
| 申请 EIP | `vpc AllocateAddresses` | 否 |
| 绑定 EIP 到实例 | `vpc AssociateAddress` | 否 |
| 解绑 EIP | `vpc DisassociateAddress` | 否 |
| 释放 EIP | `vpc ReleaseAddresses` | 是 |

### `AllocateAddresses` 参数说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddressCount` | Integer | 是 | 申请数量，通常为 `1` | 填 0 → `InvalidParameterValue` |
| `InternetChargeType` | String | 是 | `TRAFFIC_POSTPAID_BY_HOUR`（按流量，推荐）、`BANDWIDTH_POSTPAID_BY_HOUR`（按带宽）、`BANDWIDTH_PREPAID_BY_MONTH`（包月带宽） | 填无效值 → `InvalidParameterValue.InternetChargeType` |
| `InternetMaxBandwidthOut` | Integer | 否 | 带宽上限（Mbps），默认 `1`。建议根据业务设置（如 `100`） | 设过低 → 公网访问带宽受限 |
| `AddressName` | String | 否 | EIP 名称，建议含业务标识便于识别 | — |
| `InternetServiceProvider` | String | 否 | `BGP`（默认）、`CMCC`、`CTCC`、`CUCC` | 指定不可用线路 → `InvalidParameterValue.InternetServiceProvider` |
| `Tags` | Array | 否 | 标签列表，格式 `[{"Key":"env","Value":"test"}]` | — |

### `AssociateAddress` 参数说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddressId` | String | 是 | EIP 实例 ID，格式 `eip-xxxxxxxx`。必须为 `UNBIND` 状态 | 已绑定 → `InvalidParameterValue.AddressAlreadyBind`；不存在 → `InvalidParameterValue.AddressIdNotFound` |
| `InstanceId` | String | 是* | CVM/原生节点实例 ID，格式 `ins-xxxxxxxx` | 实例不存在或不属于当前地域 → `InvalidInstanceId.NotFound` |
| `NetworkInterfaceId` | String | 否 | 弹性网卡 ID，格式 `eni-xxxxxxxx`。不指定则绑定实例主网卡 | 网卡不属于目标实例 → `InvalidParameterValue.NetworkInterfaceId` |
| `PrivateIpAddress` | String | 否 | 内网 IP，不指定则自动分配 | IP 不属于目标网卡 → `InvalidParameterValue.PrivateIpAddress` |

> \* `InstanceId` 与 `NetworkInterfaceId` 二选一必填。绑定原生节点实例时使用 `InstanceId`。

### `DisassociateAddress` 参数说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddressId` | String | 是 | 待解绑的 EIP ID，必须当前为 `BIND` 状态 | 未绑定 → `InvalidParameterValue.AddressIdNotBound` |
| `ReallocateNormalPublicIp` | Boolean | 否 | 是否重新分配普通公网 IP。默认 `false` | — |

## 操作步骤

### 步骤 1：查看原生节点实例

确认目标原生节点实例的 ID、状态和网络信息。

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回 InstanceSet，含 InstanceId 和 InstanceRole
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example",
            "AutoscalingGroupId": "asg-example",
            "CreatedTime": "2026-06-23T08:13:42Z"
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置 region |

> 原生节点实例在创建时通常不分配公网 IP，输出中无 `PublicIp` 字段，需通过 EIP 绑定开启公网访问。

#### 补充：检查节点池网络注解

通过 `Annotations` 可配置原生节点网络参数（如 `node.tke.cloud.tencent.com/internet-max-bandwidth-out`），确认当前节点池是否已配置公网注解。

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: 返回节点池详情，检查 Annotations 中是否含公网相关注解
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "my-node-pool",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "MaxNodesNum": 1,
        "MinNodesNum": 1,
        "DesiredNodesNum": 1,
        "NodeCountSummary": {
            "ManuallyAdded": { "Total": 0 },
            "AutoscalingAdded": { "Total": 1 }
        },
        "Annotations": []
    }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | 步骤 1 输出中的 `NodePoolId` |

> 当前节点池 `Annotations` 为空（无网络相关注解），确认需通过 EIP 绑定方式开启公网访问。若节点池有 `node.tke.cloud.tencent.com/internet-max-bandwidth-out` 等注解，可通过修改注解实现公网配置（需集群端点可达）。

### 步骤 2：确认实例无公网 IP

通过 CVM API 确认实例当前无公网访问能力。

```bash
tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' \
    --region <Region>
# expected: PublicIpAddresses 为 null，InternetMaxBandwidthOut 为 0
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceState": "RUNNING",
            "InstanceType": "S5.MEDIUM2",
            "PublicIpAddresses": null,
            "InternetAccessible": {
                "InternetChargeType": null,
                "InternetMaxBandwidthOut": 0,
                "PublicIpAssigned": null
            },
            "PrivateIpAddresses": ["172.24.0.34"]
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | 原生节点实例 ID | 格式 `ins-xxxxxxxx` | 步骤 1 输出中的 `InstanceId` |

### 步骤 3：查询可用的未绑定 EIP

列出所有未绑定状态的 EIP，选择可用的直接绑定。

```bash
tccli vpc DescribeAddresses --region <Region> \
    --Filters '[{"Name":"address-status","Values":["UNBIND"]}]'
# expected: 返回 UNBIND 状态的 EIP 列表
```

**预期输出**：

```json
{
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressIp": "159.75.114.174",
            "AddressStatus": "UNBIND",
            "InstanceId": null,
            "AddressName": "my-eip",
            "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR"
        }
    ]
}
```

若返回空列表，无可用 EIP，进入步骤 4 创建新 EIP。若有可用 EIP，记下 `AddressId`，跳过步骤 4 直接进入步骤 5。

### 步骤 4：创建弹性公网 IP（无可用 EIP 时）

当无可用的未绑定 EIP 时，需先申请一个新的 EIP。

#### 选择依据

- **计费模式**：选择 `TRAFFIC_POSTPAID_BY_HOUR`（按流量后付费）。无月租，适合测试和低流量场景。如需固定带宽的稳定大流量场景，可选 `BANDWIDTH_POSTPAID_BY_HOUR`。
- **带宽上限**：建议设置为 `100` Mbps，满足常见运维场景（SSH、kubectl 运维操作）。可根据实际业务调整。
- **线路类型**：默认 `BGP`，覆盖三网，延迟最优。特殊需求可选单线（CMCC/CTCC/CUCC）。

> 如已有可用未绑定 EIP（步骤 3 输出非空），跳过此步骤，直接进入步骤 5。

#### 最小创建（只含必填字段）

```bash
tccli vpc AllocateAddresses --region <Region> \
    --AddressCount 1 \
    --InternetChargeType TRAFFIC_POSTPAID_BY_HOUR
# expected: exit 0，返回 AddressSet 含 AddressId 和 AddressIp
```

**预期输出**：

```json
{
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressIp": "x.x.x.x"
        }
    ]
}
```

#### 增强配置（加带宽上限、名称、标签）

```bash
tccli vpc AllocateAddresses --region <Region> \
    --AddressCount 1 \
    --InternetChargeType TRAFFIC_POSTPAID_BY_HOUR \
    --InternetMaxBandwidthOut 100 \
    --AddressName <EipName> \
    --Tags '[{"Key":"env","Value":"test"}]'
# expected: exit 0，返回 AddressSet
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<EipName>` | EIP 名称 | 建议含业务标识 | 自定义，如 `my-node-eip` |

### 步骤 5：绑定 EIP 到原生节点实例

#### 选择依据

- **绑定方式**：通过 VPC `AssociateAddress` API 绑定 EIP 到实例。原生节点实例基于 CXM，创建时默认不分配公网 IP。使用 `AssociateAddress` 可将独立 EIP 绑定到实例主网卡，此方式与 CVM 实例 EIP 绑定完全一致，无需特殊权限。
- **替代方案排除**：
  - 创建节点池时设置 `InternetAccessible.PublicIpAssigned=true`：仅对新节点生效，不适用于已有实例。
  - NAT 网关：不提供公网入站能力（无独立公网 IP 直通实例）。
  - kubectl 修改 Machine CRD 注解：需集群公网端点可达，当前仅内网端点可用。

#### 最小绑定（AddressId + InstanceId）

```bash
tccli vpc AssociateAddress --region <Region> \
    --AddressId <AddressId> \
    --InstanceId <InstanceId>
# expected: exit 0，返回 TaskId
```

**预期输出**：

```json
{
    "TaskId": "218058980",
    "RequestId": "ae341465-c9ec-448e-9dfc-d5d63422821a"
}
```

#### 增强绑定（指定网卡和内网 IP）

如实例有多网卡，可指定绑定到特定网卡和 IP：

```bash
tccli vpc AssociateAddress --region <Region> \
    --AddressId <AddressId> \
    --InstanceId <InstanceId> \
    --NetworkInterfaceId <NetworkInterfaceId> \
    --PrivateIpAddress <PrivateIpAddress>
# expected: exit 0，返回 TaskId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<AddressId>` | EIP ID | 格式 `eip-xxxxxxxx`，必须 UNBIND 状态 | 步骤 3 或步骤 4 输出中的 `AddressId` |
| `<InstanceId>` | 原生节点实例 ID | 格式 `ins-xxxxxxxx` | 步骤 1 输出中的 `InstanceId` |
| `<NetworkInterfaceId>` | 弹性网卡 ID（可选） | 格式 `eni-xxxxxxxx` | `tccli vpc DescribeNetworkInterfaces --region <Region>` |
| `<PrivateIpAddress>` | 内网 IP（可选） | 实例内网 IP | 步骤 1 输出中的 `LanIP` |

> `AssociateAddress` 是异步操作，绑定通常在秒级完成。返回 `TaskId` 即表示请求已提交。

## 验证

### 控制面（tccli）

#### 验证 1：EIP 状态已变为 BIND

```bash
tccli vpc DescribeAddresses --AddressIds '["<AddressId>"]' \
    --region <Region>
# expected: AddressStatus 为 "BIND"，InstanceId 为目标实例 ID
```

**预期输出**：

```json
{
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressIp": "159.75.114.174",
            "AddressStatus": "BIND",
            "InstanceId": "ins-example",
            "NetworkInterfaceId": "eni-example",
            "PrivateAddressIp": "172.24.0.34"
        }
    ]
}
```

关键字段验证：
- `AddressStatus` 从 `UNBIND` 变为 `BIND`
- `InstanceId` 显示为目标实例 ID
- `NetworkInterfaceId` 指示 EIP 绑定到实例主网卡

#### 验证 2：实例已分配公网 IP

```bash
tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' \
    --region <Region>
# expected: PublicIpAddresses 非 null，InternetMaxBandwidthOut > 0
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "PublicIpAddresses": ["159.75.114.174"],
            "InternetAccessible": {
                "InternetMaxBandwidthOut": 1
            },
            "PrivateIpAddresses": ["172.24.0.34"]
        }
    ]
}
```

对比步骤 2 的输出：
- `PublicIpAddresses`：`null` → `["159.75.114.174"]`
- `InternetMaxBandwidthOut`：`0` → `1`

### 数据面

> 数据面验证（如通过公网 IP SSH 登录节点、`kubectl` 通过节点公网 IP 访问集群等）依赖集群公网端点可达或节点安全组已放行 SSH 端口（22）。当前环境仅内网端点可达，数据面验证需在有公网访问条件的环境执行。

## 清理

> **警告**：
> - `DisassociateAddress` 只解绑 EIP，不释放 EIP 资源。解绑后的 EIP 仍会产生闲置 IP 资源费用。
> - 解绑 EIP 前确认无业务流量依赖该公网 IP，解绑后立即生效，可能导致正在进行的连接中断。
> - EIP 按小时计费，即使未绑定也可能产生 IP 资源费用（`IPChargeType=IP_POSTPAID_BY_HOUR` 时）。
> - 如需彻底释放 EIP，解绑后执行 `tccli vpc ReleaseAddresses --AddressIds '["<AddressId>"]' --region <Region>`。

### 控制面（tccli）

#### 1. 清理前状态检查

```bash
tccli vpc DescribeAddresses --AddressIds '["<AddressId>"]' \
    --region <Region>
# 确认 AddressStatus 为 BIND，InstanceId 为目标实例，记录 AddressId
```

#### 2. 解绑 EIP

```bash
tccli vpc DisassociateAddress --AddressId <AddressId> \
    --region <Region>
# expected: exit 0，返回 TaskId
```

**预期输出**：

```json
{
    "TaskId": "218059008",
    "RequestId": "f2600371-11c4-4273-9135-4a50b0e3f23d"
}
```

> 解绑后 EIP 状态恢复为 `UNBIND`，可重新绑定到其他实例。

#### 3. 验证 EIP 已解绑

```bash
tccli vpc DescribeAddresses --AddressIds '["<AddressId>"]' \
    --region <Region>
# expected: AddressStatus 为 "UNBIND"，InstanceId 为 null
```

**预期输出**：

```json
{
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressIp": "159.75.114.174",
            "AddressStatus": "UNBIND",
            "InstanceId": null
        }
    ]
}
```

#### 4. 确认实例已无公网 IP

```bash
tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' \
    --region <Region>
# expected: PublicIpAddresses 为 null
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "PublicIpAddresses": null
        }
    ]
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AssociateAddress` 返回 `InvalidParameterValue.AddressIdNotFound` | `tccli vpc DescribeAddresses --AddressIds '["<AddressId>"]' --region <Region>` 确认 EIP 是否存在 | EIP 不存在或属于其他地域 | 确认 EIP ID 和地域正确：`tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' --region <Region>` 确认实例存在且属于该地域 |
| `AssociateAddress` 返回 `InvalidParameterValue.AddressAlreadyBind` | `tccli vpc DescribeAddresses --AddressIds '["<AddressId>"]' --region <Region>` 检查 `AddressStatus` | EIP 已绑定到其他实例 | 先执行 `DisassociateAddress --AddressId <AddressId>` 解绑，或使用其他 `UNBIND` 状态的 EIP |
| `AssociateAddress` 返回 `InvalidInstanceId.NotFound` | `tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' --region <Region>` 确认实例存在 | 实例 ID 或地域不正确 | 确认实例 ID 和地域正确，实例必须在 EIP 同一地域 |
| `AllocateAddresses` 返回 `LimitExceeded.AddressQuotaLimitExceeded` | `tccli vpc DescribeAddressQuota --region <Region>` 检查配额 | EIP 总数已达账户配额上限（此为环境限制，非命令错误） | 解绑并释放不再使用的 EIP：`tccli vpc ReleaseAddresses --AddressIds '["<AddressId>"]' --region <Region>`，或提交工单申请配额提升 |
| `DescribeAddresses` 返回空且 `AllocateAddresses` 返回 `LimitExceeded.DailyApply` | `tccli vpc DescribeAddressQuota --region <Region>` 检查当日申请配额 | 当日 EIP 申请数已达上限（`DAILY_EIP_APPLY`） | 等待次日重置，或使用已有未绑定 EIP |

### 绑定成功但实例仍无法从公网访问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| EIP 已绑定（`AddressStatus: BIND`）但无法 SSH/连接 | `tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]' --region <Region>` 确认 `PublicIpAddresses` 已更新 | 安全组未放行入站流量 | 检查实例关联安全组的入站规则，放行所需端口（如 TCP 22）：`tccli vpc DescribeSecurityGroupPolicies --SecurityGroupId <SecurityGroupId> --region <Region>` |
| EIP 已绑定但 `PublicIpAddresses` 仍为 null | 等待 30 秒后重新执行验证命令 | EIP 绑定异步生效中，需等待数秒 | 等待后重试验证；若超过 1 分钟仍为 null，保留 `AddressId`、`InstanceId`、`RequestId` → 登录 [VPC 控制台](https://console.cloud.tencent.com/vpc/eip) 查看详细状态 |

## 下一步

- [创建集群](../../../集群配置/集群管理/创建集群/tccli%20操作.md)
- [新建原生节点](../新建原生节点/tccli%20操作.md)
- [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)
- [弹性公网 IP 官方文档](https://cloud.tencent.com/document/product/1199)
- [VPC API 文档 — AssociateAddress](https://cloud.tencent.com/document/api/215/16700)

## 控制台替代

> 登录 [VPC 控制台 — 弹性公网 IP](https://console.cloud.tencent.com/vpc/eip)，选择 EIP → 更多 → 绑定，选择目标 CVM/原生节点实例完成绑定。
