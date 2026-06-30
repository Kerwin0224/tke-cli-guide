---
doc_type: How-to
subtype: 6B
fused: false
---
# 开启集群审计

> 开启/关闭集群审计日志。审计记录所有 API Server 操作到 CLS（日志服务），用于合规与事故追溯。配置型操作。

## 概述

审计日志记录"谁在什么时候对什么资源做了什么操作"。开启后，集群 API Server 的所有操作（kubectl/tccli/控制台）落盘到 CLS 日志集。

| 状态 | 含义 | 查询 |
|:-----|:-----|:-----|
| 开启 | API 操作记录到 CLS | `DescribeClusterStatus` → `ClusterAuditEnabled=true` |
| 关闭 | 不记录 | `ClusterAuditEnabled=false` |

> 审计日志存储在 CLS 服务（跨产品），TKE 无 `DescribeClusterAuditLog` 接口——查日志须到 CLS 控制台或用 `tccli cls` 检索。

## 决策依据

#### 为什么开启审计

- **开启 vs 关闭**: 开启后所有 API 操作可追溯（合规、事故定位）；关闭省 CLS 存储费
- **默认推荐**: 生产环境必须开启；测试集群可关
- **能关闭吗?**: 能，`DisableClusterAudit`。但已存的审计日志保留在 CLS

## 配置项

> 来源：`tccli tke EnableClusterAudit --generate-cli-skeleton`（实测）。需先在 CLS 创建日志集与主题。

| 字段 | 类型 | 必填 | 默认值 | 有效值 | 填错的影响 |
|:------|------|:--------:|:------:|-------|-----------|
| ClusterId | string | 是 | — | `cls-xxxxxxxx` | `ResourceNotFound` |
| LogsetId | string | 是 | — | CLS 日志集 ID | `ResourceNotFound` |
| TopicId | string | 是 | — | CLS 日志主题 ID | `ResourceNotFound` |
| TopicRegion | string | 是 | — | CLS 主题地域，如 `ap-guangzhou` | 地域不匹配 |

> `LogsetId`/`TopicId` 须先在 CLS 服务创建。`TopicRegion` 是 CLS 主题所在地域，须与日志集一致。

## 应用

### 前置：创建 CLS 日志集与主题

```bash
# 1. 创建 CLS 日志集
tccli cls CreateLogset --region <REGION> --LogsetName "tke-audit-<CLUSTER_ID>"
# expected: 返回 LogsetId

# 2. 创建日志主题（在日志集下）
tccli cls CreateTopic --region <REGION> --LogsetId "<LOGSET_ID>" --TopicName "audit"
# expected: 返回 TopicId
```

### 开启审计

```bash
tccli tke EnableClusterAudit --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogsetId "<LOGSET_ID>" --TopicId "<TOPIC_ID>" --TopicRegion "ap-guangzhou"
# expected: exit 0
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<LOGSET_ID>` | CLS 日志集 ID | CLS 创建返回 | `tccli cls CreateLogset` |
| `<TOPIC_ID>` | CLS 日志主题 ID | CLS 创建返回 | `tccli cls CreateTopic` |

## 验证

```bash
# 查看审计开关状态（实测 ClusterAuditEnabled=true）
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterAuditEnabled"
# expected: true
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 审计开关 | `DescribeClusterStatus` → `ClusterAuditEnabled` | `true` |
| 日志投递 | `tccli cls SearchLog --TopicId <ID>`（CLS 侧） | 有审计日志记录 |
| 操作可追溯 | 在 CLS 检索某 RequestId | 命中对应操作记录 |

> 审计日志写入有秒级延迟。开启后做一次操作（如 `DescribeClusters`），再到 CLS 检索确认。

## 回滚

```bash
# 关闭审计
tccli tke DisableClusterAudit --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 验证已关
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterAuditEnabled"
# expected: false
```

> 关闭后 CLS 中的历史日志仍保留，按 CLS 保留期自动过期。停止 CLS 计费需删除日志集。

## 事件持久化与加密

> 事件持久化（K8s Event 落 CLS）与集群数据加密。属安全加固，与审计日志互补。

### 事件持久化

```bash
# 开启事件持久化 (K8s Event 投递到 CLS, 需 LogsetId/TopicId/TopicRegion)
tccli tke EnableEventPersistence --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --LogsetId "<LOGSET_ID>" --TopicId "<TOPIC_ID>" --TopicRegion "<REGION>"
# expected: exit 0

# 关闭事件持久化 (DeleteLogSetAndTopic=true 同步删 CLS 日志集/主题)
tccli tke DisableEventPersistence --ClusterId "<CLUSTER_ID>" --region <REGION> --DeleteLogSetAndTopic false
# expected: exit 0
```

> `EnableEventPersistence` 需 CLS 日志集/主题（见 [前置：创建 CLS 日志集](#前置创建-cls-日志集与主题)）。`TopicRegion` 是 CLS 主题地域。关闭时 `DeleteLogSetAndTopic=true` 会删 CLS 资源停止计费。

### 数据加密

```bash
# 查询加密状态
tccli tke DescribeEncryptionStatus --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Status=Closed (未加密) / Enabled (已加密)
```
```json
{"Status": "Closed", "ErrorMsg": "", "RequestId": "..."}
```

```bash
# 关闭加密保护 (开启加密前的前置, 或完全关闭加密)
tccli tke DisableEncryptionProtection --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0
```

> `Status` 状态机：`Closed`（未加密）→ `enabling` → `Enabled`。`DisableEncryptionProtection` 关闭加密保护（非关闭加密本身），允许对加密集群做特殊操作。

```bash
# 开启加密保护（KMSConfiguration 传 KMS 加密配置）
tccli tke EnableEncryptionProtection --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --KMSConfiguration '{"KmsRegion":"<KMS_REGION>","KmsKeyId":"<KMS_KEY_ID>"}'
# expected: exit 0, Status 进入 enabling → Enabled
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<KMS_REGION>` | KMS 密钥所在地域 | `tccli kms ListKey --region <REGION>` |
| `<KMS_KEY_ID>` | KMS 主密钥 ID | `tccli kms ListKey` → `Keys[].KeyId` |

> `EnableEncryptionProtection` 的 `KmsConfiguration` 是嵌套对象（含 `KmsRegion`/`KmsKeyId`），开启后 etcd 数据用 KMS 密钥加密。开启是异步操作，用 `DescribeEncryptionStatus` 轮询 `Status` 到 `Enabled`。开启前需先在 KMS 创建密钥并授权 TKE 使用。

## 调度策略

> 集群调度器插件配置（`SchedulerPolicy`），控制 Pod 调度行为。D34 归入 Security（策略管控域）。

```bash
# 查询调度策略 (含调度器插件配置)
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, SchedulerPolicyConfig[] 含 SchedulerName/PluginConfigs
```
```json
{
    "Policy": "",
    "SchedulerPolicyConfig": [
        {"SchedulerName": "default-scheduler", "PluginConfigs": [{"Name": "NodeResourcesFit"}]}
    ]
}
```

```bash
# 修改调度策略
tccli tke ModifyClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> --Policy "<POLICY_JSON>"
# expected: exit 0
```

> `SchedulerPolicyConfig` 含调度器名（如 `default-scheduler`）与插件配置（如 `NodeResourcesFit` 资源适配）。修改 `Policy` 是调度器配置 JSON。

## 开放策略（OPA/Gatekeeper）

> 开放策略（OpenPolicy）基于 OPA/Gatekeeper 强制集群安全/合规规则（如禁止删带节点的集群）。D34 归入 Security（策略管控域）。

```bash
# 查询开放策略列表 (按 Category 分类)
tccli tke DescribeOpenPolicyList --ClusterId "<CLUSTER_ID>" --Category "<CATEGORY>" --region <REGION>
# expected: exit 0, OpenPolicyInfoList[] 含 PolicyName/EnforcementAction/EnabledStatus
```
```json
{
    "OpenPolicyInfoList": [
        {"PolicyCategory": "cluster", "PolicyName": "存在节点的集群不允许删除",
         "EnforcementAction": "deny", "EnabledStatus": "open", "Name": "example-policy-rule"}
    ]
}
```

```bash
# 修改开放策略 (OpenPolicyInfoList[] 批量, 含 EnforcementAction/Name/EnabledStatus)
tccli tke ModifyOpenPolicyList --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --OpenPolicyInfoList '[{"Name":"<POLICY_NAME>","EnforcementAction":"dryrun","EnabledStatus":"open"}]'
# expected: exit 0
```

> `EnforcementAction` 状态：`deny`（强制拒绝违规操作）/ `dryrun`（仅告警不拒绝）。`EnabledStatus`：`open`/`close`。`Category` 如 `cluster`（集群级）/ `namespace`（命名空间级）。开放策略与 [巡检](../observability/logging.md#集群巡检与日志配置) 互补——策略强制合规，巡检发现隐患。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` (LogsetId/TopicId) | `tccli cls DescribeLogsets` 核对 | CLS 日志集/主题不存在 | 先在 CLS 创建 |
| `InvalidParameterValue.TopicRegion` | 核对地域 | TopicRegion 与日志集地域不一致 | 用 CLS 日志集所在地域 |
| `FailedOperation` | `DescribeClusterStatus` 看状态 | 集群非 Running | 等集群 Running 后重试 |
| `UnauthorizedOperation.CamNoAuth` | 查 CAM 策略 | 无 `tke:EnableClusterAudit` 权限 | 授予权限 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ClusterAuditEnabled=true` 但 CLS 无日志 | `tccli cls SearchLog` | CLS 主题索引未开启或权限不对 | 在 CLS 开启主题索引 |
| 审计日志缺部分操作 | CLS 检索时间范围 | 检索范围太窄或延迟 | 扩大时间范围，等延迟 |

## 下一步

- [认证配置](auth.md) — 审计记录的认证操作
- [日志采集](../observability/logging.md) — 业务日志（非审计）采集
- [查询集群](../clusters/query.md) — 查审计开关
- [故障排查](../troubleshooting.md) — 审计缺失诊断

## 控制台替代方案

[容器服务控制台 - 集群审计](https://console.cloud.tencent.com/tke2/cluster)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `EnableClusterAudit` | 主操作 | 2018-05-25 | 开启审计日志（需 CLS 日志集/主题） |
| `EnableEventPersistence` | 主操作 | 2018-05-25 | 开启 K8s Event 落 CLS |
| `EnableEncryptionProtection` | 主操作 | 2018-05-25 | 开启加密保护 |
| `ModifyOpenPolicyList` | 主操作 | 2018-05-25 | 修改 OPA/Gatekeeper 策略 |
| `ModifyClusterSchedulerPolicy` | 主操作 | 2018-05-25 | 修改调度器插件配置 |
| `DisableClusterAudit` | 清理 | 2018-05-25 | 关闭审计（CLS 历史保留） |
| `DisableEventPersistence` | 清理 | 2018-05-25 | 关闭事件持久化（可同步删 CLS） |
| `DisableEncryptionProtection` | 清理 | 2018-05-25 | 关闭加密保护 |
| `DescribeOpenPolicyList` | 验证 | 2018-05-25 | 开放策略列表（按 Category） |
| `DescribeClusterSchedulerPolicy` | 验证 | 2018-05-25 | 调度策略与插件配置 |
| `DescribeEncryptionStatus` | 验证 | 2018-05-25 | 加密状态（Closed/enabling/Enabled） |
| `DescribeClusters` | 验证 | 2018-05-25 | 确认集群 ID |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 审计开关与集群状态 |
| `cls:CreateLogset` | 跨产品 | cls | 创建 CLS 日志集（前置） |
| `cls:CreateTopic` | 跨产品 | cls | 创建 CLS 日志主题（前置） |
| `cls:DescribeLogsets` | 跨产品 | cls | 核对日志集/主题 |
| `cls:SearchLog` | 跨产品 | cls | 检索审计日志 |
