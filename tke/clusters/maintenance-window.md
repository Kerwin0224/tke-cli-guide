---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置集群维护窗口

> 设置集群级与全局维护窗口，控制自动升级和运维操作的发生时段，并可配置排除项跳过特定窗口。
> 控制台: [容器服务 - 集群](https://console.cloud.tencent.com/tke2/cluster)

> 本文档所有 Action 属 **TKE 2018-05-25**（旧版独有，新版无）。注意集群级用 `ClusterID`（大写 ID）寻址，全局级用 `TargetRegions` + `ID` 寻址——契约不同，不可类推。

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [集群生命周期](https://cloud.tencent.com/document/product/457/32188) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 概述

维护窗口（MaintenanceWindow）限定 TKE **计划升级**（控制面与集群内系统组件等）可执行的时间段。两层模型：

- **集群级维护窗口**（`CreateClusterMaintenanceWindowAndExclusions`）：绑定单个 `ClusterID`，仅对该集群生效。
- **全局维护窗口**（`CreateGlobalMaintenanceWindowAndExclusions`）：用 `TargetRegions` 指定一批地域（`"*"` 表示全部地域），对指定地域内集群生效。

**排除项（Exclusions）** = **禁止计划升级**的时间段（业务高峰封网等），不是「放开维护窗口」。

| 半常量 | 约束 |
|:-------|:-----|
| 维护时长 | ≥ **2** 小时；本篇 API 字段 `Duration` 取值 **2~12**（小时）；控制台文案另写上限 24，以 api/本篇字段表为准 |
| 排除项数量 | 单地域全局 / 单集群各最多 **3** 个；单条排除起止跨度 ≤ **7** 天 |
| 可用窗口下限 | 每 **30** 天周期内至少保留 **4** 小时维护窗口 |
| 时区 | 维护起始时间按 **东八区** |
| 优先级 | **维护窗口**：集群级 > 全局（含「指定地域」>「全部地域」）；**排除项**：集群排除 ∪ 全局排除（合集） |
| 紧急例外 | 安全漏洞等紧急变更**可能不遵循**维护窗口，直接收敛风险 |

> 启用计划升级前须已配地域级或集群级维护窗口，否则无法启用。计划升级覆盖托管控制面与集群内系统组件（如 coredns），**暂不包括用户节点组件**。

> 配额：无额外限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 业务有明确低峰期，要把计划升级限制在此时段 — 用本文配集群级或全局维护窗口
- 大促等需**禁止**计划升级 — 用排除项划出禁止升级时段（非「跳过窗口约束」）
- 多地域一批集群要统一运维时段 — 用全局窗口 + `TargetRegions`

## 决策依据

### 用集群级还是全局

| 方案 | 最适合 | 主要限制 |
|:-----|:-------|:---------|
| 集群级窗口 | 单个核心集群需独立运维时段 | 逐集群配置，集群多时繁琐 |
| 全局窗口 | 同地域一批集群统一时段 | 对已配集群级窗口的集群不生效 |

### 时段选择

`MaintenanceTime` 为窗口起点（`HH:MM:SS`），`Duration` 为持续时长（整数，取值 2~12，单位为小时），`DayOfWeek` 为星期枚举数组。选择原则：避开业务高峰，建议低峰期（如 `22:00:00`，Duration `2`）。现网常见配置：`22:00:00` + Duration `2` + 周二/周四/周五。

## 配置项

> 注意集群级用 `ClusterID`（大写 ID），全局级用 `TargetRegions` + `ID` 寻址——契约不同，不可类推。

| 字段 | 类型 | 必填 | 默认值 | 有效值 | 填错的影响 |
|:-----|:-----|:----:|:-------|:-------|:-----------|
| `ClusterID` | String | 是（集群级） | — | 已存在的集群 ID | `Unknown options` 或资源不存在 |
| `MaintenanceTime` | String | 是 | — | `HH:MM:SS`，如 `22:00:00` | 格式不符被拒 |
| `Duration` | Integer | 是 | — | 2~12（小时） | 超范围被拒 |
| `DayOfWeek` | Array of String | 是 | — | `MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU` | 枚举错被拒 |
| `TargetRegions` | Array of String | 是（全局级） | — | 地域如 `ap-guangzhou`，或 `"*"` 表全部 | 地域无效被拒 |
| `Exclusions[].Name` | String | 否 | — | 排除项名称 | — |
| `Exclusions[].StartAt` | String | 否 | — | `HH:MM:SS` | 排除项不生效 |
| `Exclusions[].EndAt` | String | 否 | — | `HH:MM:SS` | 排除项不生效 |
| `ID` | Integer | 是（全局级 Modify/Delete） | — | 全局窗口自增主键 | 操作错误目标 |

> `DayOfWeek` 枚举值全集：`MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU`。

## 应用

> ⚠️ **高危操作**：维护窗口设置不当 → 自动升级中断业务高峰；确认维护时段与业务低峰匹配，排除了封网期。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 步骤 1：创建集群级维护窗口

```bash
tccli tke CreateClusterMaintenanceWindowAndExclusions --region <REGION> \
  --ClusterID <CLUSTER_ID> --MaintenanceTime "22:00:00" --Duration 2 --DayOfWeek '["TU","TH","FR"]'
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<CLUSTER_ID>` | 集群 ID | 已存在集群 | `tccli tke DescribeClusters --region <REGION>` |

> ⚠️ **写操作 CAM 授权**：`CreateClusterMaintenanceWindowAndExclusions` 要求目标集群带特定标签（要求 `billing` 标签）才放行——CAM 策略匹配 `qcs:resource_tag` 含 `billing`。带标签的集群调用返回 RequestId（成功）。不带标签的集群被拒，错误码与修复见 [§故障恢复](#故障恢复)。

### 步骤 2：创建全局维护窗口

```bash
tccli tke CreateGlobalMaintenanceWindowAndExclusions --region <REGION> \
  --MaintenanceTime "22:00:00" --Duration 2 --DayOfWeek '["TU","WE"]' \
  --TargetRegions '["ap-guangzhou"]'
# expected: exit 0, 返回 RequestId
```

`TargetRegions: ["*"]` 表示对所有地域生效。全局窗口创建后系统分配自增 `ID`，后续 Modify/Delete 用此 `ID` 寻址（与集群级用 `ClusterID` 寻址不同）。

### 步骤 3：修改窗口

```bash
# 集群级：覆盖式更新（ClusterID + 新参数）
tccli tke ModifyClusterMaintenanceWindowAndExclusions --region <REGION> \
  --ClusterID <CLUSTER_ID> --MaintenanceTime "03:00:00" --Duration 3 --DayOfWeek '["SU"]'

# 全局级：用 ID 寻址
tccli tke ModifyGlobalMaintenanceWindowAndExclusions --region <REGION> \
  --ID <GLOBAL_WINDOW_ID> --MaintenanceTime "04:00:00" --Duration 4 \
  --DayOfWeek '["SU"]' --TargetRegions '["ap-guangzhou"]'
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<GLOBAL_WINDOW_ID>` | 全局窗口主键 | 整数 | `DescribeGlobalMaintenanceWindowAndExclusions` 取 `ID` 字段 |

> Modify 为覆盖式：未传的 `Exclusions` 等字段会被清空，需整体传入而非增量。

## 验证

```bash
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <REGION> --Limit 20
# expected: exit 0, TotalCount >= 1, 含目标 ClusterID
```
```json
{
    "MaintenanceWindowAndExclusions": [
        {
            "MaintenanceTime": "22:00:00",
            "Duration": 2,
            "ClusterID": "cls-example",
            "DayOfWeek": ["TU", "TH", "FR"],
            "Region": "ap-guangzhou",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.24.4",
            "Exclusions": null
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

```bash
tccli tke DescribeGlobalMaintenanceWindowAndExclusions --region <REGION> --Limit 20
# expected: exit 0, TotalCount >= 1, 含目标 ID
```
```json
{
    "TotalCount": 1,
    "MaintenanceWindowAndExclusions": [
        {
            "TargetRegions": ["*"],
            "MaintenanceTime": "22:00:00",
            "Duration": 2,
            "DayOfWeek": ["TU", "WE"],
            "Exclusions": [],
            "ID": 4
        }
    ],
    "RequestId": "..."
}
```

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 集群窗口存在 | `DescribeClusterMaintenanceWindowAndExclusions` | `TotalCount >= 1`，含目标 `ClusterID` |
| 时段生效 | 同上，查 `MaintenanceTime`/`Duration`/`DayOfWeek` | 与创建参数一致 |
| 全局窗口存在 | `DescribeGlobalMaintenanceWindowAndExclusions` | `TotalCount >= 1`，含目标 `ID` |
| 排除项生效 | 同上，查 `Exclusions` | 含配置的排除时段 |

> ⚠️ **Filter 名受限**：`DescribeClusterMaintenanceWindowAndExclusions` 的 `Filters[].Name` 实际接受 **`ClusterID`**（大写 ID；无窗口时 `TotalCount=0`）。`cluster-id` / `clusterId` / `ClusterId` 报 `InvalidParameter`（`invalid filter name`）。无窗口时仍可全量 `--Limit` + 客户端按 `ClusterID` 过滤。

## 回滚

```bash
# 集群级：删除该集群的维护窗口
tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <REGION> --ClusterID <CLUSTER_ID>
# expected: exit 0

# 全局级：按 ID 删除
tccli tke DeleteGlobalMaintenanceWindowAndExclusion --region <REGION> --ID <GLOBAL_WINDOW_ID>
# expected: exit 0
```

```bash
# 验证已删除
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <REGION> --Limit 20
# expected: 目标 ClusterID 不再出现在列表中
```

> 删除窗口不影响集群本身，仅恢复"无托管运维时段约束"状态，下次自动升级可能随时发生。修改时段而非删除可用 Modify 覆盖。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `AuthFailure.UnauthorizedOperation` (tke:CreateClusterMaintenanceWindowAndExclusions) | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' --filter "Clusters[0].TagSpecification[*].Tags[*]" --output text` | CAM 策略要求集群带特定标签（要求 `billing` 标签，匹配 `qcs:resource_tag`），目标集群未带匹配标签 | 给集群加上授权要求的 `billing` 标签，或申请 `tke:CreateClusterMaintenanceWindowAndExclusions` 权限。此为环境限制，非命令错误 |
| `UnauthorizedOperation.CamNoAuth` Code=11008 (tke:CreateGlobalMaintenanceWindowAndExclusions) | 同上查账号 CAM | CAM 策略要求请求带 `billing` 标签（`qcs:request_tag`），但全局窗口入参无 Tags 字段，无法满足 → 必拒 | 申请 `tke:CreateGlobalMaintenanceWindowAndExclusions` 权限。此为账号级限制 |
| `InvalidParameter: invalid filter name` | 检查 `Filters[].Name` | Describe 的 Filter 合法 name 与猜测不符 | 移除 `Filters`，改用全量查询 + 客户端过滤 |
| `Unknown options: --ClusterId` | `tccli tke <Action> help` | 参数名拼错（是 `ClusterID` 大写，非 `ClusterId`） | 改用 `help` 输出的真实参数名 |
| `FailedOperation` (message: `maintenance window already exists for cluster ...`) | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <REGION> --Limit 20` 核对该 `ClusterID` 是否已在列表中 | 该集群已存在维护窗口，重复 `Create` 被拒（业务冲突，非 CAM 拒绝——CAM 已放行才会到此错误） | 改用 `ModifyClusterMaintenanceWindowAndExclusions --ClusterID <ID>` 覆盖式更新现有窗口；或先 `DeleteClusterMaintenanceWindowAndExclusion` 再 `Create` |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| 全局窗口已建但某集群仍随时升级 | `DescribeClusterMaintenanceWindowAndExclusions` 查该集群 | 该集群已配集群级窗口，优先于全局窗口 | 调整集群级窗口或删除它使全局窗口生效 |
| 排除项未生效 | 查 `Exclusions` 的 `StartAt`/`EndAt` 格式 | 时间格式非 `HH:MM:SS` 或与窗口时段无交集 | 修正排除项时间格式，确保落在窗口内 |
| Modify 后排除项丢失 | 对比 Modify 前后 `Exclusions` | Modify 为覆盖式，未传字段被清空 | 整体传入所有字段，非增量更新 |

> CAM 拒绝样本（ap-guangzhou，标签不匹配）：
> `code:UnauthorizedOperation.CamNoAuth message:[QCloudError] Code=11008 ... you are not authorized to perform operation (tke:CreateGlobalMaintenanceWindowAndExclusions) ... with or without condition:[{"condition":{"key":"qcs:resource_tag","value":["billing&<标签值>"],"ope":"for_any_value:string_equal"},"effect":"allow"}, ...]`
> （错误消息会列出 CAM 策略要求的标签 key/value，按其给集群打标签即可放行集群级写操作；全局级因入参无 Tags 字段仍被拒）

## 收尾确认

```bash
# 集群级窗口 + 全局窗口 + 排除项一次性核对（三层协同效果）
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <REGION> --Limit 20 \
  --filter "MaintenanceWindowAndExclusions[?ClusterID=='<CLUSTER_ID>'].{time:MaintenanceTime,dur:Duration,days:DayOfWeek,excl:Exclusions}"
# expected: 集群级时段（如 time=22:00:00, dur=2, days=["TU","TH","FR"]）+ Exclusions 含配置的排除项

tccli tke DescribeGlobalMaintenanceWindowAndExclusions --region <REGION> --Limit 20 \
  --filter "MaintenanceWindowAndExclusions[0].{time:MaintenanceTime,dur:Duration,days:DayOfWeek,regions:TargetRegions,excl:Exclusions}"
# expected: 全局窗口时段 + TargetRegions 含目标地域 → 集群级与全局窗口协同生效
```

> 集群级窗口优先于全局窗口——同集群有两层时以集群级为准。汇总核对须确认目标集群的窗口来自预期层级（集群级未配时才回落到全局），且排除项 `StartAt`/`EndAt` 落在窗口时段内才会真正跳过本次自动升级（排除项能否跳过取决于与窗口时段的交集；无交集则不生效，见 [§故障恢复](#故障恢复)）。

## 下一步

- [升级集群版本](upgrade.md) — 维护窗口控制自动升级时段，手动升级见此
- [集群概览](index.md) — 返回集群章节索引
- [集群查询](query.md) — 查询集群详情与状态
