---
doc_type: How-to
subtype: 6A
fused: true
---

# 创建集群

> 创建新的 TKE Kubernetes 集群。异步操作，约 5-10 分钟完成。
> 控制台: [容器服务 - 集群](https://console.cloud.tencent.com/tke2/cluster)

> 本文档 Action 属 **TKE 2018-05-25** 

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [创建集群](https://cloud.tencent.com/document/product/457/103981) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 概述

创建 TKE 集群即在腾讯云上运行 K8s。TKE 提供两种集群类型:

| 选项                       | 最佳场景            | 关键限制            | 升级路径         | 新建 |
| ------------------------ | --------------- | --------------- | ------------ | ---- |
| MANAGED_CLUSTER (托管)     | 生产环境，免运维 Master | Master 不可 SSH；不可改 Master/Etcd **节点规模**（控制面参数/组件仍可由 tccli 改，见 [配置集群](configure.md)） | 控制台/API 发起升级 | ✅ |
| INDEPENDENT_CLUSTER (独立) | 存量：完全控制 Master  | 需自行维护 Master HA | 手动升级 Master  | ❌ **已停止新建** |

**默认推荐**: `MANAGED_CLUSTER`。Master/Etcd 由腾讯云运维，你管理工作节点。`INDEPENDENT_CLUSTER` 已停止新建（空参创建报 `FailedOperation.Param`：`not has master NodeRole`）；勿用独立类型做新建路径。

> 控制台"新建集群"含三类形态（标准/Serverless/注册集群）+ 创建流全景（托管4步/独立5步），见 [TKE 容器服务 — 控制台创建流全景](../index.md#控制台创建流全景)。本文只覆盖标准集群的 `CreateCluster` 操作。

操作是**异步**的: 命令返回 `ClusterId` 即表示创建已提交，集群就绪需等待 5-10 分钟。

> 配额：单地域集群默认 **20**（可提工单调高）。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 账号下没有集群，或现有集群不满足需求，需新建一个 TKE 集群（`DescribeClusters` 返回的集群不满足需求）— 用本文创建
- 需要一个独立/托管集群承载新业务，且已备好 VPC+子网（见 [准备 VPC 与子网](../../getting-started/prepare-vpc.md)）— 本文从 `CreateCluster` 开始

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号（最新版或更高）

tccli tke DescribeRegions
# expected: { "TotalCount": ..., "RegionInstanceSet": [...] }  → 凭证有效 + TKE 域可达（顶层键是 RegionInstanceSet，非 RegionSet；tccli 默认剥离 Response 包装层；鉴权探针无需先假定 --region）
```

### 资源检查

按序执行；任一步失败先补齐依赖再继续。命令在代码块内；失败跳转见各步块外链接。

#### 1. 集群名未被占用

```bash
tccli tke DescribeClusters --region ap-guangzhou \
  --Filters '[{"Name":"ClusterName","Values":["<CLUSTER_NAME>"]}]'
# expected: { "TotalCount": 0 }  → 名称可用
```

#### 2. VPC 存在（托管集群 `VpcId` 必传）

```bash
tccli vpc DescribeVpcs --region ap-guangzhou --VpcIds '["<VPC_ID>"]'
# expected: { "VpcSet": [{ "VpcId": "<VPC_ID>" }] }  → VPC 可用
```

无 VPC → [准备 VPC 与子网](../../getting-started/prepare-vpc.md)

#### 3. 子网存在（节点池要从子网分 IP，创建前备好）

```bash
tccli vpc DescribeSubnets --region ap-guangzhou \
  --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[].{id:SubnetId,avail:AvailableIpAddressCount}" --output text
# expected: 至少 1 个子网，AvailableIpAddressCount ≥ 10
```

无子网 → [准备 VPC 与子网 — 创建子网](../../getting-started/prepare-vpc.md#2-创建子网)

#### 4. 集群配额未满

```bash
tccli tke DescribeClusters --region ap-guangzhou
# expected: TotalCount < 配额上限（单地域默认 20）
```

配额说明见 [配额](../reference/quotas.md)

#### 5. 服务角色：`TKE_QCSRole`（任意建集群）

```bash
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?RoleName=='TKE_QCSRole'].RoleName" --output text
# expected: TKE_QCSRole
```

空 → [配置凭证 — 补 TKE_QCSRole](../../getting-started/credentials.md#补-tke_qcsrole主服务角色)

#### 6. 服务角色：`IPAMDofTKE_QCSRole`（仅 `NetworkType=VPC-CNI` 时必查；快速入门默认 VPC-CNI）

```bash
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?RoleName=='IPAMDofTKE_QCSRole'].RoleName" --output text
# expected: IPAMDofTKE_QCSRole
```

空 → [配置凭证 — 补 IPAMD](../../getting-started/credentials.md#补-ipamdoftke_qcsrolevpc-cni-前置) 或 [VPC-CNI — IPAMD 服务角色](../networking/vpc-cni.md#ipamd-服务角色)

### 创建前必读（创建后改不了） {#创建前必读创建后改不了}

| 决策项 | 约束 | 错了怎么办 |
|:-------|:-----|:-----------|
| **地域**（`--region` / 购买地域） | 不同地域云产品内网不通；**购买后不能更换** | 只能在目标地域重建集群 |
| **容器网络插件** `NetworkType` | `GR` / `VPC-CNI` / `CiliumOverlay` 创建时选定；仅 VPC-CNI 可事后开启，GR/Cilium 换插件须重建 | 重建或走受限变更路径 |
| **容器网段 CIDR** | 决定节点/Service/每节点 Pod 上限；**暂不支持变更** | 重建或走受限扩网段（仅 GR 支持 `AddClusterCIDR`） |
| **Service CIDR** | 创建时定；与同 VPC 其他集群冲突会影响互通 | 重建；勿随意勾选「忽略冲突校验」 |
| **IPVS** | 仅新建时可开；**开启后不可关闭**；勿与 iptables 混用 | 重建集群 |
| **Kube-proxy / Dataplane v2** | iptables↔ipvs **一经选择不支持更改**；`DataPlaneV2=true` 不再装 kube-proxy | 重建集群 |
| **安全组** | 节点/Master 须按推荐放通（含容器 CIDR 的 DNS 53/udp 等） | 改错可能导致节点/Master 不可用，见 [安全组](https://cloud.tencent.com/document/product/457/9084) |
| **服务授权** | 须有 `TKE_QCSRole`（+ 默认策略）；**VPC-CNI 另须 `IPAMDofTKE_QCSRole`**；节点池另须 `AS_QCSRole` | 探测与 CLI 补齐见 [配置凭证 — 服务角色](../../getting-started/credentials.md#服务角色tke--ipamd--as--tcr--可观测)；官方说明 [43416](https://cloud.tencent.com/document/product/457/43416) |

## 关键字段

> 注意：API 层 `required` 与业务层"必需"不同——`ClusterBasicSettings` 在 API 层非必填，但不传 `ClusterName`/`VpcId` 集群无法正常使用；下表"必填"列按 API 层 `required` 标注，"业务必需"在约束列说明。

### 顶层参数（API 必填项）

| 字段                  | 类型     | 必填  | 约束                                                                                           | 填错时的错误                              |
| ------------------- | ------ | --- | -------------------------------------------------------------------------------------------- | ----------------------------------- |
| ClusterType         | string | 是   | `MANAGED_CLUSTER` / `INDEPENDENT_CLUSTER`（后者**已停止新建**；空集群创建报 `FailedOperation.Param`：`not has master NodeRole`，非 `InvalidParameterValue.ClusterType`） | 非法枚举可能 `InvalidParameterValue.ClusterType`；停新建独立见上 |
| ClusterCIDRSettings | object | 是   | ClusterCIDR + ServiceCIDR（不传报 `the following arguments are required: --ClusterCIDRSettings`） | `InvalidParameterValue.ClusterCIDR` |

> 顶层另有可选入参 `CdcId`（本地专用集群 ID，部署在 CDC 时传，`required=False`），见 [TKE 容器服务 — 托管集群「集群信息」步](../index.md#托管集群4-步)。

> ⚠️ `ClusterType` 与 `ClusterCIDRSettings` 是仅有的两个 API 层必填项。缺 `ClusterCIDRSettings` 命令直接 exit 252 失败，不会进入业务校验。

**ClusterCIDRSettings 子字段**（8 字段）：

| 子字段                         | 适用网络类型  | 说明                                       |
| --------------------------- | ------- | ---------------------------------------- |
| `ClusterCIDR`               | 全部      | 容器网段，如 `172.16.0.0/16`，不得与 VPC CIDR 冲突   |
| `ServiceCIDR`               | 全部      | 服务网段，如 `10.96.0.0/20`                    |
| `MaxNodePodNum`             | GR      | 单节点最大 Pod 数（影响 IP 分配），如 `64`             |
| `MaxClusterServiceNum`      | GR / VPC-CNI | 集群最大 Service 数；须 ≥ `ServiceCIDR` 容量（`/17`→32768）。过小报 `FailedOperation.Param`（消息含 `service ip number is N, but the ClusterServiceNum paramter is M`） |
| `IgnoreClusterCIDRConflict` | 全部      | `true` 忽略与 VPC 路由表冲突，默认 `false`          |
| `IgnoreServiceCIDRConflict` | 全部      | `true` 忽略 ServiceCIDR 冲突，默认 `false`      |
| `EniSubnetIds`              | VPC-CNI | ENI 模式子网 ID 列表（`NetworkType=VPC-CNI` 时用） |
| `ClaimExpiredSeconds`       | VPC-CNI | ENI IP 续期时长（VPC-CNI 模式）                  |

> `MaxNodePodNum`/`MaxClusterServiceNum` 是 GR 模式的容量规划参数；`EniSubnetIds`/`ClaimExpiredSeconds` 仅 VPC-CNI 模式用。网络类型由 `ClusterAdvancedSettings.NetworkType` 决定（见下表）。

### ClusterBasicSettings（业务必需，API 非必填）

| 字段                      | 类型      | 必填  | 约束                                                                                                                          | 填错时的错误                                                                |
| ----------------------- | ------- | --- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| ClusterName             | string  | 否   | ≤60 字符, 字母数字连字符；业务必需，不传集群无名                                                                                                 | `InvalidParameterValue.ClusterName`                                   |
| ClusterVersion          | string  | 否   | 如 `1.34.1`，默认 `1.10.5`；业务必需（须完整三段式，`1.30` 会失败）                                                                              | 版本不存在 → `FailedOperation.DbRecordNotFound`（非 `InvalidParameterValue`） |
| VpcId                   | string  | 否   | 已有 VPC ID；创建托管集群时业务必需（"创建托管空集群时必传"）                                                                                         | `ResourceNotFound.VpcId`                                              |
| ClusterOs               | string  | 否   | `ubuntu20.04x86_64` / `tlinux3.1x86_64` 等；不传用默认 OS                                                                          | `InvalidParameterValue.ClusterOs`                                     |
| OsCustomizeType         | string  | 否   | `GENERAL`（普通版本，默认）/ `DOCKER_CUSTOMIZE`（容器定制版）；非法值报 `InvalidParameterValue`                                                                                           | `InvalidParameterValue`                                               |
| ClusterLevel            | string  | 否   | 真实枚举 `L5`/`L20`/`L50`/`L100`/`L200`/`L500`/`L1000`/`L3000`/`L5000`（**无 L10**），默认 `L5`；是否可选用以 `DescribeClusterLevelAttribute` 返回的 `Enable` 为准（部分账号高等级可能 `Enable=false` 需工单），见 [配置集群 — 选等级](configure.md#为什么选这个等级) | `InvalidParameterValue.ClusterLevel`                                  |
| SubnetId                | string  | 否   | VPC 内子网 ID                                                                                                                  | `ResourceNotFound.SubnetId`                                           |
| ProjectId               | integer | 否   | 项目 ID，默认 `0`                                                                                                                | `InvalidParameterValue.ProjectId`                                     |
| NeedWorkSecurityGroup   | boolean | 否   | `true` 自动创建工作节点安全组；不传按默认                                                                                                    | —                                                                     |
| TagSpecification        | list    | 否   | 创建即打标签：`[{ResourceType:"cluster",Tags:[{Key,Value}]}]`；CAM 标签授权场景（如维护窗口/Master 扩缩容要求 `billing` 标签）建议创建时即打                   | `InvalidParameterValue`                                               |
| AutoUpgradeClusterLevel | object  | 否   | `{IsAutoUpgrade:true}` 集群等级自动升级（有计费风险，谨慎）                                                                                   | —                                                                     |
| ClusterDescription      | string  | 否   | 集群描述，≤200 字符                                                                                                                | —                                                                     |

### ClusterAdvancedSettings（建议设置）

| 字段                  | 类型      | 推荐值          | 作用                                                                                                                                                                                                                                         |
| ------------------- | ------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| DeletionProtection  | boolean | `true`       | 防止误删集群                                                                                                                                                                                                                                     |
| AuditEnabled        | boolean | `true`       | 开启审计日志（需配 `AuditLogsetId`/`AuditLogTopicId`，CLS 日志集/主题 ID）                                                                                                                                                                                 |
| ContainerRuntime    | string  | `containerd` | 容器运行时                                                                                                                                                                                                                                      |
| RuntimeVersion      | string  | `1.6.9`      | 运行时版本（须 `DescribeSupportedRuntime` 返回的版本）                                                                                                                                                                                                  |
| NetworkType         | string  | 按场景选（见下） | 容器网络类型：`GR` / `VPC-CNI` / `CiliumOverlay`。**契约默认 GR**（不传即 GR），不等于生产场景应选值——需固定 IP/安全组 → `VPC-CNI`；要省 VPC IP 且可后期 `AddClusterCIDR` → `GR`；要 Cilium 且不占 VPC IP → `CiliumOverlay`。详 [网络管理](../networking/index.md) |
| IPVS                | boolean | `true`       | 是否启用 IPVS。`true`=ipvs 模式（大规模集群推荐）/ `false`=iptables 模式                                                                                                                                                                                     |
| KubeProxyMode       | string  | 不设           | kube-proxy 模式枚举：不设=iptables（IPVS=false）/ `ipvs`（设 IPVS=true）/ `kube-proxy-bpf`=ipvs-bpf 模式（高性能，需集群 ≥1.14 + Tencent Linux 镜像，详见 [kube-proxy-bpf 模式](https://cloud.tencent.com/document/product/457/39238)）。**ipvs-bpf 是第三种独立模式，非 IPVS 的子项** |
| IsDualStack         | boolean | `false`      | VPC-CNI 模式下是否 IPv4/IPv6 双栈（仅 `NetworkType=VPC-CNI` 适用）                                                                                                                                                                                     |
| AsEnabled           | boolean | `false`      | 是否启用节点池弹性伸缩（AS）。**创建集群流程不支持开启此功能**，需集群创建后通过节点池 `ModifyClusterAsGroupAttribute` 开启                                                                                                                                                          |
| NodeNameType        | string  | `lan-ip`     | 节点名类型：`lan-ip`（内网 IP）/ `hostname`                                                                                                                                                                                                          |
| QGPUShareEnable     | boolean | `false`      | 是否开启 QGPU 共享（GPU 场景）                                                                                                                                                                                                                       |
| IsHighAvailability  | boolean | `true`       | 是否启用高可用模式（指导跨可用区资源打散等高可用策略，默认 `true`）                                                                                                                                                                                                      |
| SecurityModeConfig  | object  | 不设           | 安全模式配置：开启后下发 Gatekeeper 准入策略和网络隔离规则（`{Enabled:true,Namespaces:["..."]}`）                                                                                                                                                                   |
| ExtraArgs           | object  | 不设           | 集群自定义参数：`{KubeAPIServer:[...],KubeControllerManager:[...],KubeScheduler:[...]}`，可用的参数见 `tccli tke DescribeClusterAvailableExtraArgs`                                                                                                       |
| EtcdOverrideConfigs | list    | 不设           | 元数据拆分存储 Etcd 配置：把指定资源（Pods/Nodes/Deployments/Secrets 等）拆分到独立 Etcd 集群，提升大规模集群稳定性                                                                                                                                                            |

> `NetworkType=GR` 是后期 `AddClusterCIDR` 扩容 Pod 网段的前置——VPC 模式（ENI）集群不支持 `AddClusterCIDR`。见 [配置集群属性与运行时](configure.md#步骤-5扩容容器网段)。

## 跨字段约束 {#跨字段约束}

> ⚠️ 控制台创建流程按字段组合校验，传互斥组合会被服务端拦或创建中途失败。

### 网络类型 × 各字段互斥矩阵

| 字段                                   | `NetworkType=GR` | `NetworkType=VPC-CNI`                    | `NetworkType=CiliumOverlay`              |
| ------------------------------------ | ---------------- | ---------------------------------------- | ---------------------------------------- |
| `IsDualStack`（IPv4/IPv6 双栈）          | ❌ 不支持            | ✅ 可设 `true`（须前序双栈 VPC，见下"集群 IP 类型决策树"）   | ❌ 不支持                                    |
| `VpcCniType`（共享/独立网卡）                | ❌ 不适用            | ✅ `tke-route-eni`/`tke-direct-eni` | ❌ 不适用                                    |
| `EniSubnetIds`/`ClaimExpiredSeconds` | ❌ 不适用            | ✅ ENI 子网相关                               | ❌ 不适用                                    |
| `CiliumMode`/`DataPlaneV2`           | ❌ 不适用            | ❌ 不适用                                    | ✅ Cilium 数据面相关                           |
| `SubnetId`（控制面子网）                    | 不强制              | 不强制                                      | ✅ **条件必填**，须预留 ≥2 IP（TKE 取 2 IP 建内网 CLB） |
| 后期 `AddClusterCIDR` 扩 Pod 网段         | ✅ 支持             | ❌ 不支持                                    | ❌ 不支持                                    |

### kube-proxy 转发模式 × IPVS × KubeProxyMode 互斥 {#kube-proxy-转发模式-×-ipvs-×-kubeproxymode-互斥}

`IPVS` 与 `KubeProxyMode` 三模式互斥（以 `KubeProxyMode` 字段说明为准）：

| 模式               | `IPVS` | `KubeProxyMode`  | 前置条件                                                                      |
| ---------------- | ------ | ---------------- | ------------------------------------------------------------------------- |
| **iptables**（默认） | 不设     | 不设               | —                                                                         |
| **ipvs**         | `true` | 不设               | —                                                                         |
| **ipvs-bpf**     | —      | `kube-proxy-bpf` | ① 集群版本 ≥1.14；② 系统镜像须 **Tencent Linux 2.4**（`ClusterOs` 传对应镜像 ID，否则创建中途失败） |

> `kube-proxy-bpf` 是第三种独立模式，非 IPVS 的子项。`DataPlaneV2=true`（cilium 替代 kube-proxy）语义上与 `KubeProxyMode` 互斥——CiliumOverlay 走 Cilium 数据面，不再走 kube-proxy 三模式。
>
> ⚠️ **固定约束**：IPVS **仅新建时可开**，开启后**不可关闭**；iptables/ipvs 一经选择不支持更改。勿在集群内手动混用 IPVS 与 iptables。

### DataPlaneV2 × 镜像白名单

> ⚠️ `DataPlaneV2=true` 仅以下镜像支持（传非白名单 `ClusterOs` 会创建中途失败）：

| 镜像                               | ID             |
| -------------------------------- | -------------- |
| TencentOS Server 3.2 with Driver | `img-1tmhysjj` |
| TencentOS Server 3.2 For QAT     | `img-ojpp1hh1` |
| TencentOS Server 3.2 (Final)     | `img-9qrfy1xt` |
| TencentOS Server 3.2 (CGroupV2)  | `img-j65jjfpx` |
| TencentOS Server 3.1 (TK4) UEFI  | `img-39ywauzd` |
| TencentOS Server 3.1 (TK4) SPR   | `img-ppgc11rl` |
| TencentOS Server 3.1 (TK4)       | `img-eb30mz89` |
| TencentOS Server 3.1 (CGroupV2)  | `img-2kulq18f` |

传 `DataPlaneV2=true` + 非白名单 `ClusterOs` → 控制台报"当前操作系统暂不支持开启 Dataplane v2"；走 API 可能到创建中途才失败。反查某 ID：`tccli cvm DescribeImages --region <REGION> --Filters '[{"Name":"image-id","Values":["<ID>"]}]'`。

### 集群 IP 类型决策树 {#集群-ip-类型决策树}

> **控制台维度**：控制台「集群 IP 类型」一步 = tccli 多字段组合 `IsDualStack` + `NetworkType` + `VpcCniType`（双栈仅 `NetworkType=VPC-CNI` 适用；`VpcCniType` 决定共享/独立网卡）。下面决策树把控制台这一个决策步映射到 tccli 字段组合。

控制台"集群 IP 类型"步（IPv4-only vs IPv4/IPv6 双栈）对应 tccli 多字段组合：

```
需要 IPv4/IPv6 双栈 Pod?
├─ 否 → NetworkType=GR 或 VPC-CNI，IsDualStack 不设（默认 false）
└─ 是 → 须同时满足:
        ① NetworkType=VPC-CNI（双栈仅 VPC-CNI 适用，GR/Cilium 不支持）
        ② IsDualStack=true
        ③ 前序 VPC 须已开 IPv6（见 准备 VPC 与子网 — 双栈 VPC 创建）
        ④ 前序子网须已分配 IPv6 CIDR
```

> 双栈集群依赖前序资源：`IsDualStack=true` 须前序 VPC 已 `AssignIpv6CidrBlock` + 子网已 `AssignIpv6SubnetCidrBlock`。前置缺失会创建中途失败。完整双栈 VPC 创建步骤见 [准备 VPC 与子网 — 创建双栈 VPC](../../getting-started/prepare-vpc.md#create-dualstack-vpc)。

## 操作步骤

> ⚠️ **高危操作**：Region 选错不可迁移；NetworkType 创建后不可切换；IPVS 开启后不可关闭；删除保护未开则集群可被直接删除。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

> ⚠️ **本文创建的是空集群（控制面）**：`CreateCluster` 后 `ClusterState=Running` 但 `ClusterRunningNodeNum=0`，**无法运行 Pod**。
>
> "可运行 Pod 的集群"完整路径 = 本步（空集群）→ [创建节点池](../nodes/nodepool-create.md)（加工作节点）→ [开端点](../networking/endpoints.md) → [kubectl 连通](../security/auth.md)。
>
> 本步约 5-10 分钟，完成后须立即进入 [创建节点池](../nodes/nodepool-create.md) 加节点，否则集群无工作负载仍计管理费。完整生命周期见 [集群生命周期故事线](index.md#集群生命周期故事线)。

### 步骤 1：决策 — 选集群类型 {#步骤-1决策-选集群类型}

#### 为什么选 MANAGED_CLUSTER

- **托管 vs 独立**: 托管集群的 Master 由腾讯云负责 (HA、升级、备份)；独立集群需要自己维护 Master 节点
- **成本差异**: 托管集群收取集群管理费 (L5 约 ¥0.4/小时)；独立集群不收取管理费但需支付 Master 的 CVM 费用
- **默认推荐**: `MANAGED_CLUSTER`（独立模式**已停止新建**）
- **可修改**： 不能。集群类型创建后无法切换。存量独立集群须持续维护 Master

#### 创建模式决策 — 4 条路径 {#创建模式决策-4-条路径}

`CreateCluster` 的顶层入参有 4 个嵌套组合，对应 4 条创建路径。选哪条取决于"建集群时是否同时建节点 / 用已有 CVM / 装组件"。

| 路径                    | 触发条件                                 | 关键参数                                                                     | 后续                                                 |
| --------------------- | ------------------------------------ | ------------------------------------------------------------------------ | -------------------------------------------------- |
| **A. 空集群**（默认，下方步骤 2） | 先建控制面，节点后加                           | 仅 `ClusterCIDRSettings`/`ClusterBasicSettings`/`ClusterAdvancedSettings` | [创建节点池](../nodes/nodepool-create.md)               |
| **B. 一步建带节点**         | 建集群同时新建 CVM 节点                       | 加 `RunInstancesForNode`（透传 CVM `RunInstances` 全参数）                       | 集群就绪即可运行 Pod                                       |
| **C. 导入已有 CVM**       | 把现有 CVM 转成集群节点                       | 加 `ExistedInstancesForNode`（CVM 须已存在且可重装）                                | 集群就绪即可运行 Pod                                       |
| **D. 创建时装组件**         | 建集群同时装 CiliumOverlay/RuntimeConfig 等 | 加 `ExtensionAddons`（组件参数对象数组）                                            | 见 [CiliumOverlay](../networking/cilium-overlay.md) |

> **判据**: 路径 A 分步可控，失败仅回退控制面；路径 B/C 单步完成但失败须连集群带节点一起排查；路径 D 与网络模型绑定（CiliumOverlay 创建时定型）。首次部署用 A，批量部署可用 B/C。

> **4 路径共用同一** `CreateCluster` **Action**——返回结构一致（`ClusterId`/`RequestId`），区别仅在顶层嵌套入参组合。形态差异：`RunInstancesPara` = **Array of String**（每个元素是 CVM `RunInstances` 的 JSON 字符串）；`ExistedInstancesPara` = **对象**；`ExtensionAddons[].AddonParam` = 字符串化 JSON。expected 与路径 A 同构。B/C 因会真实创建/重装 CVM（计费+副作用），命令块用占位符示参，调用前先核 CVM 机型/镜像/子网库存（见 [创建节点池 — 准备工作（机型查询）](../nodes/nodepool-create.md#准备工作)）。

##### 路径 B：一步建带节点（RunInstancesForNode）

`RunInstancesForNode` 透传 CVM `RunInstances` 全部参数（机型/镜像/磁盘/网络/登录），集群创建时同时创建节点。

```bash
tccli tke CreateCluster --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<CLUSTER_NAME>","ClusterVersion":"1.34.1","VpcId":"<VPC_ID>"}' \
  --ClusterCIDRSettings '{"ClusterCIDR":"172.16.0.0/16","ServiceCIDR":"10.96.0.0/20"}' \
  --RunInstancesForNode '[{"NodeRole":"WORKER","RunInstancesPara":["{\"InstanceType\":\"S5.MEDIUM4\",\"ImageId\":\"<IMAGE_ID>\",\"SubnetId\":\"<SUBNET_ID>\",\"InstanceCount\":2,\"SecurityGroupIds\":[\"<SG_ID>\"]}"]}]'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }（含节点正在启动）
```

> ⚠️ `RunInstancesPara` 是 **Array of String**：每个元素是 CVM `RunInstances` 参数的 **JSON 字符串**（不是对象、也不是单个字符串字段）。与 [Master 运维](master-ops.md) 扩容形态一致。CVM 参数结构见 [共享字段](../reference/shared-fields.md#instanceadvancedsettings-节点高级设置) + CVM 文档；机型/镜像查询见 [创建节点池 — 准备工作](../nodes/nodepool-create.md#准备工作)。

##### 路径 C：导入已有 CVM（ExistedInstancesForNode）

把已存在的 CVM 重装系统并加入集群（CVM 数据会清空，须先备份）。

```bash
tccli tke CreateCluster --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<CLUSTER_NAME>","ClusterVersion":"1.34.1","VpcId":"<VPC_ID>"}' \
  --ClusterCIDRSettings '{"ClusterCIDR":"172.16.0.0/16","ServiceCIDR":"10.96.0.0/20"}' \
  --ExistedInstancesForNode '[{"NodeRole":"WORKER","ExistedInstancesPara":{"InstanceIds":["ins-xxx"],"LoginSettings":{"Password":"<PASSWORD>"},"EnhancedService":{}}}]'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }
```

> `ExistedInstancesPara` 是 **对象**（非 JSON 字符串）：`InstanceIds`/`LoginSettings`/`EnhancedService`/`SecurityGroupIds` 等为对象字段。`InstanceIds` 指向已存在 CVM，须与集群同 VPC 且可重装。登录/增强服务配置见 [共享字段](../reference/shared-fields.md#loginsettings-实例登录设置)。

##### 路径 D：创建时装组件（ExtensionAddons）

```bash
tccli tke CreateCluster --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<CLUSTER_NAME>","ClusterVersion":"1.34.1","ClusterOs":"tlinux3.1x86_64","VpcId":"<VPC_ID>","SubnetId":"<SUBNET_ID>"}' \
  --ClusterCIDRSettings '{"ClusterCIDR":"172.16.0.0/16","ServiceCIDR":"10.96.0.0/20"}' \
  --ClusterAdvancedSettings '{"NetworkType":"CiliumOverlay"}' \
  --ExtensionAddons '[{"AddonName":"CiliumOverlay","AddonParam":"{\"Enable\":true}"}]'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }
```

> `ExtensionAddons` 在创建时定型网络模型/运行时配置。CiliumOverlay 须 `ClusterOs=tlinux3.1x86_64` + `SubnetId`，且 AdvancedSettings **只传** `NetworkType`（禁 `CiliumMode`/`DataPlaneV2`/`IPVS=true`）——详见 [CiliumOverlay](../networking/cilium-overlay.md)。`InstanceDataDiskMountSettings`（自定义数据盘挂载）是同层独立组合，配置见 [共享字段](../reference/shared-fields.md#instanceadvancedsettings-节点高级设置)。

### 步骤 2：创建集群

`CreateCluster` 的 `ClusterType` + `ClusterCIDRSettings` 是 API 层仅有的两个必填项（缺则 exit 252），另需集群名/VPC/K8s 版本（业务必需，不传集群无法正常使用）。按场景**二选一**：A 最小化（测试/快速验证）或 B 增强（生产，容量规划+审计+运行时）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateCluster` 会建**两个集群**。集群创建后改配置（等级/标签/运行时/组件参数）用 `ModifyClusterAttribute` 等，见 [配置集群](configure.md)，**禁用第二次** `CreateCluster` **改配置**。

#### 选项 A：最小化（测试/快速验证）

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

| 占位符              | 含义        | 约束          | 如何获取                                        |
| ---------------- | --------- | ----------- | ------------------------------------------- |
| `<CLUSTER_NAME>` | 集群名称      | ≤60 字符，全局唯一 | 自定义（如 `prod-cluster`）                      |
| `<VPC_ID>`       | VPC 实例 ID | 必须存在        | `tccli vpc DescribeVpcs` → `VpcSet[].VpcId` |

> 命令中的 `ClusterCIDR`（如 `172.16.0.0/16`）与 `ServiceCIDR`（如 `10.96.0.0/20`）是字面量示例——按你的 VPC CIDR 选择不重叠的网段，不得与 VPC CIDR 冲突。

#### 选项 B：增强（生产，容量规划+审计+运行时）

在 A 的基础上加 Pod/Service 容量（`MaxNodePodNum`/`MaxClusterServiceNum`）、集群等级（`ClusterLevel`）、审计与运行时——**与 A 二选一，非在 A 之后执行**：

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

这启用了自定义 CIDR（适合多集群/VPC 互联场景）、审计日志、集群等级 L20（支持更多节点）。集群创建后调整这些配置用 [配置集群](configure.md)（`ModifyClusterAttribute` 等），非再调 `CreateCluster`。

### 步骤 3：验证

异步操作: 检查 ≥4 个维度确认集群可用。

```bash
# 用 --waiter 自动轮询集群到 Running（无需自行编写轮询循环；waiter 对所有 TKE Action 生效）
tccli tke DescribeClusterStatus --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]' \
  --waiter '{"expr":"ClusterStatusSet[0].ClusterState","to":"Running","timeout":600,"interval":10}'
# expected: 集群 Running 后返回，超时则报 ClientError

# 或手动轮询
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterStatusSet[0].ClusterState 为 Running（创建中为 Creating/Initializing，失败为 Idling/Abnormal 等，见状态机）
```

| 维度         | 命令                                                                       | 预期                                                 |
| ---------- | ------------------------------------------------------------------------ | -------------------------------------------------- |
| Status     | `DescribeClusterStatus`                                                  | `ClusterState: "Running"`                          |
| 集群属性       | `DescribeClusters --ClusterIds '["<ID>"]'`                               | ClusterName、ClusterVersion、VpcId 与创建参数一致           |
| 删除保护       | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterDeletionProtection` | `true`（布尔值，非 Enabled/Disabled）                     |
| kubeconfig | `DescribeClusterKubeconfig --ClusterId "<ID>"`                           | 返回有效 kubeconfig                                    |
| kubectl 连通 | `kubectl --kubeconfig kubeconfig.yaml get nodes`                         | exit 0（需先开端点，见 [管理端点](../networking/endpoints.md)） |
| 节点数        | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterRunningNodeNum`     | ≥0（空集群为 0，字段名是 ClusterRunningNodeNum 非 NodeCount）  |

> `--waiter` 的 `expr` 必须用 `DescribeClusterStatus` 响应字段名 `ClusterStatusSet[0].ClusterState`（非 `Clusters[0].ClusterStatus`）。`CreateCluster` 本身也可直接挂 `--waiter`，但 `CreateCluster` 响应无状态字段，需轮询 `DescribeClusterStatus`。

## 清理

> 本段只删除本文刚创建、要放弃的集群。完整删除（决策、级联、残留、退费）见 [删除集群](delete.md)。
>
> 适用：创建中途失败、未达 `Running` 的集群，或创建成功后立即销毁。与 [§收尾确认](#收尾确认) 分工不同：清理 = 资源应消失；收尾确认 = 资源应保留且创建结果符合预期。

```bash
# 若已开删除保护则先关（未开可跳过；CAM 限制见 delete.md）
tccli tke DisableClusterDeletionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0（或本就未开保护）

tccli tke DeleteCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --InstanceDeleteMode terminate
# expected: exit 0

tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "TotalCount": 0, "Clusters": [] } → 主资源已删；关联残留见 delete.md
```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象                                                                                       | 诊断                                                          | 根因                                                                        | 修复                                                                 |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `AuthFailure.SecretIdNotFound`                                                           | `tccli tke DescribeRegions`                                 | 凭证未配置或已过期                                                                 | 见 [配置凭证](../../getting-started/credentials.md) 重新配置                |
| `InvalidParameterValue.ClusterName`                                                      | 检查名称格式                                                      | 名称含特殊字符或超长                                                                | 使用 ≤60 字符的字母数字连字符                                                  |
| `FailedOperation.DbRecordNotFound`（消息含 `CompId not found ... k8s <版本> record not found`） | `tccli tke DescribeVersions --region <REGION>` 查可用版本        | 版本号不存在或未用完整三段式（如 `1.30` 而非 `1.30.0`）                                      | 用 `DescribeVersions` 返回的完整版本号（如 `1.34.1`）                          |
| `FailedOperation.Param`（消息含 `service ip number is N, but the ClusterServiceNum paramter is M`） | 核对 `ServiceCIDR` 掩码与 `MaxClusterServiceNum` | `MaxClusterServiceNum` 小于 `ServiceCIDR` 可容纳的 Service IP 数（如 `/17` → 32768，传 256 会拒） | 令 `MaxClusterServiceNum` ≥ CIDR 容量，或缩小 `ServiceCIDR`（`ServiceCIDR` 为 `/17` 时 `MaxClusterServiceNum` 须 ≥ `32768`） |
| `ResourceNotFound.VpcId` / `FailedOperation.CamNoAuth`（消息含 `tke:CreateCluster` + 资源 qcs） | `tccli vpc DescribeVpcs`                                    | VPC ID 不存在、跨账号，或 CAM 按资源 ARN 拒绝（假 VPC ID 也可能先被 CAM 拦成 CamNoAuth） | 确认 VPC 在该地域存在且 CAM 允许对该资源 `tke:CreateCluster`；假 ID 勿当参数格式问题 |
| `LimitExceeded.ClusterQuota` / `InternalError.QuotaMaxClsLimit`                          | `tccli tke DescribeClusters`                                | 单地域集群数达上限（默认 **20**）                                                    | 删除闲置集群或提交工单提升配额；见 [配额](../reference/quotas.md)                    |
| `InvalidParameterValue.ClusterCIDR`                                                      | CIDR 格式: `A.B.C.D/掩码`                                       | CIDR 格式不合法                                                                | 用合法 CIDR 格式                                                        |
| `InvalidParameter.CidrConflictWithVpcCidr`                                               | `tccli vpc DescribeVpcs --VpcIds '["<VPC_ID>"]'` 查 VPC CIDR | `ClusterCIDR` 与 VPC CIDR 重叠（如 VPC 是 `172.16.0.0/12` 则 `172.24.x.x/16` 冲突） | 换一个与 VPC CIDR 不重叠的网段（如 VPC 用 `172.16/12` 时容器网段可用 `192.168.0.0/16`） |
| 缺 `--ClusterType` / `--ClusterCIDRSettings`（tccli exit 252，无 `Error.Code`） | 查命令是否传对应必填项 | CLI/API 必填；缺则本地拒，不进业务校验 | 补 `--ClusterType MANAGED_CLUSTER` 与 `--ClusterCIDRSettings '{...}'` |
| `FailedOperation.Param`（`INDEPENDENT_CLUSTER`，消息含 `not has master NodeRole`） | 查 `ClusterType` | 独立集群已停止新建；空 `CreateCluster` 无 Master 节点角色入参 | 新建用 `MANAGED_CLUSTER`；存量独立集群运维见 [Master 运维](master-ops.md) |

### 命令成功但状态不对（exit = 0）

| 现象                                             | 诊断                                           | 根因                        | 修复                                              |
| ---------------------------------------------- | -------------------------------------------- | ------------------------- | ----------------------------------------------- |
| 集群卡在 `Creating` > 30 分钟                        | `tccli tke DescribeClusterStatus`            | 可用区资源不足或 VPC 配置冲突         | 1. 等待 10 分钟观察 2. 若超 30 分钟，删除重建并换可用区             |
| 状态为 `Running` 但 `DescribeClusterKubeconfig` 失败 | 检查返回的 kubeconfig 内容                          | 集群就绪延迟 (API Server 未完全就绪) | 等待 1-2 分钟后重试                                    |
| 状态为 `Abnormal`                                 | `tccli tke DescribeClusterStatus`            | Master 组件异常 (罕见)          | `DeleteCluster` → 重建。若重建仍异常，提交工单                |
| 删除保护无法关闭                                       | `tccli tke DisableClusterDeletionProtection` | CAM 权限不足                  | 确认账号有 `tke:DisableClusterDeletionProtection` 权限 |

## 收尾确认 {#收尾确认}

```bash
# 创建结果核对：集群 Running + 删除保护已开 + 空集群(0 节点)
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].{state:ClusterState,running:ClusterRunningNodeNum,protect:ClusterDeletionProtection}"
# expected: state=Running, running=0(空集群), protect=true

# 可选：凭证文件可拉取（不等于本机 kubectl 已通）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --filter "Kubeconfig" --output text | head -1
# expected: apiVersion: v1（仅说明可写出 YAML；默认无访问端点，本机 kubectl 通常仍失败）
```

> 集群 `Running` + `protect=true` + kubeconfig 可拉取 = **控制面创建完成**。  
> **不等于**本机/公网 CI 已能 `kubectl`：须先 [加节点](../nodes/nodepool-create.md)（无 worker 不能开公网端点），再 [管理端点](../networking/endpoints.md)（公网端点 `Created` + `SecurityPolicies` + 必要时把 kubeconfig `server` 改为 `ClusterExternalEndpoint`）。  
> `running=0` 时不能运行业务 Pod，空集群仍计管理费——立即进入下一步。

## 下一步

> 集群 `Running` 只是第 1 步（空控制面）。若目标是「本机可操作、可运行 Pod」：

- **[创建节点池](../nodes/nodepool-create.md)** / [新建 CVM 作节点](../nodes/instance-ops.md) — **必做**：无 worker 不能运行 Pod，也不能开公网端点
- **[管理端点](../networking/endpoints.md)** — **本机/公网 CI 必做**：`CreateClusterEndpoint` → ACL → `ClusterExternalEndpoint` 改写 kubeconfig → `kubectl get --raw=/healthz`
- [获取 kubeconfig](../security/auth.md) — 证书/凭证面（须配合端点）
- [查询集群](query.md) — filter + JMESPath
- [应用发布](../releases/index.md) — 集群与访问就绪后部署应用
- [独立集群 Master 运维](master-ops.md) — 独立集群场景

