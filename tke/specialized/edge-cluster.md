---
doc_type: How-to
subtype: 6A
fused: true
---
# 边缘集群 (TKEEdge)

> 创建和管理 TKE 边缘集群 —— 适合边缘计算场景的轻量级 K8s 集群。
> 控制台: [容器服务 - 边缘集群](https://console.cloud.tencent.com/tke2/edge)
>
> 官方文档：[边缘集群下线公告](https://cloud.tencent.com/document/product/457/108732) · [边缘集群迁移至标准集群](https://cloud.tencent.com/document/product/457/110447)
>
> 配额：ECM / Edge CVM 实例配额以 ECM 产品配额为准，TKE 侧无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

> ⚠️ **已下线（禁止新建）**：TKE-Edge 边缘容器服务已于 **2024-08-28** 下线；**创建入口已封闭**。边缘/IDC 场景改用 [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)（标准集群 + 注册节点）。存量边缘集群可继续运维/迁移，见下方操作；**禁止**再调 `CreateTKEEdgeCluster` 创建新集群。
>
> ⚠️ **高危操作**：产品已下线，禁止新建边缘集群（`CreateTKEEdgeCluster`）；存量集群请在迁移窗口内完成迁移，超期可能无法迁移。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- **新建边缘场景** → **禁止用本文创建**：改走标准集群 + [注册节点](../nodes/registered-nodes/overview.md)（注册节点公网版）
- 存量：`DescribeTKEEdgeClusters` 已有边缘集群，需查询状态/凭证/注册脚本/升级 — 用本文运维段
- `DescribeTKEEdgeClusterStatus` → `ClusterState` 非 `Running`，或节点未注册 — 看 [故障恢复](#故障恢复)
- 迁移存量边缘集群到注册节点公网版 — 先读官方迁移前置，再按注册节点流程接入

## 准备工作

- 已安装 TCCLI 并配置凭证 (见 [配置凭证](../../getting-started/credentials.md))
- 已确认地域支持边缘集群 (DescribeRegions 查地域)
- 边缘集群相关资源配额充足



## 概述

边缘集群是 TKE 为边缘计算场景提供的 K8s 集群类型，运行在边缘节点上。与标准集群的主要区别:

| 特性 | 边缘集群 (TKEEdge) | 标准集群 (TKE) |
|------|:---:|:---:|
| 节点位置 | 边缘/IDC/门店 | 腾讯云 CVM |
| 网络 | 公网/弱网自适应 | VPC 内网 |
| Master | 托管（云端） | 托管或独立 |
| 适用场景 | IoT、边缘 AI、CDN | 标准 Web 服务 |

## 关键操作

### 创建边缘集群（存量对照 / 禁止新建）

> ⚠️ **禁止新建**：产品已下线，创建入口已封闭；下列命令仅供存量脚本对照与参数名核对。新边缘/IDC 需求改走标准集群 + [注册节点公网版](https://cloud.tencent.com/document/product/457/57916) / [注册节点](../nodes/registered-nodes/overview.md)。调用可能返回产品侧拒绝或 `UnsupportedRegion`。

```bash
tccli tke CreateTKEEdgeCluster \
  --region <EDGE_REGION> \
  --ClusterName "<NAME>-edge" \
  --K8SVersion "1.28.3" \
  --VpcId "<VPC_ID>" \
  --PodCIDR "<POD_CIDR>" \
  --ServiceCIDR "<SERVICE_CIDR>"
# expected: 存量对照可能返回 { "ClusterId": "cls-xxxxxxxx" }；下线后以实际 Error.Code 为准；UnsupportedRegion → 换 <EDGE_REGION>（如 ap-beijing）
```

> ⚠️ **参数名核对**: `CreateTKEEdgeCluster` 顶层参数是 `K8SVersion`（非 `ClusterVersion`）、`VpcId`（无 `SubnetId`，边缘集群节点通过 VPC 接入，不指定子网）。完整入参以 `tccli tke CreateTKEEdgeCluster help --detail` 为准。`ap-guangzhou` 对 Edge Action 返回 `UnsupportedRegion`，存量运维前先用 `DescribeTKEEdgeClusters --region <候选>` 探测支持地域。

### 查询边缘集群

```bash
# 列出所有边缘集群
# 注意：ap-guangzhou 对 Edge Action 返回 UnsupportedRegion；实际可达地域含 ap-beijing（须用 DescribeTKEEdgeClusters 探测）
tccli tke DescribeTKEEdgeClusters --region <EDGE_REGION>
# expected: { "TotalCount": ..., "Clusters": [...] }；UnsupportedRegion → 换 Edge 支持地域

# 查询状态
tccli tke DescribeTKEEdgeClusterStatus --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>"
# expected: ClusterState: "Running"

# 获取凭证
tccli tke DescribeTKEEdgeClusterCredential --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回 kubeconfig
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<EDGE_REGION>` | Edge 支持的地域 | 非任意 TKE 地域；`ap-guangzhou` 报 `UnsupportedRegion` | `tccli tke DescribeTKEEdgeClusters --region <候选>` 无 `UnsupportedRegion` 即支持（如 `ap-beijing`） |

### 获取注册脚本 (添加边缘节点)

边缘节点通过脚本注册到集群，而非通过 CVM:

```bash
# Interface 为 API 必填：边缘节点上 kubelet 向 apiserver 注册使用的网卡名（如 eth0）
tccli tke DescribeTKEEdgeScript --region <EDGE_REGION> \
  --ClusterId "<CLUSTER_ID>" --Interface "<INTERFACE>"
# expected: 返回 Link/Token/Command（安装脚本）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<INTERFACE>` | 边缘节点网卡名 | API 必填；常见 `eth0` | 边缘节点 `ip -o link` / 运维约定 |

在边缘节点上执行返回的脚本来注册节点。

### 升级边缘集群

```bash
# 1. 查询升级信息（EdgeVersion 为 API 必填：目标 TKEEdge 版本）
tccli tke DescribeEdgeClusterUpgradeInfo --region <EDGE_REGION> \
  --ClusterId "<CLUSTER_ID>" --EdgeVersion "<EDGE_VERSION>"
# expected: 返回 EdgeVersionCurrent / ClusterUpgradeStatus 等

# 2. 执行升级
tccli tke UpdateEdgeClusterVersion --region <EDGE_REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --Version "<TARGET_VERSION>"
# expected: exit 0
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<EDGE_VERSION>` | 目标 TKEEdge 版本 | `DescribeEdgeClusterUpgradeInfo` 必填 | `DescribeAvailableTKEEdgeVersion` / 运维指定版本号 |

### 边缘集群日志

```bash
# 创建日志配置
tccli tke CreateEdgeLogConfig --region <EDGE_REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --LogConfig "<LOG_CONFIG_JSON>"
# expected: exit 0, 返回 RequestId

# 查询日志开关（入参是 --ClusterIds 数组，非 --ClusterId）
tccli tke DescribeEdgeLogSwitches --region <EDGE_REGION> \
  --ClusterIds '["<CLUSTER_ID>"]'
# expected: 返回各集群日志开关状态列表

# 安装日志 Agent
tccli tke InstallEdgeLogAgent --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

## 验证

```bash
# 验证存量边缘集群可用（须用 DescribeTKEEdgeClusters，非 DescribeClusters）
tccli tke DescribeTKEEdgeClusters --region <EDGE_REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{id:ClusterId,state:ClusterStatus,name:ClusterName}"
# expected: state=Running, id/name 与存量集群一致
```

> 边缘集群 state=Running = 存量集群可用, 可进入 [关键操作](#关键操作) 运维。

## 故障恢复

| 症状 | 先查 | 处理 |
|:-----|:-----|:-----|
| `DescribeTKEEdgeClusterStatus` → `ClusterState` 非 `Running` | `DescribeTKEEdgeClusters` + `DescribeTKEEdgeClusterStatus` | 等过渡态结束；长期非 Running 按官方迁移/工单，**不要**新建 Edge 集群顶替 |
| 边缘节点未出现 / 注册失败 | `DescribeTKEEdgeScript` 重取脚本；`DescribeEdgeClusterInstances` 看节点 | 核对 `<INTERFACE>` 网卡名与节点出网；弱网重跑注册脚本 |
| `UnsupportedRegion` | 当前 `--region` 是否 Edge 可达 | 换 `<EDGE_REGION>`（如 `ap-beijing`）；`ap-guangzhou` 对多数 Edge Action 不支持 |
| 需下线/迁走业务 | 官方 [迁移至标准集群](https://cloud.tencent.com/document/product/457/110447) | 标准集群 + [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)；迁完再 `DeleteTKEEdgeCluster` |

```bash
# 状态非 Running 时先看 ClusterState
tccli tke DescribeTKEEdgeClusterStatus --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>"
# expected: ClusterState 字段；非 Running 勿当新建成功

# 节点是否已注册（须 Edge 地域）
tccli tke DescribeEdgeClusterInstances --ClusterID "<CLUSTER_ID>" --region <EDGE_REGION> \
  --Offset 0 --Limit 20
# expected: TotalCount≥1 且 InstanceInfoSet 有节点；0 → 重取注册脚本
```

---

## 清理

```bash
# 1. 删除边缘集群
tccli tke DeleteTKEEdgeCluster --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 2. 验证
tccli tke DescribeTKEEdgeClusters --region <EDGE_REGION>
# expected: TotalCount 减少，目标集群不在列表中
```

## API 参考

本篇覆盖边缘集群相关 **26** 个 Action（应用转发类 ForwardTKEEdgeApplicationRequestV3 不在正文主流程演示）：

| 分类 | API | 说明 |
|------|-----|------|
| 生命周期 | `CreateTKEEdgeCluster`（禁止新建） / `DeleteTKEEdgeCluster` / `UpdateTKEEdgeCluster` | 创建对照/删除/更新 |
| 查询 | `DescribeTKEEdgeClusters` / `DescribeTKEEdgeClusterStatus` | 列表/状态 |
| 凭证 | `DescribeTKEEdgeClusterCredential` / `DescribeTKEEdgeExternalKubeconfig` | kubeconfig |
| 节点 | `DescribeEdgeClusterInstances` / `DescribeTKEEdgeScript` / `DeleteEdgeClusterInstances` | 节点/注册脚本/删节点 |
| Edge CVM | `CreateEdgeCVMInstances` / `DescribeEdgeCVMInstances` / `DeleteEdgeCVMInstances` | 边缘 CVM 实例 |
| ECM | `CreateECMInstances` / `DescribeECMInstances` / `DeleteECMInstances` | 边缘计算模块实例 |
| 升级 | `DescribeEdgeClusterUpgradeInfo` / `UpdateEdgeClusterVersion` / `DescribeAvailableTKEEdgeVersion` | 版本管理 |
| 日志 | `CreateEdgeLogConfig` / `DescribeEdgeLogSwitches` / `InstallEdgeLogAgent` / `UninstallEdgeLogAgent` | 日志采集 |
| 其他 | `CheckEdgeClusterCIDR` | CIDR 冲突检查 |
| 参数 | `DescribeEdgeAvailableExtraArgs` / `DescribeEdgeClusterExtraArgs` | 集群参数 |

## 集群更新与诊断

> 更新边缘集群属性、查可用版本与额外参数、外部 kubeconfig、CIDR 冲突检查、日志 Agent 卸载。参数见各 Action 的 `help --detail`（注意 Edge 域 `ClusterID` 大写 vs `ClusterId` 小写不一致）。
>
> ⚠️ Edge 域多数 action 在 ap-guangzhou 返回 `UnsupportedRegion`（`DescribeEdgeClusterExtraArgs`/`CheckEdgeClusterCIDR`/`DescribeEdgeClusterInstances`）或 CAM 拦截（`DescribeEdgeAvailableExtraArgs`/`UpdateTKEEdgeCluster`/`UninstallEdgeLogAgent` 返回 `UnauthorizedOperation`/`AuthFailure.UnauthorizedOperation`）。仅 `DescribeAvailableTKEEdgeVersion` 可用（但 action 已废弃，edge 产品已下线）。

```bash
# 更新边缘集群属性（ClusterId 小写 + ClusterName/ClusterDesc/PodCIDR 等覆盖式）
tccli tke UpdateTKEEdgeCluster --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --ClusterName "<NEW_NAME>" --ClusterDesc "<NEW_DESC>"
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后 exit 0

# 查询可升级到的边缘集群版本（无必填入参；action 已废弃，edge 产品下线）
tccli tke DescribeAvailableTKEEdgeVersion --region ap-guangzhou
# expected: exit 0，返回 Versions[]+EdgeVersionLatest+EdgeVersionCurrent（版本如 1.20.6/1.22.5/1.18.2）

# 查询集群已生效的额外参数（ClusterId 定位）
tccli tke DescribeEdgeClusterExtraArgs --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected:  UnsupportedRegion（ap-guangzhou 不支持）；Edge 地域可用时返回 ClusterExtraArgs

# 查询某版本可用的额外参数（ClusterVersion 定位，建集群/升级前用）
tccli tke DescribeEdgeAvailableExtraArgs --region ap-guangzhou --ClusterVersion "<VERSION>"
# expected: CAM 拦截 UnauthorizedOperation；授权后返回 ClusterVersion+AvailableExtraArgs

# 获取外部 kubeconfig（ClusterId 定位，区别于 DescribeTKEEdgeClusterCredential）
tccli tke DescribeTKEEdgeExternalKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回 UnknownParameter: failed to connect to user cluster（非边缘集群连不上）；边缘集群可用时返回 Kubeconfig

# CIDR 冲突检查（建集群前用，VpcId + PodCIDR + ServiceCIDR 三参数）
tccli tke CheckEdgeClusterCIDR --region ap-guangzhou \
  --VpcId "<VPC_ID>" --PodCIDR "<POD_CIDR>" --ServiceCIDR "<SERVICE_CIDR>"
# expected:  UnsupportedRegion（ap-guangzhou 不支持）；Edge 地域可用时返回 ConflictCode+ConflictMsg

# 卸载边缘日志 Agent（ClusterId 定位，停止边缘日志采集）
tccli tke UninstallEdgeLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后 exit 0
```

```bash
# 查询边缘集群节点列表（注意 ClusterID 大写 + Filters/Offset/Limit）
tccli tke DescribeEdgeClusterInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Offset 0 --Limit 20
# expected:  UnsupportedRegion（ap-guangzhou 不支持，需 Edge 地域）；可用时返回 TotalCount+InstanceInfoSet
```

> ⚠️ `DescribeEdgeClusterInstances` 用大写 `ClusterID`（同 `CreateEdgeCVMInstances`/`DescribeEdgeCVMInstances`），区别于 `UpdateTKEEdgeCluster`/`DescribeAvailableTKEEdgeVersion` 等的小写 `ClusterId`——Edge 域大小写不一致是真实契约，切换接口前用 `--generate-cli-skeleton` 核对。`CheckEdgeClusterCIDR` 不绑集群（建集群前用），需 `VpcId`+`PodCIDR`+`ServiceCIDR` 三参数。

## Edge 节点实例管理

> Edge 集群的 CVM 节点与 ECM（边缘计算模块）实例管理。⚠️ 这些 Action 用 `--ClusterID`（大写 ID），区别于其他 TKE 接口的 `--ClusterId`。

### Edge CVM 实例

```bash
# 创建 Edge CVM 实例 (RunInstancePara 透传 CVM JSON, CvmRegion/CvmCount)
tccli tke CreateEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --RunInstancePara "<CVM_JSON>" --CvmRegion "<CVM_REGION>" --CvmCount 1
# expected: exit 0

# 查询 Edge CVM 实例 (Filters 过滤)
tccli tke DescribeEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Filters '[{"Name":"zone","Values":["<ZONE>"]}]'
# expected: exit 0, 实例列表

# 删除 Edge CVM 实例 (CvmIdSet[] 批量)
tccli tke DeleteEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> --CvmIdSet '["<CVM_ID>"]'
# expected: exit 0

# 删除 Edge 集群节点 (InstanceIds[], 注意此接口用 ClusterId 小写)
tccli tke DeleteEdgeClusterInstances --ClusterId "<CLUSTER_ID>" --region <REGION> --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0
```

> `CreateEdgeCVMInstances` 用 `ClusterID`（大写）+ `CvmRegion`（CVM 地域）+ `CvmCount`。`DeleteEdgeClusterInstances` 用 `ClusterId`（小写）+ `InstanceIds[]`——同域大小写不一致，切换接口时核对。

### ECM 实例（边缘计算模块）

```bash
# 创建 ECM 实例 (ModuleId + ZoneInstanceCountISPSet[] 按可用区/ISP 分配)
tccli tke CreateECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --ModuleId "<MODULE_ID>" \
  --ZoneInstanceCountISPSet '[{"Zone":"<ZONE>","InstanceCount":1,"ISP":"<ISP>"}]'
# expected: exit 0

# 查询 ECM 实例 (Filters 过滤)
tccli tke DescribeECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Filters '[{"Name":"zone","Values":["<ZONE>"]}]'
# expected: exit 0

# 删除 ECM 实例 (EcmIdSet[] 批量)
tccli tke DeleteECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> --EcmIdSet '["<ECM_ID>"]'
# expected: exit 0
```

> ECM 用 `ModuleId`（边缘计算模块 ID）+ `ZoneInstanceCountISPSet[]`（按可用区+ISP 运营商分配）。`EcmIdSet[]`/`CvmIdSet[]` 是实例 ID 数组。

## 收尾确认

```bash
# 集群已 Running（上文已查状态，此处端到端核节点注册 + kubeconfig 可拉取）
tccli tke DescribeTKEEdgeClusters --region <EDGE_REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{state:ClusterStatus,name:ClusterName}"
# expected: state=Running

# 端到端：边缘节点注册成功（上文已查集群状态，此处查节点真在线）
# 注意：DescribeEdgeClusterInstances 在 ap-guangzhou 返回 UnsupportedRegion，须在 <EDGE_REGION> 执行
tccli tke DescribeEdgeClusterInstances --ClusterID "<CLUSTER_ID>" --region <EDGE_REGION> \
  --Offset 0 --Limit 20
# expected: TotalCount ≥1，InstanceInfoSet 含已注册边缘节点

# 下一步前置：kubeconfig 可拉取（进部署应用前须能连通集群）
tccli tke DescribeTKEEdgeClusterCredential --region <EDGE_REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text | head -1
# expected: apiVersion: v1 → 边缘集群闭环完成
```

> 集群 Running + 边缘节点注册在线 + kubeconfig 可拉取 = 端到端闭环。上文验证段查集群状态，此处确认边缘节点真注册（业务可用性，Edge 地域须实际核实）+ kubeconfig 可连通集群是进下一阶段（部署应用）的前置。

---

## 下一步

- [EKS / 容器实例](eks-cluster.md) — 存量 EKS 集群与容器实例（对比边缘）
- [虚拟节点 (超级节点)](../nodes/virtual-nodes.md) — 标准集群内免 CVM 容量（先 CreateCluster）
- [专用工作负载概览](index.md) — 边缘 / EKS / 虚拟节点选型
- [标准集群概览](../clusters/index.md) — 对比标准集群
