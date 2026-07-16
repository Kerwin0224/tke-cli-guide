---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理应用发布

> 用 Helm Release 在集群内部署应用。创建、升级、回滚、卸载。异步操作。
> 控制台: [容器服务 - 应用管理](https://console.cloud.tencent.com/tke2/helm)
>
> 官方文档：[组件与应用概述](https://cloud.tencent.com/document/product/457/81234)
>
> 配额：Release 受集群配额限制（单地域集群数默认 20），无额外 Release 数限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- `DescribeClusterReleases` → 目标命名空间无应用 Release，需 `CreateClusterRelease` 部署 Chart
- `DescribeClusterReleases` → `Status=failed` 或长时间 `pending-upgrade`，Release 创建/升级后状态异常
- `DescribeClusterReleaseHistory` 显示需回滚到历史版本，`RollbackClusterRelease` 后版本未生效 — 看 [故障恢复](#故障恢复)


## 概述

Release 是 TKE 封装的 Helm Release，管理应用生命周期。一个 Release = 一个 Chart 的某次部署实例。

| 操作 | 接口 | 作用 |
|:-----|:-----|:-----|
| 创建 | `CreateClusterRelease` | 部署 Chart 到集群 |
| 升级 | `UpgradeClusterRelease` | 升级 Chart 版本或改 Values |
| 回滚 | `RollbackClusterRelease` | 回滚到历史版本 |
| 查询 | `DescribeClusterReleases` | 查看 Release 列表与状态 |
| 历史 | `DescribeClusterReleaseHistory` | 查看 Release 修订历史 |
| 卸载 | `UninstallClusterRelease` | 移除 Release |

操作是**异步**的：接口返回即提交，Release 就绪需轮询 `DescribeClusterReleases` 直到 `Status=deployed`。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 查看已有 Release（返回 Name/Namespace/Revision/Status/ChartName/ChartVersion）
tccli tke DescribeClusterReleases --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --Limit 5 \
  --filter "ReleaseSet[].{name:Name,ns:Namespace,rev:Revision,status:Status,chart:ChartName,ver:ChartVersion}"
# expected: Release 列表，Status 含 deployed
```

```text
eniipamd	kube-system	4	deployed	eniipamd	3.11.0
kubejarvisservice	kube-system	2	deployed	kubejarvisservice	1.0.7
gatekeeper	kube-system	4	deployed	gatekeeper	1.3.0
```

```bash
# 查询 TKE 内置可用 Chart（Kind/Arch/ClusterType 过滤；用于选 Chart 名与版本）
# GetTkeAppChartList 的 ClusterType 仅 tke / eks（非 CreateCluster 的 MANAGED_CLUSTER）
tccli tke GetTkeAppChartList --region ap-guangzhou \
  --Kind "<CHART_KIND>" --ClusterType tke
# expected: exit 0, 返回 AppCharts[]（含内置 Chart 名/版本/架构；匹配为空时 AppCharts=[]）
```

## 关键字段

> 完整入参以 `tccli tke CreateClusterRelease help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| Name | string | 是 | Release 名，命名空间内唯一 | `InvalidParameterValue` |
| Namespace | string | 是 | 部署的命名空间 | `InvalidParameterValue` |
| Chart | string | 是 | Chart 名 | `InvalidParameterValue` |
| ChartVersion | string | 否 | Chart 版本 | `InvalidParameterValue` |
| ChartFrom | string | 否 | Chart 来源：`tke-market`（默认）/ `other` | `InvalidParameterValue` |
| ChartRepoURL | string | 否 | Chart 仓库 URL（`ChartFrom=other` 时） | `InvalidParameterValue` |
| Values | string | 否 | 自定义 Values（JSON） | `InvalidParameterValue` |
| Username | string | 否 | 私有仓库用户名 | `UnauthorizedOperation` |
| Password | string | 否 | 私有仓库密码 | `UnauthorizedOperation` |
| ClusterType | string | 否 | 集群类型：`tke` / `eks` / `tkeedge` / `external` | — |

> `ChartFrom` **仅** `tke-market`（应用市场，默认）/ `other`（第三方 repo，需 `ChartRepoURL`）。**不是** `tke`/`repo`。私有仓库用 `Username`/`Password`。`ChartFrom=tke-market` 时 `ChartNamespace` 须非空（来自 `DescribeProducts`）。

## 操作步骤

### 步骤 1：决策 — Chart 来源 {#chart-来源决策}

#### 为什么选 tke-market vs other

- **tke-market**: TKE 应用市场 Chart，无需配仓库；须传 `ChartNamespace`
- **other**: 自建或第三方 Helm 仓库，需 `ChartRepoURL`（Chart 参数可为 `*.tgz` 下载地址）
- **默认推荐**: 官方应用用 `tke-market`；自定义应用用 `other`
- **能换源吗?**: 能，`UpgradeClusterRelease` 改 Chart 来源

### 步骤 2：创建 Release

`CreateClusterRelease` 必传 `ClusterId`/`Name`/`Namespace`/`Chart`；`ChartFrom` 可选（默认 `tke-market`）。按场景**二选一**：A 最小化（应用市场）或 B 增强（外部仓库+自定义 Values）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateClusterRelease` 会装**两个同名 Release（命名空间内冲突）或两份 Release**。Release 创建后改配置（版本/Values）用 `UpgradeClusterRelease`，**禁用第二次 `CreateClusterRelease` 改配置**。

#### 选项 A：最小化（应用市场 tke-market）

```bash
tccli tke CreateClusterRelease --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>" \
  --Chart "<CHART_NAME>" --ChartFrom tke-market --ChartNamespace "<CHART_NS>"
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<RELEASE_NAME>` | Release 名 | 命名空间内唯一 | 自定义，如 `my-app` |
| `<NAMESPACE>` | 命名空间 | K8s 命名空间 | 自定义，如 `default` |
| `<CHART_NAME>` | Chart 名 | 须存在 | `tccli tke GetTkeAppChartList` |

#### 选项 B：增强（外部仓库 + 自定义 Values）

> **与 A 二选一，非在 A 之后执行**。用 `ChartFrom other` 指外部仓库，配 URL/版本/Values/认证。

```bash
tccli tke CreateClusterRelease --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>" \
  --Chart "<CHART_NAME>" --ChartFrom other \
  --ChartRepoURL "https://charts.example.com" --ChartVersion "1.2.0" \
  --Values '{"replicaCount":3}' \
  --Username "<REPO_USER>" --Password "<REPO_PASS>"
# expected: exit 0
```

### 步骤 3：升级 — 升级版本

```bash
tccli tke UpgradeClusterRelease --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>" \
  --ChartVersion "<NEW_VERSION>"
# expected: exit 0
```

### 步骤 4：回滚 — 回滚

```bash
# 查历史版本
tccli tke DescribeClusterReleaseHistory --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>"
# expected: 修订历史列表

# 回滚到某修订版
tccli tke RollbackClusterRelease --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>"
# expected: exit 0
```

### 步骤 5：验证

```bash
tccli tke DescribeClusterReleases --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "ReleaseSet[?Name=='<RELEASE_NAME>'].{name:Name,rev:Revision,status:Status,chart:ChartName,ver:ChartVersion}"
# expected: Status="deployed", Revision 递增
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| Release 状态 | `DescribeClusterReleases` → `Status` | `deployed` |
| 版本一致 | `DescribeClusterReleases` → `ChartVersion` | 等于目标版本 |
| 修订号 | `DescribeClusterReleases` → `Revision` | 创建=1，升级递增 |
| 资源就绪 | `kubectl get all -n <NAMESPACE>` | Release 管理的资源 Ready |
| 回滚生效 | `DescribeClusterReleaseHistory` | 回滚后 Revision 指向目标 |

> `Status` 枚举：`deployed`/`failed`/`pending-upgrade`/`pending-rollback`。`deployed` 为终态成功。

## 清理

> **副作用警告**：卸载 Release 会删除其部署的所有 K8s 资源（Deployment/Service/ConfigMap 等）。`UninstallClusterRelease` 默认不保留历史。
>
> ⚠️ **高危操作**：回滚到不兼容的历史版本可能导致应用版本混乱；误执行 `UninstallClusterRelease` 会永久删除所有关联 K8s 资源，数据不可恢复。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

```bash
# 1. 卸载
tccli tke UninstallClusterRelease --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>"
# expected: exit 0

# 2. 验证已卸载
tccli tke DescribeClusterReleases --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "ReleaseSet[?Name=='<RELEASE_NAME>']"
# expected: 空数组
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameterValue.Chart` | `GetTkeAppChartList` 查可用 Chart | Chart 名错或不存在 | 用存在的 Chart 名 |
| `ResourceNotFound` | `DescribeClusters` 核对 ID | ClusterId 错 | 确认集群 ID |
| `UnauthorizedOperation` | 查仓库凭证 | 私有仓库用户名/密码错 | 核对 `Username`/`Password` |
| `ResourceInUse` | `DescribeClusterReleases` 看是否已存在 | Release 名已占用 | 换名或先卸载 |
| `FailedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Status=failed` | `DescribeClusterReleases` → `Description` | Chart 模板错或 Values 不兼容 | 查 Description，修正 Values 或 Chart |
| 长时间 `pending-upgrade` | `kubectl get pods -n <NAMESPACE>` | 升级中 Pod 滚动未完成 | 等；查 Pod 事件定位卡住 |
| 升级后版本未变 | `DescribeClusterReleases` → `ChartVersion` | 升级未完成或版本号同 | 等异步完成；确认版本号不同 |
| 回滚后资源异常 | `kubectl get all -n <NAMESPACE>` | 回滚到不兼容版本 | 查历史，回滚到更早版本 |

## Release 详情与灰度序列

> Release 详情/待处理查询，与灰度发布序列（RollOutSequence）管理。

### Release 查询

```bash
# 查询 Release 详情
tccli tke DescribeClusterReleaseDetails --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ID "<RELEASE_ID>"
# expected: exit 0, Release 详情

# 查询待处理 Release
tccli tke DescribeClusterPendingReleases --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 待处理 Release 列表

# 取消 Release
tccli tke CancelClusterRelease --ClusterId "<CLUSTER_ID>" --region <REGION> --ID "<RELEASE_ID>"
# expected: exit 0
```

### 灰度发布序列（RollOutSequence）

> 灰度发布按节点标签分批次滚动，控制升级节奏。

```bash
# 查询灰度发布序列
tccli tke DescribeRollOutSequences --region <REGION> --Limit 10
# expected: exit 0, Sequences[] 含 Name/SequenceFlows[]
```
```json
{
    "Sequences": [
        {
            "Name": "example-sequence",
            "SequenceFlows": [
                {"Tags": [{"Key": "Env", "Value": ["prod"]}]}
            ]
        }
    ]
}
```

```bash
# 创建灰度发布序列 (Name + SequenceFlows[] 按标签分批)
tccli tke CreateRollOutSequence --region <REGION> \
  --Name "<SEQUENCE_NAME>" --SequenceFlows '[{"Tags":[{"Key":"<K>","Value":["<V>"]}]}]'
# expected: exit 0, 返回 ID

# 修改灰度序列 (按 ID)
tccli tke ModifyRollOutSequence --region <REGION> --ID <SEQUENCE_ID> --Name "<NEW_NAME>"
# expected: exit 0

# 删除灰度序列 (按 ID)
tccli tke DeleteRollOutSequence --region <REGION> --ID <SEQUENCE_ID>
# expected: exit 0
```

> `RollOutSequence` 用 `ID`（Integer）定位，`SequenceFlows[]` 按节点 `Tags` 分批（每批一个 flow）。集群/序列标签管理用 `DescribeClusterRollOutSequenceTags`/`ModifyClusterRollOutSequenceTags`。

```bash
# 查询集群灰度序列标签（Filters + Offset/Limit 分页）
tccli tke DescribeClusterRollOutSequenceTags --region ap-guangzhou \
  --Offset 0 --Limit 20
# expected: exit 0，返回 ClusterTags[]+TotalCount

# 修改集群灰度序列标签（ClusterID 大写 + Tags[] 覆盖）
tccli tke ModifyClusterRollOutSequenceTags --region ap-guangzhou \
  --ClusterID "<CLUSTER_ID>" \
  --Tags '[{"Key":"env","Value":"canary"}]'
# expected: CAM 拦截 UnauthorizedOperation.CamNoAuth；授权后 exit 0
```

> ⚠️ `ModifyClusterRollOutSequenceTags` 用大写 `ClusterID`（区别于多数 TKE 接口的小写 `ClusterId`），`Tags[]` 是覆盖式整体更新。`DescribeClusterRollOutSequenceTags` 不需 ClusterId，按 Offset/Limit 翻页。两者参数见各 Action 的 `help --detail`。灰度序列用这些标签按节点 `Tags` 分批发布。

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli管Release生命周期，kubectl查部署资源状态；helm为能力边界（tccli无helm原生命令），非tccli边界 -->
```bash
# 汇总核对三项：Release Status=deployed + 历史版本数 + 关联资源 Ready
# 1. Release 已部署（上文已查 Status/版本/Revision，此处汇总）
tccli tke DescribeClusterReleases --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "ReleaseSet[?Name=='<RELEASE_NAME>'].{name:Name,status:Status,rev:Revision}"
# expected: status=deployed

# 2. 历史版本数（回滚/升级后核对修订号递增）
tccli tke DescribeClusterReleaseHistory --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Name "<RELEASE_NAME>" --Namespace "<NAMESPACE>" \
  --filter "Total"
# expected: Total ≥1（创建=1，升级/回滚后递增）

# 3. 关联资源 Ready（端到端：Release 管理的 K8s 资源真就绪）
kubectl get all -n <NAMESPACE> -l app.kubernetes.io/instance=<RELEASE_NAME> \
  --no-headers | awk '{print $1, $2}' | head -10
# expected: 资源列表非空且 STATUS 无 Failed → 应用发布闭环完成
```

> Release deployed + 历史版本数 ≥1 + 关联资源 Ready = 配置完成。上文验证段查 Release 字段，此处汇总 Release 状态 + 修订历史 + 部署资源就绪三项，确认应用可用。

---

## 下一步

- [插件管理](../addons/manage.md) — 插件本质是 Release
- [创建集群](../clusters/create.md) — 建集群后部署应用
- [故障排查](../troubleshooting.md) — Release 失败诊断
