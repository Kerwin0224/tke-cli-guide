---
doc_type: How-to
subtype: 6A
fused: true
---
# 实例同步（跨地域复制）

> 配置 TCR 企业版实例间的镜像跨地域同步，将主实例的镜像自动复制到从实例。
> 控制台: [容器镜像服务 - 实例同步](https://console.cloud.tencent.com/tcr/replication)

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
| `DestinationRegionId` | ManageReplication | 是 | 从实例地域数字 ID（如 `4`=ap-shanghai） |
| `Rule.Name` | ManageReplication | 是 | 规则名 |
| `Rule.DestNamespace` | ManageReplication | 是 | 目标命名空间 |
| `RegistryId` | CreateReplicationInstance | 是 | 主实例 ID |
| `ReplicationRegionId` | CreateReplicationInstance | 是 | 从实例地域数字 ID |
| `SyncTag` | CreateReplicationInstance | 否 | 是否同步标签 |

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

```bash
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

> 上图为空结果示例（未创建从实例时）。

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
# expected: exit 0，返回 TaskDetail+Status（无任务时为空；从实例创建进度 Creating→Success/Failed）
```

> `DescribeReplicationPolicies` 用 `Page`/`PageSize`（从 1 开始的页码），与同域 `DescribeReplicationInstances` 的 `Offset`/`Limit` 不同——切换接口必须改分页参数。`DescribeReplicationInstanceCreateTasks` 用 `ReplicationRegistryId`（从实例 ID，步骤 1 返回）+ `ReplicationRegionId`（从实例地域数字 ID）查创建任务，区别于 `DescribeReplicationInstanceSyncStatus` 查的是同步状态。

## 下一步

- 创建 standard 实例（同步前提）：[创建实例](../instances/create.md)
- 镜像推送（同步的源头）：[推送与拉取镜像](../images/push-pull.md)
- 仓库与命名空间管理：[仓库管理](../repositories/manage.md)
