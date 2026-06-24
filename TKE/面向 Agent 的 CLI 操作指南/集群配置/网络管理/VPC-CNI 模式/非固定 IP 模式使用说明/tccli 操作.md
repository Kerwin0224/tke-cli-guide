# 非固定 IP 模式使用说明（tccli）

> 对照官方：[非固定 IP 模式使用说明](https://cloud.tencent.com/document/product/457/64940) · page_id `64940`

## 概述

VPC-CNI 非固定 IP 模式：Pod 使用 VPC 子网 IP，但 Pod 重建后 IP 会重新分配（不保留），适用于无状态多副本服务、离线批处理等不依赖固定 IP 的场景。开启 VPC-CNI 时通过 `EnableStaticIp=false`（或设置 `IsNonStaticIpMode=true` 混部）启用。非固定 IP 模式支持预绑定机制（IP 预热），可加速 Pod 启动。

### 非固定 IP vs 固定 IP

| 维度 | 非固定 IP | 固定 IP |
|------|:--:|:--:|
| Pod IP 跨重建保持 | 否 | 是 |
| IP 分配机制 | 按需从预热池分配，释放后归还 | 绑定到 Pod 生命周期 |
| 子网要求 | 无限制 | Pod 只能调度到同子网的节点 |
| 适用场景 | 无状态服务、批处理 | 有状态服务（DB、MQ） |
| IP 预热 | 支持（min/max 预绑定） | 不适用 |

### VPC-CNI 模式对比

| 维度 | tke-route-eni（共享网卡） | tke-direct-eni（独占网卡） |
|------|------|------|
| ENI 归属 | ENI 与节点共享 | 每个 Pod 独占一个 ENI |
| Pod 密度 | 较高（S5.MEDIUM2 最多 27 Pod） | 较低（S5.MEDIUM2 最多 9 Pod） |
| 适用场景 | 大多数应用（默认模式） | 需要独立安全组的 Pod |
| IP 分配 | 从 ENI 分配辅助 IP 给 Pod | 每个 Pod 绑定一个独立 ENI |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeEnableVpcCniProgress, tke:DescribeAddon
#    tke:DescribeVpcCniPodLimits, tke:EnableVpcCniNetworkType, tke:AddVpcCniSubnets
#    vpc:DescribeSubnets, vpc:CreateSubnet, vpc:DeleteSubnet
#    验证
tccli tke DescribeClusters --region REGION
# expected: exit 0，返回集群列表
```

### 资源检查

```bash
# 4. 查询目标集群 VPC-CNI 状态
tccli tke DescribeClusters --region REGION --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {Cni, VpcId, Subnets}'
# expected: Cni=true 表示已开启 VPC-CNI

# 5. 确认 VPC-CNI 启用进度
tccli tke DescribeEnableVpcCniProgress --region REGION \
    --ClusterId CLUSTER_ID
# expected: Status "Running"

# 6. 查看 eniipamd 组件版本
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd \
    | jq '{AddonName, AddonVersion, Phase}'
# expected: AddonVersion 如 "3.11.0"，Phase "Running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群 VPC-CNI 配置 | `DescribeClusters` | 是 |
| 查看 VPC-CNI 启用进度 | `DescribeEnableVpcCniProgress` | 是 |
| 开启 VPC-CNI（非固定 IP） | `EnableVpcCniNetworkType` | 部分 |
| 查看 eniipamd 组件 | `DescribeAddon` | 是 |
| 查询 Pod IP 上限 | `DescribeVpcCniPodLimits` | 是 |
| 添加 VPC-CNI 子网 | `AddVpcCniSubnets` | 否 |
| 查看 IP 预热配置 | 数据面 `kubectl get nec` | 是 |
| 修改预热值 | 数据面 `kubectl annotate nec` | 否 |
| 开启快释放 | 控制台组件管理 / 数据面 `kubectl edit ds tke-eni-agent` | 否 |

## 关键字段说明

### EnableVpcCniNetworkType 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound` |
| `VpcCniType` | String | 是 | `tke-route-eni`（共享网卡）或 `tke-direct-eni`（独占网卡）。默认推荐共享网卡 | 填错枚举 → `InvalidParameter.VpcCniType` |
| `EnableStaticIp` | Boolean | 是 | `false` 表示非固定 IP 模式 | 设为 `true` 则开启固定 IP，不支持预热 |
| `Subnets` | Array | 是 | VPC-CNI 容器网络使用的子网 ID 列表。须与集群节点在同一 VPC | 子网不存在 → `InvalidParameter.SubnetId` |
| `IsNonStaticIpMode` | Boolean | 否 | `true` 时允许非固定 IP Pod 与固定 IP 混部 | 混部模式下 IP 管理复杂度增加 |

### DescribeVpcCniPodLimits 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Zone` | String | 是 | 可用区，如 `ap-guangzhou-6` | 可用区不支持 → 返回空列表 |
| `InstanceFamily` | String | 是 | 实例机型族，如 `S5` | 机型族不存在 → 返回空列表 |
| `InstanceType` | String | 是 | 实例规格，如 `S5.MEDIUM2` | 规格不存在 → 返回空列表 |

### AddVpcCniSubnets 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound` |
| `SubnetIds` | Array | 是 | 要添加的子网 ID 列表，须与集群在同一 VPC | 子网不在同一 VPC → `InvalidParameter` |
| `VpcId` | String | 是 | VPC ID，格式 `vpc-xxxxxxxx` | VPC ID 不匹配 → `InvalidParameter.VpcId` |
| `SkipAddingNonMasqueradeCIDRs` | Boolean | 否 | 是否跳过添加非伪装 CIDR，默认 `false` | — |

### 预热相关注解

| 注解 / 参数 | 位置 | 作用 | 默认值 |
|------------|------|------|--------|
| `tke.cloud.tencent.com/route-eni-ip-min-warm-target` | NEC | 共享网卡最小预绑定 IP 数 | 5 |
| `tke.cloud.tencent.com/route-eni-ip-max-warm-target` | NEC | 共享网卡最大预绑定 IP 数 | 5 |
| `tke.cloud.tencent.com/direct-eni-min-warm-target` | NEC | 独占网卡最小预绑定 ENI 数 | 1 |
| `tke.cloud.tencent.com/direct-eni-max-warm-target` | NEC | 独占网卡最大预绑定 ENI 数 | 1 |
| `--enable-quick-release` | tke-eni-agent args | 快释放开关（每 2 分钟释放所有多余 ENI/IP） | 关闭（慢释放） |

## 操作步骤

### IP 地址管理原理

TKE 组件在每个节点上维护弹性伸缩的 ENI/IP 预热池：

| 条件 | 行为 |
|------|------|
| 已绑定数 < 当前 Pod 数 + min 预绑定 | 绑定 ENI/IP，直到 = Pod 数 + min 预绑定 |
| 已绑定数 > 当前 Pod 数 + max 预绑定 | 每 ~2 分钟释放 1 个多余 ENI/IP（慢释放），直到 = Pod 数 + max 预绑定 |
| 最大可绑定数 < 当前已绑定数 | 立即释放多余闲置 ENI/IP |

### 步骤 1：查看集群 VPC-CNI 配置

在操作前先了解集群当前 VPC-CNI 网络配置和信息。

```bash
tccli tke DescribeClusters --region REGION --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回集群信息和 VPC-CNI 网络配置
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterName": "CLUSTER_NAME",
      "ClusterVersion": "1.32.2",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "VPC_ID",
        "Cni": true,
        "ServiceCIDR": "10.1.0.0/20",
        "Subnets": ["SUBNET_ID_2"],
        "SubnetId": "SUBNET_ID_1",
        "DataPlaneV2": false
      },
      "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"GR\",\"VpcCniType\":\"tke-route-eni\",\"IsSupportMultiENI\":true,\"IsNetworkWithApp\":true}",
      "ContainerRuntime": "containerd",
      "DeletionProtection": false,
      "ClusterLevel": "L5"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**关键字段解读**：

| 字段 | 含义 | 本例值 |
|------|------|--------|
| `ClusterNetworkSettings.Cni` | VPC-CNI 是否已启用 | `true` |
| `ClusterNetworkSettings.Subnets` | VPC-CNI 容器网络使用的子网列表 | `["SUBNET_ID_2"]` |
| `ClusterNetworkSettings.SubnetId` | 节点所在子网 | `SUBNET_ID_1` |
| `Property.VpcCniType` | VPC-CNI 模式（需从 JSON 字符串解析） | `tke-route-eni`（共享网卡） |
| `ClusterNetworkSettings.ClusterCIDR` | Pod 容器网段（非 VPC-CNI Pod 使用） | `10.0.0.0/16` |
| `ClusterNetworkSettings.ServiceCIDR` | Service 网段 | `10.1.0.0/20` |

### 步骤 2：确认 VPC-CNI 启用状态

在操作 VPC-CNI 相关配置前，确认 VPC-CNI 已完成初始化。

```bash
tccli tke DescribeEnableVpcCniProgress --region REGION \
    --ClusterId CLUSTER_ID
# expected: exit 0，Status "Running"
```

**预期输出**：

```json
{
  "Status": "Running",
  "ErrorMessage": "",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 状态 | 含义 | 下一步 |
|------|------|--------|
| `Running` | VPC-CNI 正常运行 | 可进行后续操作 |
| `Enabling` | VPC-CNI 开启中 | 等待 Status 变为 `Running`（通常 2-5 分钟） |
| `Disabling` | VPC-CNI 关闭中 | 等待关闭完成后再操作 |

### 步骤 3：查看 eniipamd 组件版本

eniipamd 是 VPC-CNI 的核心组件，负责 ENI 和 IP 的分配管理。确认组件版本和运行状态。

```bash
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd
# expected: exit 0，AddonVersion ≥ 3.2.0
```

**预期输出**：

```json
{
  "Addons": [
    {
      "AddonName": "eniipamd",
      "AddonVersion": "3.11.0",
      "Phase": "Upgrading",
      "Reason": "",
      "CreateTime": "2026-06-23T08:11:19Z"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> ⚠️ **注意**：如果 `Phase` 为 `Upgrading`（升级中），eniipamd 组件在此期间不可用，VPC-CNI 功能（Pod IP 分配、NEC 管理等）将无法正常工作。需等待 Phase 变为 `Running` 后再操作。若长时间处于 Upgrading，参见 [排障](#排障)。

### 步骤 4：查询 Pod IP 上限

在部署应用前，确认目标机型在不同 VPC-CNI 模式下支持的 Pod IP 上限，以便规划节点容量。

#### 选择依据

- **共享网卡非固定 IP**（`TKERouteENINonStaticIP`）：最常用的模式，Pod IP 不保留。S5.MEDIUM2 最多 27 个 Pod，适用于大多数无状态应用。
- **独占网卡**（`TKEDirectENI`）：每个 Pod 独占一个 ENI，Pod 密度低（S5.MEDIUM2 仅 9 个），适用于需要独立安全组的 Pod。
- **子 ENI 模式**（`TKESubENI`）：Pod 密度最高（最高 100），需要特定机型支持。

```bash
# 查询 S5.MEDIUM2 的 Pod IP 上限
tccli tke DescribeVpcCniPodLimits --region REGION \
    --Zone ap-guangzhou-6 --InstanceFamily S5 --InstanceType S5.MEDIUM2
# expected: exit 0，返回各模式下的 Pod 上限
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "PodLimitsInstanceSet": [
    {
      "Zone": "ap-guangzhou-6",
      "InstanceFamily": "S5",
      "InstanceType": "S5.MEDIUM2",
      "PodLimits": {
        "TKERouteENINonStaticIP": 27,
        "TKERouteENIStaticIP": 27,
        "TKEDirectENI": 9,
        "TKESubENI": 100
      }
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 查询 S5.LARGE8 的 Pod IP 上限（对比）
tccli tke DescribeVpcCniPodLimits --region REGION \
    --Zone ap-guangzhou-6 --InstanceFamily S5 --InstanceType S5.LARGE8
# expected: exit 0，较大规格支持的 Pod 数更多
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "PodLimitsInstanceSet": [
    {
      "Zone": "ap-guangzhou-6",
      "InstanceFamily": "S5",
      "InstanceType": "S5.LARGE8",
      "PodLimits": {
        "TKERouteENINonStaticIP": 27,
        "TKERouteENIStaticIP": 27,
        "TKEDirectENI": 19,
        "TKESubENI": 100
      }
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**Pod 上限对比**：

| 机型 | TKERouteENINonStaticIP | TKERouteENIStaticIP | TKEDirectENI | TKESubENI |
|------|:--:|:--:|:--:|:--:|
| S5.MEDIUM2（2C2G） | 27 | 27 | 9 | 100 |
| S5.LARGE8（4C8G） | 27 | 27 | 19 | 100 |

| 字段 | 含义 |
|------|------|
| `TKERouteENINonStaticIP` | 共享网卡非固定 IP 模式下的 Pod 上限（1 个 ENI + N 个辅助 IP） |
| `TKERouteENIStaticIP` | 共享网卡固定 IP 模式下的 Pod 上限 |
| `TKEDirectENI` | 独占网卡模式下的 Pod 上限（每个 Pod 占 1 个 ENI） |
| `TKESubENI` | 子 ENI 模式下的 Pod 上限（需机型支持，弹性网卡子接口） |

### 步骤 5：动态扩展 VPC-CNI 子网

当现有 VPC-CNI 子网 IP 不足时，可创建新子网并通过 `AddVpcCniSubnets` 注册到容器网络，扩展可用 IP 池。

> ⚠️ **关键警告**：子网一旦通过 `AddVpcCniSubnets` 注册为 VPC-CNI 容器网络后，**不可直接通过 `vpc:DeleteSubnet` 删除**。直接删除会留下悬空引用（VPC-CNI 仍引用已删除的子网），导致 Pod 调度到该子网时 IP 分配失败。必须先禁用 VPC-CNI（`DisableVpcCniNetworkType`，会重建所有 Pod）或确保子网不再被使用后再删除。

#### 选择依据

- **为什么需要扩展子网**：当现有 VPC-CNI 子网的可用 IP 耗尽时，新 Pod 将因无法分配 IP 而一直 Pending。扩展子网是最小影响的解决方案（无需重建现有 Pod）。
- **为什么不能直接删子网**：`AddVpcCniSubnets` 将子网注册到 VPC-CNI 容器网络后，该子网被 TKE 管控面管理。直接调用 `vpc:DeleteSubnet` 不会自动从 VPC-CNI 注销，造成悬空引用。
- **子网规划**：新子网的 CIDR 必须在 VPC 网段内，且不与已有子网及 VPC-CNI 子网重叠。

#### 5.1 创建新子网

```bash
tccli vpc CreateSubnet --region REGION \
    --VpcId VPC_ID \
    --SubnetName SUBNET_NAME \
    --CidrBlock CIDR_BLOCK \
    --Zone ZONE
# expected: exit 0，返回子网 ID 和可用 IP 数
```

**预期输出**：

```json
{
  "Subnet": {
    "SubnetId": "SUBNET_ID_NEW",
    "SubnetName": "SUBNET_NAME",
    "CidrBlock": "172.28.0.0/28",
    "Zone": "ap-guangzhou-3",
    "VpcId": "VPC_ID",
    "AvailableIpAddressCount": 13,
    "TotalIpAddressCount": 13
  },
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 5.2 将子网添加到 VPC-CNI 容器网络

```bash
tccli tke AddVpcCniSubnets --region REGION \
    --ClusterId CLUSTER_ID \
    --VpcId VPC_ID \
    --SubnetIds '["SUBNET_ID_NEW"]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 5.3 验证子网已加入

```bash
tccli tke DescribeClusters --region REGION \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Subnets'
# expected: 列表中包含新添加的子网 ID
```

**预期输出**：

```json
["SUBNET_ID_2", "SUBNET_ID_NEW"]
```

### 步骤 6：开启 VPC-CNI 非固定 IP 模式

在新建集群时，可通过 `EnableVpcCniNetworkType` 开启 VPC-CNI 非固定 IP 模式。

#### 选择依据

- **VpcCniType: tke-route-eni（共享网卡）**：默认模式，ENI 与节点共享，每个 ENI 可分配多个辅助 IP 给 Pod。Pod 密度高（S5.MEDIUM2 最多 27 Pod），适用于绝大多数场景。
- **不在生产集群重复执行**：`EnableVpcCniNetworkType` 是集群级一次性配置。如果集群已开启 VPC-CNI（`Cni=true`），不要再次调用此接口，否则会触发重复初始化。
- **EnableStaticIp=false**：非固定 IP 模式下 Pod IP 不保留，适合无状态服务。可叠加 `IsNonStaticIpMode=true` 实现混部（同一集群同时支持固定 IP 和非固定 IP Pod）。

`enable-nonstatic-ip.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "VpcCniType": "tke-route-eni",
  "EnableStaticIp": false,
  "Subnets": ["SUBNET_ID_1", "SUBNET_ID_2"]
}
```

```bash
tccli tke EnableVpcCniNetworkType --region REGION --cli-input-json file://enable-nonstatic-ip.json
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
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region REGION` |
| `SUBNET_ID_1` | 子网 ID（节点子网） | 须与集群节点在同一 VPC | `tccli vpc DescribeSubnets --region REGION --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'` |
| `SUBNET_ID_2` | 子网 ID（VPC-CNI 容器网络子网） | 须与集群在同一 VPC，有足够可用 IP | 同上 |

### 步骤 7：配置节点预热（数据面，须 IOA/VPN/专线 或公网端点）

> ⚠️ 以下 kubectl 命令需集群端点可达。内网端点（`ClusterIntranetEndpoint`）需通过 IOA/VPN/专线 连接 VPC；公网端点（`ClusterExternalEndpoint`）需确认 CAM 策略允许创建（详见 [排障](#端点连接问题)）。

```bash
# 查看当前节点预热配置
kubectl get nec NODE_NAME -o yaml | yq '.metadata.annotations'
# expected: 显示 route-eni-ip-min-warm-target 等注解
```

```text
NAME         STATUS    AGE
NODE_NAME    Active    2d
```

```bash
# 修改预热值（须成对设置，0 <= min <= max）
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/route-eni-ip-min-warm-target"="3" --overwrite
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/route-eni-ip-max-warm-target"="8" --overwrite
# expected: nodeeniconfig.networking.tke.cloud.tencent.com/NODE_NAME annotated
```

NEC 注解含义：

| 注解 | 含义 | 约束 |
|------|------|------|
| `route-eni-ip-min-warm-target` | 共享网卡最小预绑定 IP 数 | 0 ≤ min ≤ max |
| `route-eni-ip-max-warm-target` | 共享网卡最大预绑定 IP 数 | ≥ min |
| `direct-eni-min-warm-target` | 独占网卡最小预绑定 ENI 数 | 0 ≤ min ≤ max |
| `direct-eni-max-warm-target` | 独占网卡最大预绑定 ENI 数 | ≥ min |

### 步骤 8：指定节点最大绑定数（数据面，须 IOA/VPN/专线 或公网端点）

```bash
# 共享网卡：最大 ENI 数
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/route-eni-max-attach"="4" --overwrite

# 共享网卡：单 ENI 最大 IP 数
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/max-ip-per-route-eni"="20" --overwrite

# 独占网卡：最大 ENI 数
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/direct-eni-max-attach"="5" --overwrite
# expected: 值 ≥ 当前已用 ENI/IP 数量，否则修改失败
```

### 步骤 9：启用快释放（数据面，须 IOA/VPN/专线 或公网端点）

默认慢释放每 2 分钟仅释放 1 个多余 ENI/IP。快释放每 2 分钟释放所有多余 ENI/IP。

- **≥ v3.5.0**：控制台 → 组件管理 → eniipamd → 更新配置 → 勾选「快释放」。
- **< v3.5.0**：编辑 DaemonSet 添加启动参数：

```bash
kubectl edit ds tke-eni-agent -n kube-system
# 在 containers[0].args 中添加：--enable-quick-release
# 保存后等待 DaemonSet 滚动更新完成
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认 VPC-CNI 非固定 IP 状态
tccli tke DescribeClusters --region REGION \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.ClusterNetworkSettings | {Cni, Subnets}'
# expected: Cni=true, Subnets 包含所有 VPC-CNI 子网
```

**预期输出**：

```json
{
  "Cni": true,
  "Subnets": ["SUBNET_ID_2", "SUBNET_ID_NEW"]
}
```

```bash
# 2. 确认 VPC-CNI 启用进度
tccli tke DescribeEnableVpcCniProgress --region REGION \
    --ClusterId CLUSTER_ID
# expected: Status "Running"
```

**预期输出**：

```json
{
  "Status": "Running",
  "ErrorMessage": "",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 3. 确认 eniipamd 组件运行正常
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd \
    | jq '{AddonName, AddonVersion, Phase}'
# expected: Phase "Running", AddonVersion ≥ 3.2.0
```

**预期输出**：

```json
{
  "AddonName": "eniipamd",
  "AddonVersion": "3.11.0",
  "Phase": "Running"
}
```

### 数据面（kubectl — 须 IOA/VPN/专线 或公网端点）

```bash
# 1. 确认预热注解生效
kubectl get nec NODE_NAME -o yaml | grep -E "min-warm-target|max-warm-target"
# expected: 显示配置的预热值

# 2. 确认快释放已启用
kubectl describe ds tke-eni-agent -n kube-system | grep enable-quick-release
# expected: 返回 --enable-quick-release（如已启用）
```

```text
NAME         STATUS    AGE
NODE_NAME    Active    2d
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| VPC-CNI 状态 | `DescribeClusters` → `Cni` | `true` |
| VPC-CNI 进度 | `DescribeEnableVpcCniProgress` → `Status` | `Running` |
| 非固定 IP | `EnableVpcCniNetworkType` 的 `EnableStaticIp` | `false` |
| eniipamd 版本 | `DescribeAddon` → `Phase` | `Running`, ≥ 3.2.0 |
| VPC-CNI 子网 | `DescribeClusters` → `ClusterNetworkSettings.Subnets` | 包含全部注册子网 |
| Pod IP 上限 | `DescribeVpcCniPodLimits` | 返回目标机型的各模式上限 |
| 预热配置 | `kubectl get nec -o yaml` | 注解与设置值一致 |

## 清理

> **注意**：移除 NEC 注解将恢复默认预热行为。VPC-CNI 为集群级一次性配置，开启后一般不关闭。

### 控制面（tccli）

> ⚠️ **关键警告**：
> - `DisableVpcCniNetworkType` 会重建所有使用 VPC IP 的 Pod，导致业务中断。生产环境禁用前务必确认影响范围。
> - 删除注册到 VPC-CNI 的子网前，必须先确认 VPC-CNI 不再使用该子网。直接删除会留下悬空引用，导致后续 Pod 调度到该子网时 IP 分配失败。
> - 如果 VPC-CNI 仍引用该子网（`DescribeClusters` → `ClusterNetworkSettings.Subnets` 中仍存在），需先通过 `DisableVpcCniNetworkType` 禁用 VPC-CNI（会重建所有 Pod），再删除子网。

```bash
# 1. 清理前检查：确认 VPC-CNI 使用的子网列表
tccli tke DescribeClusters --region REGION \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Subnets'
# 记录当前 VPC-CNI 子网列表，确认哪些需要保留

# 2. 删除测试子网（仅当子网不再被 VPC-CNI 使用）
# ⚠️ 如果子网仍在 Subnets 列表中，禁止直接删除！
tccli vpc DeleteSubnet --region REGION --SubnetId SUBNET_ID_NEW
# expected: exit 0

# 3. 验证子网已删除
tccli vpc DescribeSubnets --region REGION \
    --SubnetIds '["SUBNET_ID_NEW"]'
# expected: ResourceNotFound 或空列表
```

### 数据面（kubectl — 须 IOA/VPN/专线 或公网端点）

```bash
# 清除节点预热注解（恢复默认值）
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/route-eni-ip-min-warm-target"-
kubectl annotate nec NODE_NAME \
    "tke.cloud.tencent.com/route-eni-ip-max-warm-target"-
# expected: 注解已移除

# 确认恢复默认
kubectl get nec NODE_NAME -o yaml | grep warm-target
# expected: 空（无自定义值）
```

```text
NAME         STATUS    AGE
NODE_NAME    Active    2d
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableVpcCniNetworkType` 返回 `InvalidParameter.VpcCniType` | 检查 JSON 中的 `VpcCniType` 值 | 填了不支持的 VpcCniType 枚举 | 使用 `"tke-route-eni"`（共享网卡）或 `"tke-direct-eni"`（独占网卡） |
| `AddVpcCniSubnets` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region REGION --SubnetIds '["SUBNET_ID"]'` 确认子网是否存在及 VPC ID | 子网不属于集群所在 VPC 或不存在 | 确认子网 VPC ID 与集群 VPC ID 一致 |
| `kubectl annotate nec` 返回 `not found` | `kubectl get nec` 确认 NEC 是否存在 | 该节点非 VPC-CNI 节点或 eniipamd 未运行 | 确认节点已加入 VPC-CNI 集群且 eniipamd Phase=Running |
| `kubectl annotate nec` 值被拒绝 | 检查当前已用 ENI/IP 数 | 设置值 < 当前已用数量 | 改大值，确保 ≥ 当前使用量 |
| `DeleteSubnet` 成功但 VPC-CNI 子网列表未更新 | `tccli tke DescribeClusters --region REGION --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.Subnets'` | 直接删除子网不会自动从 VPC-CNI 注销 | 删除 VPC-CNI 子网前必须先禁用 VPC-CNI（`DisableVpcCniNetworkType`，慎用！会重建所有 Pod），或确保子网不再被引用 |

### 创建已提交但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableVpcCniNetworkType` 返回成功但 `DescribeEnableVpcCniProgress` Status 长时间非 Running | `tccli tke DescribeEnableVpcCniProgress --ClusterId CLUSTER_ID` 查看 Status 和 ErrorMessage | eniipamd 组件安装或初始化卡住（可能因子网不可用、安全组配置等） | 查看 ErrorMessage 定位原因；保留 region、ClusterId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| eniipamd Phase 长时间为 `Upgrading` | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName eniipamd` 查看 Phase | 组件升级未完成，后端可能因网络、资源等问题导致升级卡住 | ① 等待 5-10 分钟（正常升级通常在 5 分钟内完成）；② 如超过 10 分钟，保留 RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) → 组件管理 → eniipamd 查看升级详情 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| Pod 一直 Pending 提示 IP 不足 | `kubectl describe pod POD_NAME` 查看 Events：是否有 "failed to allocate IP from subnet" | VPC-CNI 子网可用 IP 耗尽或 Pod 数量超过节点 IP 上限 | ① `tccli tke DescribeVpcCniPodLimits` 确认节点类型支持的 Pod 上限；② `tccli vpc DescribeSubnets --SubnetIds '["SUBNET_ID"]'` 确认可用 IP；③ 如 IP 不足，创建新子网并 `AddVpcCniSubnets` 扩展 |

### 端点连接问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterEndpoint`（公网）返回 `InvalidParameter.Param`，含 `CAM deny: tke:clusterExtranetEndpoint=true, strategyId:240463971` | `tccli tke DescribeClusterEndpoints --ClusterId CLUSTER_ID` 确认公网端点是否已存在 | **组织级 CAM 策略** `strategyId:240463971` 以 `effect:deny` 硬拒绝 `tke:clusterExtranetEndpoint=true`。该策略为组织级别，自建安全组无法绕过 | 此为环境限制，非命令错误。使用内网端点（`ClusterIntranetEndpoint`）作为替代，需通过 IOA/VPN/专线 连接 VPC 后可达。或联系 CAM 管理员评估移除 deny 策略的可行性 |
| kubectl 连接 `ClusterIntranetEndpoint` 返回 `server gave HTTP response to HTTPS client` | `kubectl cluster-info`，检查 endpoint 是否为 HTTPS 地址 | 内网端点可能因网络中间设备导致协议不匹配 | 确认 kubeconfig 中 server 地址为 `https://`；检查是否有 HTTP 代理中间人 |
| kubectl 连接外网域名 `cls-xxxxxxxx.ccs.tencent-cloud.com` 返回 `no such host` | `nslookup cls-xxxxxxxx.ccs.tencent-cloud.com`，确认 DNS 解析 | 公网端点不存在（被 CAM 策略拒绝创建），域名无法解析 | 确认公网端点状态：`tccli tke DescribeClusterEndpoints --ClusterId CLUSTER_ID`。如 `ClusterExternalEndpoint` 为空，说明公网端点未创建（可能被 CAM 拒绝）→ 使用内网端点 + IOA/VPN/专线 |
| JNS Gateway 未配置（`JnsGwEndpoint=None`） | `tccli tke DescribeClusterSecurity --ClusterId CLUSTER_ID` 查看 JnsGwEndpoint | 集群未配置 JNS Gateway 隧道，内网端点本地不可达 | 需通过 IOA/VPN/专线 或同 VPC CVM 跳板机连接集群。JNS Gateway 为控制台功能，tccli 无对应 API |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 预热注解不生效 | `kubectl get nec NODE_NAME -o yaml` 检查注解 | min > max，配置被忽略 | 确保 min ≤ max，且两者都设置（成对） |
| 快释放未生效 | `kubectl rollout status ds tke-eni-agent -n kube-system` | DaemonSet 滚动更新未完成 | 等待所有 Pod 更新完成（注意：更新期间新 Pod 启动延迟） |
| 默认预热值不应用于存量节点 | `kubectl get nec NODE_NAME -o yaml` 查看注解 | 默认预热值（eniipamd 启动参数）仅影响**新节点** | 对存量节点逐个执行 `kubectl annotate nec` |
| Pod 创建慢（等 IP） | `kubectl describe pod POD_NAME` 查看 Events | 预热池 IP 耗尽，min_warm 设置过低 | 增大 `min-warm-target` 值 |

## 下一步

- [VPC-CNI 模式介绍](../VPC-CNI 模式介绍/tccli 操作.md) -- page_id `45709`
- [固定 IP 模式使用说明](../固定 IP 模式使用说明/tccli 操作.md) -- page_id `34994`
- [固定 IP 相关特性](../固定 IP 相关特性/tccli 操作.md) -- page_id `50358`
- [VPC-CNI（eniipamd）组件介绍](../VPC-CNI（eniipamd）组件介绍/tccli 操作.md)
- [Pod 间独占网卡模式](../Pod 间独占网卡模式/tccli 操作.md)
- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI 模式 Pod 数量限制/tccli 操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 基本信息 → VPC-CNI 模式](https://console.cloud.tencent.com/tke2/cluster) → 查看 VPC-CNI 配置、子网列表、eniipamd 组件版本。组件管理 → eniipamd → 更新配置，可设置默认预热值和快释放。
