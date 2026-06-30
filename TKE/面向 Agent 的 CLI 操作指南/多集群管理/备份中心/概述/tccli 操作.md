# 备份中心概述

> 对照官方：[备份概述](https://cloud.tencent.com/document/product/457/90490) · page_id `90490`

## 概述

TKE 备份中心为容器化应用提供备份、恢复、迁移一体化方案，底层基于开源 Velero 实现。备份中心管理平面由 `tccli tke` 的 `BackupStorageLocation` 系列接口完成，集群内备份/恢复策略由 `kubectl` 操作 Velero CRD 完成。

**核心概念**：

| 概念 | 说明 | 管理方式 |
|------|------|---------|
| 备份仓库（BackupStorageLocation） | 备份数据存储位置，对应 COS 存储桶 | `tccli tke` API |
| 备份（Backup） | 一次性备份任务，创建即触发 | `kubectl apply -f backup.yaml` |
| 定时备份（Schedule） | 按 Cron 表达式定期自动备份 | `kubectl apply -f schedule.yaml` |
| 恢复（Restore） | 从备份恢复 K8s 资源对象 | `kubectl apply -f restore.yaml` |

**能力边界**：当前支持 K8s 资源对象（Namespace、Deployment、Service、ConfigMap 等）的备份恢复，不支持云硬盘 CBS、负载均衡 CLB 等云基础设施资源的恢复。COS 存储桶名称必须以 `tke-backup` 为前缀。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateBackupStorageLocation
#    tke:DescribeBackupStorageLocations
#    tke:DeleteBackupStorageLocation
#    tke:InstallAddon
# 验证：
tccli tke DescribeBackupStorageLocations --region <Region>
# expected: exit 0，返回仓库列表（可为空）

# 4. 检查 kubectl 版本（操作 Velero CRD 需 kubectl 可达集群）
kubectl version --client
# expected: Client Version 含版本号

# 5. 检查 COS 服务可用
tccli cos ListBuckets --region <Region>
# expected: exit 0，返回桶列表（可为空）
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

### 资源检查

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

```bash
# 6. 确认目标集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"
```

**预期输出**（以 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-test-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-xxxxxxxx",
                "ClusterCIDR": "10.0.0.0/16"
            }
        }
    ],
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建备份仓库 | `tccli tke CreateBackupStorageLocation --region <Region> --StorageRegion REGION --Bucket tke-backup-BUCKET_SUPPLIER-APPID --Name LOCATION_NAME` | 否 |
| 查看备份仓库 | `tccli tke DescribeBackupStorageLocations --region <Region>` | 是 |
| 删除备份仓库 | `tccli tke DeleteBackupStorageLocation --region <Region> --Name LOCATION_NAME` | 是 |
| 安装 tke-backup 组件 | `tccli tke InstallAddon --ClusterId CLUSTER_ID --AddonName tke-backup --AddonVersion VERSION` | 是 |
| 创建备份 | `kubectl apply -f backup.yaml` | 否 |
| 创建定时备份 | `kubectl apply -f schedule.yaml` | 否 |
| 创建恢复 | `kubectl apply -f restore.yaml` | 否 |

## 操作步骤

### 概念 1：备份仓库（BackupStorageLocation）

备份仓库定义备份数据存储位置。每个备份仓库对应一个 COS 存储桶。创建备份仓库前须先创建 COS 桶。

```bash
# 创建备份仓库
tccli tke CreateBackupStorageLocation --region <Region> \
    --StorageRegion REGION \
    --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID' \
    --Name LOCATION_NAME
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "b3f35196-1fc5-4c26-8025-0f74378c054e"
}
```

```bash
# 查看备份仓库
tccli tke DescribeBackupStorageLocations --region <Region>
# expected: BackupStorageLocationSet 含已创建的仓库
```

预期输出：

```json
{
    "BackupStorageLocationSet": [
        {
            "Name": "backup-default",
            "Bucket": "tke-backup-example-1234567890",
            "StorageRegion": "ap-guangzhou",
            "State": "Available"
        }
    ],
    "RequestId": "..."
}
```

### 概念 2：备份（Backup）

定义一个一次性备份任务。创建 Backup CRD 即触发备份操作。备份数据存储在 COS 中，删除 Backup CRD 不会删除 COS 中的备份数据。

> **前提**：需 Velero 在集群内安装（`tke-backup` 组件），kubectl 端点可达。

```bash
kubectl apply -f backup.yaml
# expected: backup.velero.io/BACKUP_NAME created
```

### 概念 3：定时备份（Schedule）

定义一个周期性备份策略，按 Cron 表达式定期生成 Backup 对象并触发备份。

> **前提**：需 Velero 在集群内安装，kubectl 端点可达。

```bash
kubectl apply -f schedule.yaml
# expected: schedule.velero.io/SCHEDULE_NAME created
```

### 概念 4：恢复（Restore）

从已有备份中恢复 K8s 资源对象到目标集群。创建即触发恢复操作。

> **前提**：需 Velero 在集群内安装，kubectl 端点可达。

```bash
kubectl apply -f restore.yaml
# expected: restore.velero.io/RESTORE_NAME created
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
# 验证 Velero 组件运行
kubectl get pods -n velero
# expected: velero pod Running
```

```text
NAME  STATUS  AGE
...
```

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。

## 清理

> **警告**：删除备份仓库不会删除 COS 中的备份数据。删除 Backup CRD 也不会删除 COS 中已存储的备份数据。彻底清理需先清空 COS 桶。

```bash
# 删除备份仓库
tccli tke DeleteBackupStorageLocation --region <Region> \
    --Name LOCATION_NAME
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateBackupStorageLocation` 返回 `InvalidParameter` "bucket prefix must be tke-backup" | 检查 `--Bucket` 参数值 | COS 桶名不以 `tke-backup` 为前缀（API 硬约束） | `tccli cos PutBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'` |
| `CreateBackupStorageLocation` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 缺少 `tke:CreateBackupStorageLocation` 权限 | 联系管理员授予 TKE 备份仓库管理权限 |
| `kubectl apply -f backup.yaml` 返回 `no matches for kind "Backup"` | `kubectl api-resources \| grep velero` | tke-backup 组件未安装 | `tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName tke-backup --AddonVersion VERSION` |
| kubectl unreachable | `kubectl cluster-info` | 集群公网端点 CAM 策略拒绝（strategyId:240463971） | 联系管理员放通 CAM 策略，或通过控制台 Web Shell 操作；内网需VPN/IOA |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 备份仓库 State 非 Available | `tccli tke DescribeBackupStorageLocations --region <Region> --Names '["LOCATION_NAME"]'` 查看 State | COS 桶不存在或权限不足 | 确认 COS 桶存在且 CAM 策略允许访问 |

## 下一步

- [备份仓库](../备份仓库/tccli%20操作.md) — 创建和管理备份存储位置
- [备份管理](../备份管理/tccli%20操作.md) — 创建备份和定时备份策略
- [恢复管理](../恢复管理/tccli%20操作.md) — 从备份恢复集群资源
- [备份恢复实践](../备份恢复实践/tccli%20操作.md) — 端到端备份恢复示例

## 控制台替代

参见 [TKE 备份中心 - 腾讯云控制台](https://console.cloud.tencent.com/tke2/backup)
