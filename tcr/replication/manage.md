---
doc_type: How-to
subtype: 6A
fused: true
---
# 实例同步（跨地域复制）

> 配置 TCR 企业版实例间的镜像跨地域同步，将主实例的镜像自动复制到从实例。
> 控制台: [实例同步](https://console.cloud.tencent.com/tcr/sync) · [实例复制](https://console.cloud.tencent.com/tcr/replication)（运维中心「同步复制」下两个入口）。**基础版不支持**实例同步——控制台在 basic 实例上打开同步页会提示「基础版实例不支持，请前往实例管理调整实例规格」；API 返回 `UnsupportedOperation: only supports standard and premium instance`。
> 官方文档: [同实例多地域复制镜像](https://cloud.tencent.com/document/product/1141/52095) · [跨实例同步镜像](https://cloud.tencent.com/document/product/1141/41945)

## 触发条件

- `tccli tcr DescribeInstances --Registryids '["<ID>"]'` 返回 `RegistryType: basic`，跨地域容灾/就近拉取需求，basic 不支持同步（`ManageReplication` 报 `UnsupportedOperation: only supports standard and premium instance`）
- `DescribeReplicationInstances --RegistryId "<ID>"` 返回 `TotalCount: 0`，主实例无从实例，镜像无法跨地域复制
- 跨地域 CI/CD 拉取超时，需在目标地域建从实例就近拉取（`docker pull` 延迟高）

## 准备工作

- 已创建主实例 (src) + 目标地域有配额建从实例
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))



## 概述

实例同步（Replication）在主实例与从实例间建立同步规则，主实例推送的镜像按规则自动复制到从实例所在地域。典型场景：跨地域容灾、就近拉取加速。

> ⚠️ **实例规格限制**：实例同步**仅支持 standard / premium 实例**，basic 实例不支持。在 basic 实例上调用 `ManageReplication` 返回 `UnsupportedOperation: ManageReplication function only supports standard and premium instance`。使用前确认实例规格。

## 决策依据

### 主从实例选型

| 角色 | 要求 | 说明 |
|:-----|:-----|:-----|
| 主实例（Source） | standard 或 premium | 镜像来源，地域 A |
| 从实例（Destination） | standard 或 premium | 镜像目标，地域 B |

basic 实例不能作主也不能作从。需先 [创建实例](../instances/create.md) 升级到 standard。

### 同步规则（Rule）

`ManageReplication` 的 `Rule` 决定哪些镜像同步：

| 字段 | 含义 | 示例 |
|:-----|:-----|:-----|
| `Name` | 规则名 | `sync-prod` |
| `DestNamespace` | 目标命名空间 | `prod` |
| `Override` | 同名 tag 是否覆盖 | `false`（保留旧版） |
| `Filters[].Type` | 过滤类型 | `name`（按镜像名） |
| `Filters[].Value` | 过滤值 | `app-*` |
| `Deletion` | 是否同步删除 | `false`（主删从不删） |

### 跨账号同步

若主从实例属不同账号，需用 `PeerReplicationOption`（`PeerRegistryUin`/`PeerRegistryToken`/`EnablePeerReplication`）建立对等关系。同账号无需此参数。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `SourceRegistryId` | ManageReplication | 是 | 主实例 ID |
| `DestinationRegistryId` | ManageReplication | 是 | 从实例 ID |
| `DestinationRegionId` | ManageReplication | 否 | 从实例地域数字 ID（如 `4`=ap-shanghai）；可选，建议与从实例地域一致 |
| `Rule.Name` | ManageReplication | 是 | 规则名 |
| `Rule.DestNamespace` | ManageReplication | 是 | 目标命名空间 |
| `RegistryId` | CreateReplicationInstance | 是 | 主实例 ID |
| `ReplicationRegionId` | CreateReplicationInstance | 条件 | 从实例地域数字 ID；与 `ReplicationRegionName` **二选一或同传**（API 层二者均 Optional，实际须给地域） |
| `ReplicationRegionId` | DeleteReplicationInstance | 是 | 待删除从实例的地域数字 ID |
| `ReplicationRegionId` | DescribeReplicationInstanceCreateTasks | 是 | 与 `ReplicationRegistryId` 联合定位从实例创建任务 |
| `ReplicationRegionName` | CreateReplicationInstance | 条件 | 与 `ReplicationRegionId` 二选一时必传；从实例地域名（如 `ap-beijing`），也可与数字 ID 同传 |
| `SyncTag` | CreateReplicationInstance | 否 | 是否同步 TCR 云标签至 COS Bucket |

### TCR 地域数字 ID（ReplicationRegionId 取值）

`tccli tcr DescribeRegions`：

| 地域 | ID | 别名 |
|:-----|:--:|:----:|
| ap-guangzhou | 1 | gz |
| ap-shanghai | 4 | sh |
| ap-beijing | 8 | bj |
| ap-hongkong | 5 | hk |
| ap-chengdu | 16 | cd |
| ap-nanjing | 33 | nj |
| ap-tianjin | 36 | tsn |
| ap-zhongwei | 102 | zw |

## 操作步骤

### 步骤 1：创建从实例（目标地域）

```bash
tccli tcr CreateReplicationInstance --region <SOURCE_REGION> \
  --RegistryId <SOURCE_REGISTRY_ID> \
  --ReplicationRegionId <DEST_REGION_ID> \
  --ReplicationRegionName <DEST_REGION_NAME> \
  --SyncTag true
# expected: exit 0, 返回 ReplicationRegistryId
```

### 步骤 2：配置同步规则

同账号主从实例：只传 `Rule`。跨账号：另加 `PeerReplicationOption`（见上文「跨账号同步」）。

```bash
# 同账号：配置同步规则（无 PeerReplicationOption）
tccli tcr ManageReplication --region <SOURCE_REGION> \
  --SourceRegistryId <SOURCE_REGISTRY_ID> \
  --DestinationRegistryId <DEST_REGISTRY_ID> \
  --DestinationRegionId <DEST_REGION_ID> \
  --Rule '{"Name":"<RULE_NAME>","DestNamespace":"<DEST_NAMESPACE>","Override":false,"Filters":[{"Type":"name","Value":"<IMAGE_PATTERN>"}],"Deletion":false}'
# expected: exit 0
```

### 步骤 3：查询同步状态

```bash
tccli tcr DescribeReplicationInstanceSyncStatus --region <SOURCE_REGION> \
  --RegistryId <SOURCE_REGISTRY_ID> \
  --ReplicationRegistryId <REPLICATION_REGISTRY_ID> \
  --ReplicationRegionId <DEST_REGION_ID>
# expected: exit 0, 返回同步进度与日志
```

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 从实例已创建 | `DescribeReplicationInstances --RegistryId <SOURCE_REGISTRY_ID>` | `TotalCount >= 1` |
| 同步规则存在 | `DescribeReplicationPolicies --RegistryId <SOURCE_REGISTRY_ID>` | 返回规则 |
| 同步完成 | `DescribeReplicationInstanceSyncStatus` | 状态=完成 |
| 镜像可从从实例拉取 | 在从实例地域 `docker pull <DEST_ENDPOINT>/<DEST_NAMESPACE>/<IMAGE>` | 拉取成功 |

```bash
tccli tcr DescribeReplicationInstances --region <SOURCE_REGION> --RegistryId <SOURCE_REGISTRY_ID>
# expected: exit 0, TotalCount >= 1
```
```json
{
    "TotalCount": 0,
    "ReplicationRegistries": null,
    "RequestId": "..."
}
```

> 上例为空结果示例（未创建从实例时）。

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<SOURCE_REGISTRY_ID>` | 主实例 ID | standard/premium | `tccli tcr DescribeInstances --region <SOURCE_REGION>` |
| `<DEST_REGISTRY_ID>` | 从实例 ID | 同上 | 步骤 1 返回 |
| `<SOURCE_REGION>` | 主实例地域 | 如 `ap-guangzhou` | 自定义 |
| `<DEST_REGION_ID>` | 从实例地域数字 ID | 见上表 | `tccli tcr DescribeRegions` |
| `<DEST_REGION_NAME>` | 从实例地域名 | 如 `ap-shanghai` | `tccli tcr DescribeRegions` |
| `<RULE_NAME>` | 规则名 | 实例内唯一 | 自定义 |
| `<DEST_NAMESPACE>` | 目标命名空间 | 从实例已建 | `tccli tcr DescribeNamespaces` |
| `<IMAGE_PATTERN>` | 镜像过滤 | 通配符 | 自定义 |

## 清理

### 删除同步规则

```bash
tccli tcr DeleteReplicationRule --region <SOURCE_REGION> \
  --SourceRegistryId <SOURCE_REGISTRY_ID> --RuleName <RULE_NAME>
# expected: exit 0
```

### 删除从实例

```bash
tccli tcr DeleteReplicationInstance --region <SOURCE_REGION> \
  --RegistryId <SOURCE_REGISTRY_ID> \
  --ReplicationRegistryId <REPLICATION_REGISTRY_ID> \
  --ReplicationRegionId <DEST_REGION_ID>
# expected: exit 0
```

## 副作用

- **同步规则 `Override=true`** 会用主实例 tag 覆盖从实例同名 tag，从实例原有镜像版本会丢失。
- **`Deletion=true`** 主实例删镜像时从实例同步删除，可能误删从实例镜像。
- **删除从实例**会销毁目标地域所有同步的镜像，不可恢复。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `UnsupportedOperation: only supports standard and premium instance` | 实例规格是 basic | 升级实例到 standard/premium，或换符合条件的实例 |
| `ReplicationRegionId` 无效 | 用了地域名而非数字 ID | 用数字 ID（gz=1, sh=4, bj=8…，见上表） |
| 同步规则创建后无镜像同步 | `Filters` 不匹配任何镜像 | 调整 `Filters[].Value` 通配符，或推一个匹配的镜像触发 |
| 跨账号同步失败 | 未配 `PeerReplicationOption` | 同账号免配；跨账号需 `PeerRegistryUin`/`PeerRegistryToken` |
| `DescribeReplicationPolicies` 分页无效 | 该接口用 `Page/PageSize` 非 `Offset/Limit` | 改用 `--Page 1 --PageSize 20` |

> ⚠️ **分页字段不一致**：`DescribeReplicationInstances` 用 `Offset/Limit`，`DescribeReplicationPolicies` 用 `Page/PageSize`。同一功能域分页契约不同，切换接口时核对。

```bash
# 查询同步规则列表（RegistryId + Page/PageSize 分页，非 Offset/Limit）
tccli tcr DescribeReplicationPolicies --region <SOURCE_REGION> \
  --RegistryId <SOURCE_REGISTRY_ID> --Page 1 --PageSize 20
# expected: exit 0，返回 ReplicationPolicyInfoList[]+TotalCount

# 查询从实例创建任务状态（ReplicationRegistryId + ReplicationRegionId 定位）
tccli tcr DescribeReplicationInstanceCreateTasks --region <SOURCE_REGION> \
  --ReplicationRegistryId <REPLICATION_REGISTRY_ID> \
  --ReplicationRegionId <DEST_REGION_ID>
# expected: exit 0，返回 TaskDetail+Status（无任务时为空；整体 Status / TaskDetail.TaskStatus 示例为 SUCCESS，以实际响应为准）
```

> `DescribeReplicationPolicies` 用 `Page`/`PageSize`（从 1 开始的页码），与同域 `DescribeReplicationInstances` 的 `Offset`/`Limit` 不同——切换接口必须改分页参数。`DescribeReplicationInstanceCreateTasks` 用 `ReplicationRegistryId`（从实例 ID，步骤 1 返回）+ `ReplicationRegionId`（从实例地域数字 ID）查创建任务，区别于 `DescribeReplicationInstanceSyncStatus` 查的是同步状态。

## 收尾确认

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 汇总核对：从实例创建 + 同步规则就绪 + 同步状态完成
tccli tcr DescribeReplicationInstances --region <SOURCE_REGION> --RegistryId "<SOURCE_REGISTRY_ID>" \
  --filter "{total:TotalCount,regs:ReplicationRegistries[].{dest:ReplicationRegistryId,region:ReplicationRegionId}}"
# expected: total>=1, 含目标从实例（字段名 ReplicationRegistries 非 ReplicationInstances）

tccli tcr DescribeReplicationInstanceSyncStatus --region <SOURCE_REGION> \
  --RegistryId "<SOURCE_REGISTRY_ID>" \
  --ReplicationRegistryId "<REPLICATION_REGISTRY_ID>" \
  --ReplicationRegionId <DEST_REGION_ID>
# expected: 同步状态=完成（无 Pending/InProgress）

# 端到端：镜像可从从实例拉取
docker pull <DEST_REGISTRY_DOMAIN>/<DEST_NAMESPACE>/<IMAGE>:<TAG>
# expected: Pull complete（从从实例公网端点拉取，DEST_REGISTRY_DOMAIN 从从实例 DescribeInstances 取 PublicDomain）
```

> 从实例已建 + 同步状态完成 + 从实例 docker pull 成功 = 实例同步配置完成，主实例镜像已跨地域复制可达。

---

## 下一步

- 创建 standard 实例（同步前提）：[创建实例](../instances/create.md)
- 镜像推送（同步的源头）：[推送与拉取镜像](../images/push-pull.md)
- 仓库与命名空间管理：[仓库管理](../repositories/manage.md)
