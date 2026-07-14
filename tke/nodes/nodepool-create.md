---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建节点池

> 给集群添加工作节点。原生节点池 (Native) —— 基于 2022-05-01 新版 API 的强类型 `Native` 抽象，支持原地升级与细粒度生命周期管理。
> 控制台: [容器服务 - 节点池](https://console.cloud.tencent.com/tke2/nodepool)
>
> **API 版本**: 本文档命令走 **2022-05-01**（官方当前版本）。节点池的 Native 强类型抽象仅新版提供，所有命令显式带 `--version`。完整版本选择见 [API 版本选择](../index.md#api-版本选择)。
>
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [节点池概述](https://cloud.tencent.com/document/product/457/104096) · [超级节点资源规格](https://cloud.tencent.com/document/product/457/39808)
>
> 配额：节点池上限 20、每节点池节点数受 ASG MaxSize 限制、单集群节点 5000。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：子网选错不可改、OS 镜像影响节点运行时行为、删除保护未开致节点池可被误删。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- DescribeClusterStatus 返回 ClusterState=Running 且需添加工作节点（Pod 待调度到节点）— 从 [步骤 1 决策](#步骤-1决策-—-选节点池类型与-api-版本) 开始
- 你要用 2022-05-01 新版 Native 强类型节点池（本文主路径）— 本文档主路径走 Native
- 你要选节点池类型（Native/Regular/Super/External）但不知差异 — 看 [选项对比](#选项) 决策表

## 选项

| 类型 | 最佳场景 | 节点来源 | 扩缩容 |
|------|---------|---------|--------|
| Native (原生) | 生产环境适用 | 腾讯云 CVM | MachineSet 托管 |
| Regular (普通) | 兼容旧版 | 腾讯云 CVM | ASG (Auto Scaling Group) |
| Super (超级) | Serverless 弹性 | 虚拟节点 | 无限制 |
| External (第三方) | 混合云/自建机房 | 非腾讯云机器 | 手动管理 |

## 准备工作

```bash
# 确认集群 Running
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "ClusterStatusSet": [{ "ClusterState": "Running" }] }

# 确认可选机型（2022-05-01 强类型可用区机型查询，Filters 按可用区过滤）
tccli tke DescribeZoneInstanceConfigInfos \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --Filters '[{"Name":"zone","Values":["ap-guangzhou-3"]}]'
# expected: exit 0（Filters 不匹配时仅返回 RequestId；匹配时返回机型配置，输出字段不固定，以实际响应为准）

# 跨产品对照（CVM 服务的同义查询，用于核对机型库存）
tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou \
  --Filters '[{"Name":"zone","Values":["ap-guangzhou-3"]}]'
# expected: InstanceType 列表
```

### 安全组（节点加入前）

> 同一集群节点建议绑定**同一**安全组，且该组不挂其他无关 CVM。安全组只开放最小权限；须放通容器网段与节点网段互访，否则跨节点 Service/DNS 失败。

| 方向 | 须放通（默认规则要点） | 说明 |
|:-----|:----------------------|:-----|
| 入站 | 容器网络 CIDR → ALL；集群网络 CIDR → ALL | Pod 间 / 节点间通信 |
| 入站 | TCP/UDP `30000-32768`（按需收紧源） | NodePort / LB 经 NodePort 转发 |
| 入站 | ICMP（按需） | Ping 诊断 |
| 出站 | 放通（或至少放通节点网段 + 容器网段） | 自定义出站时勿漏网段 |
| 可选 | TCP 22 | 仅当需要 SSH 登录节点 |

完整默认规则表见 [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084)。改错安全组可能导致节点 NotReady / Master 不可用（见 [故障排查 — 高危操作](../troubleshooting.md#高危操作后果速查)）。

```bash
# 核对目标安全组是否存在（创建节点池前）
tccli vpc DescribeSecurityGroups --region ap-guangzhou \
  --SecurityGroupIds '["<SECURITY_GROUP_ID>"]'
# expected: SecurityGroupSet 含该 ID
```

## 关键字段

> 完整入参以 `tccli tke CreateNodePool --version 2022-05-01 help --detail` 为准（新版 Native 强类型抽象; 旧版 `CreateClusterNodePool` 透传 AS 字符串，参数模型不同）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound.ClusterId` |
| Name | string | 是 | ≤60 字符 | `InvalidParameterValue.Name` |
| Type | string | 是 | `Native` / `Regular` / `Super` / `External`。`Super`=虚拟节点见 [虚拟节点](virtual-nodes.md)，`External`=注册节点见 [注册节点（专线版）](registered-nodes/dedicated-line.md) | `InvalidParameterValue.Type` |
| Native.SubnetIds | array | 是（`Type=Native` 时） | 子网 ID 列表，在 `Native` 对象内（非顶层） | `ResourceNotFound.SubnetId` |
| Native.InstanceTypes | array | 是（`Type=Native` 时） | 如 `["S5.MEDIUM4"]`，在 `Native` 对象内 | `InvalidParameterValue.InstanceTypes` |
| Native.InstanceChargeType | string | 是（`Type=Native` 时） | `POSTPAID_BY_HOUR` / `PREPAID`，在 `Native` 对象内 | `InvalidParameterValue` |
| Native.SystemDisk | object | 是（`Type=Native` 时） | 系统盘配置（`DiskType`/`DiskSize`），在 `Native` 对象内 | `InvalidParameterValue` |
| Native.Scaling | object | 是（`Type=Native` 时） | 弹性伸缩配置（`MinReplicas`/`MaxReplicas`/`CreatePolicy`），在 `Native` 对象内 | `InvalidParameterValue` |
| Native.MachineType | string | 否（`Type=Native` 时） | 原生节点机器类型枚举：`Native`（CXM 类型原生节点，默认）/ `NativeCVM`（CVM 类型原生节点）。CXM 是腾讯云原生节点新型态。在 `Native` 对象内 | `InvalidParameterValue` |
| Native.RuntimeRootDir | string | 否（`Type=Native` 时） | 容器运行时根目录（如 `/var/lib/containerd`），在 `Native` 对象内。v2022 无 `ContainerRuntime` 字段（旧版 `CreateClusterNodePool` 才有），运行时由 `RuntimeRootDir` 表达 | `InvalidParameterValue` |
| Labels | array | 否 | `[{"Name":"env","Value":"prod"}]` | `InvalidParameterValue.Labels` |
| Taints | array | 否 | `[{"Key":"dedicated","Value":"gpu","Effect":"NoSchedule"}]` | `InvalidParameterValue.Taints` |

| DeletionProtection | boolean | 否 | `true` / `false` | — |
| Native.DataDisks[].FileSystem | string | 否（透传 CVM 数据盘） | 数据盘文件系统枚举：`ext3` / `ext4` / `xfs`。未格式化的盘自动格式化为 ext4（tlinux 格式化为 xfs）。在 `Native.DataDisks` 对象数组内 | `InvalidParameterValue` |
| (Label/Taint/TagSpecification) | — | — | 结构与传参模式见 [共享字段 — Label/Taint/Annotation](../reference/shared-fields.md#label-taint-annotation) | — |

## 操作步骤

### 步骤 1：决策 — 选节点池类型与 API 版本

#### 为什么选 Native

- **Native vs Regular**: Native 节点池基于 MachineSet，支持原地升级、更细粒度的生命周期管理；Regular 基于 ASG，功能较基础
- **Native vs Super**: Super 是虚拟节点（Serverless），按 Pod 计费，不需要管理节点；Native 需要 CVM，按实例计费。Super 专题见 [虚拟节点](virtual-nodes.md)
- **Native vs External**: External 是自建 IDC 节点接入（混合云），注册流程与 CVM 节点池完全不同。External 专题见 [注册节点（专线版）](registered-nodes/dedicated-line.md)
- **默认推荐**: Native —— 基于 MachineSet，支持原地升级与细粒度生命周期管理，生产环境适用
- **可修改**： 节点池类型创建后无法切换。Regular 可迁移（新建 Native 节点池 + 移出旧节点）

#### 为什么选 2022-05-01 (CreateNodePool) 而非 2018-05-25 (CreateClusterNodePool)

- **新版 `CreateNodePool`（2022-05-01）**: 用 `Native` 强类型对象表达节点池语义（`SubnetIds`/`InstanceTypes`/`Scaling` 结构化字段），是官方文档「原生节点池」的字面对应
- **旧版 `CreateClusterNodePool`（2018-05-25）**: 透传 AS 弹性伸缩组 JSON 字符串（`AutoScalingGroupPara`/`LaunchConfigurePara`），是 AS 的薄封装，需调用方自己拼 AS 原始 JSON
- **默认推荐**: 新版 `CreateNodePool` —— 强类型参数更清晰、更安全，且 2022-05-01 是官方当前版本，长期维护方向
- **何时回退旧版**: 需要 AS 级精细控制（直接调 AS 弹性伸缩组参数）时，回退 `CreateClusterNodePool --version 2018-05-25`
- **同名≠同契约**: 两版节点池查询 Action 命名不同（新版 `DescribeNodePools` vs 旧版 `DescribeClusterNodePools`），跨版本切换前必核契约

### 步骤 2：创建节点池

`CreateNodePool` 必传 `ClusterId`/`Name`/`Type`，`Type=Native` 时 `Native` 对象内必传子网/机型/计费/系统盘/伸缩。按场景**二选一**：A 最小化（单节点测试）或 B 增强（生产，多副本+Labels+Taints+机型）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateNodePool` 会建**两个节点池**。节点池创建后改配置（机型/Labels/伸缩）用 `ModifyNodePool`/`ModifyNodePoolInstanceTypes`，**禁用第二次 `CreateNodePool` 改配置**。

#### 选项 A：最小化（单节点测试）

```bash
# Native 对象整段传 JSON（非 --cli-unfold-argument 点号展开：点号展开对嵌套对象会报 ambiguous option）
# CreatePolicy 为可选，且 API 不接受 "manual"/"ZoneEven"（传不支持的值报 CreatePolicy xxx is not supported），省略即可
# 标签字段为 --Tags（非 --TagSpecification）；Native.SecurityGroupIds 必填（非空），否则报 SecurityGroupIds is empty
tccli tke CreateNodePool \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Name "<POOL_NAME>" \
  --Type Native \
  --Native '{"SubnetIds":["<SUBNET_ID>"],"InstanceTypes":["S5.MEDIUM4"],"InstanceChargeType":"POSTPAID_BY_HOUR","SystemDisk":{"DiskType":"CLOUD_PREMIUM","DiskSize":50},"Scaling":{"MinReplicas":1,"MaxReplicas":1},"SecurityGroupIds":["<SECURITY_GROUP_ID>"]}' \
  --Tags '[{"ResourceType":"nodepool","Tags":[{"Key":"billing","Value":"<TAG_VALUE>"}]}]' \
  --DeletionProtection true
# expected: { "NodePoolId": "np-xxxxxxxx", "RequestId": "..." }
```

> ⚠️ **参数层级**: v2022 `CreateNodePool` 的 `SubnetIds`/`InstanceTypes`/`InstanceChargeType`/`SystemDisk`/`Scaling`/`MachineType` 等**都在 `Native` 对象内，非顶层**。用 `--Native '<JSON>'` 整段传（推荐），避免 `--cli-unfold-argument` 点号展开对 `SystemDisk` 这类嵌套对象报 `ambiguous option`。顶层只有 `ClusterId`/`Name`/`Type`/`Labels`/`Taints`/`Tags`/`DeletionProtection`/`Unschedulable`/`Native`/`Annotations`。标签字段是 `--Tags`（不是 `--TagSpecification`）。完整入参以 `tccli tke CreateNodePool --version 2022-05-01 help --detail` 为准。

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<CLUSTER_ID>` | 目标集群 ID | `cls-` 开头 | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<POOL_NAME>` | 节点池名称 | ≤60 字符 | 自己定义，如 `prod-pool` |
| `<SUBNET_ID>` | VPC 子网 ID | 必须在集群 VPC 内 | `tccli vpc DescribeSubnets` |
| `<SECURITY_GROUP_ID>` | 节点安全组 ID | 必须非空（`Native.SecurityGroupIds` 必填）；在集群 VPC 内 | `tccli vpc DescribeSecurityGroups --region <REGION>` |

#### 选项 B：增强（生产，多副本+Labels+Taints+机型）

> **与 A 二选一，非在 A 之后执行**。`Native.InstanceTypes` 须是 CVM 支持的机型，`Native.SystemDisk.DiskType` 须是 CVM 支持的云盘类型——先查可用机型与云盘，取实际值填入。

```bash
# 查可用机型 (按可用区)
tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou --Filters '[{"Name":"zone","Values":["ap-guangzhou-3"]}]'
# expected: { "InstanceTypeQuotaSet": [ { "InstanceType": "S5.LARGE8", "Status": "Sell", ... } ], "RequestId": "..." }
```

> 返回 `InstanceTypeQuotaSet[]`，每项含 `InstanceType`（填入 `Native.InstanceTypes` 的值）/`Status`（`Sell` 可售）。机型库存按可用区分，跨可用区须重查。

**查 TKE 支持的 OS 镜像**（`Native` 不直接指定 OS，但集群创建时 `ClusterBasicSettings.ClusterOs` 须是 TKE 支持值）：

```bash
tccli tke DescribeImages --region ap-guangzhou
# expected: { "TotalCount": 84, "ImageInstanceSet": [ { "Alias": "...", "OsName": "ubuntu20.04x86_64", "ImageId": "img-xxx" } ], "RequestId": "..." }
```

> 返回 `ImageInstanceSet[]`，每项含 `Alias`/`OsName`/`ImageId`。`OsName` 是 TKE 支持的 OS 标识，填入集群创建的 `ClusterOs`。与 `DescribeOSImages`（返回 OS 聚合信息）互补。

指定机型、Labels、Taints、弹性伸缩:

```bash
tccli tke CreateNodePool \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --cli-unfold-argument \
  --ClusterId "<CLUSTER_ID>" \
  --Name "prod-worker-pool" \
  --Type Native \
  --Native.SubnetIds '["<SUBNET_ID>"]' \
  --Native.InstanceTypes '["S5.LARGE8"]' \
  --Native.InstanceChargeType POSTPAID_BY_HOUR \
  --Native.SystemDisk '{"DiskType":"CLOUD_PREMIUM","DiskSize":100}' \
  --Native.Scaling '{"MinReplicas":2,"MaxReplicas":5,"CreatePolicy":"ZoneEven"}' \
  --Native.MachineType NativeCVM \
  --Labels '[
    {"Name":"env","Value":"production"},
    {"Name":"team","Value":"backend"}
  ]' \
  --Taints '[
    {"Key":"workload-type","Value":"cpu","Effect":"NoSchedule"}
  ]' \
  --DeletionProtection true
# expected: { "NodePoolId": "np-xxxxxxxx", "RequestId": "..." }
```

### 步骤 3：验证

```bash
tccli tke DescribeNodePools \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]'
# expected: 顶层列表字段为 NodePools（非 NodePoolSet）；LifeState 为 Running（就绪）
```

> Filter `Name` 支持 `NodePoolsName`（按名）和 `NodePoolsId`（按 ID）。用 `tccli tke DescribeNodePools --version 2022-05-01 help --detail` 查合法 Name。

> ⚠️ 新版查询 Action 名为 `DescribeNodePools`（去 `Cluster` 前缀），旧版为 `DescribeClusterNodePools`/`DescribeClusterNodePoolDetail`。两版命名不同，本指南统一用新版 + `--version 2022-05-01`。响应顶层为 `NodePools`；Native 副本在 `Native.Replicas` / `Native.ReadyReplicas`（无旧版 `NodeCountSummary`）。

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| 生命周期状态 | `DescribeNodePools --version 2022-05-01` | `LifeState: "Running"` |
| Labels | `DescribeNodePools` → Labels | 与创建参数一致 |
| Taints | `DescribeNodePools` → Taints | 与创建参数一致 |
| 副本数 | `DescribeNodePools` → `Native.Replicas` / `ReadyReplicas` | 与创建时 `Replicas`/`Scaling` 一致（最小化可为 0） |
| 删除保护 | `DescribeNodePools` → DeletionProtection | 与创建参数一致（未开则为 `false`） |

### 步骤 3.5：节点池弹性健康度诊断（2022-05-01）

`DescribeNodePoolsElasticityStrength` 返回集群下各节点池的弹性健康度评分（容量水位 / 副本调度余量），是节点池弹性伸缩调优的辅助观测接口，仅 `--version 2022-05-01` 提供。

```bash
# 查询集群节点池弹性健康度（需 ClusterId；真实命令，已实测）
tccli tke DescribeNodePoolsElasticityStrength \
  --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>"
# expected: 返回各节点池弹性健康度（字段随版本演进，以实际响应为准）
```

> **能力边界红线（实测 2026-07-14, tccli v3.1.126.1）**：本接口**需 `ClusterId`**，但当前 tccli 构建**未给该 Action 注册 CLI 入参**，执行上报 `Unknown options: --ClusterId, <id>`（SDK `MissingParameter` 要求 `ClusterId`，CLI 参数解析层却拒绝接收，属 tccli 注册缺口，非调用方错误）。**结论**：该接口在本文档环境**不可经 tccli 成功调用**，命令句仅供「识别该能力存在 + 未来 tccli 修复后可直跑」；弹性健康度/容量余量请改用 `DescribeNodePools --version 2022-05-01` → `Native.Replicas` / `Native.ReadyReplicas` / `Native.Scaling`（Min/Max）判断，**勿依赖本接口做容量决策**。若必须取弹性评分，走控制台或等 tccli 版本更新。

## 旧版路径：CreateClusterNodePool (2018-05-25)

> 需要 AS 弹性伸缩组级精细控制时回退旧版。旧版透传 AS 原始 JSON 字符串，调用方自己拼 AS 参数；新版 `CreateNodePool` (2022-05-01) 用 `Native` 强类型对象，生产环境用新版。参数名以 `tccli tke CreateClusterNodePool --version 2018-05-25 help --detail` 为准。

```bash
# 旧版创建节点池（透传 AS JSON 字符串：AutoScalingGroupPara + LaunchConfigurePara）
# tccli 强制要求 --InstanceAdvancedSettings（即使空对象 {}），缺失报 "the following arguments are required: --InstanceAdvancedSettings"
tccli tke CreateClusterNodePool \
  --version 2018-05-25 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Name "<POOL_NAME>" \
  --AutoScalingGroupPara '<AS_GROUP_JSON>' \
  --LaunchConfigurePara '<AS_LAUNCH_CONFIG_JSON>' \
  --InstanceAdvancedSettings '{}' \
  --EnableAutoscale false
# expected: exit 0, 返回 NodePoolId
```

| 占位符 | 含义 | 约束 |
|:-------|:-----|:-----|
| `<AS_GROUP_JSON>` | AS 弹性伸缩组配置 | JSON 字符串，含 MinSize/MaxSize/VpcId/SubnetId 等，需先 `tccli as CreateAutoScalingGroup` 拼 |
| `<AS_LAUNCH_CONFIG_JSON>` | AS 启动配置 | JSON 字符串，含 InstanceType/ImageId/SecurityGroupIds 等 |
| `InstanceAdvancedSettings` | 节点高级设置 | **TCCLI 强制必填**，无自定义传 `{}` |
| `EnableAutoscale` | 是否启用弹性扩缩容 | `false`=固定节点数，`true`=按 AS 规则弹性 |

> 旧版与新版契约不同：旧版 `CreateClusterNodePool` 透传 AS 字符串（`AutoScalingGroupPara`/`LaunchConfigurePara`），新版 `CreateNodePool` 用结构化 `Native` 对象（`SubnetIds`/`InstanceTypes`）。两版查询 Action 命名也不同（旧 `DescribeClusterNodePools` vs 新 `DescribeNodePools`），跨版本切换前用 `--generate-cli-skeleton` 逐字段核契约。

## 清理

> ⚠️ 删除节点池会销毁池内**所有节点** (CVM)，Pod 会被驱逐（除非有 PDB 保护）。节点池成员管理（移出单个节点/机型变更）见 [节点运维](instance-ops.md)。

```bash
# 关删除保护 + 删节点池 + 验证（删除保护开时先关）
tccli tke ModifyNodePool --version 2022-05-01 --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>" --DeletionProtection false
# expected: exit 0

tccli tke DeleteNodePool --version 2022-05-01 --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>"
# expected: exit 0

tccli tke DescribeNodePools --version 2022-05-01 --ClusterId "<CLUSTER_ID>"
# expected: 目标节点池不在列表中 → 已删
```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `InvalidParameterValue.InstanceTypes` | `tccli cvm DescribeZoneInstanceConfigInfos` | 指定机型在该可用区不存在 | 查询可用机型，换一个实际存在的机型 |
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets --SubnetIds '["<ID>"]'` | 子网 ID 错误或不属于集群 VPC | 使用集群 VPC 内的子网 ID |
| `LimitExceeded.NodePoolQuota` | `tccli tke DescribeNodePools --version 2022-05-01 --ClusterId "<ID>"` | 节点池数量达上限 | 删除闲置节点池或提工单 |
| `ResourceNotFound.ClusterId` | `tccli tke DescribeClusters` | 集群 ID 错误 | 确认集群 ID 格式为 `cls-xxxxxxxx` |
| `UnknownParameter` | `tccli tke CreateNodePool --version 2022-05-01 --generate-cli-skeleton` 核契约 | 误用旧版参数名到新版 Action | 改用新版 `CreateNodePool` 的参数名 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 节点未自动创建 | `DescribeNodePools --version 2022-05-01` → `Native.Replicas` / `ReadyReplicas` | 未配置扩缩容或 `Replicas=0` | [扩容节点池](nodepool-scale.md) |
| 节点创建后立即 Terminating | `DescribeClusterInstances` → InstanceState | 机型资源不足 | 换机型重建 |
| 节点加入但 NotReady | `DescribeClusterInstances` → InstanceState | 节点初始化脚本失败或安全组不通 | 检查安全组 443 出方向 |

## 节点池成员与机型管理

> 节点加入节点池、节点保护、节点池机型变更。

```bash
# 添加已有节点到节点池 (InstanceIds[] 已存在的节点)
tccli tke AddNodeToNodePool --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --NodePoolId "<NODEPOOL_ID>" --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0

# 设置节点池节点保护 (防止缩容时驱逐, InstanceIds[] 指定节点)
tccli tke SetNodePoolNodeProtection --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --NodePoolId "<NODEPOOL_ID>" --InstanceIds '["<INSTANCE_ID>"]' --Protected true
# expected: exit 0

# 修改节点池机型 (InstanceTypes[] 机型列表, 滚动变更)
tccli tke ModifyNodePoolInstanceTypes --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --NodePoolId "<NODEPOOL_ID>" --InstanceTypes '["<INSTANCE_TYPE>"]'
# expected: exit 0
```

> `AddNodeToNodePool` 把已存在节点（非新建）加入节点池，区别于节点池自动创建节点。`SetNodePoolNodeProtection` 的 `Protected=true` 防止缩容驱逐（与 [扩缩容](nodepool-scale.md) 配合）。`ModifyNodePoolInstanceTypes` 变更机型会滚动重建节点。

## 收尾确认

```bash
# ②业务可用性端到端: CreatePolicy=manual 时初始 0 节点，需扩容才有节点真加入 K8s 可调度
tccli tke DescribeClusterInstances --version 2018-05-25 --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceIds '["<INSTANCE_ID>"]' \
  --filter "InstanceSet[].{id:InstanceId,state:InstanceState}" --output text
# expected: 若已扩容，扩容节点 InstanceState=running；未扩容时无节点（CreatePolicy=manual 初始为 0，需进扩缩容）

# ④衔接下一步前置就绪: 节点池 LifeState=Running 且扩容后 ≥1 节点 ready 才可进扩缩容
tccli tke DescribeNodePools --version 2022-05-01 --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]' \
  --filter "NodePools[0].{name:Name,state:LifeState,desired:Native.Replicas}"
# expected: LifeState=Running；Native.Replicas 反映期望副本数（CreatePolicy=manual 初始 Replicas=0，需 [扩缩容](nodepool-scale.md) 设值才建节点）
```

> 节点池 `LifeState=Running`（2022-05-01 Native）= 创建闭环完成。但 CreatePolicy=manual 时 Replicas 初始为 0、无节点加入集群（业务可用性边界），须进 [扩缩容](nodepool-scale.md) 设期望副本触发建 CVM，待节点 InstanceState=running 后方可调度 Pod。

## 下一步

- [扩缩容节点池](nodepool-scale.md) — 调整节点数量
- [节点运维](instance-ops.md) — 查询/启动/停止/重启节点
- [配置网络](../networking/index.md) — 管理集群访问端点
