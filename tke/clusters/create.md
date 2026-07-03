---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建集群

> 创建新的 TKE Kubernetes 集群。异步操作，通常 5-10 分钟完成。
> 控制台: [容器服务 - 集群](https://console.cloud.tencent.com/tke2/cluster)

> 本文档 Action 属 **TKE 2018-05-25**（`CreateCluster` 是旧版独有 Action，新版无）。文中 `DescribeClusters` 是两版同名 Action，本文走旧版（取丰富字段，见 [查询集群](query.md#两版同名-action-describeclusters)）；两版入参一致，跨版本仅响应结构不同。

## 概述

创建 TKE 集群是在腾讯云上运行 K8s 的第一步。TKE 提供了两种集群类型:

| 选项 | 最佳场景 | 关键限制 | 升级路径 |
|------|---------|:---:|---------|
| MANAGED_CLUSTER (托管) | 生产环境，免运维 Master | Master 不可 SSH | 控制台/API 一键升级 |
| INDEPENDENT_CLUSTER (独立) | 完全控制 Master 节点 | 需自行维护 Master HA | 手动升级 Master |

**默认推荐**: 托管集群。99% 的场景下托管集群是最优解 —— Master 由腾讯云运维，你只需要管理工作节点。

操作是**异步**的: 命令返回 `ClusterId` 即表示创建已提交，集群就绪需等待 5-10 分钟。

## 触发条件

- 账号下没有集群，或现有集群不够用，需新建一个 TKE 集群（`DescribeClusters` 返回的集群不满足需求）— 用本文创建
- 需要一个独立/托管集群承载新业务，且已备好 VPC+子网（见 [准备 VPC 与子网](../../getting-started/prepare-vpc.md)）— 本文从 `CreateCluster` 开始

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号 (如 3.1.117.1)

tccli cvm DescribeRegions --region ap-guangzhou
# expected: { "RegionSet": [...] }  → 凭证有效（tccli 默认剥离 Response 包装层）
```

#### 凭证模式（含集群内免密）

tccli 支持四种凭证模式，除显式 `SecretId/SecretKey` 外，还有三种 agent 友好的免密模式（在 TKE 服务下自动生效，无需额外配置）：

| 模式 | 触发方式 | 适用场景 |
|:-----|:---------|:---------|
| 显式凭证 | `tccli configure` 配置 | 本地开发，默认 |
| CVM 角色 | `--use-cvm-role` | tccli 跑在绑定了 CAM 角色的 CVM 上（如 Master/工作节点） |
| TKE OIDC 角色 | 环境变量 `TKE_REGION`+`TKE_PROVIDER_ID`+`TKE_WEB_IDENTITY_TOKEN_FILE`+`TKE_ROLE_ARN` 全设 | tccli 跑在 TKE 集群 Pod 内，用 OIDC 角色免密（推荐，集群内 agent 场景） |
| STS 角色 | `--role-arn`+`--role-session-name` | 跨账号 STS 角色切换 |

```bash
# 集群内 Pod 跑 tccli：用 TKE OIDC 免密（Pod 关联了 CAM 角色，无需 SecretKey）
# 四个环境变量由 TKE 注入，tccli 自动检测并启用
tccli tke DescribeClusters --region <REGION> --filter "TotalCount" --output text
# expected: 集群总数（无需 configure，凭证来自 OIDC 角色）

# CVM 上跑 tccli：用 CVM 角色（实例绑定了 CAM 角色）
tccli tke DescribeClusters --region <REGION> --use-cvm-role --filter "TotalCount" --output text
```

> 集群内 agent 用 OIDC 免密是生产推荐——避免把 SecretKey 注入 Pod。前提是 Pod 关联的 CAM 角色有相应 TKE Action 权限。

### 资源检查

```bash
# 1. 确认集群名未被占用
tccli tke DescribeClusters --region ap-guangzhou \
  --Filters '[{"Name":"ClusterName","Values":["<CLUSTER_NAME>"]}]'
# expected: { "TotalCount": 0 }  → 名称可用

# 2. 确认 VPC 存在（创建托管集群时 VpcId 必传）
tccli vpc DescribeVpcs --region ap-guangzhou --VpcIds '["<VPC_ID>"]'
# expected: { "VpcSet": [{ "VpcId": "<VPC_ID>" }] }  → VPC 可用
# 无 VPC → 见 准备 VPC 与子网 (../../getting-started/prepare-vpc.md)

# 3. 确认子网存在（集群本身不强制要子网，但节点池要从子网分配节点 IP——提前备好）
tccli vpc DescribeSubnets --region ap-guangzhou \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[].{id:SubnetId,avail:AvailableIpAddressCount}" --output text
# expected: 至少 1 个子网，AvailableIpAddressCount ≥ 10
# 无子网 → 见 准备 VPC 与子网 (../../getting-started/prepare-vpc.md) 的"创建子网"段

# 4. 确认集群配额未满
tccli tke DescribeClusters --region ap-guangzhou
# expected: TotalCount < 配额上限 (默认 50)
```

## 关键字段

> 注意：API 层 `required` 与业务层"必需"不同——`ClusterBasicSettings` 在 API 层非必填，但不传 `ClusterName`/`VpcId` 集群无法正常使用；下表"必填"列按 API 层 `required` 标注，"业务必需"在约束列说明。

### 顶层参数（API 必填项）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterType | string | 是 | `MANAGED_CLUSTER` / `INDEPENDENT_CLUSTER` | `InvalidParameterValue.ClusterType` |
| ClusterCIDRSettings | object | 是 | ClusterCIDR + ServiceCIDR（不传报 `the following arguments are required: --ClusterCIDRSettings`） | `InvalidParameterValue.ClusterCIDR` |

> ⚠️ `ClusterType` 与 `ClusterCIDRSettings` 是仅有的两个 API 层必填项。缺 `ClusterCIDRSettings` 命令直接 exit 252 失败，不会进入业务校验。

**ClusterCIDRSettings 子字段**（8 字段）：

| 子字段 | 适用网络类型 | 说明 |
|:-------|:-----------|:-----|
| `ClusterCIDR` | 全部 | 容器网段，如 `172.16.0.0/16`，不得与 VPC CIDR 冲突 |
| `ServiceCIDR` | 全部 | 服务网段，如 `10.96.0.0/20` |
| `MaxNodePodNum` | GR | 单节点最大 Pod 数（影响 IP 分配），如 `64` |
| `MaxClusterServiceNum` | GR | 集群最大 Service 数，如 `4096` |
| `IgnoreClusterCIDRConflict` | 全部 | `true` 忽略与 VPC 路由表冲突，默认 `false` |
| `IgnoreServiceCIDRConflict` | 全部 | `true` 忽略 ServiceCIDR 冲突，默认 `false` |
| `EniSubnetIds` | VPC-CNI | ENI 模式子网 ID 列表（`NetworkType=VPC-CNI` 时用） |
| `ClaimExpiredSeconds` | VPC-CNI | ENI IP 续期时长（VPC-CNI 模式） |

> `MaxNodePodNum`/`MaxClusterServiceNum` 是 GR 模式的容量规划参数；`EniSubnetIds`/`ClaimExpiredSeconds` 仅 VPC-CNI 模式用。网络类型由 `ClusterAdvancedSettings.NetworkType` 决定（见下表）。

### ClusterBasicSettings（业务必需，API 非必填）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterName | string | 否 | ≤60 字符, 字母数字连字符；业务必需，不传集群无名 | `InvalidParameterValue.ClusterName` |
| ClusterVersion | string | 否 | 如 `1.34.1`，默认 `1.10.5`；业务必需 | `InvalidParameterValue.ClusterVersion` |
| VpcId | string | 否 | 已有 VPC ID；创建托管集群时业务必需（"创建托管空集群时必传"） | `ResourceNotFound.VpcId` |
| ClusterOs | string | 否 | `ubuntu20.04x86_64` / `tlinux3.1x86_64` 等；不传用默认 OS | `InvalidParameterValue.ClusterOs` |
| OsCustomizeType | string | 否 | `GENERAL`（通用，默认）/ `Rh8` 等；OS 定制类型 | `InvalidParameterValue` |
| ClusterLevel | string | 否 | 可用等级 L5/L20/L50/L100/L200/L500（无 L10），默认 `L5`；L1000/L3000/L5000 需工单开通（`Enable=false`），见 [配置集群 — 选等级](configure.md#为什么选这个等级) | `InvalidParameterValue.ClusterLevel` |
| SubnetId | string | 否 | VPC 内子网 ID | `ResourceNotFound.SubnetId` |
| ProjectId | integer | 否 | 项目 ID，默认 `0` | `InvalidParameterValue.ProjectId` |
| NeedWorkSecurityGroup | boolean | 否 | `true` 自动创建工作节点安全组；不传按默认 | — |
| TagSpecification | list | 否 | 创建即打标签：`[{ResourceType:"cluster",Tags:[{Key,Value}]}]`；CAM 标签授权场景（如维护窗口/Master 扩缩容要求 `billing` 标签）建议创建时即打 | `InvalidParameterValue` |
| AutoUpgradeClusterLevel | object | 否 | `{IsAutoUpgrade:true}` 集群等级自动升级（有计费风险，谨慎） | — |

### ClusterAdvancedSettings（建议设置）

| 字段 | 类型 | 推荐值 | 作用 |
|-------|------|--------|------|
| DeletionProtection | boolean | `true` | 防止误删集群 |
| AuditEnabled | boolean | `true` | 开启审计日志（需配 `AuditLogsetId`/`AuditLogTopicId`，CLS 日志集/主题 ID） |
| ContainerRuntime | string | `containerd` | 容器运行时 |
| RuntimeVersion | string | `1.6.9` | 运行时版本（须 `DescribeSupportedRuntime` 返回的版本） |
| NetworkType | string | `GR` | 容器网络类型枚举：`GR`（全局路由，默认，支持后期 `AddClusterCIDR` 扩容）/ `VPC-CNI`（ENI 模式，见 [配置 VPC-CNI](../networking/vpc-cni.md)）/ `CiliumOverlay`（Cilium，见 [配置 CiliumOverlay](../networking/cilium-overlay.md)）。**选 GR 才能用 `AddClusterCIDR` 扩 Pod 网段** |
| IPVS | boolean | `true` | 是否启用 IPVS。`true`=ipvs 模式（大规模集群推荐）/ `false`=iptables 模式 |
| KubeProxyMode | string | 不设 | kube-proxy 模式枚举：不设=iptables（IPVS=false）/ `ipvs`（设 IPVS=true）/ `kube-proxy-bpf`=ipvs-bpf 模式（高性能，需集群 ≥1.14 + Tencent Linux 镜像，详见 [kube-proxy-bpf 模式](https://cloud.tencent.com/document/product/457/39238)）。**ipvs-bpf 是第三种独立模式，非 IPVS 的子项** |
| IsDualStack | boolean | `false` | VPC-CNI 模式下是否 IPv4/IPv6 双栈（仅 `NetworkType=VPC-CNI` 适用） |
| AsEnabled | boolean | `false` | 是否启用节点池弹性伸缩（AS）。**创建集群流程不支持开启此功能**，需集群创建后通过节点池 `ModifyClusterAsGroupAttribute` 开启 |
| NodeNameType | string | `lan-ip` | 节点名类型：`lan-ip`（内网 IP）/ `hostname` |
| QGPUShareEnable | boolean | `false` | 是否开启 QGPU 共享（GPU 场景） |

> `NetworkType=GR` 是后期 `AddClusterCIDR` 扩容 Pod 网段的前置——VPC 模式（ENI）集群不支持 `AddClusterCIDR`。见 [配置集群属性与运行时](configure.md#步骤-5扩容容器网段)。

## 操作步骤

> ⚠️ **本文创建的是空集群（控制面）**：`CreateCluster` 后 `ClusterState=Running` 但 `ClusterRunningNodeNum=0`，**不能跑 Pod**。
>
> "能跑 Pod 的集群"完整闭环 = 本步（空集群）→ [创建节点池](../nodes/nodepool-create.md)（加工作节点）→ [开端点](../networking/endpoints.md) → [kubectl 连通](../security/auth.md)。
>
> 本步约 5-10 分钟，完成后须立即进入 [创建节点池](../nodes/nodepool-create.md) 加节点，否则集群空转仍计管理费。完整生命周期见 [集群生命周期故事线](index.md#集群生命周期故事线)。

### 步骤 1：决策 — 选集群类型

#### 为什么选 MANAGED_CLUSTER

- **托管 vs 独立**: 托管集群的 Master 由腾讯云负责 (HA、升级、备份)；独立集群需要自己维护 3 台 Master 节点
- **成本差异**: 托管集群收取集群管理费 (L5 约 ¥0.4/小时)；独立集群不收取管理费但需支付 3 台 Master 的 CVM 费用
- **默认推荐**: MANAGED_CLUSTER — 你不需要管理 Master
- **能改吗?**: 不能。集群类型创建后无法切换。选了独立就得一直维护 Master

### 步骤 2：创建 — 最小化

最少必需参数: ClusterType + ClusterCIDRSettings（API 必填）+ 集群名 + VPC + K8s 版本（业务必需）。

```bash
tccli tke CreateCluster \
  --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{
    "ClusterName": "<CLUSTER_NAME>",
    "ClusterVersion": "1.34.1",
    "ClusterOs": "ubuntu20.04x86_64",
    "VpcId": "<VPC_ID>"
  }' \
  --ClusterCIDRSettings '{
    "ClusterCIDR": "172.16.0.0/16",
    "ServiceCIDR": "10.96.0.0/20"
  }' \
  --ClusterAdvancedSettings '{
    "DeletionProtection": true
  }'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }（tccli 默认剥离 Response 包装层，顶层直接是 ClusterId/RequestId）
```

> ⚠️ **此命令创建的是无工作节点的空集群**（无 `RunInstancesForNode`，`ClusterRunningNodeNum=0`）。集群会进入 `Running` 状态但**不能直接运行 Pod**——必须接着 [创建节点池](../nodes/nodepool-create.md) 添加工作节点。`ClusterCIDR` 不得与 VPC CIDR 冲突。

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<CLUSTER_NAME>` | 集群名称 | ≤60 字符，全局唯一 | 自己定义 |
| `<VPC_ID>` | VPC 实例 ID | 必须存在 | `tccli vpc DescribeVpcs` → `VpcSet[].VpcId` |

> 命令中的 `ClusterCIDR`（如 `172.16.0.0/16`）与 `ServiceCIDR`（如 `10.96.0.0/20`）是字面量示例——按你的 VPC CIDR 选择不重叠的网段，不得与 VPC CIDR 冲突。

### 步骤 3：创建 — 增强：容量规划与审计

在最小化基础上规划 Pod/Service 容量（`MaxNodePodNum`/`MaxClusterServiceNum`）、提升集群等级（`ClusterLevel`）、开启审计与指定运行时（适合生产）:

```bash
tccli tke CreateCluster \
  --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{
    "ClusterName": "<CLUSTER_NAME>",
    "ClusterVersion": "1.34.1",
    "ClusterOs": "ubuntu20.04x86_64",
    "VpcId": "<VPC_ID>",
    "ClusterLevel": "L20"
  }' \
  --ClusterCIDRSettings '{
    "ClusterCIDR": "172.16.0.0/16",
    "ServiceCIDR": "10.96.0.0/20",
    "MaxNodePodNum": 64,
    "MaxClusterServiceNum": 4096
  }' \
  --ClusterAdvancedSettings '{
    "DeletionProtection": true,
    "AuditEnabled": true,
    "ContainerRuntime": "containerd"
  }'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }
```

这启用了自定义 CIDR（适合多集群/VPC 互联场景）、审计日志、集群等级 L20（支持更多节点）。

### 步骤 4：验证

异步操作: 检查 ≥4 个维度确认集群可用。

```bash
# 用 --waiter 自动轮询集群到 Running（避免手写循环；waiter 对所有 TKE Action 生效）
tccli tke DescribeClusterStatus --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]' \
  --waiter '{"expr":"ClusterStatusSet[0].ClusterState","to":"Running","timeout":600,"interval":10}'
# expected: 集群 Running 后返回，超时则报 ClientError

# 或手动轮询
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
```

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| Status | `DescribeClusterStatus` | `ClusterState: "Running"` |
| 集群属性 | `DescribeClusters --ClusterIds '["<ID>"]'` | ClusterName、ClusterVersion、VpcId 与创建参数一致 |
| 删除保护 | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterDeletionProtection` | `true`（布尔值，非 Enabled/Disabled） |
| kubeconfig | `DescribeClusterKubeconfig --ClusterId "<ID>"` | 返回有效 kubeconfig |
| kubectl 连通 | `kubectl --kubeconfig kubeconfig.yaml get nodes` | exit 0（需先开端点，见 [管理端点](../networking/endpoints.md)） |
| 节点数 | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterRunningNodeNum` | ≥0（空集群为 0，字段名是 ClusterRunningNodeNum 非 NodeCount） |

> `--waiter` 的 `expr` 必须用 `DescribeClusterStatus` 响应字段名 `ClusterStatusSet[0].ClusterState`（非 `Clusters[0].ClusterStatus`）。`CreateCluster` 本身也可直接挂 `--waiter`，但 `CreateCluster` 响应无状态字段，需轮询 `DescribeClusterStatus`。

## 清理

> ⚠️ **Side-effect 警告**: 删除集群会:
> - 销毁集群内所有**工作节点** (CVM)，数据不可恢复
> - **不自动删除**: CBS 云硬盘、弹性公网 IP (EIP)、CLB 负载均衡器
> - 托管集群的 Master 由腾讯云自动回收
> - 计费停止时间: 集群删除即停止收取管理费；CVM、CBS、EIP 按各自计费规则继续

### 1. 清理前检查

```bash
# 确认要删除的是正确的集群
tccli tke DescribeClusters \
  --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]'
# expected: 确认 ClusterName 和 ClusterId 匹配你的目标
```

### 2. 关闭删除保护

```bash
tccli tke DisableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

### 3. 删除

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceDeleteMode terminate
# expected: exit 0
```

### 4. 验证已删除

```bash
tccli tke DescribeClusters \
  --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "TotalCount": 0, "Clusters": [] }
```

> **Billing warning**: 集群管理费在删除后立即停止。但 CBS 盘和 EIP 会持续扣费，需手动清理:
> ```bash
> tccli cbs DescribeDisks --region ap-guangzhou  # 检查残留 CBS
> tccli vpc DescribeAddresses --region ap-guangzhou  # 检查残留 EIP
> ```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `AuthFailure.SecretIdNotFound` | `tccli tke DescribeRegions` | 凭证未配置或已过期 | 见 [配置凭证](../../getting-started/credentials.md) 重新配置 |
| `InvalidParameterValue.ClusterName` | 检查名称格式 | 名称含特殊字符或超长 | 使用 ≤60 字符的字母数字连字符 |
| `InvalidParameterValue.ClusterVersion` | 名称格式正确但仍报错 | 版本号不存在 | `tccli tke DescribeVersions` 查询可用版本列表 |
| `ResourceNotFound.VpcId` | `tccli vpc DescribeVpcs` | VPC ID 不存在或地域错误 | 确认 VPC 在该地域存在，或 `tccli vpc CreateVpc` 新建 |
| `LimitExceeded.ClusterQuota` | `tccli tke DescribeClusters` | 集群数量达上限 (默认 50) | 删除闲置集群或提交工单提升配额 |
| `InvalidParameterValue.ClusterCIDR` | CIDR 格式: `A.B.C.D/掩码` | CIDR 格式不合法 | 用合法 CIDR 格式 |
| `InvalidParameter.CidrConflictWithVpcCidr` | `tccli vpc DescribeVpcs --VpcIds '["<VPC_ID>"]'` 查 VPC CIDR | `ClusterCIDR` 与 VPC CIDR 重叠（如 VPC 是 `172.16.0.0/12` 则 `172.24.x.x/16` 冲突） | 换一个与 VPC CIDR 不重叠的网段（如 VPC 用 `172.16/12` 时容器网段可用 `192.168.0.0/16`） |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 集群卡在 `Creating` > 30 分钟 | `tccli tke DescribeClusterStatus` | 可用区资源不足或 VPC 配置冲突 | 1. 等待 10 分钟观察 2. 若超 30 分钟，删除重建并换可用区 |
| 状态为 `Running` 但 `DescribeClusterKubeconfig` 失败 | 检查返回的 kubeconfig 内容 | 集群就绪延迟 (API Server 未完全就绪) | 等待 1-2 分钟后重试 |
| 状态为 `Abnormal` | `tccli tke DescribeClusterStatus` | Master 组件异常 (罕见) | `DeleteCluster` → 重建。若重建仍异常，提交工单 |
| 删除保护无法关闭 | `tccli tke DisableClusterDeletionProtection` | CAM 权限不足 | 确认账号有 `tke:DisableClusterDeletionProtection` 权限 |

## 收尾确认

```bash
# 一次性核对：集群已 Running + 删除保护已开 + 空集群(0 节点，待加节点池)
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].{state:ClusterState,running:ClusterRunningNodeNum,protect:ClusterDeletionProtection}"
# expected: state=Running, running=0(空集群), protect=true → 创建闭环完成，可进入创建节点池
```

## 下一步

> 集群 `Running` 只是创建闭环的第 1 步（空集群）。继续完成"能跑 Pod 的集群"：

- **[创建节点池](../nodes/nodepool-create.md)** — 给集群添加工作节点（**必做**：集群没有节点不能运行 Pod，且空集群仍计管理费）
- [获取 kubeconfig](../security/auth.md) — 配置 kubectl 访问集群
- [管理端点](../networking/endpoints.md) — 开启公网/内网访问端点（kubectl 远程连接的前置）
- [查询集群](query.md) — 学习 filter + JMESPath 表达式
- [应用发布](../releases/index.md) — 集群就绪后部署应用
- [独立集群 Master 运维](master-ops.md) — 选了独立集群时，扩缩容 Master/etcd 节点

## 控制台替代方案

[容器服务控制台 - 创建集群](https://console.cloud.tencent.com/tke2/cluster/create)
