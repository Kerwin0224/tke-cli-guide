---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建节点池

> 给集群添加工作节点。原生节点池 (Native) —— 推荐类型，基于 2022-05-01 新版 API 的强类型 `Native` 抽象。
> 控制台: [容器服务 - 节点池](https://console.cloud.tencent.com/tke2/nodepool)
>
> **API 版本**: 本文档命令走 **2022-05-01**（官方当前版本）。节点池的 Native 强类型抽象仅新版提供，所有命令显式带 `--version`。完整版本选择见 [API 版本选择](../index.md#api-版本选择)。

## 选项

| 类型 | 最佳场景 | 节点来源 | 扩缩容 |
|------|---------|---------|--------|
| Native (原生) | 生产环境推荐 | 腾讯云 CVM | MachineSet 托管 |
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

## 关键字段

> 来源: `tccli tke CreateNodePool --version 2022-05-01 --generate-cli-skeleton`（新版 Native 强类型抽象; 旧版 `CreateClusterNodePool` 透传 AS 字符串，参数模型不同）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound.ClusterId` |
| Name | string | 是 | ≤60 字符 | `InvalidParameterValue.Name` |
| Type | string | 是 | `Native` / `Regular` / `Super` / `External`。`Super`=虚拟节点见 [虚拟节点](virtual-nodes.md)，`External`=扩展节点见 [扩展节点接入](external-nodes.md) | `InvalidParameterValue.Type` |
| SubnetIds | array | 是 | 子网 ID 列表 | `ResourceNotFound.SubnetId` |
| InstanceTypes | array | no (Native 自动) | 如 `["S5.MEDIUM4"]` | `InvalidParameterValue.InstanceTypes` |
| Labels | array | 否 | `[{"Name":"env","Value":"prod"}]` | `InvalidParameterValue.Labels` |
| Taints | array | 否 | `[{"Key":"dedicated","Value":"gpu","Effect":"NoSchedule"}]` | `InvalidParameterValue.Taints` |
| DeletionProtection | boolean | 否 | `true` / `false` | — |
| ContainerRuntime | string | 否 | `containerd` / `docker` | `InvalidParameterValue.ContainerRuntime` |
| NodePoolOs | string | 否 | `ubuntu20.04x86_64` / `tlinux3.1x86_64` | `InvalidParameterValue.Os` |
| MachineType | string | 否（2022 新版，`Type=Native` 时） | 原生节点机器类型枚举：`Native`（CXM 类型原生节点，默认）/ `NativeCVM`（CVM 类型原生节点）。CXM 是腾讯云原生节点新型态 | `InvalidParameterValue` |
| DataDisk.FileSystem | string | 否（透传 CVM 数据盘） | 数据盘文件系统枚举：`ext3` / `ext4` / `xfs`。未格式化的盘自动格式化为 ext4（tlinux 格式化为 xfs） | `InvalidParameterValue` |

## 操作步骤

### 步骤 1：决策 — 选节点池类型与 API 版本

#### 为什么选 Native

- **Native vs Regular**: Native 节点池基于 MachineSet，支持原地升级、更细粒度的生命周期管理；Regular 基于 ASG，功能较基础
- **Native vs Super**: Super 是虚拟节点（Serverless），按 Pod 计费，不需要管理节点；Native 需要 CVM，按实例计费。Super 专题见 [虚拟节点](virtual-nodes.md)
- **Native vs External**: External 是自建 IDC 节点接入（混合云），注册流程与 CVM 节点池完全不同。External 专题见 [扩展节点接入](external-nodes.md)
- **默认推荐**: Native —— 生产环境首选，功能最完整
- **能改吗?**: 节点池类型创建后无法切换。Regular 可迁移（新建 Native 节点池 + 移出旧节点）

#### 为什么选 2022-05-01 (CreateNodePool) 而非 2018-05-25 (CreateClusterNodePool)

- **新版 `CreateNodePool`（2022-05-01）**: 用 `Native` 强类型对象表达节点池语义（`SubnetIds`/`InstanceTypes`/`Scaling` 结构化字段），是官方文档「原生节点池」的字面对应
- **旧版 `CreateClusterNodePool`（2018-05-25）**: 透传 AS 弹性伸缩组 JSON 字符串（`AutoScalingGroupPara`/`LaunchConfigurePara`），是 AS 的薄封装，需调用方自己拼 AS 原始 JSON
- **默认推荐**: 新版 `CreateNodePool` —— 强类型参数更清晰、更安全，且 2022-05-01 是官方当前版本，长期维护方向
- **何时回退旧版**: 需要 AS 级精细控制（直接调 AS 弹性伸缩组参数）时，回退 `CreateClusterNodePool --version 2018-05-25`
- **同名≠同契约**: 两版节点池查询 Action 命名不同（新版 `DescribeNodePools` vs 旧版 `DescribeClusterNodePools`），跨版本切换前必核契约

### 步骤 2：创建 — 最小化

```bash
tccli tke CreateNodePool \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Name "<POOL_NAME>" \
  --Type Native \
  --SubnetIds '["<SUBNET_ID>"]' \
  --DeletionProtection true
# expected: { "NodePoolId": "np-xxxxxxxx", "RequestId": "..." }
```

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<CLUSTER_ID>` | 目标集群 ID | `cls-` 开头 | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<POOL_NAME>` | 节点池名称 | ≤60 字符 | 自己定义，如 `prod-pool` |
| `<SUBNET_ID>` | VPC 子网 ID | 必须在集群 VPC 内 | `tccli vpc DescribeSubnets` |

### 步骤 3：创建 — 增强：生产就绪

> **选 OS 镜像**：`NodePoolOs` 取值（如 `ubuntu20.04x86_64`）须是 TKE 支持的镜像 `OsName`。先查可用镜像列表，取其 `OsName` 字段填入 `NodePoolOs`。

```bash
tccli tke DescribeImages --region ap-guangzhou
# expected: { "TotalCount": 84, "ImageInstanceSet": [ { "Alias": "...", "OsName": "ubuntu20.04x86_64", "ImageId": "img-xxx", "OsCustomizeType": "GENERAL" } ], "RequestId": "..." }
```

> 返回 `ImageInstanceSet[]`，每项含 `Alias`（展示名）/`OsName`（填入 `NodePoolOs` 的值）/`ImageId`。`DescribeOSImages`（[集群配置](../clusters/configure.md)）返回的是 OS 聚合信息，与此处的镜像明细列表互补。

指定机型、Labels、Taints、自定义 OS:

```bash
tccli tke CreateNodePool \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Name "prod-worker-pool" \
  --Type Native \
  --SubnetIds '["<SUBNET_ID>"]' \
  --InstanceTypes '["S5.LARGE8"]' \
  --ContainerRuntime containerd \
  --NodePoolOs ubuntu20.04x86_64 \
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

### 步骤 4：验证

```bash
tccli tke DescribeNodePools \
  --version 2022-05-01 \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolIds '["<NODE_POOL_ID>"]'
# expected: LifeState: "normal", Labels 和 Taints 与创建参数一致
```

> ⚠️ 新版查询 Action 名为 `DescribeNodePools`（去 `Cluster` 前缀），旧版为 `DescribeClusterNodePools`/`DescribeClusterNodePoolDetail`。两版命名不同，本指南统一用新版 + `--version 2022-05-01`。响应字段名以 `--generate-cli-skeleton` 为准。

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| 生命周期状态 | `DescribeNodePools --version 2022-05-01` | `LifeState: "normal"` |
| Labels | `DescribeNodePools` → Labels | 与创建参数一致 |
| Taints | `DescribeNodePools` → Taints | 与创建参数一致 |
| 节点数 | `DescribeNodePools` → NodeCountSummary | 初始为 0，需手动扩容 |
| 删除保护 | `DescribeNodePools` → DeletionProtection | `true` |

## 旧版路径：CreateClusterNodePool (2018-05-25)

> 需要 AS 弹性伸缩组级精细控制时回退旧版。旧版透传 AS 原始 JSON 字符串，调用方自己拼 AS 参数；新版 `CreateNodePool` (2022-05-01) 用 `Native` 强类型对象，生产首选新版。参数名以 `tccli tke CreateClusterNodePool --version 2018-05-25 --generate-cli-skeleton` 为准。

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
| `InstanceAdvancedSettings` | 节点高级设置 | **tccli 强制必填**，无自定义传 `{}` |
| `EnableAutoscale` | 是否启用弹性扩缩容 | `false`=固定节点数，`true`=按 AS 规则弹性 |

> 旧版与新版契约不同：旧版 `CreateClusterNodePool` 透传 AS 字符串（`AutoScalingGroupPara`/`LaunchConfigurePara`），新版 `CreateNodePool` 用结构化 `Native` 对象（`SubnetIds`/`InstanceTypes`）。两版查询 Action 命名也不同（旧 `DescribeClusterNodePools` vs 新 `DescribeNodePools`），跨版本切换前用 `--generate-cli-skeleton` 逐字段核契约。

## 清理

> ⚠️ 删除节点池会销毁池内**所有节点** (CVM)，Pod 会被驱逐（除非有 PDB 保护）。

```bash
# 1. 如果开启了删除保护，先关闭
tccli tke ModifyNodePool \
  --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODE_POOL_ID>" \
  --DeletionProtection false

# 2. 删除节点池
tccli tke DeleteNodePool \
  --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODE_POOL_ID>"
# expected: exit 0

# 3. 验证
tccli tke DescribeNodePools \
  --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>"
# expected: 目标节点池不在列表中
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
| 节点未自动创建 | `DescribeNodePools --version 2022-05-01` → NodeCountSummary | 未配置扩缩容或 DesiredNodes=0 | [扩容节点池](nodepool-scale.md) |
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

## 下一步

- [扩缩容节点池](nodepool-scale.md) — 调整节点数量
- [节点运维](instance-ops.md) — 查询/启动/停止/重启节点
- [配置网络](../networking/index.md) — 管理集群访问端点

## 控制台替代方案

[容器服务控制台 - 创建节点池](https://console.cloud.tencent.com/tke2/nodepool/create)
