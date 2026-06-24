# 备份管理（tccli）

> 对照官方：[备份管理](https://cloud.tencent.com/document/product/457/90492) · page_id `90492`

## 概述

备份中心通过 Velero Backup CRD 创建集群级备份（即时备份）和定时备份策略（Schedule）。备份内容涵盖 K8s 资源对象（Deployment、Service、ConfigMap 等）和持久卷数据。操作均在数据面（kubectl）完成，控制面（tccli）仅管理备份仓库。

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

# 3. 检查 kubectl 版本
kubectl version --client
# expected: Client Version 含版本号

# 4. 检查 kubectl 集群连通性
kubectl cluster-info
# expected: Kubernetes control plane 可达
```

### 资源检查

```bash
# 5. 验证备份仓库可用
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: State: "Available"

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

以下说明 Velero Backup 和 Schedule CRD 的主要字段。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.includedNamespaces` | Array | 否 | 要备份的命名空间列表。省略则备份所有命名空间 | 范围过大 → 备份数据量膨胀、耗时长 |
| `spec.storageLocation` | String | 否 | 指定备份仓库名称，默认 `default` | 仓库不存在 → Phase 报错 |
| `spec.ttl` | String | 否 | 备份保留时长，格式 `{h}h{m}m{s}s`。默认 720h（30 天） | 过短则恢复时备份可能已被自动清理 |
| `spec.schedule` | String | 是 (Schedule) | Cron 表达式，如 `"0 2 * * *"`（每天凌晨 2 点） | Cron 表达式非法 → Schedule 创建失败 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建即时备份 | `kubectl apply -f backup.yaml` | 否 |
| 创建定时备份 | `kubectl apply -f schedule.yaml` | 是 |
| 查看备份列表 | `kubectl get backups -n velero` | 是 |
| 查看备份详情 | `kubectl describe backup BACKUP_NAME -n velero` | 是 |
| 删除备份 | `kubectl delete backup BACKUP_NAME -n velero` | 否 |

## 操作步骤

### 1. 创建即时备份

#### 选择依据

- **namespace 范围**：指定 `includedNamespaces` 仅备份目标命名空间，避免全集群备份导致数据膨胀。
- **TTL**：生产环境建议 720h（30 天），测试环境可缩短至 168h（7 天）。

#### 最小创建

`backup.yaml`：

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: backup-full
  namespace: velero
spec:
  includedNamespaces:
  - default
  - production
  storageLocation: LOCATION_NAME
  ttl: 720h0m0s
```

```bash
kubectl apply -f backup.yaml
# expected: backup.velero.io/backup-full created
```

预期输出：

```text
backup.velero.io/backup-full created
```

### 2. 创建定时备份策略

`schedule.yaml`：

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
    - "*"
    storageLocation: LOCATION_NAME
    ttl: 168h0m0s
```

```bash
kubectl apply -f schedule.yaml
# expected: schedule.velero.io/daily-backup created
```

预期输出：

```text
schedule.velero.io/daily-backup created
```

### 3. 查看备份列表与状态

```bash
# 列出所有备份
kubectl get backups -n velero
# expected: STATUS 为 Completed
```

预期输出：

```text
NAME          STATUS      ERRORS   WARNINGS   CREATED
backup-full   Completed   0        0          2026-06-16T10:00:00Z
```

```bash
# 查看备份详情
kubectl describe backup backup-full -n velero
# expected: Phase: Completed
```

```text
NAME  STATUS  AGE
...
```

```bash
# 列出定时备份策略
kubectl get schedules -n velero
# expected: 至少返回 1 条
```

预期输出：

```text
NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
daily-backup   0 2 * * *   False     0        14h             1d
```

## 验证

### 控制面（tccli）

```bash
# 验证备份仓库正常
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
# 验证备份完成
kubectl get backup backup-full -n velero \
    -o jsonpath='{.status.phase}'
# expected: Completed

# 验证定时备份已注册
kubectl get schedules -n velero --no-headers | wc -l
# expected: >= 1
```

```text
NAME  STATUS  AGE
...
```

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。

## 清理

> **注意**：删除 Backup CRD 不会删除 COS 中已存储的备份数据。彻底清理需在 COS 控制台清空对应桶。

### 数据面

```bash
# 1. 删除即时备份
kubectl delete backup backup-full -n velero
# expected: backup.velero.io "backup-full" deleted

# 2. 删除定时备份
kubectl delete schedule daily-backup -n velero
# expected: schedule.velero.io "daily-backup" deleted
```

### 控制面（tccli）

```bash
# 3. 删除备份仓库（可选，需确认无其他备份依赖）
tccli tke DeleteBackupStorageLocation --region <Region> \
    --Name LOCATION_NAME
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f backup.yaml` 返回 `no matches for kind "Backup"` | `kubectl api-resources \| grep velero` | tke-backup 组件未安装 | `tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName tke-backup --AddonVersion VERSION` |
| kubectl `You are not authorized to perform this action` | `kubectl cluster-info` | 集群公网端点 CAM 策略拒绝(strategyId:240463971) | 联系管理员放通 CAM 策略（内网需VPN/IOA），或通过控制台 Web Shell 操作 |
| 备份仓库不可用 | `kubectl get backupstoragelocations -n velero` | BSL Phase 非 Available | 重新创建备份仓库，确认 COS 桶可访问 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 备份长时间处于 `InProgress` | `kubectl describe backup BACKUP_NAME -n velero` 查看日志 | 备份数据量大或 COS 上传慢 | 缩小 `includedNamespaces` 范围，减少备份对象数 |
| Schedule 未生成 Backup | `kubectl describe schedule SCHEDULE_NAME -n velero` 查看 Last Schedule | Cron 表达式错误或时区问题 | 修正 Cron 表达式，确认时区为 UTC |

## 下一步

- [恢复管理](../恢复管理/tccli%20操作.md) — 从备份中恢复集群资源
- [备份仓库](../备份仓库/tccli%20操作.md) — 管理备份存储位置
- [备份恢复实践](../备份恢复实践/tccli%20操作.md) — 端到端备份恢复示例

## 控制台替代

控制台：集群 → 备份中心 → 备份管理 → 新建备份 / 定时备份。
