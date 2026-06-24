# 使用对象存储 COS

> 对照官方：[使用对象存储 COS](https://cloud.tencent.com/document/product/457/44232) · page_id `44232`

## 概述

在 TKE 集群中使用腾讯云对象存储（COS）作为持久化存储。通过安装 COS-CSI 组件，以静态 PV/PVC 方式将 COS 存储桶挂载到 Pod。

**限制**：COS 仅支持静态 PV 绑定，不支持 StorageClass 动态供给。需先在 COS 控制台或通过 `cos` CLI 创建存储桶，再手动创建 PV 和 PVC。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:InstallAddon, tke:DescribeAddon, tke:DescribeClusterAddons
#    cos:HeadBucket, cos:GetBucket, cos:PutObject
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 确认 COS 存储桶存在
tccli cos HeadBucket --region <Region> --Bucket BUCKET_NAME
# expected: exit 0，存储桶可访问
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 COS-CSI 组件 | `InstallAddon --AddonName COS` | 否 |
| 查询组件安装状态 | `DescribeAddon --AddonName COS` | 是 |
| 创建 COS 存储桶 | `cos CreateBucket` | 否 |
| 创建 Secret（存储凭证） | `kubectl create secret generic` | 否 |
| 创建 PV/PVC 绑定 COS | `kubectl apply -f cos-pv-pvc.yaml` | 否 |

## 关键字段说明

以下说明 `InstallAddon` 中与 COS-CSI 相关的关键参数。完整参数定义见 `tccli tke InstallAddon --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddonName` | String | 是 | 固定值 `COS` | 填 `cos`（小写）→ `InvalidParameter.AddonName` |
| `AddonVersion` | String | 是 | 组件版本号，如 `1.0.0`。`DescribeAddonValues` 查询可用版本 | 版本不存在 → "addon version not found" |
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `RawValues` | String | 否 | 组件自定义配置 JSON 字符串 | 格式错误 → `InvalidParameter.RawValues` |

## 操作步骤

### 步骤 1：安装 COS-CSI 组件

#### 选择依据

- **AddonName**：固定为 `COS`，对应 cosfs 驱动（`com.tencent.cloud.csi.cosfs`）。
- **AddonVersion**：通过 `DescribeAddonValues` 查询当前集群可用的最新版本。

```bash
# 查询可用版本
tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName COS
# expected: 返回可用版本列表，取最新 Version
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

#### 最小安装

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName COS \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `ADDON_VERSION` | COS-CSI 组件版本 | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName COS` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName COS
# expected: Phase: "Succeeded", Status: "Running"
```

**预期输出**：

```json
{
    "Addon": {
        "AddonName": "COS",
        "AddonVersion": "1.0.0",
        "Phase": "Succeeded",
        "Status": "Running"
    }
}
```

### 步骤 3：创建 COS 访问凭证 Secret

#### 选择依据

COS-CSI 通过 Kubernetes Secret 存储访问 COS 的 SecretId/SecretKey，Pod 挂载时自动读取。

```bash
kubectl create secret generic cos-secret \
    --from-literal=SecretId=SECRET_ID \
    --from-literal=SecretKey=SECRET_KEY \
    -n default
# expected: secret/cos-secret created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `SECRET_ID` | 腾讯云 API 密钥 SecretId | [访问管理 → API 密钥管理](https://console.cloud.tencent.com/cam/capi) |
| `SECRET_KEY` | 腾讯云 API 密钥 SecretKey | 同上 |

### 步骤 4：创建 PV 和 PVC

#### 选择依据

- **driver**：`com.tencent.cloud.csi.cosfs`，COS-CSI 组件注册的 CSI 驱动名。
- **accessModes**：`ReadWriteMany`，COS 支持多 Pod 同时读写。
- **storageClassName**：自定义为 `cos`，区分其他存储类型。
- **persistentVolumeReclaimPolicy**：`Retain`，删除 PVC 时保留 COS 存储桶数据。

#### 最小配置

`cos-pv-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cos-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cosfs
    volumeHandle: cos-pv
    volumeAttributes:
      url: "https://BUCKET_NAME.cos.REGION.myqcloud.com"
      bucket: "BUCKET_NAME"
      path: "/"
    nodePublishSecretRef:
      name: cos-secret
      namespace: default
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cos
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cos-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cos
  volumeName: cos-pv
```

```bash
kubectl apply -f cos-pv-pvc.yaml
# expected: persistentvolume/cos-pv created, persistentvolumeclaim/cos-pvc created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `BUCKET_NAME` | COS 存储桶名称（不含 `-APPID` 后缀） | [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) |

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName COS` | `Phase: "Succeeded"` |
| 存储桶可访问 | `tccli cos HeadBucket --region <Region> --Bucket BUCKET_NAME` | exit 0 |

### 数据面（kubectl）

```bash
kubectl get pv cos-pv
# expected: STATUS: Available（PVC 未绑定时）

kubectl get pvc cos-pvc -n default
# expected: STATUS: Bound
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`kubectl delete pv cos-pv` 会移除 PV 资源，但**不会删除 COS 存储桶及其中数据**。存储桶数据需在 [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) 手动清理，否则持续产生 COS 存储费用。

### 1. 清理前状态检查

```bash
kubectl get pv cos-pv
kubectl get pvc cos-pvc -n default
# 确认是待删除的目标资源
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC 和 PV

```bash
kubectl delete pvc cos-pvc -n default
# expected: persistentvolumeclaim "cos-pvc" deleted

kubectl delete pv cos-pv
# expected: persistentvolume "cos-pv" deleted
```

### 3. 删除 Secret（可选）

```bash
kubectl delete secret cos-secret -n default
# expected: secret "cos-secret" deleted
```

### 4. 验证已删除

```bash
kubectl get pv cos-pv 2>&1
# expected: Error from server (NotFound): ... "cos-pv" not found

kubectl get pvc cos-pvc -n default 2>&1
# expected: Error from server (NotFound): ... "cos-pvc" not found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName COS` 查看可用版本 | 指定的 `AddonVersion` 在目标集群中不存在 | 从 `DescribeAddonValues` 输出中取正确的版本号填入 |
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 字段值 | 填写了小写 `cos` 而非大写 `COS` | 使用 `--AddonName COS`（大写） |
| `kubectl apply` 报 `driver com.tencent.cloud.csi.cosfs not found` | `kubectl get csidriver` 查看已注册的 CSI 驱动 | COS-CSI 组件未安装或未就绪 | 回到步骤 1 安装 COS-CSI，确认 `Phase: "Succeeded"` |
| `HeadBucket` 返回 `NoSuchBucket` | 检查存储桶名称是否包含 APPID 后缀 | 存储桶名称错误或存储桶不存在 | 在 [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) 确认存储桶名称 |

### 安装成功但 PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 Pending | `kubectl describe pvc cos-pvc -n default` 查看 Events | PV 不存在或 `volumeName` 不匹配 | 确认 PV 已创建（`kubectl get pv cos-pv`），且 PVC 的 `volumeName` 与 PV 的 `name` 一致 |
| Pod 挂载 COS 失败 | `kubectl describe pod POD_NAME` 查看 Events | Secret 中的 SecretId/SecretKey 错误或存储桶 URL 错误 | 检查 `cos-pv-pvc.yaml` 中 `url`、`bucket` 字段，以及 `cos-secret` 中的 SecretId/SecretKey |
| Pod 挂载后无法写入 | `kubectl exec POD_NAME -- ls /mnt/cos` 检查挂载点 | COS 存储桶权限不足或 Secret 凭证无写入权限 | 确认 API 密钥含 `cos:PutObject` 权限，存储桶 ACL 允许写入 |

## 下一步

- [使用云硬盘 CBS](../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md)
- [使用文件存储 CFS](../使用文件存储CFS/PV和PVC管理文件存储/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../PV和PVC的绑定规则/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 COS-CSI 组件；[COS 控制台](https://console.cloud.tencent.com/cos5) 管理存储桶。
