---
doc_type: How-to
subtype: 6A
fused: true
---
# 升级集群版本

> 升级集群 Master 的 Kubernetes 版本。异步操作，升级期间控制面不可管理但工作负载正常运行。**不可回滚**——只能继续升级到更高版本。

## 概述

升级分小版本（如 1.30.0 → 1.30.5）与大版本（如 1.30 → 1.34）。大版本升级风险高，建议逐版本升级而非跳版本。

| 策略 | 适用 | 风险 | 耗时 |
|:-----|:-----|:-----|:-----|
| 小版本升级 | 补丁修复、安全更新 | 低 | 5-15 分钟 |
| 大版本逐版本 | 生产环境（1.30→1.32→1.34） | 中 | 每跳 10-20 分钟 |
| 大版本跳版本 | 紧急场景（1.30→1.34） | 高 | 不推荐，API 弃用风险 |

操作是**异步**的：`UpdateClusterVersion` 返回即提交，集群进入 `Upgrading` 状态，需轮询直到 `Running`。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterState = "Running"（非 Running 不允许升级）
```

### 资源检查

```bash
# 1. 查可升级版本（Versions 为空表示已是最新）
tccli tke DescribeAvailableClusterVersion --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Versions 含目标版本；为空 [] 则无需升级

# 2. 节点兼容性前置检查（UpgradeType 枚举: reset 重装升级 / hot 原地滚动小版本 / major 原地滚动大版本）
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --UpgradeType reset
# expected: UpgradeAbleInstances[] 为空或 Total=0（无可升级节点=已最新）；返回 ClusterVersion/LatestVersion

# 3. 确认无进行中的升级任务（DescribeUpgradeTasks 不接受 --ClusterId，按 Offset/Limit 翻页）
tccli tke DescribeUpgradeTasks --region ap-guangzhou --Offset 0 --Limit 20
# expected: UpgradeTasks[] 为空，或用返回的 ID 逐个 DescribeUpgradeTaskDetail --ID "<TASK_ID>" 查 UpgradePlans[].Status（非 Running）

# 4. 备份 kubeconfig（升级不可回滚，失败需重建；--filter + --output text 提取纯 kubeconfig，无 --output text 会带引号导致 kubectl 报 JSON parse error）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --filter "Kubeconfig" --output text > backup-kubeconfig.yaml
# expected: kubeconfig 文件生成（可直接 kubectl --kubeconfig backup-kubeconfig.yaml）
```

## 关键字段

> 来源：`tccli tke UpdateClusterVersion --generate-cli-skeleton`（实测）。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| DstVersion | string | 是 | 目标版本，如 `1.34.1`，须在 `DescribeAvailableClusterVersion` 返回列表中 | `InvalidParameterValue` / `FailedOperation` |
| MaxNotReadyPercent | float | 否 | 升级容忍度，0-100，默认低值（保守） | `InvalidParameterValue` |
| SkipPreCheck | boolean | 否 | 跳过前置检查，**危险**，默认 false | 跳过后可能升级失败 |
| ExtraArgs | object | 否 | Master 组件自定义参数（Etcd/KubeAPIServer/KubeControllerManager/KubeScheduler） | 各子字段校验 |

> `DstVersion` 必须是 `DescribeAvailableClusterVersion` 返回的版本之一，不能用 `DescribeVersions` 的全量版本——后者含不可升级到的版本。

## 操作步骤

### 步骤 1：决策 — 选升级策略

#### 为什么逐版本升级

- **逐版本 vs 跳版本**: 逐版本（1.30→1.32→1.34）每跳风险可控，API 弃用逐版本暴露；跳版本（1.30→1.34）一次性暴露多个版本的弃用，工作负载可能因 API 移除而崩溃
- **默认推荐**: 逐版本升级，生产环境尤其如此
- **能回滚吗?**: 不能。集群版本升级不可回滚，只能继续升级到更高版本。节点版本可单独升级/降级，但 Master 不可

### 步骤 2：前置检查

```bash
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --UpgradeType reset
# expected: UpgradeAbleInstances[] 为空或 Total=0（节点已最新）；若有不兼容，先处理节点
```

### 步骤 3：升级 — 最小化

```bash
tccli tke UpdateClusterVersion --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --DstVersion "<TARGET_VERSION>"
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 目标集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<TARGET_VERSION>` | 目标版本 | 须在可升级列表 | `tccli tke DescribeAvailableClusterVersion` → `Versions[]` |

### 步骤 4：升级 — 增强：金丝雀

设置较高容忍度，允许部分节点先升级（适合大规模集群）：

```bash
tccli tke UpdateClusterVersion --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --DstVersion "<TARGET_VERSION>" \
  --MaxNotReadyPercent 30
# expected: exit 0
```

> `MaxNotReadyPercent 30` 允许 30% 节点在升级期间 NotReady。值越大升级越快但风险越高，生产环境建议默认（低值）。

### 步骤 5：验证

异步操作，检查 ≥4 个维度：

```bash
# 轮询集群状态（升级中为 Upgrading，完成后回 Running）
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterState"
# expected: 升级中 "Upgrading" → 完成后 "Running"
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 集群状态 | `DescribeClusterStatus` → `ClusterState` | `Upgrading` → `Running` |
| 版本号 | `DescribeClusters --ClusterIds '["<ID>"]'` → `ClusterVersion` | 等于 `<TARGET_VERSION>` |
| 升级任务 | `DescribeUpgradeTasks --Offset 0 --Limit 20` 取 `UpgradeTasks[].ID`，再 `DescribeUpgradeTaskDetail --ID "<ID>"` → `UpgradePlans[].Status` | `Succeeded` |
| 节点版本 | `CheckInstancesUpgradeAble --ClusterId "<ID>" --UpgradeType reset` → `ClusterVersion`/`UpgradeAbleInstances[].Version`（`DescribeClusterInstances` 不返回节点 K8s 版本） | 节点 Version 与 Master 一致 |
| kubectl 可用 | `kubectl get nodes`（用 kubeconfig） | 节点列表返回 |

> 升级期间 `ClusterState=Upgrading`，可 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 暂停（不是回滚，注意 `ClusterID`/`PlanID` 均大写 ID）。暂停后恢复需重新触发。

## 清理

> **不可回滚**：集群版本升级后无法降级到旧版本。若升级失败导致集群不可用，只能 `DeleteCluster` 重建并用备份的 kubeconfig/配置恢复工作负载（P8 显式标注）。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Versions` 为空 `[]` | `DescribeAvailableClusterVersion` | 已是最新版本，无更高版本可升级 | 无需升级 |
| `FailedOperation` | `CheckInstancesUpgradeAble` 看不兼容节点 | 节点版本/组件不兼容目标版本 | 先升级节点或移除不兼容节点 |
| `ResourceInUse` | `DescribeUpgradeTasks --Offset 0 --Limit 20` 看是否有任务，有则 `DescribeUpgradeTaskDetail --ID "<ID>"` 查 `UpgradePlans[].Status` | 已有升级任务在跑 | 等待或 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 后重试 |
| `InvalidParameterValue.DstVersion` | 核对 `DstVersion` | 版本号不在可升级列表 | 用 `DescribeAvailableClusterVersion` 返回的版本 |
| `UnsupportedOperation` | `DescribeClusterStatus` 看状态 | 集群非 `Running`（升级中/异常） | 等集群 `Running` 后重试 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `Upgrading` | `DescribeUpgradeTaskDetail --ID "<ID>"` → `UpgradePlans[].Status` + `GetUpgradeInstanceProgress --ClusterId "<ID>"` 查节点进度 | 某节点升级卡住 | `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 暂停，定位卡住节点 |
| 升级后部分节点版本不一致 | `CheckInstancesUpgradeAble --ClusterId "<ID>" --UpgradeType reset` → `UpgradeAbleInstances[].Version`（`DescribeClusterInstances` 不返回节点版本） | 节点未跟随升级 | `UpgradeClusterInstances` 单独升级节点 |
| `Running` 但 kubectl 报 API 版本弃用 | `kubectl get nodes --v=6` | 工作负载用了已弃用 API | 更新工作负载 YAML，移除弃用 API |
| 升级任务 `Failed` | `DescribeUpgradeTaskDetail` | 资源不足或组件冲突 | 查 TaskDetail，修复后重新 `UpdateClusterVersion` |

> 升级卡住超 30 分钟属异常，用 `DescribeUpgradeTaskDetail --ID "<ID>"` 看具体步骤，必要时 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 后提工单附 RequestId。

## 单独升级节点版本

> Master 升级后节点未跟随升级时，用 `UpgradeClusterInstances` 对指定节点单独升级。这是异步任务型接口：`Operation` 控制任务生命周期，`UpgradeType` 仅在 `Operation=create` 时生效。

`Operation` 枚举（任务生命周期，来自 API 定义）：

| Operation | 含义 |
|:----------|:-----|
| `create` | 开始一次升级任务（需配合 `UpgradeType` + `InstanceIds`） |
| `pause` | 暂停进行中的任务 |
| `resume` | 继续已暂停的任务 |
| `abort` | 终止任务 |

```bash
# 开始升级指定节点（UpgradeType 同 CheckInstancesUpgradeAble: reset/hot/major）
tccli tke UpgradeClusterInstances --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Operation create \
  --UpgradeType reset \
  --InstanceIds '["<INSTANCE_ID>"]' \
  --MaxNotReadyPercent 30
# expected: exit 0, 返回 RequestId（异步任务已提交）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:-------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<INSTANCE_ID>` | 节点 CVM ID | `ins-xxxxxxxx` | `tccli tke DescribeClusterInstances`（旧版用 `--InstanceIds`/`--InstanceRole`，新版用 `--SortBy`/`--NeedTags`，两版入参不兼容 D30；本文走旧版 2018-05-25）→ 节点 InstanceId |

任务状态用 `GetUpgradeInstanceProgress` 轮询；暂停/终止用 `--Operation pause`/`abort`。完整节点运维（接入/删除/升级）见 [节点实例操作](../nodes/instance-ops.md)。

### 升级任务进度与计划控制

> 节点升级/Master 升级都是异步任务，三个 Action 覆盖「查进度—查详情—停计划」控制面。参数名大小写以 `--generate-cli-skeleton` 实测为准（注意 `ClusterID`/`PlanID` 大写 vs `ClusterId` 小写）。

```bash
# 1. 查节点升级进度（ClusterId 小写 + Offset/Limit 分页）
tccli tke GetUpgradeInstanceProgress --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Offset 0 --Limit 20
# expected: 返回各节点升级进度（InstanceProgress[]）

# 2. 查升级任务详情（ID 为任务 ID，来自 DescribeUpgradeTasks[].ID）
tccli tke DescribeUpgradeTaskDetail --region ap-guangzhou \
  --ID "<TASK_ID>" --Offset 0 --Limit 20
# expected: UpgradePlans[].Status = Succeeded / Running / Failed

# 3. 暂停升级计划（ClusterID/PlanID 均大写；暂停非回滚，恢复需重新触发）
tccli tke CancelUpgradePlan --region ap-guangzhou \
  --ClusterID "<CLUSTER_ID>" --PlanID <PLAN_ID>
# expected: exit 0
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<TASK_ID>` | 升级任务 ID | `tccli tke DescribeUpgradeTasks --Offset 0 --Limit 20` → `UpgradeTasks[].ID` |
| `<PLAN_ID>` | 升级计划 ID（整数） | `DescribeUpgradeTaskDetail --ID "<TASK_ID>"` → `UpgradePlans[].PlanID` |

> ⚠️ 大小写不一致是真实契约：`GetUpgradeInstanceProgress`/`DescribeUpgradeTaskDetail` 用任务语义参数，`CancelUpgradePlan` 的 `ClusterID`/`PlanID` 大写。切换接口前用 `--generate-cli-skeleton` 核对，勿凭记忆。

## 下一步

- [节点版本升级](../nodes/instance-ops.md) — Master 升级后单独升级节点
- [集群状态机](../reference/states.md) — `Upgrading` 等状态含义
- [创建集群](create.md) — 升级失败需重建时参考
- [故障排查](../troubleshooting.md) — 升级卡住的诊断路径
- [独立集群 Master 运维](master-ops.md) — 独立集群扩缩容 Master/etcd 节点

## 控制台替代方案

[容器服务控制台 - 集群升级](https://console.cloud.tencent.com/tke2/cluster)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `UpdateClusterVersion` | 主操作 | 2018-05-25 | 升级 Master 版本（异步，不可回滚） |
| `UpgradeClusterInstances` | 主操作 | 2018-05-25 | 单独升级节点版本 |
| `CancelUpgradePlan` | 清理 | 2018-05-25 | 暂停升级计划（`--ClusterID`/`--PlanID` 大写） |
| `DeleteCluster` | 清理 | 2018-05-25 | 升级失败回退重建 |
| `CheckInstancesUpgradeAble` | 验证 | 2018-05-25 | 节点升级兼容性前置检查 |
| `DescribeAvailableClusterVersion` | 验证 | 2018-05-25 | 查可升级版本（区别于全量版本） |
| `DescribeUpgradeTasks` | 验证 | 2018-05-25 | 升级任务列表（按 Offset/Limit） |
| `DescribeUpgradeTaskDetail` | 验证 | 2018-05-25 | 升级任务详情与 Plan 状态 |
| `GetUpgradeInstanceProgress` | 验证 | 2018-05-25 | 节点升级进度 |
| `DescribeClusterInstances` | 验证 | 2018-05-25 | 查节点版本一致性 |
| `DescribeClusterKubeconfig` | 验证 | 2018-05-25 | 备份 kubeconfig |
| `DescribeClusters` | 验证 | 2018-05-25 | 确认版本号 |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 轮询 Upgrading→Running |
| `DescribeVersions` | 验证 | 2018-05-25 | 全量版本列表 |
