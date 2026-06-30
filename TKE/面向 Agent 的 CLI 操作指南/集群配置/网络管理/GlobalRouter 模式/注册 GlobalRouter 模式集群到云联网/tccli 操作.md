# 注册 GlobalRouter 模式集群到云联网（tccli）

> 对照官方：[注册 GlobalRouter 模式集群到云联网](https://cloud.tencent.com/document/product/457/35087) · page_id `35087`

## 概述

将集群容器网段注册到云联网（CCN），使容器集群与 CCN 内其他资源（跨地域 VPC、专线网关）实现互通。

**VPC-CNI 与 GlobalRouter（GR）模式的关键差异**：

| 维度 | GR 模式 | VPC-CNI 模式 |
|------|---------|-------------|
| Pod IP 来源 | TKE 独立容器网段（ClusterCIDR），由 `DescribeClusterRouteTables` 管理 | VPC 子网 IP，Pod 直接获取 VPC IP |
| 路由表 | TKE GR 路由表（`DescribeClusterRouteTables` 返回本集群路由条目） | VPC 路由表（`DescribeRouteTables`），无独立 GR 路由表 |
| CCN 注册 | 需通过 `NotifyRoutes` 将容器网段路由发布到 CCN | VPC 关联 CCN 后 Pod IP 自动可达（因 Pod IP 即为 VPC IP），无需额外路由发布 |
| 确认路由表 | `tccli tke DescribeClusterRouteTables` 返回本集群路由条目 | 同一命令返回空或仅含其他 VPC 的 GR 路由表，本 VPC 无 GR 路由表 |

云联网（CCN）是跨产品资源，使用 `tccli vpc` 命令操作。将 VPC 关联到 CCN 后，同 CCN 内的所有 VPC 可互通。

以下以 VPC-CNI 模式集群为例（`ClusterNetworkSettings.Cni=true`，Pod IP 为 VPC 子网 IP），演示操作流程并标注 GR 模式下的差异。

**操作链**：获取集群信息 → 确认网络模式（VPC-CNI / GR）→ 查询/创建 CCN → 关联 VPC → （仅 GR 模式）发布容器网段路由 → 验证路由状态

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。

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
#    vpc:CreateCcn, vpc:AttachCcnInstances, vpc:DescribeCcns
#    vpc:DescribeCcnAttachedInstances, vpc:NotifyRoutes
#    vpc:DescribeCcnRoutes, vpc:DescribeRouteTables
# 验证权限：
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

tccli vpc DescribeCcns --region <Region>
# expected: exit 0，返回 CCN 列表（可为空）
```

### 资源检查

```bash
# 4. 确认集群网络模式
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterNetworkSettings.Cni 标识网络模式，true=VPC-CNI，false=GR

# 5. 确认容器网段与 CCN 内其他网段不冲突
tccli tke DescribeRouteTableConflicts --region <Region> \
    --RouteTableCidrBlock "10.0.0.0/16" \
    --VpcId "<VpcId>"
# expected: HasConflict=false 表示无冲突
```

- VPC-CNI 模式集群：`DescribeClusters` 返回 `Cni=true`，Pod 使用 VPC 子网 IP。VPC 关联 CCN 后 Pod IP 自动可达。
- GR 模式集群：需额外确认 `DescribeClusterRouteTables` 返回本集群路由表，后续需执行 `NotifyRoutes`。
- 容器网段与 CCN 内其他资源网段不冲突。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群容器网络 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` → `ClusterCIDR`、`VpcId`、`Cni` | 是 |
| 确认路由表 | `tccli tke DescribeClusterRouteTables --region <Region>` | 是 |
| 查询已有 CCN 实例 | `tccli vpc DescribeCcns --region <Region>` | 是 |
| 新建云联网实例 | `tccli vpc CreateCcn --CcnName <CcnName> --QosLevel <QosLevel> --BandwidthLimitType <Type>` | 否 |
| 关联 VPC 到云联网 | `tccli vpc AttachCcnInstances --CcnId <CcnId> --Instances '[{"InstanceType":"VPC","InstanceId":"<VpcId>","InstanceRegion":"<Region>"}]'` | 是（重复报 `ResourceInUse`） |
| 查看 CCN 已关联实例 | `tccli vpc DescribeCcnAttachedInstances --region <Region>` | 是 |
| 查看 VPC 路由表 | `tccli vpc DescribeRouteTables --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` | 是 |
| 发布路由到云联网（仅 GR 模式） | `tccli vpc NotifyRoutes --RouteTableId <RouteTableId> --RouteItemIds '["<RouteId>"]'` | 是 |
| 查看 CCN 路由表 | `tccli vpc DescribeCcnRoutes --CcnId <CcnId> --region <Region>` | 是 |
| 启用冲突路由 | `tccli vpc EnableCcnRoutes --CcnId <CcnId> --RouteIds '["<RouteId>"]'` | 否 |

### CreateCcn API 参数参考

| 参数 | 类型 | 必填 | 说明 |
|------|------|:--:|------|
| `CcnName` | String | 是 | 云联网名称，建议含业务标识 |
| `QosLevel` | String | 否 | 服务等级：`PT`（白金）、`AU`（金）、`AG`（银）、`CU`（铜） |
| `BandwidthLimitType` | String | 否 | 带宽限制类型：`INTER_REGION_LIMIT`（地域间）、`OUTER_REGION_LIMIT`（地域出口） |
| `CcnDescription` | String | 否 | 实例描述 |
| `InstanceChargeType` | String | 否 | `PREPAID`（预付费）或 `POSTPAID`（后付费） |
| `Tags` | Array | 否 | 标签列表，每项含 `Key`、`Value` |

### AttachCcnInstances API 参数参考

| 参数 | 类型 | 必填 | 说明 |
|------|------|:--:|------|
| `CcnId` | String | 是 | 目标 CCN 实例 ID |
| `Instances` | Array | 是 | 关联实例列表，每项含 `InstanceType`（`VPC` 或 `DIRECTCONNECT`）、`InstanceId`、`InstanceRegion` |

## 操作步骤

### 步骤一：获取集群 VPC 与容器网段

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，返回 ClusterNetworkSettings 含 ClusterCIDR、VpcId、Cni
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "example-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "ServiceCIDR": "10.1.0.0/20",
        "VpcId": "<VpcId>",
        "Cni": true
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<VpcId>` | 集群所属 VPC ID | 含于 `ClusterNetworkSettings.VpcId` | 同上命令输出 |
| `<Region>` | 目标地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

记录 `ClusterCIDR`（容器网段）、`<VpcId>`（集群所属 VPC）和 `Cni` 字段（网络模式）。

- `Cni=true`：VPC-CNI 模式。Pod 使用 VPC 子网 IP，VPC 关联 CCN 后 Pod IP 自动可达，无需额外路由发布。
- `Cni=false`（或字段不存在）：GR 模式。容器网段由 TKE 独立管理，需通过 `NotifyRoutes` 发布到 CCN。

### 步骤二：确认路由表

```bash
tccli tke DescribeClusterRouteTables --region <Region>
# expected: exit 0，返回 RouteTableSet 列表
```

```json
{
  "TotalCount": 7,
  "RouteTableSet": [
    {
      "RouteTableName": "other-gr-cluster",
      "RouteTableCidrBlock": "10.200.0.0/16",
      "VpcId": "vpc-xxxxxxxx",
      "RouteTableId": "rtb-xxxxxxxx"
    }
  ],
  "RouteTablesForThisVPC": [],
  "RequestId": "<RequestId>"
}
```

> **VPC-CNI 模式**：`DescribeClusterRouteTables` 返回的 `RouteTableSet` 为其他 VPC 的 GR 集群路由表（`TotalCount=7`），本 VPC 在 `RouteTablesForThisVPC` 中为空——这是 VPC-CNI 的预期行为。VPC-CNI 模式下 Pod 直接获取 VPC 子网 IP，不需要独立的 GR 路由表。GR 模式集群才会在 `RouteTableSet` 中返回本集群的路由条目。

**常见错误**：在 VPC-CNI 集群上试图通过 `DescribeClusterRouteTables` 查找路由表并执行 `NotifyRoutes`。VPC-CNI 集群无需此操作。确认网络模式：回看步骤一的 `Cni` 字段。

### 步骤三：查询已有 CCN 实例

```bash
tccli vpc DescribeCcns --region <Region>
# expected: exit 0，返回 CcnSet 列表
```

```json
{
  "TotalCount": 21,
  "CcnSet": [
    {
      "CcnId": "ccn-xxxxxxxx",
      "CcnName": "existing-ccn",
      "CcnDescription": "Existing CCN instance",
      "State": "AVAILABLE",
      "QosLevel": "AG",
      "InstanceCount": 3,
      "CreateTime": "2025-01-01 00:00:00"
    }
  ],
  "RequestId": "<RequestId>"
}
```

若已有可复用的 CCN（需确认所有权），记录 `CcnId` 跳到步骤五；否则执行步骤四。

筛选可复用 CCN：

```bash
tccli vpc DescribeCcns --region <Region> | jq '.CcnSet[] | {CcnId, CcnName, IsAttached: (.InstanceCount > 0)}'
# 优先选择 InstanceCount=0 或业务标签匹配的空闲 CCN
```

### 步骤四：创建 CCN 实例（如无可复用）

#### 选择依据

- **QosLevel**：选择 `AG`（银），满足大多数混合云场景。金（`AU`）和白金（`PT`）适用于对延迟敏感的核心业务。
- **BandwidthLimitType**：`INTER_REGION_LIMIT`（地域间限速），按各地域间带宽独立配置，推荐默认选择。
- **CCN 配额**：CCN 配额为账户级全局限制（默认上限 20），非 per-region。切换地域创建无效。如配额已满，需提工单提升配额或删除无用 CCN。

```bash
tccli vpc CreateCcn \
    --CcnName 'tke-ccn-<Region>' \
    --QosLevel 'AG' \
    --BandwidthLimitType 'INTER_REGION_LIMIT' \
    --region <Region>
# expected: exit 0，返回 CcnId；或 LimitExceeded（配额满）
```

> **Blocked**：CCN 配额满时返回 `LimitExceeded`（CCN数量达到上限）。此为账户级全局限制（上限 20，不限 region）。实测穷尽 16 个 region 均返回同一错误（requestId: `95de01f0-772b-46e9-961d-abf39a87f0a0`）。
>
> **解决路径**：
> 1. 提工单提升 CCN 配额
> 2. 删除无用 CCN 释放配额
> 3. 复用已有 CCN（需确认所有权：`tccli vpc DescribeCcns --region <Region>` 核对 CCN 标签和归属）

以下步骤五至八的命令因 CCN 实例缺失暂未真机验证，作为参考格式保留。

### 步骤五：将集群 VPC 关联到 CCN

```bash
tccli vpc AttachCcnInstances \
    --CcnId <CcnId> \
    --Instances '[{"InstanceType":"VPC","InstanceId":"<VpcId>","InstanceRegion":"<Region>"}]' \
    --region <Region>
# expected: exit 0
```

> 重复关联同一 VPC 返回 `UnsupportedOperation.CcnAttached`，为幂等错误可忽略。

### 步骤六：验证 VPC 已关联至 CCN

```bash
tccli vpc DescribeCcnAttachedInstances --region <Region>
# expected: exit 0，返回已关联实例列表
```

```json
{
  "TotalCount": 69,
  "InstanceSet": [
    {
      "CcnId": "ccn-xxxxxxxx",
      "InstanceId": "<VpcId>",
      "InstanceRegion": "<Region>",
      "InstanceType": "VPC",
      "State": "ACTIVE"
    }
  ],
  "RequestId": "<RequestId>"
}
```

此时 CCN 路由表已自动学习 VPC 的 CIDR（`172.16.0.0/12`）。

- **VPC-CNI 模式**：操作完成。VPC 关联 CCN 后，Pod IP（即 VPC 子网 IP）自动可达，无需额外路由发布。
- **GR 模式**：容器网段（`10.0.0.0/16`）尚未在 CCN 路由表发布，需继续执行步骤七。

### 步骤七：将容器网段路由发布到 CCN（仅 GR 模式需要）

> **VPC-CNI 模式跳过此步骤**。VPC-CNI 集群的 Pod IP 即为 VPC 子网 IP，VPC 关联 CCN 后 Pod IP 自动可达。以下操作仅适用于 GR 模式集群。

先查询 VPC 路由表，确认容器网段路由条目：

```bash
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: exit 0，返回 VPC 路由表
```

```json
{
  "TotalCount": 1,
  "RouteTables": [
    {
      "RouteTableId": "<RouteTableId>",
      "RouteTableName": "default",
      "VpcId": "<VpcId>",
      "Main": true,
      "RouteCount": 0
    }
  ],
  "RequestId": "<RequestId>"
}
```

> **VPC-CNI 模式**：VPC 路由表为空（`RouteCount=0`）。Pod 使用 VPC 子网 IP，路由表中无容器网段（`ClusterCIDR`）路由。GR 模式集群的路由表中才包含容器网段对应的路由条目（`DestinationCidrBlock` 为容器网段）。

GR 模式下的路由发布命令（参考）：

```bash
tccli vpc NotifyRoutes \
    --RouteTableId <RouteTableId> \
    --RouteItemIds '["<RouteId>"]' \
    --region <Region>
# expected: exit 0，返回 RequestId
```

控制台「注册到云联网」按钮本质即调用 `NotifyRoutes`。

### 步骤八：查看 CCN 路由表容器网段状态（仅 GR 模式）

```bash
tccli vpc DescribeCcnRoutes --CcnId <CcnId> --region <Region>
# expected: exit 0，返回 CCN 路由列表
```

```json
{
  "TotalCount": 1,
  "RouteSet": [
    {
      "RouteId": "ccnr-xxxxxxxx",
      "DestinationCidrBlock": "10.0.0.0/16",
      "InstanceId": "<VpcId>",
      "InstanceRegion": "<Region>",
      "InstanceType": "VPC",
      "Enabled": true
    }
  ],
  "RequestId": "<RequestId>"
}
```

- `Enabled = true`：路由已生效，容器集群与 CCN 内其他资源可互通。
- `Enabled = false`：路由因网段冲突被禁用，需手动启用：

```bash
tccli vpc EnableCcnRoutes \
    --CcnId <CcnId> \
    --RouteIds '["<RouteId>"]' \
    --region <Region>
# expected: exit 0
```

> 启用冲突路由可能导致网络不可达，请先评估影响。路由冲突原则参见 [CCN 路由限制](https://cloud.tencent.com/document/product/877/18679)。

## 验证

### 控制面验证（tccli）

```bash
# 1. CCN 实例状态
tccli vpc DescribeCcns --region <Region> --CcnIds '["<CcnId>"]'
# expected: State = "AVAILABLE"

# 2. VPC 已关联至 CCN
tccli vpc DescribeCcnAttachedInstances \
    --Filters '[{"Name":"ccn-id","Values":["<CcnId>"]}]' \
    --region <Region>
# expected: InstanceSet 含 VpcId = "<VpcId>"，State = "ACTIVE"

# 3. VPC-CNI 模式：验证 VPC 路由表正常
tccli vpc DescribeRouteTables --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: VPC-CNI 下 RouteCount=0 为正常

# 4. （仅 GR 模式）容器网段路由在 CCN 路由表已启用
tccli vpc DescribeCcnRoutes --CcnId <CcnId> --region <Region>
# expected: RouteSet 含 DestinationCidrBlock = "10.0.0.0/16"，Enabled = true
```

## 清理

> **副作用警告**：
> - `DetachCcnInstances` 会中断通过 CCN 的所有互通，确认无业务流量后再执行。
> - CCN 解关联 VPC 前确认无其他业务依赖该 CCN 实例。
> - CCN 实例产生带宽费用（按实际流量计费），解关联 VPC 不会销毁 CCN，不使用请手动删除。测试完成后及时解关联 VPC 并删除 CCN（如自建）。

```bash
# 1. 撤销路由发布（仅 GR 模式）
tccli vpc WithdrawNotifyRoutes \
    --RouteTableId <RouteTableId> \
    --RouteItemIds '["<RouteId>"]' \
    --region <Region>

# 2. 解关联 VPC（确认无其他业务依赖）
tccli vpc DetachCcnInstances \
    --CcnId <CcnId> \
    --Instances '[{"InstanceType":"VPC","InstanceId":"<VpcId>","InstanceRegion":"<Region>"}]' \
    --region <Region>

# 3. 删除 CCN 实例（可选，解关联所有实例后执行）
tccli vpc DeleteCcn --CcnId <CcnId> --region <Region>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateCcn` 返回 `LimitExceeded` | `tccli vpc DescribeCcns --region <Region>` 确认已有 CCN 数量 | CCN 实例数达账户级上限（默认 20），非 per-region 限制。requestId 参考：`95de01f0-772b-46e9-961d-abf39a87f0a0` | 此为环境限制，非命令错误。三种修复路径：① 提工单提升 CCN 配额；② 删除无用 CCN 释放配额；③ 复用已有 CCN（需确认所有权）。切换 region 无效（配额为账户级全局限制） |
| `DescribeCcns` 返回空 | 检查地域 | 当前地域无 CCN 实例 | 执行 `CreateCcn` 创建新实例（注意配额限制） |
| `AttachCcnInstances` 返回 `UnsupportedOperation.CcnAttached` | VPC 已在 CCN 中 | 幂等冲突 | 若在目标 CCN 中则忽略；否则先 `DetachCcnInstances` |
| `AttachCcnInstances` 返回 `InvalidParameterValue.CcnIdMalformed` | CCN ID 格式错误 | CCN ID 不存在或格式不正确 | 核对 `DescribeCcns` 返回的 CCN ID |
| `NotifyRoutes` 返回 `RouteNotFound` | 路由 ID 不匹配 | `RouteItemIds` 参数错误 | 从 `DescribeRouteTables` 精确获取 `RouteId`。注意：VPC-CNI 集群无需 `NotifyRoutes` |
| `DescribeCcnRoutes` 返回空 | 容器网段未发布 | `NotifyRoutes` 未执行（仅 GR 模式） | 确认集群网络模式：VPC-CNI 跳过此步骤；GR 模式重新执行步骤七 |
| `EnableCcnRoutes` 返回 `InvalidParameterValue.RouteConflict` | 路由冲突 | 网段与其他成员重叠 | 确认冲突来源，规划新网段或移除冲突成员 |

### 状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| VPC 已关联但 CCN 路由表无容器网段 | 检查集群网络模式：`tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 查看 `ClusterNetworkSettings.Cni` | VPC-CNI 模式：无需容器网段路由（Pod IP = VPC IP）；GR 模式：未执行 `NotifyRoutes` | VPC-CNI → 正常，无需处理；GR → 执行步骤七 |
| GR 模式 CCN 路由 `Enabled = false` | 查看 CCN 路由表冲突详情 | 容器网段与已有路由重叠 | 评估后手动 `EnableCcnRoutes` 或重新规划网段 |
| 路由启用但不通 | 检查安全组 | 安全组或 ACL 阻拦 | 放行容器网段 `10.0.0.0/16` 与目标网段双向流量 |
| 间歇性丢包或高延迟 | 持续 ping 测试 | CCN 跨地域带宽不足 | 调整带宽限制或升级 QoS 等级 |
| `DescribeClusterRouteTables` 返回大量路由表但本 VPC 无 | `tccli tke DescribeClusterRouteTables --region <Region> \| jq '.RouteTablesForThisVPC'` | VPC-CNI 集群无 GR 路由表，返回的是其他 VPC 的 GR 路由表 | 正常，VPC-CNI 模式预期行为。需查询 VPC 路由表时用 `DescribeRouteTables` |
| `CreateCcn` 切换 region 仍报 `LimitExceeded` | 已在多个 region 测试均失败 | CCN 配额为账户级全局限制，非 per-region | 确认配额限制类型：切换 region 无效。参见上方 `LimitExceeded` 修复路径 |

## 下一步

- [同地域及跨地域 GlobalRouter 模式集群间互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [GlobalRouter 模式集群与 IDC 互通](../GlobalRouter%20模式集群与%20IDC%20互通/tccli%20操作.md)
- CCN 官方文档：[云联网](https://cloud.tencent.com/document/product/877)
- CCN 路由限制：[路由限制原则](https://cloud.tencent.com/document/product/877/18679)

## 控制台替代

[控制台 → 私有网络 → 云联网](https://console.cloud.tencent.com/vpc/ccn) → 管理实例 → 路由表 查看容器网段路由状态；[控制台 → VPC → 路由表](https://console.cloud.tencent.com/vpc/route) → 选择集群对应 VPC 路由表 → 路由策略 → 发布到云联网。
