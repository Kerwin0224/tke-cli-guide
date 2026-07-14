---
doc_type: How-to
subtype: 6B
fused: false
---
# 开启集群审计

> 控制台: [容器服务控制台 - 集群审计](https://console.cloud.tencent.com/tke2/cluster)
> 开启/关闭集群审计日志。审计记录所有 API Server 操作到 CLS（日志服务），用于合规与事故追溯。配置型操作。

## 触发条件

- `DescribeClusterStatus` → `ClusterAuditEnabled=false`，生产集群需开启审计满足合规要求
- 合规审计要求追溯"谁在什么时候对什么资源做了什么操作"，当前集群无 API 操作记录可查
- `ClusterAuditEnabled=true` 但 CLS 检索无日志，需排查投递链路 — 看 [故障恢复]段


## 概述

审计日志记录"谁在什么时候对什么资源做了什么操作"。开启后，集群 API Server 的所有操作（kubectl <!-- tccli管控审计开关/日志集配置，kubectl操作作为审计对象被记录，非tccli边界 --> /TCCLI/控制台）写入 CLS 日志集。

| 状态 | 含义 | 查询 |
|:-----|:-----|:-----|
| 开启 | API 操作记录到 CLS | `DescribeClusterStatus` → `ClusterAuditEnabled=true` |
| 关闭 | 不记录 | `ClusterAuditEnabled=false` |

> 审计日志存储在 CLS 服务（跨产品），TKE 无 `DescribeClusterAuditLog` 接口——查日志须到 CLS 控制台或用 `tccli cls` 检索。

> 官方文档：[身份验证和授权概述](https://cloud.tencent.com/document/product/457/11542) · [服务授权相关角色权限说明](https://cloud.tencent.com/document/product/457/43416) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
> 配额：CLS 日志集/主题存储限制（按 CLS 计费模式），日志集数量默认 20。[配额限制](https://cloud.tencent.com/document/product/457/9087)
> ⚠️ **高危操作**：关闭审计日志致安全事件不可追溯；审计日志投递中断期间的所有操作无记录。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 决策依据

#### 为什么开启审计

- **开启 vs 关闭**: 开启后所有 API 操作可追溯（合规、事故定位）；关闭省 CLS 存储费
- **默认推荐**: 生产环境开启（合规追溯需要）；测试集群可关
- **可关闭**：是，`DisableClusterAudit`。但已存的审计日志保留在 CLS

## 配置项

> 完整入参以 `tccli tke EnableClusterAudit help --detail` 为准。需先在 CLS 创建日志集与主题。

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
# 查看审计开关状态（ClusterAuditEnabled=true）
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterAuditEnabled"
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
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterAuditEnabled"
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
# expected: exit 0, Status=Closed (未加密) / Opened (已加密)
```
```json
{"Status": "Closed", "ErrorMsg": "", "RequestId": "..."}
```

```bash
# 关闭加密保护 (开启加密前的前置, 或完全关闭加密)
tccli tke DisableEncryptionProtection --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0
```

> `Status` 状态机：`Closed`（未加密）→ `Opening`（开启中）→ `Opened`（已开启）。`DisableEncryptionProtection` 关闭加密保护（非关闭加密本身），允许对加密集群做特殊操作。

```bash
# 开启加密保护（KMSConfiguration 传 KMS 加密配置）
tccli tke EnableEncryptionProtection --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --KMSConfiguration '{"KmsRegion":"<KMS_REGION>","KmsKeyId":"<KMS_KEY_ID>"}'
# expected: exit 0, Status 进入 Opening → Opened; 集群不存在报 ResourceNotFound
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<KMS_REGION>` | KMS 密钥所在地域 | `tccli kms ListKeys --region <REGION>` |
| `<KMS_KEY_ID>` | KMS 主密钥 ID | `tccli kms ListKeys` → `Keys[].KeyId` |

> `EnableEncryptionProtection` 的 `KmsConfiguration` 是嵌套对象（含 `KmsRegion`/`KmsKeyId`），开启后 etcd 数据用 KMS 密钥加密。开启是异步操作，用 `DescribeEncryptionStatus` 轮询 `Status` 到 `Opened`。开启前需先在 KMS 创建密钥并授权 TKE 使用。

## 开放策略（OPA/Gatekeeper）

> 开放策略（OpenPolicy）基于 OPA/Gatekeeper 强制集群安全/合规规则（如禁止删带节点的集群）。属准入控制，是集群加固的一环。

```bash
# 查询开放策略列表（入参 Category ≠ 响应 PolicyCategory）
tccli tke DescribeOpenPolicyList --ClusterId "<CLUSTER_ID>" --Category "baseline" --region <REGION>
# expected: exit 0；OpenPolicyInfoList[] + GatekeeperStatus；省略 Category 返回全量
```
```json
{
    "OpenPolicyInfoList": [
        {"PolicyCategory": "cluster", "PolicyName": "NodeExistBlockDeleteCluster",
         "EnforcementAction": "deny", "EnabledStatus": "open", "Name": "block-cluster-deletion-rule",
         "Kind": "blockclusterdeletion"}
    ],
    "GatekeeperStatus": 1
}
```

```bash
# 修改开放策略 (OpenPolicyInfoList[] 批量, 含 EnforcementAction/Name/EnabledStatus)
tccli tke ModifyOpenPolicyList --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --OpenPolicyInfoList '[{"Name":"<POLICY_NAME>","EnforcementAction":"dryrun","EnabledStatus":"open"}]'
# expected: exit 0
```

> `EnforcementAction`：`deny` / `dryrun`。`EnabledStatus`：`open` / `close`。入参 `--Category` 与响应 `PolicyCategory` **不是同一枚举**（见下表）。开放策略与 [巡检](../observability/logging.md#集群巡检与日志配置) 互补。

入参 `--Category`（`help --detail`：基线 / 优选 / 可选）：

| Category（入参） | 含义 | 返回条数（空托管集群） |
|:-----------------|:-----|:---------------------------|
| `baseline` | 基线 | 少（约 1 条） |
| `priority` | 优选 | 中（约十余条） |
| `optional` | 可选 | 多（约数十条） |

> 非法值（如 `soft`）不报错，行为未文档化——**只用上表三值**。勿把响应里的 `PolicyCategory`（cluster/node/…）填进 `--Category`。

响应 `PolicyCategory` 枚举：

| PolicyCategory | 分类 |
|:---------------|:-----|
| `cluster` | 集群策略 |
| `node` | 节点策略 |
| `namespace` | 命名空间策略 |
| `configuration` | 配置相关策略 |
| `compute` | 计算资源策略 |
| `storage` | 存储资源策略 |
| `network` | 网络资源策略 |

`Kind` 策略模板枚举。用户按需搜对应 Kind 启用：

| Kind | 作用 | 典型场景 |
|:-----|:-----|:---------|
| `blocknamespacedeletion` | 存在 pod 的命名空间不允许删除 | 防误删带工作负载的命名空间 |
| `blockcrddeletion` | 存在 CR 的 CRD 不允许删除 | 防误删有自定义资源的 CRD |
| `blockmountablevolumetype` | 禁止挂载指定的 volume 类型 | 限制存储类型合规 |
| `disallowalwayspullimage` | 禁止镜像拉取策略用 Always | 强制 imagePullPolicy 非 Always |
| `tkeallowedrepos` | 容器镜像来源限制 | 限制镜像仓库白名单 |
| `blockunknowndaemonset` | 禁止未知的 DaemonSet 部署 | 防 DaemonSet 蔓延 |
| `blockpvdeletion` | PV 绑定状态不允许删除 | 防 PV 误删丢数据 |
| `corednsprotect` | CoreDNS 组件删除保护 | 防 CoreDNS 被误删致 DNS 瘫痪 |
| `blockschedulablenodedelete` | 非封锁状态 Node 不允许删除 | 防删运行中节点 |
| `resourcesdeletionprotection` | 资源删除保护 | 通用删除保护 |
| `tkeenirequest` | 弹性网卡资源配置限制 | ENI 资源合规 |
| `blockworkloadcrossversionupgrade` | 工作负载镜像版本升级策略管控 | 管控跨版本升级 |
| `blockserviceaccountgranthighprivilegepermission` | ServiceAccount 高权限制约 | 防 SA 过权 |

> 完整 Kind 列表以 `tccli tke DescribeOpenPolicyList --ClusterId "<ID>" --Category "baseline|priority|optional"` 返回为准。启用时 `ModifyOpenPolicyList --OpenPolicyInfoList '[{"Name":"<规则 Name>","EnforcementAction":"deny","EnabledStatus":"open"}]'`（`Name` 用返回的规则名，如 `block-cluster-deletion-rule`，非 Kind）。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` (LogsetId/TopicId) | `tccli cls DescribeLogsets` 核对 | CLS 日志集/主题不存在 | 先在 CLS 创建 |
| `InvalidParameterValue.TopicRegion` | 核对地域 | TopicRegion 与日志集地域不一致 | 用 CLS 日志集所在地域 |
| `FailedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running 后重试 |
| `UnauthorizedOperation.CamNoAuth` | 查 CAM 策略 | 无 `tke:EnableClusterAudit` 权限 | 授予权限 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ClusterAuditEnabled=true` 但 CLS 无日志 | `tccli cls SearchLog` | CLS 主题索引未开启或权限不对 | 在 CLS 开启主题索引 |
| 审计日志缺部分操作 | CLS 检索时间范围 | 检索范围太窄或延迟 | 扩大时间范围，等延迟 |

## 收尾确认

```bash
# 跨步骤汇总三项合一：审计开关已开 + CLS 日志集/主题存在 + 投递可查
# 1. 审计开关（TKE 侧）
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterAuditEnabled"
# expected: true

# 2. CLS 日志集/主题存在（跨产品 cls）
tccli cls DescribeLogsets --region <REGION> --LogsetId "<LOGSET_ID>"
# expected: 返回日志集，含 TopicId

# 3. 业务可用性端到端：做一次操作后 CLS 能检索到该操作
tccli tke DescribeClusters --region ap-guangzhou --filter "TotalCount"
tccli cls SearchLog --region <REGION> --TopicId "<TOPIC_ID>" --Content '"DescribeClusters"'
# expected: 命中含 DescribeClusters 的审计记录 → 审计日志闭环完成
```

> 审计开关 true + CLS 日志集/主题存在 + 操作可检索 = 端到端闭环。Verify 段查开关状态与日志投递维度，此处跨步骤汇总 TKE 侧开关 + CLS 侧资源存在 + 真实操作可检索三项，确认投递链路完整可用。

---

## 下一步

- [认证配置](auth.md) — 审计记录的认证操作
- [日志采集](../observability/logging.md) — 业务日志（非审计）采集
- [查询集群](../clusters/query.md) — 查审计开关
- [故障排查](../troubleshooting.md) — 审计缺失诊断
