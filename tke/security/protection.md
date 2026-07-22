---
doc_type: How-to
subtype: 6B
fused: false
---
# 集群保护策略

> 控制台: [容器服务控制台 - 集群安全](https://console.cloud.tencent.com/tke2/cluster)
> 加密 etcd、删除保护、OPA 准入策略、事件持久化。配置型操作（改变安全行为，不创建资源）。

## 触发条件

- 集群含敏感数据（Secret/ConfigMap），需开启 etcd 加密 — `DescribeEncryptionStatus` 返回 `Status: Closed`
- 防止误删集群，需开启删除保护 — 新建集群默认 `ClusterDeletionProtection: false`（须 `EnableClusterDeletionProtection`）
- 多租户/合规场景需收紧 OPA — `DescribeOpenPolicyList --Category baseline|priority|optional` 已有策略列表，按需 `ModifyOpenPolicyList` 改 EnforcementAction/EnabledStatus（非「返回空」才配置）
- 需持久化 K8s 事件用于审计追溯 — `DescribeLogSwitches` 返回事件开关未开

## 概述

集群保护策略覆盖四个安全维度：

| 维度 | 作用 | 关键 Action |
|:-----|:-----|:-----------|
| etcd 加密 | KMS 加密 etcd 数据，防数据泄露 | `EnableEncryptionProtection` / `DisableEncryptionProtection` / `DescribeEncryptionStatus` |
| 删除保护 | 阻止控制台/API 误删集群 | `EnableClusterDeletionProtection` / `DisableClusterDeletionProtection` |
| OPA 准入 | Gatekeeper 准入策略控制部署合规 | `DescribeOpenPolicyList` / `ModifyOpenPolicyList` |
| 事件持久化 | K8s 事件存 CLS 用于审计 | `EnableEventPersistence` / `DisableEventPersistence` |
| 集群密钥 | 查询集群证书等敏感信息 | `DescribeClusterSecurity` |

> 官方文档：[策略管理](https://cloud.tencent.com/document/product/457/103179) · [使用 KMS 进行 ETCD 加密](https://cloud.tencent.com/document/product/457/45594) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
> 配额：集群配置限制参见 [配额限制](https://cloud.tencent.com/document/product/457/9087)。
> ⚠️ **高危操作**：关闭删除保护后误删集群不可逆；加密密钥丢失致 ETCD 数据不可恢复。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 配置项

> 入参字段名与必填以 `tccli tke <Action> help --detail` 为准。

### etcd 加密

| 字段 | 类型 | 必填 | 说明 |
|:------|------|:--------:|------|
| ClusterId | string | 是 | 集群 ID |
| KMSConfiguration | object | 是 | KMS 加密配置（`KeyId`/`KmsRegion`） |

> etcd 加密需先在 KMS 服务创建密钥（非 TKE 范畴）。`KMSConfiguration.KeyId` 不指定时采用默认生成密钥（TKE-KMS）。

### 删除保护 / OPA 准入 / 事件持久化

- 删除保护 Action 仅需 `ClusterId`。
- `DescribeOpenPolicyList` 需 `ClusterId`，`Category` 可选；`ModifyOpenPolicyList` 要实际修改策略时必须提供 `OpenPolicyInfoList`，先从 Describe 返回中取得规则 `Name` 与 `Kind`，再回传要修改的 `EnforcementAction`（每项 `Name`/`Kind`/`EnforcementAction` 均为 Required）。`help --detail` 将该列表标为 Optional，但不传便没有待修改策略，不能据此概括为“OPA 仅需 ClusterId”。
- 事件持久化除 `ClusterId` 外，还需 `LogsetId`/`TopicId`/`TopicRegion`（CLS 日志集/主题）。

## 应用

### 开启 etcd 加密（创建后单独开）

```bash
tccli tke EnableEncryptionProtection --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --KMSConfiguration '{"KeyId":"<KMS_KEY_ID>","KmsRegion":"ap-guangzhou"}'
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 如何获取 |
|:------|:-----|:--------|
| `<CLUSTER_ID>` | 集群 ID | `tccli tke DescribeClusters --filter "Clusters[].ClusterId"` |
| `<KMS_KEY_ID>` | KMS 密钥 ID | `tccli kms ListKeys --region ap-guangzhou` |

> etcd 加密创建后单独开启，非 `CreateCluster` 入参。集群中使用注册节点的暂不支持 KMS 加密。

### 开启删除保护

```bash
tccli tke EnableClusterDeletionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 返回 RequestId；集群须已带 CAM 要求的 resource_tag（如 billing）；若策略另要求 request_tag 而本接口无 Tags 入参 → 可能 AuthFailure（见故障表）
```

> 删除保护开启后，`DeleteCluster` 会被拒绝。删除集群前须先 `DisableClusterDeletionProtection`。

### 查询 OPA 准入策略

```bash
# 入参 Category 仅 baseline|priority|optional（≠ 响应 PolicyCategory）；省略则返回全量
tccli tke DescribeOpenPolicyList --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --Category "baseline"
# expected: OpenPolicyInfoList[]（PolicyCategory/EnforcementAction/EnabledStatus=open|close）+ GatekeeperStatus
```

入参枚举详 [审计日志](audit.md)

### 修改 OPA 准入策略

```bash
# 先查询并取得真实规则 Name 与 Kind
tccli tke DescribeOpenPolicyList --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Category "baseline" \
  --filter "OpenPolicyInfoList[].{name:Name,kind:Kind,action:EnforcementAction}"

# 将选定规则的 EnforcementAction 修改后回传（OpenPolicyInfoList 每项 Kind 与 Name 均为 Required）
tccli tke ModifyOpenPolicyList --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Category "baseline" \
  --OpenPolicyInfoList '[{"Name":"<POLICY_NAME>","Kind":"<POLICY_KIND>","EnforcementAction":"dryrun"}]'
# expected: exit 0, 返回 RequestId；再 DescribeOpenPolicyList 回读该 Name 的 EnforcementAction
```

> `tccli tke ModifyOpenPolicyList help --detail` 说明目前仅支持修改 `EnforcementAction`。`OpenPolicyInfoList` 在入参中虽标为 Optional，但执行实际修改时必须传入待修改规则；每项的 `Name`/`Kind`/`EnforcementAction` 均为 Required——`Kind` 从 `DescribeOpenPolicyList` 返回的同名字段取（如 `blocknamespacedeletion`），不要只传 Name。

### 开启事件持久化

```bash
tccli tke EnableEventPersistence --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogsetId "<CLS_LOGSET_ID>" --TopicId "<CLS_TOPIC_ID>" --TopicRegion ap-guangzhou
# expected: exit 0, 返回 RequestId
```

### 查询集群密钥信息

```bash
tccli tke DescribeClusterSecurity --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: { "UserName": "...", "Password": "...", "CertificationAuthority": "...", "Kubeconfig": "...", "RequestId": "..." }
# ⚠️ 返回含敏感凭证（用户名/密码/证书），勿明文记录
```

## 验证

```bash
# 加密状态
tccli tke DescribeEncryptionStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status=Opening(开启中)/Opened(已开启)

# 删除保护状态
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection"
# expected: true（已开启）

# 事件持久化开关
tccli tke DescribeLogSwitches --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: 含事件日志开关状态
```

## 回滚

> 配置型操作，回滚 = 关闭对应保护：

```bash
# 发起关闭 etcd 加密流程
tccli tke DisableEncryptionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 返回 RequestId；RequestId 仅表示请求已受理

# 轮询关闭状态机
tccli tke DescribeEncryptionStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Status" --output text
# expected: Closing 表示关闭中；Closed 表示已关闭，只有 Closed 才是完成终态

# 关闭删除保护（删集群前必须）
tccli tke DisableClusterDeletionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 返回 RequestId；此后 DeleteCluster 可正常执行

# 关闭事件持久化（可选删除 CLS 日志集）
tccli tke DisableEventPersistence --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --DeleteLogSetAndTopic false
# expected: exit 0, 返回 RequestId；DescribeLogSwitches 返回 Event.Enable=false
```

---

## 限制与故障恢复

### 固有限制

| 限制 | 影响 | 规避 |
|:-----|:-----|:-----|
| 注册节点集群不支持 KMS 加密 | 使用注册节点的集群无法开启 etcd 加密 | 不使用注册节点，或接受无加密 |
| 删除保护不阻止 CBS/EIP 等关联资源删除 | 集群删了但残留资源仍计费 | 删集群前手动清理关联资源，见 [删除集群](../clusters/delete.md) |
| OPA 策略修改后需几秒生效 | 立即部署可能未被策略拦截 | 等待 10 秒后重试 |

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.ClusterId` | `tccli tke DescribeClusters` | 集群 ID 不存在或地域错误 | 确认集群在该地域存在 |
| `UnauthorizedOperation` | 检查 CAM 策略 | 子账号无 `tke:EnableEncryptionProtection` 等权限 | 主账号授予对应 Action 权限 |
| `AuthFailure.UnauthorizedOperation`（`Enable`/`DisableClusterDeletionProtection`，消息含 `qcs:resource_tag` 和/或 `qcs:request_tag`） | `DescribeClusters` 看集群 Tags；读错误条件 | 无匹配 `resource_tag`（如缺 `billing`）会拒；部分 CAM 还要求 `request_tag`，而本接口**无 Tags 入参**时可能仍拒 | 先保证资源带 `billing`；若错误仍列 `request_tag` 且无法在请求带标签 → 控制台开关或调整 CAM（环境限制） |
| KMS 密钥不存在 | `tccli kms ListKeys` | `KMSConfiguration.KeyId` 指向不存在的密钥 | 用有效 KMS 密钥 ID |

## 收尾确认

> 汇总核对 + 删除保护终态：除各功能逐项验证外，还须确认三策略整体已生效且集群处于「删除受保护」安全终态（逐功能核对不能代替安全终态汇总）。

```bash
# 汇总核对：加密 + 删除保护 + 事件持久化 + OPA 准入（OPA 须单独核对）
tccli tke DescribeEncryptionStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Status"
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection"
tccli tke DescribeOpenPolicyList --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status=Opened, ClusterDeletionProtection=true, DescribeOpenPolicyList 返回已启用策略 → 保护策略整体生效且集群删除受保护
```

## 下一步

- [集群认证](auth.md) — kubeconfig/OIDC/RBAC 认证配置
- [审计日志](audit.md) — 集群操作审计
- [删除集群](../clusters/delete.md) — 删除前须关闭删除保护
- [创建集群](../clusters/create.md) — 创建时可配删除保护（`DeletionProtection: true`）
