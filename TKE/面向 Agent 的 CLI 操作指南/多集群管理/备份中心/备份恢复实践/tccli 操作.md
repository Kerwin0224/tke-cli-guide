# 备份恢复实践

> 对照官方：[备份恢复实践](https://cloud.tencent.com/document/product/457/107707) · page_id `107707`

## 概述

端到端演示如何通过 CLI 完成 TKE 集群备份与恢复全流程：使用 `tccli tke` 管理备份仓库，使用 `kubectl` 操作 Velero CRD 进行集群资源的定期备份和灾难恢复。

**模拟场景**：`kube-system` 命名空间下 `service-controller` 资源误删后从已有备份恢复。

> **注意**：步骤 3-5 依赖 kubectl 访问集群。需确保 Velero（tke-backup 组件）在集群内安装，kubectl 端点可达。

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
#    tke:DescribeAddon
# 验证：
tccli tke DescribeBackupStorageLocations --region <Region>
# expected: exit 0，返回仓库列表（可为空）

# 4. 检查 kubectl 集群连通性
kubectl cluster-info
# expected: Kubernetes control plane 可达

# 5. 检查 COS 服务可用
tccli cos ListBuckets --region <Region>
# expected: exit 0
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

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

### 资源检查

```bash
# 6. 确认目标集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"

# 7. 确认 COS 桶存在（桶名以 tke-backup 为前缀）
tccli cos HeadBucket --region <Region> \
    --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'
# expected: exit 0
```

预期输出（DescribeClusters）：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-test-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "172.16.0.0/16",
                "ServiceCIDR": "10.96.0.0/16",
                "VpcId": "vpc-xxxxxxxx",
                "MaxNodePodNum": 64,
                "MaxClusterServiceNum": 32768
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建备份仓库 | `tccli tke CreateBackupStorageLocation` | 否 |
| 安装 tke-backup 组件 | `tccli tke InstallAddon --AddonName tke-backup` | 是 |
| 创建定时备份 | `kubectl apply -f backupschedule.yaml` | 否 |
| 执行即时备份 | `kubectl apply -f backup.yaml` | 否 |
| 执行恢复 | `kubectl apply -f restore.yaml` | 否 |
| 查看恢复状态 | `kubectl get restores -n velero` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.existingResourcePolicy` | String | 否 | `none`（不覆盖已存在同命资源）或 `update`（覆盖）。生产建议 `none` | 选 `update` 可能覆盖正在运行的健康资源 |
| `spec.includedNamespaces` | Array | 否 | 备份/恢复的目标命名空间列表。省略则含全部命名空间 | 范围过大可能致备份数据量膨胀、恢复超时 |
| `spec.ttl` | String | 否 | 备份保留时长，格式 `{h}h{m}m{s}s`。默认 720h（30 天） | 过短则恢复时备份可能已被自动清理 |
| `Bucket` | String | 是 | COS 桶完整名称，格式 `BUCKET_NAME-APPID`。桶名必须以 `tke-backup` 为前缀 | 桶名不以 `tke-backup` 开头 → `InvalidParameter` |

## 操作步骤

### 步骤 1：创建备份仓库

```bash
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

验证仓库可用：

```bash
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

### 步骤 2：安装 tke-backup 组件

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName tke-backup \
    --AddonVersion VERSION
# expected: exit 0，返回 RequestId
```

验证组件安装：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName tke-backup
# expected: Status: "Enabled"
```

预期输出：

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

### 步骤 3：创建定时备份策略

按 Cron 表达式定时备份指定命名空间（kubectl 操作）。

> **前提**：需 Velero 在集群内安装，kubectl 端点可达。

`backupschedule.yaml`：

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 1 * * *"
  template:
    includedNamespaces:
    - kube-system
    - default
    ttl: 720h0m0s
```

```bash
kubectl apply -f backupschedule.yaml
# expected: schedule.velero.io/daily-backup created
```

预期输出：

```text
schedule.velero.io/daily-backup created
```

### 步骤 4：手动触发即时备份

模拟误删前先创建一次即时备份确保有可恢复的数据。

`backup.yaml`：

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: pre-accident-backup
  namespace: velero
spec:
  includedNamespaces:
  - kube-system
  ttl: 168h0m0s
```

```bash
kubectl apply -f backup.yaml
# expected: backup.velero.io/pre-accident-backup created
```

预期输出：

```text
backup.velero.io/pre-accident-backup created
```

查看备份状态：

```bash
kubectl get backups -n velero
# expected: pre-accident-backup STATUS 为 Completed
```

预期输出：

```text
NAME                 AGE   STATUS
pre-accident-backup  2m    Completed
```

### 步骤 5：模拟误删并执行恢复

模拟误删 `kube-system` 下的 `service-controller`：

```bash
kubectl delete deployment service-controller -n kube-system
# expected: deployment.apps "service-controller" deleted
```

预期输出：

```text
deployment.apps "service-controller" deleted
```

从备份恢复：

`restore.yaml`：

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-service-controller
  namespace: velero
spec:
  backupName: pre-accident-backup
  includedNamespaces:
  - kube-system
  includedResources:
  - deployments
  namespaceMapping:
    kube-system: kube-system
  existingResourcePolicy: none
```

```bash
kubectl apply -f restore.yaml
# expected: restore.velero.io/restore-service-controller created
```

预期输出：

```text
restore.velero.io/restore-service-controller created
```

监控恢复进度：

```bash
kubectl get restores -n velero -w
# expected: STATUS 由 InProgress 变为 Completed
```

预期输出：

```text
NAME                         AGE   STATUS
restore-service-controller   1m    Completed
```

验证资源已恢复：

```bash
kubectl get deployment service-controller -n kube-system
# expected: deployment 存在且 READY
```

预期输出：

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
service-controller    2/2     2            2           2m
```

## 验证

### 控制面（tccli）

```bash
# 验证备份仓库
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: State: "Available"

# 验证 tke-backup 组件
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName tke-backup
# expected: Status: "Enabled"
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
# 验证 Velero Pod 运行
kubectl get pods -n velero
# expected: velero pod Running

# 验证备份完成
kubectl get backups -n velero
# expected: pre-accident-backup STATUS Completed

# 验证恢复完成
kubectl get restores -n velero
# expected: restore-service-controller STATUS Completed
```

```text
NAME  STATUS  AGE
...
```

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。

## 清理

> **警告**：删除 Backup CRD 不会删除 COS 中备份数据。删除 BackupStorageLocation 也不会删除 COS 桶或其中备份文件。彻底清理需先清空 COS 桶。

### 数据面 — 先于控制面

```bash
# 1. 删除恢复记录
kubectl delete restore restore-service-controller -n velero
# expected: restore.velero.io "restore-service-controller" deleted

# 2. 删除备份和定时备份
kubectl delete backup pre-accident-backup -n velero
# expected: backup.velero.io "pre-accident-backup" deleted

kubectl delete schedule daily-backup -n velero
# expected: schedule.velero.io "daily-backup" deleted
```

### 控制面（tccli）

```bash
# 3. 卸载 tke-backup 组件（可选）
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName tke-backup
# expected: exit 0

# 4. 删除备份仓库
tccli tke DeleteBackupStorageLocation --region <Region> \
    --Name LOCATION_NAME
# expected: exit 0

# 5. 验证已删除
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: BackupStorageLocationSet 返回空列表
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateBackupStorageLocation` 返回 `InvalidParameter` "bucket prefix must be tke-backup" | 检查 `--Bucket` 参数值 | COS 桶名不以 `tke-backup` 为前缀 | `tccli cos PutBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'` |
| `InstallAddon --AddonName tke-backup` 返回 `ResourceNotFound` | `tccli tke GetTkeAppChartList --Kind addon --ClusterType MANAGED_CLUSTER` | AddonName 拼写错误或集群类型不匹配 | 确认 AddonName 为 `tke-backup`（小写，连字符） |
| `kubectl apply -f backup.yaml` 返回 `no matches for kind "Backup"` | `kubectl api-resources \| grep velero` | tke-backup 组件未安装 | `tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName tke-backup --AddonVersion VERSION` |
| `kubectl apply -f restore.yaml` 完成但资源未恢复 | `kubectl describe restore RESTORE_NAME -n velero` 查看 warnings | `existingResourcePolicy: none` 且目标资源已存在 | 设置 `existingResourcePolicy: update` 或先 `kubectl delete` 目标资源 |
| kubectl 不可达 | `kubectl cluster-info` | 集群公网端点 CAM 策略拒绝(strategyId:240463971) | 联系管理员放通 CAM 策略（内网需VPN/IOA），或通过控制台 Web Shell 操作 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 备份仓库 State 非 Available | `tccli tke DescribeBackupStorageLocations --region <Region> --Names '["LOCATION_NAME"]'` 查看 State | COS 桶不存在或权限不足 | 确认 COS 桶存在、地域一致、CAM 策略允许访问 |
| 恢复任务 Phase Failed | `kubectl describe restore RESTORE_NAME -n velero` 查看 errors | backupName 不存在或备份数据已过期 | 确认备份仍存在且未过期，重新创建 Restore |

## 下一步

- [备份中心概述](../概述/tccli%20操作.md) — 核心概念总览
- [备份管理](../备份管理/tccli%20操作.md) — 备份和定时备份策略详解
- [恢复管理](../恢复管理/tccli%20操作.md) — 恢复操作详解

## 控制台替代

参见 [TKE 备份中心 - 腾讯云控制台](https://console.cloud.tencent.com/tke2/backup)
