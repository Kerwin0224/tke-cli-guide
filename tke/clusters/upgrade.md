---
doc_type: How-to
subtype: 6A
fused: true
---
# 升级集群版本

> 控制台: [容器服务控制台 - 集群升级](https://console.cloud.tencent.com/tke2/cluster)
> 升级集群 Master 的 Kubernetes 版本。异步操作，升级期间控制面不可变更但工作负载正常运行。**不可回滚**——只能继续升级到更高版本。

> 本文档 Action 属 **TKE 2018-05-25**（`UpdateClusterVersion`/`DescribeAvailableClusterVersion`/`CancelUpgradePlan` 等均旧版独有）。注意 `CancelUpgradePlan` 用 `ClusterID`/`PlanID`（大写 ID），与多数集群接口的 `ClusterId`（小写 d）不同。`DescribeClusterInstances` 是两版同名且入参不兼容（旧 `InstanceIds`/`InstanceRole` vs 新 `SortBy`/`NeedTags`），本文走旧版，见 [节点实例操作](../nodes/instance-ops.md)。

> 官方文档：[升级集群](https://cloud.tencent.com/document/product/457/32192) · [集群生命周期](https://cloud.tencent.com/document/product/457/32188) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 概述

升级分小版本（如 1.30.0 → 1.30.5）与大版本（如 1.30 → 1.34）。大版本升级风险高，建议逐版本升级而非跳版本。

### K8s 1.34+ 节点初始化行为变更（升级前必读）

升到 **TKE Kubernetes 1.34 及以上**时，节点初始化行为与旧版不同：

| 项 | 旧行为（&lt;1.34） | 新行为（≥1.34） |
|:---|:------------------|:----------------|
| 节点注册凭证 | 控制面给 kubelet 下发**长期有效** kubeconfig 证书（早期约 20 年，后约 30 年） | 下发 **bootstrap token（24 小时有效）**；kubelet 启动后用 token 向 apiserver 换正式证书；证书目录 `/var/lib/kubelet/pki/` |
| root 的 `/root/.kube/config` | 按 `TKE_ADMIN_KUBECONFIG` 白名单：白名单内长期 admin；非白名单 12 小时 admin（可访问集群内全部资源） | 白名单机制**失效**；改为软链接指向 kubelet 当前 kubeconfig，权限与 kubelet 一致，**仅能操作当前节点资源** |

> 依赖「节点上 root admin kubeconfig 管全集群」的运维脚本，在 1.34+ **会失效**——改用集群 Endpoint + 合法凭证，或节点级权限内操作。完整说明见 [集群版本相关的节点变更说明](https://cloud.tencent.com/document/product/457/126536)。

| 策略 | 适用 | 风险 | 耗时 |
|:-----|:-----|:-----|:-----|
| 小版本升级 | 补丁修复、安全更新 | 低 | 5-15 分钟 |
| 大版本逐版本 | 生产环境（1.30→1.32→1.34） | 中 | 每跳 10-20 分钟 |
| 大版本跳版本 | 紧急场景（1.30→1.34） | 高 | 不推荐，API 弃用风险 |

操作是**异步**的：`UpdateClusterVersion` 返回即提交，集群进入 `Upgrading` 状态，需轮询直到 `Running`。

> 配额：升级窗口内需确认节点配额满足目标集群规格。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- `DescribeAvailableClusterVersion` 返回的 `Versions` 非空（有更高版本可升）— 用本文升级 Master
- 当前 K8s 版本将停止维护，或业务需新版 API/特性 — 逐版本升级（勿跳版本）
- Master 升级后节点未跟随（`CheckInstancesUpgradeAble` 显示节点版本落后于 Master）— 跳到 [§单独升级节点版本](#单独升级节点版本)

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterState = "Running"（非 Running 不允许升级）
```

### 资源检查

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）

#### 1. 查可升级版本

```bash
# Versions 为空表示已是最新
tccli tke DescribeAvailableClusterVersion --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 顶层返回 Versions + Clusters 字段；Versions 含目标版本，为空 [] 则无需升级
```

`DescribeAvailableClusterVersion.ClusterId` 在 API 层可选；本文单集群流程要求传入以限定目标集群。不传时返回当前地域的可升级版本信息，不限定单一集群。

| 字段 | Action | 必填 | 说明 |
|:-----|:-------|:----:|:-----|
| `ClusterId` | DescribeAvailableClusterVersion | 条件 | 当查询单个集群可升级版本时必填；地域级查询可省略 |

#### 2. 节点兼容性前置检查

`UpgradeType` 枚举: `reset` 重装升级 / `hot` 原地滚动小版本 / `major` 原地滚动大版本。

```bash
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --UpgradeType reset
# expected: 返回 ClusterVersion/LatestVersion/UpgradeAbleInstances[]/Total/UnavailableVersionReason；UpgradeAbleInstances[] 为空或 Total=0（无可升级节点=已最新）
```

#### 3. 确认无进行中的升级任务

`DescribeUpgradeTasks` 不接受 `--ClusterId`，按 Offset/Limit 翻页。

```bash
tccli tke DescribeUpgradeTasks --region ap-guangzhou --Offset 0 --Limit 20
# expected: UpgradeTasks[] 为空，或用返回的 ID 逐个 DescribeUpgradeTaskDetail --ID "<TASK_ID>" 查 UpgradePlans[].Status（非 Running）
```

#### 4. 备份 kubeconfig

升级不可回滚，失败需重建；`--filter` + `--output text` 提取纯 kubeconfig，无 `--output text` 会带引号导致 kubectl 报 JSON parse error。

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --filter "Kubeconfig" --output text > backup-kubeconfig.yaml
# expected: kubeconfig 文件生成（可直接 kubectl --kubeconfig backup-kubeconfig.yaml）
```

## 关键字段

> 两套 Action 入参不同，勿混装：`UpdateClusterVersion` 只升 Master 控制面；`UpgradeClusterInstances` 升节点并带任务生命周期。

### UpdateClusterVersion（Master 控制面）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| DstVersion | string | 是 | 目标版本，如 `1.34.1`，须在 `DescribeAvailableClusterVersion` 返回列表中 | `InvalidParameterValue` / `FailedOperation` |
| MaxNotReadyPercent | float | 否 | 升级容忍度，0-100，help 默认 `0`（最保守） | `InvalidParameterValue` |
| SkipPreCheck | boolean | 否 | 跳过前置检查，**危险**，默认 false | 跳过后可能升级失败 |
| ExtraArgs | object | 否 | Master 组件自定义参数（Etcd/KubeAPIServer/KubeControllerManager/KubeScheduler） | 各子字段校验 |

### UpgradeClusterInstances（节点跟随 / 任务生命周期）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| Operation | string | 是 | 升级任务生命周期：`create` 开始 / `pause` 停止 / `resume` 继续 / `abort` 终止（缺失报 `arguments are required: --Operation`） | `InvalidParameter` |
| UpgradeType | string | 否（`Operation=create` 时需设置） | 枚举：`reset`（重装，大/小版本）/ `hot`（原地滚动**小版本**）/ `major`（原地滚动**大版本**）。`CheckInstancesUpgradeAble` 前置检查须传同一值 | `InvalidParameterValue`（如对大版本传 `hot`） |
| InstanceIds | list | 否（`create` 时常需） | 待升级节点 CVM ID 列表 | `ResourceNotFound` |
| MaxNotReadyPercent | float | 否 | 升级容忍度，0-100 | `InvalidParameterValue` |
| SkipPreCheck | boolean | 否 | 跳过前置检查，**危险**，默认 false | 跳过后可能升级失败 |

> `DstVersion` 必须是 `DescribeAvailableClusterVersion` 返回的版本之一，不能用 `DescribeVersions` 的全量版本——后者含不可升级到的版本。`UpdateClusterVersion` **无** `Operation`/`UpgradeType`；节点路径的 `UpgradeType` 三值不可互换：`reset` 重装系统盘（兼容性最强）；`hot` 仅小版本原地滚动；`major` 仅大版本原地滚动。选错（如大版本传 `hot`）报 `InvalidParameterValue`。三条路径的决策见 [步骤 1](#步骤-1决策--选升级策略)。

## 操作步骤

> ⚠️ **高危操作**：集群版本升级**不可回滚**，失败需重建集群。升级前须备份 kubeconfig 并验证工作负载 API 兼容性。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 步骤 1：决策 — 选升级策略 {#步骤-1决策--选升级策略}

#### 为什么逐版本升级

- **逐版本 vs 跳版本**: 逐版本（1.30→1.32→1.34）每跳风险可控，API 弃用逐版本暴露；跳版本（1.30→1.34）一次性暴露多个版本的弃用，工作负载可能因 API 移除而崩溃
- **默认推荐**: 逐版本升级（生产环境适用：每跳风险可控，API 弃用逐版本暴露）
- **能回滚吗?**: 不能。集群版本升级不可回滚，只能继续升级到更高版本。节点版本可单独升级/降级，但 Master 不可

#### 选哪个 UpgradeType（reset / hot / major）

`UpgradeType` 决定节点跟随 Master 升级的方式，三值代表三条独立路径，**不可互换**：

| UpgradeType | 做什么 | 适用版本 | 节点影响 | 风险 |
|:------------|:-------|:---------|:---------|:-----|
| `reset` | 重装升级（节点系统盘重装到新版本） | 大版本 + 小版本 | 系统盘重装，数据盘保留但 Pod 重调度 | 高（重装不可逆，失败需重建节点） |
| `hot` | 原地滚动小版本热升级 | 仅小版本 | 滚动重启节点，不重装 | 低（原地，Pod 漂移少） |
| `major` | 原地滚动大版本升级 | 仅大版本 | 滚动重启节点，不重装 | 中（API 弃用风险，须先核兼容） |

- **reset vs hot/major**: reset 重装节点系统盘（兼容性最强但破坏性最大）；hot/major 原地滚动（保留节点，仅重启）
- **hot vs major**: 同为原地滚动，区别在版本跨度——`hot` 只能小版本（如 1.30.0→1.30.5），`major` 只能大版本（如 1.30→1.34）
- **默认推荐**: 小版本升级用 `hot`（风险最低）；大版本升级用 `major`（原地，比 reset 温和）；仅当 `hot`/`major` 兼容性检查不通过时回退 `reset`
- **选错会怎样**: 大版本升级传 `hot` 报 `InvalidParameterValue`；`CheckInstancesUpgradeAble` 的 `UpgradeType` 必须与 `UpgradeClusterInstances` 一致，否则兼容性检查结果不对应

> 三值各自的前置检查都走 `CheckInstancesUpgradeAble --UpgradeType <值>`，须与后续 `UpgradeClusterInstances --UpgradeType <值>` 传同一值。

### 步骤 2：前置检查

`UpgradeType` 须与步骤 3 的升级命令传同一值。小版本升级用 `hot`，大版本用 `major`，回退场景用 `reset`：

```bash
# 小版本前置检查（UpgradeType=hot）
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --UpgradeType hot
# expected: UpgradeAbleInstances[] 为空或 Total=0（节点已最新）；有不兼容先处理节点；UnavailableVersionReason 含原因

# 大版本前置检查（UpgradeType=major）——大版本升级必做，核 API 弃用兼容
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --UpgradeType major
# expected: UpgradeAbleInstances[] 兼容；若 UnavailableVersionReason 非空，先处理不兼容节点或回退 reset

# 回退场景：reset 重装前置检查（hot/major 不兼容时用）
tccli tke CheckInstancesUpgradeAble --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --UpgradeType reset
# expected: UpgradeAbleInstances[] 兼容（reset 兼容性最强）
```

可用 `Filters` 缩小节点范围。`Filters[].Name` 仅支持 `ip`、`instanceId`、`hostname`、`label`；对应的 `Value` 分别填写节点 IP、实例 ID、主机名或 Kubernetes label。

### 步骤 3：升级 Master

`UpdateClusterVersion` 必传 `ClusterId` + `DstVersion`（不带 `UpgradeType`，只升 Master 控制面；节点跟随升级见步骤 4 的 `UpgradeClusterInstances`）。按场景**二选一**：A 默认容忍度或 B 金丝雀（允许部分节点先升级）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `UpdateClusterVersion` 升的是同一个 Master，第二次会报版本已在升级中。集群版本升级**不可回滚**，只能继续升级到更高版本。

#### 选项 A：默认容忍度

```bash
tccli tke UpdateClusterVersion --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --DstVersion "<TARGET_VERSION>"
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 目标集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<TARGET_VERSION>` | 目标版本 | 须在可升级列表 | `tccli tke DescribeAvailableClusterVersion` → `Versions[]` |

#### 选项 B：金丝雀（允许部分节点 NotReady）

> **与 A 二选一，非在 A 之后执行**。设置较高容忍度，允许部分节点先升级（适合大规模集群）：

```bash
tccli tke UpdateClusterVersion --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --DstVersion "<TARGET_VERSION>" \
  --MaxNotReadyPercent 30
# expected: exit 0
```

> `MaxNotReadyPercent 30` 允许 30% 节点在升级期间 NotReady。值越大升级越快但风险越高，生产环境建议默认（低值）。

### 步骤 4：节点跟随升级 — 按 UpgradeType 选路径 {#步骤-4节点跟随升级--按-upgradetype-选路径}

Master 升级后节点未自动跟随时，用 `UpgradeClusterInstances` 按选定的 `UpgradeType` 升级节点。三值三条路径，命令结构相同，仅 `--UpgradeType` 不同：

```bash
# 路径 1: hot 小版本热升级（原地滚动，风险最低）
tccli tke UpgradeClusterInstances --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Operation create --UpgradeType hot \
  --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0, 返回 RequestId（异步任务已提交）

# 路径 2: major 大版本原地滚动（API 弃用风险，须先 CheckInstancesUpgradeAble --UpgradeType major 通过）
tccli tke UpgradeClusterInstances --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Operation create --UpgradeType major \
  --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0, 返回 RequestId

# 路径 3: reset 重装升级（hot/major 不兼容时回退；系统盘重装，数据盘保留）
tccli tke UpgradeClusterInstances --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Operation create --UpgradeType reset \
  --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0, 返回 RequestId
```

> 三路径的 `UpgradeType` 必须与步骤 2 `CheckInstancesUpgradeAble` 一致，否则兼容性检查结果不对应。`reset` 会重装节点系统盘（Pod 重调度），`hot`/`major` 原地滚动重启（Pod 漂移少）。
>
> `UpgradeType=reset` 时可传 `ResetParam`，在节点重装后重新加入集群时透传接入已有节点参数。`ResetParam` 中的重装、登录和高级设置须遵循 [接入已有实例字段约束](../nodes/instance-ops.md#接入已有实例字段约束)；其中逐实例覆盖项与 `InstanceIds[]` 等长、同序对应。

### 步骤 5：验证

异步操作，检查 ≥4 个维度：

```bash
# 轮询集群状态（升级中为 Upgrading，完成后回 Running）
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: 升级中 "Upgrading" → 完成后 "Running"
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 集群状态 | `DescribeClusterStatus` → `ClusterState` | `Upgrading` → `Running` |
| 版本号 | `DescribeClusters --ClusterIds '["<ID>"]'` → `ClusterVersion` | 等于 `<TARGET_VERSION>` |
| 升级任务 | `DescribeUpgradeTasks --Offset 0 --Limit 20` 取 `UpgradeTasks[].ID`，再 `DescribeUpgradeTaskDetail --ID "<ID>"` → `UpgradePlans[].Status` | `Succeed` |
| 节点版本 | `CheckInstancesUpgradeAble --ClusterId "<ID>" --UpgradeType reset` → `ClusterVersion`/`UpgradeAbleInstances[].Version`（`DescribeClusterInstances` 不返回节点 K8s 版本） | 节点 Version 与 Master 一致 |
| kubectl 可用 | `kubectl get nodes`（用 kubeconfig） | 节点列表返回 |

> 版本号带 `-tke` 后缀是正常形态（如 `1.34.1-tke.5`），`ClusterVersion`/`LatestVersion` 均用此格式。`UpdateClusterVersion --DstVersion` 传纯版本号（如 `1.34.1`），由服务端匹配 `-tke` 衍生版。

> 升级期间 `ClusterState=Upgrading`，可 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 暂停（不是回滚，注意 `ClusterID`/`PlanID` 均大写 ID）。暂停后恢复需重新触发。

## 清理 {#清理}

> **不可回滚**：集群版本升级后无法降级到旧版本。若升级失败导致集群不可用，只能 `DeleteCluster` 重建并用备份的 kubeconfig/配置恢复工作负载（升级前必备份，见 [集群备份](backup.md)；重建见 [删除集群](delete.md) + [创建集群](create.md)）。
>
> **维护窗口交叉**：手动升级前建议设维护窗口排除项，冻结 TKE 托管组件的自动升级，避免自动升级与手动升级撞期。维护窗口配置见 [维护窗口](maintenance-window.md)。升级期间 `ClusterState=Upgrading` 不受维护窗口控制（手动操作优先）。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Versions` 为空 `[]` | `DescribeAvailableClusterVersion` | 已是最新版本，无更高版本可升级 | 无需升级 |
| `FailedOperation` | `CheckInstancesUpgradeAble` 看不兼容节点 | 节点版本/组件不兼容目标版本 | 先升级节点或移除不兼容节点 |
| `ResourceInUse` | `DescribeUpgradeTasks --Offset 0 --Limit 20` 看是否有任务，有则 `DescribeUpgradeTaskDetail --ID "<ID>"` 查 `UpgradePlans[].Status` | 已有升级任务在执行中 | 等待或 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 后重试 |
| `InvalidParameterValue.DstVersion` | 核对 `DstVersion` | 版本号不在可升级列表 | 用 `DescribeAvailableClusterVersion` 返回的版本 |
| `UnsupportedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 `Running`（升级中/异常） | 等集群 `Running` 后重试 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `Upgrading` | `DescribeUpgradeTaskDetail --ID "<ID>"` → `UpgradePlans[].Status` + `GetUpgradeInstanceProgress --ClusterId "<ID>"` 查节点进度 | 某节点升级卡住 | `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 暂停，定位卡住节点 |
| 升级后部分节点版本不一致 | `CheckInstancesUpgradeAble --ClusterId "<ID>" --UpgradeType reset` → `UpgradeAbleInstances[].Version`（`DescribeClusterInstances` 不返回节点版本） | 节点未跟随升级 | `UpgradeClusterInstances` 单独升级节点 |
| `Running` 但 kubectl 报 API 版本弃用 | `kubectl get nodes --v=6` | 工作负载用了已弃用 API | 更新工作负载 YAML，移除弃用 API |
| 升级任务 `Failed` | `DescribeUpgradeTaskDetail` | 资源不足或组件冲突 | 查 TaskDetail，修复后重新 `UpdateClusterVersion` |

> 升级卡住超 30 分钟属异常，用 `DescribeUpgradeTaskDetail --ID "<ID>"` 看具体步骤，必要时 `CancelUpgradePlan --ClusterID "<ID>" --PlanID "<PLAN_ID>"` 后提工单附 RequestId。

## 单独升级节点版本 {#单独升级节点版本}

> Master 升级后节点未跟随升级时，用 `UpgradeClusterInstances` 对指定节点单独升级。这是异步任务型接口：`Operation` 控制任务生命周期，`UpgradeType` 仅在 `Operation=create` 时生效。

## 跨字段约束

| `Operation` | `UpgradeType` | `ResetParam` | 关系 |
|:------------|:--------------|:-------------|:-----|
| `create` | 必填：`hot`、`major` 或 `reset` | 仅 `UpgradeType=reset` 时可传 | 创建升级任务 |
| `pause` / `resume` / `abort` | 不传 | 不传 | 操作已有任务，不创建新升级方案 |

`InstanceIds` 是 `Operation=create` 的升级目标。`ResetParam` 不是所有创建任务的通用参数，只在节点重装后重新入群的 `reset` 路径使用。

`Operation` 枚举（任务生命周期，来自 API 定义）：

| Operation | 含义 |
|:----------|:-----|
| `create` | 开始一次升级任务（需配合 `UpgradeType` + `InstanceIds`） |
| `pause` | 暂停进行中的任务 |
| `resume` | 继续已暂停的任务 |
| `abort` | 终止任务 |

`Operation=create` 的三值命令演示见 [步骤 4](#步骤-4节点跟随升级--按-upgradetype-选路径)（hot/major/reset 三路径）。本段补 `MaxNotReadyPercent` 金丝雀变体：

```bash
# 金丝雀升级指定节点（UpgradeType 同步骤 4，加 MaxNotReadyPercent 容忍度）
tccli tke UpgradeClusterInstances --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Operation create \
  --UpgradeType hot \
  --InstanceIds '["<INSTANCE_ID>"]' \
  --MaxNotReadyPercent 30
# expected: exit 0, 返回 RequestId（异步任务已提交）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:-------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<INSTANCE_ID>` | 节点 CVM ID | `ins-xxxxxxxx` | `tccli tke DescribeClusterInstances`（旧版用 `--InstanceIds`/`--InstanceRole`，新版用 `--SortBy`/`--NeedTags`，两版入参不兼容；本文走旧版 2018-05-25）→ 节点 InstanceId |

任务状态用 `GetUpgradeInstanceProgress` 轮询；暂停/终止用 `--Operation pause`/`abort`。完整节点运维（接入/删除/升级）见 [节点实例操作](../nodes/instance-ops.md)。

### 升级任务进度与计划控制

> 节点升级/Master 升级都是异步任务，三个 Action 覆盖「查进度—查详情—停计划」控制面。参数名大小写以各 Action 入参为准（注意 `ClusterID`/`PlanID` 大写 vs `ClusterId` 小写）。

#### 1. 查节点升级进度

`ClusterId` 小写 + Offset/Limit 分页。

```bash
tccli tke GetUpgradeInstanceProgress --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Offset 0 --Limit 20
# expected: 集群无升级任务报 ResourceNotFound；有任务时返回 Total/Done/LifeState/Instances[]/ClusterStatus
```

#### 2. 查升级任务详情

`ID` 为任务 ID，来自 `DescribeUpgradeTasks[].ID`。

```bash
tccli tke DescribeUpgradeTaskDetail --region ap-guangzhou \
  --ID "<TASK_ID>" --Offset 0 --Limit 20
# expected: exit 0，返回 UpgradePlans[]+TotalCount（UpgradePlans[].Status = Pending/Processing/Running/Succeed/Failed/Cancelled）
```

#### 3. 暂停升级计划

`ClusterID`/`PlanID` 均大写；暂停非回滚，恢复需重新触发。

```bash
tccli tke CancelUpgradePlan --region ap-guangzhou \
  --ClusterID "<CLUSTER_ID>" --PlanID <PLAN_ID>
# expected: CAM 拦截 UnauthorizedOperation.CamNoAuth；授权后 exit 0
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<TASK_ID>` | 升级任务 ID | `tccli tke DescribeUpgradeTasks --Offset 0 --Limit 20` → `UpgradeTasks[].ID` |
| `<PLAN_ID>` | 升级计划 ID（整数） | `DescribeUpgradeTaskDetail --ID "<TASK_ID>"` → `UpgradePlans[].ID`（字段名是 `ID`，非 `PlanID`） |

> ⚠️ 大小写不一致是真实契约：`GetUpgradeInstanceProgress`/`DescribeUpgradeTaskDetail` 用任务语义参数，`CancelUpgradePlan` 的 `ClusterID`/`PlanID` 大写。切换接口前用 `tccli tke <Action> help` 核对入参名，不要沿用其他 Action 的参数名。

## 收尾确认

```bash
# Master 版本 + 全节点版本 + 升级任务 Succeed 三项一并核对（先分别查状态/版本/任务，再确认三者均达成）
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{state:ClusterStatus,version:ClusterVersion}"
# expected: state=Running, version=目标版本（如 1.34.1）

tccli tke CheckInstancesUpgradeAble --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --UpgradeType reset \
  --filter "ClusterVersion"
# expected: 节点版本与 Master 一致（ClusterVersion=目标版本）→ Master+节点版本一致

tccli tke DescribeUpgradeTasks --region ap-guangzhou --Offset 0 --Limit 20 \
  --filter "UpgradeTasks[0].ID"
# expected: 取最新任务 ID，再 DescribeUpgradeTaskDetail --ID "<ID>" 核 UpgradePlans[].Status=Succeed
```

> Master 版本=目标 + 全节点版本=目标 + 升级任务 `Succeed` 三项均符合预期 = 升级完成。须分别核对集群状态/版本号/任务/节点版本，并确认三项均达成（单查任一项不足以证明升级完成——Master 升级而节点未跟随，或任务未 Succeed，均未完成）。**升级不可回滚**，失败只能 `DeleteCluster` 重建（见 [§清理](#清理)）。

## 下一步

- [节点版本升级](../nodes/instance-ops.md) — Master 升级后单独升级节点
- [集群状态机](../reference/states.md) — `Upgrading` 等状态含义
- [创建集群](create.md) — 升级失败需重建时参考
- [故障排查](../troubleshooting.md) — 升级卡住的诊断路径
- [独立集群 Master 运维](master-ops.md) — 独立集群扩缩容 Master/etcd 节点
