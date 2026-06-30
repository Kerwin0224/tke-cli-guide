# 恢复管理（tccli）

> 对照官方：[恢复管理](https://cloud.tencent.com/document/product/457/90493) · page_id `90493`

## 概述

备份中心通过 Velero Restore CRD 从已有备份中恢复集群资源到目标 TKE 集群。支持全量恢复和选择性恢复（按命名空间、资源类型维度过滤）。

> **注意**：本页面操作依赖 kubectl 访问集群。需确保 Velero（tke-backup 组件）在集群内安装，kubectl 端点可达。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 kubectl 集群连通性
kubectl cluster-info
# expected: Kubernetes control plane 可达
```

### 资源检查

```bash
# 4. 验证备份仓库可用
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: State: "Available"

# 5. 验证已有备份（至少一个 Completed）
kubectl get backups -n velero
# expected: 至少一个备份 STATUS 为 Completed

# 6. 验证 tke-backup 组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName tke-backup
# expected: Status: "Enabled"
```

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

预期输出（DescribeAddon）：

```json
{
    "Addons": [
        {
            "AddonName": "tke-backup",
            "AddonVersion": "1.1.0",
            "Status": "Enabled"
        }
    ],
    "RequestId": "..."
}
```

## 关键字段说明

以下说明 Velero Restore CRD 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.backupName` | String | 是 | 要恢复的备份名称。需是已存在的 Backup 对象 | 备份不存在 → Phase: Failed |
| `spec.includedNamespaces` | Array | 否 | 指定要恢复的命名空间。省略则恢复备份中所有命名空间 | 范围过大 → 恢复耗时长、可能冲突 |
| `spec.includedResources` | Array | 否 | 指定要恢复的资源类型（deployments、services、configmaps 等） | 资源类型名拼写错误 → 该类型被跳过 |
| `spec.namespaceMapping` | Object | 否 | 恢复时的命名空间映射（old → new），如 `{"default": "default-restored"}` | 映射目标不存在 → Restore 报 warning |
| `spec.existingResourcePolicy` | String | 否 | `none`（跳过已存在资源，默认）或 `update`（覆盖已存在资源） | 选 `update` 可能覆盖正在运行的健康资源 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看可用备份 | `kubectl get backups -n velero` | 是 |
| 创建恢复任务 | `kubectl apply -f restore.yaml` | 否 |
| 查看恢复状态 | `kubectl get restores -n velero` | 是 |
| 查看恢复详情 | `kubectl describe restore RESTORE_NAME -n velero` | 是 |
| 删除恢复记录 | `kubectl delete restore RESTORE_NAME -n velero` | 否 |

## 操作步骤

### 1. 查看可用备份

```bash
kubectl get backups -n velero
# expected: 至少一个备份 STATUS 为 Completed
```

预期输出：

```text
NAME          STATUS      AGE
backup-full   Completed   1h
```

### 2. 创建恢复任务

#### 选择依据

- **全量恢复 vs 选择性恢复**：生产环境推荐选择性恢复（指定 namespace + resource type），避免覆盖正常运行的服务。
- **existingResourcePolicy**：目标集群已有同名资源时选择 `none`（跳过），避免意外覆盖。如需更新则选 `update`。
- **namespaceMapping**：可将备份中的命名空间恢复到不同名称，用于蓝绿恢复验证。

**全量恢复** (`restore-full.yaml`)：

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-full
  namespace: velero
spec:
  backupName: backup-full
```

**选择性恢复——指定命名空间** (`restore-ns.yaml`)：

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-default
  namespace: velero
spec:
  backupName: backup-full
  includedNamespaces:
  - default
  namespaceMapping:
    default: default-restored
```

**选择性恢复——指定资源类型** (`restore-resources.yaml`)：

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-deployments
  namespace: velero
spec:
  backupName: backup-full
  includeClusterResources: false
  includedResources:
  - deployments
  - services
  - configmaps
  existingResourcePolicy: none
```

```bash
kubectl apply -f restore-full.yaml
# expected: restore.velero.io/restore-full created
```

预期输出：

```text
restore.velero.io/restore-full created
```

### 3. 监控恢复进度

```bash
kubectl get restore restore-full -n velero -w
# expected: STATUS 由 InProgress 变为 Completed
```

预期输出：

```text
NAME           BACKUP        STATUS      AGE
restore-full   backup-full   Completed   2m
```

### 4. 验证恢复结果

```bash
# 检查恢复出的资源
kubectl get all -n default
# expected: 被恢复的资源已就绪

# 按资源类型检查
kubectl get deploy,svc,cm -n default
# expected: 对应资源类型列出目标对象
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证备份仓库可用
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: State: "Available"
```

```json
{
  "BackupStorageLocationSet": "<BackupStorageLocationSet>",
  "Name": "<Name>",
  "StorageRegion": "<StorageRegion>",
  "Provider": "<Provider>",
  "Bucket": "<Bucket>",
  "Path": "<Path>"
}
```

### 数据面

```bash
# 验证恢复任务完成
kubectl get restore restore-full -n velero \
    -o jsonpath='{.status.phase}'
# expected: Completed

# 查看恢复详情（含警告信息）
kubectl describe restore restore-full -n velero
# expected: Phase: Completed，warnings 数为 0
```

```text
NAME  STATUS  AGE
...
```

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。

## 清理

> **注意**：删除 Restore CRD 不会删除已恢复出的资源。删除 Restore 仅删除恢复记录。如需清理恢复出的资源，需手动删除。

### 数据面

```bash
# 1. 删除恢复记录（不会删除已恢复出的资源）
kubectl delete restore restore-full -n velero
# expected: restore.velero.io "restore-full" deleted

# 2. 如需删除恢复出的资源，手动清理对应命名空间
kubectl delete namespace default-restored
# expected: namespace "default-restored" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f restore.yaml` 返回 `no matches for kind "Restore"` | `kubectl api-resources \| grep velero` | tke-backup 组件未安装 | `tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName tke-backup --AddonVersion VERSION` |
| kubectl 不可达 | `kubectl cluster-info` | 集群公网端点 CAM 策略拒绝(strategyId:240463971) | 联系管理员放通 CAM 策略（内网需VPN/IOA），或通过控制台 Web Shell 操作 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 恢复冲突（AlreadyExists） | `kubectl describe restore RESTORE_NAME -n velero` 查看 warnings | 目标命名空间中已存在同名资源 | 设置 `spec.existingResourcePolicy: update` 覆盖，或先 `kubectl delete` 清理目标资源 |
| 恢复后 Service 不可用 | `kubectl describe svc SERVICE_NAME -n NAMESPACE` 检查 endpoints | 目标集群网络模式与备份时不一致 | 确认集群 CNI 模式相同，或修改 Service 配置适配当前网络 |
| 恢复长时间处于 InProgress | `kubectl describe restore RESTORE_NAME -n velero` 查看日志 | 资源量大或 API Server 限流 | 缩小恢复范围（按命名空间/资源类型过滤），分批恢复 |

## 下一步

- [备份管理](../备份管理/tccli%20操作.md) — 创建集群备份
- [备份仓库](../备份仓库/tccli%20操作.md) — 管理备份存储位置
- [备份恢复实践](../备份恢复实践/tccli%20操作.md) — 端到端备份恢复示例

## 控制台替代

控制台：集群 → 备份中心 → 备份管理 → 选择备份 → 恢复。
